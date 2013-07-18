---
layout: post
title: "Adding a Foreground Selector to a View/ViewGroup"
date: 2013-07-17 23:34
comments: true
keywords: "android, selector, foreground, background, card, cards, android add foreground selector, touch feedback, google play"
description: "You will see in this post how to implement a foreground selector for either a ViewGroup or a basic View in Android. This effect is often used in Android cards (e.g. in the play store) and is a great touch feedback."
categories: [UI/UX]
---

Everyone has already seen the android cards touch feedback. The selector is drawn in the foreground instead of in the background (as we usually implement it). This effect is in fact pretty simple to implement, and is already implemented in some cases.

Here is a screenshot of the press state effect in the Google Play app:

{% img center /images/screenshot_googleplay.png %}


<!-- more -->

I.	If your view is a `FrameLayout`
================================

This is the easiest way to add a foreground selector because there is a [method][setforeground method] for that! Indeed, you just have to pass your selector as `android:foreground` in your xml or programmatically, calling `setForeground(Drawable)`.

II. If your view is not a `FrameLayout`
====================================

Don't worry, this is pretty simple. Basically, we just have to set right state to the selector (pressed, focused, etc.), set the bounds and draw it after the view itself. In that way, the selector will be drawn after, and as a consequence, over the view. 

###Changing the state

In the `View` class, a method is called each time the state of the view changes. This method is `drawableStateChanged()` ([DOC HERE][drawablestatechanged method]). 

```java
@Override
protected void drawableStateChanged() {
	super.drawableStateChanged();

	mForegroundSelector.setState(getDrawableState());

	//redraw
	invalidate();
}
```

###Updating the drawable bounds

A method is called each time the size of the view changes. This method is `onSizeChanged(int, int, int, int)` ([DOC HERE][onsizechanged method]).

```java
@Override
protected void onSizeChanged(int width, int height, int oldwidth, int oldheight) {
	super.onSizeChanged(width, height, oldwidth, oldheight);

	mForegroundSelector.setBounds(0, 0, width, height);
}
```

###Drawing the selector

There is 2 cases:

####1.	Your view is a **not** a `ViewGroup` 

The selector has to be drawn after calling `onDraw(Canvas canvas)`

```java
@Override
protected void onDraw(Canvas canvas) {
	super.onDraw(canvas);

	mForegroundSelector.draw(canvas);
}
```

####2.	Your view is a `ViewGroup`


The selector has to be drawn after all his children, that means after calling `dispatchDraw(Canvas canvas)`

```java
@Override
protected void dispatchDraw(Canvas canvas) {
	super.dispatchDraw(canvas);

	mForegroundSelector.draw(canvas);
}
```

Conclusion
==========

I made here a very basic example of how to add a foreground selector to a custom view, with the main methods. To see a complete implementation (Callback, paddings, etc.), you can look at the source code of FrameLayout.



[onsizechanged method]: http://developer.android.com/reference/android/view/View.html#onSizeChanged(int, int, int, int)
[drawablestatechanged method]: http://developer.android.com/reference/android/view/View.html#drawableStateChanged()
[setforeground method]: http://developer.android.com/reference/android/widget/FrameLayout.html#setForeground(android.graphics.drawable.Drawable)
