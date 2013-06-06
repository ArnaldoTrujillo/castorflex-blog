---
layout: post
title: "Making a bouncing animation for your sliding menu"
date: 2013-04-12 13:35
comments: true
categories: Animations
description: "This article will show you a way to introduce your sliding
menu with a bouncing animation"
keywords: "sliding menu, android, bounce, bouncing, bouncing animation,
fly-in-app menu, fly in app menu, animation, scroller, custom scroller"
---


The Sliding Menu, aka Fly in App Menu, appeared about one year ago (`@see` [Prixing], [Facebook], etc.). Because of the ease of use and the effectiveness, the users quickly adopted this UI pattern. You can read more about this pattern [here][sliding menu pattern].

One of the cons of this pattern is the fact that it's hidden, so you need to inform the user he can open it. There are many ways to do this, such as guides, but there is a convenient way to do this, which is a bouncing animation when the application is first launched. That's what we are going to talk.

<!-- more -->

The bouncing animation is used by Prixing, and I think this is a very good way to make the user discover the menu, because it does not require any popups, tutorials, or whatever that can be so annoying for the user. If you do not know what I am talking about, I strongly suggest you to download Prixing and look at the first launch.

I made a sample apk showing this effect. This should be available [here](apk).

Interpolators
-------------
From the [DOC][Interpolator Doc]:

> An interpolator defines the rate of change of an animation. This allows the basic animation effects (alpha, scale, translate, rotate) to be accelerated, decelerated, repeated, etc.

But how?

An `Interpolator` is a function defined on [0,1], returning a value, generally between 0 and 1. The animation will use an `Interpolator` to interpolate as many values as the number of the frames needed during the animation. For example, if you have a 2 sec. animation, and a 30 fps screen, you will need 60 values.

There is some native interpolators:

{% img center /images/interpolators.png %}

The problem here with the sdk interpolators, is that they create values from 0 (for t=0) to x (for t=1). As the sliding menu needs to be closed at the end of the animation, we need to create a custom bounce interpolator. Here is the function:

$$
\begin{align}
\mbox f(t) = abs(sin(pi * (t+1)²) * (1-t) \\
\end{align}
$$

{% img center /images/interpolators2.png %}

Bouncing Animation with Sliding Menu
----------------------------------
> TL;DR; All you need is a [GIST]

This part will talk about the implementation of the bouncing animation on the android lib `SlidingMenu` (credits to Jeremy Feinstein) available [on Github][SlidingMenu]. 

This lib uses a `Scroller` to make the menu slide. The problem with this class is that it requires a final position, used for the interpolation AND for the end of the animation:

```java
//in computeScrollOffset method 	
if (timePassed < mDuration) {
	//...
}else{
	//end of the animation
	mCurrX = mFinalX;
    mCurrY = mFinalY;
    mFinished = true;
}
```

We do not want to set the final position at the end of the animation, but 0. The easier way to do this (I think) is to copy paste the Scroller class and add a way to set the final position:

```java
/**
* New startScroll method, with a finalX and finalY
*/
public void startScroll(int startX, int startY, int dx, int dy, int duration, int finalX, int finalY) {
	mMode = SCROLL_MODE;
    mFinished = false;
    mDuration = duration;
    mStartTime = AnimationUtils.currentAnimationTimeMillis();
    mStartX = startX;
    mStartY = startY;
    mThresholdX = startX + dx;
    mThresholdY = startY + dy;
    mDeltaX = dx;
    mDeltaY = dy;
    mDurationReciprocal = 1.0f / (float) mDuration;
    mFinalX = finalX;
    mFinalY = finalY;
}


/**
* Old startScroll method
*/    
public void startScroll(int startX, int startY, int dx, int dy, int duration) {
	startScroll(startX, startY, dx, dy, duration, startX + dx, startY + dy);
}

```

We also have to set the appropriate value when the animation is done, and remove a the following test:

```java
if (mCurrX == mThresholdX && mCurrY == mThresholdY) {
    mFinished = true;
}
```

Indeed, as we are waiting for the end of the animation, we don't want to stop if we reach the threshold.

Then modify the Sliding Menu lib to use the new scroller:

`SlidingMenu.java`

```java
public void scrollWithBounceInterpolator(){
	mViewAbove.scrollWithBounceInterpolator();
}
```

`CustomViewAbove.java`

```java
public void scrollWithBounceInterpolator() {
	int x = mViewBehind.getBehindWidth(); //or whatever you want :D
	int y = 0;
	if (getChildCount() == 0) {
		// Nothing to do.
		setScrollingCacheEnabled(false);
		return;
	}
	int sx = getScrollX();
	int sy = getScrollY();
	int dx = x - sx;
	int dy = y - sy;
	mBounceScroller.startScroll(sx, sy, dx, dy, 1000);
	invalidate();
}
```

add the following on the `computeScroll` method:

```java
if (!mBounceScroller.isFinished()) {
	if (mBounceScroller.computeScrollOffset()) {
		int x = mBounceScroller.getCurrX();
		int y = mBounceScroller.getCurrY();

		scrollTo(x, y);
		// Keep on drawing until the animation has finished.
		invalidate();
		return;
	}
}
```


You should now be able to call `mSlidingMenu.scrollWithBounceInterpolator()` from your activity!

Conclusion
----------

The sources of my posts are on [Github], so do not hesitate to contribute by making pull requests, etc.

TL;DR; I made a gist with all that you need! [GIST]

[apk]: https://play.google.com/store/apps/details?id=fr.castorflex.android.BounceSlidingMenuProject
[GIST]:https://gist.github.com/castorflex/5337238
[Interpolator Doc]: http://developer.android.com/reference/android/view/animation/Interpolator.html
[Github]: https://github.com/castorflex/castorflex-blog/blob/master/source/_posts/2013-04-12-making-a-bounce-animation-for-your-sliding-menu.markdown
[SlidingMenu]: https://github.com/jfeinstein10/SlidingMenu
[Prixing]: https://play.google.com/store/apps/details?id=fr.epicdream.beamy
[Facebook]: https://play.google.com/store/apps/details?id=com.facebook.katana
[sliding menu pattern]: http://www.androiduipatterns.com/2012/06/emerging-ui-pattern-side-navigation.html
[Scroller]: http://developer.android.com/reference/android/widget/Scroller.html
