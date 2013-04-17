---
layout: post
title: "Flip your ImageViews"
date: 2013-04-16 20:06
comments: true
categories: Animations
description: "This tutorial will show you how to flip your ImageViews on Android."
keywords: "android, flip, imageview, drawable, view, image view, image, animation, flipimageview, image view animation, imageview animation"
---

This post will be cut in two parts. In the first part, I'll present my small android library [FlipImageView] which allows you to create easily an `ImageView` which can turn into another `Drawable`. Then I'll show you how the animation is performed and how you can modify it.

You can download a sample app [on the store][playstore] if you want to see what it looks like.

<!-- more -->

FlipImageView Library
---------------------

As I needed it, I recently created a very small lib called `FlipImageView` available on [Github][FlipImageView]. This lib is based on [FlipAnimator], made by Coomar, so all credits goes to him.

`FlipImageView extends ImageView`, that means if you have your own custom `ImageView`, I suggest you to extend `FlipImageView`, or look at the source code and take the animation part.

There are two ways to create your FlipImageView:

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

Here is the transformation part:

```java
@Override
protected void applyTransformation(float interpolatedTime, Transformation t) {
    // Angle around the y-axis of the rotation at the given time. It is
    // calculated both in radians and in the equivalent degrees.
    final double radians = Math.PI * interpolatedTime;
    float degrees = (float) (180.0 * radians / Math.PI);

    // Once we reach the midpoint in the animation, we need to hide the
    // source view and show the destination view. We also need to change
    // the angle by 180 degrees so that the destination does not come in
    // flipped around.
    if (interpolatedTime >= 0.5f) {
        degrees -= 180.f;

        if (!visibilitySwapped) {
            setImageDrawable(toDrawable);
            visibilitySwapped = true;
        }
    }

    if (mIsRotationReversed) {
        degrees = -degrees;
    }

    final Matrix matrix = t.getMatrix();

    camera.save();
    //We make a small translation in z axis, this is a cool effect :)
    //Note that you can custom this too, by making a translation in the 
    // other direction for example, to make the image move in the foreground
    camera.translate(0.0f, 0.0f, (float) (150.0 * Math.sin(radians)));
    
    //rotations
    camera.rotateX(mIsRotationXEnabled ? degrees : 0);
    camera.rotateY(mIsRotationYEnabled ? degrees : 0);
    camera.rotateZ(mIsRotationZEnabled ? degrees : 0);
    camera.getMatrix(matrix);
    camera.restore();

    matrix.preTranslate(-centerX, -centerY);
    matrix.postTranslate(centerX, centerY);
}
```

Well, I think the code is pretty clear (I added some comments too). Do not hesitate to custom the animation by trying some interpolators or modifying directly the `applyTransformation` method!

Conclusion
----------

That was a pretty short post to introduce my lib, but I'm sure some guys will be happy to discover that kind of small tricks which can make your app better. But however, please do not make all your controls animated, just when needed, or your app will be really annoying.

Do not hesitate to contribute to the [Lib][FlipImageView] or to this [article][Blog] by leaving comments or making pull requests. Thanks!


[Blog]: https://github.com/castorflex/castorflex-blog/blob/master/source/_posts/2013-04-16-flipping-your-imageview.markdown
[FlipImageView]: https://github.com/castorflex/FlipImageView
[FlipAnimator]: https://code.google.com/p/myandroidwidgets/source/browse/trunk/FlipAnimatorExample/src/com/beanie/examples/animation/FlipAnimator/FlipAnimator.java
[playstore]: https://play.google.com/store/apps/details?id=fr.castorflex.android.flipimageview