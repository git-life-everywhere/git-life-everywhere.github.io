# ViewDraw 第六篇 draw 流程分析

title: ViewDraw 第六篇 draw 流程分析
date: 2020/04/21 20:46:25 
catalog: true
categories: 

- View 视图
- View 的加载和绘制

tags: ViewDraw
copyright: true

------

本篇文章基于 Android N - 7.1.1 主要分析下 draw 方法的执行流程；

# 1 回顾

在上面的 performTraversals 文章中，我们知道 perfomDraw 分为硬件绘制和软件绘制，这里我们只看软件绘制：



## 1.1 mHardwareRenderer.draw - 硬件绘制

我们先来看看硬件绘制：

```java
    if (mAttachInfo.mHardwareRenderer != null && mAttachInfo.mHardwareRenderer.isEnabled()) {
        // If accessibility focus moved, always invalidate the root.
        boolean invalidateRoot = accessibilityFocusDirty || mInvalidateRootRequested;
        mInvalidateRootRequested = false;

        // Draw with hardware renderer.
        mIsAnimating = false;

        if (mHardwareYOffset != yOffset || mHardwareXOffset != xOffset) {
            mHardwareYOffset = yOffset;
            mHardwareXOffset = xOffset;
            invalidateRoot = true;
        }

        if (invalidateRoot) {
            mAttachInfo.mHardwareRenderer.invalidateRoot();
        }

        dirty.setEmpty();

        final boolean updated = updateContentDrawBounds();

        if (mReportNextDraw) {
            mAttachInfo.mHardwareRenderer.setStopped(false);
        }

        if (updated) {
            requestDrawWindow();
        }
        //【1】通过 ThreadedRenderer 进行绘制；
        mAttachInfo.mHardwareRenderer.draw(mView, mAttachInfo, this);
    } 
```





## 1.2 drawSoftware - 软件绘制

```java
    private boolean drawSoftware(Surface surface, AttachInfo attachInfo, int xoff, int yoff,
            boolean scalingRequired, Rect dirty) {

        final Canvas canvas;
        try {
            final int left = dirty.left;
            final int top = dirty.top;
            final int right = dirty.right;
            final int bottom = dirty.bottom;
            //【1】锁定 canvas，并返回画布 Canvas；
            canvas = mSurface.lockCanvas(dirty);

            // The dirty rectangle can be modified by Surface.lockCanvas()
            // noinspection ConstantConditions
            if (left != dirty.left || top != dirty.top || right != dirty.right
                    || bottom != dirty.bottom) {
                attachInfo.mIgnoreDirtyState = true;
            }

            // TODO: Do this in native
            canvas.setDensity(mDensity);
        } catch (Surface.OutOfResourcesException e) {
            handleOutOfResourcesException(e);
            return false;
        } catch (IllegalArgumentException e) {
            Log.e(mTag, "Could not lock surface", e);
            // Don't assume this is due to out of memory, it could be
            // something else, and if it is something else then we could
            // kill stuff (or ourself) for no reason.
            mLayoutRequested = true;    // ask wm for a new surface next time.
            return false;
        }

        try {
            if (DEBUG_ORIENTATION || DEBUG_DRAW) {
                Log.v(mTag, "Surface " + surface + " drawing to bitmap w="
                        + canvas.getWidth() + ", h=" + canvas.getHeight());
                //canvas.drawARGB(255, 255, 0, 0);
            }

            // 如果该位图的格式包含 Alpha 通道，则我们需要在绘制之前清除它，以便 child 可以在透明背景上正确地重新组合其图形。
            // 这将自动考虑裁剪区域或者脏区，或者如果我们要应用偏移，则需要清除没有出现偏移的区域，以避免在空白区域中留下垃圾。
            if (!canvas.isOpaque() || yoff != 0 || xoff != 0) {
                canvas.drawColor(0, PorterDuff.Mode.CLEAR);
            }

            dirty.setEmpty(); // 将 dirty 区域置空；
            mIsAnimating = false;
            mView.mPrivateFlags |= View.PFLAG_DRAWN;

            if (DEBUG_DRAW) {
                Context cxt = mView.getContext();
                Log.i(mTag, "Drawing: package:" + cxt.getPackageName() +
                        ", metrics=" + cxt.getResources().getDisplayMetrics() +
                        ", compatibilityInfo=" + cxt.getResources().getCompatibilityInfo());
            }
            try {
                canvas.translate(-xoff, -yoff);
                if (mTranslator != null) {
                    mTranslator.translateCanvas(canvas);
                }
                canvas.setScreenDensity(scalingRequired ? mNoncompatDensity : 0);
                attachInfo.mSetIgnoreDirtyState = false;

                //【2-core】调用 DecorView（GroupView）的 draw 方法；
                mView.draw(canvas);

                drawAccessibilityFocusedDrawableIfNeeded(canvas);
            } finally {
                if (!attachInfo.mSetIgnoreDirtyState) {
                    // Only clear the flag if it was not set during the mView.draw() call
                    attachInfo.mIgnoreDirtyState = false;
                }
            }
        } finally {
            try {
                //【3】解锁 canvas
                surface.unlockCanvasAndPost(canvas);
            } catch (IllegalArgumentException e) {
                Log.e(mTag, "Could not unlock surface", e);
                mLayoutRequested = true;    // ask wm for a new surface next time.
                //noinspection ReturnInsideFinallyBlock
                return false;
            }

            if (LOCAL_LOGV) {
                Log.v(mTag, "Surface " + surface + " unlockCanvasAndPost");
            }
        }
        return true;
    }
```

这里的关键代码就是：

```java
//【2-core】调用 DecorView（GroupView）的 draw 方法；
mView.draw(canvas);
```

这里的 mView 是 DecorView：

# 2 DecorView

## 2.1 draw

DecorView 复写了 draw 方法，但是其内部依然是调用了父类 View 的 draw 方法：

```java
    @Override
    public void draw(Canvas canvas) {
        //【-->4.1】进入 view；
        super.draw(canvas);

        if (mMenuBackground != null) {
            mMenuBackground.draw(canvas);
        }
    }
```



## 2.2 onDraw

DecorView 复写了 view 的 onDraw 方法，但是其内部主要逻辑依然是调用了父类的逻辑：

```java
    @Override
    public void onDraw(Canvas c) {
        //【-->4.1】开启 view 的绘制；
        super.onDraw(c);
        mBackgroundFallback.draw(mContentRoot, c, mWindow.mContentParent);
    }
```



# 3 ViewGroup

## 3.1 dispatchDraw - 核心

ViewGroup 复写了 view 的 dispatchDraw 方法：

```java
    @Override
    protected void dispatchDraw(Canvas canvas) {
        //【0】是否使用 renderNode 属性，实际上是针对于硬件加速的，软件绘制返回的是 false；
        boolean usingRenderNodeProperties = canvas.isRecordingFor(mRenderNode);
        final int childrenCount = mChildrenCount;
        final View[] children = mChildren;
        int flags = mGroupFlags;
        //【1】判断是否有动画，如果有动画，同时 child view 是可见的 VISIBLE，那么这里会设置动画参数；
        // FLAG_RUN_ANIMATION 标志位表示要运行动画。
        if ((flags & FLAG_RUN_ANIMATION) != 0 && canAnimate()) {
            final boolean buildCache = !isHardwareAccelerated();
            for (int i = 0; i < childrenCount; i++) {
                final View child = children[i];
                if ((child.mViewFlags & VISIBILITY_MASK) == VISIBLE) {
                    //【1.1】动画的参数是保存在 view 的 LayoutParams 中的；
                    final LayoutParams params = child.getLayoutParams();
                    attachLayoutAnimationParameters(child, params, i, childrenCount);
                    bindLayoutAnimation(child);
                }
            }
            //【1.2】将布局动画器的应用保存到 controller，并启动动画；
            // 布局动画控制器用于对布局或视图组的 child view 进行动画处理。每个孩子都使用相同的动画，但是对于每个孩子，动画在不同的时间开始。
            final LayoutAnimationController controller = mLayoutAnimationController;
            if (controller.willOverlap()) {
                mGroupFlags |= FLAG_OPTIMIZE_INVALIDATE;
            }

            controller.start();
            //【1.3】去掉上面的 FLAG_RUN_ANIMATION 标志位，因为动画开始了，同时要去掉 FLAG_ANIMATION_DONE 标志位
            // FLAG_ANIMATION_DONE 表示动画结束了或者没有动画；
            mGroupFlags &= ~FLAG_RUN_ANIMATION;
            mGroupFlags &= ~FLAG_ANIMATION_DONE;

            //【1.4】如果有动画监听器，那就回调，显然，这个是我们要设置进去的；
            if (mAnimationListener != null) {
                mAnimationListener.onAnimationStart(controller.getAnimation());
            }
        }

        int clipSaveCount = 0;
        //【2】这里判断了下 viewgroup 是否有 padding 区域，如果有的话，那么绘制的区域就要减去 padding
        // 所以下面 save 的画布的状态，裁剪了 canvas，去掉了 padding 区域；
        final boolean clipToPadding = (flags & CLIP_TO_PADDING_MASK) == CLIP_TO_PADDING_MASK;
        if (clipToPadding) {
            clipSaveCount = canvas.save(Canvas.CLIP_SAVE_FLAG);
            canvas.clipRect(mScrollX + mPaddingLeft, mScrollY + mPaddingTop,
                    mScrollX + mRight - mLeft - mPaddingRight,
                    mScrollY + mBottom - mTop - mPaddingBottom);
        }

        //【3】这里又去掉了 PFLAG_DRAW_ANIMATION 和 FLAG_INVALIDATE_REQUIRED 标志位；
        // 如果设置了 FLAG_INVALIDATE_REQUIRED ，dispatchDraw 将调用 invalidate，
        // 当一个 child 需要 invalidate 并且设置了 FLAG_OPTIMIZE_INVALIDATE 时，
        // drawChild 会设置该标志位；
        mPrivateFlags &= ~PFLAG_DRAW_ANIMATION;
        mGroupFlags &= ~FLAG_INVALIDATE_REQUIRED;

        boolean more = false;
        final long drawingTime = getDrawingTime();

        if (usingRenderNodeProperties) canvas.insertReorderBarrier();
        final int transientCount = mTransientIndices == null ? 0 : mTransientIndices.size();
        int transientIndex = transientCount != 0 ? 0 : -1;

        //【4】这里的意思：如果没有开启硬件加速（HW accelerated），那么会对 child 的绘制顺序做一个预排列 preorderedList
        // 如果开启硬件加速，那么这里无需排列，因为硬件管道会在内部进行重新排序；
        final ArrayList<View> preorderedList = usingRenderNodeProperties // 软件绘制返回 false
                ? null : buildOrderedChildList(); //【-->3.1.1】这里是会对 view 预排列；
        //【5】如果开启了硬件加速（没有预排列），同时我们指定了 ViewGroup 按照 getChildDrawingOrder 定义的顺序绘制其子级 view
        // 是否按照指定顺序，看是否指定了 FLAG_USE_CHILD_DRAWING_ORDER 标志位；
        final boolean customOrder = preorderedList == null
                && isChildrenDrawingOrderEnabled();
        for (int i = 0; i < childrenCount; i++) {
            while (transientIndex >= 0 && mTransientIndices.get(transientIndex) == i) {
                final View transientChild = mTransientViews.get(transientIndex);
                if ((transientChild.mViewFlags & VISIBILITY_MASK) == VISIBLE ||
                        transientChild.getAnimation() != null) {
                    //【-->3.2】绘制 transientChild（可能是 view/viewGroup）
                    more |= drawChild(canvas, transientChild, drawingTime);
                }
                transientIndex++;
                if (transientIndex >= transientCount) {
                    transientIndex = -1;
                }
            }

            //【6】这里就是绘制 child view 了，这里会对 child 的绘制顺序做调整，有一个容器顺序和绘制顺序的不同；
            //【-->3.1.1】获取调整后的绘制顺序；
            final int childIndex = getAndVerifyPreorderedIndex(childrenCount, i, customOrder);
            //【-->3.1.2】根据调整后的绘制顺序返回 view；
            final View child = getAndVerifyPreorderedView(preorderedList, children, childIndex);
            if ((child.mViewFlags & VISIBILITY_MASK) == VISIBLE || child.getAnimation() != null) {
                //【-->3.2】绘制 child（可能是 view/viewGroup）
                more |= drawChild(canvas, child, drawingTime);
            }
        }

        while (transientIndex >= 0) {
            final View transientChild = mTransientViews.get(transientIndex);
            if ((transientChild.mViewFlags & VISIBILITY_MASK) == VISIBLE ||
                    transientChild.getAnimation() != null) {
                //【-->3.2】绘制额外的 transient child view（可能是 view/viewGroup）
                more |= drawChild(canvas, transientChild, drawingTime);
            }
            transientIndex++;
            if (transientIndex >= transientCount) {
                break;
            }
        }
        if (preorderedList != null) preorderedList.clear();

        //【7】绘制所有的正在消失的有动画的 view；
        if (mDisappearingChildren != null) {
            final ArrayList<View> disappearingChildren = mDisappearingChildren;
            final int disappearingCount = disappearingChildren.size() - 1;
            for (int i = disappearingCount; i >= 0; i--) {
                final View child = disappearingChildren.get(i);
                //【-->3.2】绘制 child（可能是 view/viewGroup）
                more |= drawChild(canvas, child, drawingTime);
            }
        }
        if (usingRenderNodeProperties) canvas.insertInorderBarrier();

        if (debugDraw()) {
            onDebugDraw(canvas);
        }
        //【8】如果前面针对 padding 做了画布裁剪，这里就恢复它；
        if (clipToPadding) {
            canvas.restoreToCount(clipSaveCount);
        }

        //【9】mGroupFlags 可能会被 drawChild() 更改，所以这里将副本保存到 flags 中；
        flags = mGroupFlags;

        if ((flags & FLAG_INVALIDATE_REQUIRED) == FLAG_INVALIDATE_REQUIRED) {
            invalidate(true);
        }
        //【10】判断完成了吗
        if ((flags & FLAG_ANIMATION_DONE) == 0 && (flags & FLAG_NOTIFY_ANIMATION_LISTENER) == 0 &&
                mLayoutAnimationController.isDone() && !more) {
            //【10。1】设置 FLAG_NOTIFY_ANIMATION_LISTENER，调用 mAnimationListener.onAnimationEnd() 并在必要时删除子级Bitmap缓存。
            // 当布局动画结束时（设置FLAG_ANIMATION_DONE之后）设置此标志。
            mGroupFlags |= FLAG_NOTIFY_ANIMATION_LISTENER;
            final Runnable end = new Runnable() {
                @Override
                public void run() {
                    notifyAnimationListener(); // 这里面会调用 mAnimationListener.onAnimationEnd()
                }
            };
            post(end);
        }
    }
```



### 3.1.1 buildOrderedChildList

该方法会返回 child view 的预排序列表 mPreSortedChildren，首先按 Z 排序，然后按 child view 绘制顺序排序（如果适用）。

使用后必须清除此列表，以免泄漏子视图。

使用稳定的插入排序，通常为 O（n）时间复杂度。

```java
    ArrayList<View> buildOrderedChildList() {
        final int childrenCount = mChildrenCount;
        //【1】如果 child 不超过 2 个，或者没有 view 设置了 android:elevation android:translationZ
        // 那就无需排序；
        if (childrenCount <= 1 || !hasChildWithZ()) return null;
       
        if (mPreSortedChildren == null) {
            mPreSortedChildren = new ArrayList<>(childrenCount);
        } else {
            // callers should clear, so clear shouldn't be necessary, but for safety...
            mPreSortedChildren.clear();
            mPreSortedChildren.ensureCapacity(childrenCount);
        }
        //【-->3.1.1.1】判断是否开启自定义绘制顺序；
        final boolean customOrder = isChildrenDrawingOrderEnabled();
        for (int i = 0; i < childrenCount; i++) {
            //【-->3.1.2】获取绘制顺序（可能被更改）
            final int childIndex = getAndVerifyPreorderedIndex(childrenCount, i, customOrder);
            final View nextChild = mChildren[childIndex];
            final float currentZ = nextChild.getZ();

            // 将 Z 值越大的 view 查到 list 的前面，其他的相对位置不变；
            int insertIndex = i;
            while (insertIndex > 0 && mPreSortedChildren.get(insertIndex - 1).getZ() > currentZ) {
                insertIndex--;
            }
            mPreSortedChildren.add(insertIndex, nextChild);
        }
        return mPreSortedChildren;
    }
```

每个 view 内部都有一个，它包含 View 属性（软硬都用），也可能包含 View 内容的 DisplayList（硬件加速使用）。

```java
 RenderNode mRenderNode
```

我们看看 hasChildWithZ ：

```java
    // Child views of this ViewGroup
    private View[] mChildren;

	private boolean hasChildWithZ() {
        for (int i = 0; i < mChildrenCount; i++) {
            //【1】这里调用了 View 的 getZ 方法：
            if (mChildren[i].getZ() != 0) return true;
        }
        return false;
    }
```

判断该 View 的是否有可视 z 位置（以像素为单位）。该方法等价于  setTranslationZ + getElevation 属性；

```java
    @ViewDebug.ExportedProperty(category = "drawing")
    public float getZ() {
        return getElevation() + getTranslationZ();
    }
```

也就是 android:elevation 和 android:translationZ 

#### 3.1.1.1 isChildrenDrawingOrderEnabled

判断下 ViewGroup 是否按照 getChildDrawingOrder 返回的方式绘制 view：

```java
    @ViewDebug.ExportedProperty(category = "drawing")
    protected boolean isChildrenDrawingOrderEnabled() {
        return (mGroupFlags & FLAG_USE_CHILD_DRAWING_ORDER) == FLAG_USE_CHILD_DRAWING_ORDER;
    }
```

### 3.1.2 getAndVerifyPreorderedIndex

获取 view 实际绘制顺序；

```java
    private int getAndVerifyPreorderedIndex(int childrenCount, int i, boolean customOrder) {
        final int childIndex;
        if (customOrder) {
            //【-->3.1.2】如果 view group 重写了绘制顺序，那么这会获取重写后的绘制顺序；
            final int childIndex1 = getChildDrawingOrder(childrenCount, i);
            if (childIndex1 >= childrenCount) {
                throw new IndexOutOfBoundsException("getChildDrawingOrder() "
                        + "returned invalid index " + childIndex1
                        + " (child count is " + childrenCount + ")");
            }
            childIndex = childIndex1;
        } else {
            childIndex = i;
        }
        return childIndex;
    }
```



### 3.1.3 getChildDrawingOrder

修改绘制顺序，如果要更改子代的绘制顺序，ViewGroup 要重写此方法。默认情况下，它返回 drawingPosition！

想要这个方法生效，那么就要先调用：setChildrenDrawingOrderEnabled

参数：drawingPosition，当前的绘制顺序；

```java
    protected int getChildDrawingOrder(int childCount, int drawingPosition) {
        return drawingPosition;
    }
```



## 3.2 drawChild - 核心

绘制 child view。

这个是直接翻译的注视：此方法负责使画布处于正确的状态。这包括剪切，平移以使孩子的滚动原点位于 0、0，并应用任何动画转换。

可以看出子至少有个作用：

1、修正 canvas 的自己状态；2、动画处理；

```java
    protected boolean drawChild(Canvas canvas, View child, long drawingTime) {
        //【-->4.2】绘制 child 开始了，调用了每个 child 的 draw 方法；
        return child.draw(canvas, this, drawingTime);
    }
```

绘制 child 的逻辑是在 view 的 draw[3] 里面；



## 3.3 dispatchGetDisplayList - 核心

ViewGroup 通知 child view 重新创建自身的 DisplayList：

```java
    @Override
    protected void dispatchGetDisplayList() {
        final int count = mChildrenCount;
        final View[] children = mChildren;
        for (int i = 0; i < count; i++) {
            final View child = children[i];
            if (((child.mViewFlags & VISIBILITY_MASK) == VISIBLE || child.getAnimation() != null)) {
                //【-->3.3.1】对于每个可见或者有动画的 child view，创建其 DisplayList！
                recreateChildDisplayList(child);
            }
        }
        if (mOverlay != null) {
            View overlayView = mOverlay.getOverlayView();
            recreateChildDisplayList(overlayView);
        }
        if (mDisappearingChildren != null) {
            final ArrayList<View> disappearingChildren = mDisappearingChildren;
            final int disappearingCount = disappearingChildren.size();
            for (int i = 0; i < disappearingCount; ++i) {
                final View child = disappearingChildren.get(i);
                recreateChildDisplayList(child);
            }
        }
    }
```





### 3.3.1 recreateChildDisplayList

```java
    private void recreateChildDisplayList(View child) {
        child.mRecreateDisplayList = (child.mPrivateFlags & PFLAG_INVALIDATED) != 0;
        child.mPrivateFlags &= ~PFLAG_INVALIDATED;
        //【-->4.2.1】这里又调用了 updateDisplayListIfDirty 方法：
        child.updateDisplayListIfDirty();
        child.mRecreateDisplayList = false;
    }
```



# 4 View

## 4.1 draw[1] - 核心

View 的方法是最核心的部分：

将 View（及其所有 child）绘制（渲染）到给定的 Canvas。 在调用此函数之前，View 必须完成测量和布局。

自定义 View 时，请实现 onDraw 而不要重写 draw（draw 方法会调用 onDraw）。

 如果确实需要重写 draw，一定要调用 super.draw!!

```java
    @CallSuper
    public void draw(Canvas canvas) {
        final int privateFlags = mPrivateFlags;
        final boolean dirtyOpaque = (privateFlags & PFLAG_DIRTY_MASK) == PFLAG_DIRTY_OPAQUE &&
                (mAttachInfo == null || !mAttachInfo.mIgnoreDirtyState);
        mPrivateFlags = (privateFlags & ~PFLAG_DIRTY_MASK) | PFLAG_DRAWN;

        int saveCount;

        //【1】绘制背景：
        if (!dirtyOpaque) {
            //【-->4.1.1】绘制背景；
            drawBackground(canvas);
        }
        
        //【2】是否跳过 2 到 5 的步骤，这里判断了 ViewFlags 是否设置了 FADING_EDGE_HORIZONTAL/FADING_EDGE_VERTICAL
        // 这两个标志位表示，如果 view 滚动的话，那么他的边缘会变淡（淡入淡出效果）；
        final int viewFlags = mViewFlags;
        boolean horizontalEdges = (viewFlags & FADING_EDGE_HORIZONTAL) != 0;
        boolean verticalEdges = (viewFlags & FADING_EDGE_VERTICAL) != 0;
        if (!verticalEdges && !horizontalEdges) {
            //【-->4.1.2】绘制当前 view 的内容；
            if (!dirtyOpaque) onDraw(canvas);

            //【-->4.1.2】如果是 view 的话，只是一个空实现；
            //【-->3.1】如果是 ViewGroup 的话，尝试绘制 child view 的内容；
            dispatchDraw(canvas);

            //【5】绘制叠加层；
            if (mOverlay != null && !mOverlay.isEmpty()) {
                mOverlay.getOverlayView().dispatchDraw(canvas);
            }

            //【-->4.1.4】绘制装饰前景（前景，滚动条等等）
            onDrawForeground(canvas);

            // 正常来说，到这里绘制就结束了；
            return;
        }

        //【7】如果设置了 fade 效果的话，那么上面会跳过一系列的绘制操作；
        // 因为下面会特殊的处理；
        boolean drawTop = false;
        boolean drawBottom = false;
        boolean drawLeft = false;
        boolean drawRight = false;

        float topFadeStrength = 0.0f;
        float bottomFadeStrength = 0.0f;
        float leftFadeStrength = 0.0f;
        float rightFadeStrength = 0.0f;

        //【8】如果是设置了 fade 效果的话，下面会对 canvas 的 layer 图层进行保存；
        // 下面先是计算了下要绘制的边界距离；
        int paddingLeft = mPaddingLeft;

        final boolean offsetRequired = isPaddingOffsetRequired();
        if (offsetRequired) {
            paddingLeft += getLeftPaddingOffset();
        }

        int left = mScrollX + paddingLeft;
        int right = left + mRight - mLeft - mPaddingRight - paddingLeft;
        int top = mScrollY + getFadeTop(offsetRequired);
        int bottom = top + getFadeHeight(offsetRequired);

        if (offsetRequired) {
            right += getRightPaddingOffset();
            bottom += getBottomPaddingOffset();
        }

        final ScrollabilityCache scrollabilityCache = mScrollCache;
        final float fadeHeight = scrollabilityCache.fadingEdgeLength;
        int length = (int) fadeHeight;

        // clip the fade length if top and bottom fades overlap
        // overlapping fades produce odd-looking artifacts
        if (verticalEdges && (top + length > bottom - length)) {
            length = (bottom - top) / 2;
        }

        // also clip horizontal fades if necessary
        if (horizontalEdges && (left + length > right - length)) {
            length = (right - left) / 2;
        }
        //【9】判断下四个方向是否要绘制效果；
        if (verticalEdges) {
            topFadeStrength = Math.max(0.0f, Math.min(1.0f, getTopFadingEdgeStrength()));
            drawTop = topFadeStrength * fadeHeight > 1.0f;
            bottomFadeStrength = Math.max(0.0f, Math.min(1.0f, getBottomFadingEdgeStrength()));
            drawBottom = bottomFadeStrength * fadeHeight > 1.0f;
        }

        if (horizontalEdges) {
            leftFadeStrength = Math.max(0.0f, Math.min(1.0f, getLeftFadingEdgeStrength()));
            drawLeft = leftFadeStrength * fadeHeight > 1.0f;
            rightFadeStrength = Math.max(0.0f, Math.min(1.0f, getRightFadingEdgeStrength()));
            drawRight = rightFadeStrength * fadeHeight > 1.0f;
        }

        saveCount = canvas.getSaveCount();

        int solidColor = getSolidColor();
        if (solidColor == 0) {
            final int flags = Canvas.HAS_ALPHA_LAYER_SAVE_FLAG;
            //【10】这里开始保存了画布的 layer，length 表示这个 fade 区域的宽度。
            // 这里相当于创建了一个新的 layer 并入栈，四个坐标构成了一个矩形区域；
            if (drawTop) {
                canvas.saveLayer(left, top, right, top + length, null, flags);
            }

            if (drawBottom) {
                canvas.saveLayer(left, bottom - length, right, bottom, null, flags);
            }

            if (drawLeft) {
                canvas.saveLayer(left, top, left + length, bottom, null, flags);
            }

            if (drawRight) {
                canvas.saveLayer(right - length, top, right, bottom, null, flags);
            }
        } else {
            scrollabilityCache.setFadeColor(solidColor);
        }

        //【-->4.1.2】保存完 layer 后，我们先绘制当前 view 的内容；
        if (!dirtyOpaque) onDraw(canvas);

        //【-->4.1.3】然后绘制 child 的内容；
        dispatchDraw(canvas);

        //【11】绘制淡入淡出效果，并还原图层，淡入淡出实际上是在另外一个 layer 上绘制的；
        final Paint p = scrollabilityCache.paint;
        final Matrix matrix = scrollabilityCache.matrix;
        final Shader fade = scrollabilityCache.shader;

        if (drawTop) {
            matrix.setScale(1, fadeHeight * topFadeStrength);
            matrix.postTranslate(left, top);
            fade.setLocalMatrix(matrix);
            p.setShader(fade);
            canvas.drawRect(left, top, right, top + length, p);
        }

        if (drawBottom) {
            matrix.setScale(1, fadeHeight * bottomFadeStrength);
            matrix.postRotate(180);
            matrix.postTranslate(left, bottom);
            fade.setLocalMatrix(matrix);
            p.setShader(fade);
            canvas.drawRect(left, bottom - length, right, bottom, p);
        }

        if (drawLeft) {
            matrix.setScale(1, fadeHeight * leftFadeStrength);
            matrix.postRotate(-90);
            matrix.postTranslate(left, top);
            fade.setLocalMatrix(matrix);
            p.setShader(fade);
            canvas.drawRect(left, top, left + length, bottom, p);
        }

        if (drawRight) {
            matrix.setScale(1, fadeHeight * rightFadeStrength);
            matrix.postRotate(90);
            matrix.postTranslate(right, top);
            fade.setLocalMatrix(matrix);
            p.setShader(fade);
            canvas.drawRect(right - length, top, right, bottom, p);
        }

        canvas.restoreToCount(saveCount); // 还原图层；

        //【12】绘制叠加层；
        if (mOverlay != null && !mOverlay.isEmpty()) {
            mOverlay.getOverlayView().dispatchDraw(canvas);
        }

        //【13】绘制装饰前景（前景，滚动条等等）
        onDrawForeground(canvas);
    }
```
从注视我们可以看出，绘制过程有如下的绘制步骤，这些绘制步骤必须以适当的顺序执行：

1、绘制背景 
2、如有必要，保存画布的图层以准备淡入淡出 
3、绘制视图的内容
4、绘制 child 
5、如有必要，绘制褪色边缘并还原图层 
6、绘制装饰（例如滚动条）

对于增加了淡入淡出效果情况，也是很类似，只不过增加了 savelayer 和 restore 的操作；

### 4.1.1 drawBackground

```java
    private void drawBackground(Canvas canvas) {
        final Drawable background = mBackground;
        if (background == null) {
            return;
        }

        setBackgroundBounds();

        //【1】如果是硬件加速的话，通过 renderNode 记录绘制操作；
        if (canvas.isHardwareAccelerated() && mAttachInfo != null
                && mAttachInfo.mHardwareRenderer != null) {
            mBackgroundRenderNode = getDrawableRenderNode(background, mBackgroundRenderNode);

            final RenderNode renderNode = mBackgroundRenderNode;
            if (renderNode != null && renderNode.isValid()) {
                setBackgroundRenderNodeProperties(renderNode);
                //【1.1】可以看到硬件加速的 canvas 是 DisplayListCanvas 类型的；
                ((DisplayListCanvas) canvas).drawRenderNode(renderNode);
                return;
            }
        }

        final int scrollX = mScrollX;
        final int scrollY = mScrollY;
        //【2】软件绘制，就是调用 Drawable.draw 方法；
        if ((scrollX | scrollY) == 0) {
            background.draw(canvas);
        } else {
            canvas.translate(scrollX, scrollY);
            background.draw(canvas);
            canvas.translate(-scrollX, -scrollY);
        }
    }
```

不多说了；



### 4.1.2 onDraw - 核心

默认是空方法，需要自己实现；

```java
protected void onDraw(Canvas canvas) {
}
```

这个方法是绘制 background 的；

### 4.1.3 dispatchDraw - 核心

默认是空方法，需要自己实现；

这个方法由 draw 调用，用于绘制 child view，一般来说 view group 肯定是需要重写这个方法，在 onDraw 方法调用后再调用；

```java
protected void dispatchDraw(Canvas canvas) {

}
```



### 4.1.4 onDrawForeground

绘制 view 的所有前景内容。

前景内容可能包括滚动条、可绘制对象或其他特定于视图的装饰。前景会被绘制在 view 内容的顶部。

前景可以通过 setForeground 方法设置；

```java
    public void onDrawForeground(Canvas canvas) {
        onDrawScrollIndicators(canvas);
        onDrawScrollBars(canvas);
        //【1】依然是获取 Drawable，然后将 Drawable 会知道 canvas 上去；
        final Drawable foreground = mForegroundInfo != null ? mForegroundInfo.mDrawable : null;
        if (foreground != null) {
            if (mForegroundInfo.mBoundsChanged) {
                mForegroundInfo.mBoundsChanged = false;
                final Rect selfBounds = mForegroundInfo.mSelfBounds;
                final Rect overlayBounds = mForegroundInfo.mOverlayBounds;

                if (mForegroundInfo.mInsidePadding) {
                    selfBounds.set(0, 0, getWidth(), getHeight());
                } else {
                    selfBounds.set(getPaddingLeft(), getPaddingTop(),
                            getWidth() - getPaddingRight(), getHeight() - getPaddingBottom());
                }

                final int ld = getLayoutDirection();
                Gravity.apply(mForegroundInfo.mGravity, foreground.getIntrinsicWidth(),
                        foreground.getIntrinsicHeight(), selfBounds, overlayBounds, ld);
                foreground.setBounds(overlayBounds);
            }
            //【1】核心逻辑，软件绘制；
            foreground.draw(canvas);
        }
    }
```





## 4.2 draw[3] - 核心

 这个方法由 ViewGroup.drawChild() 触发，绘制 child；

其实可以看到，无论是软件绘制，还是硬件绘制，都会进入这个方法，最终都会调用到`draw(Canvas canvas)`方法来绘制，

```java
boolean draw(Canvas canvas, ViewGroup parent, long drawingTime) {
    //【1】判断是否开启了硬件加速，软件绘制返回 false，硬件绘制对应的是 DisplayListCanvas 返回的是 true；
    final boolean hardwareAcceleratedCanvas = canvas.isHardwareAccelerated();

    //【2】如果使用的是硬件加速，那么是通过 RenderNode + DisplayList 的方式绘制，那么 drawingWithRenderNode 为 true；
    //（对于没有 attach 的 view（不需要显示），他是没有 DisplayList；如果不是通过硬件减速的话，那么不会通过 RenderNode 绘制）
    boolean drawingWithRenderNode = mAttachInfo != null
        && mAttachInfo.mHardwareAccelerated
        && hardwareAcceleratedCanvas;

    boolean more = false;
    final boolean childHasIdentityMatrix = hasIdentityMatrix();
    final int parentFlags = parent.mGroupFlags;

    if ((parentFlags & ViewGroup.FLAG_CLEAR_TRANSFORMATION) != 0) {
        parent.getChildTransformation().clear();
        parent.mGroupFlags &= ~ViewGroup.FLAG_CLEAR_TRANSFORMATION;
    }

    Transformation transformToApply = null;
    boolean concatMatrix = false;
    //【3】对动画的一些处理；
    final boolean scalingRequired = mAttachInfo != null && mAttachInfo.mScalingRequired;
    final Animation a = getAnimation(); // 获取当前 view 关联的动画；
    if (a != null) {
        more = applyLegacyAnimation(parent, drawingTime, a, scalingRequired);
        concatMatrix = a.willChangeTransformationMatrix();
        if (concatMatrix) {
            mPrivateFlags3 |= PFLAG3_VIEW_IS_ANIMATING_TRANSFORM;
        }
        transformToApply = parent.getChildTransformation();
    } else { // 没有动画的话，就就清除掉动画相关的数据 matrix；
        if ((mPrivateFlags3 & PFLAG3_VIEW_IS_ANIMATING_TRANSFORM) != 0) {
            mRenderNode.setAnimationMatrix(null);
            mPrivateFlags3 &= ~PFLAG3_VIEW_IS_ANIMATING_TRANSFORM;
        }
        if (!drawingWithRenderNode
            && (parentFlags & ViewGroup.FLAG_SUPPORT_STATIC_TRANSFORMATIONS) != 0) {
            final Transformation t = parent.getChildTransformation();
            final boolean hasTransform = parent.getChildStaticTransformation(this, t);
            if (hasTransform) {
                final int transformType = t.getTransformationType();
                transformToApply = transformType != Transformation.TYPE_IDENTITY ? t : null;
                concatMatrix = (transformType & Transformation.TYPE_MATRIX) != 0;
            }
        }
    }

    concatMatrix |= !childHasIdentityMatrix;

    //【4】设置该标志位，draw 才能成功调用 invalidate 方法；
    mPrivateFlags |= PFLAG_DRAWN;

    if (!concatMatrix &&
        (parentFlags & (ViewGroup.FLAG_SUPPORT_STATIC_TRANSFORMATIONS |
                        ViewGroup.FLAG_CLIP_CHILDREN)) == ViewGroup.FLAG_CLIP_CHILDREN &&
        canvas.quickReject(mLeft, mTop, mRight, mBottom, Canvas.EdgeType.BW) &&
        (mPrivateFlags & PFLAG_DRAW_ANIMATION) == 0) {
        mPrivateFlags2 |= PFLAG2_VIEW_QUICK_REJECTED;
        return more;
    }
    mPrivateFlags2 &= ~PFLAG2_VIEW_QUICK_REJECTED;

    //【5】如果是使用的硬件加速，那么会处理下 INVALIDATED 标记；
    if (hardwareAcceleratedCanvas) {
        //【5.1】清除 PFLAG_INVALIDATED 标志，让 invalidate 在渲染过程中发生，但将标志的值临时保留在 mRecreateDisplayList 标志中
        mRecreateDisplayList = (mPrivateFlags & PFLAG_INVALIDATED) != 0;
        mPrivateFlags &= ~PFLAG_INVALIDATED;
    }

    RenderNode renderNode = null; // 硬件绘制用到的渲染节点（gpu）
    Bitmap cache = null; // 软件绘制用到的绘制缓存（cpu）
    
    //【6】针对 layer 做处理；
    int layerType = getLayerType();
    if (layerType == LAYER_TYPE_SOFTWARE || !drawingWithRenderNode) {
        if (layerType != LAYER_TYPE_NONE) {
            // 如果不是硬件绘制，也没制定 layer 类型，则将 HW 图层视为 SW，同时创建绘制缓存；
            layerType = LAYER_TYPE_SOFTWARE;
            //【-->4.2.2】创建绘制缓存；
            buildDrawingCache(true);
        }
        //【-->4.2.3】获取绘制缓存；
        cache = getDrawingCache(true);
    }
    //【7】如果是硬件绘制的话，这里会对 display list 做处理；
    if (drawingWithRenderNode) {
        //【-->4.2.1】更新 gpu 绘制列表，保存在 RenderNode 中；
        renderNode = updateDisplayListIfDirty();
        if (!renderNode.isValid()) {
            // 如果在调用 getDisplayList() 的过程中将 View 从层次结构中删除，
            // 显示列表将被标记为无效，那么这就不会是使用 renderNode 绘制；
            renderNode = null;
            drawingWithRenderNode = false;
        }
    }

    int sx = 0;
    int sy = 0;
    if (!drawingWithRenderNode) {
        computeScroll();
        sx = mScrollX;
        sy = mScrollY;
    }
    //【8】判断下是否使用缓存进行绘制（不是硬件绘制，可能是上面的 renderNode inValid 了，同时缓存不为 null）
    final boolean drawingWithDrawingCache = cache != null && !drawingWithRenderNode;
    final boolean offsetForScroll = cache == null && !drawingWithRenderNode;

    int restoreTo = -1;
    if (!drawingWithRenderNode || transformToApply != null) {
        restoreTo = canvas.save();
    }
    if (offsetForScroll) {
        canvas.translate(mLeft - sx, mTop - sy);
    } else {
        if (!drawingWithRenderNode) {
            canvas.translate(mLeft, mTop);
        }
        if (scalingRequired) {
            if (drawingWithRenderNode) {
                // TODO: Might not need this if we put everything inside the DL
                restoreTo = canvas.save();
            }
            // mAttachInfo cannot be null, otherwise scalingRequired == false
            final float scale = 1.0f / mAttachInfo.mApplicationScale;
            canvas.scale(scale, scale);
        }
    }
    //【9】处理 alpha 的变化；
    float alpha = drawingWithRenderNode ? 1 : (getAlpha() * getTransitionAlpha());
    if (transformToApply != null
        || alpha < 1
        || !hasIdentityMatrix()
        || (mPrivateFlags3 & PFLAG3_VIEW_IS_ANIMATING_ALPHA) != 0) {
        if (transformToApply != null || !childHasIdentityMatrix) {
            int transX = 0;
            int transY = 0;

            if (offsetForScroll) {
                transX = -sx;
                transY = -sy;
            }

            if (transformToApply != null) {
                if (concatMatrix) {
                    if (drawingWithRenderNode) {
                        renderNode.setAnimationMatrix(transformToApply.getMatrix());
                    } else {
                        // Undo the scroll translation, apply the transformation matrix,
                        // then redo the scroll translate to get the correct result.
                        canvas.translate(-transX, -transY);
                        canvas.concat(transformToApply.getMatrix());
                        canvas.translate(transX, transY);
                    }
                    parent.mGroupFlags |= ViewGroup.FLAG_CLEAR_TRANSFORMATION;
                }

                float transformAlpha = transformToApply.getAlpha();
                if (transformAlpha < 1) {
                    alpha *= transformAlpha;
                    parent.mGroupFlags |= ViewGroup.FLAG_CLEAR_TRANSFORMATION;
                }
            }

            if (!childHasIdentityMatrix && !drawingWithRenderNode) {
                canvas.translate(-transX, -transY);
                canvas.concat(getMatrix());
                canvas.translate(transX, transY);
            }
        }

        // Deal with alpha if it is or used to be <1
        if (alpha < 1 || (mPrivateFlags3 & PFLAG3_VIEW_IS_ANIMATING_ALPHA) != 0) {
            if (alpha < 1) {
                mPrivateFlags3 |= PFLAG3_VIEW_IS_ANIMATING_ALPHA;
            } else {
                mPrivateFlags3 &= ~PFLAG3_VIEW_IS_ANIMATING_ALPHA;
            }
            parent.mGroupFlags |= ViewGroup.FLAG_CLEAR_TRANSFORMATION;
            if (!drawingWithDrawingCache) {
                final int multipliedAlpha = (int) (255 * alpha);
                if (!onSetAlpha(multipliedAlpha)) {
                    if (drawingWithRenderNode) {
                        renderNode.setAlpha(alpha * getAlpha() * getTransitionAlpha());
                    } else if (layerType == LAYER_TYPE_NONE) {
                        canvas.saveLayerAlpha(sx, sy, sx + getWidth(), sy + getHeight(),
                                              multipliedAlpha);
                    }
                } else {
                    // Alpha is handled by the child directly, clobber the layer's alpha
                    mPrivateFlags |= PFLAG_ALPHA_SET;
                }
            }
        }
    } else if ((mPrivateFlags & PFLAG_ALPHA_SET) == PFLAG_ALPHA_SET) {
        onSetAlpha(255);
        mPrivateFlags &= ~PFLAG_ALPHA_SET;
    }

    //【10】对于软件绘制的情况，处理 clip
    if (!drawingWithRenderNode) {
        if ((parentFlags & ViewGroup.FLAG_CLIP_CHILDREN) != 0 && cache == null) {
            if (offsetForScroll) {
                canvas.clipRect(sx, sy, sx + getWidth(), sy + getHeight());
            } else {
                if (!scalingRequired || cache == null) {
                    canvas.clipRect(0, 0, getWidth(), getHeight());
                } else {
                    canvas.clipRect(0, 0, cache.getWidth(), cache.getHeight());
                }
            }
        }

        if (mClipBounds != null) {
            // clip bounds ignore scroll
            canvas.clipRect(mClipBounds);
        }
    }

    //【11】如果不使用绘制缓存的话，会进行不同的处理；
    if (!drawingWithDrawingCache) {
        //【11.1】如果是硬件加速，这里会调用 DisplayListCanvas.drawRenderNode 方法，根据
        // 收集到的 renderNode 树进行绘制；
        if (drawingWithRenderNode) {
            mPrivateFlags &= ~PFLAG_DIRTY_MASK;
            ((DisplayListCanvas) canvas).drawRenderNode(renderNode);
            
        } else {
            //【11.2】对于软件绘制，这里会判断下当前的 view 是否要跳过绘制；
            // 这里可能是硬件绘制被取消了，也可能就是软件绘制～～
            if ((mPrivateFlags & PFLAG_SKIP_DRAW) == PFLAG_SKIP_DRAW) {
                mPrivateFlags &= ~PFLAG_DIRTY_MASK;
                //【-->4.1.3】如果跳过绘制，那就会触发 dispatchDraw 方法；
                dispatchDraw(canvas);
            } else {
                //【-->4.1】如果需要绘制，就绘制自身，如果是 ViewGroup 的话，还会触发 dispatchDraw 方法；
                draw(canvas);
            }
        }
    } else if (cache != null) {
        //【10.3】这里是使用绘制缓存，必须是软件绘制才行；
        mPrivateFlags &= ~PFLAG_DIRTY_MASK;
        if (layerType == LAYER_TYPE_NONE || mLayerPaint == null) {
            // no layer paint, use temporary paint to draw bitmap
            Paint cachePaint = parent.mCachePaint;
            if (cachePaint == null) {
                cachePaint = new Paint();
                cachePaint.setDither(false);
                parent.mCachePaint = cachePaint;
            }
            cachePaint.setAlpha((int) (alpha * 255));
            //【10.4】把存储 cpu 绘制缓存的 Bitmap 绘制到 canvas 上 ( skia 渲染引擎 )
            // 下面也是一样的；
            canvas.drawBitmap(cache, 0.0f, 0.0f, cachePaint);
        } else {
            // use layer paint to draw the bitmap, merging the two alphas, but also restore
            int layerPaintAlpha = mLayerPaint.getAlpha();
            if (alpha < 1) {
                mLayerPaint.setAlpha((int) (alpha * layerPaintAlpha));
            }
            canvas.drawBitmap(cache, 0.0f, 0.0f, mLayerPaint);
            if (alpha < 1) {
                mLayerPaint.setAlpha(layerPaintAlpha);
            }
        }
    }

    if (restoreTo >= 0) {
        canvas.restoreToCount(restoreTo);
    }

    if (a != null && !more) {
        if (!hardwareAcceleratedCanvas && !a.getFillAfter()) {
            onSetAlpha(255);
        }
        parent.finishAnimatingView(this, a);
    }

    if (more && hardwareAcceleratedCanvas) {
        if (a.hasAlpha() && (mPrivateFlags & PFLAG_ALPHA_SET) == PFLAG_ALPHA_SET) {
            // alpha animations should cause the child to recreate its display list
            invalidate(true);
        }
    }

    mRecreateDisplayList = false;

    return more;
}
```

可以看到：

- 对于软件绘制：
    - 会有一个叫绘制缓存的概念，buildDrawingCache 负责创建绘制缓存，view 会将自身绘制到缓存上，然后再绘制到 parent 的 canvas 上；
    - 如果不使用绘制缓存的话，就会直接绘制到  parent 传进来的 canvas 上；

- 对于硬件绘制：
    - 会有一个叫 RenderBode 和 DisplayList 的概念；

### 4.2.1 updateDisplayListIfDirty

 获取 View 的 RenderNode，同时更新显示列表 DisplayList；

```java
    @NonNull
    public RenderNode updateDisplayListIfDirty() {
        //【1】将自身的 rn 保存到 renderNode 临时变量中；
        final RenderNode renderNode = mRenderNode;
        if (!canHaveDisplayList()) {
            // 如果不是硬件加速，不能有 DisplayList，直接返回 renderNode 内部有属性呀；
            return renderNode;
        }

        //【2】这里是针对硬件绘制的情况，PFLAG_DRAWING_CACHE_VALID 标志位表示是否通过缓存绘制；
        if ((mPrivateFlags & PFLAG_DRAWING_CACHE_VALID) == 0
                || !renderNode.isValid()
                || (mRecreateDisplayList)) {
            //【2.1】如果没有设置了 PFLAG_DRAWING_CACHE_VALID 标志位的话，那么会进入这里：
            if (renderNode.isValid()  // isValid：返回 RenderNode 的显示列表内容当前是否可用。
                    && !mRecreateDisplayList) { // 无需重新创建 DisplayList；
                //【2.2】这里设置了标志位 PFLAG_DRAWING_CACHE_VALID
                mPrivateFlags |= PFLAG_DRAWN | PFLAG_DRAWING_CACHE_VALID;
                mPrivateFlags &= ~PFLAG_DIRTY_MASK;
                //【-->3.3】硬件绘制，无需重新创建 DisplayList，只需要告诉 child 恢复/重新创建自身即可；
                dispatchGetDisplayList();

                return renderNode;
            }

            //【4】表示要重新创建显示列表；
            // 如此标记以确保我们将子显示列表中的内容复制到 drawChild() 中？？
            mRecreateDisplayList = true;

            int width = mRight - mLeft;
            int height = mBottom - mTop;
            int layerType = getLayerType();

            //【4.1】这是硬件绘制的情况，开始记录渲染节点的显示列表，返回一个 DisplayListCanvas，用于记录绘制操作；
            final DisplayListCanvas canvas = renderNode.start(width, height);
            canvas.setHighContrastText(mAttachInfo.mHighContrastText);

            try {
                //【5】这里针对 layer 的情况做了区分；
                if (layerType == LAYER_TYPE_SOFTWARE) {
                    //【-->4.2.2】如果是 SOFTWARE 层的话，那么会构建绘制缓存，通过缓存绘制；
                    buildDrawingCache(true);
                    //【-->】获取缓存 cache，并绘制；
                    Bitmap cache = getDrawingCache(true);
                    if (cache != null) {
                        canvas.drawBitmap(cache, 0, 0, mLayerPaint);
                    }
                } else {
                    //【5.1】如果是 HARDWARE 层的话，那就不使用缓存了；
                    computeScroll();

                    canvas.translate(-mScrollX, -mScrollY);
                    mPrivateFlags |= PFLAG_DRAWN | PFLAG_DRAWING_CACHE_VALID;
                    mPrivateFlags &= ~PFLAG_DIRTY_MASK;

                    // Fast path for layouts with no backgrounds
                    if ((mPrivateFlags & PFLAG_SKIP_DRAW) == PFLAG_SKIP_DRAW) {
                        //【-->4.1.3】如果跳过自身的话，那就分发出去；
                        dispatchDraw(canvas);
                        if (mOverlay != null && !mOverlay.isEmpty()) {
                            mOverlay.getOverlayView().draw(canvas);
                        }
                    } else {
                        //【-->4.1】绘制自身；
                        draw(canvas);
                    }
                }
            } finally {
                //【6】结束记录；
                renderNode.end(canvas);
                setDisplayListProperties(renderNode);
            }
        } else {
            mPrivateFlags |= PFLAG_DRAWN | PFLAG_DRAWING_CACHE_VALID;
            mPrivateFlags &= ~PFLAG_DIRTY_MASK;
        }
        return renderNode;
    }
```

这里有几个标志位：

```java
mRecreateDisplayList = false;
```

表示 View 在当前图形绘制之前被标记为 INVALIDATED 或者起 DisplayList 无效。

如果为 true，则 View 必须重新计算其 DisplayList，仅在硬件加速时使用。

系统会清除 INVALIDATED 标志，同时将是否 INVALIDATED 保存到该变量中，硬件加速后续会用到

（ 比如在 getDisplayList() 时和在 drawChild() 中，当我们决定将视图的子级显示列表绘制到我们自己的状态中）。

### 4.2.2 buildDrawingCache

构建绘制缓存，参数 autoScale 表示是否自动缩放；

如果未启用自动缩放，则此方法将创建与该 view 相同大小的 bitmap 缓存。

```java
    public void buildDrawingCache(boolean autoScale) {
        //【1】如果没有设置 PFLAG_DRAWING_CACHE_VALID 有效性标志位，或者要强制进入的话；
        if ((mPrivateFlags & PFLAG_DRAWING_CACHE_VALID) == 0 || (autoScale ?
                mDrawingCache == null : mUnscaledDrawingCache == null)) {
            if (Trace.isTagEnabled(Trace.TRACE_TAG_VIEW)) {
                Trace.traceBegin(Trace.TRACE_TAG_VIEW,
                        "buildDrawingCache/SW Layer for " + getClass().getSimpleName());
            }
            try {
                //【-->4.2.2.1】创建绘制缓存；
                buildDrawingCacheImpl(autoScale);
            } finally {
                Trace.traceEnd(Trace.TRACE_TAG_VIEW);
            }
        }
    }
```

如果图形缓存无效，则强制构建图形缓存，如果只调用  buildDrawingCache() 而没有调用 setDrawingCacheEnabled(true)
则应在之后调用  destroyDrawingCache()；

启用硬件加速后，不应使用此方法。如果不需要图形缓存，则调用此方法将增加内存使用量，并使 view 一次性通过软件绘制出来，从而对性能产生负面影响。



#### 4.2.2.1 buildDrawingCacheImpl

构建绘制缓存：

```java
    private void buildDrawingCacheImpl(boolean autoScale) {
        mCachingFailed = false;
		//【1】计算一下绘制的宽/高；
        int width = mRight - mLeft;
        int height = mBottom - mTop;

        final AttachInfo attachInfo = mAttachInfo;
        //【2】是否需要缩放，如果需要的话，会根据 view root 设置的缩放比，对绘制宽高缩放；
        final boolean scalingRequired = attachInfo != null && attachInfo.mScalingRequired;
        if (autoScale && scalingRequired) {
            width = (int) ((width * attachInfo.mApplicationScale) + 0.5f);
            height = (int) ((height * attachInfo.mApplicationScale) + 0.5f);
        }

        final int drawingCacheBackgroundColor = mDrawingCacheBackgroundColor;
        final boolean opaque = drawingCacheBackgroundColor != 0 || isOpaque();
        final boolean use32BitCache = attachInfo != null && attachInfo.mUse32BitDrawingCache;

        final long projectedBitmapSize = width * height * (opaque && !use32BitCache ? 2 : 4);
        final long drawingCacheSize =
                ViewConfiguration.get(mContext).getScaledMaximumDrawingCacheSize();
        //【3】这里比较了下要绘制的图案的内存大小是不是比允许缓存的最大值大，如果是的话，不允许缓存；
        if (width <= 0 || height <= 0 || projectedBitmapSize > drawingCacheSize) {
            if (width > 0 && height > 0) {
                Log.w(VIEW_LOG_TAG, getClass().getSimpleName() + " not displayed because it is"
                        + " too large to fit into a software layer (or drawing cache), needs "
                        + projectedBitmapSize + " bytes, only "
                        + drawingCacheSize + " available");
            }
            destroyDrawingCache();
            mCachingFailed = true;
            return;
        }

        boolean clear = true;
        //【4】选择哪一种缓存；
        Bitmap bitmap = autoScale ? mDrawingCache : mUnscaledDrawingCache;
        
        //【5】选择缓存对应的质量；
        if (bitmap == null || bitmap.getWidth() != width || bitmap.getHeight() != height) {
            Bitmap.Config quality;
            if (!opaque) {
                // Never pick ARGB_4444 because it looks awful
                // Keep the DRAWING_CACHE_QUALITY_LOW flag just in case
                switch (mViewFlags & DRAWING_CACHE_QUALITY_MASK) {
                    case DRAWING_CACHE_QUALITY_AUTO:
                    case DRAWING_CACHE_QUALITY_LOW:
                    case DRAWING_CACHE_QUALITY_HIGH:
                    default:
                        quality = Bitmap.Config.ARGB_8888;
                        break;
                }
            } else {
                // Optimization for translucent windows
                // If the window is translucent, use a 32 bits bitmap to benefit from memcpy()
                quality = use32BitCache ? Bitmap.Config.ARGB_8888 : Bitmap.Config.RGB_565;
            }

            if (bitmap != null) bitmap.recycle();
            
            //【6】创建一个 Bitmap 缓存保存到 mDrawingCache/mUnscaledDrawingCache 中；
            try {
                bitmap = Bitmap.createBitmap(mResources.getDisplayMetrics(),
                        width, height, quality);
                bitmap.setDensity(getResources().getDisplayMetrics().densityDpi);
                if (autoScale) {
                    mDrawingCache = bitmap;
                } else {
                    mUnscaledDrawingCache = bitmap;
                }
                if (opaque && use32BitCache) bitmap.setHasAlpha(false);
            } catch (OutOfMemoryError e) {
				// 如果没有足够的内存的话，那就不使用缓存了；
                if (autoScale) {
                    mDrawingCache = null;
                } else {
                    mUnscaledDrawingCache = null;
                }
                mCachingFailed = true;
                return;
            }

            clear = drawingCacheBackgroundColor != 0;
        }

        Canvas canvas;
        //【7】这里的 attachInfo 来自于 ViewRootImpl。下面会对 canvas 做一下调整；
        if (attachInfo != null) {
            canvas = attachInfo.mCanvas;
            if (canvas == null) {
                canvas = new Canvas();
            }
            //【7.1】canvas 的 Bitmap 设置为我们创建的缓存 Bitmap；
            canvas.setBitmap(bitmap);
            // Temporarily clobber the cached Canvas in case one of our children
            // is also using a drawing cache. Without this, the children would
            // steal the canvas by attaching their own bitmap to it and bad, bad
            // thing would happen (invisible views, corrupted drawings, etc.)
            attachInfo.mCanvas = null;
        } else {
            // 这是极端的情况，view 没有 attach 上；
            canvas = new Canvas(bitmap);
        }

        if (clear) {
            bitmap.eraseColor(drawingCacheBackgroundColor);
        }

        computeScroll();
        final int restoreCount = canvas.save();

        if (autoScale && scalingRequired) {
            final float scale = attachInfo.mApplicationScale;
            canvas.scale(scale, scale);
        }

        canvas.translate(-mScrollX, -mScrollY);

        mPrivateFlags |= PFLAG_DRAWN;
        if (mAttachInfo == null || !mAttachInfo.mHardwareAccelerated ||
                mLayerType != LAYER_TYPE_NONE) {
            mPrivateFlags |= PFLAG_DRAWING_CACHE_VALID;
        }

        //【8】注意：这里又执行了一次绘制，但这是将自身绘制到这个缓存 bitmap 上；
        // 同时将缓存 canvas 传递给了 child view（如果有的话）
        if ((mPrivateFlags & PFLAG_SKIP_DRAW) == PFLAG_SKIP_DRAW) {
            mPrivateFlags &= ~PFLAG_DIRTY_MASK;
            //【-->4.1.3】分发绘制；
            dispatchDraw(canvas);
            if (mOverlay != null && !mOverlay.isEmpty()) {
                mOverlay.getOverlayView().draw(canvas);
            }
        } else {
            //【-->4.1】绘制自身以及 child view
            draw(canvas);
        }

        canvas.restoreToCount(restoreCount);
        canvas.setBitmap(null);

        if (attachInfo != null) {
            //【9】将缓存画布保存到 attachInfo，下次再用；
            attachInfo.mCanvas = canvas;
        }
    }
```



### 4.2.3 getDrawingCache

返回一个绘制缓存：

```java
    public Bitmap getDrawingCache(boolean autoScale) {
        //【1】该 view 不创建绘制缓存，返回 null；
        if ((mViewFlags & WILL_NOT_CACHE_DRAWING) == WILL_NOT_CACHE_DRAWING) {
            return null;
        }
        if ((mViewFlags & DRAWING_CACHE_ENABLED) == DRAWING_CACHE_ENABLED) {
            //【-->4.2.2】创建一个绘制缓存；
            buildDrawingCache(autoScale);
        }
        //【2】根据是否自动缩放，返回对应的 cache；
        return autoScale ? mDrawingCache : mUnscaledDrawingCache;
    }
```





## 4.3 dispatchGetDisplayList - 核心

ViewGroup 使用该方法来还原或重新创建 child view 的显示列表。

常规的 draw/dispatchDraw 过程中，当 ViewGroup 不需要重新创建自己的显示列表时，getDisplayList() 会调用此方法；

```java
    protected void dispatchGetDisplayList() {}
```

可以看到。真正的实现在 ViewGroup 中。

# 5 ThreadedRenderer - 硬件绘制简单记录

硬件绘制的操作要比软件绘制复杂的多，这里只简单的分析下：

## 5.1 draw - 核心

我们来看看硬件绘制的流程：

```java
    void draw(View view, AttachInfo attachInfo, HardwareDrawCallbacks callbacks) {
        attachInfo.mIgnoreDirtyState = true;

        final Choreographer choreographer = attachInfo.mViewRootImpl.mChoreographer;
        choreographer.mFrameInfo.markDrawStart();
        //【-->5.2】更新 DecorView 的 DisplayList 显示列表；
        // 该方法会遍历整个树形视图结构，从 DecorView 开始，更新所有视图的 DisplayList。
        updateRootDisplayList(view, callbacks);

        attachInfo.mIgnoreDirtyState = false;

        // register animating rendernodes which started animating prior to renderer
        // creation, which is typical for animators started prior to first draw
        if (attachInfo.mPendingAnimatingRenderNodes != null) {
            final int count = attachInfo.mPendingAnimatingRenderNodes.size();
            for (int i = 0; i < count; i++) {
                registerAnimatingRenderNode(
                        attachInfo.mPendingAnimatingRenderNodes.get(i));
            }
            attachInfo.mPendingAnimatingRenderNodes.clear();
            // We don't need this anymore as subsequent calls to
            // ViewRootImpl#attachRenderNodeAnimator will go directly to us.
            attachInfo.mPendingAnimatingRenderNodes = null;
        }

        final long[] frameInfo = choreographer.mFrameInfo.mFrameInfo;
        //【2】同步帧数据，最终目的是将 OpenGL 指令写入 gpu；
        int syncResult = nSyncAndDrawFrame(mNativeProxy, frameInfo, frameInfo.length);
        if ((syncResult & SYNC_LOST_SURFACE_REWARD_IF_FOUND) != 0) {
            setEnabled(false);
            attachInfo.mViewRootImpl.mSurface.release();
            // Invalidate since we failed to draw. This should fetch a Surface
            // if it is still needed or do nothing if we are no longer drawing
            attachInfo.mViewRootImpl.invalidate();
        }
        if ((syncResult & SYNC_INVALIDATE_REQUIRED) != 0) {
            attachInfo.mViewRootImpl.invalidate();
        }
    }
```

硬件绘制的 canvas 它是一个 DisplayListCanvas 对象，它的每一个 drawxxx 的方法并不是真正的绘制，而是在记录绘制的操作；

和软件绘制绘制类似，由于 View 体系是一个树形结构，所以硬件绘制也要遍历这个 tree，但是他的遍历是记录每个 view 的绘制操作，而不是直接绘制；

而触发这个遍历的方法就是：updateRootDisplayList，updateViewTreeDisplayList 还有 updateDisplayListIfDirty 方法；



## 5.2 updateRootDisplayList - 核心

继续看：

```java
    private void updateRootDisplayList(View view, HardwareDrawCallbacks callbacks) {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "Record View#draw()");
        //【-->5.3】遍历整个树形视图结构，从 DecorView 开始，更新所有视图的 DisplayList。
        updateViewTreeDisplayList(view);

        if (mRootNodeNeedsUpdate || !mRootNode.isValid()) {
            //【1】开始记录渲染节点的显示列表，返回一个 DisplayListCanvas，用于记录绘制操作；
            DisplayListCanvas canvas = mRootNode.start(mSurfaceWidth, mSurfaceHeight);
            try {
                final int saveCount = canvas.save();
                canvas.translate(mInsetLeft, mInsetTop);
                callbacks.onHardwarePreDraw(canvas);
                //【2】插入栅栏，隔离 canvas 操作
                canvas.insertReorderBarrier();
                //【3】绘制顶层视图 RenderNode；
                canvas.drawRenderNode(view.updateDisplayListIfDirty());
                canvas.insertInorderBarrier();

                callbacks.onHardwarePostDraw(canvas);
                canvas.restoreToCount(saveCount);
                mRootNodeNeedsUpdate = false;
            } finally {
                //【4】停止记录；
                mRootNode.end(canvas);
            }
        }
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
```

继续看：

## 5.3 updateViewTreeDisplayList - 核心

```java
private void updateViewTreeDisplayList(View view) {
    view.mPrivateFlags |= View.PFLAG_DRAWN;
    view.mRecreateDisplayList = (view.mPrivateFlags & 
                View.PFLAG_INVALIDATED)== View.PFLAG_INVALIDATED;
    view.mPrivateFlags &= ~View.PFLAG_INVALIDATED;
    //【-->4.2.1】更新 DecorView 的 DisplayList；
    view.updateDisplayListIfDirty();
    view.mRecreateDisplayList = false;
}
```



# 6 总结

我们用一张图来总结下整个的软件绘制的流程；

最近加班多，要休息下～～～（略）