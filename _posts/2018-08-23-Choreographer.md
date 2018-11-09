---
layout:     post
title:      Choreographer
subtitle:   
header-img: img/android3.png
author:     Allen Vork
catalog: true
tags:
    - performance
---

## Synopsis
该类是用于协调动画、输入和绘图的时间。它接收来自显示子系统的时间脉冲（如垂直同步信号），然后安排下一帧的渲染工作。应用程序一般不直接与它进行交互，而是在动画框架或者 view 层级中使用更高级的抽象来进行操作。

## constructor
```java
    private Choreographer(Looper looper, int vsyncSource) {
        // 将当前线程的 looper 给 Handler
        mLooper = looper;
        mHandler = new FrameHandler(looper);

        //创建VSYNC的信号接受对象
        mDisplayEventReceiver = USE_VSYNC
                ? new FrameDisplayEventReceiver(looper, vsyncSource)
                : null;

        //初始化上一次frame渲染的时间点
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

## Choreographer 使用
### 添加 Runnable 对象
在 View 调用 invalidate() 方法后，最终会调到 ViewRootImpl 中并执行 scheduleTraversals() 方法：
```java
    public ViewRootImpl(Context context, Display display) {
        mChoreographer = Choreographer.getInstance();
    }

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
这里调用了 postCallback 方法将类型为 Choreographer.CALLBACK_TRAVERSAL 的 mTraversalRunnable 传进去了。然后 Choreographer 内部会调用：
```java
    public void postCallbackDelayed(FrameCallback callback, long delayMillis) {
        postCallbackDelayedInternal(callbackType, action, token, delayMillis)
    }
```

### 添加 FrameCallback 对象
创建一个 FragmeCallback
```java
    private class FrameCallback implements Choreographer.FrameCallback {
        @Override
        public void doFrame(long frameTimeNanos) {
            sendEmptyMessage(UPDATE);
        }
    };
```
然后调用 Choreographer 的方法传进去：
```java
    Choreographer.getInstance().postFrameCallback(new FrameCallback());
```
然后 Choreographer 内部会调用：
```java
    public void postFrameCallbackDelayed(FrameCallback callback, long delayMillis) {
        postCallbackDelayedInternal(CALLBACK_ANIMATION,
                callback, FRAME_CALLBACK_TOKEN, delayMillis);
    }
```

> 可以看出这添加 Runnable 和 FrameCallback 最终都会调用到 postCallbackDelayedInternal()，注意第三个参数 token 一个为空，一个为 FRAME_CALLBACK_TOKEN。 FrameCallback 其实相当于给 Runnable 中的 run() 方法添加了一个参数。

```java
    private void postCallbackDelayedInternal(int callbackType,
            Object action, Object token, long delayMillis) {

        synchronized (mLock) {
            final long now = SystemClock.uptimeMillis();
            final long dueTime = now + delayMillis;
            // 后面会详细讲这个数组
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
最终它们都会根据类型添加到对应的 mCallbackQueues 中。如果该任务不需要 delay，那么立马执行 scheduleFrameLocked(now)，否则发送一个类型为 MSG_DO_SCHEDULE_CALLBACK 的异步消息。

我们来看看 scheduleFrameLocked:
```java
    private void scheduleFrameLocked(long now) {
        if (!mFrameScheduled) {
            mFrameScheduled = true;
            if (USE_VSYNC) { // 使用 VSYNC 信号，这个参数由系统值确定
                // 如果运行在 Looper 线程则执行 scheduleVsyncLocked() 方法
                if (isRunningOnLooperThreadLocked()) {
                    // 请求 VSYNC 信号，它会触发 FrameDisplayEventReceiver 的 onVsync 回调，
                    // 然后调用 doFrame(long frameTimeNanos, int frame) 方法。该方法后面会详细讲
                    scheduleVsyncLocked();
                } else {
                    // 发送类型为 MSG_DO_SCHEDULE_VSYNC 的消息，最终也会调到上面的 scheduleVsyncLocked()
                    Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_VSYNC);
                    msg.setAsynchronous(true);
                    mHandler.sendMessageAtFrontOfQueue(msg);
                }
            } else {
                final long nextFrameTime = Math.max(
                        mLastFrameTimeNanos / TimeUtils.NANOS_PER_MS + sFrameDelay, now);
                // 发送 MSG_DO_FRAME 消息，它会调用 doFrame(long frameTimeNanos, int frame) 方法
                Message msg = mHandler.obtainMessage(MSG_DO_FRAME);
                msg.setAsynchronous(true);
                mHandler.sendMessageAtTime(msg, nextFrameTime);
            }
        }
    }
```
可以看出，如果是使用 VSYNC 信号，那么当它运行在 Looper 线程的话，会调用 scheduleVsyncLocked() 方法来触发 FrameDisplayEventReceiver 的 onVsync 回调，从而调到 doFrame() 方法中，否则发送一个 MSG_DO_SCHEDULE_VSYNC 消息。如果不适用 VSYNC 信号，则会发送一个 MSG_DO_FRAME 消息。

## CallbackQueue
前面我们传给 Choreographer 的 Runnable 和 FrameCallback 都会添加到 CallbackQueue 中，构造函数中会创建容量为4的 CallbackQueue 数组来存放4种不同的 Callback，我们来看下这4种 CallbackQueue:
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
可以看出这个类里面会存放一个链表的头结点，并提供了添加，删除，读取来操作该链表。
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
链表的结点的结构比较简单，注释里讲的比较清楚。前面在我们注册 Runnable 之类的时，会调用到 mCallbackQueues[callbackType].addCallbackLocked(dueTime, action, token)，也就是将该 Runnalbe 存放到相应类型的 CallbackQueue 中的链表的相应位置。

## FrameDisplayEventReceiver
前面讲过，当调用 postCallback() 将 Runnable 等传进来时，如果该 Runnable 等没有设置 delay，则会调用 scheduleVsyncLocked() 来请求 VSYNC 信号，接收到信号时就会触发它的 onVsync：    
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
接收到垂直同步信号后就会将自己作为 Runnable 发送出去，然后会调用它的 run 方法从而调用 doFrame()。
```java
    void doFrame(long frameTimeNanos, int frame) {
        final long startNanos;
        synchronized (mLock) {
            if (!mFrameScheduled) {
                return; // no work to do
            }

            long intendedFrameTimeNanos = frameTimeNanos;
            startNanos = System.nanoTime();
            // 抖动间隔：用当前时间减去接收到刷新信号的时间计算出时间间隔
            final long jitterNanos = startNanos - frameTimeNanos;

            // 抖动间隔大于16.7ms时就说明出现了掉帧
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
首先取出执行时间在当前时间之前部分的 CallbackRecord，然后遍历该链表并执行 callback 的 run 方法:
```java
        public void run(long frameTimeNanos) {
            if (token == FRAME_CALLBACK_TOKEN) {
                ((FrameCallback)action).doFrame(frameTimeNanos);
            } else {
                ((Runnable)action).run();
            }
        }
```

我们来看下 FrameHandler 是处理什么消息的：
```java
    private final class FrameHandler extends Handler {
        public FrameHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case MSG_DO_FRAME: // 前面已经讲过，用于执行4种 Callback 的
                    doFrame(System.nanoTime(), 0); 
                    break;
                case MSG_DO_SCHEDULE_VSYNC: // 这个前面讲过，用于请求 VSYNC 信号，信号到达后调用 doFrame()
                    doScheduleVsync();
                    break;
                case MSG_DO_SCHEDULE_CALLBACK: // 这个会调到 scheduleFrameLocked（）前面讲过
                    doScheduleCallback(msg.arg1);
                    break;
            }
        }
    }
```
可以看出，FrameHandler 真正是做2件事，直接执行 doFrame() 来执行那4种 Callback，要么请求 VSYNC 信号，然后在下个信号到来时调用 doFrame()。

## Sample
![]({{site.url}}/img/android/basic/performance/ProjectButter/9.png)
+ 1.调用 postCallBack(runnable) ，它会调用
```java
    private void postCallbackDelayedInternal(int callbackType,
            Object action, Object token, long delayMillis) {

        synchronized (mLock) {
            final long now = SystemClock.uptimeMillis();
            final long dueTime = now + delayMillis;
            // 将 action 添加到 mCallbackQueues 中
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
然后会调用到 scheduleFrameLocked() 方法：
```java
    private void scheduleFrameLocked(long now) {
        if (!mFrameScheduled) {
            mFrameScheduled = true;
            if (USE_VSYNC) {
                if (isRunningOnLooperThreadLocked()) {
                    scheduleVsyncLocked();
                }
                ...
            }
        }
    }
```
此时就会调用到 scheduleVsyncLocked() 方法，该方法是用来监听 VSYNC 信号的。    
+ 2.frameTimeNanos1 时刻收到 VSync 信号：
```java
    private final class FrameDisplayEventReceiver extends DisplayEventReceiver
            implements Runnable {
        ...

        @Override
        public void onVsync(long timestampNanos, int builtInDisplayId, int frame) {
            ...
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
它会发送一个消息，然后执行 run() 方法来触发 doFrame()。doFrame() 会判断是否有掉帧，然后执行传进来的所有应该执行的 runnable。
+ 3.runnable都执行完成后，又调用 postCallback(runnable) ，它会走第一步一样的过程，在 scheduleFrameLocked() 方法中：
```java
    private void scheduleFrameLocked(long now) {
        if (!mFrameScheduled) {
            mFrameScheduled = true;
        }
    }
``` 
将 mFrameScheduled 设为 true。然后调用 scheduleVsyncLocked() 来监听下一帧的信号
+ 4.这时又执行了一次 postCallback(runnable)，将 runnable 放到 mCallbackQueues，由于 mFrameScheduled 之前被设为 true，所以走到 scheduleFrameLocked 将不做任何事。    
+ 5.当 frameTimeNanos2 时刻接收到 VSYNC 信号时，由于 CPU 被占用（上图的黄色部分），就没有立即回调 onVsync 方法。虽然稍微延时，但并没有超过16ms，所以没有掉帧。它同样是发送消息到主线程，然后执行 doFrame() 方法来执行所有小于当前时间的 runnable。
+ 6.在非 UI 线程再次调用 postCallback(runnable) ，由于 Choreographer 为线程单例，所以当前的 Choreographer 为新的实例, mFrameScheduled 为 false:
```java
    private void scheduleFrameLocked(long now) {
        if (!mFrameScheduled) {
            mFrameScheduled = true;
            if (USE_VSYNC) {
                if (isRunningOnLooperThreadLocked()) { // 是不是在主线程
                    scheduleVsyncLocked();
                } else { // 由于当前是在子线程，所以会执行这里
                    Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_VSYNC);
                    msg.setAsynchronous(true);
                    mHandler.sendMessageAtFrontOfQueue(msg);
                }
            }
        }
    }
```
这时它会发个消息到主线程，消息的类型为 MSG_DO_SCHEDULE_VSYNC。可以看到在 frameTimeNanos3 时，runnable 还没执行完，那么这个消息就还不会被处理。由于前面只有在第4步调用了 postCallBack() 会调用监听下一帧信号的回调，从而接收到 frameTimeNanos2 的 VSync 信号，后面都没有触发 scheduleVsyncLocked() 方法来监听下一帧的信号，所以 frameTimeNanos3 不会接收到信号，也就不做任何处理。
+ 7.在 frameTimeNanos3 时间点之后，runnable 执行完了，这时主线程可以处理第6步发送的消息了：
```java
    private final class FrameHandler extends Handler {

        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case MSG_DO_FRAME:
                    doFrame(System.nanoTime(), 0);
                    break;
                case MSG_DO_SCHEDULE_VSYNC:
                    doScheduleVsync(); // 设置监听 VSYNC 信号
                    break;
                case MSG_DO_SCHEDULE_CALLBACK:
                    doScheduleCallback(msg.arg1);
                    break;
            }
        }
    }
```
它调用 doScheduleVsync() 来监听下一帧的信号。
+ 8.当 frameTimeNanos4 信号到来时，它同样是向主线程发送一个消息。由于此时 CPU 一致被占用，fameTimeNanos5 到来时，CPU 一直被占用，那么就无法处理消息。在 fameTimeNanos5 时间点之后就会处理 VSync 信号，然后执行 doFrames()。可以看出第四个红箭头传进来的 runnable 本应在 frameTimeNanos3 处理，但是被延时到了 frameTimeNanos5 之后，也就是中间调了2帧，那么 doFrame 方法就会修正 frameTimeNanos 为 frameTimeNanos5，然后执行 runnable。












## 参考文献
+ [Android系统的编舞者Choreographer](https://www.jianshu.com/p/fb645ea98474)   
+ [Android Choreographer](https://blog.csdn.net/ahence/article/details/78417186) 
+ [Android系统的编舞者Choreographer](https://juejin.im/entry/5ae1a4aef265da0b7e0bf94a)
+ [Android Choreographer 源码分析](https://www.jianshu.com/p/996bca12eb1d)
+ [](https://www.jianshu.com/p/dd32ec35db1d)