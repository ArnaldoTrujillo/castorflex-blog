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

We do not want to set the final position at the end of the animation, but 0. The easier way to do this (I think) is to rewrite the scroller class like this:

```java
import android.content.Context;
import android.hardware.SensorManager;
import android.os.Build;
import android.util.FloatMath;
import android.view.ViewConfiguration;
import android.view.animation.AnimationUtils;
import android.view.animation.Interpolator;

/**
 * Created with IntelliJ IDEA, best IDE in the world. User: Castorflex Date: 08/04/13 Time: 16:21
 */
public class BounceScroller {

	private static final Interpolator sBounceInterpolator = new Interpolator() {
		@Override
		public float getInterpolation(float t) {
			return -(float) Math.abs(Math.sin((float) Math.PI * (t + 1) * (t + 1)) * (1 - t));
		}
	};

	private int mMode;

	private int mStartX;
	private int mStartY;
	private int mFinalX;
	private int mFinalY;

	private int mMinX;
	private int mMaxX;
	private int mMinY;
	private int mMaxY;

	private int          mCurrX;
	private int          mCurrY;
	private long         mStartTime;
	private int          mDuration;
	private float        mDurationReciprocal;
	private float        mDeltaX;
	private float        mDeltaY;
	private boolean      mFinished;
	private Interpolator mInterpolator;
	private boolean      mFlywheel;

	private float mVelocity;
	private float mCurrVelocity;
	private int   mDistance;

	private float mFlingFriction = ViewConfiguration.getScrollFriction();

	private static final int DEFAULT_DURATION = 250;
	private static final int SCROLL_MODE      = 0;
	private static final int FLING_MODE       = 1;

	private static       float DECELERATION_RATE = (float) (Math.log(0.78) / Math.log(0.9));
	private static final float INFLEXION         = 0.35f; // Tension lines cross at (INFLEXION, 1)
	private static final float START_TENSION     = 0.5f;
	private static final float END_TENSION       = 1.0f;
	private static final float P1                = START_TENSION * INFLEXION;
	private static final float P2                = 1.0f - END_TENSION * (1.0f - INFLEXION);

	private static final int     NB_SAMPLES      = 100;
	private static final float[] SPLINE_POSITION = new float[NB_SAMPLES + 1];
	private static final float[] SPLINE_TIME     = new float[NB_SAMPLES + 1];

	private       float mDeceleration;
	private final float mPpi;

	// A context-specific coefficient adjusted to physical values.
	private float mPhysicalCoeff;

	static {
		float x_min = 0.0f;
		float y_min = 0.0f;
		for (int i = 0; i < NB_SAMPLES; i++) {
			final float alpha = (float) i / NB_SAMPLES;

			float x_max = 1.0f;
			float x, tx, coef;
			while (true) {
				x = x_min + (x_max - x_min) / 2.0f;
				coef = 3.0f * x * (1.0f - x);
				tx = coef * ((1.0f - x) * P1 + x * P2) + x * x * x;
				if (Math.abs(tx - alpha) < 1E-5) {
					break;
				}
				if (tx > alpha) {
					x_max = x;
				} else {
					x_min = x;
				}
			}
			SPLINE_POSITION[i] = coef * ((1.0f - x) * START_TENSION + x) + x * x * x;

			float y_max = 1.0f;
			float y, dy;
			while (true) {
				y = y_min + (y_max - y_min) / 2.0f;
				coef = 3.0f * y * (1.0f - y);
				dy = coef * ((1.0f - y) * START_TENSION + y) + y * y * y;
				if (Math.abs(dy - alpha) < 1E-5) {
					break;
				}
				if (dy > alpha) {
					y_max = y;
				} else {
					y_min = y;
				}
			}
			SPLINE_TIME[i] = coef * ((1.0f - y) * P1 + y * P2) + y * y * y;
		}
		SPLINE_POSITION[NB_SAMPLES] = SPLINE_TIME[NB_SAMPLES] = 1.0f;

		// This controls the viscous fluid effect (how much of it)
		sViscousFluidScale = 8.0f;
		// must be set to 1.0 (used in viscousFluid())
		sViscousFluidNormalize = 1.0f;
		sViscousFluidNormalize = 1.0f / viscousFluid(1.0f);

	}

	private static float sViscousFluidScale;
	private static float sViscousFluidNormalize;

	/**
	 * Create a Scroller with the specified interpolator. If the interpolator is null, the default
	 * (viscous) interpolator will be used. "Flywheel" behavior will be in effect for apps targeting
	 * Honeycomb or newer.
	 */
	public BounceScroller(Context context) {
		this(context, sBounceInterpolator,
				context.getApplicationInfo().targetSdkVersion >= Build.VERSION_CODES.HONEYCOMB);
	}

	/**
	 * Create a Scroller with the specified interpolator. If the interpolator is null, the default
	 * (viscous) interpolator will be used. Specify whether or not to support progressive "flywheel"
	 * behavior in flinging.
	 */
	public BounceScroller(Context context, Interpolator interpolator, boolean flywheel) {
		mFinished = true;
		mInterpolator = interpolator;
		mPpi = context.getResources().getDisplayMetrics().density * 160.0f;
		mDeceleration = computeDeceleration(ViewConfiguration.getScrollFriction());
		mFlywheel = flywheel;

		mPhysicalCoeff = computeDeceleration(0.84f); // look and feel tuning
	}

	/**
	 * The amount of friction applied to flings. The default value is {@link
	 * ViewConfiguration#getScrollFriction}.
	 *
	 * @param friction A scalar dimension-less value representing the coefficient of friction.
	 */
	public final void setFriction(float friction) {
		mDeceleration = computeDeceleration(friction);
		mFlingFriction = friction;
	}

	private float computeDeceleration(float friction) {
		return SensorManager.GRAVITY_EARTH   // g (m/s^2)
				* 39.37f               // inch/meter
				* mPpi                 // pixels per inch
				* friction;
	}

	/**
	 * Returns whether the scroller has finished scrolling.
	 *
	 * @return True if the scroller has finished scrolling, false otherwise.
	 */
	public final boolean isFinished() {
		return mFinished;
	}

	/**
	 * Force the finished field to a particular value.
	 *
	 * @param finished The new finished value.
	 */
	public final void forceFinished(boolean finished) {
		mFinished = finished;
	}

	/**
	 * Returns how long the scroll event will take, in milliseconds.
	 *
	 * @return The duration of the scroll in milliseconds.
	 */
	public final int getDuration() {
		return mDuration;
	}

	/**
	 * Returns the current X offset in the scroll.
	 *
	 * @return The new X offset as an absolute distance from the origin.
	 */
	public final int getCurrX() {
		return mCurrX;
	}

	/**
	 * Returns the current Y offset in the scroll.
	 *
	 * @return The new Y offset as an absolute distance from the origin.
	 */
	public final int getCurrY() {
		return mCurrY;
	}

	/**
	 * Returns the current velocity.
	 *
	 * @return The original velocity less the deceleration. Result may be negative.
	 */
	public float getCurrVelocity() {
		return mMode == FLING_MODE ?
				mCurrVelocity : mVelocity - mDeceleration * timePassed() / 2000.0f;
	}

	/**
	 * Returns the start X offset in the scroll.
	 *
	 * @return The start X offset as an absolute distance from the origin.
	 */
	public final int getStartX() {
		return mStartX;
	}

	/**
	 * Returns the start Y offset in the scroll.
	 *
	 * @return The start Y offset as an absolute distance from the origin.
	 */
	public final int getStartY() {
		return mStartY;
	}

	/**
	 * Returns where the scroll will end. Valid only for "fling" scrolls.
	 *
	 * @return The final X offset as an absolute distance from the origin.
	 */
	public final int getFinalX() {
		return mFinalX;
	}

	/**
	 * Returns where the scroll will end. Valid only for "fling" scrolls.
	 *
	 * @return The final Y offset as an absolute distance from the origin.
	 */
	public final int getFinalY() {
		return mFinalY;
	}

	/**
	 * Call this when you want to know the new location.  If it returns true, the animation is not yet
	 * finished.  loc will be altered to provide the new location.
	 */
	public boolean computeScrollOffset() {
		if (mFinished) {
			return false;
		}

		int timePassed = (int) (AnimationUtils.currentAnimationTimeMillis() - mStartTime);

		if (timePassed < mDuration) {
			switch (mMode) {
				case SCROLL_MODE:
					float x = timePassed * mDurationReciprocal;

					if (mInterpolator == null) {
						x = viscousFluid(x);
					} else {
						x = mInterpolator.getInterpolation(x);
					}

					mCurrX = mStartX + Math.round(x * mDeltaX);
					mCurrY = mStartY + Math.round(x * mDeltaY);
					break;
				case FLING_MODE:
					final float t = (float) timePassed / mDuration;
					final int index = (int) (NB_SAMPLES * t);
					float distanceCoef = 1.f;
					float velocityCoef = 0.f;
					if (index < NB_SAMPLES) {
						final float t_inf = (float) index / NB_SAMPLES;
						final float t_sup = (float) (index + 1) / NB_SAMPLES;
						final float d_inf = SPLINE_POSITION[index];
						final float d_sup = SPLINE_POSITION[index + 1];
						velocityCoef = (d_sup - d_inf) / (t_sup - t_inf);
						distanceCoef = d_inf + (t - t_inf) * velocityCoef;
					}

					mCurrVelocity = velocityCoef * mDistance / mDuration * 1000.0f;

					mCurrX = mStartX + Math.round(distanceCoef * (mFinalX - mStartX));
					// Pin to mMinX <= mCurrX <= mMaxX
					mCurrX = Math.min(mCurrX, mMaxX);
					mCurrX = Math.max(mCurrX, mMinX);

					mCurrY = mStartY + Math.round(distanceCoef * (mFinalY - mStartY));
					// Pin to mMinY <= mCurrY <= mMaxY
					mCurrY = Math.min(mCurrY, mMaxY);
					mCurrY = Math.max(mCurrY, mMinY);

					if (mCurrX == mFinalX && mCurrY == mFinalY) {
						mFinished = true;
					}

					break;
			}
		} else {
			mCurrX = 0;
			mCurrY = 0;
			mFinished = true;
		}
		return true;
	}

	/**
	 * Start scrolling by providing a starting point and the distance to travel. The scroll will use
	 * the default value of 250 milliseconds for the duration.
	 *
	 * @param startX Starting horizontal scroll offset in pixels. Positive numbers will scroll the
	 *               content to the left.
	 * @param startY Starting vertical scroll offset in pixels. Positive numbers will scroll the
	 *               content up.
	 * @param dx     Horizontal distance to travel. Positive numbers will scroll the content to the
	 *               left.
	 * @param dy     Vertical distance to travel. Positive numbers will scroll the content up.
	 */
	public void startScroll(int startX, int startY, int dx, int dy) {
		startScroll(startX, startY, dx, dy, DEFAULT_DURATION);
	}

	/**
	 * Start scrolling by providing a starting point and the distance to travel.
	 *
	 * @param startX   Starting horizontal scroll offset in pixels. Positive numbers will scroll the
	 *                 content to the left.
	 * @param startY   Starting vertical scroll offset in pixels. Positive numbers will scroll the
	 *                 content up.
	 * @param dx       Horizontal distance to travel. Positive numbers will scroll the content to the
	 *                 left.
	 * @param dy       Vertical distance to travel. Positive numbers will scroll the content up.
	 * @param duration Duration of the scroll in milliseconds.
	 */
	public void startScroll(int startX, int startY, int dx, int dy, int duration) {
		mMode = SCROLL_MODE;
		mFinished = false;
		mDuration = duration;
		mStartTime = AnimationUtils.currentAnimationTimeMillis();
		mStartX = startX;
		mStartY = startY;
		mFinalX = startX + dx;
		mFinalY = startY + dy;
		mDeltaX = dx;
		mDeltaY = dy;
		mDurationReciprocal = 1.0f / (float) mDuration;
	}

	/**
	 * Start scrolling based on a fling gesture. The distance travelled will depend on the initial
	 * velocity of the fling.
	 *
	 * @param startX    Starting point of the scroll (X)
	 * @param startY    Starting point of the scroll (Y)
	 * @param velocityX Initial velocity of the fling (X) measured in pixels per second.
	 * @param velocityY Initial velocity of the fling (Y) measured in pixels per second
	 * @param minX      Minimum X value. The scroller will not scroll past this point.
	 * @param maxX      Maximum X value. The scroller will not scroll past this point.
	 * @param minY      Minimum Y value. The scroller will not scroll past this point.
	 * @param maxY      Maximum Y value. The scroller will not scroll past this point.
	 */
	public void fling(int startX, int startY, int velocityX, int velocityY,
			int minX, int maxX, int minY, int maxY) {
		// Continue a scroll or fling in progress
		if (mFlywheel && !mFinished) {
			float oldVel = getCurrVelocity();

			float dx = (float) (mFinalX - mStartX);
			float dy = (float) (mFinalY - mStartY);
			float hyp = FloatMath.sqrt(dx * dx + dy * dy);

			float ndx = dx / hyp;
			float ndy = dy / hyp;

			float oldVelocityX = ndx * oldVel;
			float oldVelocityY = ndy * oldVel;
			if (Math.signum(velocityX) == Math.signum(oldVelocityX) &&
					Math.signum(velocityY) == Math.signum(oldVelocityY)) {
				velocityX += oldVelocityX;
				velocityY += oldVelocityY;
			}
		}

		mMode = FLING_MODE;
		mFinished = false;

		float velocity = FloatMath.sqrt(velocityX * velocityX + velocityY * velocityY);

		mVelocity = velocity;
		mDuration = getSplineFlingDuration(velocity);
		mStartTime = AnimationUtils.currentAnimationTimeMillis();
		mStartX = startX;
		mStartY = startY;

		float coeffX = velocity == 0 ? 1.0f : velocityX / velocity;
		float coeffY = velocity == 0 ? 1.0f : velocityY / velocity;

		double totalDistance = getSplineFlingDistance(velocity);
		mDistance = (int) (totalDistance * Math.signum(velocity));

		mMinX = minX;
		mMaxX = maxX;
		mMinY = minY;
		mMaxY = maxY;

		mFinalX = startX + (int) Math.round(totalDistance * coeffX);
		// Pin to mMinX <= mFinalX <= mMaxX
		mFinalX = Math.min(mFinalX, mMaxX);
		mFinalX = Math.max(mFinalX, mMinX);

		mFinalY = startY + (int) Math.round(totalDistance * coeffY);
		// Pin to mMinY <= mFinalY <= mMaxY
		mFinalY = Math.min(mFinalY, mMaxY);
		mFinalY = Math.max(mFinalY, mMinY);
	}

	private double getSplineDeceleration(float velocity) {
		return Math.log(INFLEXION * Math.abs(velocity) / (mFlingFriction * mPhysicalCoeff));
	}

	private int getSplineFlingDuration(float velocity) {
		final double l = getSplineDeceleration(velocity);
		final double decelMinusOne = DECELERATION_RATE - 1.0;
		return (int) (1000.0 * Math.exp(l / decelMinusOne));
	}

	private double getSplineFlingDistance(float velocity) {
		final double l = getSplineDeceleration(velocity);
		final double decelMinusOne = DECELERATION_RATE - 1.0;
		return mFlingFriction * mPhysicalCoeff * Math.exp(DECELERATION_RATE / decelMinusOne * l);
	}

	static float viscousFluid(float x) {
		x *= sViscousFluidScale;
		if (x < 1.0f) {
			x -= (1.0f - (float) Math.exp(-x));
		} else {
			float start = 0.36787944117f;   // 1/e == exp(-1)
			x = 1.0f - (float) Math.exp(1.0f - x);
			x = start + x * (1.0f - start);
		}
		x *= sViscousFluidNormalize;
		return x;
	}

	/**
	 * Stops the animation. Contrary to {@link #forceFinished(boolean)}, aborting the animating cause
	 * the scroller to move to the final x and y position
	 *
	 * @see #forceFinished(boolean)
	 */
	public void abortAnimation() {
		mCurrX = 0;
		mCurrY = 0;
		mFinished = true;
	}

	/**
	 * Extend the scroll animation. This allows a running animation to scroll further and longer, when
	 * used with {@link #setFinalX(int)} or {@link #setFinalY(int)}.
	 *
	 * @param extend Additional time to scroll in milliseconds.
	 * @see #setFinalX(int)
	 * @see #setFinalY(int)
	 */
	public void extendDuration(int extend) {
		int passed = timePassed();
		mDuration = passed + extend;
		mDurationReciprocal = 1.0f / mDuration;
		mFinished = false;
	}

	/**
	 * Returns the time elapsed since the beginning of the scrolling.
	 *
	 * @return The elapsed time in milliseconds.
	 */
	public int timePassed() {
		return (int) (AnimationUtils.currentAnimationTimeMillis() - mStartTime);
	}

	/**
	 * Sets the final position (X) for this scroller.
	 *
	 * @param newX The new X offset as an absolute distance from the origin.
	 * @see #extendDuration(int)
	 * @see #setFinalY(int)
	 */
	public void setFinalX(int newX) {
		mFinalX = newX;
		mDeltaX = mFinalX - mStartX;
		mFinished = false;
	}

	/**
	 * Sets the final position (Y) for this scroller.
	 *
	 * @param newY The new Y offset as an absolute distance from the origin.
	 * @see #extendDuration(int)
	 * @see #setFinalX(int)
	 */
	public void setFinalY(int newY) {
		mFinalY = newY;
		mDeltaY = mFinalY - mStartY;
		mFinished = false;
	}

	/**
	 * @hide
	 */
	public boolean isScrollingInDirection(float xvel, float yvel) {
		return !mFinished && Math.signum(xvel) == Math.signum(mFinalX - mStartX) &&
				Math.signum(yvel) == Math.signum(mFinalY - mStartY);
	}
}
```

Then modify the Sliding Menu lib to use the new scroller:

`SlidingMenu.java`

```java
public void scrollWithBounceInterpolator(){
	mViewAbove.scrollWithBounceInterpolator();
}
```

`CustomviewAbove.java`

```java
public void scrollWithBounceInterpolator() {
	int x = MeasuresUtils.DpToPx(getMeasuredWidth());
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

and the following on the `completeScroll` method:

```java
mBounceScroller.abortAnimation();
```

You should now be able to call `mSlidingMenu.scrollWithBounceInterpolator()` from your activity!

Conclusion
----------

The sources of my posts are on [Github], so do not hesitate to contribute by making pull requests, etc.

[Interpolator Doc]: http://developer.android.com/reference/android/view/animation/Interpolator.html
[Github]: https://github.com/castorflex/castorflex-blog/blob/master/source/_posts/2013-04-12-making-a-bounce-animation-for-your-sliding-menu.markdown
[SlidingMenu]: https://github.com/jfeinstein10/SlidingMenu
[Prixing]: https://play.google.com/store/apps/details?id=fr.epicdream.beamy
[Facebook]: https://play.google.com/store/apps/details?id=com.facebook.katana
[sliding menu pattern]: http://www.androiduipatterns.com/2012/06/emerging-ui-pattern-side-navigation.html
[Scroller]: http://developer.android.com/reference/android/widget/Scroller.html
