# Camera Focus

[TOC]

点击屏幕会有一系列focus的操作,由View::PreviewOverlay发起, 其通过**GestureDetector**触发

###### 补充说明

GestureDetector的使用:

1. 实例**GestureDetector.OnGestureListener**, 其下有好几个Listener类,  选择性实现.

2. 创建**GestureDetector**实例**mGestureDetector**, eg `mGestureDetector = new GestureDetector(getContext(), gestureListener);`

3. 在onTouch之类的事件中进行拦截,

   ```java
   @Override
   public boolean onTouchEvent(MotionEvent m) {
       // Pass the touch events to scale detector and gesture detector
       if (mGestureDetector != null) {
           mGestureDetector.onTouchEvent(m);
       }
       if (mTouchListener != null) {
           mTouchListener.onTouch(this, m);
       }
       return true;
   }
   ```

所以PreviewOverlay进行了事件的拦截, 并交给了mGestureDetector处理, 而mGestureDetector的listener在PhotoUI中. (其它UI里的GestureDetector.listener暂不分析)

PhotoUI判断出是单点事件, 便有PhotoModule通过FocusManager进行处理, trace流程如下:

- **at com.android.camera.ui.focus.FocusRingView.setFocusLocation(FocusRingView.java:228)**
- **at com.android.camera.FocusOverlayManager.onSingleTapUp(FocusOverlayManager.java:577)**
- **at com.android.camera.PhotoModule.onSingleTapUp(PhotoModule.java:3201)**
- **at com.android.camera.PhotoUI$1.onSingleTapUp(PhotoUI.java:121)**
- **at android.view.GestureDetector.onTouchEvent(GestureDetector.java:640)**
- **at com.android.camera.ui.PreviewOverlay.onTouchEvent(PreviewOverlay.java:219)**



###### Foucs 流程图

![](https://raw.githubusercontent.com/kevinkissum/picture/master/camera_focus.png)



#### Focus绘画

Focus界面初始化在FocusView中, 通过FocusOverlayManager进行focus相关的操作, FocusView中按自动对焦和手动对焦又分为

- AutoFocusRing
- ManualFocusRing

auto和manual又分为auto和autoSuccesed两种状态.

Focus的动画是由**DynamicAnimator**控制和管理, 暂分析手动对焦的流程

由trace可知, FocusManager通知FocusRIngVIew进行对焦处理.

```java
@Override
public void startActiveFocus() {
    //DynamicAnimator先进行invalidate
    mAnimator.update();
    mAnimator.invalidate(); //View draw
    long tMs = mAnimator.getTimeMillis();
    if (mAutoFocusRing.isActive() && !mAutoFocusRing.isExiting()) {
        mAutoFocusRing.stop(tMs);
    }
    if (mAutoFocusSuccessRing.isActive() && !mAutoFocusSuccessRing.isExiting()) {
        mAutoFocusSuccessRing.stop(tMs);
    }
    if (mManualFocusSuccessRing.isActive() && !mManualFocusSuccessRing.isExiting()) {
        mManualFocusSuccessRing.stop(tMs);
    }
    mManualFocusRing.start(tMs, 0.0f, mLastRadiusPx);
}
```
这里看一下**mAnimator.invalidate()**内容,  未能明白这段代码的意义 :(  注释掉之后圆圈画不出来

```java
@Override
public void invalidate() {
    if (!mIsDrawing && !mUpdateRequested) {
        mInvalidator.invalidate(); //?????
        mLastDrawTimeMillis = mClock.getTimeMillis();
    }

    mUpdateRequested = true;
}
```
之后走了ManualFocusRing.start流程,

```java
public void start(long startMs, float initialRadius, float targetRadius) {
    if (mFocusState != FocusState.STATE_INACTIVE) {
        Log.w(TAG, "start() called while the ring was still focusing!");
    }
    mRingRadius.stop();
    mRingRadius.setValue(initialRadius);
    mRingRadius.setTarget(targetRadius);
    mEnterStartMillis = startMs;
	//这里将FocusState状态修改
    mFocusState = FocusState.STATE_ENTER;
    //再一次调用这个, 会执行FocusRingView Draw
    mInvalidator.invalidate();
}
```
```java
FocusRingView.java

@Override
protected void onDraw(Canvas canvas) {
    if (isFirstDraw) {
        isFirstDraw = false;
        centerAutofocusRing();
    }

    if (mPreviewSize != null) {
        canvas.clipRect(mPreviewSize, Region.Op.REPLACE);
    }
    //调用Animator去draw
    mAnimator.draw(canvas);
}
```
```java
DynamicAnimator.java

public void draw(Canvas canvas) {
    mIsDrawing = true;
    mUpdateRequested = false;

    mDrawTimeMillis = mClock.getTimeMillis();

    if (mLastDrawTimeMillis <= 0) {
        mLastDrawTimeMillis = mDrawTimeMillis; // On the initial draw, dt is zero.
    }
    long dt = mDrawTimeMillis - mLastDrawTimeMillis;
    mLastDrawTimeMillis = mDrawTimeMillis;

    // Run the animation
    for (DynamicAnimation renderer : animations) {
        if (renderer.isActive()) {
            //render会依据dt的大小进行半径的变化
            renderer.draw(mDrawTimeMillis, dt, canvas);
        }
    }

    // If either the update or the draw methods requested new frames, then
    // invalidate the view which should give us another frame to work with.
    // Otherwise, stopAt the last update time.
    if (mUpdateRequested) {
        mInvalidator.invalidate();
    } else {
        mLastDrawTimeMillis = -1;
    }

    mIsDrawing = false;
}
```
ManualFocusRing继承FocusRingRenderer, render的draw在其子类中进行实现

```java
@Override
public void draw(long t, long dt, Canvas canvas) {
    //改变半径
    float ringRadius = mRingRadius.update(dt);
    processStates(t);

    if (!isActive()) {
        return;
    }

    mInvalidator.invalidate();
    int ringAlpha = 255;

    if (mFocusState == FocusState.STATE_FADE_OUT) {
        float rFade = InterpolateUtils.unitRatio(t, mExitStartMillis, mExitDurationMillis);
        ringAlpha = (int) InterpolateUtils.lerp(255, 0, mExitOpacityCurve.valueAt(rFade));
    } else if (mFocusState == FocusState.STATE_HARD_STOP) {
        float rFade = InterpolateUtils.unitRatio(t, mHardExitStartMillis,
              mHardExitDurationMillis);
        ringAlpha = (int) InterpolateUtils.lerp(255, 0, mExitOpacityCurve.valueAt(rFade));
    } else if (mFocusState == FocusState.STATE_INACTIVE) {
        ringAlpha = 0;
    }

    mRingPaint.setAlpha(ringAlpha);
    canvas.drawCircle(getCenterX(), getCenterY(), ringRadius/6, mRingPaint);
    canvas.drawCircle(getCenterX(), getCenterY(), ringRadius, mRingPaint);
}
```
###### TODO

这段实在看不懂!

后续由底层focus 成功会focusRingView会进行startFocusFocused绘画, 将现在的正在进行的绘制打断 或者 通过设定的时间判断在draw::**processState**中, 改变focus的状态值来进行停止.



#### Focus底层实现





