# ViewDraw 第三篇 performTraversals 流程分析

title: ViewDraw 第三篇 performTraversals 流程分析
date: 2018/06/21 20:46:25 
catalog: true
categories: 

- View 视图
- View 的加载和绘制

tags: ViewDraw
copyright: true

------

篇文章基于 Android N - 7.1.1 主要分析下 performTraversals 方法的执行流程；

# 1 回顾

我们来回顾下，当请求到 Vsync 信号后就会触发 callback：

## 1.1 TraversalRunnable.run

TraversalRunnable 只是一个 runnable，其内部会调用另外一个方法：**doTraversal()**

```java
    final class TraversalRunnable implements Runnable {
        @Override
        public void run() {
            //【-->1.2】开始遍历；
            doTraversal();
        }
    }
    final TraversalRunnable mTraversalRunnable = new TraversalRunnable();
```

## 1.2 doTraversal

开始遍历了：

```java
    void doTraversal() {
        if (mTraversalScheduled) {
            //【1】要设置为 false，否则下一次重绘制没法触发；
            mTraversalScheduled = false;
            //【2】移出障栅；
            mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);

            if (mProfile) {
                Debug.startMethodTracing("ViewAncestor");
            }
            //【-->2.1】执行视图遍历；
            performTraversals();

            if (mProfile) {
                Debug.stopMethodTracing();
                mProfile = false;
            }
        }
    }
```

我们看到，最后调用了 performTraversals 方法！！

# 2 ViewRootImpl

## 2.1 performTraversals - 核心

执行视图遍历，这个方法很长很长：

```java
private void performTraversals() {
    //【1】讲 DecorView 保存到 host 中；
    final View host = mView;

    if (DBG) {
        System.out.println("======================================");
        System.out.println("performTraversals");
        host.debug();
    }

    if (host == null || !mAdded)
        return;
    //【2】均设置为 true，正在遍历/即将绘制；
    mIsInTraversal = true;
    mWillDrawSoon = true;
    boolean windowSizeMayChange = false; // 视图的大小可能改变
    boolean newSurface = false;
    boolean surfaceChanged = false;
    
    //【3】获得布局参数；
    WindowManager.LayoutParams lp = mWindowAttributes;

    //【4】DecorView 所需要的宽度和高度；
    int desiredWindowWidth;
    int desiredWindowHeight;
    
    //【-->2.1.1】获取 Decorview 的可见性，默认为 true；
    final int viewVisibility = getHostVisibility();
    //【5】判断 Decorview 可见性是否变化：不是第一次加载 && （可见性和之前不一样 || 需要新的 surface）
    // 当然第一次加载 mFirst 为 true，所以为 false；
    final boolean viewVisibilityChanged = !mFirst
        && (mViewVisibility != viewVisibility || mNewSurfaceNeeded);
    
    //【6】判断 Decorview 用户可感知的可见性是否变化：这是的用户可见性指的是能感知到的变化
    // 上面的变化还包括 Surface 的变化，整个用户不一定能感知的到；
    // 当然第一次加载 mFirst 为 true；所以为 false
    final boolean viewUserVisibilityChanged = !mFirst &&
        ((mViewVisibility == View.VISIBLE) != (viewVisibility == View.VISIBLE));
    
    //【7】对布局参数再做一次调整；
    WindowManager.LayoutParams params = null;
    if (mWindowAttributesChanged) {
        mWindowAttributesChanged = false;
        surfaceChanged = true;
        params = lp;
    }
    CompatibilityInfo compatibilityInfo =
        mDisplay.getDisplayAdjustments().getCompatibilityInfo();
    if (compatibilityInfo.supportsScreen() == mLastInCompatMode) {
        params = lp;
        mFullRedrawNeeded = true;
        mLayoutRequested = true;
        if (mLastInCompatMode) {
            params.privateFlags &= ~WindowManager.LayoutParams.PRIVATE_FLAG_COMPATIBLE_WINDOW;
            mLastInCompatMode = false;
        } else {
            params.privateFlags |= WindowManager.LayoutParams.PRIVATE_FLAG_COMPATIBLE_WINDOW;
            mLastInCompatMode = true;
        }
    }

    mWindowAttributesChangesFlag = 0;

    //【8】用来保存窗口宽度和高度，这个 mWinFrame 保存了窗口最新尺寸；
    // 是由 wms 返回的，是一个 Rect 实例；此时他的意义是上一次请求的大小；
    // 这里的 desiredWindowWidth 和 desiredWindowHeight 表示的是期望的宽高，但不一定是实际的；
    Rect frame = mWinFrame;
    //【9】这里 mFirst 表示是否是第一次显示，在 new ViewRootImpl 的时候是设置为 true！
    if (mFirst) {
        // 是否需要全部重绘/是否要求重新 Layout 界面
        mFullRedrawNeeded = true;
        mLayoutRequested = true;
        
        //【-->2.1.2】这里针对了一个特殊的 type 做了判断，他们使用除去 action bar 后屏幕的宽高
        if (shouldUseDisplaySize(lp)) {
            // 这里会设置成除去 action bar 后屏幕的宽高；
            Point size = new Point();
            mDisplay.getRealSize(size);
            desiredWindowWidth = size.x;
            desiredWindowHeight = size.y;
        } else {
            // 否则所需要窗口的宽度和高度就是整个屏幕的宽高；
            Configuration config = mContext.getResources().getConfiguration();
            desiredWindowWidth = dipToPx(config.screenWidthDp);
            desiredWindowHeight = dipToPx(config.screenHeightDp);
        }

        //【*10】设置下 mAttachInfo 中的一些参数：
        // 使用 32 为绘制缓存 true
        // 窗口持有焦点 false
        // 窗口可见性：viewVisibility 一般是 true；
        // 重新计算全局的布局属性：false；
        // 我们以前使用以下条件来选择32位图形缓存： PixelFormat.hasAlpha（lp.format）
        // || lp.format == PixelFormat.RGBX_8888，但是，现在默认情况下，窗口始终为 32位，
        // 因此请选择 32 位；
        mAttachInfo.mUse32BitDrawingCache = true;
        mAttachInfo.mHasWindowFocus = false;
        mAttachInfo.mWindowVisibility = viewVisibility;
        mAttachInfo.mRecomputeGlobalAttributes = false;
        //【11】设置上一次的配置和上一的可见性，也就是本次的设置；
        mLastConfiguration.setTo(host.getResources().getConfiguration());
        mLastSystemUiVisibility = mAttachInfo.mSystemUiVisibility;

        //【12】设置布局方向，如果还没有设置，默认是 inherit
        if (mViewLayoutDirectionInitial == View.LAYOUT_DIRECTION_INHERIT) {
            host.setLayoutDirection(mLastConfiguration.getLayoutDirection());
        }
        //【-->3.1】讲 mAttachInfo 设置到 DecorView 中，host 是 DecorView！
        host.dispatchAttachedToWindow(mAttachInfo, 0);
        
        //【-->5.2】通过 ViewTreeObserver 发出通知：已经 attach 到了 window 上；
        mAttachInfo.mTreeObserver.dispatchOnWindowAttachedChange(true);
        // 使用 windowInserts；
        dispatchApplyInsets(host);
        // Log.i(mTag, "Screen on initialized: " + attachInfo.mKeepScreenOn);

    } else {
        // 对于不是第一次显示的情况，这里就不会有 attach 的过程；
        // 也就是上一次储存的宽高值；
        desiredWindowWidth = frame.width();
        desiredWindowHeight = frame.height();
        if (desiredWindowWidth != mWidth || desiredWindowHeight != mHeight) {
            if (DEBUG_ORIENTATION) Log.v(mTag, "View " + host + " resized to: " + frame);
            mFullRedrawNeeded = true;
            mLayoutRequested = true;
            windowSizeMayChange = true;
        }
    }

    //【13】如果 DecorView 的可见性发生变化，第一次进入为 false；
    if (viewVisibilityChanged) {
        //【*13.1】更新  mAttachInfo.mWindowVisibility（窗口可见性）为 viewVisibility；
        mAttachInfo.mWindowVisibility = viewVisibility;
        //【-->4.2】通知所有的 child；
        host.dispatchWindowVisibilityChanged(viewVisibility);
        //【13.2】如果 DecorView 的用户可见性发生变化；
        if (viewUserVisibilityChanged) {
            //【-->4.3】通知所有的 child；
            host.dispatchVisibilityAggregated(viewVisibility == View.VISIBLE);
        }
        if (viewVisibility != View.VISIBLE || mNewSurfaceNeeded) {
            endDragResizing();
            destroyHardwareResources();
        }
        if (viewVisibility == View.GONE) {
            //【13.3】如果 DecorView 的变为不可见，那么 mHasHadWindowFocus 为 false，表示不持有 window 焦点；
            mHasHadWindowFocus = false;
        }
    }

    //【14】上面有更新过 mWindowVisibility，如果 window 不可见，那么其不能持有辅助服务的焦点；
    if (mAttachInfo.mWindowVisibility != View.VISIBLE) {
        host.clearAccessibilityFocus(); // 这个方法就先不看了；
    }

    //【15】在主线程上执行内部任务队列的 runnable 任务，这里的任务是用于触发布局的，
    // 如果我们在布局的过程中触发 requestLayout 的话，那么在布局完成后会再次针对请求进行布局；
    // 如果第二次布局完成后，还有 requestLayout，那么会被加入到 RunQueue 中，在下一帧，也就是这里，进行布局；
    getRunQueue().executeActions(mAttachInfo.mHandler);

    boolean insetsChanged = false;

    //【16】判断是否请求布局，mLayoutRequested 在 requestLayout() 设置为了 true；
    boolean layoutRequested = mLayoutRequested && (!mStopped || mReportNextDraw);
    if (layoutRequested) {
        final Resources res = mView.getContext().getResources();
        //【16.1】对于第一次显示 view，显然 mFirst 为 true，那么会进入这里；
        if (mFirst) {
            // 处理和 touch mode 相关逻辑；
            mAttachInfo.mInTouchMode = !mAddedTouchMode;
            ensureTouchModeLocally(mAddedTouchMode);

        } else {
            //【16.2】对于不是第一次加载，那么会进入这里；处理 Overscan 区域的处理，这个我们先不关注；
            // 先判断一下几个 insects 的值和上一次相比有没有什么变化，不想等的话就将 insetsChanged 标志位变为 ture;
            if (!mPendingOverscanInsets.equals(mAttachInfo.mOverscanInsets)) {
                insetsChanged = true;
            }
            if (!mPendingContentInsets.equals(mAttachInfo.mContentInsets)) {
                insetsChanged = true;
            }
            if (!mPendingStableInsets.equals(mAttachInfo.mStableInsets)) {
                insetsChanged = true;
            }
            if (!mPendingVisibleInsets.equals(mAttachInfo.mVisibleInsets)) {
                mAttachInfo.mVisibleInsets.set(mPendingVisibleInsets);
                if (DEBUG_LAYOUT) Log.v(mTag, "Visible insets changing to: "
                                        + mAttachInfo.mVisibleInsets);
            }
            if (!mPendingOutsets.equals(mAttachInfo.mOutsets)) {
                insetsChanged = true;
            }
            if (mPendingAlwaysConsumeNavBar != mAttachInfo.mAlwaysConsumeNavBar) {
                insetsChanged = true;
            }
            //【16.3】这里判断了如果布局参数 lp 指定的宽/高为 WRAP_CONTENT，也就是适应内部布局，那么窗口大小可能会改变
            // 所以设置 windowSizeMayChange 为 true；
            if (lp.width == ViewGroup.LayoutParams.WRAP_CONTENT
                || lp.height == ViewGroup.LayoutParams.WRAP_CONTENT) {
                windowSizeMayChange = true;
           		// 这里和上面的逻辑一样，设置 window 的显示大小；
                if (shouldUseDisplaySize(lp)) {
                    Point size = new Point();
                    mDisplay.getRealSize(size);
                    desiredWindowWidth = size.x;
                    desiredWindowHeight = size.y;
                } else {
                    Configuration config = res.getConfiguration();
                    desiredWindowWidth = dipToPx(config.screenWidthDp);
                    desiredWindowHeight = dipToPx(config.screenHeightDp);
                }
            }
        }

        //【-->2.2】为了确定 Window 的大小而执行预测量，预判下 window 的大小会不会变化；
        windowSizeMayChange |= measureHierarchy(host, lp, res,
                                                desiredWindowWidth, desiredWindowHeight);
    }
    
    //【17】这里之前是有分析过，更新了 mAttachInfo 的一些属性，不多看；
    // 然后把 pl 保存到 params 中；
    if (collectViewAttributes()) {
        params = lp;
    }
    if (mAttachInfo.mForceReportNewAttributes) {
        mAttachInfo.mForceReportNewAttributes = false;
        params = lp;
    }

    //【18】如果是第一次，或者说 DecorView 可见性变化了，那么就会进入这里，调整 resize mode；
    if (mFirst || mAttachInfo.mViewVisibilityChanged) {
        mAttachInfo.mViewVisibilityChanged = false;
        int resizeMode = mSoftInputMode &
            WindowManager.LayoutParams.SOFT_INPUT_MASK_ADJUST;
        if (resizeMode == WindowManager.LayoutParams.SOFT_INPUT_ADJUST_UNSPECIFIED) {
            final int N = mAttachInfo.mScrollContainers.size();
            for (int i=0; i<N; i++) {
                if (mAttachInfo.mScrollContainers.get(i).isShown()) {
                    resizeMode = WindowManager.LayoutParams.SOFT_INPUT_ADJUST_RESIZE;
                }
            }
            if (resizeMode == 0) {
                resizeMode = WindowManager.LayoutParams.SOFT_INPUT_ADJUST_PAN;
            }
            if ((lp.softInputMode &
                 WindowManager.LayoutParams.SOFT_INPUT_MASK_ADJUST) != resizeMode) {
                lp.softInputMode = (lp.softInputMode &
                                    ~WindowManager.LayoutParams.SOFT_INPUT_MASK_ADJUST) |
                    resizeMode;
                params = lp;
            }
        }
    }
    //【19】对参数进行新一步的调整；
    if (params != null) {
        //【19.1】如果 DecorView 设置成了透明背景，那么这里会设置下 format 为 PixelFormat.TRANSLUCENT；
        if ((host.mPrivateFlags & View.PFLAG_REQUEST_TRANSPARENT_REGIONS) != 0) {
            if (!PixelFormat.formatHasAlpha(params.format)) {
                params.format = PixelFormat.TRANSLUCENT;
            }
        }
        //【19.2】如果有窗口标记 FLAG_LAYOUT_IN_OVERSCAN，则允许窗口内容扩展到屏幕的 OverScan 区域
        // 这样的话，会导致部分 content 进入  OverScan 区域显示不全，这里可以使用 View.setFitsSystemWindows(boolean)
        // 进行调整；
        mAttachInfo.mOverscanRequested = (params.flags
                                          & WindowManager.LayoutParams.FLAG_LAYOUT_IN_OVERSCAN) != 0;
    }

    //【20】通过 requestFitSystemWindows() 方法设置其为 true，然后会重新执行 scheduleTraversals() 方法；
    if (mApplyInsetsRequested) {
        mApplyInsetsRequested = false;
        mLastOverscanRequested = mAttachInfo.mOverscanRequested;
        // 使用 windowInserts；
        dispatchApplyInsets(host);
        if (mLayoutRequested) {
            //【-->2.2】 这里又进行了一次预测量；
            windowSizeMayChange |= measureHierarchy(host, lp,
                                                    mView.getContext().getResources(),
                                                    desiredWindowWidth, desiredWindowHeight);
        }
    }

    if (layoutRequested) {
        //【20.1】这里会清除 layout requested 标记，这样下面的代码如果要再请求布局的话，我们会重新开始执行布局流程；
        mLayoutRequested = false;
    }

    //【21】判断窗口是否需要重新调整大小；
    // 1、layoutRequested 为 true，说明程序已经发起了一次测量，布局，绘制流程。
    // 2、windowSizeMayChange 为 true, 说明前面预测量已经检测到了 Activity 窗口的变化，或者 h/w 为 wrap_content
    // 3、当测量出来的大小和当前大小不一致；
    boolean windowShouldResize = layoutRequested && windowSizeMayChange
        && ((mWidth != host.getMeasuredWidth() || mHeight != host.getMeasuredHeight())
            || (lp.width == ViewGroup.LayoutParams.WRAP_CONTENT &&
                frame.width() < desiredWindowWidth && frame.width() != mWidth)
            || (lp.height == ViewGroup.LayoutParams.WRAP_CONTENT &&
                frame.height() < desiredWindowHeight && frame.height() != mHeight));
    
    windowShouldResize |= mDragResizing && mResizeMode == RESIZE_MODE_FREEFORM;
    //【22】如果活动刚刚重新启动，则它可能已经冻结了任务边界（重新启动时），因此我们需要强制通过 wms 获取最新的边界。
    windowShouldResize |= mActivityRelaunched;

    //【23】检查 Activity 窗口是否需要指定有额外的内容边衬区域和可见边衬区域。
    // Activity 窗口指定额外的内容边衬区域和可见边衬区域是为了放置一些额外的东西。
    final boolean computesInternalInsets =
        mAttachInfo.mTreeObserver.hasComputeInternalInsetsListeners()
        || mAttachInfo.mHasNonEmptyGivenInternalInsets;

    boolean insetsPending = false;
    int relayoutResult = 0;
    boolean updatedConfiguration = false;

    //【-->6.1.1】mSurface 是 ViewRootImpl 内部成员，是一个 Surface 实例内部，持有一个 Canvas;
    // 我们的绘制实际上就是发生其上的；
    //【-->6.1.2】这里是获取其 id 标示；
    final int surfaceGenerationId = mSurface.getGenerationId();

    final boolean isViewVisible = viewVisibility == View.VISIBLE; // 这里又判断了下 DecorView 可见性；
    
    //【24】如果是第一次加载，或者 window 需要重新调整大小，或者 insets 发生了改变，或者 view 可见性变化了；
    // 或者布局参数不为 null，或者强制视图重新布局，都会进入这里；
    if (mFirst || windowShouldResize || insetsChanged ||
        viewVisibilityChanged || params != null || mForceNextWindowRelayout) {
        mForceNextWindowRelayout = false;

        if (isViewVisible) {
            //【24.1】检查接下来请求 WMS 服务计算大小时，是否要告诉 WMS 服务它指定了额外的内容区域边衬和可见区域边衬，
            // 但是这些额外的内容区域边衬和可见区域边衬又还没确定，
            // 这种情况发生在 Activity 窗口第一次执行测量、布局和绘制操作或者由不可见变化可见时；
            insetsPending = computesInternalInsets && (mFirst || viewVisibilityChanged);
        }

        //【24.3】我们知道默认的情况下 mSurfaceHolder 是 null 的，这里不进入；
        if (mSurfaceHolder != null) {
            mSurfaceHolder.mSurfaceLock.lock();
            mDrawingAllowed = true;
        }

        boolean hwInitialized = false;
        boolean contentInsetsChanged = false;
        //【24.4】判断 surface 是否有效，目前是无效的；
        boolean hadSurface = mSurface.isValid();

        try {
            if (DEBUG_LAYOUT) {
                Log.i(mTag, "host=w:" + host.getMeasuredWidth() + ", h:" +
                      host.getMeasuredHeight() + ", params=" + params);
            }
            //【25】如果开启了硬件加速，那么这里会对 suface 进行锁定，防止 relayoutWindow 时；
            // wms 对 surface 进行销毁；
            if (mAttachInfo.mHardwareRenderer != null) {
                if (mAttachInfo.mHardwareRenderer.pauseSurface(mSurface)) {
                    // Animations were running so we need to push a frame
                    // to resume them
                    mDirty.set(0, 0, mWidth, mHeight);
                }
                mChoreographer.mFrameInfo.addFlags(FrameInfo.FLAG_WINDOW_LAYOUT_CHANGED);
            }
            
            //【-->2.3】请求 WMS 计算Activity窗口的大小以及过扫描区域边衬大小和可见区域边衬大小
            // 同时，Surface 只有通过 request 后才是有效的；
            // 该流程会进入 wms，最终就通过 Binder 调用，该方法会返回，请求的大小，然后设置到 mWinFrame 中；
            relayoutResult = relayoutWindow(params, viewVisibility, insetsPending);

            if (DEBUG_LAYOUT) Log.v(mTag, "relayout: frame=" + frame.toShortString()
                                    + " overscan=" + mPendingOverscanInsets.toShortString()
                                    + " content=" + mPendingContentInsets.toShortString()
                                    + " visible=" + mPendingVisibleInsets.toShortString()
                                    + " visible=" + mPendingStableInsets.toShortString()
                                    + " outsets=" + mPendingOutsets.toShortString()
                                    + " surface=" + mSurface);

            if (mPendingConfiguration.seq != 0) {
                if (DEBUG_CONFIGURATION) Log.v(mTag, "Visible with new config: "
                                               + mPendingConfiguration);
                updateConfiguration(new Configuration(mPendingConfiguration), !mFirst);
                mPendingConfiguration.seq = 0;
                updatedConfiguration = true;
            }

            //【26】判断边衬是否发生了变化；
            final boolean overscanInsetsChanged = !mPendingOverscanInsets.equals(
                mAttachInfo.mOverscanInsets);
            contentInsetsChanged = !mPendingContentInsets.equals(
                mAttachInfo.mContentInsets);
            final boolean visibleInsetsChanged = !mPendingVisibleInsets.equals(
                mAttachInfo.mVisibleInsets);
            final boolean stableInsetsChanged = !mPendingStableInsets.equals(
                mAttachInfo.mStableInsets);
            final boolean outsetsChanged = !mPendingOutsets.equals(mAttachInfo.mOutsets);
            
            //【27】判断 surface 大小是否发生变化；
            final boolean surfaceSizeChanged = (relayoutResult
                                                & WindowManagerGlobal.RELAYOUT_RES_SURFACE_RESIZED) != 0;
            final boolean alwaysConsumeNavBarChanged =
                mPendingAlwaysConsumeNavBar != mAttachInfo.mAlwaysConsumeNavBar;
            
            //【28】更新新的边衬数据；
            if (contentInsetsChanged) {
                mAttachInfo.mContentInsets.set(mPendingContentInsets);
                if (DEBUG_LAYOUT) Log.v(mTag, "Content insets changing to: "
                                        + mAttachInfo.mContentInsets);
            }
            if (overscanInsetsChanged) {
                mAttachInfo.mOverscanInsets.set(mPendingOverscanInsets);
                if (DEBUG_LAYOUT) Log.v(mTag, "Overscan insets changing to: "
                                        + mAttachInfo.mOverscanInsets);
                // Need to relayout with content insets.
                contentInsetsChanged = true;
            }
            if (stableInsetsChanged) {
                mAttachInfo.mStableInsets.set(mPendingStableInsets);
                if (DEBUG_LAYOUT) Log.v(mTag, "Decor insets changing to: "
                                        + mAttachInfo.mStableInsets);
                // Need to relayout with content insets.
                contentInsetsChanged = true;
            }
            if (alwaysConsumeNavBarChanged) {
                mAttachInfo.mAlwaysConsumeNavBar = mPendingAlwaysConsumeNavBar;
                contentInsetsChanged = true;
            }
            if (contentInsetsChanged || mLastSystemUiVisibility !=
                mAttachInfo.mSystemUiVisibility || mApplyInsetsRequested
                || mLastOverscanRequested != mAttachInfo.mOverscanRequested
                || outsetsChanged) {
                mLastSystemUiVisibility = mAttachInfo.mSystemUiVisibility;
                mLastOverscanRequested = mAttachInfo.mOverscanRequested;
                mAttachInfo.mOutsets.set(mPendingOutsets);
                mApplyInsetsRequested = false;
                dispatchApplyInsets(host);
            }
            if (visibleInsetsChanged) {
                mAttachInfo.mVisibleInsets.set(mPendingVisibleInsets);
                if (DEBUG_LAYOUT) Log.v(mTag, "Visible insets changing to: "
                                        + mAttachInfo.mVisibleInsets);
            }

            //【29】处理 surface；
            if (!hadSurface) {
                //【29.1】如果之前还没持有 surface，显然会进入这里，而此时 requestWindow 返回后；
                // Surface 就有效了，isValid 返回了 true；
                if (mSurface.isValid()) {
                    // 如果要创建一个新 surface，则需要完全重绘它。同样，当我们到达绘制点时，我们将推迟并安排新的遍历。
                    // 这样一来，我们可以在实际绘制窗口之前告诉窗口管理器所有正在显示的窗口，以便随后显示。
                    newSurface = true;
                    mFullRedrawNeeded = true; // 重绘；
                    mPreviousTransparentRegion.setEmpty();

                    // 仅在不要求透明区域的情况下才预先初始化，否则请推迟查看整个窗口。
                    if (mAttachInfo.mHardwareRenderer != null) {
                        try {
                            hwInitialized = mAttachInfo.mHardwareRenderer.initialize(
                                mSurface);
                            if (hwInitialized && (host.mPrivateFlags
                                                  & View.PFLAG_REQUEST_TRANSPARENT_REGIONS) == 0) {
                                // 如果要求透明区域，则不要预先分配，因为可能不需要透明区域；
                                mSurface.allocateBuffers();
                            }
                        } catch (OutOfResourcesException e) {
                            handleOutOfResourcesException(e);
                            return;
                        }
                    }
                }
            } else if (!mSurface.isValid()) {
                //【29.2】如果已经持有 surface，但是已经失效了，那就进入这里；
                // 这里会重置滚动位置；
                if (mLastScrolledFocus != null) {
                    mLastScrolledFocus.clear();
                }
                mScrollY = mCurScrollY = 0;
                if (mView instanceof RootViewSurfaceTaker) {
                    ((RootViewSurfaceTaker) mView).onRootViewScrollYChanged(mCurScrollY);
                }
                if (mScroller != null) {
                    mScroller.abortAnimation();
                }
                
                if (mAttachInfo.mHardwareRenderer != null &&
                    mAttachInfo.mHardwareRenderer.isEnabled()) {
                    mAttachInfo.mHardwareRenderer.destroy();
                }
            } else if ((surfaceGenerationId != mSurface.getGenerationId()
                        || surfaceSizeChanged)
                       && mSurfaceHolder == null
                       && mAttachInfo.mHardwareRenderer != null) {
                //【29.3】如果已经持有 surface，但是并且是有效的，如果 Surface 发生了更改（ID 变化了）
                // 或 WindowManager 更改了 Surface 大小，那么就会进入这里；
                mFullRedrawNeeded = true;
                try {
                    // 需要执行 updateSurface（这会导致 CanvasContext::setSurface 并重新创建 EGLSurface）
                    // 后者是因为在某些芯片上，除非我们创建新的 EGLSurface，否则更改用户端 BufferQueue 的大小可能不会立即生效。
                    // 请注意，框架尺寸的改变并不总是意味着表面尺寸的改变（例如，拖动调整大小使用全屏表面），
                    // 需要从 WindowManager 中检查 surfaceSizeChanged 标志。
                    mAttachInfo.mHardwareRenderer.updateSurface(mSurface);
                } catch (OutOfResourcesException e) {
                    handleOutOfResourcesException(e);
                    return;
                }
            }
			
            //【30】下面事处理 resize 的情况；
            // 判断是否是因为 freeform 导致窗口大小变化，通过拖动窗口角之一来调整窗口的大小；
            final boolean freeformResizing = (relayoutResult
                                              & WindowManagerGlobal.RELAYOUT_RES_DRAG_RESIZING_FREEFORM) != 0;
            // 判断是否是因为 Dock 模式导致窗口大小变化，也就是通过拖动停靠的分隔线来调整窗口大小；
            final boolean dockedResizing = (relayoutResult
                                            & WindowManagerGlobal.RELAYOUT_RES_DRAG_RESIZING_DOCKED) != 0;
            // 判断是否是因为拖动导致大小变化；
            final boolean dragResizing = freeformResizing || dockedResizing;
            
            // 如果是大小变化和大小不变的状态之间发生了切换，进入这里；
            if (mDragResizing != dragResizing) {
                if (dragResizing) { 
                    // 如果这一次是发生变化了，那么会计算计算下 resize mode；
                    mResizeMode = freeformResizing
                        ? RESIZE_MODE_FREEFORM
                        : RESIZE_MODE_DOCKED_DIVIDER;
                    // 回调通知；
                    startDragResizing(mPendingBackDropFrame,
                                      mWinFrame.equals(mPendingBackDropFrame), mPendingVisibleInsets,
                                      mPendingStableInsets, mResizeMode);
                } else {
                    // 停止大小变化，回调通知；
                    endDragResizing();
                }
            }
            // 该常量用于设置是否使用多线程渲染器，默认是 true 也就是使用，如果为 false 的话；
            // WindowCallbacks 将不会触发;
            if (!USE_MT_RENDERER) {
                if (dragResizing) {
                    mCanvasOffsetX = mWinFrame.left;
                    mCanvasOffsetY = mWinFrame.top;
                } else {
                    mCanvasOffsetX = mCanvasOffsetY = 0;
                }
            }
        } catch (RemoteException e) {
        }

        if (DEBUG_ORIENTATION) Log.v(
            TAG, "Relayout returned: frame=" + frame + ", surface=" + mSurface);
        //【31】获得 wms 计算的窗口的最新的 top 和 left 距离，这个我就不说了；
        mAttachInfo.mWindowLeft = frame.left;
        mAttachInfo.mWindowTop = frame.top;

        // !!FIXME!! This next section handles the case where we did not get the
        // window size we asked for. We should avoid this by getting a maximum size from
        // the window session beforehand.
        //【32】这里是用 frame 更新下 mWidth/mHeight 的值；
        if (mWidth != frame.width() || mHeight != frame.height()) {
            mWidth = frame.width();
            mHeight = frame.height();
        }

        //【33】mSurfaceHolder 不为 null 说明应用创建了一个 holder，其可以对 Surface
        // 进行直接操作，这里是针对应用持有 SurfaceHolder 的情况；
        if (mSurfaceHolder != null) {
            //【33.1】surface 保存到 SurfaceHolder 内部；
            if (mSurface.isValid()) {
                // XXX .copyFrom() doesn't work!
                //mSurfaceHolder.mSurface.copyFrom(mSurface);
                mSurfaceHolder.mSurface = mSurface;
            }
            //【33.2】设置 surface 的大小；
            mSurfaceHolder.setSurfaceFrameSize(mWidth, mHeight);
            mSurfaceHolder.mSurfaceLock.unlock();
            if (mSurface.isValid()) {
                // 说明 surface 是有效的，如果 hadSurface 为 false，
                // 这里会回调 SurfaceHolder.Callback 的 surfaceCreated 方法，通知 surface 被创建；
                if (!hadSurface) {
                    mSurfaceHolder.ungetCallbacks();

                    mIsCreating = true;
                    mSurfaceHolderCallback.surfaceCreated(mSurfaceHolder);
                    SurfaceHolder.Callback callbacks[] = mSurfaceHolder.getCallbacks();
                    if (callbacks != null) {
                        for (SurfaceHolder.Callback c : callbacks) {
                            c.surfaceCreated(mSurfaceHolder);
                        }
                    }
                    surfaceChanged = true;
                }
                // 这里会回调 SurfaceHolder.Callback 的 surfaceChanged 方法，通知 surface 发生了变化；
                if (surfaceChanged || surfaceGenerationId != mSurface.getGenerationId()) {
                    mSurfaceHolderCallback.surfaceChanged(mSurfaceHolder,
                                                          lp.format, mWidth, mHeight);
                    SurfaceHolder.Callback callbacks[] = mSurfaceHolder.getCallbacks();
                    if (callbacks != null) {
                        for (SurfaceHolder.Callback c : callbacks) {
                            c.surfaceChanged(mSurfaceHolder, lp.format,
                                             mWidth, mHeight);
                        }
                    }
                }
                mIsCreating = false;
            } else if (hadSurface) {
                // 说明 surface 无效的，但是此时已经创建了 surface；
                // 这里会回调 SurfaceHolder.Callback 的 surfaceDestroyed 方法，通知 surface 被销毁；
                // 然后重新 new Surface；
                mSurfaceHolder.ungetCallbacks();
                SurfaceHolder.Callback callbacks[] = mSurfaceHolder.getCallbacks();
                mSurfaceHolderCallback.surfaceDestroyed(mSurfaceHolder);
                if (callbacks != null) {
                    for (SurfaceHolder.Callback c : callbacks) {
                        c.surfaceDestroyed(mSurfaceHolder);
                    }
                }
                mSurfaceHolder.mSurfaceLock.lock();
                try {
                    mSurfaceHolder.mSurface = new Surface();
                } finally {
                    mSurfaceHolder.mSurfaceLock.unlock();
                }
            }
        }

        //【34】判断是否开启了硬件加速，如果开启了，同时窗口的宽高和硬件加速的宽/高；
        // 这里要讲新的 width/height 设置到硬件加速环境中；
        final ThreadedRenderer hardwareRenderer = mAttachInfo.mHardwareRenderer;
        if (hardwareRenderer != null && hardwareRenderer.isEnabled()) {
            if (hwInitialized
                || mWidth != hardwareRenderer.getWidth()
                || mHeight != hardwareRenderer.getHeight()
                || mNeedsHwRendererSetup) {
                hardwareRenderer.setup(mWidth, mHeight, mAttachInfo,
                                       mWindowAttributes.surfaceInsets);
                mNeedsHwRendererSetup = false;
            }
        }
  
        //【35】如果此窗口的所有者处于停止状态，则设置 mStopped 为 true，此时该窗口不处于活动状态。
        if (!mStopped || mReportNextDraw) {
            boolean focusChangedDueToTouchMode = ensureTouchModeLocally(
                (relayoutResult&WindowManagerGlobal.RELAYOUT_RES_IN_TOUCH_MODE) != 0);
            //【35.1】如果要进入 touch mode，或者之前预测量的宽高和请求的宽高不一样；
            // 或者边衬的距离变化了，或者配置更新了，这都会开始测量；
            if (focusChangedDueToTouchMode || mWidth != host.getMeasuredWidth()
                || mHeight != host.getMeasuredHeight() || contentInsetsChanged ||
                updatedConfiguration) {
                //【-->2.2.1】计算出 root view 的测量规格 Measure Spec，此时 mWidth 和 mHeight 是 wms 返回的大小；
                // 他们是具体的数值；
                int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);
                int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);

                if (DEBUG_LAYOUT) Log.v(mTag, "Ooops, something changed!  mWidth="
                                        + mWidth + " measuredWidth=" + host.getMeasuredWidth()
                                        + " mHeight=" + mHeight
                                        + " measuredHeight=" + host.getMeasuredHeight()
                                        + " coveredInsetsChanged=" + contentInsetsChanged);

                //【-->2.4】开始测量；
                performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);

                //【35.1.1】getMeasuredWidth() 所获得 View 的宽高绝大部分情况下等于 View 最终的宽高；
                int width = host.getMeasuredWidth();
                int height = host.getMeasuredHeight();
                boolean measureAgain = false;

                //【35.1.2】处理设置了权重的情况，如果有权重的话，那么会分配剩余的空间；
                // 这里的权重是浮点数，也就是百分比；
                if (lp.horizontalWeight > 0.0f) {
                    width += (int) ((mWidth - width) * lp.horizontalWeight);
                    childWidthMeasureSpec = MeasureSpec.makeMeasureSpec(width,
                                                                        MeasureSpec.EXACTLY);
                    measureAgain = true;
                }
                if (lp.verticalWeight > 0.0f) {
                    height += (int) ((mHeight - height) * lp.verticalWeight);
                    childHeightMeasureSpec = MeasureSpec.makeMeasureSpec(height,
                                                                         MeasureSpec.EXACTLY);
                    measureAgain = true;
                }

                if (measureAgain) {
                    if (DEBUG_LAYOUT) Log.v(mTag,
                                            "And hey let's measure once more: width=" + width
                                            + " height=" + height);
                    //【-->2.4】再次测量；
                    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
                }

                layoutRequested = true;
            }
        }
    } else {
        //【-->2.1.5】如果不是第一次加载，并且窗口/边衬/可见性没有变化，但是窗口可能已经移动了，
        // 这里会检查一下是否要更新 attachInfo 中的 left/top 的值。
        maybeHandleWindowMove(frame);
    }

    // 判断是否需要布局，显然是需要的；
    final boolean didLayout = layoutRequested && (!mStopped || mReportNextDraw);
    //【36】是否触发全局布局的回调；
    boolean triggerGlobalLayoutListener = didLayout
        || mAttachInfo.mRecomputeGlobalAttributes;
    if (didLayout) {
        //【-->2.4】开始布局；
        performLayout(lp, mWidth, mHeight);

        // 至此，所有视图的大小和位置都已确定，我们可以计算出透明区域；
        if ((host.mPrivateFlags & View.PFLAG_REQUEST_TRANSPARENT_REGIONS) != 0) {
            // 设置透明区域；
            host.getLocationInWindow(mTmpLocation);
            mTransparentRegion.set(mTmpLocation[0], mTmpLocation[1],
                                   mTmpLocation[0] + host.mRight - host.mLeft,
                                   mTmpLocation[1] + host.mBottom - host.mTop);

            host.gatherTransparentRegion(mTransparentRegion);
            if (mTranslator != null) {
                mTranslator.translateRegionInWindowToScreen(mTransparentRegion);
            }

            if (!mTransparentRegion.equals(mPreviousTransparentRegion)) {
                mPreviousTransparentRegion.set(mTransparentRegion);
                mFullRedrawNeeded = true;
                // 将透明区域传递给 wms，进一步设置；
                try {
                    mWindowSession.setTransparentRegion(mWindow, mTransparentRegion);
                } catch (RemoteException e) {
                }
            }
        }

        if (DBG) {
            System.out.println("======================================");
            System.out.println("performTraversals -- after setFrame");
            host.debug();
        }
    }
    //【37】第一次加载的情况，肯定是为 true；
    if (triggerGlobalLayoutListener) {
        mAttachInfo.mRecomputeGlobalAttributes = false;
        //【-->5.4】全局布局回调；
        mAttachInfo.mTreeObserver.dispatchOnGlobalLayout();
    }

    //【38】设置边衬距离，并通知给 wms；
    if (computesInternalInsets) {
        //【38.1】清空旧的边衬信息；
        final ViewTreeObserver.InternalInsetsInfo insets = mAttachInfo.mGivenInternalInsets;
        insets.reset();

        //【-->5.5】回调相关接口计算当前插入边衬。
        mAttachInfo.mTreeObserver.dispatchOnComputeInternalInsets(insets);
        mAttachInfo.mHasNonEmptyGivenInternalInsets = !insets.isEmpty();

        if (insetsPending || !mLastGivenInsets.equals(insets)) {
            mLastGivenInsets.set(insets);

            // Translate insets to screen coordinates if needed.
            final Rect contentInsets;
            final Rect visibleInsets;
            final Region touchableRegion;
            if (mTranslator != null) {
                contentInsets = mTranslator.getTranslatedContentInsets(insets.contentInsets);
                visibleInsets = mTranslator.getTranslatedVisibleInsets(insets.visibleInsets);
                touchableRegion = mTranslator.getTranslatedTouchableArea(insets.touchableRegion);
            } else {
                contentInsets = insets.contentInsets;
                visibleInsets = insets.visibleInsets;
                touchableRegion = insets.touchableRegion;
            }

            try {
                //【38.2】通知给 wms；
                mWindowSession.setInsets(mWindow, insets.mTouchableInsets,
                                         contentInsets, visibleInsets, touchableRegion);
            } catch (RemoteException e) {
            }
        }
    }

    //【39】如果是第一次加载，这里会请求交点操作！
    if (mFirst) {
        if (DEBUG_INPUT_RESIZE) Log.v(mTag, "First: mView.hasFocus()="
                                      + mView.hasFocus());
        if (mView != null) {
            if (!mView.hasFocus()) {
                //【*39.1】请求焦点，这里先不分析。
                mView.requestFocus(View.FOCUS_FORWARD);
                if (DEBUG_INPUT_RESIZE) Log.v(mTag, "First: requested focused view="
                                              + mView.findFocus());
            } else {
                if (DEBUG_INPUT_RESIZE) Log.v(mTag, "First: existing focused view="
                                              + mView.findFocus());
            }
        }
    }

    final boolean changedVisibility = (viewVisibilityChanged || mFirst) && isViewVisible;
    final boolean hasWindowFocus = mAttachInfo.mHasWindowFocus && isViewVisible;
    final boolean regainedFocus = hasWindowFocus && mLostWindowFocus;
    if (regainedFocus) {
        mLostWindowFocus = false;
    } else if (!hasWindowFocus && mHadWindowFocus) {
        mLostWindowFocus = true;
    }

    if (changedVisibility || regainedFocus) {
        // Toast 以通知的形式显示，而不是 window；
        boolean isToast = (mWindowAttributes == null) ? false
            : (mWindowAttributes.type == WindowManager.LayoutParams.TYPE_TOAST);
        if (!isToast) {
            host.sendAccessibilityEvent(AccessibilityEvent.TYPE_WINDOW_STATE_CHANGED);
        }
    }

    mFirst = false;
    mWillDrawSoon = false;
    mNewSurfaceNeeded = false;
    mActivityRelaunched = false;
    mViewVisibility = viewVisibility;
    mHadWindowFocus = hasWindowFocus;
    //【40】针对于输入法的处理；
    if (hasWindowFocus && !isInLocalFocusMode()) {
        final boolean imTarget = WindowManager.LayoutParams
            .mayUseInputMethod(mWindowAttributes.flags);
        if (imTarget != mLastWasImTarget) {
            mLastWasImTarget = imTarget;
            InputMethodManager imm = InputMethodManager.peekInstance();
            if (imm != null && imTarget) {
                imm.onPreWindowFocus(mView, hasWindowFocus);
                imm.onPostWindowFocus(mView, mView.findFocus(),
                                      mWindowAttributes.softInputMode,
                                      !mHasHadWindowFocus, mWindowAttributes.flags);
            }
        }
    }

    // 如果请求结果设置了 RELAYOUT_RES_FIRST_TIME，表示：
    // 这是第一次被绘制的窗口，所以做的时候，客户端必须调用 drawingFinshed（）
    if ((relayoutResult & WindowManagerGlobal.RELAYOUT_RES_FIRST_TIME) != 0) {
        mReportNextDraw = true;
    }

    //【-->5.6】判断是否取消 draw；
    boolean cancelDraw = mAttachInfo.mTreeObserver.dispatchOnPreDraw() || !isViewVisible;

    if (!cancelDraw && !newSurface) {
        if (mPendingTransitions != null && mPendingTransitions.size() > 0) {
            for (int i = 0; i < mPendingTransitions.size(); ++i) {
                mPendingTransitions.get(i).startChangingAnimations();
            }
            mPendingTransitions.clear();
        }
		//【-->2.5】开始绘制；
        performDraw();
    } else {
        if (isViewVisible) {
            // Try again
            scheduleTraversals();
        } else if (mPendingTransitions != null && mPendingTransitions.size() > 0) {
            for (int i = 0; i < mPendingTransitions.size(); ++i) {
                mPendingTransitions.get(i).endChangingAnimations();
            }
            mPendingTransitions.clear();
        }
    }

    mIsInTraversal = false;
}
```



### 2.1.1 getHostVisibility

获取显示配置，看 DecorView（根 view group）是否显示：

```java
    int getHostVisibility() {
        //【1】如果应用是可见的，或者强制 DecorView 可见，那么取值为 mView.getVisibility()
        return (mAppVisible || mForceDecorViewVisibility) ? mView.getVisibility() : View.GONE;
    }
```

当然，对于 DecorView，默认是可见的，所以这个方法返回的是 true；



### 2.1.2 shouldUseDisplaySize

是否要设置成屏幕的大小；

```java
    private static boolean shouldUseDisplaySize(final WindowManager.LayoutParams lp) {
        return lp.type == TYPE_STATUS_BAR_PANEL
                || lp.type == TYPE_INPUT_METHOD
                || lp.type == TYPE_VOLUME_OVERLAY;
    }
```

这里会根据 window 的类型，判断是否设置成屏幕的大小；



### 2.1.3 dispatchApplyInsets

使用 windowInserts

```java
    void dispatchApplyInsets(View host) {
        host.dispatchApplyWindowInsets(getWindowInsets(true /* forceConstruct */));
    }
```

### 2.1.4 ensureTouchModeLocally

确保已设置此窗口的触摸模式：

- 参数 boolean inTouchMode：取值为 relayoutResult & WindowManagerGlobal.RELAYOUT_RES_IN_TOUCH_MODE) != 0，是否要进入 touch 模式；

```java
    private boolean ensureTouchModeLocally(boolean inTouchMode) {
        if (DBG) Log.d("touchmode", "ensureTouchModeLocally(" + inTouchMode + "), current "
                + "touch mode is " + mAttachInfo.mInTouchMode);
        //【1】判断 touch mode 状态是否变化，如果没有，就退出；
        if (mAttachInfo.mInTouchMode == inTouchMode) return false;
        //【2】跟新 touch mode 状态；
        mAttachInfo.mInTouchMode = inTouchMode;
        //【-->5.3】通知 TreeObserver；
        mAttachInfo.mTreeObserver.dispatchOnTouchModeChanged(inTouchMode);
        //【4】如果要退出或者进入 touch mode，那么会调用响应的方法打开/关闭；
        return (inTouchMode) ? enterTouchMode() : leaveTouchMode();
    }
```



### 2.1.5 maybeHandleWindowMove

这里会处理下 window 移动的情况：

```java
    private void maybeHandleWindowMove(Rect frame) {
        //【1】这里判断了下 window 是否发生了移动，也就是 left/top 的值不一样；
        final boolean windowMoved = mAttachInfo.mWindowLeft != frame.left
                || mAttachInfo.mWindowTop != frame.top;
        //【2】如果窗口移动了，那就更新 attachInfo 中的 Left/top
        if (windowMoved) {
            if (mTranslator != null) {
                mTranslator.translateRectInScreenToAppWinFrame(frame);
            }
            mAttachInfo.mWindowLeft = frame.left;
            mAttachInfo.mWindowTop = frame.top;
        }
        if (windowMoved || mAttachInfo.mNeedsUpdateLightCenter) {
            // 更新光源位置以获得新的偏移量(na ni?)
            if (mAttachInfo.mHardwareRenderer != null) {
                mAttachInfo.mHardwareRenderer.setLightCenter(mAttachInfo);
            }
            mAttachInfo.mNeedsUpdateLightCenter = false;
        }
    }
```



## 2.2 measureHierarchy - 预测量

这里是在正式测量绘制前，做一次预测量，测量结果会保存在 mMeasuredWidth 和 mMeasuredHeight 中：

```java
    private boolean measureHierarchy(final View host, final WindowManager.LayoutParams lp,
            final Resources res, final int desiredWindowWidth, final int desiredWindowHeight) {
        int childWidthMeasureSpec;
        int childHeightMeasureSpec;
        boolean windowSizeMayChange = false;

        if (DEBUG_ORIENTATION || DEBUG_LAYOUT) Log.v(mTag,
                "Measuring " + host + " in display " + desiredWindowWidth
                + "x" + desiredWindowHeight + "...");

        boolean goodMeasure = false;
        //【1】这里针对于布局参数为 WRAP_CONTENT 的情况出了特殊处理，也就是 Dialog 的情况；
        if (lp.width == ViewGroup.LayoutParams.WRAP_CONTENT) {
            //【1.1】在大屏幕上，我们不希望对话框仅拉伸以填充屏幕的整个宽度以显示一行文本。
            // 首先尝试以较小的尺寸进行布局，以查看是否适合。
            final DisplayMetrics packageMetrics = res.getDisplayMetrics();
            res.getValue(com.android.internal.R.dimen.config_prefDialogWidth, mTmpValue, true);
            int baseSize = 0;
            //【1.2】这里是获取系统内置的 TYPE_DIMENSION，保存到 baseSize；
            if (mTmpValue.type == TypedValue.TYPE_DIMENSION) {
                baseSize = (int)mTmpValue.getDimension(packageMetrics);
            }
            if (DEBUG_DIALOG) Log.v(mTag, "Window " + mView + ": baseSize=" + baseSize
                    + ", desiredWindowWidth=" + desiredWindowWidth);
            //【1.3】如果 desiredWindowWidth 大于 baseSize，那么我们以 baseSize 为宽度基准；
            if (baseSize != 0 && desiredWindowWidth > baseSize) {
                //【-->2.2.1】根据 view 的布局参数计算出其布局度量规范 measure spec;
                childWidthMeasureSpec = getRootMeasureSpec(baseSize, lp.width);
                childHeightMeasureSpec = getRootMeasureSpec(desiredWindowHeight, lp.height);
                
                //【-->2.4】执行测量；
                performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
                if (DEBUG_DIALOG) Log.v(mTag, "Window " + mView + ": measured ("
                        + host.getMeasuredWidth() + "," + host.getMeasuredHeight()
                        + ") from width spec: " + MeasureSpec.toString(childWidthMeasureSpec)
                        + " and height spec: " + MeasureSpec.toString(childHeightMeasureSpec));
  
                //【1.3】这里判断了下测量结果是不是太小了，如果合适，goodMeasure 为 true；
                if ((host.getMeasuredWidthAndState()&View.MEASURED_STATE_TOO_SMALL) == 0) {
                    goodMeasure = true;
                } else {
                    //【1.4】这里在前面的基础上再增加一些，重新测量；
                    baseSize = (baseSize+desiredWindowWidth)/2;
                    if (DEBUG_DIALOG) Log.v(mTag, "Window " + mView + ": next baseSize="
                            + baseSize);
                    //【-->2.2.1】再次根据 view 的布局参数计算出其布局度量规范 measure spec;
                    childWidthMeasureSpec = getRootMeasureSpec(baseSize, lp.width);

                    //【-->2.4】开始测量：
                    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);

                    if (DEBUG_DIALOG) Log.v(mTag, "Window " + mView + ": measured ("
                            + host.getMeasuredWidth() + "," + host.getMeasuredHeight() + ")");

                    //【1.5】如果测量结果合适，goodMeasure 为 true；
                    if ((host.getMeasuredWidthAndState()&View.MEASURED_STATE_TOO_SMALL) == 0) {
                        if (DEBUG_DIALOG) Log.v(mTag, "Good!");
                        goodMeasure = true;
                    }
                }
            }
        }

        //【2】如果上面的测量不合适，那么下面会用 desiredWindowWidth 来进行测量
        if (!goodMeasure) {
            //【-->2.2.1】最后再根据 view 的布局参数计算出其布局度量规范 measure spec;
            childWidthMeasureSpec = getRootMeasureSpec(desiredWindowWidth, lp.width);
            childHeightMeasureSpec = getRootMeasureSpec(desiredWindowHeight, lp.height);
            
            //【-->2.4】开始测量：
            performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
            //【2.1】这里会判断下测量的结果
            if (mWidth != host.getMeasuredWidth() || mHeight != host.getMeasuredHeight()) {
                windowSizeMayChange = true;
            }
        }

        if (DBG) {
            System.out.println("======================================");
            System.out.println("performTraversals -- after measure");
            host.debug();
        }

        return windowSizeMayChange;
    }
```

这里是提前做一次预测量。



### 2.2.1 getRootMeasureSpec

根据 view 的布局参数计算出其布局度量规范 measure spec：

- **int windowSize**：窗口的可用宽度或高度（mHeight/ mWidth）
- **int rootDimension**：布局参数参数指定的宽度或高度（match_parent/ wrap_content/ lp.height(width)）

```java
    private static int getRootMeasureSpec(int windowSize, int rootDimension) {
        int measureSpec;
        switch (rootDimension) {

        case ViewGroup.LayoutParams.MATCH_PARENT:
            //【-->4.4.2】计算度量规范，此时父布局 decorview 的布局参数是 MATCH_PARENT，那么，
            // 窗口无法调整大小，这里就会强制以父布局的大小来设置度量规范，同时由于确定了大小，所以 mode 为 MeasureSpec.EXACTLY
            measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.EXACTLY);
            break;
        case ViewGroup.LayoutParams.WRAP_CONTENT:
            //【-->4.4.2】计算度量规范，此时父布局 decorview 的布局参数是 MATCH_PARENT，那么，
            // 意味着窗口可以调整大小，这里就会强制以父布局的大小来设置度量规范，由于其可以调整大小，
            // 但是不能超过 windowSize，所以 mode 为 WRAP_CONTENT.
            measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.AT_MOST);
            break;
        default:
            //【-->4.4.2】计算度量规范，这种情况 rootDimension 为具体的参数；
            // 意味着窗口明确的指定了 windowSize，显然此时以 rootDimension 为准了。
            measureSpec = MeasureSpec.makeMeasureSpec(rootDimension, MeasureSpec.EXACTLY);
            break;
        }
        return measureSpec;
    }
```

对于 DecorView 来说，显然是第一个分支：

- measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.EXACTLY);

## 2.3 relayoutWindow - 通过 wms 计算窗口大小

请求 Wms 计算 Activity 窗口的大小以及过扫描区域边衬大小和可见区域边衬大小，同时返回可用的 Surface：

```java
    private int relayoutWindow(WindowManager.LayoutParams params, int viewVisibility,
            boolean insetsPending) throws RemoteException {

        float appScale = mAttachInfo.mApplicationScale;
        boolean restore = false;
        if (params != null && mTranslator != null) {
            restore = true;
            params.backup();
            mTranslator.translateWindowLayout(params);
        }
        if (params != null) {
            if (DBG) Log.d(mTag, "WindowLayout in layoutWindow:" + params);
        }
        mPendingConfiguration.seq = 0;
        //Log.d(mTag, ">>>>>> CALLING relayout");
        if (params != null && mOrigWindowType != params.type) {
            // For compatibility with old apps, don't crash here.
            if (mTargetSdkVersion < Build.VERSION_CODES.ICE_CREAM_SANDWICH) {
                Slog.w(mTag, "Window type can not be changed after "
                        + "the window is added; ignoring change of " + mView);
                params.type = mOrigWindowType;
            }
        }
        //【*1】通过 WindowSession 来请求 relayout；
        // 我们会传入 W 实例，布局参数 params，以及 view 的测量宽高，以及可见性，边衬边框，以及 Surface 对象；
        // 通过 request 后，surface 才是有效的；
        int relayoutResult = mWindowSession.relayout(
                mWindow, mSeq, params,
                (int) (mView.getMeasuredWidth() * appScale + 0.5f),
                (int) (mView.getMeasuredHeight() * appScale + 0.5f),
                viewVisibility, insetsPending ? WindowManagerGlobal.RELAYOUT_INSETS_PENDING : 0,
                mWinFrame, mPendingOverscanInsets, mPendingContentInsets, mPendingVisibleInsets,
                mPendingStableInsets, mPendingOutsets, mPendingBackDropFrame, mPendingConfiguration,
                mSurface);
 
        //【2】在多窗口中，我们强制显示导航栏。
        mPendingAlwaysConsumeNavBar =
                (relayoutResult & WindowManagerGlobal.RELAYOUT_RES_CONSUME_ALWAYS_NAV_BAR) != 0;

        //Log.d(mTag, "<<<<<< BACK FROM relayout");
        if (restore) {
            params.restore();
        }

        if (mTranslator != null) {
            mTranslator.translateRectInScreenToAppWinFrame(mWinFrame);
            mTranslator.translateRectInScreenToAppWindow(mPendingOverscanInsets);
            mTranslator.translateRectInScreenToAppWindow(mPendingContentInsets);
            mTranslator.translateRectInScreenToAppWindow(mPendingVisibleInsets);
            mTranslator.translateRectInScreenToAppWindow(mPendingStableInsets);
        }
        return relayoutResult;
    }
```

可以看到，这里的核心逻辑就是：

- 通过 WindowSession 来请求 wms 计算 view 的宽高；
- 同时会初始化 mWinFrame, mPendingOverscanInsets,  mPendingContentInsets,  mPendingVisibleInsets,  mPendingStableInsets, mPendingOutsets,  mPendingBackDropFrame,  mPendingConfiguration,  mSurface，对于 Surface，触发其 readFromParcel 方法；

我们去看看 IWindowSession.aidl 文件：

```java
    int relayout(IWindow window, int seq, in WindowManager.LayoutParams attrs,
            int requestedWidth, int requestedHeight, int viewVisibility,
            int flags, out Rect outFrame, out Rect outOverscanInsets,
            out Rect outContentInsets, out Rect outVisibleInsets, out Rect outStableInsets,
            out Rect outOutsets, out Rect outBackdropFrame, out Configuration outConfig,
            out Surface outSurface);
```

可以看到，我们初始化的那几个成员变量都是用 out 修饰的，这里我们简单回顾下 in 和 out 的区别：

- in 修饰的参数传递到服务方，服务方对实参的任何改变，不会反应给调用方。
- out 修饰的参数，不会真正传到服务方，只是传一个实参的初始值过去，但服务方对实参的任何改变，在调用结束后会反应给调用方。
	- 这里实参只是作为返回值来使用的，这样除了 return 那里的返回值，还可以返回另外的东西；
- inout 参数则是上面二者的结合，实参会顺利传到服务方，且服务方对实参的任何改变，在调用结束后会反应回调用方。



## 2.4 performMeasure - 测量

这里的参数：childWidthMeasureSpec 和 childHeightMeasureSpec 是 root view，也就是 DecorView 的测量标准！

```java
private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
    Trace.traceBegin(Trace.TRACE_TAG_VIEW, "measure");
    try {
        //【1】最终调用了 DecorView（ViewGroup）的 measure 方法，并讲自身的测量规范传递下去；
        mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
}
```

对于 mView.measure 我们在下一篇文章中分析：

## 2.5 performLayout - 布局

开始布局：

```java
    private void performLayout(WindowManager.LayoutParams lp, int desiredWindowWidth,
            int desiredWindowHeight) {
        mLayoutRequested = false;
        mScrollMayChange = true;
        mInLayout = true;

        final View host = mView;
        if (host == null) {
            return;
        }
        if (DEBUG_ORIENTATION || DEBUG_LAYOUT) {
            Log.v(mTag, "Laying out " + host + " to (" +
                    host.getMeasuredWidth() + ", " + host.getMeasuredHeight() + ")");
        }

        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "layout");
        try {
            //【1-core】调用 DecorView（GroupView）的 layout；
            host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());

            mInLayout = false;
            int numViewsRequestingLayout = mLayoutRequesters.size();
            //【2】如果 mLayoutRequesters 大小不为 0 ；说明我们在布局的过程中调用了 requestLayout 方法；
            if (numViewsRequestingLayout > 0) {
                // 在布局期间调用了 requestLayout()。
                // 如果在请求的视图上未设置布局请求标志，则没有问题。
                // 如果某些请求仍在等待处理中，那么我们需要清除这些标志并进行完整的请求/度量/布局传递以处理这种情况。
                //【-->2.5.1】获取有效的布局请求；
                ArrayList<View> validLayoutRequesters = getValidLayoutRequesters(mLayoutRequesters,
                        false);
                if (validLayoutRequesters != null) {
                    //【3】设置此标志，表示即将进入第二次布局中，这样的话，如果再发起 requestlayout 的话；
                    // 这些请求会延迟到下一帧
                    mHandlingLayoutInLayoutRequest = true;

                    //【4】处理有效的布局请求；
                    int numValidRequests = validLayoutRequesters.size();
                    for (int i = 0; i < numValidRequests; ++i) {
                        final View view = validLayoutRequesters.get(i);
                        Log.w("View", "requestLayout() improperly called by " + view +
                                " during layout: running second layout pass");
                        //【4.1-core】调用每一个 View 的 layout 方法；
                        view.requestLayout();
                    }
                    //【-->2.2】再次预测量；
                    measureHierarchy(host, lp, mView.getContext().getResources(),
                            desiredWindowWidth, desiredWindowHeight);
                    mInLayout = true;
                    
                    //【5-core】调用 DecorView（GroupView）的 layout，再次布局；
                    host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());

                    mHandlingLayoutInLayoutRequest = false;

                    //【-->2.5.1】再次获取有效的布局请求，注意这里第二个参数传入的是 true
                    // 这次不会清除布局标志，因为在第二次布局中的请求会延迟到下一帧;
                    validLayoutRequesters = getValidLayoutRequesters(mLayoutRequesters, true);
                    if (validLayoutRequesters != null) {
                        final ArrayList<View> finalRequesters = validLayoutRequesters;
                        //【6】将第二次的请求发布到下一帧，这个 runnable 会在 performTraversals 中执行；
                        getRunQueue().post(new Runnable() {
                            @Override
                            public void run() {
                                int numValidRequests = finalRequesters.size();
                                for (int i = 0; i < numValidRequests; ++i) {
                                    final View view = finalRequesters.get(i);
                                    Log.w("View", "requestLayout() improperly called by " + view +
                                            " during second layout pass: posting in next frame");
                                    //【6.1】触发 requestLayout 方法！
                                    view.requestLayout();
                                }
                            }
                        });
                    }
                }

            }
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
        mInLayout = false;
    }
```

核心就是 ViewGroup 的 layout！！

我们这里会再看到有一个 list 列表：

```java
    ArrayList<View> mLayoutRequesters = new ArrayList<View>();
```

当一个 View.requestLayout() 方法被调用的时候，如果此时处于布局中，那么该 view 会被添加到 mLayoutRequesters 中：

- **requestLayoutDuringLayout**

```java
    boolean requestLayoutDuringLayout(final View view) {
        if (view.mParent == null || view.mAttachInfo == null) {
            //【1】没有 parent 或者 mAttachInfo 为 null，那么其绑定异常，返回 true，对于这种情况，则不会触发布局；
            return true;
        }
        //【2】将其加入到 mLayoutRequesters 列表中；
        if (!mLayoutRequesters.contains(view)) {
            mLayoutRequesters.add(view);
        }
        if (!mHandlingLayoutInLayoutRequest) {
            //【3】表示是在第一次布局过程收到的请求，返回 true，下面会强制二次布局；
            return true;
        } else {
            //【4】表示是在第二次强制布局过程收到的请求，返回 false，推迟到下一帧；
            return false;
        }
    }
```

（下面来自方法的翻译，感觉已经很清晰了）

一般不应该在布局过程中调用 requestLayout 方法，也就是说：正常情况下，应该是对该容器层次结构中的所有子级进行统一的一次性测量并在布局结束后进行统一布局。

如果仍然调用 requestLayout()，我们将通过在帧进行过程中将这些请求缓存下来，在布局结束后，我们会检查下，看是否还有待处理的布局请求，表示它们未被其容器层次结构正确处理。

如果真是这样，我们将清除树中的所有此类标志，并在该帧中强制第二次请求/度量/布局传递。
如果在第二次布局传递过程中收到了更多的 requestLayout 调用，我们会将这些请求发布到下一帧，以避免可能的无限循环。

此方法的返回值指示请求是否应该继续二次布局（如果是在第一次布局过程中收到的请求）
还是应该跳过并发布到下一个帧（如果是在第二个过程中收到的请求）

### 2.5.1 getValidLayoutRequesters

获取有效的布局请求！

如果在布局期间调用了 requestLayout 方法，则在布局期间会出发此方法。

它遍历了请求布局的视图列表，以根据层次结构中的可见性以及是否已经处理了它们来确定仍需要布局的视图（通常是 ListView 子级的情况）。

- 参数 layoutRequesters 保存的是 requestLayoutDuringLayout 的 View；
- 参数 secondLayoutRequests 表示的是是否是第二次布局，第一次布局时传入的 false；

```java
    private ArrayList<View> getValidLayoutRequesters(ArrayList<View> layoutRequesters,
            boolean secondLayoutRequests) {

        int numViewsRequestingLayout = layoutRequesters.size();
        ArrayList<View> validLayoutRequesters = null;
        //【1】遍历 layoutRequesters 中的 view；
        for (int i = 0; i < numViewsRequestingLayout; ++i) {
            View view = layoutRequesters.get(i);
            //【2】如果 view 已经绑定完成，并且 本次是第二次布局或者 view 仅设置了 PFLAG_FORCE_LAYOUT 标识位；
            if (view != null && view.mAttachInfo != null && view.mParent != null &&
                    (secondLayoutRequests || (view.mPrivateFlags & View.PFLAG_FORCE_LAYOUT) ==
                            View.PFLAG_FORCE_LAYOUT)) {
                boolean gone = false;
                View parent = view;
                //【3】这里看到，只会对非 gone 层次结构的视图重新触发布局；no gone 的 view 会被加入到 
                // validLayoutRequesters 中；
                while (parent != null) {
                    if ((parent.mViewFlags & View.VISIBILITY_MASK) == View.GONE) {
                        gone = true;
                        break;
                    }
                    // 这里是判断 view 以及其 parent 是否有 gone，只要有就算 gone！
                    if (parent.mParent instanceof View) {
                        parent = (View) parent.mParent;
                    } else {
                        parent = null;
                    }
                }
                if (!gone) {
                    if (validLayoutRequesters == null) {
                        validLayoutRequesters = new ArrayList<View>();
                    }
                    validLayoutRequesters.add(view);
                }
            }
        }
        //【4】对于第一次布局，这里会清空掉 view 的 PFLAG_FORCE_LAYOUT 标志位；
        // 对于 view 以及其 parent 都要清掉；
        if (!secondLayoutRequests) {
            for (int i = 0; i < numViewsRequestingLayout; ++i) {
                View view = layoutRequesters.get(i);
                while (view != null &&
                        (view.mPrivateFlags & View.PFLAG_FORCE_LAYOUT) != 0) {
                    view.mPrivateFlags &= ~View.PFLAG_FORCE_LAYOUT;
                    if (view.mParent instanceof View) {
                        view = (View) view.mParent;
                    } else {
                        view = null;
                    }
                }
            }
        }
        //【5】清空 mLayoutRequesters，然后返回 validLayoutRequesters；
        layoutRequesters.clear();
        return validLayoutRequesters;
    }

```

不多说了！



## 2.6 performDraw - 绘制

开始绘制：

```java
    private void performDraw() {
        //【1】如果屏幕状态是灭屏，或者不需要 next draw，那就不绘制；
        if (mAttachInfo.mDisplayState == Display.STATE_OFF && !mReportNextDraw) {
            return;
        }

        final boolean fullRedrawNeeded = mFullRedrawNeeded; // 判断是否全量绘制；
        mFullRedrawNeeded = false;

        mIsDrawing = true;
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "draw");
        try {
            //【-->2.6.1】执行绘制；
            draw(fullRedrawNeeded);
        } finally {
            mIsDrawing = false;
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }

        // For whatever reason we didn't create a HardwareRenderer, end any
        // hardware animations that are now dangling
        if (mAttachInfo.mPendingAnimatingRenderNodes != null) {
            final int count = mAttachInfo.mPendingAnimatingRenderNodes.size();
            for (int i = 0; i < count; i++) {
                mAttachInfo.mPendingAnimatingRenderNodes.get(i).endAllAnimators();
            }
            mAttachInfo.mPendingAnimatingRenderNodes.clear();
        }

        if (mReportNextDraw) {
            mReportNextDraw = false;

            // if we're using multi-thread renderer, wait for the window frame draws
            if (mWindowDrawCountDown != null) {
                try {
                    mWindowDrawCountDown.await();
                } catch (InterruptedException e) {
                    Log.e(mTag, "Window redraw count down interruped!");
                }
                mWindowDrawCountDown = null;
            }

            if (mAttachInfo.mHardwareRenderer != null) {
                mAttachInfo.mHardwareRenderer.fence();
                mAttachInfo.mHardwareRenderer.setStopped(mStopped);
            }

            if (LOCAL_LOGV) {
                Log.v(mTag, "FINISHED DRAWING: " + mWindowAttributes.getTitle());
            }
            // 如果 window 创建了 mSurfaceHolder，并且 surface 是有效的，那就要触发回调；
            // 这里的 mSurfaceHolder 一般是 null；
            if (mSurfaceHolder != null && mSurface.isValid()) {
                mSurfaceHolderCallback.surfaceRedrawNeeded(mSurfaceHolder);
                SurfaceHolder.Callback callbacks[] = mSurfaceHolder.getCallbacks();
                if (callbacks != null) {
                    for (SurfaceHolder.Callback c : callbacks) {
                        if (c instanceof SurfaceHolder.Callback2) {
                            ((SurfaceHolder.Callback2)c).surfaceRedrawNeeded(mSurfaceHolder);
                        }
                    }
                }
            }
            try {
                //【2】通过 Window Session 
                mWindowSession.finishDrawing(mWindow);
            } catch (RemoteException e) {
            }
        }
    }
```



### 2.6.1 draw

执行绘制：

```java
    private void draw(boolean fullRedrawNeeded) {
        //【1】这个是否的 Surface 已经是有效的了；
        Surface surface = mSurface;
        if (!surface.isValid()) {
            return;
        }

        if (DEBUG_FPS) {
            trackFPS(); // debug 相关；
        }

        if (!sFirstDrawComplete) {
            synchronized (sFirstDrawHandlers) {
                sFirstDrawComplete = true;
                final int count = sFirstDrawHandlers.size();
                for (int i = 0; i< count; i++) {
                    mHandler.post(sFirstDrawHandlers.get(i));
                }
            }
        }

        scrollToRectOrFocus(null, false);

        //【2】如果 view 的 scroll 变化了，触发 TreeObserver.dispatchOnScrollChanged 方法；
        if (mAttachInfo.mViewScrollChanged) {
            mAttachInfo.mViewScrollChanged = false;
            mAttachInfo.mTreeObserver.dispatchOnScrollChanged();
        }

        boolean animating = mScroller != null && mScroller.computeScrollOffset();
        final int curScrollY;
        if (animating) {
            curScrollY = mScroller.getCurrY();
        } else {
            curScrollY = mScrollY;
        }
        if (mCurScrollY != curScrollY) {
            mCurScrollY = curScrollY;
            fullRedrawNeeded = true;
            if (mView instanceof RootViewSurfaceTaker) {
                ((RootViewSurfaceTaker) mView).onRootViewScrollYChanged(mCurScrollY);
            }
        }

        final float appScale = mAttachInfo.mApplicationScale;
        final boolean scalingRequired = mAttachInfo.mScalingRequired;

        int resizeAlpha = 0;
        //【3】处理要绘制的区域 mDirty，首先是先获取了 mDirty 值，该值保存了需要重绘的区域的信息；
        final Rect dirty = mDirty;
        if (mSurfaceHolder != null) {
            // 不绘制 app 自己持有的 surface；
            dirty.setEmpty();
            if (animating && mScroller != null) {
                mScroller.abortAnimation();
            }
            return;
        }
        //【4】判断下是否是全量绘制，如果是的话，那么 dirty 就是整个 window 的范围了，表示整个视图都需要绘制
        // 第一次绘制流程，需要绘制所有视图；
        if (fullRedrawNeeded) {
            mAttachInfo.mIgnoreDirtyState = true;
            dirty.set(0, 0, (int) (mWidth * appScale + 0.5f), (int) (mHeight * appScale + 0.5f));
        }

        if (DEBUG_ORIENTATION || DEBUG_DRAW) {
            Log.v(mTag, "Draw " + mView + "/"
                    + mWindowAttributes.getTitle()
                    + ": dirty={" + dirty.left + "," + dirty.top
                    + "," + dirty.right + "," + dirty.bottom + "} surface="
                    + surface + " surface.isValid()=" + surface.isValid() + ", appScale:" +
                    appScale + ", width=" + mWidth + ", height=" + mHeight);
        }
        //【5】触发 TreeObserver.dispatchOnDraw 方法；
        mAttachInfo.mTreeObserver.dispatchOnDraw();

        int xOffset = -mCanvasOffsetX;
        int yOffset = -mCanvasOffsetY + curScrollY;
        final WindowManager.LayoutParams params = mWindowAttributes;
        final Rect surfaceInsets = params != null ? params.surfaceInsets : null;
        if (surfaceInsets != null) {
            xOffset -= surfaceInsets.left;
            yOffset -= surfaceInsets.top;

            // Offset dirty rect for surface insets.
            dirty.offset(surfaceInsets.left, surfaceInsets.right);
        }

        boolean accessibilityFocusDirty = false;
        final Drawable drawable = mAttachInfo.mAccessibilityFocusDrawable;
        if (drawable != null) {
            final Rect bounds = mAttachInfo.mTmpInvalRect;
            final boolean hasFocus = getAccessibilityFocusedRect(bounds);
            if (!hasFocus) {
                bounds.setEmpty();
            }
            if (!bounds.equals(drawable.getBounds())) {
                accessibilityFocusDirty = true;
            }
        }

        mAttachInfo.mDrawingTime =
                mChoreographer.getFrameTimeNanos() / TimeUtils.NANOS_PER_MS;
        //【6】下面就要进入绘制了；
        if (!dirty.isEmpty() || mIsAnimating || accessibilityFocusDirty) {
            //【6.1】硬件绘制，当然前提是开启了硬件加速；
            if (mAttachInfo.mHardwareRenderer != null && mAttachInfo.mHardwareRenderer.isEnabled()) {
                // If accessibility focus moved, always invalidate the root.
                boolean invalidateRoot = accessibilityFocusDirty || mInvalidateRootRequested;
                mInvalidateRootRequested = false;

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

                // 更新绘制的内容大小。它将在 draw 命令发送到渲染器之前不久将其传输到渲染器。
                final boolean updated = updateContentDrawBounds();

                if (mReportNextDraw) {
                    // mReportNextDraw 会覆盖 setStop 方法，在处理完 reportNextDraw 的绘制后
                    // 值；会恢复；
                    mAttachInfo.mHardwareRenderer.setStopped(false);
                }

                if (updated) {
                    requestDrawWindow();
                }
                //【6.2】使用 render thread 进行绘制，这个是在另外一个线程中；
                mAttachInfo.mHardwareRenderer.draw(mView, mAttachInfo, this);
            } else {
                //【6.3】软件绘制；
				// 如果硬件渲染被禁，但是本次绘制缺请求了硬件渲染，那么这里会执行初始化硬件渲染；
                if (mAttachInfo.mHardwareRenderer != null &&
                        !mAttachInfo.mHardwareRenderer.isEnabled() &&
                        mAttachInfo.mHardwareRenderer.isRequested()) {

                    try {
                        //【6.4】初始化硬件渲染；
                        mAttachInfo.mHardwareRenderer.initializeIfNeeded(
                                mWidth, mHeight, mAttachInfo, mSurface, surfaceInsets);
                    } catch (OutOfResourcesException e) {
                        handleOutOfResourcesException(e);
                        return;
                    }

                    mFullRedrawNeeded = true;
                    //【6.5-core】请求下一帧；
                    scheduleTraversals();
                    return;
                }
                //【-->2.6.2】执行软件绘制，返回值表示绘制是否成功；
                if (!drawSoftware(surface, mAttachInfo, xOffset, yOffset, scalingRequired, dirty)) {
                    return;
                }
            }
        }

        if (animating) {
            mFullRedrawNeeded = true;
            scheduleTraversals();
        }
    }
```

核心逻辑是：

- 如果是硬件绘制，那就通过 mHardwareRenderer draw，这个是在 render 线程；
- 如果是软件绘制，那就通过 drawSoftware 触发 view draw，这个是在主线程；

### 2.6.2 drawSoftware - 软件绘制

软件绘制开始：

```java
    private boolean drawSoftware(Surface surface, AttachInfo attachInfo, int xoff, int yoff,
            boolean scalingRequired, Rect dirty) {

        final Canvas canvas;
        try {
            final int left = dirty.left;
            final int top = dirty.top;
            final int right = dirty.right;
            final int bottom = dirty.bottom;
            //【1】锁定 canvas，canvas 的大小由 dirty 区域决定；
            canvas = mSurface.lockCanvas(dirty);

            // The dirty rectangle can be modified by Surface.lockCanvas()
            // noinspection ConstantConditions
            if (left != dirty.left || top != dirty.top || right != dirty.right
                    || bottom != dirty.bottom) {
                attachInfo.mIgnoreDirtyState = true;
            }

            //【2】设置像素信息；
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

核心逻辑：

- 锁定 canvas，并返回一个 Canvas，大小和 dirty 一样；
- 调用 DecorView 的 draw 方法开始绘制；

# 2-mid DecorView

## 2.1-mid onAttachedToWindow

DecorView 复写了这个方法：

```java
    @Override
    protected void onAttachedToWindow() {
        //【-->4.1.3】这里是先调用了父类 View 的方法；
        super.onAttachedToWindow();
        //【1】通过 PhoneWindow 获取回调 callback，实际上是 Activity，这里其实是触发
        // Activity.onAttachedToWindow；
        final Window.Callback cb = mWindow.getCallback();
        if (cb != null && !mWindow.isDestroyed() && mFeatureId < 0) {
            cb.onAttachedToWindow();
        }

        if (mFeatureId == -1)  // 针对于 activity 的情况；
			// 窗口已经被绑定，这里会尝试恢复以前可能已打开的所有 panels。
            // 在活动因配置更改而被杀死并且菜单已打开的情况下，将调用此方法。
            // 重新创建活动后，应再次显示菜单（没看懂）。
            mWindow.openPanelsAfterRestore();
        }

        if (!mWindowResizeCallbacksAdded) {
            //【2】这里会设置 window call back，就是 DecorView 到 ViewRootImpl 中；
            getViewRootImpl().addWindowCallbacks(this);
            mWindowResizeCallbacksAdded = true;
        } else if (mBackdropFrameRenderer != null) {
            // 由于配置更改，大小调整，这里是通知到渲染器；
            mBackdropFrameRenderer.onConfigurationChange();
        }
    }
```

这里的核心逻辑是：

- 调用了 Activity.onAttachedToWindow 方法；
- 将 DecorView 作为 callback 注册到 ViewRootImpl 中；



# 3 ViewGroup

由于 DecorView 是一个 FlameLayout，所以其本质是一个 ViewGroup，对于 DecorView，其 parent 是 ViewRootImpl，他需要 attach 到 ViewRootImpl 上才行：



## 3.1 dispatchAttachedToWindow - 核心（设置 attrachInfo）

将 AttachInfo 设置到 ViewGroup 中，这个方法是从 DecorView 开始出发的！

```java
    @Override
    void dispatchAttachedToWindow(AttachInfo info, int visibility) {
        mGroupFlags |= FLAG_PREVENT_DISPATCH_ATTACHED_TO_WINDOW;
        //【-->4.1】调用父类 View 的 dispatchAttachedToWindow；
        super.dispatchAttachedToWindow(info, visibility);
        mGroupFlags &= ~FLAG_PREVENT_DISPATCH_ATTACHED_TO_WINDOW;
        //【2】紧接着，处理 childs，可能是 View，也可能是 ViewGroup；
        final int count = mChildrenCount;
        final View[] children = mChildren;
        for (int i = 0; i < count; i++) {
            final View child = children[i];
            //【-->3.1/-->4.1】调用 childs(ViewGroup/View)的 dispatchAttachedToWindow
            // 第二个参数：可见性，这里会在父亲可见性和 child 可见性中求最大值； 
            child.dispatchAttachedToWindow(info,
                    combineVisibility(visibility, child.getVisibility()));
        }
        //【3】对 TransientView 也会这样处理；
        final int transientCount = mTransientIndices == null ? 0 : mTransientIndices.size();
        for (int i = 0; i < transientCount; ++i) {
            View view = mTransientViews.get(i);
            //【-->3.1/-->4.1】调用 childs(ViewGroup/View)的 dispatchAttachedToWindow
            // 第二个参数：可见性，这里会在父亲可见性和 child 可见性中求最大值；
            view.dispatchAttachedToWindow(info,
                    combineVisibility(visibility, view.getVisibility()));
        }
    }
```

可以看到，从 DecorView 开始对每一个 Child 都会 dispatchAttachedToWindow，将 mAttachInfo 设置进去；

## 3.2 dispatchWindowVisibilityChanged

分发窗口改变给所有的 child：

```java
    @Override
    public void dispatchWindowVisibilityChanged(int visibility) {
        //【-->4.2】自身父类的方法；
        super.dispatchWindowVisibilityChanged(visibility);
        final int count = mChildrenCount;
        final View[] children = mChildren;
        for (int i = 0; i < count; i++) {
            //【-->3.2/-->4.2】遍历每个孩子；
            children[i].dispatchWindowVisibilityChanged(visibility);
        }
    }
```



## 3.3 dispatchVisibilityAggregated

执行可见性合并给所有的 child：

```java
    @Override
    boolean dispatchVisibilityAggregated(boolean isVisible) {
        //【-->4.3】自身父类的方法；
        isVisible = super.dispatchVisibilityAggregated(isVisible);
        final int count = mChildrenCount;
        final View[] children = mChildren;
        for (int i = 0; i < count; i++) {
            //【-->3.2/-->4.2】遍历每个孩子；这里只会遍历看些用户可见的 view，因为不可见的 view 和他们 child tree
            // 清楚自身是不可见的，那么他们的不可见的状态不会变化；
            if (children[i].getVisibility() == VISIBLE) {
                children[i].dispatchVisibilityAggregated(isVisible);
            }
        }
        return isVisible;
    }
```



# 4 View

ViewGroup 的父类就是 View：

## 4.1 dispatchAttachedToWindow - 核心（设置 attrachInfo）

这里来看看 View 中的操作：

```java
    void dispatchAttachedToWindow(AttachInfo info, int visibility) {
        //【1-core】将 info 保存到 View 内部的 mAttachInfo 中；
        mAttachInfo = info;
        if (mOverlay != null) {
            mOverlay.getOverlayView().dispatchAttachedToWindow(info, visibility);
        }
        //【2】计数 +1；
        mWindowAttachCount++;

        // 下面可能会修改 drawable state 所以这里好设置标记，下面会更新；
        mPrivateFlags |= PFLAG_DRAWABLE_STATE_DIRTY;
        //【3】如果 view 指定了单独的 TreeObserver，那么这里会进行一次合并；
        if (mFloatingTreeObserver != null) {
            //【-->5.1】合并 mFloatingTreeObserver 到 info.mTreeObserver 中；
            info.mTreeObserver.merge(mFloatingTreeObserver);
            mFloatingTreeObserver = null;
        }
        
        //【-->4.1.1】注册 Frame 监听器；
        registerPendingFrameMetricsObservers();

        if ((mPrivateFlags&PFLAG_SCROLL_CONTAINER) != 0) {
            mAttachInfo.mScrollContainers.add(this);
            mPrivateFlags |= PFLAG_SCROLL_CONTAINER_ADDED;
        }
  
        //【4】执行所有 View.post 添加的 Runnable 任务！
        if (mRunQueue != null) {
            mRunQueue.executeActions(info.mHandler);
            mRunQueue = null;
        }
        //【-->4.1.2】收集当前 view 的属性到 mAttachInfo 中，因为子 view 可能设置的属性和
        // DecorView 是不一样的；
        performCollectViewAttributes(mAttachInfo, visibility);
  
        //【-->4.1.3】触发生命周期函数 onAttachedToWindow，这里要重点说下，DecorView 覆盖了这个方法
        onAttachedToWindow();

        //【5】mListenerInfo 内部有很多的接口回调列表，他用来保存我们通过 View.setXXXXListener 设置的
        // 监听接口，比如 onClickLinsener 等等；
        ListenerInfo li = mListenerInfo;
        final CopyOnWriteArrayList<OnAttachStateChangeListener> listeners =
                li != null ? li.mOnAttachStateChangeListeners : null;
        if (listeners != null && listeners.size() > 0) {
            for (OnAttachStateChangeListener listener : listeners) {
                //【5.1】调用通过 setOnAttachStateChangeListener 设置的
                // OnAttachStateChangeListener.onViewAttachedToWindow 方法；
                listener.onViewAttachedToWindow(this);
            }
        }
        //【6】处理 window 可见性和 view 用户可见性的冲突；
        int vis = info.mWindowVisibility;
        if (vis != GONE) {
            //【-->4.1.4】生命周期 onWindowVisibilityChanged 方法；
            // 窗口可见性发生变化；
            onWindowVisibilityChanged(vis);
            //【-->4.1.5】判断 view 和其所有 parent 是否都用户可见，通过 current.mViewFlags 判断；
            if (isShown()) {
                //【-->4.1.6】因为 view 的用户可见性可能受 view 本身，parent view 或所属的 window 的影响。
                // 这里就会对二者的可见性做整合调整；
                onVisibilityAggregated(vis == VISIBLE);
            }
        }

        // 直接发送 onVisibilityChanged 而不是 dispatchVisibilityChanged。
        // 由于子树中的所有视图都将已经收到 dispatchAttachedToWindow，因此此处不再需要遍历子树。
        //【*7】生命周期函数 onVisibilityChanged 回调：说明 View 或者 view 的 parent 的可见性发生了变化；
        onVisibilityChanged(this, visibility);

        if ((mPrivateFlags&PFLAG_DRAWABLE_STATE_DIRTY) != 0) {
            //【*8】更新 view 的 drawable 状态；
            refreshDrawableState();
        }
        //【*9】判断是否需要重新计算属性
        needGlobalAttributesUpdate(false);
    }
```

我们来看看核心的流程：

- **将 AttachInfo info 保存到 View.mAttachInfo 中，表示已经绑定；**
- 该 view 的 mWindowAttachCount 引用计数加 1，表示已经 attach 到了 window 上；
- 合并 view 的 TreeObserver 内部的监听接口到 info.mTreeObserver 中；
- 执行所有 View.post 添加的 Runnable 任务；
- 触发生命周期函数 onAttachedToWindow；
- 如果 view 通过 setOnAttachStateChangeListener 设置了监听器，触发其 OnAttachStateChangeListener.onViewAttachedToWindow 方法；
- 触发 view 生命周期 onWindowVisibilityChanged 方法；
- 触发 view 生命周期 onVisibilityChanged 方法；



其他的一些函数：

- **onVisibilityChanged**

对于 onVisibilityChanged 方法，View 并没有实现具体的逻辑，参数 View changedView 表示可见变化的 view，可能是 view 自身，也可能是 view 的 parent；

```java
protected void onVisibilityChanged(@NonNull View changedView, @Visibility int visibility) {
}
```

- **refreshDrawableState**

刷新 Drawable State：

```java
public void refreshDrawableState() {
    mPrivateFlags |= PFLAG_DRAWABLE_STATE_DIRTY;
    //【2】会触发 view 的 drawableStateChanged 方法；
    drawableStateChanged();

    ViewParent parent = mParent;
    if (parent != null) {
        //【2】如果有 parent，也会刷新 parent 的 Drawable State；
        parent.childDrawableStateChanged(this);
    }
}
```

我们可以通过 getDrawableState 获取最新的 DrawableState；

- **needGlobalAttributesUpdate**

用于判断是否需要更新全局属性，参数 force 表示是否强制更新：

```java
void needGlobalAttributesUpdate(boolean force) {
    final AttachInfo ai = mAttachInfo;
    if (ai != null && !ai.mRecomputeGlobalAttributes) {
        //【1】更新的条件：强制、保持屏幕常亮、系统 ui 可见性、设置了 sys ui 监听器；
        if (force || ai.mKeepScreenOn || (ai.mSystemUiVisibility != 0)
            || ai.mHasSystemUiListeners) {
            ai.mRecomputeGlobalAttributes = true;
        }
    }
}
```

整个我们先不关注。



### 4.1.1 registerPendingFrameMetricsObservers

```java
    private void registerPendingFrameMetricsObservers() {
        if (mFrameMetricsObservers != null) {
            ThreadedRenderer renderer = getHardwareRenderer();
            if (renderer != null) {
                for (FrameMetricsObserver fmo : mFrameMetricsObservers) {
                    renderer.addFrameMetricsObserver(fmo);
                }
            } else {
                Log.w(VIEW_LOG_TAG, "View not hardware-accelerated. Unable to observe frame stats");
            }
        }
    }
```



### 4.1.2 performCollectViewAttributes

收集 view 的属性到 attachInfo 中：

```java
    void performCollectViewAttributes(AttachInfo attachInfo, int visibility) {
        if ((visibility & VISIBILITY_MASK) == VISIBLE) {
            if ((mViewFlags & KEEP_SCREEN_ON) == KEEP_SCREEN_ON) {
                attachInfo.mKeepScreenOn = true;
            }
            attachInfo.mSystemUiVisibility |= mSystemUiVisibility;
            ListenerInfo li = mListenerInfo;
            if (li != null && li.mOnSystemUiVisibilityChangeListener != null) {
                attachInfo.mHasSystemUiListeners = true;
            }
        }
    }
```





### 4.1.3 onAttachedToWindow

生命周期回调，表示 attached 到了 window 上：

```java
    @CallSuper
    //【-->2.1-mid】DecorView 复写了这个方法。
    protected void onAttachedToWindow() {
        if ((mPrivateFlags & PFLAG_REQUEST_TRANSPARENT_REGIONS) != 0) {
            mParent.requestTransparentRegion(this);
        }

        mPrivateFlags3 &= ~PFLAG3_IS_LAID_OUT;

        jumpDrawablesToCurrentState();

        resetSubtreeAccessibilityStateChanged();

        // rebuild, since Outline not maintained while View is detached
        rebuildOutline();

        if (isFocused()) {
            InputMethodManager imm = InputMethodManager.peekInstance();
            if (imm != null) {
                imm.focusIn(this);
            }
        }
    }
```

注意，这里对于 DecorView 有一些不一样，DecorView 本质上也是一个 View，而它复写了这个方法；

### 4.1.4 onWindowVisibilityChanged

窗口可见性发生变化，回调整个方法：

```java
    protected void onWindowVisibilityChanged(@Visibility int visibility) {
        if (visibility == VISIBLE) {
            //【1】触发 Scrollbars 去绘制，这里就不细分析了；
            initialAwakenScrollBars();
        }
    }
```

暂时不分析；



### 4.1.5 isShown

判断该 view 以及其所有的 parent view 是否都用户可见的；

```java
    public boolean isShown() {
        View current = this;
        do {
            //【1】只要有一个 view 不可见，那就回返回 false；
            if ((current.mViewFlags & VISIBILITY_MASK) != VISIBLE) {
                return false;
            }
            ViewParent parent = current.mParent;
            //【2】如果没有 attach 到 view root，也就是不再 view tree 上
            // 也返回 false；
            if (parent == null) {
                return false; 
            }
            //【3】不是 view 的子类，也就是 ViewRootImpl，这里返回 false；
            if (!(parent instanceof View)) {
                return true;
            }
            current = (View) parent;
        } while (current != null);

        return false;
    }
```

这里的 mViewFlags 表示的用户指定可见性，就是通过 android:visibable 或者 view.seVisibable 指定的可见性；



### 4.1.5 onVisibilityAggregated

isVisible 表示 window 是否是可见的；

```java
    @CallSuper
    public void onVisibilityAggregated(boolean isVisible) {
        if (isVisible && mAttachInfo != null) {
            //【1】触发 Scrollbars 去绘制，这里就不细分析了；
            initialAwakenScrollBars();
        }
        //【2】这里会对 Background Drawable 和 Foreground Drawable 的可见性做了调整；
        // 只要其可见性和 window 不一样那么，那就将其可见性设置成 window 可见性；
        final Drawable dr = mBackground;
        if (dr != null && isVisible != dr.isVisible()) {
            dr.setVisible(isVisible, false);
        }
        final Drawable fg = mForegroundInfo != null ? mForegroundInfo.mDrawable : null;
        if (fg != null && isVisible != fg.isVisible()) {
            fg.setVisible(isVisible, false);
        }
    }
```

调整 Background Drawable 和 Foreground Drawable 的可见性；

## 4.2 dispatchWindowVisibilityChanged

响应窗口可见性变化：

```java
    public void dispatchWindowVisibilityChanged(@Visibility int visibility) {
        //【-->4.1.4】窗口可见性发生了变化；
        onWindowVisibilityChanged(visibility);
    }
```

不多说；



## 4.3 dispatchVisibilityAggregated

响应可见性调整：

```java
    boolean dispatchVisibilityAggregated(boolean isVisible) {
        //【1】获取用户可见性；
        final boolean thisVisible = getVisibility() == VISIBLE;
        //【2】如果可见性不同，那就触发回调；
        if (thisVisible || !isVisible) {
            //【-->4.1.5】响应可见性调整；
            onVisibilityAggregated(isVisible);
        }
        return thisVisible && isVisible;
    }
```





## 4.4 MeasureSpec - 测量规格

MeasureSpec 是 View 的内部类。他表示一种测量规格，是父布局传递给子布局的布局要求。

### 4.4.1 测量模式

- **UNSPECIFIED**：不对 View 大小做限制，例如：ListView，ScrollView

```java
/**
   父视图没有对 child 施加任何约束。它可以是任何大小；
 */
public static final int UNSPECIFIED = 0 << MODE_SHIFT;
```

- **EXACTLY**：确切的大小，例如：100dp 或者 march_parent

```java
/**
   父视图已经确定了孩子的确切尺寸。不管孩子想要多大，都会给孩子以这些界限；
 */
public static final int EXACTLY     = 1 << MODE_SHIFT;
```

- **AT_MOST**：大小不可超过某数值，例如：wrap_content

```java
/**
   child 可以根据自身需要的大小而确定大小，但是存在上限，上限一般为父视图大小。
 */
public static final int AT_MOST     = 2 << MODE_SHIFT;
```



### 4.4.2 makeMeasureSpec

根据提供的大小和模式创建度量规范：

```java
public static int makeMeasureSpec(@IntRange(from = 0, to = (1 << MeasureSpec.MODE_SHIFT) - 1) int size,
                                  @MeasureSpecMode int mode) {
    //【1】使用旧的（破碎的）方式建立 MeasureSpecs，sUseBrokenMakeMeasureSpec 值为 false。
    if (sUseBrokenMakeMeasureSpec) { 
        return size + mode;
    } else {
        //【2】模式占高 2 位，大小占低 30 位，合成 MeasureSpec
        return (size & ~MODE_MASK) | (mode & MODE_MASK);
    }
}
```

我们知道，MeasureSpec 是 32 位的  Int 型，高两位表示 mode，低 30 位表示 size，这里的 MODE_MASK 的作用实际上就是做位操作！

MODE_MASK 取如下的值：

```java
private static final int MODE_SHIFT = 30;
private static final int MODE_MASK  = 0x3 << MODE_SHIFT;
```

其中 0x3 是十六进制，转为二进制是 11，向左移位30，结果是 11000.....0000（一共 30 个 0）！

- size & ~MODE_MASK：获取 size 的低 30 位；
- mode & MODE_MASK：获取 mode 的高两位；

最终合成度量规格；

# 5 ViewTreeObserver - 视图树的观察者

前面我们有分析过 ViewTreeObserver 用来动态监听布局的变化；

## 5.1 merge

合并 observer 中的接口回调到当前的 ViewTreeObserver 中：

```java
    void merge(ViewTreeObserver observer) {
        //【1】合并 mOnWindowAttachListeners 接口；
        if (observer.mOnWindowAttachListeners != null) {
            if (mOnWindowAttachListeners != null) {
                mOnWindowAttachListeners.addAll(observer.mOnWindowAttachListeners);
            } else {
                mOnWindowAttachListeners = observer.mOnWindowAttachListeners;
            }
        }
		//【2】合并 mOnWindowFocusListeners 接口；
        if (observer.mOnWindowFocusListeners != null) {
            if (mOnWindowFocusListeners != null) {
                mOnWindowFocusListeners.addAll(observer.mOnWindowFocusListeners);
            } else {
                mOnWindowFocusListeners = observer.mOnWindowFocusListeners;
            }
        }
        //【3】合并 mOnGlobalFocusListeners 接口；
        if (observer.mOnGlobalFocusListeners != null) {
            if (mOnGlobalFocusListeners != null) {
                mOnGlobalFocusListeners.addAll(observer.mOnGlobalFocusListeners);
            } else {
                mOnGlobalFocusListeners = observer.mOnGlobalFocusListeners;
            }
        }
        //【4】合并 mOnGlobalLayoutListeners 接口；
        if (observer.mOnGlobalLayoutListeners != null) {
            if (mOnGlobalLayoutListeners != null) {
                mOnGlobalLayoutListeners.addAll(observer.mOnGlobalLayoutListeners);
            } else {
                mOnGlobalLayoutListeners = observer.mOnGlobalLayoutListeners;
            }
        }
		//【5】合并 mOnPreDrawListeners 接口；
        if (observer.mOnPreDrawListeners != null) {
            if (mOnPreDrawListeners != null) {
                mOnPreDrawListeners.addAll(observer.mOnPreDrawListeners);
            } else {
                mOnPreDrawListeners = observer.mOnPreDrawListeners;
            }
        }
		//【6】合并 mOnTouchModeChangeListeners 接口；
        if (observer.mOnTouchModeChangeListeners != null) {
            if (mOnTouchModeChangeListeners != null) {
                mOnTouchModeChangeListeners.addAll(observer.mOnTouchModeChangeListeners);
            } else {
                mOnTouchModeChangeListeners = observer.mOnTouchModeChangeListeners;
            }
        }
        //【7】合并 mOnComputeInternalInsetsListeners 接口；
        if (observer.mOnComputeInternalInsetsListeners != null) {
            if (mOnComputeInternalInsetsListeners != null) {
                mOnComputeInternalInsetsListeners.addAll(observer.mOnComputeInternalInsetsListeners);
            } else {
                mOnComputeInternalInsetsListeners = observer.mOnComputeInternalInsetsListeners;
            }
        }
		//【8】合并 mOnScrollChangedListeners 接口；
        if (observer.mOnScrollChangedListeners != null) {
            if (mOnScrollChangedListeners != null) {
                mOnScrollChangedListeners.addAll(observer.mOnScrollChangedListeners);
            } else {
                mOnScrollChangedListeners = observer.mOnScrollChangedListeners;
            }
        }
		//【9】合并 mOnWindowShownListeners 接口；
        if (observer.mOnWindowShownListeners != null) {
            if (mOnWindowShownListeners != null) {
                mOnWindowShownListeners.addAll(observer.mOnWindowShownListeners);
            } else {
                mOnWindowShownListeners = observer.mOnWindowShownListeners;
            }
        }

        observer.kill();
    }
```

在将 mAttachInfo 设置到 view 去后，会将 view 自身设置的 ViewTreeObserver 的接口合并到 mAttachInfo.observer 中:



## 5.2 dispatchOnWindowAttachedChange

分发窗口绑定状态变化的消息：

```java
    final void dispatchOnWindowAttachedChange(boolean attached) {
        //【1】内部有一个 mOnWindowAttachListeners 列表，保存了所有的 view 设置的 OnWindowAttachListener 回调
        // merge 后统一回调；
        final CopyOnWriteArrayList<OnWindowAttachListener> listeners
                = mOnWindowAttachListeners;
        if (listeners != null && listeners.size() > 0) {
            for (OnWindowAttachListener listener : listeners) {
                //【2】根据是否 attached 的调用不同的接口；
                if (attached) listener.onWindowAttached();
                else listener.onWindowDetached();
            }
        }
    }
```

- **onWindowAttached**：绑定成功；
- **onWindowDetached**：接触绑定；

## 5.3 dispatchOnTouchModeChanged

通知注册的收听者触摸模式已更改。

```java
    final void dispatchOnTouchModeChanged(boolean inTouchMode) {
        //【1】内部有一个 mOnTouchModeChangeListeners 列表，保存了所有的 view 设置的 OnTouchModeChangeListener 回调
        // merge 后统一回调；
        final CopyOnWriteArrayList<OnTouchModeChangeListener> listeners =
                mOnTouchModeChangeListeners;
        if (listeners != null && listeners.size() > 0) {
            for (OnTouchModeChangeListener listener : listeners) {
                //【2】触发回调；
                listener.onTouchModeChanged(inTouchMode);
            }
        }
    }
```

参数表示是否 进入 / 退出 touch mode；

## 5.4 dispatchOnGlobalLayout

通知全局布局已经发生，在布局完成后会触发：

```java
// 通知已注册的侦听器全局布局已发生。
// 如果您要强制在未附加到 Window 或处于 GONE 状态的 View 或 View 层次结构上进行布局，则可以手动调用此方法。
public final void dispatchOnGlobalLayout() {
    final CopyOnWriteArray<OnGlobalLayoutListener> listeners = mOnGlobalLayoutListeners;
    if (listeners != null && listeners.size() > 0) {
        CopyOnWriteArray.Access<OnGlobalLayoutListener> access = listeners.start();
        try {
            int count = access.size();
            for (int i = 0; i < count; i++) {
                // 回调 onGlobalLayout 方法；
                access.get(i).onGlobalLayout();
            }
        } finally {
            listeners.end();
        }
    }
}
```



## 5.5 dispatchOnComputeInternalInsets

调用所有侦听器以计算当前插入边衬：

```java
    final void dispatchOnComputeInternalInsets(InternalInsetsInfo inoutInfo) {
        final CopyOnWriteArray<OnComputeInternalInsetsListener> listeners =
                mOnComputeInternalInsetsListeners;
        if (listeners != null && listeners.size() > 0) {
            CopyOnWriteArray.Access<OnComputeInternalInsetsListener> access = listeners.start();
            try {
                int count = access.size();
                for (int i = 0; i < count; i++) {
                    // 回调处理，计算结果保存到 inoutInfo 中；
                    access.get(i).onComputeInternalInsets(inoutInfo);
                }
            } finally {
                listeners.end();
            }
        }
    }
```



## 5.6 dispatchOnPreDraw

在 draw 之前会触发，用于取消回调：

```java
    @SuppressWarnings("unchecked")
    public final boolean dispatchOnPreDraw() {
        boolean cancelDraw = false;
        final CopyOnWriteArray<OnPreDrawListener> listeners = mOnPreDrawListeners;
        if (listeners != null && listeners.size() > 0) {
            CopyOnWriteArray.Access<OnPreDrawListener> access = listeners.start();
            try {
                int count = access.size();
                for (int i = 0; i < count; i++) {
                    //【1】回调 onPreDraw 方法；
                    cancelDraw |= !(access.get(i).onPreDraw());
                }
            } finally {
                listeners.end();
            }
        }
        return cancelDraw;
    }
```

# 6 Surface 相关

## 6.1 Surface

ViewRootImpl 内部有一个 Surface 变量：

```java
    final Surface mSurface = new Surface();
```

### 6.1.1 new Surface

创建了一个空的 Surface 实例，其内部的数据会将在 readFromParcel 方法中填充：

```java
public class Surface implements Parcelable {
	public Surface() {
    }
}
```

其实就是 requestWindow 会回传数据；

### 6.1.2 getGenerationId

获得该 surface 关联的 id 标志，如果 native 层的 surface 变化的话，那么这个 id 值会增加；

```java
    public int getGenerationId() {
        synchronized (mLock) {
            return mGenerationId; // 内部变量；
        }
    }
```

### 6.1.3 readFromParcel

relayoutWindow 会将 Surface 数据跨进程传递过来，初始化客户端的 Surface，Surface 其实就是一个 Parcel：

```java
    public void readFromParcel(Parcel source) {
        if (source == null) {
            throw new IllegalArgumentException("source must not be null");
        }

        synchronized (mLock) {
            // nativeReadFromParcel() will either return mNativeObject, or
            // create a new native Surface and return it after reducing
            // the reference count on mNativeObject.  Either way, it is
            // not necessary to call nativeRelease() here.
            // NOTE: This must be kept synchronized with the native parceling code
            // in frameworks/native/libs/Surface.cpp
            mName = source.readString();
            mIsSingleBuffered = source.readInt() != 0;
            setNativeObjectLocked(nativeReadFromParcel(mNativeObject, source));
        }
    }
```





# 7 总结

下面我们来通过 pic 看看整个过程的总结：

图先省略下。。。

# 遗留汇总

本篇文章木有跟踪和探究的很重要的源码：

## View 相关

```java
ViewGroup.measure(childWidthMeasureSpec, childHeightMeasureSpec); // 测量；

ViewGroup.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight()); // 布局；

View.requestFocus(View.FOCUS_FORWARD);  // 请求焦点；

view.requestLayout(); // 请求布局；

View.draw // 绘制；
```



## Window 相关

```java
// WindowSession 的创建；
mWindowSession = WindowManagerGlobal.getWindowSession();

// addToDisplay 的流程；
res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                                  getHostVisibility(), mDisplay.getDisplayId(),
                                  mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                                  mAttachInfo.mOutsets, mInputChannel);

// request 的流程；
int relayoutResult = mWindowSession.relayout(mWindow, mSeq, params,
   									 (int) (mView.getMeasuredWidth() * appScale + 0.5f),
  								     (int) (mView.getMeasuredHeight() * appScale + 0.5f),
  				  viewVisibility, insetsPending ? WindowManagerGlobal.RELAYOUT_INSETS_PENDING : 0,
    			  mWinFrame, mPendingOverscanInsets, mPendingContentInsets, mPendingVisibleInsets,
  				  mPendingStableInsets, mPendingOutsets, mPendingBackDropFrame, mPendingConfiguration,
                  mSurface);

// finishDrawing 的流程；
mWindowSession.finishDrawing(mWindow);
```

## Surface 相关

```java
// Surface 创建；
// WindowSession.relayout 执行后，Surface 如何变的有效的；
```


