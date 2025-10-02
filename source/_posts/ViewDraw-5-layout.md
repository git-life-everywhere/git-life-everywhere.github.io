# ViewDraw 第五篇 layout 流程分析

title: ViewDraw 第五篇 layout 流程分析
date: 2020/04/21 20:46:25 
catalog: true
categories: 
- View 视图
- View 的加载和绘制

tags: ViewDraw
copyright: true

------

基于 Android N 分析下 View 的 layout，N 虽然看起来略有些旧，但是框架的核心思想才是最重要的，新的一天开始了。

# 1 回顾

我们来回顾下，performLayout 请求布局的地方：

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
            //【1-core】调用 DecorView（ViewGroup）的 layout；
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

可以看到，在测量后布局阶段：

```java
//【-->2.1】调用 DecorView（ViewGroup）的 layout；
host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());
```

是通过 view 的 layout 方法出发布局的；

# 2 ViewGroup

## 2.1 layout - 核心

DecorView 是一个 ViewGroup，所以会先执行 ViewGroup 的 layout：

```java
    @Override
    public final void layout(int l, int t, int r, int b) {
        if (!mSuppressLayout && (mTransition == null || !mTransition.isChangingLayout())) {
            if (mTransition != null) {
                mTransition.layoutChange(this);
            }
            //【-->2.1】执行 view 的 layout 方法：
            super.layout(l, t, r, b);
        } else {
            // record the fact that we noop'd it; request layout when transition finishes
            mLayoutCalledWhileSuppressed = true;
        }
    }
```

ViewGroup -> View

# 3 View

## 3.1 layout - 核心

执行布局，参数是左上角，右下角的测量坐标，是相对于 parent 的；

```java
@SuppressWarnings({"unchecked"})
public void layout(int l, int t, int r, int b) {
    //【1】如果设置了这个的 flags 的话，那么在 measure 阶段实际上是没有执行，而是放到了 layout 阶段
    // 这个 flags 是在 measure 时期判断是否；使用缓存的时候调用的；
    if ((mPrivateFlags3 & PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT) != 0) {
        //【1.1】执行测量操作，这里就不再分析了；
        onMeasure(mOldWidthMeasureSpec, mOldHeightMeasureSpec);
        mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
    }

    int oldL = mLeft;
    int oldT = mTop;
    int oldB = mBottom;
    int oldR = mRight;

    boolean changed = isLayoutModeOptical(mParent) ? // 这里是针对于 Optical bounds 的处理，先不关注；
            setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);

    //【2】如果设置了 PFLAG_LAYOUT_REQUIRED 标志位的话，那么就要执行布局了。
    // 这个 flags 依然是在 measure 阶段设置的；
    if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
        //【-->4.1-->5.1】执行布局，这里 DecorView，FlameLayout 均复写了这个方法；
        onLayout(changed, l, t, r, b);

        if (shouldDrawRoundScrollbar()) {
            if(mRoundScrollbarRenderer == null) {
                mRoundScrollbarRenderer = new RoundScrollbarRenderer(this);
            }
        } else {
            mRoundScrollbarRenderer = null;
        }

        mPrivateFlags &= ~PFLAG_LAYOUT_REQUIRED;

        ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnLayoutChangeListeners != null) {
            ArrayList<OnLayoutChangeListener> listenersCopy =
                    (ArrayList<OnLayoutChangeListener>)li.mOnLayoutChangeListeners.clone();
            int numListeners = listenersCopy.size();
            for (int i = 0; i < numListeners; ++i) {
                //【2.2】如果有注册 LayoutChangeListener 的话，这里会回调通知；
                listenersCopy.get(i).onLayoutChange(this, l, t, r, b, oldL, oldT, oldR, oldB);
            }
        }
    }

    mPrivateFlags &= ~PFLAG_FORCE_LAYOUT;
    mPrivateFlags3 |= PFLAG3_IS_LAID_OUT;
}
```

所以这里的核心的方法就是执行了 onLayout 方法：



## 3.2 onLayout - 核心

可以看到 view 的 onLayout  方法是一个空的实现!!

- 当 View 需要为每个 child view 分配大小和位置时，会通过 layout 方法调用。
- 有 child 的 ViewGroup 要重写该方法，并在每个子级上调用布局。

参数分析：

- boolean changed：表示 VIew 的大小或位置是否发生了变化；
- int left：相对于父级的左侧位置
- int top：相对于父级的顶部位置
- int right：相对于父级的右侧位置
- int bottom：相对于父级的底部位置

```java
    protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
    }
```

不多说；

## 3.3 getLayoutDirection


用于获取水平的布局方向：

```java
    @ViewDebug.ExportedProperty(category = "layout", mapping = {
        @ViewDebug.IntToString(from = LAYOUT_DIRECTION_LTR, to = "RESOLVED_DIRECTION_LTR"),
        @ViewDebug.IntToString(from = LAYOUT_DIRECTION_RTL, to = "RESOLVED_DIRECTION_RTL")
    })
    @ResolvedLayoutDir
    public int getLayoutDirection() {
        final int targetSdkVersion = getContext().getApplicationInfo().targetSdkVersion;
        //【1】当目标 sdk 小于 JELLY_BEAN_MR1，默认返回的是 LTR；
        if (targetSdkVersion < JELLY_BEAN_MR1) {
            mPrivateFlags2 |= PFLAG2_LAYOUT_DIRECTION_RESOLVED;
            return LAYOUT_DIRECTION_RESOLVED_DEFAULT;
        }
        //【2】否则，根据属性色设置返回；
        return ((mPrivateFlags2 & PFLAG2_LAYOUT_DIRECTION_RESOLVED_RTL) ==
                PFLAG2_LAYOUT_DIRECTION_RESOLVED_RTL) ? LAYOUT_DIRECTION_RTL : LAYOUT_DIRECTION_LTR;
    }
```

返回值是通过 setLayoutDirection 方法或者 android:layoutDirection 设置的纸；

```java
// 水平方向从左到右；
public static final int LAYOUT_DIRECTION_LTR = LayoutDirection.LTR;

// 水平方向从右到左（相反）
public static final int LAYOUT_DIRECTION_RTL = LayoutDirection.RTL;

// 默认方向：LTR
static final int LAYOUT_DIRECTION_RESOLVED_DEFAULT = LAYOUT_DIRECTION_LTR;
```

# 4 DecorView

## 4.1 onLayout - 核心

```java
    @Override
    protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
        //【-->5.1】进入 FrameLayout 中，进行布局；
        super.onLayout(changed, left, top, right, bottom);
        //【1】处理下 mOutsets 的数据；
        getOutsets(mOutsets);
        if (mOutsets.left > 0) {
            offsetLeftAndRight(-mOutsets.left);
        }
        if (mOutsets.top > 0) {
            offsetTopAndBottom(-mOutsets.top);
        }
        if (mApplyFloatingVerticalInsets) {
            offsetTopAndBottom(mFloatingInsets.top);
        }
        if (mApplyFloatingHorizontalInsets) {
            offsetLeftAndRight(mFloatingInsets.left);
        }

        // If the application changed its SystemUI metrics, we might also have to adapt
        // our shadow elevation.
        updateElevation();
        mAllowUpdateElevation = true;

        if (changed && mResizeMode == RESIZE_MODE_DOCKED_DIVIDER) {
            getViewRootImpl().requestInvalidateRootRenderNode();
        }
    }
```

继续看～

# 5 FrameLayout

## 5.1 onLayout - 核心

我们看到 child layout 实际上是在 FrameLayout 中发起：

```java
    @Override
    protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
        //【-->5.2】进行 child layout 布局；
        layoutChildren(left, top, right, bottom, false /* no force left gravity */);
    }
```



## 5.2 layoutChildren - 核心

对 child 进行布局：

```java
    void layoutChildren(int left, int top, int right, int bottom, boolean forceLeftGravity) {
        //【1】获取到所有的孩子；
        final int count = getChildCount();
        //【2】child 的坐标是基于 parent 的，所以可以看到这里计算了基于 parent 的坐标；
        final int parentLeft = getPaddingLeftWithForeground();
        final int parentRight = right - left - getPaddingRightWithForeground();

        final int parentTop = getPaddingTopWithForeground();
        final int parentBottom = bottom - top - getPaddingBottomWithForeground();
        //【3】遍历每一个 child view；
        for (int i = 0; i < count; i++) {
            final View child = getChildAt(i);
            if (child.getVisibility() != GONE) {
                //【4】获取到 child 的布局参数；
                final LayoutParams lp = (LayoutParams) child.getLayoutParams();
                //【5】获取到 child 的测量宽/高，基于此决定 right/bottom；
                final int width = child.getMeasuredWidth();
                final int height = child.getMeasuredHeight();

                int childLeft;
                int childTop;
   
                //【-->2.3】根据布局参数 LayoutParams 的 gravity 属性，就是 android:layout_gravity
                // 如果没有指定的话，那么默认是 DEFAULT_CHILD_GRAVITY，也就是左上角开始；
                int gravity = lp.gravity;
                if (gravity == -1) {
                    gravity = DEFAULT_CHILD_GRAVITY;
                }

                //【-->3.3】返回 view 的布局方向：LTR 或者 RTL
                final int layoutDirection = getLayoutDirection();
                //【-->6.1】将相对对其转为绝对对齐；
                final int absoluteGravity = Gravity.getAbsoluteGravity(gravity, layoutDirection);
                //【6】获取 TOP BOTTOM 的设置情况
                final int verticalGravity = gravity & Gravity.VERTICAL_GRAVITY_MASK;

                //【7】获取 LEFT RIGHT 的设置情况，来计算 childLeft 值； 
                switch (absoluteGravity & Gravity.HORIZONTAL_GRAVITY_MASK) {
                    case Gravity.CENTER_HORIZONTAL:
                        //【7.1】如果设置了 CENTER_HORIZONTAL，那么显然是水平居中了
                        childLeft = parentLeft + (parentRight - parentLeft - width) / 2 +
                        lp.leftMargin - lp.rightMargin;
                        break;
                    case Gravity.RIGHT:
                        //【7.2】如果是 RIGHT，那就要靠右布局了；
                        if (!forceLeftGravity) {
                            childLeft = parentRight - width - lp.rightMargin;
                            break;
                        }
                    case Gravity.LEFT:
                    default:
                        //【7.3】如果是 LEFT 或者其他，那就要靠左布局了；
                        childLeft = parentLeft + lp.leftMargin;
                }

				//【8】获取 TOP BOTTOM 的设置情况，来计算 childTop 值； 
                switch (verticalGravity) {
                    case Gravity.TOP:
                        //【7.1】如果设置了 TOP，那么显然是靠上布局了；
                        childTop = parentTop + lp.topMargin;
                        break;
                    case Gravity.CENTER_VERTICAL:
                        //【7.2】如果设置了 CENTER_VERTICAL，那么显然是垂直居中布局了；
                        childTop = parentTop + (parentBottom - parentTop - height) / 2 +
                        lp.topMargin - lp.bottomMargin;
                        break;
                    case Gravity.BOTTOM:
                        //【7.3】如果设置了 BOTTOM，那么显然是垂直居中布局了；
                        childTop = parentBottom - height - lp.bottomMargin;
                        break;
                    default:
                        childTop = parentTop + lp.topMargin;
                }
                //【-->3.1/2.1】这有会执行 child view 的 layout 执行布局，注意 child 可能 view 也可能是 view group
                child.layout(childLeft, childTop, childLeft + width, childTop + height);
            }
        }
    }
```

DEFAULT_CHILD_GRAVITY 的取值如下：

```java
    private static final int DEFAULT_CHILD_GRAVITY = Gravity.TOP | Gravity.START;
```

对于 Gravity 的 TOP BOTTOM LEFT RIGHT 取值如下：

```java
    /** Push object to the top of its container, not changing its size. */
    public static final int TOP = (AXIS_PULL_BEFORE|AXIS_SPECIFIED)<<AXIS_Y_SHIFT;
    /** Push object to the bottom of its container, not changing its size. */
    public static final int BOTTOM = (AXIS_PULL_AFTER|AXIS_SPECIFIED)<<AXIS_Y_SHIFT;
    /** Push object to the left of its container, not changing its size. */
    public static final int LEFT = (AXIS_PULL_BEFORE|AXIS_SPECIFIED)<<AXIS_X_SHIFT;
    /** Push object to the right of its container, not changing its size. */
    public static final int RIGHT = (AXIS_PULL_AFTER|AXIS_SPECIFIED)<<AXIS_X_SHIFT;
```

所以可以根据上看的  gravity & Gravity.VERTICAL_GRAVITY_MASK 操作获取的是什么了：

```java
    // 获取 LEFT RIGHT 的设置情况
    public static final int HORIZONTAL_GRAVITY_MASK = (AXIS_SPECIFIED |
            AXIS_PULL_BEFORE | AXIS_PULL_AFTER) << AXIS_X_SHIFT;
    // 获取 TOP BOTTOM 的设置情况
    public static final int VERTICAL_GRAVITY_MASK = (AXIS_SPECIFIED |
            AXIS_PULL_BEFORE | AXIS_PULL_AFTER) << AXIS_Y_SHIFT;
```

不多说了～

# 6 Gravity

## 6.1 getAbsoluteGravity

将相对对齐转为绝对对齐，根据 RTL 或者 LTR 进行设置：

- 如果水平方向为 LTR，则 START 将设置 LEFT，而 END 将设置 RIGHT。
- 如果水平方向是 RTL，则 START 将设置 RIGHT，而 END 将设置 LEFT。

```java
    public static int getAbsoluteGravity(int gravity, int layoutDirection) {
        int result = gravity;
        //【1】判断下对齐方式是不是相对的，start 和 end 多一个 RELATIVE_LAYOUT_DIRECTION 的标志位；
        if ((result & RELATIVE_LAYOUT_DIRECTION) > 0) {
            if ((result & Gravity.START) == Gravity.START) {
                //【1.1】去掉 START，如果是 RTL，那么就设置成 RIGHT，不是就是 LEFT
                result &= ~START;
                if (layoutDirection == View.LAYOUT_DIRECTION_RTL) {
                    result |= RIGHT;
                } else {

                    result |= LEFT;
                }
            } else if ((result & Gravity.END) == Gravity.END) {
                //【1.1】去掉 END，如果是 RTL，那么就设置成 LEFT，不是就是 RIGHT
                result &= ~END;
                if (layoutDirection == View.LAYOUT_DIRECTION_RTL) {
                    result |= LEFT;
                } else {
                    result |= RIGHT;
                }
            }
            //【2】去掉相对标志位
            result &= ~RELATIVE_LAYOUT_DIRECTION;
        }
        return result;
    }
```
- 关于 gravity：

这里的 gravity 是 android:layout_gravity 的取值，就是相对 parent 的布局比重；




- 关于绝对/相对对齐

我们在设置 view 的 android:gravity＝"" 的时候，会设置比如 “left|bottom” 这样的值，那么这些值具体的意思是什么呢？

left 和 right 代表一种绝对的对齐，而 start 和 end 表示基于阅读顺序的对齐。

如何理解呢，我们可以通过 android:layoutDirection="rtl" 来设置视图的排列顺序，对于阿拉伯语的话，需要 RTL 的显示方式的。

当使用 left 的时候，无论是 LTR 还是 RTL，总是左对齐的；

而使用 start，在 LTR 中是左对齐，而在 RTL 中则是右对齐。



# 7 总结

我们用一张图直观的看下 layout 的流程：

<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/23/View.layout-process.png" alt="View.layout-process" style="zoom: 33%;" />