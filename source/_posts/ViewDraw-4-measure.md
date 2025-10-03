# ViewDraw 第四篇 measure 流程分析

title: ViewDraw 第四篇 measure 流程分析
categories:
  - View 视图
  - View 的加载和绘制
tags:
  - ViewDraw
date: 2020/04/15 20:46:25 
---

本篇文章基于 Android N - 7.1.1 主要分析下 measure 方法的执行流程；

# 1 回顾

我们来回顾下，performMeasure 请求测量的地方：


这里的参数：childWidthMeasureSpec 和 childHeightMeasureSpec 是 root view 也就是 DecorView 的测量标准！

```java
private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
    Trace.traceBegin(Trace.TRACE_TAG_VIEW, "measure");
    try {
        //【-->2.1】最终调用了 DecorView（ViewGroup）的 measure 方法，并将测量规范传递下去；
        // 这里的 childWidthMeasureSpec 和 childHeightMeasureSpec 是 parent 指定的 child view 的测量规格，也就是 DecorView 的；
        mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
}
```

mView 就是我们 DecorView，然而 DecorView 有如下的继承实现关系：

```java
public class DecorView extends FrameLayout implements RootViewSurfaceTaker, WindowCallbacks {
public class FrameLayout extends ViewGroup {
public abstract class ViewGroup extends View implements ViewParent, ViewManager {
```

实际上 measure 是 view 的方法；



## 1.1 MeasureSpec - 测量规格

MeasureSpec 是 View 的内部类。他表示一种测量规格。MeasureSpec 由测量模式 mode 和测量大小 size 组成；

**View 的测量规格是由 parent view 的测量规格和自身的 LayoutParams 共同决定的**。

### 1.1.1 测量模式

- **UNSPECIFIED**：父视图不对 View 大小做限制，当然并不是真的说想要多大最后就真有多大，例如：ListView，ScrollView；

```java
// 父视图没有对 child 施加任何约束。它可以是任何大小；
public static final int UNSPECIFIED = 0 << MODE_SHIFT;
```

- **EXACTLY**：父试图已经知道了 view 确切的大小，例如：100dp 或者 march_parent

```java
// 父视图已经确定了孩子的确切尺寸。不管孩子想要多大，都会给孩子以这些界限；
public static final int EXACTLY     = 1 << MODE_SHIFT;
```

- **AT_MOST**：大小不可超过某数值，例如：wrap_content

```java
// child 可以根据自身需要的大小而确定大小，但是存在上限，上限一般为父视图大小。
public static final int AT_MOST     = 2 << MODE_SHIFT;
```



### 1.1.2 makeMeasureSpec

根据测量大小 size 和测量模式 mode 创建测量规格：

```java
public static int makeMeasureSpec(@IntRange(from = 0, to = (1 << MeasureSpec.MODE_SHIFT) - 1) int size,
                                  @MeasureSpecMode int mode) {
    //【1】使用旧的方式建立 MeasureSpecs，sUseBrokenMakeMeasureSpec 值默认为 false。
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

最终合成测量规格；

# 2 View

## 2.1 measure - 核心1

View 的这个方法是被它的 parent view 调用的，而 widthMeasureSpec 和 heightMeasureSpec 则是 parent 指定的当前 view 的测量规格；

```java
    public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
        boolean optical = isLayoutModeOptical(this);
        if (optical != isLayoutModeOptical(mParent)) { // 这里是对其做一个调整，我们先不看；
            Insets insets = getOpticalInsets();
            int oWidth  = insets.left + insets.right;
            int oHeight = insets.top  + insets.bottom;
            widthMeasureSpec  = MeasureSpec.adjust(widthMeasureSpec,  optical ? -oWidth  : oWidth);
            heightMeasureSpec = MeasureSpec.adjust(heightMeasureSpec, optical ? -oHeight : oHeight);
        }

        //【1】计算缓存的 key，高 32 低 32 位交错取值；
        long key = (long) widthMeasureSpec << 32 | (long) heightMeasureSpec & 0xffffffffL;
        if (mMeasureCache == null) mMeasureCache = new LongSparseLongArray(2);
        //【2】判断是否强制布局；
        final boolean forceLayout = (mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT;

        // Optimize layout by avoiding an extra EXACTLY pass when the view is
        // already measured as the correct size. In API 23 and below, this
        // extra pass is required to make LinearLayout re-distribute weight.
        // 如果视图已经被测量获得了正确的尺寸，那么这里会判断下测量模式是否是 EXACTLY 如果是的话，那么就可能不会重新测量；
        // 在 API 23 及以下版本中，需要这样的额外遍历才能使 LinearLayout 重新分配权重。
        final boolean specChanged = widthMeasureSpec != mOldWidthMeasureSpec
                || heightMeasureSpec != mOldHeightMeasureSpec; // 测量标准是否变化
        final boolean isSpecExactly = MeasureSpec.getMode(widthMeasureSpec) == MeasureSpec.EXACTLY
                && MeasureSpec.getMode(heightMeasureSpec) == MeasureSpec.EXACTLY; // 是否是精确布局
        final boolean matchesSpecSize = getMeasuredWidth() == MeasureSpec.getSize(widthMeasureSpec)
                && getMeasuredHeight() == MeasureSpec.getSize(heightMeasureSpec); // 尺寸是否变化
        final boolean needsLayout = specChanged // 判断是否需要布局；
                && (sAlwaysRemeasureExactly || !isSpecExactly || !matchesSpecSize);

        if (forceLayout || needsLayout) {
            // 这里会清除掉测量的标志位 measured dimension flag；
            mPrivateFlags &= ~PFLAG_MEASURED_DIMENSION_SET;

            resolveRtlPropertiesIfNeeded(); // 处理 RTL 左右翻转属性；
            
            //【3】判断是否需要使用缓存，forceLayout 的话，或者忽视缓存的话，那就会使用本次新的测量模式；
            int cacheIndex = forceLayout ? -1 : mMeasureCache.indexOfKey(key);
            if (cacheIndex < 0 || sIgnoreMeasureCache) {
                //【-->3.1】开始测量自身，这里会把清除掉测量的标志位 measured dimension flag 设置回去；
                // 这里会先调用 DecorView 的 onMeasure 方法；
                // (当然如果是 view 的话，这里会直接进入 view 的 onMeasure 的【-->2.2】)
                onMeasure(widthMeasureSpec, heightMeasureSpec);
                mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;

            } else {
                long value = mMeasureCache.valueAt(cacheIndex);
                // Casting a long to int drops the high 32 bits, no mask needed
                setMeasuredDimensionRaw((int) (value >> 32), (int) value);
                mPrivateFlags3 |= PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
            }

            // 如果 measured dimension flag 没有设置或者 setMeasuredDimension 方法咩有执行，抛出异常；
            if ((mPrivateFlags & PFLAG_MEASURED_DIMENSION_SET) != PFLAG_MEASURED_DIMENSION_SET) {
                throw new IllegalStateException("View with id " + getId() + ": "
                        + getClass().getName() + "#onMeasure() did not set the"
                        + " measured dimension by calling"
                        + " setMeasuredDimension()");
            }

            mPrivateFlags |= PFLAG_LAYOUT_REQUIRED; // 设置需要布局的标志
        }
		//【4】保存本次的测量模式；
        mOldWidthMeasureSpec = widthMeasureSpec;
        mOldHeightMeasureSpec = heightMeasureSpec;

        //【5】将其保存到布局缓存中；
        mMeasureCache.put(key, ((long) mMeasuredWidth) << 32 |
                (long) mMeasuredHeight & 0xffffffffL); // suppress sign extension
    }
```

执行测量！



## 2.2 onMeasure - 核心2.1

view 的 onMeasure 实际上很简单：

```java
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        //【-->2.2.1】设置测量后的宽高像素值；
        //【-->2.2.3】获取默认的宽/高；
        setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
                getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
    }
```

该方法用于测量视图及其内容，以确定测量的宽度和测量的高度。 

measure 方法会调用此方法，并且子类必须重写此方法，同时提供对其自身的准确测量。

复写此方法时， 必须调用 setMeasuredDimension 来存储此视图的测量宽度和高度。否则，将抛出 IllegalStateException 的异常。

调用超类的 onMeasure 是有效的用法。

子类应重写 onMeasure，以提供对其内容更好的度量。

如果重写此方法，则子类必须要确保测量的高度和宽度至少为视图的最小高度和宽度 getSuggestedMinimumHeight  getSuggestedMinimumWidth

### 2.2.1 setMeasuredDimension - 核心7

保存测量的宽度和测量的高度：

```java
    protected final void setMeasuredDimension(int measuredWidth, int measuredHeight) {
        boolean optical = isLayoutModeOptical(this);
        if (optical != isLayoutModeOptical(mParent)) { // 这部份是根据 outInsert 调整；
            Insets insets = getOpticalInsets();
            int opticalWidth  = insets.left + insets.right;
            int opticalHeight = insets.top  + insets.bottom;

            measuredWidth  += optical ? opticalWidth  : -opticalWidth;
            measuredHeight += optical ? opticalHeight : -opticalHeight;
        }
        //【-->2.2.2】继续设置；
        setMeasuredDimensionRaw(measuredWidth, measuredHeight);
    }
```

这个方法必须要被 onMeasure(int, int) 调用，不然的话会抛出异常；

### 2.2.2 setMeasuredDimensionRaw - 核心8

存储测量的宽度和测量的高度：

```java
    private void setMeasuredDimensionRaw(int measuredWidth, int measuredHeight) {
        mMeasuredWidth = measuredWidth;
        mMeasuredHeight = measuredHeight;
        //【1】更新标志位，不然上面会抛异常的；
        mPrivateFlags |= PFLAG_MEASURED_DIMENSION_SET;
    }
```

### 2.2.3 getDefaultSize - 核心3.1

参数 size 表示的是这个 view 默认的测量大小，measureSpec 则是 view 的测量规格，返回值表示这个 view 测量大小: 

```java
    public static int getDefaultSize(int size, int measureSpec) {
        int result = size;
        //【1】获取测量约束指定的模式和距离；
        int specMode = MeasureSpec.getMode(measureSpec);
        int specSize = MeasureSpec.getSize(measureSpec);

        switch (specMode) {
        //【2】如果是 view 的测量模式是 UNSPECIFIED 数值，那么就去取自己的默认值；
        case MeasureSpec.UNSPECIFIED:
            result = size;
            break;
        //【3】如果是其他的两种情况，取测量规格指定的大小；       
        case MeasureSpec.AT_MOST:
        case MeasureSpec.EXACTLY:
            result = specSize;
            break;
        }
        return result;
    }
```

可以看到，自定义 view，重写 onMeasure 方法的，如果不针对 AT_MOST 和 EXACTLY 做处理，那么使用 wrap_content 时与 match_parent 的效果将会是一样；

#### 2.2.3.1 getSuggestedMinimumXXXX

返回最小的宽/高：

```java
    protected int getSuggestedMinimumWidth() {
        return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinimumWidth());
    }
    
    protected int getSuggestedMinimumHeight() {
        return (mBackground == null) ? mMinHeight : max(mMinHeight, mBackground.getMinimumHeight());

    }
```

如果有设置背景，则获取背景 Background 的宽/高度，如果没有设置背景，则取 xml 中 android:minWidth/mMinHeight 的值；

## 2.3 combineMeasuredStates

合并测量状态：

```java
    public static int combineMeasuredStates(int curState, int newState) {
        return curState | newState;
    }
```



## 2.4 getMeasuredState

返回测量状态，实际上就是宽和高的复合整型数；

```java
    public final int getMeasuredState() {
        return (mMeasuredWidth & MEASURED_STATE_MASK) // 取宽的高 8 位；
                | ((mMeasuredHeight >> MEASURED_HEIGHT_STATE_SHIFT) // 取高的高 8 位置，然后右移了 16 位；
                        & (MEASURED_STATE_MASK >> MEASURED_HEIGHT_STATE_SHIFT));
    }
```

int 是 32 位，可以看到：

- 将 mMeasuredWidth 左起高 8 位作为宽的测量状态；

- 将 mMeasuredHeight 左起高 8 位作为高的测量状态；

但它将二者存到了一个复合整型中，对于 mMeasuredWidth 取左起高 8 位，mMeasuredHeight 也是左起高 8 位；但是右移了 16 位；

```java
    // 高 8 位都是 1；
    public static final int MEASURED_STATE_MASK = 0xff000000;    

    // 用于将宽高合并成一个复合整型；
    public static final int MEASURED_HEIGHT_STATE_SHIFT = 16; // 用于将高的测量值右移 16 位；
```

实际结果如下：

`wwww 0000 hhhh 0000`



## 2.5 resolveSizeAndState-核心6

回顾下前面的代码，

```java
        // 这里的 maxWidth 和 maxHeight 表示了 child view 中最大的宽度和高度，那么对于 view group，
        // 需要基于这个值来设置自身的宽高；
        setMeasuredDimension(resolveSizeAndState(maxWidth, widthMeasureSpec, childState), // 宽的测量状态在高 8 位；
                resolveSizeAndState(maxHeight, heightMeasureSpec,
                        childState << MEASURED_HEIGHT_STATE_SHIFT)); // 高的测量状态在第三个 8 位，所以要左移 16 位；
```

参数 size：表示的是这个 view 想显示的大小；

参数 measureSpec：是 parent view 指定的测量规格（或者是自身的测量规格，这个要看情况而定）；

参数 childMeasuredState：为 child view 的测量状态；

此时上面的代码调用，我们要基于 child view 的测量结果，设置 view group 的宽高；

```java
    public static int resolveSizeAndState(int size, int measureSpec, int childMeasuredState) {
        //【1】获取测量规格指定的测量模式和测量大小；
        final int specMode = MeasureSpec.getMode(measureSpec);
        final int specSize = MeasureSpec.getSize(measureSpec);
        final int result;
        switch (specMode) {
            //【2】如果此时 view (parent) 的测量模式是 AT_MOST，那么说明 view (parent) 最大不能超过 specSize；
            case MeasureSpec.AT_MOST:
                if (specSize < size) {
                    //【2.1】如果 view (parent) 的测量大小 specSize 小于 view 实际想要 size，
                    // 应该以测量大小 specSize 为准，同时设置 MEASURED_STATE_TOO_SMALL 标志位；
                    result = specSize | MEASURED_STATE_TOO_SMALL;
                } else {
                    //【2.2】否则的话，测量大小为 view 期望的大小 size；
                    result = size;
                }
                break;
            case MeasureSpec.EXACTLY:
                //【3】如果此时 view (parent) 的测量模式为 EXACTLY，那么 view (parent) 最大不能超过 specSize；
                result = specSize;
                break;
                //【4】未指定为 size
            case MeasureSpec.UNSPECIFIED:
            default:
                result = size;
        }
        //【5】最后的结果是一个复合整数，这里会 | 上 child 的测量状态（高 8 位）；
        return result | (childMeasuredState & MEASURED_STATE_MASK);
    }
```

返回值是一个复合整数，实际的大小在 MEASURED_SIZE_MASK 位中，可以看到 32 位，24 位用来存储测量大小，剩下的 8 位是测量状态的；

```java
    /**
     * Bits of {@link #getMeasuredWidthAndState()} and
     * {@link #getMeasuredWidthAndState()} that provide the actual measured size.
     */
    public static final int MEASURED_SIZE_MASK = 0x00ffffff; // 低 24 位；

    /**
     * Bits of {@link #getMeasuredWidthAndState()} and
     * {@link #getMeasuredWidthAndState()} that provide the additional state bits.
     */
    public static final int MEASURED_STATE_MASK = 0xff000000; // 高 8 位；
```

那么这个状态是什么呢，如下：

```java
    /**
     * Bit of {@link #getMeasuredWidthAndState()} and
     * {@link #getMeasuredWidthAndState()} that indicates the measured size
     * is smaller that the space the view would like to have.
     */
    public static final int MEASURED_STATE_TOO_SMALL = 0x01000000; // 表示测量值小于 view 想显示的大小；
```

如果如果测量约束指定的 size 小于 view 想要的 size ，那么会设置 MEASURED_STATE_TOO_SMALL 标志位



# 3 DecorView

## 3.1 onMeasure - 核心2

参数是 parent 指定的测量规格，里面包含了测量模式和 parent 的剩余大小，这里根据 parent 的测量规格，计算自身的测量模式；

```java
    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        // 获得屏幕的信息;
        final DisplayMetrics metrics = getContext().getResources().getDisplayMetrics();
        final boolean isPortrait = // 判断下横屏竖屏;
                getResources().getConfiguration().orientation == ORIENTATION_PORTRAIT;

        //【1】获得宽高的测量模式;
        final int widthMode = getMode(widthMeasureSpec);
        final int heightMode = getMode(heightMeasureSpec);

        boolean fixedWidth = false;
        mApplyFloatingHorizontalInsets = false;
        //【2】如果宽的测量模式是 AT_MOST，表示不能超过 parent 指定的值;
        if (widthMode == AT_MOST) {
            //【2.1】这会根据横竖屏，判断宽度的最大值，调整宽度;
            final TypedValue tvw = isPortrait ? mWindow.mFixedWidthMinor : mWindow.mFixedWidthMajor;
            if (tvw != null && tvw.type != TypedValue.TYPE_NULL) {
                final int w;
                if (tvw.type == TypedValue.TYPE_DIMENSION) { // 具体像素值;
                    w = (int) tvw.getDimension(metrics);
                } else if (tvw.type == TypedValue.TYPE_FRACTION) { // 百分比因子;
                    w = (int) tvw.getFraction(metrics.widthPixels, metrics.widthPixels);
                } else {
                    w = 0;
                }
                if (DEBUG_MEASURE) Log.d(mLogTag, "Fixed width: " + w);
                //【2.2】获取测量约束的指定的宽度，然后计算新的测量模式;
                final int widthSize = MeasureSpec.getSize(widthMeasureSpec);
                if (w > 0) {
                    widthMeasureSpec = MeasureSpec.makeMeasureSpec(
                            Math.min(w, widthSize), EXACTLY); // 如果 w 大于 0，说明已经可以计算出确切的大小;
                    // 标示已经修复了宽度;
                    fixedWidth = true;
                } else {
                    // 特殊情况，w 为 0，这里会用 DecorView 的宽度减去 mFloatingInsets 和 AT_MOST
                    // 计算新的测量模式，child 会以此为标准;
                    widthMeasureSpec = MeasureSpec.makeMeasureSpec(
                            widthSize - mFloatingInsets.left - mFloatingInsets.right,
                            AT_MOST);
                    mApplyFloatingHorizontalInsets = true;
                }
            }
        }

        mApplyFloatingVerticalInsets = false;
        //【3】如果高的测量模式是 AT_MOST，那么就不能超过 parent 指定的 size 值;
        if (heightMode == AT_MOST) {
            //【3.1】这会根据横竖屏，判断宽度的最大值，调整组件的宽度;
            final TypedValue tvh = isPortrait ? mWindow.mFixedHeightMajor
                    : mWindow.mFixedHeightMinor;
            if (tvh != null && tvh.type != TypedValue.TYPE_NULL) {
                final int h;
                if (tvh.type == TypedValue.TYPE_DIMENSION) {
                    h = (int) tvh.getDimension(metrics);
                } else if (tvh.type == TypedValue.TYPE_FRACTION) {
                    h = (int) tvh.getFraction(metrics.heightPixels, metrics.heightPixels);
                } else {
                    h = 0;
                }
                if (DEBUG_MEASURE) Log.d(mLogTag, "Fixed height: " + h);
                //【3.2】获取设置的高度，然后计算孩子的测量模式；
                final int heightSize = MeasureSpec.getSize(heightMeasureSpec);
                if (h > 0) {
                    heightMeasureSpec = MeasureSpec.makeMeasureSpec(
                            Math.min(h, heightSize), EXACTLY);
                } else if ((mWindow.getAttributes().flags & FLAG_LAYOUT_IN_SCREEN) == 0) {
                    heightMeasureSpec = MeasureSpec.makeMeasureSpec(
                            heightSize - mFloatingInsets.top - mFloatingInsets.bottom, AT_MOST);
                    mApplyFloatingVerticalInsets = true;
                }
            }
        }

        //【4】如果设置了 outset 区域，那么要对测量模式再次进行调整；
        getOutsets(mOutsets);
        if (mOutsets.top > 0 || mOutsets.bottom > 0) {
            int mode = MeasureSpec.getMode(heightMeasureSpec);
            if (mode != MeasureSpec.UNSPECIFIED) {
                int height = MeasureSpec.getSize(heightMeasureSpec);
                heightMeasureSpec = MeasureSpec.makeMeasureSpec(
                        height + mOutsets.top + mOutsets.bottom, mode);
            }
        }
        if (mOutsets.left > 0 || mOutsets.right > 0) {
            int mode = MeasureSpec.getMode(widthMeasureSpec);
            if (mode != MeasureSpec.UNSPECIFIED) {
                int width = MeasureSpec.getSize(widthMeasureSpec);
                widthMeasureSpec = MeasureSpec.makeMeasureSpec(
                        width + mOutsets.left + mOutsets.right, mode);
            }
        }

        //【-->4.1】进入父类的 onMeasure 方法，传入新的测量模式；
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);

        int width = getMeasuredWidth();
        boolean measure = false;
        
        // 这里会获取测量的宽度，然后测试模式使用 EXACTLY 定义新的测量标准，进行第二次调整；
        widthMeasureSpec = MeasureSpec.makeMeasureSpec(width, EXACTLY);

        if (!fixedWidth && widthMode == AT_MOST) {
            final TypedValue tv = isPortrait ? mWindow.mMinWidthMinor : mWindow.mMinWidthMajor;
            if (tv.type != TypedValue.TYPE_NULL) {
                final int min;
                if (tv.type == TypedValue.TYPE_DIMENSION) {
                    min = (int)tv.getDimension(metrics);
                } else if (tv.type == TypedValue.TYPE_FRACTION) {
                    min = (int)tv.getFraction(mAvailableWidth, mAvailableWidth);
                } else {
                    min = 0;
                }
                if (DEBUG_MEASURE) Log.d(mLogTag, "Adjust for min width: " + min + ", value::"
                        + tv.coerceToString() + ", mAvailableWidth=" + mAvailableWidth);

                if (width < min) {
                    widthMeasureSpec = MeasureSpec.makeMeasureSpec(min, EXACTLY);
                    measure = true;
                }
            }
        }

        //【-->4.1】进入父类的 onMeasure 方法；
        if (measure) {
            super.onMeasure(widthMeasureSpec, heightMeasureSpec);
        }
    }
```

我们可以看到，上面的方法对 measure 传入的测量规格做了调整；

代码逻辑很简单，就不多说了；

### 3.1.1 getMeasuredWidth

```java
    /**
     * Like {@link #getMeasuredWidthAndState()}, but only returns the
     * raw width component (that is the result is masked by
     * {@link #MEASURED_SIZE_MASK}).
     *
     * @return The raw measured width of this view.
     */
    public final int getMeasuredWidth() {
        return mMeasuredWidth & MEASURED_SIZE_MASK;
    }
```

### 3.1.2 getMode

获取测量模式：

```java
    @MeasureSpecMode
    public static int getMode(int measureSpec) {
        return (measureSpec & MODE_MASK);
    }
```



# 4 FrameLayout

## 4.1 onMeasure - 核心3

DecorView.onMeasure 进入 FrameLayout.onMeasure 方法：

```java
    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        //【-->4.1.1】获取所有的 child view;
        int count = getChildCount();
        
        //【1】EXACTLY 表示父视图已经确定了孩子的确切尺寸。不管孩子想要多大，都会给孩子这些值；
        // 比如具体数值，或者 match_parent；
        // 但是如果不是 EXACTLY 那就要每个 child 重新测量;
        final boolean measureMatchParentChildren =
                MeasureSpec.getMode(widthMeasureSpec) != MeasureSpec.EXACTLY ||
                MeasureSpec.getMode(heightMeasureSpec) != MeasureSpec.EXACTLY;
        mMatchParentChildren.clear();

        // 这里的 maxHeight 和 maxWidth 用来保存当前 view 的最大宽/高;
        int maxHeight = 0;
        int maxWidth = 0;
        int childState = 0;

        //【2】遍历每个 child view，可以看到，对于 ViewGroup，必须要先测量一下内部的 child view;
        for (int i = 0; i < count; i++) {
            final View child = getChildAt(i);
            if (mMeasureAllChildren || child.getVisibility() != GONE) {
                //【-->5.1】测量 child view;
                measureChildWithMargins(child, widthMeasureSpec, 0, heightMeasureSpec, 0);
                
                //【2.1】获取到 child view 的布局参数，同时确定所有 child 的最大高度和最大宽度
                // 需要考虑 child 占用的 margin 距离;
                final LayoutParams lp = (LayoutParams) child.getLayoutParams();
                maxWidth = Math.max(maxWidth,
                        child.getMeasuredWidth() + lp.leftMargin + lp.rightMargin);
                maxHeight = Math.max(maxHeight,
                        child.getMeasuredHeight() + lp.topMargin + lp.bottomMargin);
                
                //【-->2.4】获取每个 child view 的测量状态，同时【-->2.3】合并 child 的测量状态;
                childState = combineMeasuredStates(childState, child.getMeasuredState());
                
                //【2.2】如果需要对 child view 进行重新测量的话，同时 child view 的 lp 的 width/height
                // 至少有一个为 MATCH_PARENT，那么这个 view 就会被添加到 mMatchParentChildren 中;
                if (measureMatchParentChildren) {
                    if (lp.width == LayoutParams.MATCH_PARENT ||
                            lp.height == LayoutParams.MATCH_PARENT) {
                        mMatchParentChildren.add(child);
                    }
                }
            }
        }

        //【3】累加 padding 区域，这个是 FlameLayout 独有的；
        maxWidth += getPaddingLeftWithForeground() + getPaddingRightWithForeground();
        maxHeight += getPaddingTopWithForeground() + getPaddingBottomWithForeground();

        //【4】检查下最小的宽/高；
        maxHeight = Math.max(maxHeight, getSuggestedMinimumHeight());
        maxWidth = Math.max(maxWidth, getSuggestedMinimumWidth());

        //【5】检查下前景的宽/高；
        final Drawable drawable = getForeground();
        if (drawable != null) {
            maxHeight = Math.max(maxHeight, drawable.getMinimumHeight());
            maxWidth = Math.max(maxWidth, drawable.getMinimumWidth());
        }

        //【-->2.2.1】设置 DecorView/FlameLayout 自身的测量宽高值，可以看到 viewGroup 的宽高值依赖 child view 的 max 宽高；
        //【-->2.5】这里会根据 child 的测量状态，调整 FlameLayout 的测量状态；
        // 这里的 maxWidth 和 maxHeight 表示了 child view 中最大的宽度和高度，那么对于 view group，
        // 需要基于这个值来设置自身的宽高；
        setMeasuredDimension(resolveSizeAndState(maxWidth, widthMeasureSpec, childState),
                resolveSizeAndState(maxHeight, heightMeasureSpec,
                        childState << MEASURED_HEIGHT_STATE_SHIFT));

        // 下面是正式处理 child view；
        count = mMatchParentChildren.size();
        if (count > 1) {
            // 【3】count 大于 0，说明部分 child 也需要测量调整！
            for (int i = 0; i < count; i++) {
                final View child = mMatchParentChildren.get(i);
                final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();
                //【3.1】重新计算 child view 的宽/高测量约束；
                final int childWidthMeasureSpec;
                if (lp.width == LayoutParams.MATCH_PARENT) {
                    // 如果 child view 的布局参数是 match parent 的话，那么 child 的测量模式为 EXACTLY；
                    // 同时会计算新的测量宽度 width（在父 view 的距离基础上 - 父 view 的 padding - child 的 margin）
                    final int width = Math.max(0, getMeasuredWidth()
                            - getPaddingLeftWithForeground() - getPaddingRightWithForeground()
                            - lp.leftMargin - lp.rightMargin);
                    // 计算新的测量规格；
                    childWidthMeasureSpec = MeasureSpec.makeMeasureSpec(
                            width, MeasureSpec.EXACTLY);
                } else {
                    //【-->5.2】如果 child view 的布局参数不是 match parent 的话，这里会调用
                    // 新的方法计算测量规格；
                    childWidthMeasureSpec = getChildMeasureSpec(widthMeasureSpec,
                            getPaddingLeftWithForeground() + getPaddingRightWithForeground() +
                            lp.leftMargin + lp.rightMargin,
                            lp.width);
                }

                final int childHeightMeasureSpec;
                if (lp.height == LayoutParams.MATCH_PARENT) {
                    final int height = Math.max(0, getMeasuredHeight()
                            - getPaddingTopWithForeground() - getPaddingBottomWithForeground()
                            - lp.topMargin - lp.bottomMargin);
                    childHeightMeasureSpec = MeasureSpec.makeMeasureSpec(
                            height, MeasureSpec.EXACTLY);
                } else {
                    childHeightMeasureSpec = getChildMeasureSpec(heightMeasureSpec,
                            getPaddingTopWithForeground() + getPaddingBottomWithForeground() +
                            lp.topMargin + lp.bottomMargin,
                            lp.height);
                }

                //【-->2.1】child view 处理，如果是 ViewGroup 的话，流程和 DecorView 类似；
                child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
            }
        }
    }
```

我们看到了，对于 ViewGroup u 

### 4.1.1 getChildCount

返回内部的 child view 的个数：

```java
    public int getChildCount() {
        return mChildrenCount;
    }
```



# 5 ViewGroup

## 5.1 measureChildWithMargins - 核心4

测量 child view 自身，但是要考虑到 child 的 Margin 距离和 parent 的 padding 距离；

- 参数 View child：要测量的 child view；
- 参数 int parentWidthMeasureSpec：parent 的 width 测量约束；
- 参数 int widthUsed：parent view 已经被使用的距离（这个距离可能是被其他 view 使用了）

```java
    protected void measureChildWithMargins(View child,
            int parentWidthMeasureSpec, int widthUsed,
            int parentHeightMeasureSpec, int heightUsed) {
        //【1】获取到 child 布局参数 LayoutParams；
        final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();
        
        //【-->5.1.1】根据 parent 的测量规格和 child 的布局参数，计算 child view 的测量规格！！
        final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
                mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                        + widthUsed, lp.width);
        final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
                mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin
                        + heightUsed, lp.height);
        
        //【-->2.1】每个 view 进行自身的测量的递归；
        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    }
```

ViewGroup 继承了 View，他没有 measure 方法呀，调用的依然是 View 的 measure 方法；

### 5.1.1 getChildMeasureSpec - 核心5

getChildMeasureSpec 将 parent view 的 MeasureSpec 信息与 child 的 LayoutParams 结合起来，为一个 child 视图的高度或宽度，确定正确的 MeasureSpec。

- 参数 int spec：parent view 的测量规格，parent 指定，child 需要根据他来计算自身实际的测量规格；

- 参数 int padding：parent view 里已经被占用的空间；

- 参数 int childDimension：child view 的  LayoutParams 指定的显示的大小；

```java
    public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
        //【1】根据测量约束返回测量模式和测量距离；
        int specMode = MeasureSpec.getMode(spec);
        int specSize = MeasureSpec.getSize(spec);
        //【2】由于 specSize 是 parent 指定的测量距离，换要减去 padding 的距离，才是 child 的测量距离；
        int size = Math.max(0, specSize - padding);

        int resultSize = 0;
        int resultMode = 0;

        //【3】处理测量模式；
        switch (specMode) {
        //【3.1】如果 parent 的模式是 EXACTLY，表示 parent 知道了自己确切的大小；
        case MeasureSpec.EXACTLY:
            if (childDimension >= 0) {
                //【3.1.1】如果 child 布局参数指定了具体大小，那么 child 的测量规格定下来了，
                // 测量大小就是 child 布局参数指定的，测量模式为 EXACTLY；
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                //【3.1.2】如果 child 布局参数指定了 MATCH_PARENT，意味着 child 要和 parent 一样，而 parent 已经确定自己的大小了，
                // 那么 child 的测量大小就是 parent 的大小，测量模式就是 EXACTLY；
                resultSize = size;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                //【3.1.3】如果 child 布局参数指定了 WRAP_CONTENT，意味着 child 自适应内容，但是不能超过 parent
                // 那么 child 可以自适应，但大小不能超过 parent size，而测试模式为 AT_MOST；
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;

        //【3.2】如果 parent 的模式是 AT_MOST，也就是说 parent 还未确定自己的尺寸，但是大小不能超过 size
        case MeasureSpec.AT_MOST:
            if (childDimension >= 0) {
                //【3.2.1】如果 child 布局参数指定了大小，那么 child 的测量规格可以定下来了，虽然 parent 不定，但是
                // 依然可以满足 child 的要求，测试大小为 child 布局参数指定的，测试模式为 EXACTLY；
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                //【3.2.2】如果 child 没有指定具体的大小，而是 MATCH_PARENT，但是由于 parent 的模式是 AT_MOST，无法确定自己的大小
                // 所以，child 和 parent 一样，测量大小是 parent 的上限 size，测试模式为 AT_MOST；
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                //【3.2.3】如果 child 布局参数指定了 WRAP_CONTENT，意味着 child 自适应内容，但是不能超过 parent
                // 那么 child 可以自适应，但大小不能超过 parent size，而测试模式同样为 AT_MOST；
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;

        //【3.3】如果是 UNSPECIFIED，表示 parent 无法去确定自己的尺寸大小；
        case MeasureSpec.UNSPECIFIED:
            if (childDimension >= 0) {
                //【3.3.1】如果 child 指定了大小，那么实际的测量大小就是 child 想要的，模式为 EXACTLY；
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                //【3.3.2】如果 child 没有指定具体的大小，而是 MATCH_PARENT，这里会计算出它实际的大小；
                // 模式为 UNSPECIFIED
                resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                resultMode = MeasureSpec.UNSPECIFIED;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                //【3.3.3】如果 child 没有指定具体的大小，而是 WRAP_CONTENT，
                // 那么 child 想要自适应，自己调整大小，大小不能超过 size，模式为 AT_MOST；
                resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                resultMode = MeasureSpec.UNSPECIFIED;
            }
            break;
        }
        //【4】生成 child 的测量约束；
        return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
    }

```

这个方法实际上就是 ViewGroup 的核心之一，他会根据 parent 传入的测量模式，计算每个  child 的测量模式；

# 6 总结



## 6.1 测量的条件

每次都会 measure 嘛？未必，情况下面：

```java
    // 如果视图已经被测量获得了正确的尺寸，那么这里会判断下测量模式是否是 EXACTLY 如果是的话，那么就可能不会重新测量；
    // 在 API 23 及以下版本中，需要这样的额外遍历才能使 LinearLayout 重新分配权重。
    final boolean specChanged = widthMeasureSpec != mOldWidthMeasureSpec
        || heightMeasureSpec != mOldHeightMeasureSpec; // 测量标准是否变化
    final boolean isSpecExactly = MeasureSpec.getMode(widthMeasureSpec) == MeasureSpec.EXACTLY
        && MeasureSpec.getMode(heightMeasureSpec) == MeasureSpec.EXACTLY; // 是否是精确布局
    final boolean matchesSpecSize = getMeasuredWidth() == MeasureSpec.getSize(widthMeasureSpec)
        && getMeasuredHeight() == MeasureSpec.getSize(heightMeasureSpec); // 尺寸是否变化
    final boolean needsLayout = specChanged // 判断是否需要布局；
        && (sAlwaysRemeasureExactly || !isSpecExactly || !matchesSpecSize);

    if (forceLayout || needsLayout) {
```

可以看到，其实需要一些条件的；

## 6.2 View 和 ViewGroup 的测量不同点

View：

```java
measure() -> onMeasure()
```

这里传递进来的就是 view 的测量布局；

View Group：

```java
measure() -> measureChild()/
```

开始是 view group 的测量规格，后面在 measureChild 中，会根据 view group 的测量规格，计算 view 的测量规格，然后进入上面 view 的逻辑：

## 6.3 measure 测量

上面的流程里面 measureChildWithMargins 方法，考虑到了 child view 的 margin 距离，而 ViewGroup 内部还有一个方法：

```java
    protected void measureChild(View child, int parentWidthMeasureSpec,
            int parentHeightMeasureSpec) {
        final LayoutParams lp = child.getLayoutParams();
        //【1】可以看到，这里只考虑到了 parent view 的 padding 距离
        final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
                mPaddingLeft + mPaddingRight, lp.width);
        final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
                mPaddingTop + mPaddingBottom, lp.height);

        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    }
```

但是实际上，正常情况来说，margin 距离肯定是要有的，所以 measureChildWithMargins 会更有用些；



<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/23/View.measure-process.png" alt="View.measure-process" style="zoom: 33%;" />