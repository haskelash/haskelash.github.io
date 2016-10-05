---
layout: post
title: "Detecting Changes in a UIView's Frame or Bounds"
date: 2016-09-30 16:06:17
image: '/assets/img/'
description:
tags:
- swift
- uikit
categories:
twitter_text:
---

Often you need to detect when a custom view's `frame` or `bounds` is about to change or just did change. To do this, you can override the `frame` or `bounds` variables. Then, in curly braces, add two methods called `willSet` and `didSet`. Inside these methods you can access arguements called `newValue` and `oldValue`, respectively.  You can also access the current value of `frame` or `bounds` from within both of these methods via their normal variable names.

{% highlight swift %}
class SomeSubview: UIView {
  override var frame: CGRect {
    willSet {
      //use 'frame' and 'newValue' to access the current and soon-to-be-current frames
      print("\(frame) is about to be replaced by \(newValue)")
      //do stuff here
    }
    didSet {
      //use 'frame' and 'oldValue' to access the current and just-was-current frames
      print("\(frame) just replaced \(oldValue)")
      //do stuff here
    }
  }

  override var bounds: CGRect {
      willSet {
      //use 'bounds' and 'newValue' to access the current and soon-to-be-current bounds
      print("\(bounds) is about to be replaced by \(newValue)")
      //do stuff here
    }
    didSet {
      //use 'bounds' and 'oldValue' to access the current and just-was-current bounds
      print("\(bounds) just replaced \(oldValue)")
      //do stuff here
    }
  }
}
{% endhighlight %}
