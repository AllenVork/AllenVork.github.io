---
layout:     post
title:      android 屏幕刷新
subtitle:   
header-img: img/android3.png
author:     Allen Vork
catalog: true
tags:
    - android basics  
---

## Synopsis
该类是用于协调动画、输入和绘图的时间。它接收来自显示子系统的时间脉冲（如垂直同步信号），然后安排下一帧的渲染工作。应用程序一般不直接与它进行交互，而是在动画框架或者 view 层级中使用更高级的抽象来进行操作。

## constructor
```java
    private Choreographer(Looper looper, int vsyncSource) {
        // 将当前线程的 looper 给 Handler
        mLooper = looper;
        mHandler = new FrameHandler(looper);

        mDisplayEventReceiver = USE_VSYNC
                ? new FrameDisplayEventReceiver(looper, vsyncSource)
                : null;
        mLastFrameTimeNanos = Long.MIN_VALUE;

        // 即：1s/60f = 16.7ms每帧
        mFrameIntervalNanos = (long)(1000000000 / getRefreshRate());

        // 创建一个容量为4的数组，用于存放4种不同的 callback
        mCallbackQueues = new CallbackQueue[CALLBACK_LAST + 1];
        for (int i = 0; i <= CALLBACK_LAST; i++) {
            mCallbackQueues[i] = new CallbackQueue();
        }
    }
```
可以看出它是创建了一个 FrameHandler，然后计算了下系统刷新频率，最后创建了一个容量为4的数组来存放4种 Callback。    

这个构造函数是私有的，我们来看下如何获取它的实例：
```java
    /**
     * Gets the choreographer for the calling thread.  Must be called from
     * a thread that already has a {@link android.os.Looper} associated with it.
     *
     * @return The choreographer for this thread.
     * @throws IllegalStateException if the thread does not have a looper.
     */
    public static Choreographer getInstance() {
        return sThreadInstance.get();
    }

    // Thread local storage for the choreographer.
    private static final ThreadLocal<Choreographer> sThreadInstance =
            new ThreadLocal<Choreographer>() {
        @Override
        protected Choreographer initialValue() {
            Looper looper = Looper.myLooper();
            if (looper == null) {
                throw new IllegalStateException("The current thread must have a looper!");
            }
            return new Choreographer(looper, VSYNC_SOURCE_APP);
        }
    };
```
可以看出它的实例是由 ThreadLocal 创建的，这样每个线程都有自己独立的 Choreographer。

我们来看下4种 CallbackQueue:
```java
   /**
     * Must be kept in sync with CALLBACK_* ints below, used to index into this array.
     * @hide
     */
    private static final String[] CALLBACK_TRACE_TITLES = {
            "input", "animation", "traversal", "commit"
    };

    /**
     * Callback type: Input callback.  Runs first.
     * @hide
     */
    public static final int CALLBACK_INPUT = 0;

    /**
     * Callback type: Animation callback.  Runs before traversals.
     * @hide
     */
    public static final int CALLBACK_ANIMATION = 1;

    /**
     * Callback type: Traversal callback.  Handles layout and draw.  Runs
     * after all other asynchronous messages have been handled.
     * @hide
     */
    public static final int CALLBACK_TRAVERSAL = 2;

    /**
     * Callback type: Commit callback.  Handles post-draw operations for the frame.
     * Runs after traversal completes.  The {@link #getFrameTime() frame time} reported
     * during this callback may be updated to reflect delays that occurred while
     * traversals were in progress in case heavy layout operations caused some frames
     * to be skipped.  The frame time reported during this callback provides a better
     * estimate of the start time of the frame in which animations (and other updates
     * to the view hierarchy state) actually took effect.
     * @hide
     */
    public static final int CALLBACK_COMMIT = 3;
```
可以看出这4种 Callback 分别是:    
+ `输入回调`，它是第一个执行的
+ `动画回调`，在布局绘制回调之前执行
+ `布局绘制回调`，用于处理 layout 和 draw，它是在所有其余异步消息处理完成后才会执行
+ `提交回调`，处理 draw() 方法执行完成后的操作。它是在布局绘制回调执行完后执行。

我们来看下 CallbackQueue，它是个内部类：
```java
    private final class CallbackQueue {
        // 链表头部
        private CallbackRecord mHead;

        public boolean hasDueCallbacksLocked(long now) {
            return mHead != null && mHead.dueTime <= now;
        }

        // 返回当前链表中从 dueTime <= now 的链表的 CallbackRecord 的头部
        public CallbackRecord extractDueCallbacksLocked(long now) {
            CallbackRecord callbacks = mHead;
            if (callbacks == null || callbacks.dueTime > now) {
                return null;
            }

            CallbackRecord last = callbacks;
            CallbackRecord next = last.next;
            while (next != null) {
                if (next.dueTime > now) { // 将链表从 last 位置断开
                    last.next = null;
                    break;
                }
                last = next;
                next = next.next;
            }
            // 将断开后的链表的前面一部分的头节点返回，断开的后面一部分的头节点作为 head
            mHead = next; 
            return callbacks;
        }

        // 往链表中添加结点
        public void addCallbackLocked(long dueTime, Object action, Object token) {
            CallbackRecord callback = obtainCallbackLocked(dueTime, action, token);
            CallbackRecord entry = mHead;
            if (entry == null) {
                mHead = callback;
                return;
            }
            // 开始时间小于头结点的开始时间的话，加入的这个结点就是新的头节点
            if (dueTime < entry.dueTime) {
                callback.next = entry;
                mHead = callback;
                return;
            }
            // 遍历链表，将结点按照 dueTime 的大小插入到合适位置
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
可以看出这个类是个链表，提供了添加，删除，读取操作。
```java
    private static final class CallbackRecord {
        public CallbackRecord next; // 下一个结点
        public long dueTime; // action 执行的时间
        public Object action; // Runnable or FrameCallback
        public Object token; // 标志 action 为 Runnable 还是 FrameCallback

        public void run(long frameTimeNanos) {
            if (token == FRAME_CALLBACK_TOKEN) {
                ((FrameCallback)action).doFrame(frameTimeNanos);
            } else { 
                ((Runnable)action).run();
            }
        }
    }
```
链表的结点的结构比较简单，注释里讲的比较清楚。我们来看另一个内部类：    
```java
    private final class FrameDisplayEventReceiver extends DisplayEventReceiver
            implements Runnable {
        private boolean mHavePendingVsync;
        private long mTimestampNanos;
        private int mFrame;

        public FrameDisplayEventReceiver(Looper looper, int vsyncSource) {
            super(looper, vsyncSource);
        }

        // 接收到 VSYNC 信号时调用，timestampNanos 表示接收到信号的时间，frame 表示帧数（每接收到一次信号就会增加）
        @Override
        public void onVsync(long timestampNanos, int builtInDisplayId, int frame) {
            // ...

            mTimestampNanos = timestampNanos;
            mFrame = frame;

            // FrameDisplayEventReceiver 本身就是一个 Runnalble, 将自己封装到 Message 中去
            Message msg = Message.obtain(mHandler, this);
            // 发送异步消息
            msg.setAsynchronous(true);
            mHandler.sendMessageAtTime(msg, timestampNanos / TimeUtils.NANOS_PER_MS);
        }

        @Override
        public void run() {
            // 该变量标志同一时刻只能有一个 VSYNC 信号事件
            mHavePendingVsync = false;
            doFrame(mTimestampNanos, mFrame);
        }
    }
```
可以看出 FrameDisplayEventReceiver 是用于接收垂直同步信号的，接收到之后就发送一个消息，然后会执行 runnable，从而调用 doFrame()。
```java
    void doFrame(long frameTimeNanos, int frame) {
        final long startNanos;
        synchronized (mLock) {
            if (!mFrameScheduled) {
                return; // no work to do
            }

            long intendedFrameTimeNanos = frameTimeNanos;
            startNanos = System.nanoTime();
            // 用当前时间减去接收到刷新信号的时间计算出时间间隔
            final long jitterNanos = startNanos - frameTimeNanos;

            // 间隔大于16.7ms时就说明出现了掉帧
            if (jitterNanos >= mFrameIntervalNanos) {
                // 掉帧次数
                final long skippedFrames = jitterNanos / mFrameIntervalNanos;
                final long lastFrameOffset = jitterNanos % mFrameIntervalNanos;
                // 将之修改为最后一次 VSYNC 的时间
                frameTimeNanos = startNanos - lastFrameOffset;
            }

            // 如果上次刷新的时间比当前接收到刷新信号的时间还要新
            // 的话（修改手机时间的话，System.nanoTime() 发生倒退时会出现）
            // 就不去刷新，而是忽略当前信号，等待下一个屏幕刷新信号
            if (frameTimeNanos < mLastFrameTimeNanos) {
                scheduleVsyncLocked();
                return;
            }

            mFrameInfo.setVsync(intendedFrameTimeNanos, frameTimeNanos);
            mFrameScheduled = false;
            mLastFrameTimeNanos = frameTimeNanos;
        }

        try {
            Trace.traceBegin(Trace.TRACE_TAG_VIEW, "Choreographer#doFrame");
            AnimationUtils.lockAnimationClock(frameTimeNanos / TimeUtils.NANOS_PER_MS);

            // 开始处理前面讲的4种 Callback
            mFrameInfo.markInputHandlingStart();
            doCallbacks(Choreographer.CALLBACK_INPUT, frameTimeNanos);

            mFrameInfo.markAnimationsStart();
            doCallbacks(Choreographer.CALLBACK_ANIMATION, frameTimeNanos);

            mFrameInfo.markPerformTraversalsStart();
            doCallbacks(Choreographer.CALLBACK_TRAVERSAL, frameTimeNanos);

            doCallbacks(Choreographer.CALLBACK_COMMIT, frameTimeNanos);
        } finally {
            AnimationUtils.unlockAnimationClock();
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
    }
```
doFrame 所做的事情就是计算当前时间与接收到信号的时间，如果时间大于16ms则说明掉帧了，然后将当前的帧时间修改为最后一次接收到信号的时间。然后执行 doCallbacks。
```java
    void doCallbacks(int callbackType, long frameTimeNanos) {
        CallbackRecord callbacks;
        synchronized (mLock) {
            final long now = System.nanoTime();
            // 根据 callbackType 拿到相应的 callback 队列，并取出该队列中时间小于 now 的队列
            callbacks = mCallbackQueues[callbackType].extractDueCallbacksLocked(
                    now / TimeUtils.NANOS_PER_MS);
            if (callbacks == null) {
                return;
            }
            mCallbacksRunning = true;

            // 如果类型为 COMMIT 并且当前帧的渲染时间超过两帧的时间，则将时间修改为上上个
            if (callbackType == Choreographer.CALLBACK_COMMIT) {
                final long jitterNanos = now - frameTimeNanos;

                if (jitterNanos >= 2 * mFrameIntervalNanos) {
                    final long lastFrameOffset = jitterNanos % mFrameIntervalNanos
                            + mFrameIntervalNanos;
                    frameTimeNanos = now - lastFrameOffset;
                    // 当前时间的前2帧的时间
                    mLastFrameTimeNanos = frameTimeNanos;
                }
            }
        }
        try {
            Trace.traceBegin(Trace.TRACE_TAG_VIEW, CALLBACK_TRACE_TITLES[callbackType]);
            // 遍历执行链表
            for (CallbackRecord c = callbacks; c != null; c = c.next) {
                c.run(frameTimeNanos);
            }
        } finally {
            synchronized (mLock) { // 释放链表所有结点
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

    void doScheduleVsync() {
        synchronized (mLock) {
            if (mFrameScheduled) {
                scheduleVsyncLocked();
            }
        }
    }
```
首先取出执行时间在当前时间之前部分的 CallbackRecord，然后遍历该链表并执行。



我们来看下 FrameHandler 是处理什么消息的：
```java
    private final class FrameHandler extends Handler {
        public FrameHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case MSG_DO_FRAME:
                    doFrame(System.nanoTime(), 0);
                    break;
                case MSG_DO_SCHEDULE_VSYNC:
                    doScheduleVsync();
                    break;
                case MSG_DO_SCHEDULE_CALLBACK:
                    doScheduleCallback(msg.arg1);
                    break;
            }
        }
    }
```







## 参考文献
+ [android 动画系列 (2) - interpolator 插值器](https://www.jianshu.com/p/48317612c164)   
+ [模拟自然动画的精髓——TimeInterpolator与TypeEvaluator](https://www.jianshu.com/p/b239d14060a8) 