+++
title = "Why .minute is Better Than 3600"
description = "Making time intervals readable and maintainable in Swift"
type = ["posts","post"]
tags = [
    "swift",
    "ios",
    "code-quality",
    "best-practices",
]
date = "2025-11-02"
categories = [
    "Development",
    "Swift",
]
[ author ]
  name = "Blishtan"
+++

Naming variables is said to be the hardest part of software development. Many developers don't care, many don't know how to do it. When I see a magic number in the code, I create a constant. It immediately communicates the purpose or what this number stands for.

`TimeInterval` is a typealias for `Double` and it represents seconds in various functions or constructors. We've all been there where `TimeInterval` is needed for postponing notifications, scheduling checks... you name it.

## When Magic Numbers Work

It is okay to create an animation that takes 4 seconds:

```swift
UIView.animate(withDuration: 4, animations: {
    // animation code
})
```

The number `4` is clear and simple. But what about more complex time intervals?

## The Problem with Magic Numbers

Setting a check after 30 days might look confusing:

```swift
Date.addingTimeInterval(2_592_000)
```

This creates mental overhead and the code is not readable. What does `2_592_000` represent? You'd have to calculate it mentally or add a comment.

## The Solution

Almost every package and project I work on contains this extension for `TimeInterval`:

```swift
extension TimeInterval {
    static let minute: TimeInterval = 60
    static let hour: TimeInterval = 60 * .minute
    static let day: TimeInterval = 24 * .hour
    static let week: TimeInterval = 7 * .day
    static let month: TimeInterval = 31 * .day
    static let year: TimeInterval = 365 * .day
}
```

Now the code becomes self-documenting:

```swift
Date.addingTimeInterval(30 * .day)
```

This is immediately clear, maintainable, and eliminates the need for mental arithmetic. The intent is obvious, and future developers (including yourself) will thank you for it.
