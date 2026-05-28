+++
title = "Bridging Callback APIs to async/await"
description = "Safely bridging Operation-based callback APIs to Swift Structured Concurrency, with cancellation handled correctly"
type = ["posts","post"]
tags = [
    "swift",
    "ios",
    "concurrency",
    "async_await",
    "best_practices",
]
date = "2026-05-28"
categories = [
    "Development",
    "Swift",
]
[ author ]
  name = "Blishtan"
+++

New Swift projects are utilising Structured Concurrency. Long-living codebases on the other hand are full of older callback-based APIs that hand you an `Operation` and call you back with a `Result`.
The gap between those two worlds looks trivial to close — until you think about **cancellation**.

---

## The problem

Suppose you have a legacy API shaped like this:

```swift
// Hands back an `Operation` you can cancel, and calls `completion` when done.
func fetch(
    argument: String,
    completion: @escaping (Result<String, Error>) -> Void
) -> Operation
```

You want to call it like this instead:

```swift
let result = try await fetch(argument: "Foo")
```

The standard tool for this is `withCheckedThrowingContinuation`. A first attempt looks clean:

```swift
func fetch(argument: String) async throws -> String {
    try await withCheckedThrowingContinuation { continuation in
        _ = fetch(argument: argument) { result in
            continuation.resume(with: result)
        }
    }
}
```

This *works* — but it quietly ignores two things:

1. The `Operation` is discarded (`_ =`). Nothing ever calls `.cancel()` on it.
2. If the surrounding `Task` is cancelled, the `await` keeps waiting anyway.

A continuation must be resumed **exactly once**. Cancellation is where that guarantee gets hard: the cancel signal and the operation's own completion can arrive in *either order*, possibly on *different threads*.

---

## The public API

`OperationBridge` is a value type whose only job is to run one bridged operation:

```swift
public struct OperationBridge {
    public let tag: String   // a label, used for logging/diagnostics

    public init(tag: String) {
        self.tag = tag
    }

    public func execute<T>(
        operation: (@escaping ResultHandler<T>) -> Operation
    ) async throws -> T {
        // ...
    }
}

public extension OperationBridge {
    typealias ResultHandler<T> = (Result<T, Error>) -> Void
}
```

You give `execute` a closure. That closure receives a `ResultHandler` (the completion callback) and must return the `Operation` it kicked off. Usage at the call site is a one-liner:

```swift
func fetch(argument: String) async throws -> String {
    try await OperationBridge(tag: "Fetch").execute { resultHandler in
        fetch(argument: argument, completion: resultHandler)
    }
}
```

---

## Execute function

Here is the whole method. It is small, and every line earns its place:

```swift
public func execute<T>(
    operation: (@escaping ResultHandler<T>) -> Operation
) async throws -> T {
    let operationHolder = CancellableOperation<T>(tag: tag)

    return try await withTaskCancellationHandler {
        try await withCheckedThrowingContinuation { continuation in
            // Start the work; the callback resumes the continuation.
            let operation = operation { result in
                continuation.resume(with: result)
            }
            // Hand the live operation + continuation to the holder.
            operationHolder.hold(operation, continuation: continuation)
        }
    } onCancel: {
        operationHolder.cancel()
    }
}
```

Two structured-concurrency primitives are stacked here:

- **`withCheckedThrowingContinuation`** turns the callback into a task. When the operation calls back, `continuation.resume(with:)` delivers the `Result` to the awaiting code.
- **`withTaskCancellationHandler`** gives us an `onCancel` block that runs the moment the enclosing `Task` is cancelled.

But notice the bridge doesn't *directly* connect those two. The `Operation` and the `continuation` are both handed to a separate object — `CancellableOperation` — and `onCancel` just calls `.cancel()` on it. That indirection exists for one reason — a race.

---

## The race condition

`onCancel` and the operation's completion callback are **not** synchronized with each other. Consider the timeline when a `Task` is cancelled:

**Case A** — cancel arrives BEFORE the operation is created
```swfit
  1. Task.cancel()
  2. onCancel runs           — but hold() hasn't been called yet
  3. continuation body runs  — Operation is created
```

**Case B** — cancel arrives WHILE the operation is in flight
```swift
  1. hold(operation, continuation)  — operation is tracked
  2. Task.cancel()
  3. onCancel runs                  — live operation is cancelled
```

In **Case A**, `onCancel` fires before there is anything to cancel. If we did nothing, the operation would be created *after* cancellation and run to completion anyway — and the continuation might be resumed by a stale callback.

We also must guarantee the continuation is resumed **exactly once**:

- Resume it zero times → the `await` hangs forever.
- Resume it twice → a crash (`SWIFT_TASK_DEBUG`: continuation misuse).

So we need a small, thread-safe **state machine** that records *Has cancel happened?* and *Do we have an operation to cancel yet?* — and reconciles the two no matter which arrives first.

---

## CancellableOperation — the state machine

`CancellableOperation<T>` is a `Sendable` class guarding three states behind a lock:

```swift
public final class CancellableOperation<T>: Sendable {
    private let lock = OSAllocatedUnfairLock<State<T>>(initialState: .waiting)

    private enum State<ContinuationResult> {
        case waiting                                              // nothing yet
        case holding(Operation, CheckedContinuation<ContinuationResult, Error>)
        case cancelled                                            // terminal
    }
}
```

`OSAllocatedUnfairLock` holds the state *inside* the lock, so the state can only be touched while the lock is held — the type makes the unsafe access impossible to forget.

There are exactly two entry points, and each is a transition table.

### hold — "here is the live operation"

Called from inside the continuation body, once the operation exists:

```swift
public func hold(_ operation: Operation, continuation: CheckedContinuation<T, Error>) {
    lock.withLock { state in
        switch state {
        case .waiting:
            // Normal path: start tracking it.
            state = .holding(operation, continuation)

        case .cancelled:
            // Case A! Cancel already happened. Tear down immediately.
            operation.cancel()
            continuation.resume(throwing: CancellationError())

        case .holding:
            // Defensive: a second operation on the same bridge. Replace it.
            state = .holding(operation, continuation)
        }
    }
}
```

The `.cancelled` branch is the fix for **Case A**: the operation was created *after* the cancel, so `hold` itself does the cancelling and resumes the continuation with `CancellationError`.

### cancel — "the Task was cancelled"

Called from `onCancel`:

```swift
public func cancel() {
    lock.withLock { state in
        switch state {
        case .waiting:
            // Case A: no operation yet. Just remember the cancel.
            state = .cancelled

        case let .holding(operation, continuation):
            // Case B: cancel the live operation, resume the continuation.
            operation.cancel()
            continuation.resume(throwing: CancellationError())
            state = .cancelled

        case .cancelled:
            break   // idempotent — already done
        }
    }
}
```

The `.waiting → .cancelled` transition is the other half of **Case A**: `cancel` can't act yet, so it leaves a note. When `hold` later runs and sees `.cancelled`, it finishes the job.

---

## Takeaways

`OperationBridge` is barely 60 lines, but it packs a few lessons worth mentioning:

1. **Bridging callbacks to `async`/`await` is easy; bridging *cancellation* is the hard part.** `withCheckedThrowingContinuation` alone is not enough.
2. **A continuation must resume exactly once.** Treat that as an invariant and design backwards from it.
3. **When two events can arrive in any order, model it as a state machine.** Three states — `waiting`, `holding`, `cancelled` — collapse every race into a handful of explicit, testable transitions.
4. **Put the lock around the state, not next to it.** `OSAllocatedUnfairLock` storing the value makes unsynchronized access a compile-time impossibility.
