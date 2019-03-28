---
layout:     post
title:      requestLayout、invalidate、postInvalidate 原理
subtitle:   从源码角度讲解
header-img: img/android3.png
author:     Allen Vork
catalog: true
tags:
    - java
---

## 概述
+ **requestLayout**：会触发三大流程。
+ **invalidate**：触发 onDraw 流程，在 UI 线程调用。
+ **postInvalidate**：触发 onDraw 流程，在非 UI 线程中调用。

## requestLayout
直接来看 View 中的 requestLayout 流程：    
```java
    public void requestLayout() {
        if (mMeasureCache != null) mMeasureCache.clear();

        // mViewRequestingLayout 为正在执行 requestLayout 方法的视图
        if (mAttachInfo != null && mAttachInfo.mViewRequestingLayout == null) {
            // Only trigger request-during-layout logic if this is the view requesting it,
            // not the views in its parent hierarchy
            ViewRootImpl viewRoot = getViewRootImpl();
            // 
            if (viewRoot != null && viewRoot.isInLayout()) {
                if (!viewRoot.requestLayoutDuringLayout(this)) {
                    return;
                }
            }
            // 将当前正在执行 requestLayout 的 View 保存到 mAttachInfo 中
            mAttachInfo.mViewRequestingLayout = this;
        }

        mPrivateFlags |= PFLAG_FORCE_LAYOUT;
        mPrivateFlags |= PFLAG_INVALIDATED;
        // 这个 mParent 就是 ViewParent，ViewGroup 实现了 ViewParent
        // 之所以不使用 ViewGroup 是因为 DecorView 的 parent 不是 View。
        if (mParent != null && !mParent.isLayoutRequested()) {
            mParent.requestLayout();
        }
        if (mAttachInfo != null && mAttachInfo.mViewRequestingLayout == this) {
            mAttachInfo.mViewRequestingLayout = null;
        }
    }
```
它主要分为3大流程：    
1. 判断如果 View 的结构还处于 layout 阶段之前，则调用 ViewRootImpl#requestLayoutDuringLayout(this)，将当前的 View 对象传递进去。这点待会看 ViewRootImpl#performLayout 就知道了。
2. 更新 mPrivateFlags 变量： PFLAG_FORCE_LAYOUT、PFLAG_INVALIDATED。
3. 调用 parent.requestLayout。

由于 ViewGroup 并没有重写 requestLayout，所以它调用的也是 View 的 requestLayout 方法。最终会调到 ViewRootImpl 中：    
```java
---------ViewRootImpl------------

    @Override
    public void requestLayout() {
        if (!mHandlingLayoutInLayoutRequest) {
            checkThread();
            mLayoutRequested = true;
            scheduleTraversals();
        }
    }
```
它主要是检查了一下线程，如果不在主线程就会抛异常。继续来看 scheduleTraversals()：    
```java
---------ViewRootImpl------------
    void scheduleTraversals() {
        if (!mTraversalScheduled) {
            mTraversalScheduled = true;
            // 插入同步屏障，表示当前任务不可打断
            mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
            // 将绘制任务交给 Choreographer，下次信号到来时会执行该任务。
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
它会先插入同步屏障，然后将任务交给 Choreographer，当垂直信号到来时就会执行。我们看下它执行了什么：    
```java
---------ViewRootImpl------------
    void doTraversal() {
        if (mTraversalScheduled) {
            mTraversalScheduled = false;
            // 将同步屏障移除
            mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);
            // 它就会执行 measure、layout、draw 的流程
            performTraversals();

            if (mProfile) {
                Debug.stopMethodTracing();
                mProfile = false;
            }
        }
    }
```
可以看出 requestLayout 就是重新执行 View 的三大流程。整个过程都是先向上传递再向下传递。整个 View 树结构就是一个双向指针结构。可以看出他就是一个责任链模式。     

执行到 measure 后，我们来看下它做了什么：    
```java
---------View------------
        // 获取之前在 view 中设置的 PFLAG_FORCE_LAYOUT
        final boolean forceLayout = (mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT;
        if (forceLayout || needsLayout) {
            // first clears the measured dimension flag
            mPrivateFlags &= ~PFLAG_MEASURED_DIMENSION_SET;

            resolveRtlPropertiesIfNeeded();

            int cacheIndex = forceLayout ? -1 : mMeasureCache.indexOfKey(key);
            if (cacheIndex < 0 || sIgnoreMeasureCache) {
                // measure ourselves, this should set the measured dimension flag back
                onMeasure(widthMeasureSpec, heightMeasureSpec);
                mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
            } else {
                long value = mMeasureCache.valueAt(cacheIndex);
                // Casting a long to int drops the high 32 bits, no mask needed
                setMeasuredDimensionRaw((int) (value >> 32), (int) value);
                mPrivateFlags3 |= PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
            }
            // 然后再添加一个 LAYOUT_REQUERED flag
            mPrivateFlags |= PFLAG_LAYOUT_REQUIRED;
        }
```
我们添加了 forcelayout 标志后就会去执行 onMeasure 操作，后面又添加了一个 LAYOUT_REQUIRED 标志，它是用在 layout 中的：    
```java
---------View------------
    public void layout(int l, int t, int r, int b) {
        ...
        if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
            onLayout(changed, l, t, r, b);
            ...
        }
    }
```
可以看出设置那个标志的目的就是强制布局。可以看出，它只会更新有设置这些 flag 的 View，并不是会将所有 View 的三大流程都执行一遍。

## 为什么在 ViewGroup 的 onLayout 中调用 requestLayout 不会导致死递归
我们知道 onLayout 调用 requestLayout 会触发 View 的三大流程重新执行，这时又会执行到 onLayout，又会触发 requestLayout，这样就会导致死递归。那 google 是怎么解决的呢？    
```java
---------View------------
    public void requestLayout() {
        if (mMeasureCache != null) mMeasureCache.clear();
        // View.AttachInfo 保存 View 和 Window 之间的信息，每一个被添加到窗口上的 View 都有对应的 AttachInfo。
        if (mAttachInfo != null && mAttachInfo.mViewRequestingLayout == null) {
            ViewRootImpl viewRoot = getViewRootImpl();
            // 当前整个 View 树的 onLayout 还没执行完的话，如果有 View 调用 requestLayout，
            // 就将这个 View 放到 ViewRootImpl 的任务队列中。当 onLayout 执行完成后就会调用这个
            // view 的 layout 方法。
            if (viewRoot != null && viewRoot.isInLayout()) {
                if (!viewRoot.requestLayoutDuringLayout(this)) {
                    return;
                }
            }
            mAttachInfo.mViewRequestingLayout = this;
        }
        // ······
    }
```
如果 View 树处于 layout 阶段，即 DecorView 的 layout 没执行完，如果此时正在 layout 的某个 View 调用 requestLayout，会将这个任务放到 ViewRootImpl 的任务队列中，这个任务队列的执行时在 ViewRootImpl#performLayout 中：    
```java
    private void performLayout(WindowManager.LayoutParams lp, int desiredWindowWidth,
            int desiredWindowHeight) {
        mLayoutRequested = false;
        try {
            // host 即 DecorView，它会执行 decorView 的 layout 方法
            host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());

            // 执行完成后将 mInLayout 设为 false。
            mInLayout = false;
            // 然后再去执行任务队列中的任务
            int numViewsRequestingLayout = mLayoutRequesters.size();
            if (numViewsRequestingLayout > 0) {
                ArrayList<View> validLayoutRequesters = getValidLayoutRequesters(mLayoutRequesters,
                        false);
                if (validLayoutRequesters != null) {
                    mHandlingLayoutInLayoutRequest = true;
                    int numValidRequests = validLayoutRequesters.size();
                    for (int i = 0; i < numValidRequests; ++i) {
                        final View view = validLayoutRequesters.get(i);
                        view.requestLayout(); // 调用任务队列里的 View 的 requestLayout
                    }
                    measureHierarchy(host, lp, mView.getContext().getResources(),
                            desiredWindowWidth, desiredWindowHeight);
                    mInLayout = true;
                    // 又调用了 decorView 的 layout 方法
                    host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());

                    mHandlingLayoutInLayoutRequest = false;
                }

            }
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
        mInLayout = false;
    }
```
这个方法是由 Choreographer 执行的，用于遍历 View 树执行3大流程，具体逻辑：    
1. 执行 DecorView#layout，他就会触发所有的子 view 的三大流程
2. 整个 View 树执行完成后就去将 mInLayout 设为 false，代表当前没有在布局。那么某个 view 如果在 onLayout 中调用了 requestLayout 方法的话，就可以直接触发正常的流程，而不会被添加到任务队列中。
3. 然后执行任务队列中的任务。他就是取出队列中所有的 view，然后调用它的 requestLayout 方法。

那么问题来了，执行 View 的 requestLayout 方法，这样又会从下往上传递，再从上往下执行三大流程，执行到这个 View 的 onLayout 时，又会调用 requestLayout 方法，因为当时处于 layout 状态，所以会将这个 requestLayout 任务添加到任务队列中。等到整个任务执行完成后，它又回去执行任务队列中的任务，循环往复。 那么问题还是没有解决。    
我们来想一下 layout 的目的是干什么的，他就是将 view 绘制屏幕上的确切位置上。如果 View 的坐标没变的话，为什么还需要重新绘制？所以答案就在 layout 里面：    
```java
    public void layout(int l, int t, int r, int b) {

        boolean changed = isLayoutModeOptical(mParent) ?
                setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);

        if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
            onLayout(changed, l, t, r, b);
            // ······
        }
    }
```
这个 change 变量就是判断坐标有没有发生改变，发生了才会调用 onLayout 进行重新布局。    


### 总结
requestLayout 的流程：    
1. 给 view 的 flag 添加2个变量：FORCE_LAYOUT 和 INVALIDATE。然后调用 parent 的 requestLayout，执行相同的流程
2. 最终执行到 ViewRootImpl 中。他会将执行遍历的操作封装到 Runnable 中并丢给 Choreographer 处理。Choreographer 在下一个垂直信号到来的时候就会执行这个 Runnable。它里面就会去执行那3大流程。
3. 首先它会执行到 measure 方法，它里面就会根据 requestLayout 中设置的 FORCE_LAYOUT 的标志来调用 onMeasure 方法。执行完成之后就会设置一个 LAYOUT_REQUIRE 标志。然后 layout 方法中又会根据这个这个标志来调用 onLayout 方法。 onDraw 方法也是类似的流程。

可以看出它是一个从下往上回溯，然后从上往下遍历的过程，它是一个责任链模式。那么如果我们直接在 onLayout 中调用 requestLayout 的话，那么这个责任链模式是一个环形的，那么就会造成死递归。但是你真正去执行的时候，它是不会出现这种现象的。    
原因：我们来看下如果直接在 onLayout 中调用 requestLayout 会发生什么。requestLayout 首先会判断当前是不是正在 layout，如果是的话，则将执行这个 requestLayout 的 view 放到 ViewRootImpl 中的任务队列中，而不会去像之前那样往上回溯。当 layout 过程结束后，他就会去执行这个任务队列。他就是取出 view，然后调用它的 requestLayout 方法。然后走一遍之前的流程，调用到这个 view 的 onLayout 方法中，这样又会调用 requestLayout 方法。那么问题还是没有解决，但是我们可以发现，它其实是没有调用的，那么就往上找到 layout 方法中，前面讲到了执行 onLayout 方法是会根据你设置的 FORCE_LAYOUT 之类的调用进来。由于它是在 layout 时调用的 requestLayout，是直接将 view 加入任务队列中，然后 return 的，并没有添加那些参数，那么就不会强制 layout。除了这个参数外，还有一个参数会控制它是否执行 onLayout 方法，就是 changed，changed 就是看你这个布局的位置有没有发生改变，这个时候肯定为 false。那么2个条件都不满足，就不会去执行 onLayout 了，也就不会造成死递归。

## invalidate
invalidate 分为全局刷新和局部刷新。全局刷新的话是刷新整个 view 树，而局部刷新会一当前 view 为根开始刷新。

```java
    public void invalidate() {
        invalidate(true); // true 表示局部刷新
    }

    public void invalidate(boolean invalidateCache) {
        // 前四个参数为 View 相对于自己的坐标
        invalidateInternal(0, 0, mRight - mLeft, mBottom - mTop, invalidateCache, true);
    }

    void invalidateInternal(int l, int t, int r, int b, boolean invalidateCache,
            boolean fullInvalidate) {
        if (mGhostView != null) {
            mGhostView.invalidate(true);
            return;
        }
        
        // 跳过刷新条件：View 不可见，并且自己没有动画执行，并且当前 View 为 DecorView 或者父布局也没有过度动画执行
        // 如果 parent 为 ViewRootImpl 的话，说明当前 View 为 DecorView，
        if (skipInvalidate()) {
            return;
        }

        if ((mPrivateFlags & (PFLAG_DRAWN | PFLAG_HAS_BOUNDS)) == (PFLAG_DRAWN | PFLAG_HAS_BOUNDS)
                || (invalidateCache && (mPrivateFlags & PFLAG_DRAWING_CACHE_VALID) == PFLAG_DRAWING_CACHE_VALID)
                || (mPrivateFlags & PFLAG_INVALIDATED) != PFLAG_INVALIDATED
                || (fullInvalidate && isOpaque() != mLastIsOpaque)) {
            if (fullInvalidate) {
                mLastIsOpaque = isOpaque();
                mPrivateFlags &= ~PFLAG_DRAWN;
            }

            mPrivateFlags |= PFLAG_DIRTY;

            if (invalidateCache) {
                mPrivateFlags |= PFLAG_INVALIDATED;
                mPrivateFlags &= ~PFLAG_DRAWING_CACHE_VALID;
            }

            // Propagate the damage rectangle to the parent view.
            final AttachInfo ai = mAttachInfo;
            final ViewParent p = mParent;
            if (p != null && ai != null && l < r && t < b) {
                final Rect damage = ai.mTmpInvalRect; 
                damage.set(l, t, r, b); // 这些坐标是需要 invalidate 的

                // 调用 ViewGroup 的 invalidateChild
                p.invalidateChild(this, damage);
            }

            // Damage the entire projection receiver, if necessary.
            if (mBackground != null && mBackground.isProjected()) {
                final View receiver = getProjectionReceiver();
                if (receiver != null) {
                    receiver.damageInParent();
                }
            }
        }
    }
```
1. 根据 View 的可见性，动画之类的判断需不需要跳过刷新
2. 不跳过的话，将记录了该 view 相对于他的 parent 的坐标放到 damage 中，然后调用 ViewParent#invalidateChild 来调整这个区域。

```java
    public final void invalidateChild(View child, final Rect dirty) {
        final AttachInfo attachInfo = mAttachInfo;
        if (attachInfo != null && attachInfo.mHardwareAccelerated) {
            // 硬件加速的话，直接调用下面这个方法
            onDescendantInvalidated(child, child);
            return;
        }

        ViewParent parent = this;
        if (attachInfo != null) {
            // 当前 View 是否正在执行动画
            final boolean drawAnimation = (child.mPrivateFlags & PFLAG_DRAW_ANIMATION) != 0;

            // Check whether the child that requests the invalidate is fully opaque
            // Views being animated or transformed are not considered opaque because we may
            // be invalidating their old position and need the parent to paint behind them.
            Matrix childMatrix = child.getMatrix();
            // 不透明并且没有（执行）动画，并且变化矩阵没有变化（Matrix 是用于平移，缩放等）
            final boolean isOpaque = child.isOpaque() && !drawAnimation &&
                    child.getAnimation() == null && childMatrix.isIdentity();
            // Mark the child as dirty, using the appropriate flag
            // Make sure we do not set both flags at the same time
            int opaqueFlag = isOpaque ? PFLAG_DIRTY_OPAQUE : PFLAG_DIRTY;

            if (child.mLayerType != LAYER_TYPE_NONE) {
                mPrivateFlags |= PFLAG_INVALIDATED;
                mPrivateFlags &= ~PFLAG_DRAWING_CACHE_VALID;
            }

            // 记录子 view 相对于 parent 的左上角的坐标
            final int[] location = attachInfo.mInvalidateChildLocation;
            location[CHILD_LEFT_INDEX] = child.mLeft;
            location[CHILD_TOP_INDEX] = child.mTop;
            if (!childMatrix.isIdentity() ||
                    (mGroupFlags & ViewGroup.FLAG_SUPPORT_STATIC_TRANSFORMATIONS) != 0) {
                RectF boundingRect = attachInfo.mTmpTransformRect;
                boundingRect.set(dirty);
                Matrix transformMatrix;
                if ((mGroupFlags & ViewGroup.FLAG_SUPPORT_STATIC_TRANSFORMATIONS) != 0) {
                    Transformation t = attachInfo.mTmpTransformation;
                    boolean transformed = getChildStaticTransformation(child, t);
                    if (transformed) {
                        transformMatrix = attachInfo.mTmpMatrix;
                        transformMatrix.set(t.getMatrix());
                        if (!childMatrix.isIdentity()) {
                            transformMatrix.preConcat(childMatrix);
                        }
                    } else {
                        transformMatrix = childMatrix;
                    }
                } else {
                    transformMatrix = childMatrix;
                }
                transformMatrix.mapRect(boundingRect);
                dirty.set((int) Math.floor(boundingRect.left),
                        (int) Math.floor(boundingRect.top),
                        (int) Math.ceil(boundingRect.right),
                        (int) Math.ceil(boundingRect.bottom));
            }

            // 从当前 View 向上一直遍历到 ViewRootImpl
            do {
                View view = null;
                if (parent instanceof View) { // 不是 ViewRootImpl 就将 ViewParent 转换为 View
                    view = (View) parent;
                }

                if (drawAnimation) { // 当前 View 在执行动画，添加动画标志
                    if (view != null) {
                        view.mPrivateFlags |= PFLAG_DRAW_ANIMATION;
                    } else if (parent instanceof ViewRootImpl) {
                        ((ViewRootImpl) parent).mIsAnimating = true;
                    }
                }

                // 给 View 添加 dirty 标志表示需要重绘
                if (view != null) {
                    if ((view.mViewFlags & FADING_EDGE_MASK) != 0 &&
                            view.getSolidColor() == 0) {
                        opaqueFlag = PFLAG_DIRTY;
                    }
                    if ((view.mPrivateFlags & PFLAG_DIRTY_MASK) != PFLAG_DIRTY) {
                        view.mPrivateFlags = (view.mPrivateFlags & ~PFLAG_DIRTY_MASK) | opaqueFlag;
                    }
                }

                // 调用当前 view 的 invalidateChildInParent，返回当前 view 的 parent，最后返回的是 ViewRootImpl
                // 参数就是告诉父视图，绘制的子视图以及子视图相对自己的坐标系
                parent = parent.invalidateChildInParent(location, dirty);
                if (view != null) {
                    // Account for transform on current parent
                    Matrix m = view.getMatrix();
                    if (!m.isIdentity()) {
                        RectF boundingRect = attachInfo.mTmpTransformRect;
                        boundingRect.set(dirty);
                        m.mapRect(boundingRect);
                        dirty.set((int) Math.floor(boundingRect.left),
                                (int) Math.floor(boundingRect.top),
                                (int) Math.ceil(boundingRect.right),
                                (int) Math.ceil(boundingRect.bottom));
                    }
                }
            } while (parent != null);
        }
    }
```
可以看出它就是给 view 添加动画，dirty 之类的标志，然后调用它的 invalidateInParent 方法：    
```java
    public ViewParent invalidateChildInParent(final int[] location, final Rect dirty) {
        if ((mPrivateFlags & (PFLAG_DRAWN | PFLAG_DRAWING_CACHE_VALID)) != 0) {
            if ((mGroupFlags & (FLAG_OPTIMIZE_INVALIDATE | FLAG_ANIMATION_DONE))
                    != FLAG_OPTIMIZE_INVALIDATE) { // 没有动画，或者动画执行完了
                // dirty 变换为相对于父视图的坐标系的值。相对于父视图左上角的坐标减去当前 view 滑动的距离
                // 整个得到的就是当前 view 相对于他的 parent 的坐标
                dirty.offset(location[CHILD_LEFT_INDEX] - mScrollX,
                        location[CHILD_TOP_INDEX] - mScrollY);
                // 不裁剪（取最大值来刷新），则取并集，取最大区域
                if ((mGroupFlags & FLAG_CLIP_CHILDREN) == 0) {
                    dirty.union(0, 0, mRight - mLeft, mBottom - mTop);
                }

                // 获取当前 View 相对于 Parent 的左上角的坐标
                final int left = mLeft;
                final int top = mTop;
                // 裁减的话，则取交集（仅仅绘制子 View 内有效的区间）
                if ((mGroupFlags & FLAG_CLIP_CHILDREN) == FLAG_CLIP_CHILDREN) {
                    if (!dirty.intersect(0, 0, mRight - left, mBottom - top)) {
                        dirty.setEmpty();
                    }
                }

                location[CHILD_LEFT_INDEX] = left;
                location[CHILD_TOP_INDEX] = top;
            } else {//如果当前ViewGroup中有动画要执行

                if ((mGroupFlags & FLAG_CLIP_CHILDREN) == FLAG_CLIP_CHILDREN) {
                    dirty.set(0, 0, mRight - mLeft, mBottom - mTop);
                } else {
                    // in case the dirty rect extends outside the bounds of this container
                    dirty.union(0, 0, mRight - mLeft, mBottom - mTop);
                }
                location[CHILD_LEFT_INDEX] = mLeft;
                location[CHILD_TOP_INDEX] = mTop;

                mPrivateFlags &= ~PFLAG_DRAWN;
            }
            mPrivateFlags &= ~PFLAG_DRAWING_CACHE_VALID;
            if (mLayerType != LAYER_TYPE_NONE) {
                mPrivateFlags |= PFLAG_INVALIDATED;
            }

            return mParent;
        }

        return null;
    }
```
可以看出它整个就是根据当前 view 相对自己的坐标，找到最终相对于父视图的坐标。若设置 FLAG_CLIP_CHILDREN 则仅绘制调用 invalidate 方法的视图区域，若没有这个标志，则会与父视图做并集，最终绘制父视图的区域。    
最后触发 ViewRootImpl 的 invalidateChildInParent：    
```java
    public ViewParent invalidateChildInParent(int[] location, Rect dirty) {
        checkThread();
        if (DEBUG_DRAW) Log.v(mTag, "Invalidate child: " + dirty);
 
        if (dirty == null) { // 表示要重绘当前ViewRootImpl指示的整个区域
            invalidate();
            return null;
        } else if (dirty.isEmpty() && !mIsAnimating) { // 表示不需要重绘
            return null;
        }

        ...

        invalidateRectOnScreen(dirty);

        return null;
    }

    private void invalidateRectOnScreen(Rect dirty) {
        final Rect localDirty = mDirty; // 要重绘的区域
        if (!localDirty.isEmpty() && !localDirty.contains(dirty)) {
            mAttachInfo.mSetIgnoreDirtyState = true;
            mAttachInfo.mIgnoreDirtyState = true;
        }

        // 当前已有的 dirty 区域与此次 dirty 区域做并集
        localDirty.union(dirty.left, dirty.top, dirty.right, dirty.bottom);
        // Intersect with the bounds of the window to skip
        // updates that lie outside of the visible region
        final float appScale = mAttachInfo.mApplicationScale;
        final boolean intersected = localDirty.intersect(0, 0,
                (int) (mWidth * appScale + 0.5f), (int) (mHeight * appScale + 0.5f));
        if (!intersected) {
            localDirty.setEmpty();
        }
        if (!mWillDrawSoon && (intersected || mIsAnimating)) {
            scheduleTraversals();
        }
    }
```
可以看出，它整个过程就是计算所要绘制的区域，然后还是和前面一样走 scheduleTraversals 流程，最终是将绘制操作放到 Choreographer 中。然后去请求监听 VSYNC 信号。信号回来后就会打印掉帧情况，减去偏移时间来纠正帧时间之类的。然后遍历那4个链表来执行回调。最终执行 scheduleTraversals 执行 performDraw。

### 总结
invalidate 方法就是一个去执行 draw 方法的过程，在执行之前它需要去确认所需要绘制的大小。它会通过自己的大小与 parent 的左上角的坐标来算出 View 相对于 parent 的坐标矩阵，然后根据 clip_child 属性来看是与父布局做∩还是∪ 来得到新的要绘制的区域。一直这样回溯到 ViewRootImpl 中计算出最终需要绘制的 dirty 区域，然后和 requestLayout 一样，将执行绘制的 runnable 交给 Choreographer 来处理。最终会调用 ViewRootImpl#performDraw。

## PostInvalidate
就是通过 handler 发送一个消息到 mainlooper 中，然后在主线程中执行 invalidate 方法。

## Choreographer 应用场景
监控App性能、卡顿和帧率。 TinyDancer 就是利用 Choreographer 来检测卡顿的。他就是 post 一个 FrameDataCallback 进去，然后下一个帧时间到来时，计算2帧之间的时间有没有超过 16.6ms，超过了则掉帧。然后它每隔 700ms 统计这段事件内所有掉帧个数，以及连续掉2帧以上的个数。然后计算没有掉帧占用的比例，以及连续掉帧占用的比例然后显示出来。


## 参考文献
+ [Android 源码分析 - View的requestLayout、invalidate和postInvalidate的实现原理](https://www.jianshu.com/p/ce30f200209e)  
+ [View与窗口：AttachInfo](https://blog.csdn.net/savelove911/article/details/51822266) 
+ [invalidate原理](https://www.jianshu.com/p/10a2bbc5d56a)
+ [requestLayout()与invalidate()流程及Choroegrapher类分析](https://www.cnblogs.com/tiger-wang-ms/p/6592189.html)