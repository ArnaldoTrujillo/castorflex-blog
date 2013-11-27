---
layout: post
title: "Make your ProgressBar smoother"
date: 2013-11-12 21:00
comments: true
categories: UI
description: "We will see in the post a way to reproduce the progress bar in the gmail app. This progress bar use a custom indeterminateDrawable to make it smoother!"
keywords: "android, smooth progress bar, progressbar, smooth, smoother, gmail, indeterminate drawable, custom, indeterminateDrawable"
---

If you use the android Gmail application, you probably noticed that the progress bar is a bit customized.	I am not talking about the pull to resfresh but the indeterminate progress bar which appears just after. This indeterminate drawable is much smoother than the usual.

I will show you in this post a way to reproduce this smooth indeterminate horizontal progress bar. Here is the result: the first progress bar uses the default indeterminate drawable while the others are all custom.

{% youtube qt4lvQmY0F0 %}

*There is apparently a problem with the size of the video, so you might want to click [on this link][Youtube] to see it on Youtube.*

<!-- more -->

#How does the default animation work

First of all, you will need to use the Horizontal progress bar style: `Widget.ProgressBar.Horizontal` for pre ICS and `Widget.Holo.ProgressBar.Horizontal` for post ICS devices. This gives us two important parameters: `indeterminateOnly=false` and `indeterminateDrawable`.

The default indeterminate drawable looks like this:

```xml
<animation-list xmlns:android="http://schemas.android.com/apk/res/android" android:oneshot="false">
    <item android:drawable="@drawable/progressbar_indeterminate_holo1" android:duration="50" />
    <item android:drawable="@drawable/progressbar_indeterminate_holo2" android:duration="50" />
    <item android:drawable="@drawable/progressbar_indeterminate_holo3" android:duration="50" />
    <item android:drawable="@drawable/progressbar_indeterminate_holo4" android:duration="50" />
    <item android:drawable="@drawable/progressbar_indeterminate_holo5" android:duration="50" />
    <item android:drawable="@drawable/progressbar_indeterminate_holo6" android:duration="50" />
    <item android:drawable="@drawable/progressbar_indeterminate_holo7" android:duration="50" />
    <item android:drawable="@drawable/progressbar_indeterminate_holo8" android:duration="50" />
</animation-list>
```

It's a simple animation drawable. That's why we cannot have the smoothness we want. We just have to make a custom indeterminate drawable which dynamically draws lines instead of drawing bitmaps.

#The custom indeterminate drawable

Here is my original idea (don't be impressed by my graphic skills):

{% img center /images/smoothprogressbar_exponential.png %}

Basically we have some kind of exponential function we use to compute the length of each part of the drawable. In fact, the code is even simpler than that. We just have to loop and draw lines bigger than the previous one. You also have to take the offset in account, as this value will let you to have an animation.

***Note**: My math skills are a bit poor, so I might be saying some stupid stuff, don't hesitate to comment :)*

Here is the code I used.

```java
int i = 0;
// we loop to draw lines until we reach the max width
while (prev < width) {
	float value = (float) Math.expm1(++i + offset);
    canvas.drawLine(prev, centerY, prev + value - mSeparatorWidth, centerY, mPaint);
    prev = value + prev;
}
```

I used the expm1 function (which returns exp(x) - 1) but you can try any growing function. We could imagine a function which multiply by two the previous part each time, or some kind of fibonacci sequence.

Here is the result for the code above:

{% img center /images/smoothprogressbar.gif %}

I could stop here but I wanted to go further and give to our drawable the possibility to be fully customizable.

#Use interpolators!

I've always loved the relation between these curves and animations. It allows developers to radically change their animation in just one line. You can make it bounce, overshoot,... just by changing the interpolator. I used interpolators here to change the way the animation looks like.

A chart is better than words:

{% img center /images/smoothprogressbar_interpolator.png %}

Well. Seeing this, code would be better than my chart. Sorry about that.

Basically, we set a number of sections. The interpolator will give us the ratio of each section's length. The animation is done by an offset, incremented at each frame.

```java
int width = mBounds.width() + mSeparatorWidth;
int centerY = mBounds.centerY();
float xSectionWidth = 1f / mSectionsCount;

//line before the first section
int offset = (int) (Math.abs(mInterpolator.getInterpolation(mCurrentOffset) - mInterpolator.getInterpolation(0)) * width);
if (offset > 0) {
    if (offset > mSeparatorWidth) {
        canvas.drawLine(0, centerY, offset - mSeparatorWidth, centerY, mPaint);
    }
}

int prev;
int end;
int spaceLength;
for (int i = 0; i < mSectionsCount; ++i) {
    float xOffset = xSectionWidth * i + mCurrentOffset;
    prev = (int) (mInterpolator.getInterpolation(xOffset) * width);
    float ratioSectionWidth =
            Math.abs(mInterpolator.getInterpolation(xOffset) -
                    mInterpolator.getInterpolation(Math.min(xOffset + xSectionWidth, 1f)));
    
    //separator between each piece of line                
    int sectionWidth = (int) (width * ratioSectionWidth);
    if (sectionWidth + prev < width)
        spaceLength = Math.min(sectionWidth, mSeparatorWidth);
    else 
    	spaceLength = 0;
    int drawLength = sectionWidth > spaceLength ? sectionWidth - spaceLength : 0;
    end = prev + drawLength;
    
    if (end > prev) {
        canvas.drawLine(prev, centerY, end, centerY, mPaint);
    }
}
```

Here is the result with some basic interpolators:

{% img center /images/smoothprogressbar_all.gif %}

Sorry about the lag in the gif, a sample apk will be available soon, so stay tuned ;)

**Limitations**: Your interpolator must be a monotonic function!




#Library

I made a small lib available [on Github][SmoothProgressBarLib]! do not hesitate to fork it and make pull requests!




[SmoothProgressBarLib]: https://github.com/castorflex/SmoothProgressBar
[Youtube]: http://www.youtube.com/watch?v=qt4lvQmY0F0

