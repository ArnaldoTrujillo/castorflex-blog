---
layout: post
title: "Flipping your ImageView"
date: 2013-04-16 20:06
comments: true
categories: Animations
description: "This tutorial will show you how to flip your ImageViews on Android."
keywords: "android, flip, imageview, drawable, view, image view, image, animation, flipimageview, image view animation, imageview animation"
---

This post will be cut in two parts. In the first part, I'll present my small android library [FlipImageView] which allows you to create easily an `ImageView` which can flip into another `Drawable`. Then I'll show you how the animation is performed and how you can modify it.

You can download a sample app [on the store][playstore] if you want to see what it looks like.

<!-- more -->

FlipImageView Library
---------------------

As I needed it, I created a few time ago a very small lib called `FlipImageView` available on [Github][FlipImageView]. This lib is based on [FlipAnimator], made by Coomar, so all credits goes to him.

`FlipImageView extends ImageView`, so if you have your own custom `ImageView`, I suggest you to extend `FlipImageView`, or look at the source code to take the animation part.

There is two ways to create your FlipImageView:

-	Via XML

```xml
	<fr.castorflex.android.flipimageview.library.FlipImageView
       xmlns:fiv="http://schemas.android.com/apk/res-auto"
       android:id="@+id/imageview"
       android:layout_width="match_parent"
       android:layout_height="match_parent"
       android:src="@drawable/YOUR_DEFAULT_DRAWABLE"
       fiv:flipDrawable="@drawable/YOUR_FLIPPED_DRAWABLE"
       fiv:flipDuration="YOUR_DURATION_IN_MS"
       fiv:flipInterpolator="@android:anim/YOUR_INTERPOLATOR"
       fiv:flipRotations="none|x|y|z"
       fiv:isAnimated="true|false"
       fiv:isFlipped="true|false"
       fiv:reverseRotation="true|false"/>
```	

-	Or programmatically

```java
//Standard way
FlipImageView fiv = new FlipImageView(mContext);
fiv.setDrawable(mDrawable);	//Note that this is different from setImageDrawable(...)
fiv.setFlippedDrawable(mDrawable);
fiv.setIsFlipped(false);
//etc.
```

***Note:*** *setDrawable is different from setImageDrawable which one will set directly the drawable to the ImageView.*

You can disable the flip animation by calling `setAnimated(false)`.

The flip animation will be launched each time you click on the image. You can of course call it programmatically, with `toggleFlip()` method or `setFlipped(boolean flipped)` method. You can also Override onClick or set another `View.OnClickListener` if you do not want the image view to flip when clicked.

You can set a `OnFlipListener` to your FlipImageView. Three events will be triggered:

1.	`onClick(boolean flipped)`: called when the imageView is clicked
2.	`onFlipStart()` : called at the beginning of the flip animation
3.	`onFlipEnd()`: called at the end of the animation

***Note:*** *onFlipStart and OnFlipEnd won't be called if you disabled the flip animation.*

You can also reverse the animation, for example if you want your ImageView to flip in the other direction, to make a "flip back" transition. To do that, just call `setRotationReversed(true)` every time you flip your image.

You can enable/disable x, y and z rotation axis via xml or programmatically too.

I strongly recommend you to play with existing `interpolators` (bounce/overshoot are funny :p) or to create new ones. You could be surprised by the effect :p


How does this work?
-------------------




[FlipImageView]: https://github.com/castorflex/FlipImageView
[FlipAnimator]: https://code.google.com/p/myandroidwidgets/source/browse/trunk/FlipAnimatorExample/src/com/beanie/examples/animation/FlipAnimator/FlipAnimator.java
[playstore]: https://play.google.com/store/apps/details?id=fr.castorflex.android.flipimageview