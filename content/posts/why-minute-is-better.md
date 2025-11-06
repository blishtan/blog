+++
title = "Why .hour is Better Than 3600"
description = "Making TimeIntervals readable in Swift"
type = ["posts","post"]
tags = [
    "swift",
    "ios",
    "code_quality",
    "swift_tips",
    "best_practices",
]
date = "2025-11-06"
categories = [
    "Development",
    "Swift",
]
[ author ]
  name = "Blishtan"
+++

Naming variables is said to be the hardest part of software development. Many developers don't care, many don't know how to do it. When I see a magic number in the code, I create a constant. It immediately communicates the purpose or what this number stands for.

In Swift `TimeInterval` is a typealias for `Double` and it represents seconds in various functions or constructors. We've all been there - needing `TimeInterval` for postponing notifications, scheduling checks... you name it.

## Magic Numbers

It is okay to create an animation that takes 4 seconds:

```swift
UIView.animate(withDuration: 4, animations: {
    // animation code
})
```

The number `4` is clear and simple. 

Setting a check after 30 days might look confusing:

```swift
Date().addingTimeInterval(2_592_000)
```

This creates mental overhead and the code is not readable. What does `2_592_000` represent? You'd have to calculate it mentally or add a comment.

## The Solution - Swift Extension

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
Date().addingTimeInterval(30 * .day)
```

For working with past dates, I created this little extension:

```swift
extension TimeInterval {
    var ago: Self {
        -self
    }
}
```

...which can combine with extension before and could lead to nice readable code:

```swift
  let lastValidDate = Date(timeInterval: .year.ago, since: Date())
```


This is immediately clear, maintainable, and eliminates the need for mental arithmetic. The intent is obvious, and future developers (including yourself) will thank you for it.
