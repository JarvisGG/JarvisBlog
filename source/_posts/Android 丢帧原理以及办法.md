---
title: Android 丢帧原理以及办法
date: 2017-01-22 17:53:56
tags: Android 源码
comments: true
---


## 接近年底，想分享点儿东西给大家。

## Android UI绘制过程

开发中的卡顿我想没跟人都遇到过，之前也是搜博客看看怎么个解决办法，没有认真研究过，今天我打算跟大家聊一聊。

先从View 说吧。相信大家应该都知道View的绘制过程，measure，layout，draw。丢帧一定是在16ms内没有把这些事儿干完就对了，这里我们简单的分一下，主要是计算时间，以及绘图时间。

计算时间：这里的measure，layout的过程，都是会向下递归计算的，学过数据结构的话，应该知道，深搜的代价是很大的。所以尽量让树的高度降低，这里就引出扁平化布局。

绘图时间：这里需要着重讲一下，因为有时候这才是我们UI卡顿的主要原因。在这里我们要把android的试图看成是三维的，就像photoshop的图层一样。android在绘制的时候就会一层一层的“粉刷”，好了，那么造成卡顿，也就是丢帧，说白了最后没有在16ms内做完。好了，让我们剖析一下：

### invalidate()：

我们知道invalidate 是用来请求View 重绘的

``` Java
// Propagate the damage rectangle to the parent view.
final AttachInfo ai = mAttachInfo;
final ViewParent p = mParent;
if (p != null && ai != null && l < r && t < b) {
    final Rect damage = ai.mTmpInvalRect;
    damage.set(l, t, r, b);
    p.invalidateChild(this, damage);
}
```

invalidateInternal
这里可以看出来draw的过程其实就是拿到AttachInfo 里面包含着绘制信息，以及将绘制区域拿到，通过parent去绘制。让我们跟进去。

``` Java
public ViewParent invalidateChildInParent(final int[] location, final Rect dirty) {
    if ((mPrivateFlags & PFLAG_DRAWN) == PFLAG_DRAWN ||
            (mPrivateFlags & PFLAG_DRAWING_CACHE_VALID) == PFLAG_DRAWING_CACHE_VALID) {
        if ((mGroupFlags & (FLAG_OPTIMIZE_INVALIDATE | FLAG_ANIMATION_DONE)) !=
                    FLAG_OPTIMIZE_INVALIDATE) {
            dirty.offset(location[CHILD_LEFT_INDEX] - mScrollX,
                    location[CHILD_TOP_INDEX] - mScrollY);
            if ((mGroupFlags & FLAG_CLIP_CHILDREN) == 0) {
                dirty.union(0, 0, mRight - mLeft, mBottom - mTop);
            }

            final int left = mLeft;
            final int top = mTop;

            if ((mGroupFlags & FLAG_CLIP_CHILDREN) == FLAG_CLIP_CHILDREN) {
                if (!dirty.intersect(0, 0, mRight - left, mBottom - top)) {
                    dirty.setEmpty();
                }
            }
            mPrivateFlags &= ~PFLAG_DRAWING_CACHE_VALID;

            location[CHILD_LEFT_INDEX] = left;
            location[CHILD_TOP_INDEX] = top;

            if (mLayerType != LAYER_TYPE_NONE) {
                mPrivateFlags |= PFLAG_INVALIDATED;
            }

            return mParent;

        } else {
            mPrivateFlags &= ~PFLAG_DRAWN & ~PFLAG_DRAWING_CACHE_VALID;

            location[CHILD_LEFT_INDEX] = mLeft;
            location[CHILD_TOP_INDEX] = mTop;
            if ((mGroupFlags & FLAG_CLIP_CHILDREN) == FLAG_CLIP_CHILDREN) {
                dirty.set(0, 0, mRight - mLeft, mBottom - mTop);
            } else {
                // in case the dirty rect extends outside the bounds of this container
                dirty.union(0, 0, mRight - mLeft, mBottom - mTop);
            }

            if (mLayerType != LAYER_TYPE_NONE) {
                mPrivateFlags |= PFLAG_INVALIDATED;
            }

            return mParent;
        }
    }

    return null;
}
```
invalidateChildInParent
这里的dirty代表你绘制的这块区域是否透明。

``` Java
void invalidate() {
    mDirty.set(0, 0, mWidth, mHeight);
    if (!mWillDrawSoon) {
        scheduleTraversals();
    }
}
```
invalidate
这里我们看到了个关键函数 scheduleTraversals ，为什么说神奇。我们看一下

``` Java
void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
        mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        if (!mUnbufferedInputDispatch) {
            scheduleConsumeBatchedInput();
        }
        notifyRendererOfFramePending();
        pokeDrawLockIfNeeded();
    }
}
```
scheduleTraversals
这里最重要的是Choreographer 这个，我们最终算出来的绘制信息都要通过它回调，开始他会注册一个广播用来接收时钟信息，然后他会在内部建立一个UI绘制队列：CallbackQueue，我们在外部CallBack的时候，会将我们的绘制信息作为CallbackRecord 然后会在接收到一个时钟信号的时候进行doFrame操作，并打印Traces信息，从而来绘制一帧。

``` Java
private static final class CallbackRecord {
    public CallbackRecord next;
    public long dueTime;
    public Object action; // Runnable or FrameCallback
    public Object token;

    public void run(long frameTimeNanos) {
        if (token == FRAME_CALLBACK_TOKEN) {
            ((FrameCallback)action).doFrame(frameTimeNanos);
        } else {
            ((Runnable)action).run();
        }
    }
}
private final class CallbackQueue {
    private CallbackRecord mHead;

    public boolean hasDueCallbacksLocked(long now) {
        return mHead != null && mHead.dueTime <= now;
    }

    public CallbackRecord extractDueCallbacksLocked(long now) {
        CallbackRecord callbacks = mHead;
        if (callbacks == null || callbacks.dueTime > now) {
            return null;
        }

        CallbackRecord last = callbacks;
        CallbackRecord next = last.next;
        while (next != null) {
            if (next.dueTime > now) {
                last.next = null;
                break;
            }
            last = next;
            next = next.next;
        }
        mHead = next;
        return callbacks;
    }

    public void addCallbackLocked(long dueTime, Object action, Object token) {
        CallbackRecord callback = obtainCallbackLocked(dueTime, action, token);
        CallbackRecord entry = mHead;
        if (entry == null) {
            mHead = callback;
            return;
        }
        if (dueTime < entry.dueTime) {
            callback.next = entry;
            mHead = callback;
            return;
        }
        while (entry.next != null) {
            if (dueTime < entry.next.dueTime) {
                callback.next = entry.next;
                break;
            }
            entry = entry.next;
        }
        entry.next = callback;
    }

    public void removeCallbacksLocked(Object action, Object token) {
        CallbackRecord predecessor = null;
        for (CallbackRecord callback = mHead; callback != null;) {
            final CallbackRecord next = callback.next;
            if ((action == null || callback.action == action)
                    && (token == null || callback.token == token)) {
                if (predecessor != null) {
                    predecessor.next = next;
                } else {
                    mHead = next;
                }
                recycleCallbackLocked(callback);
            } else {
                predecessor = callback;
            }
            callback = next;
        }
    }
}
```
CallbackQueue and CallbackRecord

``` Java
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
postCallbackDelayedInternal
可以看到这里我们把我们的绘制内容扔到队列里，等待轮训。

``` Java
private final class FrameDisplayEventReceiver extends DisplayEventReceiver
            implements Runnable {
    private boolean mHavePendingVsync;
    private long mTimestampNanos;
    private int mFrame;

    public FrameDisplayEventReceiver(Looper looper) {
        super(looper);
    }

    @Override
    public void onVsync(long timestampNanos, int builtInDisplayId, int frame) {
        // Ignore vsync from secondary display.
        // This can be problematic because the call to scheduleVsync() is a one-shot.
        // We need to ensure that we will still receive the vsync from the primary
        // display which is the one we really care about.  Ideally we should schedule
        // vsync for a particular display.
        // At this time Surface Flinger won't send us vsyncs for secondary displays
        // but that could change in the future so let's log a message to help us remember
        // that we need to fix this.
        if (builtInDisplayId != SurfaceControl.BUILT_IN_DISPLAY_ID_MAIN) {
            Log.d(TAG, "Received vsync from secondary display, but we don't support "
                    + "this case yet.  Choreographer needs a way to explicitly request "
                    + "vsync for a specific display to ensure it doesn't lose track "
                    + "of its scheduled vsync.");
            scheduleVsync();
            return;
        }

        // Post the vsync event to the Handler.
        // The idea is to prevent incoming vsync events from completely starving
        // the message queue.  If there are no messages in the queue with timestamps
        // earlier than the frame time, then the vsync event will be processed immediately.
        // Otherwise, messages that predate the vsync event will be handled first.
        long now = System.nanoTime();
        if (timestampNanos > now) {
            Log.w(TAG, "Frame time is " + ((timestampNanos - now) * 0.000001f)
                    + " ms in the future!  Check that graphics HAL is generating vsync "
                    + "timestamps using the correct timebase.");
            timestampNanos = now;
        }

        if (mHavePendingVsync) {
            Log.w(TAG, "Already have a pending vsync event.  There should only be "
                    + "one at a time.");
        } else {
            mHavePendingVsync = true;
        }

        mTimestampNanos = timestampNanos;
        mFrame = frame;
        Message msg = Message.obtain(mHandler, this);
        msg.setAsynchronous(true);
        mHandler.sendMessageAtTime(msg, timestampNanos / TimeUtils.NANOS_PER_MS);
    }

    @Override
    public void run() {
        mHavePendingVsync = false;
        doFrame(mTimestampNanos, mFrame);
    }
}
```
FrameDisplayEventReceiver
接收时钟脉冲信号的广播，16ms一次，我们的目的就是在这个时钟脉冲里搞定整个 view

### Android 动画

Animator，ScrollTo，offsetLeftAndRight，这里面我们先单列这几项，都是同一个原理。这里我们可以大胆的猜想，一定是频繁执行我们的 Choreographer.CallBack 来绘制，因为只要在16ms内绘制成功，那就是流畅的动画。下面我们验证一下

ScrollTo:

我们先看一下 View 中这个方法

``` Java
public void scrollTo(int x, int y) {
    if (mScrollX != x || mScrollY != y) {
        int oldX = mScrollX;
        int oldY = mScrollY;
        mScrollX = x;
        mScrollY = y;
        invalidateParentCaches();
        onScrollChanged(mScrollX, mScrollY, oldX, oldY);
        if (!awakenScrollBars()) {
            postInvalidateOnAnimation();
        }
    }
}
```
scrollTo
很简单，我们都可以看懂，开始位置，结束位置，这里我们重点关注 postInvalidateOnAnimation()  这个方法

``` Java
public void postInvalidateOnAnimation() {
    // We try only with the AttachInfo because there's no point in invalidating
    // if we are not attached to our window
    final AttachInfo attachInfo = mAttachInfo;
    if (attachInfo != null) {
        attachInfo.mViewRootImpl.dispatchInvalidateOnAnimation(this);
    }
}
```
postInvalidateOnAnimation
我们可以看到，这里的动画过程绘制他还是扔到了ViewRootImpl 代理做这件事。

``` Java
public void dispatchInvalidateRectOnAnimation(AttachInfo.InvalidateInfo info) {
    mInvalidateOnAnimationRunnable.addViewRect(info);
}
```
dispatchInvalidateOnAnimation
这里我们看到他开了个线程 mInvalidateOnAnimationRunnable 去添加我们这个将要绘制的 view，接下来我们继续庖丁解牛

``` Java
final class InvalidateOnAnimationRunnable implements Runnable {
    private boolean mPosted;
    private final ArrayList<View> mViews = new ArrayList<View>();
    private final ArrayList<AttachInfo.InvalidateInfo> mViewRects =
            new ArrayList<AttachInfo.InvalidateInfo>();
    private View[] mTempViews;
    private AttachInfo.InvalidateInfo[] mTempViewRects;

    public void addView(View view) {
        synchronized (this) {
            mViews.add(view);
            postIfNeededLocked();
        }
    }

    public void addViewRect(AttachInfo.InvalidateInfo info) {
        synchronized (this) {
            mViewRects.add(info);
            postIfNeededLocked();
        }
    }

    public void removeView(View view) {
        synchronized (this) {
            mViews.remove(view);

            for (int i = mViewRects.size(); i-- > 0; ) {
                AttachInfo.InvalidateInfo info = mViewRects.get(i);
                if (info.target == view) {
                    mViewRects.remove(i);
                    info.recycle();
                }
            }

            if (mPosted && mViews.isEmpty() && mViewRects.isEmpty()) {
                mChoreographer.removeCallbacks(Choreographer.CALLBACK_ANIMATION, this, null);
                mPosted = false;
            }
        }
    }

    @Override
    public void run() {
        final int viewCount;
        final int viewRectCount;
        synchronized (this) {
            mPosted = false;

            viewCount = mViews.size();
            if (viewCount != 0) {
                mTempViews = mViews.toArray(mTempViews != null
                        ? mTempViews : new View[viewCount]);
                mViews.clear();
            }

            viewRectCount = mViewRects.size();
            if (viewRectCount != 0) {
                mTempViewRects = mViewRects.toArray(mTempViewRects != null
                        ? mTempViewRects : new AttachInfo.InvalidateInfo[viewRectCount]);
                mViewRects.clear();
            }
        }

        for (int i = 0; i < viewCount; i++) {
            mTempViews[i].invalidate();
            mTempViews[i] = null;
        }

        for (int i = 0; i < viewRectCount; i++) {
            final View.AttachInfo.InvalidateInfo info = mTempViewRects[i];
            info.target.invalidate(info.left, info.top, info.right, info.bottom);
            info.recycle();
        }
    }

    private void postIfNeededLocked() {
        if (!mPosted) {
            mChoreographer.postCallback(Choreographer.CALLBACK_ANIMATION, this, null);
            mPosted = true;
        }
    }
}
```
InvalidateOnAnimationRunnable
终于，应了我们的猜想，ViewRootImpl 有一个专门执行动画绘制操作的线程，我们可以看到 run() 里面不断地CallBack，然后回收，当然里面有些线程锁啥的不涉及本文就不细说了。

### ValueAnimator：

这里我们有个 AnimationHandler 来执行动画操作，这其中我们可以看到

``` Java
for (int i = 0; i < numAnims; ++i) {
    ValueAnimator anim = mTmpAnimations.get(i);
    if (mAnimations.contains(anim) && anim.doAnimationFrame(frameTime)) {
        mEndingAnims.add(anim);
    }
}
mTmpAnimations.clear();
```
doAnimationFrame
这里在不断循环我们所有的anim，并在不断执行 scheduleAnimation 方法

``` Java
private void scheduleAnimation() {
    if (!mAnimationScheduled) {
        mChoreographer.postCallback(Choreographer.CALLBACK_ANIMATION, mAnimate, null);
        mAnimationScheduled = true;
    }
}
```
scheduleAnimation
剩下的大家自己翻阅源码把。

这里总结一下。我们所有界面上视图的变化都是都是 ViewRootImpl 把需要重绘的东西填充 Choreographer 中的 mCallbackQueues 队列，然后在时钟脉冲的广播下进行轮训执行。

既然提到队列，假如我们在16ms内大量的填充 AttachInfo 之类的绘制OBJ，就会导致无法再一次时钟脉冲内绘制完毕，就会在造成丢帧，UI阻塞。

避免 Android UI 卡顿解决办法

解决办法：分析了好多，这里说两个方法。

1.避免重绘，这里避免图层（View）迭代。这里我们可以去开发者模式中对“显示GPU视图更新”打钩

![Alt text](http://upload-images.jianshu.io/upload_images/2710533-cade3554a7db68ee.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

过度绘制

![Alt text](http://upload-images.jianshu.io/upload_images/2710533-eb07b0f9a442d6f3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

优化以后
这里引用 http://hukai.me/android-performance-render 这篇博客的作者，盗个图。😂

这里可以进行，选择制定画布绘制，而不是整个view去绘制。可以在onDraw中进行限制，去限制绘制区域，例如

canvas.clipRect(100,100,350,600, Region.Op.INTERSECT);

2.扁平化布局，归根结底也是减少 mCallbackQueues 队列大小。保证尽量在16ms内绘制完毕，再有就是可以减少视图 ViewTree 的高度，减少时间复杂度，从而优化计算过程


![Alt text](http://upload-images.jianshu.io/upload_images/2710533-4c8e729db92f694a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

xml代码


![Alt text](http://upload-images.jianshu.io/upload_images/2710533-0dbcc3353bd36d2f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

优化后的xml代码
＊附：


![Alt text](http://upload-images.jianshu.io/upload_images/2710533-5ddc05554c24d6d9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


绘制层级
通过打开刚才说的开发者选项，来根据颜色来判断页面绘制情况。

距离回家还有8 个小时，17年希望可以发觉更多的东西给大家，并且希望大家可以积极指出文章中的错误。祝大家新年快乐！😄