# ViewDraw 第二篇 new ViewRootImpl 流程分析

title: ViewDraw 第二篇 new ViewRootImpl 流程分析
date: 2018/06/15 20:46:25 
catalog: true
categories: 

- View 视图
- View 的加载和绘制

tags: ViewDraw
copyright: true

------

本篇文章基于 Android N - 7.1.1 主要分析下 Activity 中 setContentView  的流程，但是此时 UI 还没有显示出来；

真正显示 ui 是在 **onResume** 方法调用后，这个流程会创建 **ViewRootImpl** 类，并设置 setView 操作；

# 1 回顾

我们来回顾下 activity 的 onResume 方法的执行。

## 1.1 ActivityThread

核心的入口是在 handleResumeActivity 方法中；

###  1.1.1 handleResumeActivity

我们来看看核心的代码：

```java
final void handleResumeActivity(IBinder token,
                                boolean clearHide, boolean isForward, boolean reallyResume, int seq, String reason) {
    ActivityClientRecord r = mActivities.get(token);
    ... ... ...

        //【1】拉起 activity 的 onResume 方法；
        r = performResumeActivity(token, clearHide, reason);

    if (r != null) {
        final Activity a = r.activity;

        ... ... ...
 
        if (r.window == null && !a.mFinished && willBeVisible) {
            //【1】获得 activity 的 PhoneWindow 对象，setContentView 时创建；
            r.window = r.activity.getWindow();
            //【2】获得 PhoneWindow 内部的 DecorView 对象，并设置其为不可见；
            View decor = r.window.getDecorView();
            decor.setVisibility(View.INVISIBLE);
            //【3】获得 WindowManagerImpl 对象；
            ViewManager wm = a.getWindowManager();
            WindowManager.LayoutParams l = r.window.getAttributes();
            a.mDecor = decor;
            l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
            l.softInputMode |= forwardBit;
            if (r.mPreserveWindow) { // 这是复用之前的 window，我们不关注；
                a.mWindowAdded = true;
                r.mPreserveWindow = false;
                // 通常，ViewRoot 在 addView-> ViewRootImpl.setView 中使用 Activity 设置回调。 
                // 如果我们改为重用装饰视图，则必须通知视图根回调可能已更改。
                ViewRootImpl impl = decor.getViewRootImpl();
                if (impl != null) {
                    impl.notifyChildRebuilt();
                }
            }
            //【4】如果 activity 还没有被加入到 wms，那么这里会 add window！
            if (a.mVisibleFromClient && !a.mWindowAdded) {
                a.mWindowAdded = true;
                //【-->2.2】添加 DecorView；
                wm.addView(decor, l);
            }

            // If the window has already been added, but during resume
            // we started another activity, then don't yet make the
            // window visible.
        } else if (!willBeVisible) {
            if (localLOGV) Slog.v(
                TAG, "Launch " + r + " mStartedActivity set");
            r.hideForNow = true;
        }

        // Get rid of anything left hanging around.
        cleanUpPendingRemoveWindows(r, false /* force */);

        // The window is now visible if it has been added, we are not
        // simply finishing, and we are not starting another activity.
        if (!r.activity.mFinished && willBeVisible
            && r.activity.mDecor != null && !r.hideForNow) {
            if (r.newConfig != null) {
                performConfigurationChangedForActivity(r, r.newConfig, REPORT_TO_ACTIVITY);
                if (DEBUG_CONFIGURATION) Slog.v(TAG, "Resuming activity "
                                                + r.activityInfo.name + " with newConfig " + r.activity.mCurrentConfig);
                r.newConfig = null;
            }
            if (localLOGV) Slog.v(TAG, "Resuming " + r + " with isForward="
                                  + isForward);
            WindowManager.LayoutParams l = r.window.getAttributes();
            if ((l.softInputMode
                 & WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION)
                != forwardBit) {
                l.softInputMode = (l.softInputMode
                                   & (~WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION))
                    | forwardBit;
                if (r.activity.mVisibleFromClient) {
                    ViewManager wm = a.getWindowManager();
                    View decor = r.window.getDecorView();
                    wm.updateViewLayout(decor, l);
                }
            }
            r.activity.mVisibleFromServer = true;
            mNumVisibleActivities++;
            if (r.activity.mVisibleFromClient) {
                r.activity.makeVisible();
            }
        }

        ... ... ...

    } else {
        try {
            ActivityManagerNative.getDefault()
                .finishActivity(token, Activity.RESULT_CANCELED, null,
                                Activity.DONT_FINISH_TASK_WITH_ACTIVITY);
        } catch (RemoteException ex) {
            throw ex.rethrowFromSystemServer();
        }
    }
}
```

这里，我们省略了一些不重要的代码，关注核心！

- 核心逻辑：

```java
            //【1】获得 activity 的 PhoneWindow 对象，setContentView 时创建；
            r.window = r.activity.getWindow();
            //【2】获得 PhoneWindow 内部的 DecorView 对象，并设置其为不可见；
            View decor = r.window.getDecorView();
            decor.setVisibility(View.INVISIBLE);
            //【3】获得 WindowManagerImpl 对象；
            ViewManager wm = a.getWindowManager();
            WindowManager.LayoutParams l = r.window.getAttributes();
            a.mDecor = decor;
            l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
            l.softInputMode |= forwardBit;
            
            ... ... ...      

            //【-->3.2】如果 activity 还没有被加入到 wms，那么这里会 add window！
            if (a.mVisibleFromClient && !a.mWindowAdded) {
                a.mWindowAdded = true;
                wm.addView(decor, l);
            }
```

可以看到，核心的逻辑在 wm.addView(decor, l) 这个地方！

- decor：我们的 DecorView;
- l：PhoneWindow 中的布局参数；

# 2 WindowManagerImpl

这里我们进入了 WindowManagerImpl，这个类是在 activity.attach 方法中创建的。

## 2.1 new WindowManagerImpl

我们来看下 WindowManagerImpl 的创建；

```java
public final class WindowManagerImpl implements WindowManager {
    //【-->3.1】内部有一个 wmGlobal 单例对象，impl 都是通过这个 global 来操作 view 的；
    private final WindowManagerGlobal mGlobal = WindowManagerGlobal.getInstance();
    private final Context mContext;
    //【1】这个是我们的 PhoneWindow 实例；
    private final Window mParentWindow;

    private IBinder mDefaultToken;
    
    public WindowManagerImpl(Context context) {
        this(context, null);
    }

    private WindowManagerImpl(Context context, Window parentWindow) {
        mContext = context;
        mParentWindow = parentWindow;
    }
```

WindowManager 又继承了 ViewManager：


```java
public interface WindowManager extends ViewManager {
    ... ... ...// 先省略下；
}
```

ViewManager 是一个接口，提供了两个函数接口：

```java
public interface ViewManager {
    public void addView(View view, ViewGroup.LayoutParams params);
    public void updateViewLayout(View view, ViewGroup.LayoutParams params);
}
  
```

根据注释：

- addView 的操作主要是：添加一个 view 到一个 window，同时分配布局属性给 view；
- 会抛出如下异常：
	- android.view.WindowManager.BadTokenException：在没有移除已经存在的 view 的情况下，添加新的 view，也就是说一个 window 对应一个 View！
	- android.view.WindowManager.InvalidDisplayException：当我们讲一个 window 添加到 display，但是其不存在或者无效的时候；

可以看到，根据继承和实现的关系，我们知道：

- WindowManagerImpl： 是最终的实现类，和 window、view 相关的操作都在其内部实现；
- WindowManager：提供了和 window 相关的操作，已经一些和 window 相关的常量，共有变量，异常定义等等；
- ViewManager：提供了和 view 添加和更新相关的操作；

各司其职，各司其职！



## 2.2 addView

我们来看看 

```java
    @Override
    public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
        //【-->2.2.1】申请默认的令牌 token；
        applyDefaultToken(params);
        //【-->3.2】添加 view；
        mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
    }
```



### 2.2.1 applyDefaultToken

```java
    private void applyDefaultToken(@NonNull ViewGroup.LayoutParams params) {
        //【1】如果 mParentWindow 为 null 的话，那么这里会分配默认的 token；
        if (mDefaultToken != null && mParentWindow == null) {
            //【1.1】判断参数类型；
            if (!(params instanceof WindowManager.LayoutParams)) {
                throw new IllegalArgumentException("Params must be WindowManager.LayoutParams");
            }

            //【1.2】分配默认的 token；
            final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams) params;
            if (wparams.token == null) {
                wparams.token = mDefaultToken;
            }
        }
    }
```

如果 mParentWindow 为 null 的话，那么这里会分配默认的 token；

# 3 WindowManagerGlobal

WindowManagerImpl 将操作交给 WindowManagerGlobal 完成，这里使用了桥接模式；

##  3.1 getInstance

他是一个单例模式：

```java
    private static WindowManagerGlobal sDefaultWindowManager;

    public static WindowManagerGlobal getInstance() {
        synchronized (WindowManagerGlobal.class) {
            if (sDefaultWindowManager == null) {
                sDefaultWindowManager = new WindowManagerGlobal();
            }
            return sDefaultWindowManager;
        }
    }
```

这个不多说了；



## 3.2 addView

添加 view：

```java
    public void addView(View view, ViewGroup.LayoutParams params,
            Display display, Window parentWindow) {
        //【1】判空处理；
        if (view == null) {
            throw new IllegalArgumentException("view must not be null");
        }
        if (display == null) {
            throw new IllegalArgumentException("display must not be null");
        }
        if (!(params instanceof WindowManager.LayoutParams)) {
            throw new IllegalArgumentException("Params must be WindowManager.LayoutParams");
        }
        //【2】获得 window 的布局参数；
        final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams) params;
      
        //【3】针对 parentWindow 进行处理，我们知道对于 activity，这里的 parentWindow 实际上就是 PhoneWindow；
        // 所以是不为 null 的；
        if (parentWindow != null) {
            //【-->4.1】针对于 window 的类型，调整下布局参数
            parentWindow.adjustLayoutParamsForSubWindow(wparams);
        } else {
            // 调整硬件加速的；
            final Context context = view.getContext();
            if (context != null
                    && (context.getApplicationInfo().flags
                            & ApplicationInfo.FLAG_HARDWARE_ACCELERATED) != 0) {
                wparams.flags |= WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED;
            }
        }
        //【4】这里就是我们说的核心：ViewRootImpl
        ViewRootImpl root;
        View panelParentView = null;

        synchronized (mLock) {
            //【5】监听系统属性的变化；
            if (mSystemPropertyUpdater == null) {
                mSystemPropertyUpdater = new Runnable() {
                    @Override public void run() {
                        synchronized (mLock) {
                            for (int i = mRoots.size() - 1; i >= 0; --i) {
                                //【5.1】当系统属性变化后，会触发所有 ViewRootImpl 的 loadSystemProperties 方法；
                                mRoots.get(i).loadSystemProperties();
                            }
                        }
                    }
                };
                SystemProperties.addChangeCallback(mSystemPropertyUpdater);
            }
            //【6】判断该 view 是否已经被添加了，实际上就是去 mViews 中找；
            // 如果能找到，并且他不在 mDyingViews 中，那么会抛出异常！
            int index = findViewLocked(view, false);
            if (index >= 0) {
                if (mDyingViews.contains(view)) {
                    //【6.1】如果其在 mDyingViews 中，那么不等待 MSG_DIE 消息出发，而是立即销毁！
                    mRoots.get(index).doDie();
                } else {
                    throw new IllegalStateException("View " + view
                            + " has already been added to the window manager.");
                }
                // The previous removeView() had not completed executing. Now it has.
            }

            //【7】如果该 window 是一个 sub window，那么我们会找到一个 token 和其一样的 view，作为 prarent；
            if (wparams.type >= WindowManager.LayoutParams.FIRST_SUB_WINDOW &&
                    wparams.type <= WindowManager.LayoutParams.LAST_SUB_WINDOW) {
                final int count = mViews.size();
                for (int i = 0; i < count; i++) {
                    if (mRoots.get(i).mWindow.asBinder() == wparams.token) {
                        panelParentView = mViews.get(i);
                    }
                }
            }

            //【-->5.1】创建 ViewRootImpl 实例，并设置 view 的布局参数；
            root = new ViewRootImpl(view.getContext(), display);
            view.setLayoutParams(wparams);

            //【1】将 View，新创建的 ViewRootImpl 和 LayoutParams 加入到内部的 list 中；
            // 注意，这里是有序的排列；
            mViews.add(view);
            mRoots.add(root);
            mParams.add(wparams);
        }

        try {
            //【-->5.2】添加 view；
            root.setView(view, wparams, panelParentView);
        } catch (RuntimeException e) {
            // BadTokenException or InvalidDisplayException, clean up.
            synchronized (mLock) {
                final int index = findViewLocked(view, false);
                if (index >= 0) {
                    removeViewLocked(index, true);
                }
            }
            throw e;
        }
    }

```

WindowManagerGlobal 内部有如下的 list，作用如下：

```java
// 保存所有已经被添加的 View！
private final ArrayList<View> mViews = new ArrayList<View>();

// 保存所有的 ViewRootImpl！
private final ArrayList<ViewRootImpl> mRoots = new ArrayList<ViewRootImpl>();

// 保存所有的布局属性：WindowManager.LayoutParams；
private final ArrayList<WindowManager.LayoutParams> mParams =
  new ArrayList<WindowManager.LayoutParams>();

// 保存所有的被删除销毁的 View，是那些已经调用 removeView 方法但是删除操作还未完成的 Window 对象；
private final ArraySet<View> mDyingViews = new ArraySet<View>();
```

不多说了；



## 3.3 getWindowSession

创建一个 window session：

```java
    public static IWindowSession getWindowSession() {
        synchronized (WindowManagerGlobal.class) {
            if (sWindowSession == null) {
                try {
                    //【1】获得 imm 对象，持有 ims 的代理对象；
                    InputMethodManager imm = InputMethodManager.getInstance();
                    //【-->3.3.1】获得 wms 的代理对象
                    IWindowManager windowManager = getWindowManagerService();
                    //【2】创建一个 window session；
                    sWindowSession = windowManager.openSession(
                            new IWindowSessionCallback.Stub() {
                                @Override
                                public void onAnimatorScaleChanged(float scale) {
                                    ValueAnimator.setDurationScale(scale);
                                }
                            },
                            imm.getClient(), imm.getInputContext());
                } catch (RemoteException e) {
                    throw e.rethrowFromSystemServer();
                }
            }
            return sWindowSession;
        }
    }
```

WindowManagerGlobal 内部还有如下的属性：

```java
    private static IWindowManager sWindowManagerService; // wms 代理；
    private static IWindowSession sWindowSession; // session 代理； 
```

可以看到，这里的 sWindowSession 是一个 IWindowSession 类型的实例；



### 3.3.1 getWindowManagerService

获得 wms 的代理对象：

```java
    public static IWindowManager getWindowManagerService() {
        synchronized (WindowManagerGlobal.class) {
            if (sWindowManagerService == null) {
                //【1】获取 wms proxy 对象；
                sWindowManagerService = IWindowManager.Stub.asInterface(
                        ServiceManager.getService("window"));
                try {
                    sWindowManagerService = getWindowManagerService();
                    ValueAnimator.setDurationScale(sWindowManagerService.getCurrentAnimatorScale());
                } catch (RemoteException e) {
                    throw e.rethrowFromSystemServer();
                }
            }
            return sWindowManagerService;
        }
    }
```

 

# 4 Window（PhoneWindow）

其实这里的 parentWindow 是 PhoneWindow：

## 4.1 adjustLayoutParamsForSubWindow

该方法的作用是针对于 sub window 调整下布局参数！

```java
    // --> Window.java
		void adjustLayoutParamsForSubWindow(WindowManager.LayoutParams wp) {
        CharSequence curTitle = wp.getTitle();
        //【1】针对于 window type 来做不同的处理！
        if (wp.type >= WindowManager.LayoutParams.FIRST_SUB_WINDOW &&
                wp.type <= WindowManager.LayoutParams.LAST_SUB_WINDOW) {
            //【1.1】针对于 sub window 的处理，可以看到，token 来自于 DecorView！
            if (wp.token == null) {
                View decor = peekDecorView();
                if (decor != null) {
                    wp.token = decor.getWindowToken();
                }
            }
            // 设置 title 和包名；
            if (curTitle == null || curTitle.length() == 0) {
                final StringBuilder title = new StringBuilder(32);
                if (wp.type == WindowManager.LayoutParams.TYPE_APPLICATION_MEDIA) {
                    title.append("Media");
                } else if (wp.type == WindowManager.LayoutParams.TYPE_APPLICATION_MEDIA_OVERLAY) {
                    title.append("MediaOvr");
                } else if (wp.type == WindowManager.LayoutParams.TYPE_APPLICATION_PANEL) {
                    title.append("Panel");
                } else if (wp.type == WindowManager.LayoutParams.TYPE_APPLICATION_SUB_PANEL) {
                    title.append("SubPanel");
                } else if (wp.type == WindowManager.LayoutParams.TYPE_APPLICATION_ABOVE_SUB_PANEL) {
                    title.append("AboveSubPanel");
                } else if (wp.type == WindowManager.LayoutParams.TYPE_APPLICATION_ATTACHED_DIALOG) {
                    title.append("AtchDlg");
                } else {
                    title.append(wp.type);
                }
                if (mAppName != null) {
                    title.append(":").append(mAppName);
                }
                wp.setTitle(title);
            }
        } else if (wp.type >= WindowManager.LayoutParams.FIRST_SYSTEM_WINDOW &&
                wp.type <= WindowManager.LayoutParams.LAST_SYSTEM_WINDOW) {
            //【1.2】针对于 system window 的处理，这里不会针对于 system window 设置令牌 token；
            // 因为他的生命周期应该是独立的，当 app 的进程被杀掉/或者死亡了，system window 不会受影响；
            if (curTitle == null || curTitle.length() == 0) {
                final StringBuilder title = new StringBuilder(32);
                title.append("Sys").append(wp.type);
                if (mAppName != null) {
                    title.append(":").append(mAppName);
                }
                wp.setTitle(title);
            }
        } else {
            // 对于 activity 则会进入这里；
            //【1.3】设置令牌 token，这个令牌保存在 PhoneWindow 中；
            if (wp.token == null) {
                wp.token = mContainer == null ? mAppToken : mContainer.mAppToken;
            }
            // 设置 title；
            if ((curTitle == null || curTitle.length() == 0)
                    && mAppName != null) {
                wp.setTitle(mAppName);
            }
        }
        //【2】设置了 window 的所属包名；
        if (wp.packageName == null) {
            wp.packageName = mContext.getPackageName();
        }
        if (mHardwareAccelerated ||
                (mWindowAttributes.flags & FLAG_HARDWARE_ACCELERATED) != 0) {
            wp.flags |= FLAG_HARDWARE_ACCELERATED;
        }
    }
```

这个方法主要是针对于不同的 window 做不同的处理；

- system window 是不会有 token 的；
- sub window 的 token 来自 DecorView(View).**mAttachInfo.mWindowToken**，通过 View.getWindowToken() 获得；

我们再来看看几个方法：

- peekDecorView 返回的是 DecorView，这个可以从前面文章看出来：

```java
// ----> Window.java/PhoneWindow.java
@Override
public final View peekDecorView() {
   // 返回的是 DecorView 实例；
   return mDecor;
}
```

- decor.getWindowToken()，实际上是父类 View 的方法：

```java
public IBinder getWindowToken() {
   return mAttachInfo != null ? mAttachInfo.mWindowToken : null;
}
```

关于这个 mAttachInfo 我们后面再看；

# 5 ViewRootImpl



```java
@SuppressWarnings({"EmptyCatchBlock", "PointlessBooleanExpression"})
public final class ViewRootImpl implements ViewParent,
        View.AttachInfo.Callbacks, ThreadedRenderer.HardwareDrawCallbacks {
```

可以看到 ViewRootImpl 实现了 ViewParent 接口，View.AttachInfo.Callbacks 接口，ThreadedRenderer.HardwareDrawCallbacks 接口；

## 5.1 new ViewRootImpl

ViewRootImpl 内部的成员变量很多，这里我们来看看：

```java
    public ViewRootImpl(Context context, Display display) {
        mContext = context; // 上下文环境；
        //【-->3.3】获得一个窗口事务对象；
        mWindowSession = WindowManagerGlobal.getWindowSession();
        mDisplay = display; // 目标显示器 display；
        mBasePackageName = context.getBasePackageName();
        mThread = Thread.currentThread(); // 当前线程，其实就是主线程；
        mLocation = new WindowLeaked(null); // 是 AndroidRuntimeException 的子类
        mLocation.fillInStackTrace();

        mWidth = -1; // 宽和高；
        mHeight = -1;

        mDirty = new Rect();
        mTempRect = new Rect();
        mVisRect = new Rect();
        mWinFrame = new Rect();
      
        //【-->6.1】创建可一个 W 对象，其继承了 IWindow.Stub，用于跨进程通信；
        mWindow = new W(this);
      
        mTargetSdkVersion = context.getApplicationInfo().targetSdkVersion; // 目标 sdk
        mViewVisibility = View.GONE; // 可见行，默认为 GONE；

        mTransparentRegion = new Region(); // 和透明区域相关；
        mPreviousTransparentRegion = new Region();

        mFirst = true; // 为 true 表示这个 view 是第一次 add；
        mAdded = false;

        //【-->7.1】创建了一个 View.AttachInfo 实例；
        mAttachInfo = new View.AttachInfo(mWindowSession, mWindow, display, this, mHandler, this);
      
        // 和辅助服务功能相关，这里先不关注；
        mAccessibilityManager = AccessibilityManager.getInstance(context);
        mAccessibilityInteractionConnectionManager =
            new AccessibilityInteractionConnectionManager();
        mAccessibilityManager.addAccessibilityStateChangeListener(
                mAccessibilityInteractionConnectionManager);
        mHighContrastTextManager = new HighContrastTextManager();
        mAccessibilityManager.addHighTextContrastStateChangeListener(
                mHighContrastTextManager);
 
        mViewConfiguration = ViewConfiguration.get(context); // view 的配置属性；
        mDensity = context.getResources().getDisplayMetrics().densityDpi; // 屏幕像素
        mNoncompatDensity = context.getResources().getDisplayMetrics().noncompatDensityDpi;
        mFallbackEventHandler = new PhoneFallbackEventHandler(context);
        mChoreographer = Choreographer.getInstance(); //【-->10.5】编舞者实例；
        mDisplayManager = (DisplayManager)context.getSystemService(Context.DISPLAY_SERVICE); // 显示器管理服务；
      
        //【-->5.5】加载系统属性；
        loadSystemProperties();
    }
```

我们看到了变量还是很多的，这里一起列举下核心的几个：

- W mWindow：
- mWindowSession：

下面的还回遇到一些重要的成员变量，我们也在这里一起来看看：

- **SurfaceHolder.Callback2 mSurfaceHolderCallback**：
- **BaseSurfaceHolder mSurfaceHolder**：




## 5.2 setView - 核心代码

添加 View，参数 panelParentView 表示要 attach 的 view：

```java
    public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
        synchronized (this) {
            if (mView == null) {
                //【*1】将要 add 的 view 保存到 ViewRootImpl.mView 中；
                mView = view;
                //【*2】获得最新显示器的状态；
                mAttachInfo.mDisplayState = mDisplay.getState();
                // 同时注册显示器监听器监听变化；
                //【-->9.1】将 mDisplayListener 注册到了 mDisplayManager 中，监听显示变化；
                //【-->9.2】同时将 ViewRootHandler 注册到了 mDisplayManager 中；
                mDisplayManager.registerDisplayListener(mDisplayListener, mHandler);

                mViewLayoutDirectionInitial = mView.getRawLayoutDirection();
                mFallbackEventHandler.setView(view);
              
                //【*3】拷贝一份布局参数到 mWindowAttributes 中，更新下包名，然后设置 attrs;
                mWindowAttributes.copyFrom(attrs);
                if (mWindowAttributes.packageName == null) {
                    mWindowAttributes.packageName = mBasePackageName;
                }
                attrs = mWindowAttributes;
                setTag();

                if (DEBUG_KEEP_SCREEN_ON && (mClientWindowLayoutFlags
                        & WindowManager.LayoutParams.FLAG_KEEP_SCREEN_ON) != 0
                        && (attrs.flags&WindowManager.LayoutParams.FLAG_KEEP_SCREEN_ON) == 0) {
                    Slog.d(mTag, "setView: FLAG_KEEP_SCREEN_ON changed from true to false!");
                }

                //【4】将布局属性中的 flags 保存到 ViewRootImpl.mClientWindowLayoutFlags 中； 
                mClientWindowLayoutFlags = attrs.flags;
                
                //【5】和辅助服务功能相关，这里先不关注；
                setAccessibilityFocus(null, null);

                //【*6】DecorView 实现了 RootViewSurfaceTaker 接口，所以会进入这里；
                if (view instanceof RootViewSurfaceTaker) {
                    //【6.1】如果是 activity 这种情况，willYouTakeTheSurface() 返回值为 null；
                    // mSurfaceHolderCallback 是 SurfaceHolder.Callback2 实例，用于感知 Surface 的创建、销毁或者改变；
                    mSurfaceHolderCallback =
                            ((RootViewSurfaceTaker)view).willYouTakeTheSurface();
                    if (mSurfaceHolderCallback != null) {
                        //【6.2】如果 mSurfaceHolderCallback 不会 null，这里会再创建一个 SurfaceHolder 实例：mSurfaceHolder
                        // 提供访问和控制 Surface 相关的方法；
                        mSurfaceHolder = new TakenSurfaceHolder();
                        mSurfaceHolder.setFormat(PixelFormat.UNKNOWN);
                    }
                }

                // Compute surface insets required to draw at specified Z value.
                // TODO: Use real shadow insets for a constant max Z.
                if (!attrs.hasManualSurfaceInsets) {
                    attrs.setSurfaceInsets(view, false /*manual*/, true /*preservePrevious*/);
                }

                CompatibilityInfo compatibilityInfo =
                        mDisplay.getDisplayAdjustments().getCompatibilityInfo();
                mTranslator = compatibilityInfo.getTranslator();

                //【7】如果持有了 surface，那就不开启硬件加速，显然，这里是开启的！
                if (mSurfaceHolder == null) {
                    enableHardwareAcceleration(attrs);
                }

                boolean restore = false;
                if (mTranslator != null) {
                    mSurface.setCompatibilityTranslator(mTranslator);
                    restore = true;
                    attrs.backup();
                    mTranslator.translateWindowLayout(attrs);
                }
                if (DEBUG_LAYOUT) Log.d(mTag, "WindowLayout in setView:" + attrs);

                if (!compatibilityInfo.supportsScreen()) {
                    attrs.privateFlags |= WindowManager.LayoutParams.PRIVATE_FLAG_COMPATIBLE_WINDOW;
                    mLastInCompatMode = true;
                }

                mSoftInputMode = attrs.softInputMode;
      
                //【*8】标注 window 的属性发生了变化；
                mWindowAttributesChanged = true;
                mWindowAttributesChangesFlag = WindowManager.LayoutParams.EVERYTHING_CHANGED;
              
                //【*9】将要添加的 view 设置到 mAttachInfo.mRootView 上；
                mAttachInfo.mRootView = view;
                mAttachInfo.mScalingRequired = mTranslator != null;
                mAttachInfo.mApplicationScale =
                        mTranslator == null ? 1.0f : mTranslator.applicationScale;
                if (panelParentView != null) {
                    mAttachInfo.mPanelParentWindowToken
                            = panelParentView.getApplicationWindowToken();
                }
              
                //【*10】将 mAdded 设置为 ture，表示已经添加了；
                mAdded = true;
                int res; /* = WindowManagerImpl.ADD_OKAY; */

                //【-->5.3】请求布局，这里在将 window 添加到 wms 之前，做了一个布局操作，
                // 在和 wms 建立关系之前，完成布局，这样不影响事件的接收。
                requestLayout();
              
                //【*11】LayoutParams.INPUT_FEATURE_NO_INPUT_CHANNEL 这个 feature 表示不接受输入事件，
                // 一般没人无聊这样干，所以这里会创建一个 InputChannel，用于接收输入事件；
                if ((mWindowAttributes.inputFeatures
                        & WindowManager.LayoutParams.INPUT_FEATURE_NO_INPUT_CHANNEL) == 0) {
                    //【-->9.5】InputChannel，用于接收输入事件；
                    mInputChannel = new InputChannel();
                }
                mForceDecorViewVisibility = (mWindowAttributes.privateFlags
                        & PRIVATE_FLAG_FORCE_DECOR_VIEW_VISIBILITY) != 0;
                try {
                    //【*12】将布局参数中的 window type 类型，保存到 mOrigWindowType 中；
                    mOrigWindowType = mWindowAttributes.type;
                    //【*13】表示重新计算属性；
                    mAttachInfo.mRecomputeGlobalAttributes = true;
                    //【-->5.4】重新搜集 view 的布局属性；
                    collectViewAttributes();
       
                    //【*12】将 window 加入到 wms 中，建立当前视图窗口与系统 WindowManagerService 服务的关联；
                    // 会在 WindowManagerService 服务中为该 APP 窗口生成两个 InputQueue, 
                    // 其中一个会调用 InputQueue.transferTo() 返回到当前 APP 进程窗口;
                    // 另外一个保留在 WindowManagerService 为当前 APP 窗口创建的 WindowState 中;
                    res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                            getHostVisibility(), mDisplay.getDisplayId(),
                            mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                            mAttachInfo.mOutsets, mInputChannel);

                } catch (RemoteException e) {
                    mAdded = false;
                    mView = null;
                    mAttachInfo.mRootView = null;
                    mInputChannel = null;
                    mFallbackEventHandler.setView(null);
                    unscheduleTraversals();
                    setAccessibilityFocus(null, null);
                    throw new RuntimeException("Adding window failed", e);
                } finally {
                    if (restore) {
                        attrs.restore();
                    }
                }

                if (mTranslator != null) {
                    mTranslator.translateRectInScreenToAppWindow(mAttachInfo.mContentInsets);
                }
                mPendingOverscanInsets.set(0, 0, 0, 0);
                mPendingContentInsets.set(mAttachInfo.mContentInsets);
                mPendingStableInsets.set(mAttachInfo.mStableInsets);
                mPendingVisibleInsets.set(0, 0, 0, 0);
                mAttachInfo.mAlwaysConsumeNavBar =
                        (res & WindowManagerGlobal.ADD_FLAG_ALWAYS_CONSUME_NAV_BAR) != 0;
                mPendingAlwaysConsumeNavBar = mAttachInfo.mAlwaysConsumeNavBar;
                if (DEBUG_LAYOUT) Log.v(mTag, "Added window " + mWindow);
 
                //【13】处理添加显示的结果，如果不是 WindowManagerGlobal.ADD_OKAY，那么就是异常的；
                if (res < WindowManagerGlobal.ADD_OKAY) {
                    mAttachInfo.mRootView = null;
                    mAdded = false;
                    mFallbackEventHandler.setView(null);
                    unscheduleTraversals();
                    setAccessibilityFocus(null, null);
                    switch (res) {
                        case WindowManagerGlobal.ADD_BAD_APP_TOKEN:
                        case WindowManagerGlobal.ADD_BAD_SUBWINDOW_TOKEN:
                            throw new WindowManager.BadTokenException(
                                    "Unable to add window -- token " + attrs.token
                                    + " is not valid; is your activity running?");
                        case WindowManagerGlobal.ADD_NOT_APP_TOKEN:
                            throw new WindowManager.BadTokenException(
                                    "Unable to add window -- token " + attrs.token
                                    + " is not for an application");
                        case WindowManagerGlobal.ADD_APP_EXITING:
                            throw new WindowManager.BadTokenException(
                                    "Unable to add window -- app for token " + attrs.token
                                    + " is exiting");
                        case WindowManagerGlobal.ADD_DUPLICATE_ADD:
                            throw new WindowManager.BadTokenException(
                                    "Unable to add window -- window " + mWindow
                                    + " has already been added");
                        case WindowManagerGlobal.ADD_STARTING_NOT_NEEDED:
                            // Silently ignore -- we would have just removed it
                            // right away, anyway.
                            return;
                        case WindowManagerGlobal.ADD_MULTIPLE_SINGLETON:
                            throw new WindowManager.BadTokenException("Unable to add window "
                                    + mWindow + " -- another window of type "
                                    + mWindowAttributes.type + " already exists");
                        case WindowManagerGlobal.ADD_PERMISSION_DENIED:
                            throw new WindowManager.BadTokenException("Unable to add window "
                                    + mWindow + " -- permission denied for window type "
                                    + mWindowAttributes.type);
                        case WindowManagerGlobal.ADD_INVALID_DISPLAY:
                            throw new WindowManager.InvalidDisplayException("Unable to add window "
                                    + mWindow + " -- the specified display can not be found");
                        case WindowManagerGlobal.ADD_INVALID_TYPE:
                            throw new WindowManager.InvalidDisplayException("Unable to add window "
                                    + mWindow + " -- the specified window type "
                                    + mWindowAttributes.type + " is not valid");
                    }
                    throw new RuntimeException(
                            "Unable to add window -- unknown error code " + res);
                }
                
                //【】根据前面知道，DecorView 实现了 RootViewSurfaceTaker 接口，所以这里为 true；
                // 创建 InputQueue 的 create 和 destroy 的监听对象, 
                //【-->8.2】如果是 activity 这种情况，willYouTakeTheInputQueue() 返回值为 null；
                if (view instanceof RootViewSurfaceTaker) {
                    mInputQueueCallback =
                        ((RootViewSurfaceTaker)view).willYouTakeTheInputQueue();
                }
              
                //【14】从上面可以知道，正常情况，窗口是会接受输入事件的，所以 if 为 true；
                if (mInputChannel != null) {
                    if (mInputQueueCallback != null) { // 这里为 false，原因见上面；
                        mInputQueue = new InputQueue();
                        mInputQueueCallback.onInputQueueCreated(mInputQueue);
                    }
                    //【-->9.4】创建一个接受输入事件的处理对象，绑定当前窗口的 InputChannel；
                    mInputEventReceiver = new WindowInputEventReceiver(mInputChannel,
                            Looper.myLooper());
                }
                
                //【-->5.2.1】将 ViewRootImpl 设置为其 parent；
                view.assignParent(this);

                mAddedTouchMode = (res & WindowManagerGlobal.ADD_FLAG_IN_TOUCH_MODE) != 0;
                mAppVisible = (res & WindowManagerGlobal.ADD_FLAG_APP_VISIBLE) != 0;

                if (mAccessibilityManager.isEnabled()) {
                    mAccessibilityInteractionConnectionManager.ensureConnection();
                }

                if (view.getImportantForAccessibility() == View.IMPORTANT_FOR_ACCESSIBILITY_AUTO) {
                    view.setImportantForAccessibility(View.IMPORTANT_FOR_ACCESSIBILITY_YES);
                }

                //【15】这里设置当前各种不同类别输入事件到来时候按对应类型依次分别调用的处理对象
                CharSequence counterSuffix = attrs.getTitle();
                mSyntheticInputStage = new SyntheticInputStage();
                InputStage viewPostImeStage = new ViewPostImeInputStage(mSyntheticInputStage);
                InputStage nativePostImeStage = new NativePostImeInputStage(viewPostImeStage,
                        "aq:native-post-ime:" + counterSuffix);
                InputStage earlyPostImeStage = new EarlyPostImeInputStage(nativePostImeStage);
                InputStage imeStage = new ImeInputStage(earlyPostImeStage,
                        "aq:ime:" + counterSuffix);
                InputStage viewPreImeStage = new ViewPreImeInputStage(imeStage);
                InputStage nativePreImeStage = new NativePreImeInputStage(viewPreImeStage,
                        "aq:native-pre-ime:" + counterSuffix);

                mFirstInputStage = nativePreImeStage;
                mFirstPostImeInputStage = earlyPostImeStage;
                mPendingInputEventQueueLengthCounterName = "aq:pending:" + counterSuffix;
            }
        }
    }
```

我们看到，这个流程还涉及到了 ViewRootImpl 的一些其他核心属性，这里简单看看：

```JAVA
View mView // 就是要添加的 DecorView 实例；
WindowManager.LayoutParams mWindowAttributes // DecorView 的布局参数拷贝；

int mClientWindowLayoutFlags // 布局参数的 flags 属性；
int mOrigWindowType = -1; // 布局参数的 type 属性；

SurfaceHolder.Callback2 mSurfaceHolderCallback; // 用于感知 Surface 的创建、销毁或者改变；
BaseSurfaceHolder mSurfaceHolder; // 提供访问和控制 Surface 相关的方法;

boolean mWindowAttributesChanged = false; // 表示 window 的属性是否发生变化；
boolean mAdded; // 是否已经添加到 wms 中；

final View.AttachInfo mAttachInfo; // 当 view 绑定到 window 上时，用户保存一些核心数据；
InputChannel mInputChannel; // 输入渠道，用于接受 input 事件；
WindowInputEventReceiver mInputEventReceiver // 用于从 mInputChannel 中读取事件并分发；
```

同样的，还涉及到了 AttachInfo 的一些属性：

```

```



### 5.2.1 View.assignParent

给 View 分配 Parent！

```java
    void assignParent(ViewParent parent) {
        if (mParent == null) {
            // 这里的 mParent 是 ViewParent 的实例；
            mParent = parent;
        } else if (parent == null) {
            mParent = null;
        } else {
            throw new RuntimeException("view " + this + " being added, but"
                    + " it already has a parent");
        }
    }
```

我们知道 ViewRootImpl 是实现了 ViewParent 接口的：

```java
    // 表示这个 view 绑定的 parent，可以通过 getParent 获取；   
    protected ViewParent mParent; 
```



## 5.3 requestLayout - 核心代码

请求布局；

```java
    @Override
    public void requestLayout() {
        if (!mHandlingLayoutInLayoutRequest) {
            //【-->5.3.1】检测线程；
            checkThread();
            mLayoutRequested = true;
            //【-->5.6】这里就要进入核心的逻辑了；
            scheduleTraversals();
        }
    }
```

可以看到，这里最后调用了 scheduleTraversals 方法！



### 5.3.1 checkThread

在请求布局的时候，需要检查当前线程是否是 main thread：

```java
    void checkThread() {
        //【1】检测当前线程是否是主线程，不是的话，异常！
        if (mThread != Thread.currentThread()) {
            throw new CalledFromWrongThreadException(
                    "Only the original thread that created a view hierarchy can touch its views.");
        }
    }
```

只有主线程才能操作 ui！

## 5.4 collectViewAttributes

收集并更新 view 的属性；

```java
    private boolean collectViewAttributes() {
        //【1】这里我们知道，前面将其设置成了 true；
        if (mAttachInfo.mRecomputeGlobalAttributes) {
            //Log.i(mTag, "Computing view hierarchy attributes!");
            mAttachInfo.mRecomputeGlobalAttributes = false; // 然后将其再设置成 false；
            boolean oldScreenOn = mAttachInfo.mKeepScreenOn;
            //【2】下面会将布局参数 mWindowAttributes 和 mAttachInfo 中的属性进行比较；
            // 更新 mWindowAttributes 和 mAttachInfo；
            mAttachInfo.mKeepScreenOn = false;
            mAttachInfo.mSystemUiVisibility = 0;
            mAttachInfo.mHasSystemUiListeners = false;
            mView.dispatchCollectViewAttributes(mAttachInfo, 0);
            mAttachInfo.mSystemUiVisibility &= ~mAttachInfo.mDisabledSystemUiVisibility;
            WindowManager.LayoutParams params = mWindowAttributes;
            mAttachInfo.mSystemUiVisibility |= getImpliedSystemUiVisibility(params);
          
            if (mAttachInfo.mKeepScreenOn != oldScreenOn
                    || mAttachInfo.mSystemUiVisibility != params.subtreeSystemUiVisibility
                    || mAttachInfo.mHasSystemUiListeners != params.hasSystemUiListeners) {
                applyKeepScreenOnFlag(params);
                params.subtreeSystemUiVisibility = mAttachInfo.mSystemUiVisibility;
                params.hasSystemUiListeners = mAttachInfo.mHasSystemUiListeners;
                mView.dispatchWindowSystemUiVisiblityChanged(mAttachInfo.mSystemUiVisibility);
                return true;
            }
        }
        return false;
    }
```

下面就不再多说了；



## 5.5 loadSystemProperties

加载系统属性：

```JAVA
    public void loadSystemProperties() {
        //【-->9.1】这里的 Handler 就是 ViewRootHandler
        mHandler.post(new Runnable() {
            @Override
            public void run() {
                //【1】读取 viewroot.profile_rendering 属性，判断是否强制视图层次以 60 Hz 的频率呈现。
                mProfileRendering = SystemProperties.getBoolean(PROPERTY_PROFILE_RENDERING, false);
                profileRendering(mAttachInfo.mHasWindowFocus);

                //【2】读取和硬件渲染相关的属性；
                if (mAttachInfo.mHardwareRenderer != null) {
                    if (mAttachInfo.mHardwareRenderer.loadSystemProperties()) {
                        invalidate();
                    }
                }

                //【3】用于 debug 布局；
                boolean layout = SystemProperties.getBoolean(View.DEBUG_LAYOUT_PROPERTY, false);
                if (layout != mAttachInfo.mDebugLayout) {
                    mAttachInfo.mDebugLayout = layout;
                    //【-->9.1】发送 MSG_INVALIDATE_WORLD 给 ViewRootHandler 进行全局的重绘制；
                    if (!mHandler.hasMessages(MSG_INVALIDATE_WORLD)) {
                        mHandler.sendEmptyMessageDelayed(MSG_INVALIDATE_WORLD, 200);
                    }
                }
            }
        });
    }
```

这里不再细看。





## 5.6 scheduleTraversals - 核心

触发视图遍历：

```java
    void scheduleTraversals() {
        if (!mTraversalScheduled) {
            //【1】表示是否已经发起重绘，这是要设置为 true；
            mTraversalScheduled = true;
            //【2】在主线程的消息队列中放一个障栅；
            mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
            //【-->10.5】设置一个回调到编舞者 Choreographer 中，在下一次的绘制触发时，执行 mTraversalRunnable
            // mTraversalRunnable 是一个 runnbale 实例；
            mChoreographer.postCallback(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
            //【3】如果此时有未处理的 input 事件，
            if (!mUnbufferedInputDispatch) {
                //【-->5.8】处理 input 事件；
                scheduleConsumeBatchedInput();
            }
            notifyRendererOfFramePending();
            pokeDrawLockIfNeeded();
        }
    }
```

我们来看看 **Choreographer.postCallback** 的第一个参数：

- **Choreographer.CALLBACK_TRAVERSAL**：表示该回调是一个绘制回调；



其实在 Choreographer  一共有三种回调类型，分别是：

- **CALLBACK_INPUT**：事件回调；
- **CALLBACK_ANIMATION**：动画回调；
- **CALLBACK_TRAVERSAL**：绘制回调；



对于编舞者，这里就不详细分析了；



### 5.6.1 TraversalRunnable.run

TraversalRunnable 只是一个 runnable，其内部会调用另外一个方法：**doTraversal()**

```java
    final class TraversalRunnable implements Runnable {
        @Override
        public void run() {
            //【-->5.7】开始遍历；
            doTraversal();
        }
    }
    final TraversalRunnable mTraversalRunnable = new TraversalRunnable();
```


## 5.7 doTraversal

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
   
            performTraversals();

            if (mProfile) {
                Debug.stopMethodTracing();
                mProfile = false;
            }
        }
    }
```

我们看到，最后调用了 performTraversals 方法！

关于 performTraversals 下一篇文章来分析！


## 5.8 scheduleConsumeBatchedInput 

处理 input 事件：

```java
    void scheduleConsumeBatchedInput() {
        if (!mConsumeBatchedInputScheduled) {
            mConsumeBatchedInputScheduled = true;
            mChoreographer.postCallback(Choreographer.CALLBACK_INPUT,
                    mConsumedBatchedInputRunnable, null);
        }
    }
```

这个和之前的逻辑就很类似了！

### 5.8.1 ConsumeBatchedInputRunnable.run

和之前的逻辑是一样的，只不过是针对于 input 的：

```java
final ConsumeBatchedInputRunnable mConsumedBatchedInputRunnable =
    new ConsumeBatchedInputRunnable();

final class ConsumeBatchedInputRunnable implements Runnable {
        @Override
        public void run() {
            doConsumeBatchedInput(mChoreographer.getFrameTimeNanos());
        }
    }
```

当然对于 input 的处理，这里不是本文的重点！

# 6 ViewRootImpl.W

W 类（这名字，估计 Google 也想不到应该启什么名字了），使用于跨进程通信的，当 wms 需要通知应用客户端去做一些事的时候，那么就会通过 Binder 通信，调用 W 类的相关方法！

而每一个 W 又和一个 ViewRootImpl 相关联，所以 W 的会将操作交给 ViewRootImpl；

## 6.1 new W

我们来看看 W 的构造器：

```java
    static class W extends IWindow.Stub {
        //【-->5】可以看到，内部有一个 ViewRootImpl 弱引用：
        private final WeakReference<ViewRootImpl> mViewAncestor;
        //【1】持有一个窗口事务，来自 ViewRootImpl
        private final IWindowSession mWindowSession;

        W(ViewRootImpl viewAncestor) {
            mViewAncestor = new WeakReference<ViewRootImpl>(viewAncestor);
            mWindowSession = viewAncestor.mWindowSession;
        }
    }
```

# 7 AttachInfo

我们知道，如果一个 window 是 sub window 的话，他会被绑定（attach）到令牌 token 一样的 parent window 上，那么这个 AttachInfo 会用保存和 attach 相关的信息；

**AttachInfo 是 View 的内部类**！

## 7.1 new AttachInfo

- **参数 IWindowSession session**：窗口事务；
- **参数 IWindow window**：就是 ViewRootImpl 中的 W 实例：mWindow；
- **参数 Handler handler**：这个是 ViewRootImpl 内部的 mHandler 实例；

```java
AttachInfo(IWindowSession session, IWindow window, Display display,
           ViewRootImpl viewRootImpl, Handler handler, Callbacks effectPlayer,
           Context context) {
    mSession = session; //【1】窗口事务 session 实例；
    mWindow = window; //【2】W 对象；
    mWindowToken = window.asBinder(); //【3】将 W 实例转为 Binder 对象；
    mDisplay = display; //【4】目标显示器 Display 实例；
    mViewRootImpl = viewRootImpl; //【5】所属的 viewRootImpl 实例；
    mHandler = handler; //【-->9.1】viewRootImpl 内部的 ViewRootHandler 实例
    mRootCallbacks = effectPlayer; //【6】所属的 viewRootImpl 实例；
    //【-->9.3】ViewTreeObserver 用于监听 view tree 的全局变化，包括布局改变，绘制开始，触摸模式改变等等；
    mTreeObserver = new ViewTreeObserver(context);
}
```

关于 ViewTreeObserver 这里就先不关注了。

# 8 DecorView (RootViewSurfaceTaker)

## 8.1 willYouTakeTheSurface

是否让 View 持有监听 surface 变化的回调：

```java
    public android.view.SurfaceHolder.Callback2 willYouTakeTheSurface() {
        return mFeatureId < 0 ? mWindow.mTakeSurfaceCallback : null;
    }
```

这里的 mFeatureId 在 activity 的情况下是 -1，所以会返回 null；



## 8.2 willYouTakeTheInputQueue

是否让 View 持有 InputQueue：

```java
    public InputQueue.Callback willYouTakeTheInputQueue() {
        return mFeatureId < 0 ? mWindow.mTakeInputQueueCallback : null;
    }
```

和上面是一样的！

# 9 总结

我们来分析下整个流程：

```sequence
ActivityThread -> ActivityThread:  1.handleResumeActivity
ActivityThread -> WindowManagerImpl:  2.addView

WindowManagerImpl -> ViewRootImpl: 3.new ViewRootImpl
ViewRootImpl --> WindowManagerImpl: 4.return root:ViewRootImpl

WindowManagerImpl -> ViewRootImpl: 5.setView


ViewRootImpl -> ViewRootImpl: 6.requestLayout
```








