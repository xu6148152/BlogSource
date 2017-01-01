---
title: Animator at Android
date: 2016-12-04 21:17:02
tags:
---


### Animator

其是``Android3.0``之后加入的作用于对象属性的动画,``Animator``是所有属性动画的基类，其下有``ValueAnimator``,``AnimatorSet``,由``ValueAnimator``再拓展出``ObjectAnimator``,``TimeAnimator``

其整体结构图

![animator](./animator.png)

#### ValueAnimator

为作用对象提供简单的能够计算动画时间和值得时间引擎，它运行在自定义的``Handler``里面以确保所有的属性改变都会在``UI``线程。主要``API``:

```java
ValueAnimator ofInt(int... values) 
ValueAnimator ofArgb(int... values) //颜色值变化
ValueAnimator ofFloat(float... values) 
ValueAnimator ofPropertyValuesHolder(PropertyValuesHolder... values)
ValueAnimator ofObject(TypeEvaluator evaluator, Object... values)
```

这些方法里面都是进行值得初始化

```java
public void setIntValues(int... values) {
        if (values == null || values.length == 0) {
            return;
        }
        if (mValues == null || mValues.length == 0) {
            setValues(PropertyValuesHolder.ofInt("", values));
        } else {
            PropertyValuesHolder valuesHolder = mValues[0];
            valuesHolder.setIntValues(values);
        }
        // New property/values/target should cause re-initialization prior to starting
        mInitialized = false;
    }
```
都是重新封装转化成``PropertyValuesHolder``，这是封装动画属性和值得对象，之后会详细介绍

```java
public void setEvaluator(TypeEvaluator value) {
        if (value != null && mValues != null && mValues.length > 0) {
            mValues[0].setEvaluator(value);
        }
    }
```

设置估值器，最后也是由``PropertyValuesHolder``保存

```java
//参数代表是否反转动画
private void start(boolean playBackwards) {
        if (Looper.myLooper() == null) {
            throw new AndroidRuntimeException("Animators may only be run on Looper threads");
        }
        mReversing = playBackwards;
        mPlayingBackwards = playBackwards;
        //需要反转动画
        if (playBackwards && mSeekFraction != -1) {
            if (mSeekFraction == 0 && mCurrentIteration == 0) {
                // special case: reversing from seek-to-0 should act as if not seeked at all
                mSeekFraction = 0;
            } else if (mRepeatCount == INFINITE) {
                mSeekFraction = 1 - (mSeekFraction % 1);
            } else {
                mSeekFraction = 1 + mRepeatCount - (mCurrentIteration + mSeekFraction);
            }
            mCurrentIteration = (int) mSeekFraction;
            mSeekFraction = mSeekFraction % 1;
        }
        if (mCurrentIteration > 0 && mRepeatMode == REVERSE &&
                (mCurrentIteration < (mRepeatCount + 1) || mRepeatCount == INFINITE)) {
            // if we were seeked to some other iteration in a reversing animator,
            // figure out the correct direction to start playing based on the iteration
            if (playBackwards) {
                mPlayingBackwards = (mCurrentIteration % 2) == 0;
            } else {
                mPlayingBackwards = (mCurrentIteration % 2) != 0;
            }
        }
        int prevPlayingState = mPlayingState;
        mPlayingState = STOPPED;
        mStarted = true;
        mStartedDelay = false;
        mPaused = false;
        updateScaledDuration(); // in case the scale factor has changed since creation time
        //创建AnimatorHandler
        AnimationHandler animationHandler = getOrCreateAnimationHandler();
        animationHandler.mPendingAnimations.add(this);
        if (mStartDelay == 0) {
            // This sets the initial value of the animation, prior to actually starting it running
            if (prevPlayingState != SEEKED) {
                setCurrentPlayTime(0);
            }
            mPlayingState = STOPPED;
            mRunning = true;
            notifyStartListeners();
        }
        //开始动画
        animationHandler.start();
    }
```

开始动画的逻辑其实很简单，主要是判断动画是否需要反转。最终动画的更新交给``AnimationHandler``

##### AnimationHandler
处理所有运行中的动画的定时脉冲，这个机制能确保所有的动画都运行在``UI``线程，其使用``Choreograhper(用于调整动画输入和绘制的时机)``来执行周期的回调

```java
	private AnimationHandler() {
	    mChoreographer = Choreographer.getInstance();
	}
	
	public void start() {
	    scheduleAnimation();
	}  
	
	private void scheduleAnimation() {
            if (!mAnimationScheduled) {
                //动画未执行过，由Choreographer回调执行
                mChoreographer.postCallback(Choreographer.CALLBACK_ANIMATION, mAnimate, null);
                mAnimationScheduled = true;
            }
        }
        
    final Runnable mAnimate = new Runnable() {
            @Override
            public void run() {
                mAnimationScheduled = false;
                doAnimationFrame(mChoreographer.getFrameTime());
            }
        };
        
    void doAnimationFrame(long frameTime) {
    //参数代表执行哪一帧动画
            mLastFrameTime = frameTime;

            // mPendingAnimations holds any animations that have requested to be started
            // We're going to clear mPendingAnimations, but starting animation may
            // cause more to be added to the pending list (for example, if one animation
            // starting triggers another starting). So we loop until mPendingAnimations
            // is empty.
            while (mPendingAnimations.size() > 0) {
            //如果有等待执行的动画，清除并优先执行这些动画
                ArrayList<ValueAnimator> pendingCopy =
                        (ArrayList<ValueAnimator>) mPendingAnimations.clone();
                mPendingAnimations.clear();
                int count = pendingCopy.size();
                for (int i = 0; i < count; ++i) {
                    ValueAnimator anim = pendingCopy.get(i);
                    // If the animation has a startDelay, place it on the delayed list
                    if (anim.mStartDelay == 0) {
                        anim.startAnimation(this);
                    } else {
                        mDelayedAnims.add(anim);
                    }
                }
            }

            // Next, process animations currently sitting on the delayed queue, adding
            // them to the active animations if they are ready
            //将延时队列中的动画加入到准备完成队列，开始执行动画，清空队列
            int numDelayedAnims = mDelayedAnims.size();
            for (int i = 0; i < numDelayedAnims; ++i) {
                ValueAnimator anim = mDelayedAnims.get(i);
                if (anim.delayedAnimationFrame(frameTime)) {
                    mReadyAnims.add(anim);
                }
            }
            int numReadyAnims = mReadyAnims.size();
            if (numReadyAnims > 0) {
                for (int i = 0; i < numReadyAnims; ++i) {
                    ValueAnimator anim = mReadyAnims.get(i);
                    anim.startAnimation(this);
                    anim.mRunning = true;
                    mDelayedAnims.remove(anim);
                }
                mReadyAnims.clear();
            }

            // Now process all active animations. The return value from animationFrame()
            // tells the handler whether it should now be ended
            int numAnims = mAnimations.size();
            for (int i = 0; i < numAnims; ++i) {
                mTmpAnimations.add(mAnimations.get(i));
            }
            for (int i = 0; i < numAnims; ++i) {
                ValueAnimator anim = mTmpAnimations.get(i);
                if (mAnimations.contains(anim) && anim.doAnimationFrame(frameTime)) {
                    mEndingAnims.add(anim);
                }
            }
            mTmpAnimations.clear();
            if (mEndingAnims.size() > 0) {
                for (int i = 0; i < mEndingAnims.size(); ++i) {
                    mEndingAnims.get(i).endAnimation(this);
                }
                mEndingAnims.clear();
            }

            // Schedule final commit for the frame.
            mChoreographer.postCallback(Choreographer.CALLBACK_COMMIT, mCommit, null);

            // If there are still active or delayed animations, schedule a future call to
            // onAnimate to process the next frame of the animations.
            if (!mAnimations.isEmpty() || !mDelayedAnims.isEmpty()) {
                scheduleAnimation();
            }
        }         
        
```

流程图

![ValueAnimator](./ValueAnimator.png)

#### 插值器
插值器是描述动画变化的频率。其主要方法是  

```java
//输入动画基准值，得到计算后的某个时刻的值
float getInterpolation(float input);
```

类关系图:  
![interpolator](./interpolator.png)

系统已经为开发者准备了一些默认的插值器，我们来简单看下这些插值器的算法

##### LinearInterpolator(线性插值器)

```java
//输入既是输出
public float getInterpolation(float input) {
        return input;
}
```

![](./LinearInterpolator.png)
##### AccelerateDecelerateInterpolator(加减速插值器)

```java
public float getInterpolation(float input) {
        return (float)(Math.cos((input + 1) * Math.PI) / 2.0f) + 0.5f;
    }
```

![](./AccelerateDecelerateInterpolator.png)  

其特性是在动画开始和结束阶段变化缓慢，中间阶段变化迅速

##### AccelerateInterpolator(加速插值器)

```java
public float getInterpolation(float input) {
        if (mFactor == 1.0f) {
            return input * input;
        } else {
            return (float)Math.pow(input, mDoubleFactor);
        }
    }
```

![](./AccelerateInterpolator.png)  
其特点是动画变化速度一直加快，有个加速因子，默认为1。

##### AnticipateInterpolator

```java
public float getInterpolation(float t) {
        // a(t) = t * t * ((tension + 1) * t - tension)
        return t * t * ((mTension + 1) * t - mTension);
    }
```

![](./AnticipateInterpolator.png)

##### AnticipateOvershootInterpolator

```java
public float getInterpolation(float t) {
        // a(t, s) = t * t * ((s + 1) * t - s)
        // o(t, s) = t * t * ((s + 1) * t + s)
        // f(t) = 0.5 * a(t * 2, tension * extraTension), when t < 0.5
        // f(t) = 0.5 * (o(t * 2 - 2, tension * extraTension) + 2), when t <= 1.0
        if (t < 0.5f) return 0.5f * a(t * 2.0f, mTension);
        else return 0.5f * (o(t * 2.0f - 2.0f, mTension) + 2.0f);
    }
```

![](./AnticipateOvershootinterpolator.png)

还有很多别的插值器。这里就不多加赘述了。只要将相应的算法写入``getInterpolation``中即可

使用的地方

```java
void animateValue(float fraction) {
//计算动画值
        fraction = mInterpolator.getInterpolation(fraction);
        mCurrentFraction = fraction;
        int numValues = mValues.length;
        for (int i = 0; i < numValues; ++i) {
            mValues[i].calculateValue(fraction);
        }
        if (mUpdateListeners != null) {
            int numListeners = mUpdateListeners.size();
            for (int i = 0; i < numListeners; ++i) {
                mUpdateListeners.get(i).onAnimationUpdate(this);
            }
        }
    }
```

#### TypeEvaluator(估值器)

根据当前的动画频率，起始值，结束值计算当前值

![](./TypeEvaluator.png)

```java
public T evaluate(float fraction, T startValue, T endValue);
```

##### ArgbEvaluator

```java
public Object evaluate(float fraction, Object startValue, Object endValue) {
        int startInt = (Integer) startValue;
        int startA = (startInt >> 24) & 0xff;
        int startR = (startInt >> 16) & 0xff;
        int startG = (startInt >> 8) & 0xff;
        int startB = startInt & 0xff;

        int endInt = (Integer) endValue;
        int endA = (endInt >> 24) & 0xff;
        int endR = (endInt >> 16) & 0xff;
        int endG = (endInt >> 8) & 0xff;
        int endB = endInt & 0xff;

        return (int)((startA + (int)(fraction * (endA - startA))) << 24) |
                (int)((startR + (int)(fraction * (endR - startR))) << 16) |
                (int)((startG + (int)(fraction * (endG - startG))) << 8) |
                (int)((startB + (int)(fraction * (endB - startB))));
    }
```
算法是线性函数，主要因子``fraction``来自插值器的计算, ``y= a + kx``

##### IntEvaluator

```java
public Integer evaluate(float fraction, Integer startValue, Integer endValue) {
        int startInt = startValue;
        return (int)(startInt + fraction * (endValue - startInt));
    }
```

估值器主要实在``KeyFrames``中使用，而``KeyFrames``存在``PropertyValuesHolder``当中，最终``Choreographer``来更新动画使用

#### Choreograhper
协调动画，输入和绘制的时间。其会收到来自显示系统的定时脉冲，然后安排工作作为下一帧渲染的一部分。

```java
private Choreographer(Looper looper) {
        mLooper = looper;
        mHandler = new FrameHandler(looper);
        mDisplayEventReceiver = USE_VSYNC ? new FrameDisplayEventReceiver(looper) : null;
        mLastFrameTimeNanos = Long.MIN_VALUE;

        mFrameIntervalNanos = (long)(1000000000 / getRefreshRate());

        mCallbackQueues = new CallbackQueue[CALLBACK_LAST + 1];
        for (int i = 0; i <= CALLBACK_LAST; i++) {
            mCallbackQueues[i] = new CallbackQueue();
        }
    }
```
* 初始化，创建``FrameHandler``，创建回调队列

```java
public void postCallback(int callbackType, Runnable action, Object token) {
        postCallbackDelayed(callbackType, action, token, 0);
    }
    
private void postCallbackDelayedInternal(int callbackType,
        Object action, Object token, long delayMillis) {
    if (DEBUG_FRAMES) {
        Log.d(TAG, "PostCallback: type=" + callbackType
                + ", action=" + action + ", token=" + token
                + ", delayMillis=" + delayMillis);
    }

    synchronized (mLock) {
        final long now = SystemClock.uptimeMillis();
        final long dueTime = now + delayMillis;
        mCallbackQueues[callbackType].addCallbackLocked(dueTime, action, token);

        if (dueTime <= now) {
            scheduleFrameLocked(now);
        } else {
            Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_CALLBACK, action);
            msg.arg1 = callbackType;
            msg.setAsynchronous(true);
            mHandler.sendMessageAtTime(msg, dueTime);
        }
    }
}
```

* 将回调加入回调队列，如果没有延时，锁定当前帧，如果使用``VSYNC``，诺当前运行在相同的线程，直接通过``FrameDisplayEventReceiver``要求同步，其他情况发送消息到``Handler``中执行。``Handler``处理三种消息,``MSG_DO_FRAME``,``MSG_DO_SCHEDULE_VSYNC``, ``MSG_DO_SCHEDULE_CALLBACK``

```java
void doCallbacks(int callbackType, long frameTimeNanos) {
        CallbackRecord callbacks;
        synchronized (mLock) {
            // We use "now" to determine when callbacks become due because it's possible
            // for earlier processing phases in a frame to post callbacks that should run
            // in a following phase, such as an input event that causes an animation to start.
            final long now = System.nanoTime();
            callbacks = mCallbackQueues[callbackType].extractDueCallbacksLocked(
                    now / TimeUtils.NANOS_PER_MS);
            if (callbacks == null) {
                return;
            }
            mCallbacksRunning = true;

            // Update the frame time if necessary when committing the frame.
            // We only update the frame time if we are more than 2 frames late reaching
            // the commit phase.  This ensures that the frame time which is observed by the
            // callbacks will always increase from one frame to the next and never repeat.
            // We never want the next frame's starting frame time to end up being less than
            // or equal to the previous frame's commit frame time.  Keep in mind that the
            // next frame has most likely already been scheduled by now so we play it
            // safe by ensuring the commit time is always at least one frame behind.
            if (callbackType == Choreographer.CALLBACK_COMMIT) {
                final long jitterNanos = now - frameTimeNanos;
                Trace.traceCounter(Trace.TRACE_TAG_VIEW, "jitterNanos", (int) jitterNanos);
                if (jitterNanos >= 2 * mFrameIntervalNanos) {
                    final long lastFrameOffset = jitterNanos % mFrameIntervalNanos
                            + mFrameIntervalNanos;
                    if (DEBUG_JANK) {
                        Log.d(TAG, "Commit callback delayed by " + (jitterNanos * 0.000001f)
                                + " ms which is more than twice the frame interval of "
                                + (mFrameIntervalNanos * 0.000001f) + " ms!  "
                                + "Setting frame time to " + (lastFrameOffset * 0.000001f)
                                + " ms in the past.");
                        mDebugPrintNextFrameTimeDelta = true;
                    }
                    frameTimeNanos = now - lastFrameOffset;
                    mLastFrameTimeNanos = frameTimeNanos;
                }
            }
        }
        try {
            Trace.traceBegin(Trace.TRACE_TAG_VIEW, CALLBACK_TRACE_TITLES[callbackType]);
            for (CallbackRecord c = callbacks; c != null; c = c.next) {
                if (DEBUG_FRAMES) {
                    Log.d(TAG, "RunCallback: type=" + callbackType
                            + ", action=" + c.action + ", token=" + c.token
                            + ", latencyMillis=" + (SystemClock.uptimeMillis() - c.dueTime));
                }
                c.run(frameTimeNanos);
            }
        } finally {
            synchronized (mLock) {
            //执行所有的回调
                mCallbacksRunning = false;
                do {
                    final CallbackRecord next = callbacks.next;
                    recycleCallbackLocked(callbacks);
                    callbacks = next;
                } while (callbacks != null);
            }
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
    }
```

* ``MSG_DO_FRAME``:判断当前帧时间，低于上次绘制时间，要求同步。否则调用``doCallbacks``，执行回调队列中的回调

##### 流程图

#### ObjectAnimator

支持对目标对象的属性做动画，对``ValueAnimator``的拓展

```java
//可以直接设置target,propertyName
public static ObjectAnimator ofInt(Object target, String propertyName, int... values) {
        ObjectAnimator anim = new ObjectAnimator(target, propertyName);
        anim.setIntValues(values);
        return anim;
    }
    
public void setTarget(@Nullable Object target) {
        final Object oldTarget = getTarget();
        //老的目标与当前目标不同，取消。设置弱引用目标对象
        if (oldTarget != target) {
            if (isStarted()) {
                cancel();
            }
            mTarget = target == null ? null : new WeakReference<Object>(target);
            // New target should cause re-initialization prior to starting
            mInitialized = false;
        }
    }
    
public void start() {
        // See if any of the current active/pending animators need to be canceled
        AnimationHandler handler = sAnimationHandler.get();
        if (handler != null) {
            int numAnims = handler.mAnimations.size();
            for (int i = numAnims - 1; i >= 0; i--) {
                if (handler.mAnimations.get(i) instanceof ObjectAnimator) {
                    ObjectAnimator anim = (ObjectAnimator) handler.mAnimations.get(i);
                    if (anim.mAutoCancel && hasSameTargetAndProperties(anim)) {
                    //相同的目标和属性，取消之前的动画
                        anim.cancel();
                    }
                }
            }
            
            //等待的动画也一样
            numAnims = handler.mPendingAnimations.size();
            for (int i = numAnims - 1; i >= 0; i--) {
                if (handler.mPendingAnimations.get(i) instanceof ObjectAnimator) {
                    ObjectAnimator anim = (ObjectAnimator) handler.mPendingAnimations.get(i);
                    if (anim.mAutoCancel && hasSameTargetAndProperties(anim)) {
                        anim.cancel();
                    }
                }
            }
            
            //延时的动画也一样
            numAnims = handler.mDelayedAnims.size();
            for (int i = numAnims - 1; i >= 0; i--) {
                if (handler.mDelayedAnims.get(i) instanceof ObjectAnimator) {
                    ObjectAnimator anim = (ObjectAnimator) handler.mDelayedAnims.get(i);
                    if (anim.mAutoCancel && hasSameTargetAndProperties(anim)) {
                        anim.cancel();
                    }
                }
            }
        }
        if (DBG) {
            Log.d(LOG_TAG, "Anim target, duration: " + getTarget() + ", " + getDuration());
            for (int i = 0; i < mValues.length; ++i) {
                PropertyValuesHolder pvh = mValues[i];
                Log.d(LOG_TAG, "   Values[" + i + "]: " +
                    pvh.getPropertyName() + ", " + pvh.mKeyframes.getValue(0) + ", " +
                    pvh.mKeyframes.getValue(1));
            }
        }
        super.start();
    }
    
void animateValue(float fraction) {
        final Object target = getTarget();
        if (mTarget != null && target == null) {
            // We lost the target reference, cancel and clean up.
            cancel();
            return;
        }

        super.animateValue(fraction);
        int numValues = mValues.length;
        for (int i = 0; i < numValues; ++i) {
            mValues[i].setAnimatedValue(target);
        }
    }
    
public void setupStartValues() {
		  //初始化动画
        initAnimation();

        final Object target = getTarget();
        if (target != null) {
            final int numValues = mValues.length;
            for (int i = 0; i < numValues; ++i) {
                mValues[i].setupStartValue(target);
            }
        }
    }
    
void initAnimation() {
        if (!mInitialized) {
            // mValueType may change due to setter/getter setup; do this before calling super.init(),
            // which uses mValueType to set up the default type evaluator.
            final Object target = getTarget();
            if (target != null) {
                final int numValues = mValues.length;
                for (int i = 0; i < numValues; ++i) {
                //初始化getter setter
                   mValues[i].setupSetterAndGetter(target);
                }
            }
            super.initAnimation();
        }
    }                 
```

* ``ObjectAnimator``比``ValueAnimator``多了``target``和``Property``，这些信息都在``PropertyValuesHolder``,必须要要有getter/setter才能更新目标对象


#### PropertyValuesHolder 处理setter, getter的寻找

```java
void setupSetterAndGetter(Object target) {
        mKeyframes.invalidateCache();
        //属性不为空时,从Propery拿值，并设置到keyFrame
        if (mProperty != null) {
            // check to make sure that mProperty is on the class of target
            try {
                Object testValue = null;
                List<Keyframe> keyframes = mKeyframes.getKeyframes();
                int keyframeCount = keyframes == null ? 0 : keyframes.size();
                for (int i = 0; i < keyframeCount; i++) {
                    Keyframe kf = keyframes.get(i);
                    if (!kf.hasValue() || kf.valueWasSetOnStart()) {
                        if (testValue == null) {
                            testValue = convertBack(mProperty.get(target));
                        }
                        kf.setValue(testValue);
                        kf.setValueWasSetOnStart(true);
                    }
                }
                return;
            } catch (ClassCastException e) {
                Log.w("PropertyValuesHolder","No such property (" + mProperty.getName() +
                        ") on target object " + target + ". Trying reflection instead");
                mProperty = null;
            }
        }
        // We can't just say 'else' here because the catch statement sets mProperty to null.
        //如果property为空，从目标对象中寻找getter setter
        if (mProperty == null) {
            Class targetClass = target.getClass();
            if (mSetter == null) {
                setupSetter(targetClass);
            }
            List<Keyframe> keyframes = mKeyframes.getKeyframes();
            int keyframeCount = keyframes == null ? 0 : keyframes.size();
            for (int i = 0; i < keyframeCount; i++) {
                Keyframe kf = keyframes.get(i);
                if (!kf.hasValue() || kf.valueWasSetOnStart()) {
                    if (mGetter == null) {
                    //getter没找到，再次寻找
                        setupGetter(targetClass);
                        //依然没找到
                        if (mGetter == null) {
                            // Already logged the error - just return to avoid NPE
                            return;
                        }
                    }
                    try {
                        Object value = convertBack(mGetter.invoke(target));
                        //找到getter，将值传给keyFrame
                        kf.setValue(value);
                        kf.setValueWasSetOnStart(true);
                    } catch (InvocationTargetException e) {
                        Log.e("PropertyValuesHolder", e.toString());
                    } catch (IllegalAccessException e) {
                        Log.e("PropertyValuesHolder", e.toString());
                    }
                }
            }
        }
    }
    
void setAnimatedValue(Object target) {
        if (mProperty != null) {
            mProperty.set(target, getAnimatedValue());
        }
        if (mSetter != null) {
        //setter不为空，会调用目标对象的setter
            try {
                mTmpValueArray[0] = getAnimatedValue();
                mSetter.invoke(target, mTmpValueArray);
            } catch (InvocationTargetException e) {
                Log.e("PropertyValuesHolder", e.toString());
            } catch (IllegalAccessException e) {
                Log.e("PropertyValuesHolder", e.toString());
            }
        }
    }     
```
* ``PropertyValuesHolder``主要持有动画的相关属性信息，包括关键帧集，插值器，估值器等

#### KeyFrame
动画关键帧，保存动画某一帧的时间/值。被``ValueAnimator``使用来计算两个值之间的动画值。``PropertyValuesHolder``保存了``KeyFrames``其保存了多个``KeyFrame``。简单来说``KeyFrame``就是我们做动画的起始值，中间值，结束值，这些都是由开发者定义。

```java
public void setIntValues(int... values) {
        mValueType = int.class;
        mKeyframes = KeyframeSet.ofInt(values);
    }

keyFrameSet:
public static KeyframeSet ofInt(int... values) {
        int numKeyframes = values.length;
        IntKeyframe keyframes[] = new IntKeyframe[Math.max(numKeyframes,2)];
        if (numKeyframes == 1) {
        //当只设置了一个值时，会有两个关键帧，起始关键这和结束关键帧
            keyframes[0] = (IntKeyframe) Keyframe.ofInt(0f);
            keyframes[1] = (IntKeyframe) Keyframe.ofInt(1f, values[0]);
        } else {
        //起始关键帧
            keyframes[0] = (IntKeyframe) Keyframe.ofInt(0f, values[0]);
            for (int i = 1; i < numKeyframes; ++i) {
            //计算每个阶段的关键帧
                keyframes[i] =
                        (IntKeyframe) Keyframe.ofInt((float) i / (numKeyframes - 1), values[i]);
            }
        }
        return new IntKeyframeSet(keyframes);
    }    
```
* 动画至少会包含两个关键帧，一个起始关键帧，一个结束关键帧。中间有多少关键帧由开发者自行定义

#### AnimatorSet
允许同时播放多个动画或者定义动画的播放顺序

```java
public void start() {
        mTerminated = false;
        mStarted = true;
        mPaused = false;

        for (Node node : mNodes) {
            //禁止异步运行node.animation.setAllowRunningAsynchronously(false);
        }

        //设置所有动画的长度
        if (mDuration >= 0) {
            // If the duration was set on this AnimatorSet, pass it along to all child animations
            for (Node node : mNodes) {
                // TODO: don't set the duration of the timing-only nodes created by AnimatorSet to
                // insert "play-after" delays
                node.animation.setDuration(mDuration);
            }
        }
        //设置所有动画的插值器
        if (mInterpolator != null) {
            for (Node node : mNodes) {
                node.animation.setInterpolator(mInterpolator);
            }
        }
        // First, sort the nodes (if necessary). This will ensure that sortedNodes
        // contains the animation nodes in the correct order.
        //对所有的动画节点进行排序
        sortNodes();

		  //清除老的动画监听	
        int numSortedNodes = mSortedNodes.size();
        for (int i = 0; i < numSortedNodes; ++i) {
            Node node = mSortedNodes.get(i);
            // First, clear out the old listeners
            ArrayList<AnimatorListener> oldListeners = node.animation.getListeners();
            if (oldListeners != null && oldListeners.size() > 0) {
                final ArrayList<AnimatorListener> clonedListeners = new
                        ArrayList<AnimatorListener>(oldListeners);

                for (AnimatorListener listener : clonedListeners) {
                    if (listener instanceof DependencyListener ||
                            listener instanceof AnimatorSetListener) {
                        node.animation.removeListener(listener);
                    }
                }
            }
        }

        // nodesToStart holds the list of nodes to be started immediately. We don't want to
        // start the animations in the loop directly because we first need to set up
        // dependencies on all of the nodes. For example, we don't want to start an animation
        // when some other animation also wants to start when the first animation begins.
        final ArrayList<Node> nodesToStart = new ArrayList<Node>();
        for (int i = 0; i < numSortedNodes; ++i) {
            Node node = mSortedNodes.get(i);
            if (mSetListener == null) {
                mSetListener = new AnimatorSetListener(this);
            }
            //动画节点没有依赖别的节点，添加到即将开始的队列中
            if (node.dependencies == null || node.dependencies.size() == 0) {
                nodesToStart.add(node);
            } else {
            //如果有依赖的话，添加依赖监听，并把依赖添加到临时依赖链中
                int numDependencies = node.dependencies.size();
                for (int j = 0; j < numDependencies; ++j) {
                    Dependency dependency = node.dependencies.get(j);
                    dependency.node.animation.addListener(
                            new DependencyListener(this, node, dependency.rule));
                }
                node.tmpDependencies = (ArrayList<Dependency>) node.dependencies.clone();
            }
            node.animation.addListener(mSetListener);
        }
        // Now that all dependencies are set up, start the animations that should be started.
        //开始动画
        if (mStartDelay <= 0) {
            for (Node node : nodesToStart) {
                node.animation.start();
                mPlayingSet.add(node.animation);
            }
        } else {
            mDelayAnim = ValueAnimator.ofFloat(0f, 1f);
            mDelayAnim.setDuration(mStartDelay);
            mDelayAnim.addListener(new AnimatorListenerAdapter() {
                boolean canceled = false;
                public void onAnimationCancel(Animator anim) {
                    canceled = true;
                }
                public void onAnimationEnd(Animator anim) {
                    if (!canceled) {
                        int numNodes = nodesToStart.size();
                        for (int i = 0; i < numNodes; ++i) {
                            Node node = nodesToStart.get(i);
                            node.animation.start();
                            mPlayingSet.add(node.animation);
                        }
                    }
                    mDelayAnim = null;
                }
            });
            mDelayAnim.start();
        }
        //动画开始回调
        if (mListeners != null) {
            ArrayList<AnimatorListener> tmpListeners =
                    (ArrayList<AnimatorListener>) mListeners.clone();
            int numListeners = tmpListeners.size();
            for (int i = 0; i < numListeners; ++i) {
                tmpListeners.get(i).onAnimationStart(this);
            }
        }
        //如果没有动画，动画结束回调
        if (mNodes.size() == 0 && mStartDelay == 0) {
            // Handle unusual case where empty AnimatorSet is started - should send out
            // end event immediately since the event will not be sent out at all otherwise
            mStarted = false;
            if (mListeners != null) {
                ArrayList<AnimatorListener> tmpListeners =
                        (ArrayList<AnimatorListener>) mListeners.clone();
                int numListeners = tmpListeners.size();
                for (int i = 0; i < numListeners; ++i) {
                    tmpListeners.get(i).onAnimationEnd(this);
                }
            }
        }
    }   
```

* 流程: 
  1. 首先禁止异步运行动画，设置所有动画的时间和插值器。
  2. 对所有的动画进行排序(根据依赖关系),取消所有老的监听。如果动画没有依赖，加入即将开始的动画队列，否则创建依赖监听，创建临时依赖
  3. 没有延时，启动动画。否则创建延时动画，延时动画结束后，后续动画开始
  4. 动画开始监听回调，如果没有动画，动画结束监听回调

```java
动画排序
private void sortNodes() {
        if (mNeedsSort) {
            mSortedNodes.clear();
            ArrayList<Node> roots = new ArrayList<Node>();
            int numNodes = mNodes.size();
            //没有依赖的动画变成根节点
            for (int i = 0; i < numNodes; ++i) {
                Node node = mNodes.get(i);
                if (node.dependencies == null || node.dependencies.size() == 0) {
                    roots.add(node);
                }
            }
            ArrayList<Node> tmpRoots = new ArrayList<Node>();
            while (roots.size() > 0) {
            //遍历每个根节点
                int numRoots = roots.size();
                for (int i = 0; i < numRoots; ++i) {
                    Node root = roots.get(i);
                    mSortedNodes.add(root);
                    if (root.nodeDependents != null) {
                        int numDependents = root.nodeDependents.size();
                        //根节点有依赖
                        for (int j = 0; j < numDependents; ++j) {
                            Node node = root.nodeDependents.get(j);
                            //将此根节点从依赖中移除node.nodeDependencies.remove(root);
                            if (node.nodeDependencies.size() == 0) {
                                tmpRoots.add(node);
                            }
                        }
                    }
                }
                roots.clear();
                roots.addAll(tmpRoots);
                tmpRoots.clear();
            }
            mNeedsSort = false;
            if (mSortedNodes.size() != mNodes.size()) {
                throw new IllegalStateException("Circular dependencies cannot exist"
                        + " in AnimatorSet");
            }
        } else {
            // Doesn't need sorting, but still need to add in the nodeDependencies list
            // because these get removed as the event listeners fire and the dependencies
            // are satisfied
            int numNodes = mNodes.size();
            for (int i = 0; i < numNodes; ++i) {
                Node node = mNodes.get(i);
                if (node.dependencies != null && node.dependencies.size() > 0) {
                    int numDependencies = node.dependencies.size();
                    for (int j = 0; j < numDependencies; ++j) {
                        Dependency dependency = node.dependencies.get(j);
                        if (node.nodeDependencies == null) {
                            node.nodeDependencies = new ArrayList<Node>();
                        }
                        if (!node.nodeDependencies.contains(dependency.node)) {
                            node.nodeDependencies.add(dependency.node);
                        }
                    }
                }
                // nodes are 'done' by default; they become un-done when started, and done
                // again when ended
                node.done = false;
            }
        }
    } 
```  
* 动画排序规则: 
  * 如果需要排序
      1. 没有依赖的动画变成根节点.
      2. 循环遍历所有根节点，将根节点从依赖节点中移除
  * 不需要排序
      1. 添加回依赖

##### Node
代表动画和其依赖。其包含一个动画信息，节点动画依赖集合，独立节点集合

##### Dependency
动画节点与节点之间的依赖描述

```java
static final int WITH = 0; // dependent node must start with this dependency node
        static final int AFTER = 1; // dependent node must start when this dependency node finishes
```
* 包含两个规则，一起，后续

```java

//多个动画一起播放，添加动画``WITH``依赖
public void playTogether(Animator... items) {
        if (items != null) {
            mNeedsSort = true;
            Builder builder = play(items[0]);
            for (int i = 1; i < items.length; ++i) {
                builder.with(items[i]);
            }
        }
    }
    
public Builder with(Animator anim) {
            Node node = mNodeMap.get(anim);
            if (node == null) {
                node = new Node(anim);
                mNodeMap.put(anim, node);
                mNodes.add(node);
            }
            Dependency dependency = new Dependency(mCurrentNode, Dependency.WITH);
            node.addDependency(dependency);
            return this;
        }

//多个动画串行播放        
public void playSequentially(Animator... items) {
        if (items != null) {
            mNeedsSort = true;
            if (items.length == 1) {
                play(items[0]);
            } else {
                mReversible = false;
                for (int i = 0; i < items.length - 1; ++i) {
                    play(items[i]).before(items[i+1]);
                }
            }
        }
    }
//在某个动画之前播放，添加``AFTER``依赖
public Builder before(Animator anim) {
            mReversible = false;
            Node node = mNodeMap.get(anim);
            if (node == null) {
                node = new Node(anim);
                mNodeMap.put(anim, node);
                mNodes.add(node);
            }
            Dependency dependency = new Dependency(mCurrentNode, Dependency.AFTER);
            node.addDependency(dependency);
            return this;
        }                
``` 

#### LayoutTransition
``ViewGroup``布局发生改变时播放动画。这个动画的核心概念是两个类型的改变会引起四种不同的动画运行。这两种改变时``appearing``和``disappearing``。

```java
ViewGroup
setLayoutTransition(LayoutTransition transition)
```
动画类型，用一个字节表示

```java

private static final int FLAG_APPEARING             = 0x01;
    private static final int FLAG_DISAPPEARING          = 0x02;
    private static final int FLAG_CHANGE_APPEARING      = 0x04;
    private static final int FLAG_CHANGE_DISAPPEARING   = 0x08;
    private static final int FLAG_CHANGING              = 0x10;
    
    
private int mTransitionTypes = FLAG_CHANGE_APPEARING | FLAG_CHANGE_DISAPPEARING |
            FLAG_APPEARING | FLAG_DISAPPEARING;
            
private void setupChangeAnimation(final ViewGroup parent, final int changeReason,
            Animator baseAnimator, final long duration, final View child) {

        // If we already have a listener for this child, then we've already set up the
        // changing animation we need. Multiple calls for a child may occur when several
        // add/remove operations are run at once on a container; each one will trigger
        // changes for the existing children in the container.
        if (layoutChangeListenerMap.get(child) != null) {
            return;
        }

        // Don't animate items up from size(0,0); this is likely because the objects
        // were offscreen/invisible or otherwise measured to be infinitely small. We don't
        // want to see them animate into their real size; just ignore animation requests
        // on these views
        if (child.getWidth() == 0 && child.getHeight() == 0) {
            return;
        }

        // Make a copy of the appropriate animation
        final Animator anim = baseAnimator.clone();

        // Set the target object for the animation
        anim.setTarget(child);

        // A ObjectAnimator (or AnimatorSet of them) can extract start values from
        // its target object
        anim.setupStartValues();

        // If there's an animation running on this view already, cancel it
        Animator currentAnimation = pendingAnimations.get(child);
        if (currentAnimation != null) {
            currentAnimation.cancel();
            pendingAnimations.remove(child);
        }
        // Cache the animation in case we need to cancel it later
        pendingAnimations.put(child, anim);

        // For the animations which don't get started, we have to have a means of
        // removing them from the cache, lest we leak them and their target objects.
        // We run an animator for the default duration+100 (an arbitrary time, but one
        // which should far surpass the delay between setting them up here and
        // handling layout events which start them.
        ValueAnimator pendingAnimRemover = ValueAnimator.ofFloat(0f, 1f).
                setDuration(duration + 100);
        pendingAnimRemover.addListener(new AnimatorListenerAdapter() {
            @Override
            public void onAnimationEnd(Animator animation) {
                pendingAnimations.remove(child);
            }
        });
        pendingAnimRemover.start();

        // Add a listener to track layout changes on this view. If we don't get a callback,
        // then there's nothing to animate.
        final View.OnLayoutChangeListener listener = new View.OnLayoutChangeListener() {
            public void onLayoutChange(View v, int left, int top, int right, int bottom,
                    int oldLeft, int oldTop, int oldRight, int oldBottom) {

                // Tell the animation to extract end values from the changed object
                anim.setupEndValues();
                if (anim instanceof ValueAnimator) {
                    boolean valuesDiffer = false;
                    ValueAnimator valueAnim = (ValueAnimator)anim;
                    PropertyValuesHolder[] oldValues = valueAnim.getValues();
                    for (int i = 0; i < oldValues.length; ++i) {
                        PropertyValuesHolder pvh = oldValues[i];
                        if (pvh.mKeyframes instanceof KeyframeSet) {
                            KeyframeSet keyframeSet = (KeyframeSet) pvh.mKeyframes;
                            if (keyframeSet.mFirstKeyframe == null ||
                                    keyframeSet.mLastKeyframe == null ||
                                    !keyframeSet.mFirstKeyframe.getValue().equals(
                                            keyframeSet.mLastKeyframe.getValue())) {
                                valuesDiffer = true;
                            }
                        } else if (!pvh.mKeyframes.getValue(0).equals(pvh.mKeyframes.getValue(1))) {
                            valuesDiffer = true;
                        }
                    }
                    if (!valuesDiffer) {
                        return;
                    }
                }

                long startDelay = 0;
                switch (changeReason) {
                    case APPEARING:
                        startDelay = mChangingAppearingDelay + staggerDelay;
                        staggerDelay += mChangingAppearingStagger;
                        if (mChangingAppearingInterpolator != sChangingAppearingInterpolator) {
                            anim.setInterpolator(mChangingAppearingInterpolator);
                        }
                        break;
                    case DISAPPEARING:
                        startDelay = mChangingDisappearingDelay + staggerDelay;
                        staggerDelay += mChangingDisappearingStagger;
                        if (mChangingDisappearingInterpolator !=
                                sChangingDisappearingInterpolator) {
                            anim.setInterpolator(mChangingDisappearingInterpolator);
                        }
                        break;
                    case CHANGING:
                        startDelay = mChangingDelay + staggerDelay;
                        staggerDelay += mChangingStagger;
                        if (mChangingInterpolator != sChangingInterpolator) {
                            anim.setInterpolator(mChangingInterpolator);
                        }
                        break;
                }
                anim.setStartDelay(startDelay);
                anim.setDuration(duration);

                Animator prevAnimation = currentChangingAnimations.get(child);
                if (prevAnimation != null) {
                    prevAnimation.cancel();
                }
                Animator pendingAnimation = pendingAnimations.get(child);
                if (pendingAnimation != null) {
                    pendingAnimations.remove(child);
                }
                // Cache the animation in case we need to cancel it later
                currentChangingAnimations.put(child, anim);

                parent.requestTransitionStart(LayoutTransition.this);

                // this only removes listeners whose views changed - must clear the
                // other listeners later
                child.removeOnLayoutChangeListener(this);
                layoutChangeListenerMap.remove(child);
            }
        };
        // Remove the animation from the cache when it ends
        anim.addListener(new AnimatorListenerAdapter() {

            @Override
            public void onAnimationStart(Animator animator) {
                if (hasListeners()) {
                    ArrayList<TransitionListener> listeners =
                            (ArrayList<TransitionListener>) mListeners.clone();
                    for (TransitionListener listener : listeners) {
                        listener.startTransition(LayoutTransition.this, parent, child,
                                changeReason == APPEARING ?
                                        CHANGE_APPEARING : changeReason == DISAPPEARING ?
                                        CHANGE_DISAPPEARING : CHANGING);
                    }
                }
            }

            @Override
            public void onAnimationCancel(Animator animator) {
                child.removeOnLayoutChangeListener(listener);
                layoutChangeListenerMap.remove(child);
            }

            @Override
            public void onAnimationEnd(Animator animator) {
                currentChangingAnimations.remove(child);
                if (hasListeners()) {
                    ArrayList<TransitionListener> listeners =
                            (ArrayList<TransitionListener>) mListeners.clone();
                    for (TransitionListener listener : listeners) {
                        listener.endTransition(LayoutTransition.this, parent, child,
                                changeReason == APPEARING ?
                                        CHANGE_APPEARING : changeReason == DISAPPEARING ?
                                        CHANGE_DISAPPEARING : CHANGING);
                    }
                }
            }
        });

        child.addOnLayoutChangeListener(listener);
        // cache the listener for later removal
        layoutChangeListenerMap.put(child, listener);
    }            

```

* 主要流程
  1. 启动一个延时动画来移除为执行的动画
  2. 监听布局变化,如果是``ValueAnimator``，判断起始帧和结束帧是否一样，如果一样表示结束.否则根据布局改变的原因设置动画类型，开始动画
  3. 移除缓存动画
     
#### ViewPropertyAnimator
自动优化``View``对象的属性动画。如果想做1个或者两个属性动画，使用``ObjectAnimator``，其也可以设置属性值，合理的刷新视图。但是如果有多个属性要同时动画，或者想要使用更简洁的语法，那么可以使用``ViewPropertyAnimator``

```java
private void animateProperty(int constantName, float toValue) {
        float fromValue = getValue(constantName);
        float deltaValue = toValue - fromValue;
        animatePropertyBy(constantName, fromValue, deltaValue);
    }
    
private void animatePropertyBy(int constantName, float startValue, float byValue) {
        // First, cancel any existing animations on this property
        if (mAnimatorMap.size() > 0) {
            Animator animatorToCancel = null;
            Set<Animator> animatorSet = mAnimatorMap.keySet();
            for (Animator runningAnim : animatorSet) {
                PropertyBundle bundle = mAnimatorMap.get(runningAnim);
                if (bundle.cancel(constantName)) {
                    // property was canceled - cancel the animation if it's now empty
                    // Note that it's safe to break out here because every new animation
                    // on a property will cancel a previous animation on that property, so
                    // there can only ever be one such animation running.
                    if (bundle.mPropertyMask == NONE) {
                        // the animation is no longer changing anything - cancel it
                        animatorToCancel = runningAnim;
                        break;
                    }
                }
            }
            if (animatorToCancel != null) {
                animatorToCancel.cancel();
            }
        }

        NameValuesHolder nameValuePair = new NameValuesHolder(constantName, startValue, byValue);
        mPendingAnimations.add(nameValuePair);
        mView.removeCallbacks(mAnimationStarter);
        mView.postOnAnimation(mAnimationStarter);
    }    
```

* 上述流程, 做属性动画都会调用``animateProperty``，其内部会调用``animatePropertyBy``
   1. 首先会取消这个属性上存在的动画
   2. 构建``NameValuesHolder``
   3. 调用``removeCallbacks(runnable)``，取消之前动画
   4. 调用``postOnAnimation(runnable)``，开始动画

### Transition

主要用于做转场动画，其会保存场景转变的信息。任何转变有两个主要工作: 1. 捕获属性值 2. 基于捕获的属性值改变做动画。**无法与``TextureView``和``SurfaceView``一起使用。对于``SurfaceView``，由于其是从非UI线程更新UI，因此会造成不同步。对于``TextureView``，由于转场动画依赖``ViewOverlay``，而其又无法与``TextureView``一起工作。

#### 结构图

![transition](./transition.png)

```java
Transition.java

//捕获开始场景的属性
public abstract void captureStartValues(TransitionValues transitionValues);

//捕获结束场景的属性
public abstract void captureEndValues(TransitionValues transitionValues);


```

##### 开始转场动画
```java
public static void beginDelayedTransition(final ViewGroup sceneRoot, Transition transition) {
        if (!sPendingTransitions.contains(sceneRoot) && sceneRoot.isLaidOut()) {
            if (Transition.DBG) {
                Log.d(LOG_TAG, "beginDelayedTransition: root, transition = " +
                        sceneRoot + ", " + transition);
            }
            sPendingTransitions.add(sceneRoot);
            if (transition == null) {
                transition = sDefaultTransition;//使用默认转场
            }
            final Transition transitionClone = transition.clone();
            sceneChangeSetup(sceneRoot, transitionClone);
            Scene.setCurrentScene(sceneRoot, null);
            sceneChangeRunTransition(sceneRoot, transitionClone);
        }
    }
    
private static void sceneChangeSetup(ViewGroup sceneRoot, Transition transition) {

        // Capture current values
        ArrayList<Transition> runningTransitions = getRunningTransitions().get(sceneRoot);

        if (runningTransitions != null && runningTransitions.size() > 0) {
            for (Transition runningTransition : runningTransitions) {
                runningTransition.pause(sceneRoot);
            }
        }

        if (transition != null) {
            transition.captureValues(sceneRoot, true);
        }

        // Notify previous scene that it is being exited
        Scene previousScene = Scene.getCurrentScene(sceneRoot);
        if (previousScene != null) {
            previousScene.exit();
        }
}

private static void sceneChangeRunTransition(final ViewGroup sceneRoot,
            final Transition transition) {
        if (transition != null && sceneRoot != null) {
            MultiListener listener = new MultiListener(transition, sceneRoot);
            sceneRoot.addOnAttachStateChangeListener(listener);
            //设置绘制前监听
            sceneRoot.getViewTreeObserver().addOnPreDrawListener(listener);
        }
}    
```

* 流程
  1. 未设置``Transition``，使用默认``Transition``，也就是``AutoTransition``
  2. 暂停当前正在运行的转场动画，捕获当前场景，取消之前的转场动画
  3. 设置当前场景
  4. 设置绘制(``preDraw``)监听，在其当中捕获场景和开始转场动画

##### AutoTransition
组合转场动画，包括渐变和区域改变。``Transition``由``TransitionManger``管理。一般可以使用``beginDelayedTransition``开始一个转场动画，默认的转场动画为``AutoTransition``:

```java
TransitionManager.beginDelayedTransition(transitionGroup);

textView.setVisibility(visible ? View.VISIBLE : View.GONE);
```

效果如下:


{% raw %}
<img src="./autotransition.gif" alt="autotransition" width="200" />
{% endraw %}

###### 源码分析

> 继承自TransitionSet

```java
private void init() {
        setOrdering(ORDERING_SEQUENTIAL);
        addTransition(new Fade(Fade.OUT)).
                addTransition(new ChangeBounds()).
                addTransition(new Fade(Fade.IN));
    }
```

* 顺序播放渐出，区域变化，渐入动画

##### TransitionSet

>继承自Transition

```java
@Override
    public void captureStartValues(TransitionValues transitionValues) {
        if (isValidTarget(transitionValues.view)) {
            for (Transition childTransition : mTransitions) {
                if (childTransition.isValidTarget(transitionValues.view)) {
                    childTransition.captureStartValues(transitionValues);
                    transitionValues.targetedTransitions.add(childTransition);
                }
            }
        }
}

@Override
    public void captureEndValues(TransitionValues transitionValues) {
        if (isValidTarget(transitionValues.view)) {
            for (Transition childTransition : mTransitions) {
                if (childTransition.isValidTarget(transitionValues.view)) {
                    childTransition.captureEndValues(transitionValues);
                    transitionValues.targetedTransitions.add(childTransition);
                }
            }
        }
    }
```

* 调用各个``Transition``的方法

```java
protected void createAnimators(ViewGroup sceneRoot, TransitionValuesMaps startValues,
            TransitionValuesMaps endValues, ArrayList<TransitionValues> startValuesList,
            ArrayList<TransitionValues> endValuesList) {
        long startDelay = getStartDelay();
        int numTransitions = mTransitions.size();
        for (int i = 0; i < numTransitions; i++) {
            Transition childTransition = mTransitions.get(i);
            // We only set the start delay on the first transition if we are playing
            // the transitions sequentially.
            if (startDelay > 0 && (mPlayTogether || i == 0)) {
                long childStartDelay = childTransition.getStartDelay();
                if (childStartDelay > 0) {
                    childTransition.setStartDelay(startDelay + childStartDelay);
                } else {
                    childTransition.setStartDelay(startDelay);
                }
            }
            childTransition.createAnimators(sceneRoot, startValues, endValues, startValuesList,
                    endValuesList);
        }
    }
```

* 使用各自``Transition``创建动画。为第一个转场动画设置必要的延时

```java
@Override
    protected void runAnimators() {
        if (mTransitions.isEmpty()) {
            start();
            end();
            return;
        }
        setupStartEndListeners();
        int numTransitions = mTransitions.size();
        if (!mPlayTogether) {
        //串行播放动画
            // Setup sequence with listeners
            // TODO: Need to add listeners in such a way that we can remove them later if canceled
            for (int i = 1; i < numTransitions; ++i) {
                Transition previousTransition = mTransitions.get(i - 1);
                final Transition nextTransition = mTransitions.get(i);
                previousTransition.addListener(new TransitionListenerAdapter() {
                    @Override
                    public void onTransitionEnd(Transition transition) {
                        nextTransition.runAnimators();
                        transition.removeListener(this);
                    }
                });
            }
            Transition firstTransition = mTransitions.get(0);
            if (firstTransition != null) {
                firstTransition.runAnimators();
            }
        } else {
        //并行播放动画
            for (int i = 0; i < numTransitions; ++i) {
                mTransitions.get(i).runAnimators();
            }
        }
    }
```

* 动画开始，由各个``Transition``开始动画，在这里会区分动画是串行播放，还是并行播放。

##### ChangeText(自定义transition)

{% raw %}
<img src="./changetext.gif" alt="autotransition" width="200" />
{% endraw %}

```java
 textView.animate().alpha(0f).setListener(new AnimatorListenerAdapter() {
                    @Override public void onAnimationEnd(Animator animation) {
                        textView.setText(mSecondText ? TEXT_2 : TEXT_1);
                        textView.animate().setListener(null);
                        textView.animate().alpha(1f).start();
                    }
                }).start();
```
* 上述代码也能实现相同的功能，``ChangeText``只是对其进行了进一步的封装。这里不分析其源码

##### ChangeBound

{% raw %}
<img src="./changebound.gif" alt="autotransition" width="200" />
{% endraw %}

###### 源码分析
>继承``Transition``

```java
private void captureValues(TransitionValues values) {
        View view = values.view;

        if (view.isLaidOut() || view.getWidth() != 0 || view.getHeight() != 0) {
        //view已经布局完毕，保存当前view边界状态
            values.values.put(PROPNAME_BOUNDS, new Rect(view.getLeft(), view.getTop(),
                    view.getRight(), view.getBottom()));
            values.values.put(PROPNAME_PARENT, values.view.getParent());
            if (mReparent) {
            //如果动画scene的parent一样，记录当前屏幕位置
                values.view.getLocationInWindow(tempLocation);
                values.values.put(PROPNAME_WINDOW_X, tempLocation[0]);
                values.values.put(PROPNAME_WINDOW_Y, tempLocation[1]);
            }
            if (mResizeClip) {
                values.values.put(PROPNAME_CLIP, view.getClipBounds());
            }
        }
    }
```

* 保存当前``View``的边界状态

> 创建动画，代码很长，核心代码

```java
if (numChanges > 0) {
                Animator anim;
                if (!mResizeClip) {
                    view.setLeftTopRightBottom(startLeft, startTop, startRight, startBottom);
                    if (numChanges == 2) {
                        if (startWidth == endWidth && startHeight == endHeight) {
                            Path topLeftPath = getPathMotion().getPath(startLeft, startTop, endLeft,
                                    endTop);
                            anim = ObjectAnimator.ofObject(view, POSITION_PROPERTY, null,
                                    topLeftPath);
                        } else {
                            final ViewBounds viewBounds = new ViewBounds(view);
                            Path topLeftPath = getPathMotion().getPath(startLeft, startTop,
                                    endLeft, endTop);
                            ObjectAnimator topLeftAnimator = ObjectAnimator
                                    .ofObject(viewBounds, TOP_LEFT_PROPERTY, null, topLeftPath);

                            Path bottomRightPath = getPathMotion().getPath(startRight, startBottom,
                                    endRight, endBottom);
                            ObjectAnimator bottomRightAnimator = ObjectAnimator.ofObject(viewBounds,
                                    BOTTOM_RIGHT_PROPERTY, null, bottomRightPath);
                            AnimatorSet set = new AnimatorSet();
                            set.playTogether(topLeftAnimator, bottomRightAnimator);
                            anim = set;
                            set.addListener(new AnimatorListenerAdapter() {
                                // We need a strong reference to viewBounds until the
                                // animator ends.
                                private ViewBounds mViewBounds = viewBounds;
                            });
                        }
                    } else if (startLeft != endLeft || startTop != endTop) {
                        Path topLeftPath = getPathMotion().getPath(startLeft, startTop,
                                endLeft, endTop);
                        anim = ObjectAnimator.ofObject(view, TOP_LEFT_ONLY_PROPERTY, null,
                                topLeftPath);
                    } else {
                        Path bottomRight = getPathMotion().getPath(startRight, startBottom,
                                endRight, endBottom);
                        anim = ObjectAnimator.ofObject(view, BOTTOM_RIGHT_ONLY_PROPERTY, null,
                                bottomRight);
                    }
                } else {
                    int maxWidth = Math.max(startWidth, endWidth);
                    int maxHeight = Math.max(startHeight, endHeight);

                    view.setLeftTopRightBottom(startLeft, startTop, startLeft + maxWidth,
                            startTop + maxHeight);

                    ObjectAnimator positionAnimator = null;
                    if (startLeft != endLeft || startTop != endTop) {
                        Path topLeftPath = getPathMotion().getPath(startLeft, startTop, endLeft,
                                endTop);
                        positionAnimator = ObjectAnimator.ofObject(view, POSITION_PROPERTY, null,
                                topLeftPath);
                    }
                    final Rect finalClip = endClip;
                    if (startClip == null) {
                        startClip = new Rect(0, 0, startWidth, startHeight);
                    }
                    if (endClip == null) {
                        endClip = new Rect(0, 0, endWidth, endHeight);
                    }
                    ObjectAnimator clipAnimator = null;
                    if (!startClip.equals(endClip)) {
                        view.setClipBounds(startClip);
                        clipAnimator = ObjectAnimator.ofObject(view, "clipBounds", sRectEvaluator,
                                startClip, endClip);
                        clipAnimator.addListener(new AnimatorListenerAdapter() {
                            private boolean mIsCanceled;

                            @Override
                            public void onAnimationCancel(Animator animation) {
                                mIsCanceled = true;
                            }

                            @Override
                            public void onAnimationEnd(Animator animation) {
                                if (!mIsCanceled) {
                                    view.setClipBounds(finalClip);
                                    view.setLeftTopRightBottom(endLeft, endTop, endRight,
                                            endBottom);
                                }
                            }
                        });
                    }
                    anim = TransitionUtils.mergeAnimators(positionAnimator,
                            clipAnimator);
                }
                if (view.getParent() instanceof ViewGroup) {
                    final ViewGroup parent = (ViewGroup) view.getParent();
                    parent.suppressLayout(true);
                    TransitionListener transitionListener = new TransitionListenerAdapter() {
                        boolean mCanceled = false;

                        @Override
                        public void onTransitionCancel(Transition transition) {
                            parent.suppressLayout(false);
                            mCanceled = true;
                        }

                        @Override
                        public void onTransitionEnd(Transition transition) {
                            if (!mCanceled) {
                                parent.suppressLayout(false);
                            }
                        }

                        @Override
                        public void onTransitionPause(Transition transition) {
                            parent.suppressLayout(false);
                        }

                        @Override
                        public void onTransitionResume(Transition transition) {
                            parent.suppressLayout(true);
                        }
                    };
                    addListener(transitionListener);
                }
                return anim;
            }
        } else {
            sceneRoot.getLocationInWindow(tempLocation);
            int startX = (Integer) startValues.values.get(PROPNAME_WINDOW_X) - tempLocation[0];
            int startY = (Integer) startValues.values.get(PROPNAME_WINDOW_Y) - tempLocation[1];
            int endX = (Integer) endValues.values.get(PROPNAME_WINDOW_X) - tempLocation[0];
            int endY = (Integer) endValues.values.get(PROPNAME_WINDOW_Y) - tempLocation[1];
            // TODO: also handle size changes: check bounds and animate size changes
            if (startX != endX || startY != endY) {
                final int width = view.getWidth();
                final int height = view.getHeight();
                Bitmap bitmap = Bitmap.createBitmap(width, height, Bitmap.Config.ARGB_8888);
                Canvas canvas = new Canvas(bitmap);
                view.draw(canvas);
                final BitmapDrawable drawable = new BitmapDrawable(bitmap);
                drawable.setBounds(startX, startY, startX + width, startY + height);
                final float transitionAlpha = view.getTransitionAlpha();
                view.setTransitionAlpha(0);
                sceneRoot.getOverlay().add(drawable);
                Path topLeftPath = getPathMotion().getPath(startX, startY, endX, endY);
                PropertyValuesHolder origin = PropertyValuesHolder.ofObject(
                        DRAWABLE_ORIGIN_PROPERTY, null, topLeftPath);
                ObjectAnimator anim = ObjectAnimator.ofPropertyValuesHolder(drawable, origin);
                anim.addListener(new AnimatorListenerAdapter() {
                    @Override
                    public void onAnimationEnd(Animator animation) {
                        sceneRoot.getOverlay().remove(drawable);
                        view.setTransitionAlpha(transitionAlpha);
                    }
                });
                return anim;
            }
        }
```

* 主要功能是判断``View``的边界有没有发生改变，如果发生改变，使用对应的位置变化动画，主要包括``topLeft, bottomRight, position``。如果没有发生改变，则在``View``上绘制``Overlay``并对其做``path``动画。

设置``pathMotion``

>其是一个抽象类，用于描述``Transition``的``Path``信息


{% raw %}
<img src="./changebound_pathmotion.gif" alt="autotransition" width="200" />
{% endraw %}

```java
public abstract class PathMotion {

    public PathMotion() {}

    public PathMotion(Context context, AttributeSet attrs) {}

    /**
     * Provide a Path to interpolate between two points <code>(startX, startY)</code> and
     * <code>(endX, endY)</code>. This allows controlled curved motion along two dimensions.
     *
     * @param startX The x coordinate of the starting point.
     * @param startY The y coordinate of the starting point.
     * @param endX   The x coordinate of the ending point.
     * @param endY   The y coordinate of the ending point.
     * @return A Path along which the points should be interpolated. The returned Path
     * must start at point <code>(startX, startY)</code>, typically using
     * {@link android.graphics.Path#moveTo(float, float)} and end at <code>(endX, endY)</code>.
     */
    public abstract Path getPath(float startX, float startY, float endX, float endY);
}
```

* 实现``getPath``以便提供两点之间的变化频率

##### Slide

>继承自``Visibility``

{% raw %}
<img src="./slide.gif" alt="autotransition" width="200" />
{% endraw %}

###### 源码分析

```java
private void captureValues(TransitionValues transitionValues) {
        View view = transitionValues.view;
        int[] position = new int[2];
        view.getLocationOnScreen(position);
        transitionValues.values.put(PROPNAME_SCREEN_POSITION, position);
    }
```

* 保存当前``View``位置

> 其``createAnimator``在父类``Visibility``中被重写，当中调用``onAppear``和``onDisappear``，这两个方法分别有各自的重载方法，由子类提供具体的实现，默认无动画

```java
@Override
    public Animator onAppear(ViewGroup sceneRoot, View view,
            TransitionValues startValues, TransitionValues endValues) {
        if (endValues == null) {
            return null;
        }
        int[] position = (int[]) endValues.values.get(PROPNAME_SCREEN_POSITION);
        float endX = view.getTranslationX();
        float endY = view.getTranslationY();
        float startX = mSlideCalculator.getGoneX(sceneRoot, view);
        float startY = mSlideCalculator.getGoneY(sceneRoot, view);
        return TranslationAnimationCreator
                .createAnimation(view, endValues, position[0], position[1],
                        startX, startY, endX, endY, sDecelerate, this);
    }
```

* 使用``CalculateSlide``接口的实现计算``View``离开或者进入场景时的位置，最后由``TranslationAnimationCreator``创建动画

##### Scale

{% raw %}
<img src="./scale.gif" alt="autotransition" width="200" />
{% endraw %}

设置``alpha``

{% raw %}
<img src="./scale_alpha.gif" alt="autotransition" width="200" />
{% endraw %}

###### 源码分析

private Animator createAnimation(final View view, float startScale, float endScale, TransitionValues values) {
        final float initialScaleX = view.getScaleX();
        final float initialScaleY = view.getScaleY();
        float startScaleX = initialScaleX * startScale;
        float endScaleX = initialScaleX * endScale;
        float startScaleY = initialScaleY * startScale;
        float endScaleY = initialScaleY * endScale;

        if (values != null) {
            Float savedScaleX = (Float) values.values.get(PROPNAME_SCALE_X);
            Float savedScaleY = (Float) values.values.get(PROPNAME_SCALE_Y);
            // if saved value is not equal initial value it means that previous
            // transition was interrupted and in the onTransitionEnd
            // we've applied endScale. we should apply proper value to
            // continue animation from the interrupted state
            if (savedScaleX != null && savedScaleX != initialScaleX) {
                startScaleX = savedScaleX;
            }
            if (savedScaleY != null && savedScaleY != initialScaleY) {
                startScaleY = savedScaleY;
            }
        }

        view.setScaleX(startScaleX);
        view.setScaleY(startScaleY);

        Animator animator = TransitionUtils.mergeAnimators(
                ObjectAnimator.ofFloat(view, View.SCALE_X, startScaleX, endScaleX),
                ObjectAnimator.ofFloat(view, View.SCALE_Y, startScaleY, endScaleY));
        addListener(new TransitionListenerAdapter() {
            @Override
            public void onTransitionEnd(Transition transition) {
                view.setScaleX(initialScaleX);
                view.setScaleY(initialScaleY);
            }
        });
        return animator;
}

* 合并``x, y``缩放的动画，合并是采用``TransitionSet``，如下

```java
public static Transition mergeTransitions(Transition... transitions) {
        int count = 0;
        int nonNullIndex = -1;
        for (int i = 0; i < transitions.length; i++) {
            if (transitions[i] != null) {
                count++;
                nonNullIndex = i;
            }
        }

        if (count == 0) {
            return null;
        }

        if (count == 1) {
            return transitions[nonNullIndex];
        }

        TransitionSet transitionSet = new TransitionSet();
        for (int i = 0; i < transitions.length; i++) {
            if (transitions[i] != null) {
                transitionSet.addTransition(transitions[i]);
            }
        }
        return transitionSet;
    }
```

##### Explode

>继承自Visibility

{% raw %}
<img src="./explode.gif" alt="autotransition" width="200" />
{% endraw %}

###### 源码分析

```java
@Override
    public Animator onAppear(ViewGroup sceneRoot, View view,
            TransitionValues startValues, TransitionValues endValues) {
        if (endValues == null) {
            return null;
        }
        Rect bounds = (Rect) endValues.values.get(PROPNAME_SCREEN_BOUNDS);
        float endX = view.getTranslationX();
        float endY = view.getTranslationY();
        calculateOut(sceneRoot, bounds, mTempLoc);
        float startX = endX + mTempLoc[0];
        float startY = endY + mTempLoc[1];

        return TranslationAnimationCreator.createAnimation(view, endValues, bounds.left, bounds.top,
                startX, startY, endX, endY, sDecelerate, this);
    }
```

* 通过``calculateOut``计算扩散的距离，使用``TranslationAnimationCreator``创建动画

```java
private void calculateOut(View sceneRoot, Rect bounds, int[] outVector) {
        sceneRoot.getLocationOnScreen(mTempLoc);
        int sceneRootX = mTempLoc[0];
        int sceneRootY = mTempLoc[1];
        int focalX;
        int focalY;

        Rect epicenter = getEpicenter();
        if (epicenter == null) {
            focalX = sceneRootX + (sceneRoot.getWidth() / 2)
                    + Math.round(sceneRoot.getTranslationX());
            focalY = sceneRootY + (sceneRoot.getHeight() / 2)
                    + Math.round(sceneRoot.getTranslationY());
        } else {
            focalX = epicenter.centerX();
            focalY = epicenter.centerY();
        }

        int centerX = bounds.centerX();
        int centerY = bounds.centerY();
        double xVector = centerX - focalX;
        double yVector = centerY - focalY;

        if (xVector == 0 && yVector == 0) {
            // Random direction when View is centered on focal View.
            xVector = (Math.random() * 2) - 1;
            yVector = (Math.random() * 2) - 1;
        }
        double vectorSize = Math.hypot(xVector, yVector);
        xVector /= vectorSize;
        yVector /= vectorSize;

        double maxDistance =
                calculateMaxDistance(sceneRoot, focalX - sceneRootX, focalY - sceneRootY);

        outVector[0] = (int) Math.round(maxDistance * xVector);
        outVector[1] = (int) Math.round(maxDistance * yVector);
    }
```

* 计算场景动画的偏移量，如果为0，随机算出偏移量，算出x, y偏移量的平方根，求出最大距离
##### TransitionName

> 设置要做动画的标识，在之后的动画中会对这些标识的``View``做动画

{% raw %}
<img src="./transition_name.gif" alt="autotransition" width="200" />
{% endraw %}

##### ImageTransform

{% raw %}
<img src="./ImageTransform.gif" alt="autotransition" width="200" />
{% endraw %}

##### ReColor

{% raw %}
<img src="./recolor.gif" alt="autotransition" width="200" />
{% endraw %}

##### Rotate
>继承自Transition的旋转动画

{% raw %}
<img src="./rotate.gif" alt="autotransition" width="200" />
{% endraw %}

###### 源码分析

```java
public Animator createAnimator(ViewGroup sceneRoot, TransitionValues startValues,
            TransitionValues endValues) {
        if (startValues == null || endValues == null) {
            return null;
        }
        final View view = endValues.view;
        float startRotation = (Float) startValues.values.get(PROPNAME_ROTATION);
        float endRotation = (Float) endValues.values.get(PROPNAME_ROTATION);
        if (startRotation != endRotation) {
            view.setRotation(startRotation);
            return ObjectAnimator.ofFloat(view, View.ROTATION,
                    startRotation, endRotation);
        }
        return null;
    }
```

* 创建旋转动画

##### Progress

{% raw %}
<img src="./progress.gif" alt="autotransition" width="200" />
{% endraw %}

##### Change Scene

{% raw %}
<img src="./change_scene.gif" alt="autotransition" width="200" />
{% endraw %}


###### 源码分析

```java
private void init() {
        setOrdering(ORDERING_SEQUENTIAL);
        addTransition(new Fade(Fade.OUT)).
                addTransition(new ChangeBounds()).
                addTransition(new Fade(Fade.IN));
}
```

* 先淡出效果(``Fade``)，改变区域(``ChangeBound``)，淡入效果(``Fade``)

##### Fade
渐变转场动画，继承自``Visibility``

```java
public void captureStartValues(TransitionValues transitionValues) {
        super.captureStartValues(transitionValues);
        transitionValues.values.put(PROPNAME_TRANSITION_ALPHA,
                transitionValues.view.getTransitionAlpha());
}
```
* 捕获初始场景的``alpha``值

```java
private Animator createAnimation(final View view, float startAlpha, final float endAlpha) {
        if (startAlpha == endAlpha) {
            return null;
        }
        view.setTransitionAlpha(startAlpha);
        //创建渐变属性动画
        final ObjectAnimator anim = ObjectAnimator.ofFloat(view, "transitionAlpha", endAlpha);
        if (DBG) {
            Log.d(LOG_TAG, "Created animator " + anim);
        }
        final FadeAnimatorListener listener = new FadeAnimatorListener(view);
        anim.addListener(listener);
        anim.addPauseListener(listener);
        addListener(new TransitionListenerAdapter() {
            @Override
            public void onTransitionEnd(Transition transition) {
                view.setTransitionAlpha(1);
            }
        });
        return anim;
    }
```
* 创建渐变属性动画，其会由``onAppear()``和``onDisAppear``调用

```java
@Override
    public Animator onAppear(ViewGroup sceneRoot, View view,
            TransitionValues startValues,
            TransitionValues endValues) {
        if (DBG) {
            View startView = (startValues != null) ? startValues.view : null;
            Log.d(LOG_TAG, "Fade.onAppear: startView, startVis, endView, endVis = " +
                    startView + ", " + view);
        }
        //创建渐入动画
        return createAnimation(view, 0, 1);
    }

    @Override
    public Animator onDisappear(ViewGroup sceneRoot, final View view, TransitionValues startValues,
            TransitionValues endValues) {
            //创建渐出动画
        return createAnimation(view, 1, 0);
    }
```

* 这两个方法由``Visibility``的``createAnimator``调用

```java
@Override
    public Animator createAnimator(ViewGroup sceneRoot, TransitionValues startValues,
            TransitionValues endValues) {
        VisibilityInfo visInfo = getVisibilityChangeInfo(startValues, endValues);
        if (visInfo.visibilityChange
                && (visInfo.startParent != null || visInfo.endParent != null)) {
            if (visInfo.fadeIn) {
                return onAppear(sceneRoot, startValues, visInfo.startVisibility,
                        endValues, visInfo.endVisibility);
            } else {
                return onDisappear(sceneRoot, startValues, visInfo.startVisibility,
                        endValues, visInfo.endVisibility
                );
            }
        }
        return null;
    }
```

* ``View``显示或隐藏时会创建对应的渐变动画。``Transition``的``playTransition``会调用``createAnimators``，其内部会调用``createAnimator``。
* 整体流程是启动``Transition``->创建动画(``createAnimators``)->创建单个动画(``createAnimator``由``Transition``子类实现,默认空)->``Visibility``的``onAppear()``/``onDisappear``->子类``createAnimator``


### 总结:
> ``Transition``动画其实是对属性动画的一种高级封装。其主要流程是``TransitionManager``调用``beginDelayedTransition``,内部监听了``OnPreDrawListener``，其在``onPreDraw``调用``captureView``用于获取当前``View``的状态,调用``playTransition``开始转场动画。之后创建动画``createAnimator``。这其中``captureView``都是由子类实现，而``createAnimator``则可以由子类选择重写。最终动画的开始还有调用``animator.start()``