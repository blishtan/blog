+++
title = "I Am Honest Developer"
description = "Making code more readable, clean, and intuitive"
type = ["posts","post"]
tags = [
    "development",
    "career",
    "programming",
]
date = "2025-11-25"
categories = [
    "Development",
]
[ author ]
  name = "Blishtan"
+++

## Honesty in Clean Code

From the very beginning of my life I was taught to be honest. It sticks with me, and I apply this virtue in my professional life as well. It may sound vague, but a few principles help paint a bigger picture of what honest code means to me.

## Vague naming
  
I don’t like vague names.  
```swift  
func displayItems()  
```  
This forces me to open the implementation to understand what it actually does.
I prefer naming that is clear.  
```swift  
func displayPendingOrders()  
```  

## Exposing details
I don’t like exposing implementation details.  
```swift  
public func add(order: Order) {
  // …  
	increaseOrderCounter()  
}
  
public func increaseOrderCounter() {  
// …
}  
```  
Exposing function or property encourages user of the class, struct or in general any API to use it. Even if the author never intended to do so.
I prefer to hide such details.  
```swift  
private func increaseOrderCounter()  
```  

## Lack of honesty

I don’t like dishonesty.  
```swift  
private func increaseOrderCounter() {  
	orderCounter -= 1  
}  
```
For me it is pure lack of interest to deliver good code and to communicate what is truly happening.
I like being honest.
```swift  
private func increaseOrderCounter() {  
	orderCounter += 1  
}  
```  

## Context

I don’t like my contexts big.  
```swift
struct Dashboard: View {
// …  
  struct Section: View {
  // …  
  struct DashboardSectionPrimaryButton: View  
  }
}
```
I don't need to know every detail along the way. I like to know as little as possible.
I prefer smaller and focused contexts.
```swift
extension Dashboard.Section {  
	struct PrimaryButton {  
	// …  
	}  
}  
```  

## Verbosity

I am honest.  
```swift
let descriptionString: String
```
...but not too honest.
```swift
let description: String
```  
## Conclusion 
I am honest developer and I like to be honest to myself and to my colleagues.   
Clear intention should follow clear naming.   
Mental load should be minimal.
Writing maintainable, readable, honest code is a choice — and a daily discipline.
