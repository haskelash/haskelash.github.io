---
layout: post
title: "Making a Queue in Swift with a Recursive Enum"
date: 2021-02-23 00:00:00
image: '/assets/img/'
description:
tags:
- swift
- data-structures
categories:
twitter_text:
---

A queue is a very useful data structure, so naturally we might want to implement one in Swift.

### A Quick `Queue` Refresher

A queue is a list of generic values that allows you to insert (enqueue) values and remove (dequeue) values. The values will always be dequeued in the order in which they were enqueued (first in, first out, or FIFO order). A queue is most easily implemented by using a linked list, where you enqueue by adding to the end of the list and dequeue by popping from the start of the list. A linked list itself is made up of nodes, each of which contains a generic value and an optional reference to the next node. So what we want is to end up with something like this:

{% highlight swift %}
let queue = Queue<String>() // queue is empty
queue.enqueue("a")          // queue is "a"
queue.enqueue("b")          // queue is "a" -> "b"
queue.enqueue("c")          // queue is "a" -> "b" -> "c"
let _ = queue.dequeue()     // queue is "b" -> "c"
let _ = queue.dequeue()     // queue is "c"
queue.enqueue("d")          // queue is "c" -> "d"
let _ = queue.dequeue()     // queue is "d"
let _ = queue.dequeue()     // queue is empty
{% endhighlight swift %}

### The Sad, Simple `Struct`

Ok, on to implementing. Right off the bat we would like to use a `struct` because it’s a value type (as opposed to a `class` which is a reference type) and it would be nice for our data structure to be a value type as well. However, when we try to implement a `Node` struct, we immediately run into a problem:

{% highlight swift %}
struct Node<Element> {
  let value: Element
  let next: Node<Element>?
}
{% endhighlight swift %}

This won’t build, and the compiler will tell you that structs can’t have recursive properties. Well that’s not fun. Then next thing to do would be to use a class and store everything by reference. But this is Swift. We want it to be Swifty! Luckily, there is a third type...

### The Extraordinary `Enum` with an Incredible `Inderect` Case

While Swift disallows structs from having recursive properties, it very much allows this for `enums` with associated values. All you have to do is use the `indirect` keyword for the recursive case. We can define a queue very simply like this:

{% highlight swift %}
enum Queue<Element> {
  case empty
  indirect case node(value: Element, next: Queue<Element>)
}
{% endhighlight swift %}

The indirect keyword tells the compiler that the case contains a recursive reference to the enum itself. Alternatively you can put the `indirect` keyword before `enum`, but I prefer the clarity of having it before the recursive case.

What this implementation says is that a queue is either empty, or else it contains a value and another queue. That second queue itself may be empty (in which case the queue is finished) or hold another value and another queue, and so on. We don’t even need a node type - the queue is totally self-contained. We also don’t need to make the `next` associated value optional. It will always exist, but it will be the `empty` case if it’s the end of the list.

### Enter `Enqueue`

What does the `enqueue` function look like for this enum? Let's take a look:

{% highlight swift %}
mutating func enqueue(_ newValue: Element) { // 1
  guard case .node(let value, var next) = self else { // 2
    self = .node(value: newValue, next: .empty) // 3
    return
  }
  next.enqueue(newValue) // 4
  self = .node(value: value, next: next) // 5
}
{% endhighlight swift %}

1. The function takes a single `Element`. Since this function changes the contents of `self`, it must be marked as `mutating`.
2. Check that `self` is the `.node` case, and only then continue on to step 4. Otherwise, go to step 3 and return early. Notice that when we unwrap the `next` associated value, we do so with `var`. This is important because we will change it in step 4.
3. If we are here then the queue is empty. So we set `self` to be a new queue with a value of `newValue` and an empty `next`, and we're done.
4. We recursively enqueue `newValue` into `next`. This travels all the way down the queue until it hits an `.empty` case.
5. We set `self` to be a `.node` with an associated value of `value` (which it was already), but with the updated `next` (which contains `newValue` at the end).

### Delightful `Dequeue`

So we've implemented `enqueue` recursively, but what about `dequeue`. Thankfully `dequeue` is much simpler:

{% highlight swift %}
mutating func dequeue() -> Element? { // 1
  guard case let .node(value, next)  = self else { return nil } // 2
  self = next // 3
  return value // 4
}
{% endhighlight swift %}

1. The function removes the first `Element` from the queue and returns it. If it returns `nil`, that means the queue was empty. Note: if you're planning on using `dequeue` purely to delete items without regard to saving their values, you can add `@discardableResult` before the function declaration.
2. Check if `self` is the `.node` case and capture it's associated values. If it's not, then the queue is `.empty` and we return `nil`.
3. Before returning `value`, set `self` to be whatever `next` is. This might be another `.node`, or it might be `.empty` if we just took out the last item.
4. Return `value`, and we're done.


### Wonderful Wrappers

Yay! We've implemented a fully functional queue in Swift using a recursive enum. There's one caveat though. Did you catch it?

There’s nothing preventing consumers of this enum from creating their own instances with `Queue.empty` or `Queue.node(…)` and circumventing our `enqueue` and `dequeue` functions entirely. To avoid this we might want to make the two cases private like this:

{% highlight swift %}
enum Queue<Element> {
  private case empty
  private indirect case node(value: Element, next: Queue<Element>)
}
{% endhighlight swift %}

But this won't compile since you can’t make enum cases more private than the access level of the enum itself.

To solve this problem, we can make the enum fileprivate so that it can't be accessed outside the file, and in the same file we can wrap our enum in a `struct` like this:

{% highlight swift %}
struct QueueStruct<Element> {
  private var head = Queue<Element>.empty

  mutating func enqueue(_ value: Element) {
    head.enqueue(value)
  }

  @discardableResult mutating func dequeue() -> Element? {
    return head.dequeue()
  }
}
{% endhighlight swift %}

Personally I like naming the wrapping struct `Queue`, and then naming the underlying enum something else like `QueueNode`. It makes it cleaner to the public-facing API to  have the struct simply named `Queue`.

And there you have it! A Swift Queue built with a struct with an underlying recursive enum.

### Big-O Bonus Round

If you've been paying attention you'll have noticed that while our `dequeue` function operates in constant, O(1) time, sadly our `enqueue` function operates in linear, O(n) time. This is because of the recursive call that walks all the way down to the end of the queue to insert new values.

Can we improve `enqueue` to be constant time? To do that we would need a way to immediately access the end of the queue to append new values.

Of course, we could reverse our implementation: have our queue start at the end with each node pointing backwards. This would make `enqueue` O(1), but would make `dequeue` O(n), since we would need to traverse the whole queue to get to the front.

Can we have it both ways? Well, enums don't allow stored values. So if the enum itself refers to the front of the queue, we can't also store the back of the queue.

In theory we could use circular, doubly lined list. Each node points to a `next` and a `prev` and the first and last nodes point to each other:
```
head
 |
 v
"a" <--> "b" <--> "c" <-
 ^                      |
 |                      |
  ----------------------
```

Inserting would work like in any doubly linked list. For any node, its `prev` and `next` would point to itself:

```
prev.next == self
next.prev == self
```

If you want to insert `node` before `self`, you would need to re-wire four connections:

```
prev.next = node
node.prev = prev

node.next = self
self.prev = node
```

Unfortunately, while Swift enums *can* be recursive, they *can't* recursively end up containing themselves (as far as I can tell). On the very first enqueue, we have a problem. We would need to do something like this:

{% highlight swift %}
guard case .node(let value, var prev, let next) = self else {
  // self was the empty case,
  // so we set self to a new node,
  // and we make both its prev and next point to itself.
  self = .node(value: newValue, prev: self, next: self)
  return
}
{% endhighlight swift %}

This looks like it would work. But since Swift `enums` are value types, at the time when we do `(..., prev: self, next: self)`, the value `self` is actually still `.empty`. It isn't until after the assignment that `self` is a `.node`. So we end up with `self` being a single `.node` with a value, whose `prev` and `next` are both `.empty`, when what we really want if for `prev` and `next` to both be `self` recursively. This is something that could easily be accomplished with a `class` and pointers, but sadly is impossible with value types.

Another problem would be checking if we've deleted the last node when we `dequeue`. If that's the case we would want to be sure to set `self` to `.empty`. In order to check for this we would need to do a comparison to see if `prev`, `next`, and `self` are all the same (i.e. there is only one node pointing to itself):

{% highlight swift %}
if (prev == self && next == self) {
  self = .empty
}
{% endhighlight swift %}

The issue is that `==` is not defined for enums. We have to define it ourselves. And in our case there are two ways to do that:
1. Check the equality of the two values of the nodes, and recursively check the equality of `prev` and `next`. This means the equality check takes O(n) time, which in turn makes `dequeue` O(n).
2. Limit our queue to having `Set` behavior (unique values), and just check the equality of the two values. If they're the same, they must be the same node. This saves time on equality checks, but it means that on insertion we need to check the whole queue to make sure it doesn't already contain the new value. This brings `enqueue` back to O(n) time.

Additionally, implementing `==` means that we would need to constrain `Element` to conforming to `Compareable`, which we didn't need to do previously.

### Conclusion

We learned how to make a value-type `Queue` using a `struct` and an `enum`. In doing so we had to make either `enqueue` or `dequeue` work in linear instead of constant time. But we gained simplicity and value-type behavior. This makes our queue more Swifty, and makes it friendly with other value-type data structures like graphs, trees, and sorting algorithms.
