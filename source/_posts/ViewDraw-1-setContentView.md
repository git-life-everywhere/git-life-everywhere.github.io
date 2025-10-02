#  ViewDraw 第一篇 setContentView 流程分析

title: ViewDraw 第一篇 setContentView 流程分析
date: 2018/06/01 20:46:25 
catalog: true
categories: 
- View 视图
- View 的加载和绘制

tags: ViewDraw
copyright: true

------

本篇文章基于 Android N（7.1.1）主要分析下 View 的加载 setContentView 的流程，以对 View 架构有更好的理解。

# 1 概述

setContentView 这个方法用于设置视图布局文件，在 Activity 的 onCreate 方法中，我们会通过该方法传入一个资源 id：

```java
public class SplashActivity extends AppCompatActivity {
    private SplashHandler mSplashHandler;
    private static boolean sendMsg = true; // 避免重复进入handleMessage处理逻辑

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        //【-->2.1】核心方法，设置布局；
        setContentView(R.layout.splash_layout);
    }
  
}
```

下面我们来一起跟踪下 setContentView 的流程!

# 2 启动流程回顾

我们在之前分析 activity 启动的时候，应用进程的核心逻辑是在 ActivityThread 中，我们来去回顾下核心的代码：

## 2.1 ActivityThread

### 2.1.1 handleLaunchActivity



这个方法是在主线程的 H 中调用的，H 会收到来自 Stub 的  LAUNCH_ACTIVITY 的消息：



```java
switch (msg.what) {
  case LAUNCH_ACTIVITY: {
    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
    //【1】获取到传递的 ActivityClientRecord 实例；
    final ActivityClientRecord r = (ActivityClientRecord) msg.obj;

    r.packageInfo = getPackageInfoNoCheck(
      r.activityInfo.applicationInfo, r.compatInfo);
    //【2】调用 handleLaunchActivity 方法；
    handleLaunchActivity(r, null, "LAUNCH_ACTIVITY");
    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
  } break;
```

我们接着看：

```java
private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    ... ... ...

    //【1】初始化 WindowManagerGlobal 实例；
    WindowManagerGlobal.initialize();
    //【-->2.1.2】调用另外一个方法;
    Activity a = performLaunchActivity(r, customIntent);
    	
    ... ... ...
}
```

- 当 ActivityManagerService 接收到启动 Activity 的请求之后会通过 binder 跨进程通信，通知 activity 所在进程的 ApplicationThread 对象，然后执行：handleLauncherActivity方法；
- 会初始化一个 **WindowManagerGlobal 实例**；
- 调用 **performLaunchActivity 方法**；



###  2.1.2 performLaunchActivity

继续看：

```java
    private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        // System.out.println("##### [" + System.currentTimeMillis() + 
        // "] ActivityThread.performLaunchActivity(" + r + ")");
        ... ... ...

        Activity activity = null;
        try {
            //【1】通过反射来创建 Activity 实例；
            java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
            StrictMode.incrementExpectedActivityCount(activity.getClass());
            r.intent.setExtrasClassLoader(cl);
            r.intent.prepareToEnterProcess();
            if (r.state != null) {
                r.state.setClassLoader(cl);
            }
        } catch (Exception e) {
            if (!mInstrumentation.onException(activity, e)) {
                throw new RuntimeException(
                    "Unable to instantiate activity " + component
                    + ": " + e.toString(), e);
            }
        }

        try {
            Application app = r.packageInfo.makeApplication(false, mInstrumentation);

            ... ... ...

            if (activity != null) {
                Context appContext = createBaseContextForActivity(r, activity);
                CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
                Configuration config = new Configuration(mCompatConfiguration);
                if (r.overrideConfig != null) {
                    config.updateFrom(r.overrideConfig);
                }
                if (DEBUG_CONFIGURATION) Slog.v(TAG, "Launching activity "
                        + r.activityInfo.name + " with config " + config);
       
                //【1】尝试复用上一次的的 window 实例；
                Window window = null;
                if (r.mPendingRemoveWindow != null && r.mPreserveWindow) {
                    window = r.mPendingRemoveWindow;
                    r.mPendingRemoveWindow = null;
                    r.mPendingRemoveWindowManager = null;
                }
                //【-->2.2.1】执行 attach 绑定操作；
                activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window);

                if (customIntent != null) {
                    activity.mIntent = customIntent;
                }
                r.lastNonConfigurationInstances = null;
                activity.mStartedActivity = false;
                int theme = r.activityInfo.getThemeResource();
                if (theme != 0) {
                    activity.setTheme(theme);
                }
                //【-->3.1】这个方法会调用 activity 的 onCreate 方法；
                // 然后调用 setContextView 方法；
                activity.mCalled = false;
                if (r.isPersistable()) {
                    mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
                } else {
                    mInstrumentation.callActivityOnCreate(activity, r.state);
                }
                ... ... ...
            }
            r.paused = true;

            mActivities.put(r.token, r);

        } catch (SuperNotCalledException e) {
            throw e;

        } catch (Exception e) {
            if (!mInstrumentation.onException(activity, e)) {
                throw new RuntimeException(
                    "Unable to start activity " + component
                    + ": " + e.toString(), e);
            }
        }

        return activity;
    }
```

- 反射来创建 **Activity** 实例；
- 执行 **activity.attach** 方法；
- 调用 **onCreate** 方法，在 **onCreate** 方法中，会调用 **setContextView** 方法；



这里涉及到 ActivityClientRecord.token 这个是怎么来的呢，我们简单回顾下：

```java
// ActivityStackSupervisor --> realStartActivityLocked 方法中：
app.thread.scheduleLaunchActivity(new Intent(r.intent), r.appToken,
          System.identityHashCode(r), r.info, new Configuration(mService.mConfiguration),
          new Configuration(task.mOverrideConfig), r.compat, r.launchedFromPackage,
          task.voiceInteractor, app.repProcState, r.icicle, r.persistentState, results,
          newIntents, !andResume, mService.isNextTransitionForward(), profilerInfo);
```

可以看到传入的是 **ActivityRecord.appToken**，我们来看一个简单的调用栈：

```java
ActivityManagerService.startActivity(IApplicationThread, ....)   
  ->ActivityManagerService.startActivityAsUser(IApplicationThread, ...)    
      -->ActivityStack.startActivityMayWait(IApplicationThread, ...)
          -->ActivityStack.startActivityLocked(IApplicationThread, ...)
```

这里会创建 ActivityRecord 实例，内部会创建一个 token 实例：

```Java
    ActivityRecord(ActivityManagerService _service, ProcessRecord _caller, ...) {
        service = _service;
        appToken = new Token(this, service);
        ... ... ... ...
    }
```

这个 token 是 ActivityRecord 的内部类，继承了 **IApplicationToken.Stub** 类：

```java
    static class Token extends IApplicationToken.Stub {
        private final WeakReference<ActivityRecord> weakActivity;
        private final ActivityManagerService mService;

        Token(ActivityRecord activity, ActivityManagerService service) {
            weakActivity = new WeakReference<>(activity);
            mService = service;
        }
        ... ... ...
    }
```

这个 Token 会传递进入应用进程，保存到应用进程，然后在 addView 的时候再传递进系统进程的。



## 2.2 Activity

### 2.2.1 attach - 核心方法

关键核心代码在 **attach**：

```java
final void attach(Context context, ActivityThread aThread,
        Instrumentation instr, IBinder token, int ident,
        Application application, Intent intent, ActivityInfo info,
        CharSequence title, Activity parent, String id,
        NonConfigurationInstances lastNonConfigurationInstances,
        Configuration config, String referrer, IVoiceInteractor voiceInteractor) {
    attachBaseContext(context);

    mFragments.attachHost(null /*parent*/);

    //【-->2.3.1】创建了 PhoneWindow 实例;
    mWindow = new PhoneWindow(this, window);
    //【-->5.3.x】并设置 activity 为 callback；
    mWindow.setWindowControllerCallback(this);
    mWindow.setCallback(this);
    mWindow.setOnWindowDismissedCallback(this);
    if (info.softInputMode != WindowManager.LayoutParams.SOFT_INPUT_STATE_UNSPECIFIED) {
        mWindow.setSoftInputMode(info.softInputMode);
    }
    // 紧接着，设置了 uiOptions（包含动画等属性）;
    if (info.uiOptions != 0) {
        mWindow.setUiOptions(info.uiOptions);
    }
    mUiThread = Thread.currentThread();

    mMainThread = aThread;
    mInstrumentation = instr;
    mToken = token; // 令牌 token；
    mIdent = ident;

    ... ... ...

    //【-->2.4.1】设置 wms 的代理对象到 PhoneWindow 中；
    mWindow.setWindowManager(
            (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
            mToken, mComponent.flattenToString(),
            (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
    if (mParent != null) {
        mWindow.setContainer(mParent.getWindow());
    }
 
    //【1】获得 wms 代理的应用；
    mWindowManager = mWindow.getWindowManager();
    mCurrentConfig = config;
}
```

整体流程比较清晰，这里的 mToken 是一个 Binder 引用：

```java
    private IBinder mToken;
```

继续！

## 2.3 PhoneWindow

PhoneWindow 继承了 Window 抽象类，他也是 window 类的唯一实现！

PhoneWindow 将 DecorView 作为 root view，这里的 DecorView 实际上是一个 FrameLayout！

每一个 activity 都会有一个 PhoneWindow，他是 Activity 和 View 交互的中间层！

```java
public class PhoneWindow extends Window implements MenuBuilder.Callback {
	... ... ...
}
```

### 2.3.1 new PhoneWindow

创建一个 PhoneWindow 实例，同时创建 LayoutInflater 实例，其用于加载布局文件！

参数：Context context 是所属的 activity；

```java
// 这个构造器用于创建 activity 对应的 window；
public PhoneWindow(Context context, Window preservedWindow) {
    this(context);
    //【1】mUseDecorContext 默认是 false，只有创建 activity 对应的 window，才会使用 decor context；
    // 其他的 window 都是直接使用传入的 context；
    mUseDecorContext = true;
    if (preservedWindow != null) {
        mDecor = (DecorView) preservedWindow.getDecorView();
        mElevation = preservedWindow.getElevation();
        mLoadElevation = false;
        mForceDecorInstall = true;
        // If we're preserving window, carry over the app token from the preserved
        // window, as we'll be skipping the addView in handleResumeActivity(), and
        // the token will not be updated as for a new window.
        getAttributes().token = preservedWindow.getAttributes().token;
    }
 
    
    // 和画中画模式相关，即使系统不支持画中画模式，用于也可以通过开发者选项强行开启；
    boolean forceResizable = Settings.Global.getInt(context.getContentResolver(),
                                                    DEVELOPMENT_FORCE_RESIZABLE_ACTIVITIES, 0) != 0;
    mSupportsPictureInPicture = forceResizable || context.getPackageManager().hasSystemFeature(
        PackageManager.FEATURE_PICTURE_IN_PICTURE);
}
```

其实 PhoneWindow 里面有很多的成员，这里我们先不关注，因为太多了。。。

## 2.4 Window

Window 是一个抽象类，它提供了一系列窗口的接口，比如：**setContentView**、**findViewById **等等，而其唯一实现类则是 **PhoneWindow**。

也就是说，其内部的一些抽象方法的最终实现，均是由 PhoneWindow。

### 2.4.1 setWindowManager

```java
public void setWindowManager(WindowManager wm, IBinder appToken, String appName) {
    //【1】调用了另外一个构造器；
    setWindowManager(wm, appToken, appName, false);
}


public void setWindowManager(WindowManager wm, IBinder appToken, String appName,
                             boolean hardwareAccelerated) {
    //【2】记录下了令牌 token；
    mAppToken = appToken;
    mAppName = appName;
    mHardwareAccelerated = hardwareAccelerated
        || SystemProperties.getBoolean(PROPERTY_HARDWARE_UI, false);
    if (wm == null) {
        //【2】如果 wm 为 null，那就获取 wms 的代理对象；
        wm = (WindowManager)mContext.getSystemService(Context.WINDOW_SERVICE);
    }
    //【-->2.5】创建 wmImpl 实例；
    mWindowManager = ((WindowManagerImpl)wm).createLocalWindowManager(this);
}
```



## 2.5 WindowManagerImpl

WindowManagerImpl 实现了 WindowManager 接口，是对 wms 的代理的一个包装类：

```java
    private WindowManagerImpl(Context context, Window parentWindow) {
        mContext = context;
        mParentWindow = parentWindow;
    }
    
    // 参数是 PhoneWindow；
    public WindowManagerImpl createLocalWindowManager(Window parentWindow) {
        return new WindowManagerImpl(mContext, parentWindow);
    }
```

该类用于和 wms 进行通信！！

可以看到，createLocalWindowManager 方法传入的 parentWindow 就是我们的 PhoneWindow；

## 2.6 类图总结

我们来通过类图看看整个流程中，这几个伙伴之间的关系，也便于对整体有个把握，方便后续的分析：

![](http://assets.processon.com/chart_image/5d9e8a78e4b0c55535ead4ac.png)



# 3 Activity

下面，我们就来从 activity 入手：

## 3.1 setContentView

setContextView 有三个重载方法，但是本质上是一样的：

```java
public void setContentView(@LayoutRes int layoutResID) {
    //【-->3.2】返回 PhoneWindow，
    //【-->4.1】调用其 setContextView 方法；
  	getWindow().setContentView(layoutResID);
    //【3.4】设置 actionBar
  	initWindowDecorActionBar();
}

// 下面两个方法如出一辙，不再关注；
public void setContentView(View view) {
  	getWindow().setContentView(view);
 	 	initWindowDecorActionBar();
}

public void setContentView(View view, ViewGroup.LayoutParams params) {
  	getWindow().setContentView(view, params);
  	initWindowDecorActionBar();
}
```

## 3.2 getWindow

这里的 getWindow 返回的是什么呢，就是 PhoneWindow

```java
    public Window getWindow() {
        //【-->4.1】就是前面创建的 PhoneWindow
        return mWindow;
    }
```



## 3.3 onWindowAttributesChanged

通知 activity 窗口的属性发生变化了！

```java
    public void onWindowAttributesChanged(WindowManager.LayoutParams params) {
        // Update window manager if: we have a view, that view is
        // attached to its parent (which will be a RootView), and
        // this activity is not embedded.
        if (mParent == null) {
            View decor = mDecor;
            if (decor != null && decor.getParent() != null) {
                //【1】通知 wms 更新 view 布局；
                getWindowManager().updateViewLayout(decor, params);
            }
        }
    }
```

这里我们先不继续分析；

## 3.4 initWindowDecorActionBar

设置 actionbar：

```java
    private void initWindowDecorActionBar() {
        //【1】获得 PhoneWindow 实例；
        Window window = getWindow();

        window.getDecorView();

        if (isChild() || !window.hasFeature(Window.FEATURE_ACTION_BAR) || mActionBar != null) {
            return;
        }

        //【2】设置 actionBar；
        mActionBar = new WindowDecorActionBar(this);
        mActionBar.setDefaultDisplayHomeAsUpEnabled(mEnableDefaultActionBarUp);

        mWindow.setDefaultIcon(mActivityInfo.getIconResource());
        mWindow.setDefaultLogo(mActivityInfo.getLogoResource());
    }
```

这里有一个 isChild() ：判断一个 activity 是否嵌入到另一个 activity 中，如果是一个 chile activity，mParent 不为 null，一般哦们不用这种特性：

```java
    Activity mParent;   

    /** Is this activity embedded inside of another activity? */
    public final boolean isChild() {
        return mParent != null;
    }
```

就不多说了！

# 4 PhoneWindow

下面的逻辑就进入了 PhoneWindow 中了；

## 4.1 setContentView

我们在 activity 中一般是通过传入 layoutResID 来设置布局的：

```java
    @Override
    public void setContentView(int layoutResID) {
        // Note: FEATURE_CONTENT_TRANSITIONS may be set in the process of installing the window
        // decor, when theme attributes and the like are crystalized. Do not check the feature
        // before this happens.
        //【1】第一次进入的 mContentParent 肯定是 null 的，所以进入了 installDecor 方法中；
        if (mContentParent == null) {
            //【-->4.2】创建 DecorView！
            installDecor();
        } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            mContentParent.removeAllViews();
        }
        //【2】这里和场景动画 Scene 有关系，这里先不关注；
        if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                    getContext());
            transitionTo(newScene);
        } else {
            //【*important 3】加载我们设置的布局到 mContentParent；
            // 这里又会调用 ViewGroup.addView 方法，涛声依旧了；
            // requestLayout(); --> invalidate(true); --> addViewInner(... ...)
            mLayoutInflater.inflate(layoutResID, mContentParent);
        }
        mContentParent.requestApplyInsets();
        final Callback cb = getCallback();
        if (cb != null && !isDestroyed()) {
            cb.onContentChanged();
        }
        mContentParentExplicitlySet = true;
    }
```

可以看到，在将 setContentView 设置的 布局 加载到 mContentParent 之前，还是设置了很多的工作的：installDecor！

这里的对于 mLayoutInflater.inflate 的加载机制，就先不分析了，后面单独开一帖；

当然，**PhoneWindow** 也有其他两个 **serContentView** 方法：

- **setContentView(View view)**

```java
@Override
public void setContentView(View view) {
    //【1】调用另外一个 set 方法，第二个参数表示布局属性；
    setContentView(view, new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));
}
```
- **setContentView(View view, ViewGroup.LayoutParams params)**

```java
    @Override
    public void setContentView(View view, ViewGroup.LayoutParams params) {
        // Note: ... ...
        if (mContentParent == null) {
            //【-->4.2】创建 DecorView！！
            installDecor();
        } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            mContentParent.removeAllViews();
        }
        //【2】这里和场景动画 Scene 有关系，这里先不关注；
        if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            view.setLayoutParams(params);
            final Scene newScene = new Scene(mContentParent, view);
            transitionTo(newScene);
        } else {
            //【3】加载布局文件到 mContentParent；
            mContentParent.addView(view, params);
        }
        mContentParent.requestApplyInsets();
        final Callback cb = getCallback();
        if (cb != null && !isDestroyed()) {
            cb.onContentChanged();
        }
        mContentParentExplicitlySet = true;
    }
```

这里的 mContentParent 是一个 ViewGroup 实例，很简单不多说；

## 4.2 installDecor

创建 DecorView：

```java
    private void installDecor() {
        mForceDecorInstall = false;
        //【1】第一次的话，mDecor 肯定是 null 的，那么这里就会创建 DecorView 实例；
        if (mDecor == null) {
            //【-->4.2.1】通过 generateDecor 方法创建 DecorView，参数为 -1；
            mDecor = generateDecor(-1);
            mDecor.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
            mDecor.setIsRootNamespace(true);
            if (!mInvalidatePanelMenuPosted && mInvalidatePanelMenuFeatures != 0) {
                mDecor.postOnAnimation(mInvalidatePanelMenuRunnable);
            }
        } else {
            mDecor.setWindow(this);
        }
        //【2】同样的第一次，mContentParent 为 null；
        if (mContentParent == null) {
            //【-->4.2.2】加载 content root 布局，并返回 view 实例；
            mContentParent = generateLayout(mDecor);

            // Set up decor part of UI to ignore fitsSystemWindows if appropriate.
            mDecor.makeOptionalFitsSystemWindows();

            //【2.1】 R.id.decor_content_parent 对应的 view 是 root view；
            // 这里尝试将获取的 view 转为一个 DecorContentParent 实例，实际上 R.id.decor_content_parent 对应的
            // view 是 ActionBarOverlayLayout，其实现了 DecorContentParent 接口；
            // 根据 root 布局文件，有这个 view 的只有：screen_toolbar.xml / screen_action_bar.xml
            final DecorContentParent decorContentParent = (DecorContentParent) mDecor.findViewById(
                    R.id.decor_content_parent);
            //【2.2】针对不同的情况，做不同的处理；
            if (decorContentParent != null) {
                //【2.2.1】将其保存到 PhoneWindow.DecorContentParent
                mDecorContentParent = decorContentParent;
                //【2.2.2】给 PhoneWindow.DecorContentParent 设置 window callback(就是 acitivty)
                mDecorContentParent.setWindowCallback(getCallback());
                if (mDecorContentParent.getTitle() == null) {
                    // 设置了 title
                    mDecorContentParent.setWindowTitle(mTitle);
                }

                final int localFeatures = getLocalFeatures();
                for (int i = 0; i < FEATURE_MAX; i++) {
                    if ((localFeatures & (1 << i)) != 0) {
                        mDecorContentParent.initFeature(i);
                    }
                }

                mDecorContentParent.setUiOptions(mUiOptions);
                // 设置 icon；
                if ((mResourcesSetFlags & FLAG_RESOURCE_SET_ICON) != 0 ||
                        (mIconRes != 0 && !mDecorContentParent.hasIcon())) {
                    mDecorContentParent.setIcon(mIconRes);
                } else if ((mResourcesSetFlags & FLAG_RESOURCE_SET_ICON) == 0 &&
                        mIconRes == 0 && !mDecorContentParent.hasIcon()) {
                    mDecorContentParent.setIcon(
                            getContext().getPackageManager().getDefaultActivityIcon());
                    mResourcesSetFlags |= FLAG_RESOURCE_SET_ICON_FALLBACK;
                }
                if ((mResourcesSetFlags & FLAG_RESOURCE_SET_LOGO) != 0 ||
                        (mLogoRes != 0 && !mDecorContentParent.hasLogo())) {
                    mDecorContentParent.setLogo(mLogoRes);
                }

                // ... 这里我们不过多关注；
                PanelFeatureState st = getPanelState(FEATURE_OPTIONS_PANEL, false);
                if (!isDestroyed() && (st == null || st.menu == null) && !mIsStartingWindow) {
                    invalidatePanelMenu(FEATURE_ACTION_BAR);
                }
            } else {
                //【2.2.3】如果没有 R.id.decor_content_parent，那么肯定是其他的 root view
                // 这里尝试获取 R.id.title/R.id.title_container 然后设置可见性，标题等；
                mTitleView = (TextView) findViewById(R.id.title);
                if (mTitleView != null) {
                    if ((getLocalFeatures() & (1 << FEATURE_NO_TITLE)) != 0) {
                        final View titleContainer = findViewById(R.id.title_container);
                        if (titleContainer != null) {
                            titleContainer.setVisibility(View.GONE);
                        } else {
                            mTitleView.setVisibility(View.GONE);
                        }
                        mContentParent.setForeground(null);
                    } else {
                        mTitleView.setText(mTitle);
                    }
                }
            }

            if (mDecor.getBackground() == null && mBackgroundFallbackResource != 0) {
                mDecor.setBackgroundFallback(mBackgroundFallbackResource);
            }

            // 下面这部分和场景动画有关系，不再我们这篇文章的范围之内，省略掉先；
            if (hasFeature(FEATURE_ACTIVITY_TRANSITIONS)) {
              ... ... ...
            }
        }
    }
```

这里的 mDecor 和 mContentParent 分别是如下的类型：

```java
    // 表示当前 window 的 top level 的视图 view！
    private DecorView mDecor;

    // 这个表示的是 @android:id/content 对应的 view，我们设置的 view 会被 add 到里面；
    ViewGroup mContentParent;
```



### 4.2.1 generateDecor

创建 DecorView 实例：

- 参数 int featureId：用于表示是是否一个 activity 类型的 window（-1 表示是 activity 类型的 window）

```java
    protected DecorView generateDecor(int featureId) {
        // System process doesn't have application context and in that case we need to directly use
        // the context we have. Otherwise we want the application context, so we don't cling to the
        // activity.
        //【1】创建上下文；
        Context context;
        //【2】mUseDecorContext 表示是否使用 decor context，默认是 false 的；
        // 但是如果我们启动的是 activity，那么 mUseDecorContext 为 true；
        if (mUseDecorContext) {
            Context applicationContext = getContext().getApplicationContext();
            if (applicationContext == null) {
                context = getContext();
            } else {
                context = new DecorContext(applicationContext, getContext().getResources());
                if (mTheme != -1) {
                    context.setTheme(mTheme);
                }
            }
        } else {
            context = getContext();
        }
        //【-->6.1】创建了 DecorView 实例；
        return new DecorView(context, featureId, this, getAttributes());
    }
```

对于创建 Context 这里，我们不过多关注！

### 4.2.2 generateLayout

加载布局：

```java
    protected ViewGroup generateLayout(DecorView decor) {
        //【1】这里获取窗口的样式，读取的是 com.android.internal.R.styleable.Window 属性；
        // 其实就是 activity 的 theme 配置
        TypedArray a = getWindowStyle();

        if (false) {
            System.out.println("From style:");
            String s = "Attrs:";
            for (int i = 0; i < R.styleable.Window.length; i++) {
                s = s + " " + Integer.toHexString(R.styleable.Window[i]) + "="
                        + a.getString(i);
            }
            System.out.println(s);
        }
        //【2】判断是否是悬浮的 window；
        mIsFloating = a.getBoolean(R.styleable.Window_windowIsFloating, false);
        //【2.1】获取要更新的 falags，其实就是 FLAG_LAYOUT_IN_SCREEN | FLAG_LAYOUT_INSET_DECOR；
        int flagsToUpdate = (FLAG_LAYOUT_IN_SCREEN | FLAG_LAYOUT_INSET_DECOR)
                & (~getForcedWindowFlags());

        //【3】针对悬浮属性做处理；
        if (mIsFloating) {
            //【-->5.4】如果是悬浮 window，那么肯定不是全屏的，这里会将默认的 MATCH_PARENT 
            // 改为 WRAP_CONTENT；
            setLayout(WRAP_CONTENT, WRAP_CONTENT);
            //【-->5.5】设置 flags，去掉 flagsToUpdate；
            setFlags(0, flagsToUpdate);
 
        } else {
            //【-->5.5】设置 flags，增加 FLAG_LAYOUT_IN_SCREEN | FLAG_LAYOUT_INSET_DECOR；
            setFlags(FLAG_LAYOUT_IN_SCREEN | FLAG_LAYOUT_INSET_DECOR, flagsToUpdate);
        }
        
        //【4】统一处理一些窗口特性；
        //【-->4.2.1.1】设置 feture！
        if (a.getBoolean(R.styleable.Window_windowNoTitle, false)) {
            //【4.1】设置了 notitle；
            requestFeature(FEATURE_NO_TITLE);
            
        } else if (a.getBoolean(R.styleable.Window_windowActionBar, false)) {
            //【4.2】设置 actionbar！
            requestFeature(FEATURE_ACTION_BAR);
        }

        if (a.getBoolean(R.styleable.Window_windowActionBarOverlay, false)) {
            //【4.3】设置覆盖模式，也就是说 actionbar 会覆盖在 window content 上；
            // 和 FEATURE_ACTION_BAR 一起使用；
            requestFeature(FEATURE_ACTION_BAR_OVERLAY);
        }

        if (a.getBoolean(R.styleable.Window_windowActionModeOverlay, false)) {
            //【4.4】设置 FEATURE_ACTION_MODE_OVERLAY；
            requestFeature(FEATURE_ACTION_MODE_OVERLAY);
        }

        if (a.getBoolean(R.styleable.Window_windowSwipeToDismiss, false)) {
            //【4.5】开启了滑动退出，那就设置 FEATURE_SWIPE_TO_DISMISS；
            requestFeature(FEATURE_SWIPE_TO_DISMISS);
        }
        
        //【5】处理一些 flags 设置；
        //【-->5.4】设置 flags；
        if (a.getBoolean(R.styleable.Window_windowFullscreen, false)) {
            //【5.1】隐藏状态栏全屏显示 Window，设置 FLAG_FULLSCREEN 标志；
            setFlags(FLAG_FULLSCREEN, FLAG_FULLSCREEN & (~getForcedWindowFlags()));
        }

        if (a.getBoolean(R.styleable.Window_windowTranslucentStatus,
                false)) {
            //【5.1】使状态栏透明同时会拉伸 window 到全屏的状态（保留 NavigationBar 高度），
            // 假如有 ActionBar，ActionBar 依旧会显示，设置 FLAG_FULLSCREEN 标志；
            setFlags(FLAG_TRANSLUCENT_STATUS, FLAG_TRANSLUCENT_STATUS
                    & (~getForcedWindowFlags()));
        }

        if (a.getBoolean(R.styleable.Window_windowTranslucentNavigation,
                false)) {
            //【5.2】NavigationBar 透明同时会拉伸 Window 到全屏，不保留 StatusBar 和 NavigationBar 的高度
            // 设置 FLAG_TRANSLUCENT_NAVIGATION 位；
            setFlags(FLAG_TRANSLUCENT_NAVIGATION, FLAG_TRANSLUCENT_NAVIGATION
                    & (~getForcedWindowFlags()));
        }

        if (a.getBoolean(R.styleable.Window_windowOverscan, false)) {
            //【5.3】允许 window contents 扩展到屏幕中的缩放区域内，如果有缩放区域的话；
            // 设置 FLAG_LAYOUT_IN_OVERSCAN 位；
            setFlags(FLAG_LAYOUT_IN_OVERSCAN, FLAG_LAYOUT_IN_OVERSCAN&(~getForcedWindowFlags()));
        }

        if (a.getBoolean(R.styleable.Window_windowShowWallpaper, false)) {
            //【5.4】使用系统桌面背景作为应用的背景，设置 FLAG_SHOW_WALLPAPER 位；
            setFlags(FLAG_SHOW_WALLPAPER, FLAG_SHOW_WALLPAPER&(~getForcedWindowFlags()));
        }

        if (a.getBoolean(R.styleable.Window_windowEnableSplitTouch,
                getContext().getApplicationInfo().targetSdkVersion
                        >= android.os.Build.VERSION_CODES.HONEYCOMB)) {
            //【5.5】支持触摸事件序列的拆分，设置 FLAG_SPLIT_TOUCH 位；
            setFlags(FLAG_SPLIT_TOUCH, FLAG_SPLIT_TOUCH&(~getForcedWindowFlags()));
        }

        //【6】获取和 Width/Height 相关的属性；
        a.getValue(R.styleable.Window_windowMinWidthMajor, mMinWidthMajor);
        a.getValue(R.styleable.Window_windowMinWidthMinor, mMinWidthMinor);
        if (DEBUG) Log.d(TAG, "Min width minor: " + mMinWidthMinor.coerceToString()
                + ", major: " + mMinWidthMajor.coerceToString());
        if (a.hasValue(R.styleable.Window_windowFixedWidthMajor)) {
            if (mFixedWidthMajor == null) mFixedWidthMajor = new TypedValue();
            a.getValue(R.styleable.Window_windowFixedWidthMajor,
                    mFixedWidthMajor);
        }
        if (a.hasValue(R.styleable.Window_windowFixedWidthMinor)) {
            if (mFixedWidthMinor == null) mFixedWidthMinor = new TypedValue();
            a.getValue(R.styleable.Window_windowFixedWidthMinor,
                    mFixedWidthMinor);
        }
        if (a.hasValue(R.styleable.Window_windowFixedHeightMajor)) {
            if (mFixedHeightMajor == null) mFixedHeightMajor = new TypedValue();
            a.getValue(R.styleable.Window_windowFixedHeightMajor,
                    mFixedHeightMajor);
        }
        if (a.hasValue(R.styleable.Window_windowFixedHeightMinor)) {
            if (mFixedHeightMinor == null) mFixedHeightMinor = new TypedValue();
            a.getValue(R.styleable.Window_windowFixedHeightMinor,
                    mFixedHeightMinor);
        }
        //【7】又处理一些 flags 设置；
        //【-->5.4】设置 flags；
        if (a.getBoolean(R.styleable.Window_windowContentTransitions, false)) {
            //【7.1】开启了 content 过渡(动画)，设置 FEATURE_CONTENT_TRANSITIONS
            requestFeature(FEATURE_CONTENT_TRANSITIONS);
        }
        if (a.getBoolean(R.styleable.Window_windowActivityTransitions, false)) {
            //【7.2】开启了 acitivty 过渡(动画)，设置 FEATURE_ACTIVITY_TRANSITIONS
            requestFeature(FEATURE_ACTIVITY_TRANSITIONS);
        }

        mIsTranslucent = a.getBoolean(R.styleable.Window_windowIsTranslucent, false);

        final Context context = getContext();
        //【8】获取和平台版本相关的属性；
        final int targetSdk = context.getApplicationInfo().targetSdkVersion;
        final boolean targetPreHoneycomb = targetSdk < android.os.Build.VERSION_CODES.HONEYCOMB;
        final boolean targetPreIcs = targetSdk < android.os.Build.VERSION_CODES.ICE_CREAM_SANDWICH;
        final boolean targetPreL = targetSdk < android.os.Build.VERSION_CODES.LOLLIPOP;
        final boolean targetHcNeedsOptions = context.getResources().getBoolean(
                R.bool.target_honeycomb_needs_options_menu);
        final boolean noActionBar = !hasFeature(FEATURE_ACTION_BAR) || hasFeature(FEATURE_NO_TITLE);

        //【9】根据当前 sdk 的版本判断需不需要加入 menukey
        if (targetPreHoneycomb || (targetPreIcs && targetHcNeedsOptions && noActionBar)) {
            setNeedsMenuKey(WindowManager.LayoutParams.NEEDS_MENU_SET_TRUE);
        } else {
            setNeedsMenuKey(WindowManager.LayoutParams.NEEDS_MENU_SET_FALSE);
        }

        //【10】如果应用没有强制设置 Status Bar 和 Navigation Bar 的颜色，那就使用默认颜色；
        if (!mForcedStatusBarColor) {
            mStatusBarColor = a.getColor(R.styleable.Window_statusBarColor, 0xFF000000);
        }
        if (!mForcedNavigationBarColor) {
            mNavigationBarColor = a.getColor(R.styleable.Window_navigationBarColor, 0xFF000000);
        }
		
        //【11】获取 window 的布局参数；
        WindowManager.LayoutParams params = getAttributes();

        //【12】如果不是浮动的 window 同时环视，如果 sdk 是 5.0 以上，同时 theme 设置了
        // 可绘制 sysetem bar 的背景，那就设置 FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS 标志位；
        if (!mIsFloating && ActivityManager.isHighEndGfx()) {
            if (!targetPreL && a.getBoolean(
                    R.styleable.Window_windowDrawsSystemBarBackgrounds,
                    false)) {
                setFlags(FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS,
                        FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS & ~getForcedWindowFlags());
            }
            if (mDecor.mForceWindowDrawsStatusBarBackground) {
                params.privateFlags |= PRIVATE_FLAG_FORCE_DRAW_STATUS_BAR_BACKGROUND;
            }
        }
        //【13】设置状态栏为浅色；
        if (a.getBoolean(R.styleable.Window_windowLightStatusBar, false)) {
            decor.setSystemUiVisibility(
                    decor.getSystemUiVisibility() | View.SYSTEM_UI_FLAG_LIGHT_STATUS_BAR);
        }
        //【14】设置点击 window 的外面，就关闭 window！
        if (mAlwaysReadCloseOnTouchAttr || getContext().getApplicationInfo().targetSdkVersion
                >= android.os.Build.VERSION_CODES.HONEYCOMB) {
            if (a.getBoolean(
                    R.styleable.Window_windowCloseOnTouchOutside,
                    false)) {
                setCloseOnTouchOutsideIfNotSet(true);
            }
        }
        //【15】设置软键盘交互效果！
        if (!hasSoftInputMode()) {
            params.softInputMode = a.getInt(
                    R.styleable.Window_windowSoftInputMode,
                    params.softInputMode);
        }

        //【16】设置 window 的幕布，也就是背景层，dimAmount 在 0.0f 和 1.0f 之间，
        // 0.0f 完全不暗，即背景是可见的 ，1.0f 时候，背景全部变黑暗。
        if (a.getBoolean(R.styleable.Window_backgroundDimEnabled,
                mIsFloating)) {
            /* All dialogs should have the window dimmed */
            if ((getForcedWindowFlags()&WindowManager.LayoutParams.FLAG_DIM_BEHIND) == 0) {
                //【16.1】布局参数增加 FLAG_DIM_BEHIND 标志位；
                params.flags |= WindowManager.LayoutParams.FLAG_DIM_BEHIND;
            }
            if (!haveDimAmount()) {
                params.dimAmount = a.getFloat(
                        android.R.styleable.Window_backgroundDimAmount, 0.5f);
            }
        }
        //【17】设置窗口动画；
        if (params.windowAnimations == 0) {
            params.windowAnimations = a.getResourceId(
                    R.styleable.Window_windowAnimationStyle, 0);
        }

        //【18】返回该 window 的容器，如果为 null，那么其是 top 级别的 window，也就是我们的 activity！
        // 这里主要是一些背景色的设置；
        if (getContainer() == null) {
            if (mBackgroundDrawable == null) {
                if (mBackgroundResource == 0) {
                    mBackgroundResource = a.getResourceId(
                            R.styleable.Window_windowBackground, 0);
                }
                if (mFrameResource == 0) {
                    mFrameResource = a.getResourceId(R.styleable.Window_windowFrame, 0);
                }
                mBackgroundFallbackResource = a.getResourceId(
                        R.styleable.Window_windowBackgroundFallback, 0);
                if (false) {
                    System.out.println("Background: "
                            + Integer.toHexString(mBackgroundResource) + " Frame: "
                            + Integer.toHexString(mFrameResource));
                }
            }
            if (mLoadElevation) {
                mElevation = a.getDimension(R.styleable.Window_windowElevation, 0);
            }
            mClipToOutline = a.getBoolean(R.styleable.Window_windowClipToOutline, false);
            mTextColor = a.getColor(R.styleable.Window_textColor, Color.TRANSPARENT);
        }

        //【19】核心代码：--> 加载布局文件！
        int layoutResource;
        int features = getLocalFeatures();
        // 通过对 features 和 mIsFloating 的判断，为 layoutResource 进行赋值，
        // 所以 requestFeature 要在 setContentView 之前进行；
        if ((features & (1 << FEATURE_SWIPE_TO_DISMISS)) != 0) {
            //【19.1】如果支持滑动退出，那么会加载 R.layout.screen_swipe_dismiss 这个布局文件；
            layoutResource = R.layout.screen_swipe_dismiss;

        } else if ((features & ((1 << FEATURE_LEFT_ICON) | (1 << FEATURE_RIGHT_ICON))) != 0) {
            //【19.2】如果在标题栏显示左右的按钮，那么根据是否是浮动，加载不同的布局；
            if (mIsFloating) {
                TypedValue res = new TypedValue();
                getContext().getTheme().resolveAttribute(
                        R.attr.dialogTitleIconsDecorLayout, res, true);
                layoutResource = res.resourceId;
            } else {
                //【19.2.2】加载 R.layout.screen_title_icons 这个布局文件
                layoutResource = R.layout.screen_title_icons;
            }
            //【19.2.2】移除 action bar 特性；
            removeFeature(FEATURE_ACTION_BAR);
            // System.out.println("Title Icons!");

        } else if ((features & ((1 << FEATURE_PROGRESS) | (1 << FEATURE_INDETERMINATE_PROGRESS))) != 0
                && (features & (1 << FEATURE_ACTION_BAR)) == 0) {
            //【19.3】如果 window 只设置了一个 progress bar
            // 那么加载 R.layout.screen_progress 布局；
            layoutResource = R.layout.screen_progress;

        } else if ((features & (1 << FEATURE_CUSTOM_TITLE)) != 0) {
			//【19.4】如果 window 设置了定制的 title 特性，如果是悬浮的 window，那么会加载一个 dialog 的布局
            // 否则加载 R.layout.screen_custom_title 布局；
            if (mIsFloating) {
                TypedValue res = new TypedValue();
                getContext().getTheme().resolveAttribute(
                        R.attr.dialogCustomTitleDecorLayout, res, true);
                layoutResource = res.resourceId;
            } else {
                layoutResource = R.layout.screen_custom_title;
            }
            // 移除 action bar 特性；
            removeFeature(FEATURE_ACTION_BAR);

        } else if ((features & (1 << FEATURE_NO_TITLE)) == 0) {
            //【19.5】如果 window 设置了 no title 特性，如果是悬浮的 window，那么会加载一个 dialog 的布局
            // 否则如果设置了 action bar 特性，那么就加载 R.layout.screen_action_bar 的布局；
            // 其他情况，加载 R.layout.screen_custom_title 布局；
            if (mIsFloating) {
                TypedValue res = new TypedValue();
                getContext().getTheme().resolveAttribute(
                        R.attr.dialogTitleDecorLayout, res, true);
                layoutResource = res.resourceId;
                
            } else if ((features & (1 << FEATURE_ACTION_BAR)) != 0) {
                layoutResource = a.getResourceId(
                        R.styleable.Window_windowActionBarFullscreenDecorLayout,
                        R.layout.screen_action_bar);
                
            } else {
                layoutResource = R.layout.screen_title;
            }

        } else if ((features & (1 << FEATURE_ACTION_MODE_OVERLAY)) != 0) {
            //【19.5】如果设置了 action mode overlay 特性，那么 action bar 会覆盖在 content 上面
            // 加载 R.layout.screen_simple_overlay_action_mode 布局；
            layoutResource = R.layout.screen_simple_overlay_action_mode;

        } else {
            //【19.6】在没有设置任何 feature（也就是装饰）时选用默认布局
            layoutResource = R.layout.screen_simple;

        }
        //【-->6.3】标记 DecorView 正准备加载；
        mDecor.startChanging();
        //【-->6.4】开始加载布局文件，
        mDecor.onResourcesLoaded(mLayoutInflater, layoutResource);

        //【20】找到内置布局里面的 id 为 ID_ANDROID_CONTENT 的 ViewGroup，保存到 contentParent 临时变量中；
        // 目的很明显，要加载 setContentView 设置的布局了；
        ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
        if (contentParent == null) {
            throw new RuntimeException("Window couldn't find content container view");
        }

        if ((features & (1 << FEATURE_INDETERMINATE_PROGRESS)) != 0) {
            ProgressBar progress = getCircularProgressBar(false);
            if (progress != null) {
                progress.setIndeterminate(true);
            }
        }

        if ((features & (1 << FEATURE_SWIPE_TO_DISMISS)) != 0) {
            registerSwipeCallbacks();
        }

        //【21】只针对于 top level 的 window 才会进行处理；
        if (getContainer() == null) {
            final Drawable background;
            //【21.1】PhoneWindow 有一个 setBackgroundDrawable 方法可以用来设置 mBackgroundResource；
            if (mBackgroundResource != 0) {
                background = getContext().getDrawable(mBackgroundResource);
            } else {
                background = mBackgroundDrawable;
            }
            //【-->6.5】设置 window 背景，最终会触发 drawableChanged 方法；
            mDecor.setWindowBackground(background);

            final Drawable frame;
            if (mFrameResource != 0) {
                frame = getContext().getDrawable(mFrameResource);
            } else {
                frame = null;
            }
            //【-->6.5】设置 window Frame，最终会触发 drawableChanged 方法；
            mDecor.setWindowFrame(frame);

            mDecor.setElevation(mElevation);
            mDecor.setClipToOutline(mClipToOutline);

            if (mTitle != null) {
                setTitle(mTitle);
            }

            if (mTitleColor == 0) {
                mTitleColor = mTextColor;
            }
            setTitleColor(mTitleColor);
        }

        //【-->6.3】此时，DecorView 中的 root view 已经加载完了
        // 包括背景，标题什么都已经设置完成了，这里会结束 changing，触发 drawableChanged 方法；
        mDecor.finishChanging();

        return contentParent;
    }
```

这里大家对下面的计算可能摸不着头脑：

```java
 setFlags(FLAG_FULLSCREEN, FLAG_FULLSCREEN & (~getForcedWindowFlags()));
```

分情况下：

- 如果 getForcedWindowFlags() 没有设置 FLAG_FULLSCREEN，那么 FLAG_FULLSCREEN & (~getForcedWindowFlags()) 为 FLAG_FULLSCREEN
- 如果 getForcedWindowFlags() 设置了 FLAG_FULLSCREEN，那么 FLAG_FULLSCREEN & (~getForcedWindowFlags()) 为 0；

然后大家可以分析 setLayout 方法是如何处理的了；



其实，整个过程：

- 第一个阶段，是在处理 window 的 theme 和 flags，因为这些设置会影响 window 的特性；
- 第二个阶段，是选择 root 布局，并加载到 DecorView；



#### 4.2.2.1 requestFeature

应用窗口特性：

- 参数 **featureId** 是特性的 **id**，需要将其转为对应的二进制序列：**(1 << featureId)**；

```java
    @Override
    public boolean requestFeature(int featureId) {
        if (mContentParentExplicitlySet) {
            throw new AndroidRuntimeException("requestFeature() must be called before adding content");
        }
        //【1】获取默认的 feture！，getFeatures 返回的是 window.mFeatures，这就不再分析了；
        final int features = getFeatures();
        //【2】设置新的 feature；
        final int newFeatures = features | (1 << featureId);
        //【3】然后做一些冲突判断；
        if ((newFeatures & (1 << FEATURE_CUSTOM_TITLE)) != 0 &&
                (newFeatures & ~CUSTOM_TITLE_COMPATIBLE_FEATURES) != 0) {
            //【3.1】如果设置了 FEATURE_CUSTOM_TITLE ，那么是不能设置其他和 title 类型不兼容的 feature 的！
            throw new AndroidRuntimeException(
                    "You cannot combine custom titles with other title features");
        }
        if ((features & (1 << FEATURE_NO_TITLE)) != 0 && featureId == FEATURE_ACTION_BAR) {
            //【3.2】如果要设置的是 action bar，然而 feature 已经设置了 no title，设置失败！
            return false; 
        }
        if ((features & (1 << FEATURE_ACTION_BAR)) != 0 && featureId == FEATURE_NO_TITLE) {
            //【3.3】如果要设置的是 no title，然而 feature 已经设置了 actionbar，那就要去掉 actionbar；
            removeFeature(FEATURE_ACTION_BAR);
        }

        if ((features & (1 << FEATURE_ACTION_BAR)) != 0 && featureId == FEATURE_SWIPE_TO_DISMISS) {
            //【3.4】如果要设置的是右滑退出（swipe to dismiss），但是 feature 已经有 action bar 了
            // 这里会产生冲突；
            throw new AndroidRuntimeException(
                    "You cannot combine swipe dismissal and the action bar.");
        }
        if ((features & (1 << FEATURE_SWIPE_TO_DISMISS)) != 0 && featureId == FEATURE_ACTION_BAR) {
            //【3.5】如果要设置的是 action bar ，但是 feature 已经有 右滑退出（swipe to dismiss）了
            // 冲突！！
            throw new AndroidRuntimeException(
                    "You cannot combine swipe dismissal and the action bar.");
        }

        if (featureId == FEATURE_INDETERMINATE_PROGRESS &&
                getContext().getPackageManager().hasSystemFeature(PackageManager.FEATURE_WATCH)) {
            //【3.6】如果要设置的是 不确定的进度 特性，且此时平台是 watch，那就会抛出异常；
            throw new AndroidRuntimeException("You cannot use indeterminate progress on a watch.");
        }
        //【4】设置新的 feature 位到 mFeatures 和 mLocalFeatures！
        return super.requestFeature(featureId);
    }
```

这里就不再继续分析了～～

- 下面是默认特性（DEFAULT_FEATURES）、和 CUSTOM_TITLE 特性兼容的特性的定义！

```java
// --> Window.java
@Deprecated
@SuppressWarnings({"PointlessBitwiseExpression"})
protected static final int DEFAULT_FEATURES = (1 << FEATURE_OPTIONS_PANEL) |
    (1 << FEATURE_CONTEXT_MENU);

// --> PhoneWindow.java
private static final int CUSTOM_TITLE_COMPATIBLE_FEATURES = DEFAULT_FEATURES |
            (1 << FEATURE_CUSTOM_TITLE) |
            (1 << FEATURE_CONTENT_TRANSITIONS) |
            (1 << FEATURE_ACTIVITY_TRANSITIONS) |
            (1 << FEATURE_ACTION_MODE_OVERLAY);
```

关于上面的特性具体的意思，这里不在详细分析。



#### 4.2.2.2 Root 布局分析

上面的 xml 文件位于 frameworks/base/core/res/res/layout 下，我们选择几个来看看：

##### 4.2.2.2.1 默认布局（不使用 feature）

```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:fitsSystemWindows="true"
    android:orientation="vertical">
    <!--action bar-->
    <ViewStub android:id="@+id/action_mode_bar_stub"
              android:inflatedId="@+id/action_mode_bar"
              android:layout="@layout/action_mode_bar"
              android:layout_width="match_parent"
              android:layout_height="wrap_content"
              android:theme="?attr/actionBarTheme" />
    <!--content-->
    <FrameLayout
         android:id="@android:id/content"
         android:layout_width="match_parent"
         android:layout_height="match_parent"
         android:foregroundInsidePadding="false"
         android:foregroundGravity="fill_horizontal|top"
         android:foreground="?android:attr/windowContentOverlay" />
</LinearLayout>
```



##### 4.2.2.2.2 支持滑动退出的布局

```xml
<!--content-->
<com.android.internal.widget.SwipeDismissLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@android:id/content"
    android:fitsSystemWindows="true"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    />
```



##### 4.2.2.2.3 只有一个进度条的布局

```xml
<FrameLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:fitsSystemWindows="true">
    <!--content-->
    <FrameLayout
         android:id="@android:id/content"
         android:layout_width="match_parent"
         android:layout_height="match_parent"
         android:foregroundInsidePadding="false"
         android:foregroundGravity="fill_horizontal|top"
         android:foreground="?android:attr/windowContentOverlay" />
    <!--action bar-->
    <ViewStub android:id="@+id/action_mode_bar_stub"
              android:inflatedId="@+id/action_mode_bar"
              android:layout="@layout/action_mode_bar"
              android:layout_width="match_parent"
              android:layout_height="wrap_content"
              android:theme="?attr/actionBarTheme" />
</FrameLayout>
```



##### 4.2.2.2.4 总结

通过布局的分析，我们可以知道：

- 所有的布局都会有一个 id 为 "@android:id/content" 的 Layout，这个 Layout 就是用来放置我们 setContentView 设置的 content 的；



## 4.2 dispatchWindowAttributesChanged

```java
    @Override
    protected void dispatchWindowAttributesChanged(WindowManager.LayoutParams attrs) {
        //【-->5.6】先调用了父类的 dispatchWindowAttributesChanged 方法
        super.dispatchWindowAttributesChanged(attrs);
        if (mDecor != null) {
            mDecor.updateColorViews(null /* insets */, true /* animate */);
        }
    }
```



# 5 Window

是一个抽象类，定义了窗口的所有公共操作和属性，phoneWindow 是他的唯一实现类；

## 5.1 new Window

由于 PhoneWindow 是其唯一的实现类，所以一般只有通过 new PhoneWindow 才能触发父类构造器：

```java
    public Window(Context context) {
        //【1】就是 activity，不多说了！
        mContext = context;
        //【2】获取默认的 feature！
        mFeatures = mLocalFeatures = getDefaultFeatures(context);
    }
```

方法很简单，Window 内部也有很多的成员变量，这里我们不一一分析，我们后面会边跟代码，边分析内部的变量！

mFeatures 和 mLocalFeatures 是 window 内部的窗口特性，是一个二进制序列；

- **getDefaultFeatures**：获取默认的 feature 配置，这个方法不太重要，就不单独列出来了！



```java
    public static int getDefaultFeatures(Context context) {
        int features = 0;

        final Resources res = context.getResources();
        //【1】是否启用 “选项面板” 功能，默认已启用，设置 FEATURE_OPTIONS_PANEL 位；
        if (res.getBoolean(com.android.internal.R.bool.config_defaultWindowFeatureOptionsPanel)) {
            features |= 1 << FEATURE_OPTIONS_PANEL;
        }

        if (res.getBoolean(com.android.internal.R.bool.config_defaultWindowFeatureContextMenu)) {
            features |= 1 << FEATURE_CONTEXT_MENU;
        }

        return features;
    }

```

这里我们先来看看 Window 内部的几个接口，activity 都实现了这几个接口！

### 5.1.1 内部接口

#### 5.1.1.1 Callback

这个接口用于处理事件的分发，菜单操作，窗口属性变化等事件回调：

```java
    /**
    * API from a Window back to its caller.  This allows the client to
    * intercept key dispatching, panels and menus, etc.
    */
    public interface Callback {
        ... ... ...// 先不关注，接口太多了：
    }
```

下面是其内部的方法，具体实现都在 activity 里面：



<img src="pic/ViewDraw-1-setContentView.asserts/View-Callback.png" alt="View-Callback" style="zoom:50%;" />



其内部接口很多，我们这里不过多分析！！



#### 5.1.1.2 WindowControllerCallback

和窗口控制相关的回调接口：比如：退出 FreeForm 模式，进入画中画模式等等；

```java
    /** @hide */
    public interface WindowControllerCallback {  
        // 退出 freeform 模式，（翻译注释）其实就是将 activity 从 freeform stack 移动到 full screen stack
        void exitFreeformMode() throws RemoteException;

        // 如果 acitivity 支持画中画模式，就进入画中画模式（@see android.R.attr#supportsPictureInPicture）
        void enterPictureInPictureModeIfPossible();

        /** Returns the current stack Id for the window. */
        int getWindowStackId() throws RemoteException;
    }
```

不多说了！

#### 5.1.1.3 OnWindowDismissedCallback

window 被销毁触发的回调：

```java
    /** @hide */
    public interface OnWindowDismissedCallback {
        // window 被销毁触发，通知 activity finsh 掉自己，参数 finishTask 为 ture 表示同时清理掉所在的 task
        void onWindowDismissed(boolean finishTask);
    }
```







## 5.2 设置 window 回调

window 有下面三个成员变量，其实都是所属的 activity，因为 activity 实现了这几个接口；

```java
    private Callback mCallback;
    private OnWindowDismissedCallback mOnWindowDismissedCallback;
    private WindowControllerCallback mWindowControllerCallback;
```

具体的作用我们后面再分析；

### 5.2.1 setCallback

```java
    public void setCallback(Callback callback) {
        mCallback = callback;
    }
```



### 5.2.2 setWindowControllerCallback

```java
    /** @hide */
    public final void setWindowControllerCallback(WindowControllerCallback wccb) {
        mWindowControllerCallback = wccb;
    }
```



### 5.2.3 setOnWindowDismissedCallback

```java
    /** @hide */
    public final void setOnWindowDismissedCallback(OnWindowDismissedCallback dcb) {
        mOnWindowDismissedCallback = dcb;
    }
```



## 5.3 getAttributes

获得当前的 activity 的 window 布局属性：

```java
    public final WindowManager.LayoutParams getAttributes() {
        return mWindowAttributes;
    }
```

这里的 mWindowAttributes 是抽象类 Window 的内部属性，表示当前 window 的布局属性：

```java
// --> Window.java
private final WindowManager.LayoutParams mWindowAttributes =
        new WindowManager.LayoutParams();
```

不多说了！

## 5.4 setLayout

设置布局属性：

```java
    public void setLayout(int width, int height) {
        final WindowManager.LayoutParams attrs = getAttributes();
        attrs.width = width;
        attrs.height = height;
        //【-->4.2】分发 window 布局属性改变的消息；
        dispatchWindowAttributesChanged(attrs);
    }
```

这个方法用于设置 window 的 width/height 布局属性，默认是 MATCH_PARENT，表示的是全屏的 window！！！

如果是 WRAP_CONTENT 的话，那就不是全屏（full-screen）的  window！！

**dispatchWindowAttributesChanged** 方法，其子类 **PhoneWindow** 覆写了该方法！！



## 5.5 setFlags

用于设置窗口的 flags 位：

- int flags：表示新的 flags 位；
- int mask：要修改的窗口标志位；

```java
    public void setFlags(int flags, int mask) {
        //【--> 5.3】返回 window 的布局属性：
        final WindowManager.LayoutParams attrs = getAttributes();
        //【1】这里先是从 attrs.flags 中去掉 mask 中的标志，然后再加上了 flags 和 mask 相同的标志；
        attrs.flags = (attrs.flags&~mask) | (flags&mask);
        mForcedWindowFlags |= mask;
        //【-->4.x】分发 window 布局属性改变的消息；
        dispatchWindowAttributesChanged(attrs);
    }
```

我们看到，其实最终生效的 flags，是保存在 WindowManager.LayoutParams 中！

这里的 mForcedWindowFlags 用来保存：应用程序显式设置的 flags 标志位，默认值为 0；

```java
    private int mForcedWindowFlags = 0;

    // 该方法用于返回 mForcedWindowFlags；
    protected final int getForcedWindowFlags() {
        return mForcedWindowFlags;
    }
```

- addFlags：增加一个标志位！

```java
    public void addFlags(int flags) {
        setFlags(flags, flags);
    }
```

- clearFlags：去掉一个标志位！

```java
    public void clearFlags(int flags) {
        setFlags(0, flags);
    }
```



## 5.6 dispatchWindowAttributesChanged

分发窗口属性变化的消息：

```java
    protected void dispatchWindowAttributesChanged(WindowManager.LayoutParams attrs) {
        //【1】这里的 mCallback 就是 activity；
        if (mCallback != null) {
            //【-->3.2】回调 mCallback 的 onWindowAttributesChanged 方法：
            mCallback.onWindowAttributesChanged(attrs);
        }
    }
```

回调通知，不多说。

# 6 DecorView

可以看到 DecorView 本质上是一个 FrameLayout！

```java
/** @hide */
public class DecorView extends FrameLayout implements RootViewSurfaceTaker, WindowCallbacks {
    ... ... ...
    //【1】表示所属的 PhoneWindow；
    private PhoneWindow mWindow;
    //【2】我们通过 setContentView 设置的布局 view；
    ViewGroup mContentRoot;
    ... ... ...    
}
```

同时他也实现了 RootViewSurfaceTaker 和 WindowCallbacks 接口！

当然，对于 DecorView 其内部成员也很多，这里也就不一一分析。。。

##  6.1 new DecorView

我们来看看创建 DecorView 实例：

```java
    DecorView(Context context, int featureId, PhoneWindow window,
            WindowManager.LayoutParams params) {
        super(context);
        //【1】用于指定 window 的类型，-1 表示是 activity 的 window； 
        mFeatureId = featureId;
        //【2】这一些是和系统特性，动画相关的一些成员，我们不关注；
        mShowInterpolator = AnimationUtils.loadInterpolator(context,
                android.R.interpolator.linear_out_slow_in);
        mHideInterpolator = AnimationUtils.loadInterpolator(context,
                android.R.interpolator.fast_out_linear_in);

        mBarEnterExitDuration = context.getResources().getInteger(
                R.integer.dock_enter_exit_duration);
        mForceWindowDrawsStatusBarBackground = context.getResources().getBoolean(
                R.bool.config_forceWindowDrawsStatusBarBackground)
                && context.getApplicationInfo().targetSdkVersion >= N;
        mSemiTransparentStatusBarColor = context.getResources().getColor(
                R.color.system_bar_background_semi_transparent, null /* theme */);
 
        //【3】读取可用的屏幕宽度；
        updateAvailableWidth();
        //【-->6.2】设置所属 window；
        setWindow(window);

        updateLogTag(params);

        mResizeShadowSize = context.getResources().getDimensionPixelSize(
                R.dimen.resize_shadow_size);
        initResizingPaints();
    }
```

这个 mFeatureId 是 DecorView 的成员：

```java
    /** The feature ID of the panel, or -1 if this is the application's DecorView */
    private final int mFeatureId;
```



## 6.2 setWindow

将 PhoneWindow 和 Context 的实例保存到 DecorView 中；

```java
    void setWindow(PhoneWindow phoneWindow) {
        //【1】保存 PhoneWindow ，Context 的实例到内部成员变量中；
        mWindow = phoneWindow;
        Context context = getContext();
        //【2】如果 Context 是 DecorContext 的实例，那么 DecorContext 也会持有 PhoneWindow 的引用；
        if (context instanceof DecorContext) {
            DecorContext decorContext = (DecorContext) context;
            decorContext.setPhoneWindow(mWindow);
        }
    }
```

不多说了！

## 6.3 startChanging / finishChanging

用于（结束）标记 DecorView 正在加载：

```java
    void startChanging() {
        mChanging = true;
    }

    void finishChanging() {
        mChanging = false;
        //【-->6.5】可绘制的区域发生的变化；
        drawableChanged();
    }
```



## 6.4 onResourcesLoaded

加载布局资源：

```java
    void onResourcesLoaded(LayoutInflater inflater, int layoutResource) {
        //【1】获取当前 window 所在的 stack！
        mStackId = getStackId();

        if (mBackdropFrameRenderer != null) {
            loadBackgroundDrawablesIfNeeded();
            mBackdropFrameRenderer.onResourcesLoaded(
                    this, mResizingBackgroundDrawable, mCaptionBackgroundDrawable,
                    mUserCaptionBackgroundDrawable, getCurrentColor(mStatusColorViewState),
                    getCurrentColor(mNavigationColorViewState));
        }
        //【2】创建一个 DecorCaptionView，类似于是一个中间层的封装；
        mDecorCaptionView = createDecorCaptionView(inflater);
        //【3】通过 LayoutInflater 加载 layoutResource 对应的布局文件！
        final View root = inflater.inflate(layoutResource, null);
        if (mDecorCaptionView != null) {
            //【4】如果 mDecorCaptionView 不为 null，那么会先将 mDecorCaptionView 加入到  DecorView 中；
            // 最后将 root 加入到 mDecorCaptionView 中；
            if (mDecorCaptionView.getParent() == null) {
                addView(mDecorCaptionView,
                        new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));
            }
            mDecorCaptionView.addView(root,
                    new ViewGroup.MarginLayoutParams(MATCH_PARENT, MATCH_PARENT));
        } else {

            // Put it below the color views.
            addView(root, 0, new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));
        }
        //【4】设置 content root view！
        mContentRoot = (ViewGroup) root;
        initializeElevation();
    }
```

可以看到：

- 主要流程就是加载 layoutResource 对应的布局文件（content root），加载的 view 会保存到 DecorView.mContentRoot 中；
- 然后通过 addView 将 content root 其加载到 DecorView 中去；

我们去看看 ViewGroup 的 addView 方法，可以看到也调用了 requestLayout() 和 invalidate(true) 方法！

```java
    public void addView(View child, int index, LayoutParams params) {
        if (DBG) {
            System.out.println(this + " addView");
        }

        if (child == null) {
            throw new IllegalArgumentException("Cannot add a null child view to a ViewGroup");
        }

        //【1】请求布局操作；
        requestLayout();
        //【2】重绘；
        invalidate(true);
        //【3】会调用 child.requestLayout() 方法，因为设置了新的布局参数；
        // 必须要等到 DecorView 的 requestLayout 执行完成后才会执行；
        addViewInner(child, index, params, false);
    }
```

这里虽然调用了 requestLayout 和 invalidate 实际上并没有开始布局和绘制，原因是 ViewRootImpl 还没有创建；

## 6.5 setWindowBackground / setWindowFrame

设置 window 背景和 frame：

```java
    public void setWindowBackground(Drawable drawable) {
        if (getBackground() != drawable) {
            setBackgroundDrawable(drawable);
            if (drawable != null) {
                mResizingBackgroundDrawable = enforceNonTranslucentBackground(drawable,
                        mWindow.isTranslucent() || mWindow.isShowingWallpaper());
            } else {
                mResizingBackgroundDrawable = getResizingBackgroundDrawable(
                        getContext(), 0, mWindow.mBackgroundFallbackResource,
                        mWindow.isTranslucent() || mWindow.isShowingWallpaper());
            }
            if (mResizingBackgroundDrawable != null) {
                mResizingBackgroundDrawable.getPadding(mBackgroundPadding);
            } else {
                mBackgroundPadding.setEmpty();
            }
            //【-->6.6】可绘制的区域发生的变化
            drawableChanged();
        }
    }

    public void setWindowFrame(Drawable drawable) {
        if (getForeground() != drawable) {
            setForeground(drawable);
            if (drawable != null) {
                drawable.getPadding(mFramePadding);
            } else {
                mFramePadding.setEmpty();
            }
            //【-->6.6】可绘制的区域发生的变化
            drawableChanged();
        }
    }
```

可以看到，这里都调用了 drawableChanged 方法：



## 6.6 drawableChanged

可绘制的区域发生的变化，实际上就是 window 的属性发生了变化：

```java
    private void drawableChanged() {
        if (mChanging) {
            return;
        }
        //【1】设置内容的 padding 距离；
        setPadding(mFramePadding.left + mBackgroundPadding.left,
                mFramePadding.top + mBackgroundPadding.top,
                mFramePadding.right + mBackgroundPadding.right,
                mFramePadding.bottom + mBackgroundPadding.bottom);
        //【2】请求布局；
        requestLayout();
        //【3】触发绘制；
        invalidate();

        //【4】根据 PixelFormat 类型设置窗口的默认格式，这里我们先不看；
        int opacity = PixelFormat.OPAQUE;
        if (StackId.hasWindowShadow(mStackId)) {
            // If the window has a shadow, it must be translucent.
            opacity = PixelFormat.TRANSLUCENT;
        } else{
            // Note: If there is no background, we will assume opaque. The
            // common case seems to be that an application sets there to be
            // no background so it can draw everything itself. For that,
            // we would like to assume OPAQUE and let the app force it to
            // the slower TRANSLUCENT mode if that is really what it wants.
            Drawable bg = getBackground();
            Drawable fg = getForeground();
            if (bg != null) {
                if (fg == null) {
                    opacity = bg.getOpacity();
                } else if (mFramePadding.left <= 0 && mFramePadding.top <= 0
                        && mFramePadding.right <= 0 && mFramePadding.bottom <= 0) {
                    // If the frame padding is zero, then we can be opaque
                    // if either the frame -or- the background is opaque.
                    int fop = fg.getOpacity();
                    int bop = bg.getOpacity();
                    if (false)
                        Log.v(mLogTag, "Background opacity: " + bop + ", Frame opacity: " + fop);
                    if (fop == PixelFormat.OPAQUE || bop == PixelFormat.OPAQUE) {
                        opacity = PixelFormat.OPAQUE;
                    } else if (fop == PixelFormat.UNKNOWN) {
                        opacity = bop;
                    } else if (bop == PixelFormat.UNKNOWN) {
                        opacity = fop;
                    } else {
                        opacity = Drawable.resolveOpacity(fop, bop);
                    }
                } else {
                    // For now we have to assume translucent if there is a
                    // frame with padding... there is no way to tell if the
                    // frame and background together will draw all pixels.
                    if (false)
                        Log.v(mLogTag, "Padding: " + mFramePadding);
                    opacity = PixelFormat.TRANSLUCENT;
                }
            }
            if (false)
                Log.v(mLogTag, "Background: " + bg + ", Frame: " + fg);
        }

        if (false)
            Log.v(mLogTag, "Selected default opacity: " + opacity);

        mDefaultOpacity = opacity;
        if (mFeatureId < 0) {
            mWindow.setDefaultWindowFormat(opacity);
        }
    }

```

这里核心的代码就是：

```java
//【1】设置内容的 padding 距离；
setPadding(mFramePadding.left + mBackgroundPadding.left,
mFramePadding.top + mBackgroundPadding.top,
mFramePadding.right + mBackgroundPadding.right,
mFramePadding.bottom + mBackgroundPadding.bottom);

//【2】请求布局；
requestLayout();
//【3】触发绘制；
invalidate();
```

这里就不再继续跟踪了，requestLayout 和 invalidate 后面再分析。



# 7 总结

我们来看下整个的流程吧：

```sequence
ActivityThread -> ActivityThread : 1.handleLaunchActivity
ActivityThread -> WindowManagerGlobal : 2.initialize（初始化 wmg 进程单例）

ActivityThread -> ActivityThread : 3.performLaunchActivity
ActivityThread -> Activity : 4.attach（开始绑定）

Activity -> PhoneWindow: 5.new PhoneWindow（创建 PhoneWindow 实例）
PhoneWindow --> Activity: 5.return mWindow: PhoneWindow

Activity -> PhoneWindow: 6.setWindowControllerCallback（设置回调）
Activity -> PhoneWindow: 7.setCallback
Activity -> PhoneWindow: 8.setOnWindowDismissedCallback

Activity -> PhoneWindow: 9.setWindowManager

Activity -> PhoneWindow: 10.getWindowManager
PhoneWindow --> Activity: 10.return WindowManagerImpl

ActivityThread -> Activity: 11.OnCreate
Activity -> Activity: 12.setContentView (设置 layout 布局资源)
Activity -> Activity: 13.getWindow
Activity -> PhoneWindow: 14.setContentView

PhoneWindow -> PhoneWindow: 15.installDecor
PhoneWindow -> PhoneWindow: 16.generateDecor

PhoneWindow -> DecorView: 17.new DecorView
DecorView --> PhoneWindow: 18.return DecorView

DecorView -> DecorView: 19.setWindow (设置 PhoneWindow)
PhoneWindow -> PhoneWindow: 20.generateLayout(加载布局文件) onResourcesLoaded
PhoneWindow -> DecorView: 21.onResourcesLoaded（该方法通过 LayoutInflater 加载布局，addView）
```

我画出了整个的流程图，可以发现，其实逻辑不是很复杂；



# 8 拓展分析



## 8.1 requestFeature

前面我们分析了 window feature 的概念，其实我们也可以动态设置 feature，比如我们在自己的代码中这样：

```java
getWindow().requestFeature(..)
```

这是通过 PhoneWindow 提供的如下方法来设置的：

注意：

- 这个方法一定要在 setContentView 之前调用，可以调用多次！
- 一旦设置了 feature，那就不能关闭；
- FEATURE_CUSTOM_TITLE 不能和其他的 title feature 一起使用；

```java
    public boolean requestFeature(int featureId) {
        final int flag = 1<<featureId;
        mFeatures |= flag;
        mLocalFeatures |= mContainer != null ? (flag&~mContainer.mFeatures) : flag;
        return (mFeatures&flag) != 0;
    }
```

其实就是在 mFeatures 和 mLocalFeatures 的基础上，增加了 featureId 的值；