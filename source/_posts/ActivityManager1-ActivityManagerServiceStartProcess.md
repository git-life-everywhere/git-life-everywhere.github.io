# ActivityManager第 1 篇 - ActivityManagerService 的启动
title: ActivityManager第 1 篇 - ActivityManagerService 的启动
date: 2016/02/19 20:46:25
catalog: true
categories:
- AndroidFramework源码分析
- ActivityManager活动管理
tags: ActivityManager活动管理
---

[toc]

基于 `Android 7.1.1` 源码分析 `AMS` 的机制，本文为作者原创，转载请说明出处，谢谢！ 

# 0 综述

`Android` 系统开机时，在 `SystemService` 进程被 `Zygote` 启动后，`SystemSevice` 进程需要启动一些系统的重要服务：
```java
            Trace.traceBegin(Trace.TRACE_TAG_SYSTEM_SERVER, "StartServices");
            startBootstrapServices();
            startCoreServices();
            startOtherServices();
```
`AMS` 的启动涉及到了以上的每个阶段，以为 `AMS` 需要和其他的系统服务进行交互，下面我们来逐一分析:

# 1 第一阶段 - startBootstrapServices

我们来看看低级阶段，和 `AMS` 相关的代码；

```java
    private void startBootstrapServices() {
        
        // 启动 Installer 系统服务
        Installer installer = mSystemServiceManager.startService(Installer.class);

        //【1】第一阶段：启动 AMS
        mActivityManagerService = mSystemServiceManager.startService(
                ActivityManagerService.Lifecycle.class).getService();
        mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
        mActivityManagerService.setInstaller(installer);

        //【2】第二阶段，启动 PowerManagerService，AMS 初始化电池管理属性
        mPowerManagerService = mSystemServiceManager.startService(PowerManagerService.class);
        mActivityManagerService.initPowerManagement();

        // 启动 LED 和背光灯、显示、包管理这些重要的系统服务！
        mSystemServiceManager.startService(LightsService.class);
        mDisplayManagerService = mSystemServiceManager.startService(DisplayManagerService.class);

        mSystemServiceManager.startBootPhase(SystemService.PHASE_WAIT_FOR_DEFAULT_DISPLAY);

        // 获得系统属性 vold.decrypt 的值，表示是否只加载核心服务；
        String cryptState = SystemProperties.get("vold.decrypt");
        if (ENCRYPTING_STATE.equals(cryptState)) {
            Slog.w(TAG, "Detected encryption in progress - only parsing core apps");
            mOnlyCore = true;

        } else if (ENCRYPTED_STATE.equals(cryptState)) {
            Slog.w(TAG, "Device encrypted - only parsing core apps");
            mOnlyCore = true;

        }
        
        // 创建 PMS 服务对象，
        mPackageManagerService = PackageManagerService.main(mSystemContext, installer,
                mFactoryTestMode != FactoryTest.FACTORY_TEST_OFF, mOnlyCore);
        mFirstBoot = mPackageManagerService.isFirstBoot();
        // 和 A/B 升级相关！
        if (!mOnlyCore) {
            boolean disableOtaDexopt = SystemProperties.getBoolean("config.disable_otadexopt",
                    false);
            if (!disableOtaDexopt) {
                traceBeginAndSlog("StartOtaDexOptService");
                try {
                    OtaDexoptService.main(mSystemContext, mPackageManagerService);
                } catch (Throwable e) {
                    reportWtf("starting OtaDexOptService", e);
                } finally {
                    Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);
                }
            }
        }

        // 启动用户管理服务！
        mSystemServiceManager.startService(UserManagerService.LifeCycle.class);

        // 初始化属性缓存，用于缓存包资源
        AttributeCache.init(mSystemContext);

        //【4】设置系统进程相关的参数！
        mActivityManagerService.setSystemProcess();

        // 启动 Sensor 相关的服务。这是一个 Native 方法，与硬件相关度较大，这里不关注。
        startSensorService();
    }
```

- 启动 `AMS` 之前，需要先连接 `installd` 进程， 通过 `Installer` 这个服务，就能完成一些重要目录的创建，譬如 `/data/user`，同时应用程序的安装，也需要这个服务。
- 启动 `ActivityManagerService` 服务；
- 启动 `PowerManagerService` 服务，`AMS` 需要使用 `PowerManagerService` 的服务：譬如，在启动 `Activity` 时，要避免系统进入休眠状态，就需要获取 `WakeLock`；
- 启动 `LightsService`、 `DisplayManagerService`、 `PackageManagerService` 系统服务；
- 启动 `OtaDexoptService` 服务，这里是和 A/B 升级相关的；
- 调用 `AMS.setSystemProcess()` 设置当前进程为系统进程，设置系统进程相关的参数；
   - 因为 `AMS` 的职责之一就是维护系统中所有进程的状态，不管是应用进程还是系统进程，都是 `AMS` 的管辖范围。

接下来，我们来看看和 `AMS` 相关的流程！

## 1.1 ActivityManagerService

接下来，我们仔细的看看 `AMS` 的构造器！
```java
    public ActivityManagerService(Context systemContext) {

        // 获得系统进程的上下文环境；
        mContext = systemContext;
        mFactoryTest = FactoryTest.getMode(); // 是否是工厂模式！
        
        // 获得 SystemServer 进程的 ActivityThread 对象，每个进程都有一个！
        mSystemThread = ActivityThread.currentActivityThread();

        Slog.i(TAG, "Memory class: " + ActivityManager.staticGetMemoryClass());

        // 创建处理消息的 HandlerThread 和 Handler 对象！
        mHandlerThread = new ServiceThread(TAG,
                android.os.Process.THREAD_PRIORITY_FOREGROUND, false /*allowIo*/);
        mHandlerThread.start();
        mHandler = new MainHandler(mHandlerThread.getLooper());

        // 创建处理 ANR 相关的 Handler！
        mUiHandler = new UiHandler();

        // 这里的 sKillHandler 绑定了一个 handlerThread 线程对象，用于处理杀进程的消息！
        if (sKillHandler == null) {
            sKillThread = new ServiceThread(TAG + ":kill",
                    android.os.Process.THREAD_PRIORITY_BACKGROUND, true /* allowIo */);
            sKillThread.start();
            sKillHandler = new KillHandler(sKillThread.getLooper());
        }

        // 集合管理前台广播和后台广播的集合，有前 / 后台之分，为了区分不同的广播消息超时时间。
        mFgBroadcastQueue = new BroadcastQueue(this, mHandler,
                "foreground", BROADCAST_FG_TIMEOUT, false);
        mBgBroadcastQueue = new BroadcastQueue(this, mHandler,
                "background", BROADCAST_BG_TIMEOUT, true);
        mBroadcastQueues[0] = mFgBroadcastQueue;
        mBroadcastQueues[1] = mBgBroadcastQueue;
  
        mServices = new ActiveServices(this); // 创建管理 Service 组件的 ActiveServices 对象！
        mProviderMap = new ProviderMap(this); // 创建管理 Provider 组件的 ProviderMap 对象！

        mAppErrors = new AppErrors(mContext, this); // 创建管理 ANR 和 crash 的 AppError 对象！

        // 创建 /data/system 目录，诸如包管理packages.xml, packages.list等文件都存放于此目录
        File dataDir = Environment.getDataDirectory();
        File systemDir = new File(dataDir, "system");
        systemDir.mkdirs();
        
        // 创建电量统计服务 BatteryStatsService 对象，并将 mHandler 传入，用于通信！
        mBatteryStatsService = new BatteryStatsService(systemDir, mHandler);
        mBatteryStatsService.getActiveStatistics().readLocked();
        mBatteryStatsService.scheduleWriteToDisk();
        mOnBattery = DEBUG_POWER ? true
                : mBatteryStatsService.getActiveStatistics().getIsOnBattery();
        mBatteryStatsService.getActiveStatistics().setCallback(this);
       
        // 创建进程状态监控服务 ProcessStatsService 对象，/data/system/procstats 文件会存储
        // 进程的状态信息！
        mProcessStats = new ProcessStatsService(this, new File(systemDir, "procstats"));
        
        // 创建应用操作监控服务 AppOpsService 对象，/data/system/appops.xml文件存储app的
        // 权限设置和操作信息！
        mAppOpsService = new AppOpsService(new File(systemDir, "appops.xml"), mHandler);

        // 监听是否允许应用在后台运行的操作
        mAppOpsService.startWatchingMode(AppOpsManager.OP_RUN_IN_BACKGROUND, null,
                new IAppOpsCallback.Stub() {
                    @Override public void opChanged(int op, int uid, String packageName) {
                        if (op == AppOpsManager.OP_RUN_IN_BACKGROUND && packageName != null) {
                            if (mAppOpsService.checkOperation(op, uid, packageName)
                                    != AppOpsManager.MODE_ALLOWED) {
                                runInBackgroundDisabled(uid);
                            }
                        }
                    }
                });
        
        // 创建 urigrants.xml 文件对象！
        mGrantFile = new AtomicFile(new File(systemDir, "urigrants.xml"));
        // 创建 UserController 对象！
        mUserController = new UserController(this);

        // 获取 OpenGL 版本号
        GL_ES_VERSION = SystemProperties.getInt("ro.opengles.version",
            ConfigurationInfo.GL_ES_VERSION_UNDEFINED);

        if (SystemProperties.getInt("sys.use_fifo_ui", 0) != 0) {
            mUseFifoUiScheduling = true;
        }

        mTrackingAssociations = "1".equals(SystemProperties.get("debug.track-associations"));

        // 配置区域、语言、字体、屏幕方向等，启动 Activity 时，需要用到这个配置！
        mConfiguration.setToDefaults();
        mConfiguration.setLocales(LocaleList.getDefault());
        mConfigurationSeq = mConfiguration.seq = 1;
        
        // 初始化 processCpuTracker 对象，用来收集 ANR 情况下的 cpu 使用情况，电池状态等等！
        mProcessCpuTracker.init();

        // AndroidManifest.xml 中 compatible-screens 相关的解析工具
        mCompatModePackages = new CompatModePackages(this, systemDir, mHandler);
        
        // 创建 instent 防火墙！
        mIntentFirewall = new IntentFirewall(new IntentFirewallInterface(), mHandler);
        
        // 创建管理组件 Activity 的 ActivityStackSupervisor 和 ActivityStarter 对象！
        mStackSupervisor = new ActivityStackSupervisor(this);
        mActivityStarter = new ActivityStarter(this, mStackSupervisor);

        // 创建最近任务对象！
        mRecentTasks = new RecentTasks(this, mStackSupervisor);
        
        // 创建一个子线程，不断循环，更新 cpu 的状态信息！
        mProcessCpuThread = new Thread("CpuTracker") {
            @Override
            public void run() {
                while (true) {
                    try {
                        try {
                            synchronized(this) {
                                final long now = SystemClock.uptimeMillis();
                                long nextCpuDelay = (mLastCpuTime.get()+MONITOR_CPU_MAX_TIME)-now;
                                long nextWriteDelay = (mLastWriteTime+BATTERY_STATS_TIME)-now;
                                //Slog.i(TAG, "Cpu delay=" + nextCpuDelay
                                //        + ", write delay=" + nextWriteDelay);
                                if (nextWriteDelay < nextCpuDelay) {
                                    nextCpuDelay = nextWriteDelay;
                                }
                                if (nextCpuDelay > 0) {
                                    mProcessCpuMutexFree.set(true);
                                    this.wait(nextCpuDelay);
                                }
                            }
                        } catch (InterruptedException e) {
                        }
                        // 更新 cpu 的状态以及电池状态信息！
                        updateCpuStatsNow();
                    } catch (Exception e) {
                        Slog.e(TAG, "Unexpected exception collecting process stats", e);
                    }
                }
            }
        };

        // 将自身加入到 Watchdog 的监控中！
        Watchdog.getInstance().addMonitor(this);
        Watchdog.getInstance().addThread(mHandler);
    }

```
我们总结一下，在 AMS 的构造器中，主要做了一下几个事情：

- 创建了系统目录：`/data/system`

- 创建了几个系统服务对象：
    - `BatteryStatsService`：用于监控电池状态；
    - `ProcessStatsService`：用于监控进程状态；
    - `AppOpsService`：用于监控 `app` 的操作行为；
    - `UserController`：

- 创建了四大组件的管理对象；
    - `Broadcast`：`mFgBroadcastQueue` 和 `mBgBroadcastQueue`，分别管理前台后台广播；
    - `Service`：`ActiveServices`；
    - `Provider`：`ProviderMap`；
    - `Activity`：`mStackSupervisor`，`mActivityStarter`；

- 创建了处理 `ANR` 和 `Crash` 的对象：`mAppErrors`；
- 创建最近任务对象：`RecentTasks`；

初次之外，还有：

- 初始化 `processCpuTracker` 对象！
- 创建了几个 `Handler`，用于消息处理：
    - `MainHandler`：绑定了子线程 `ServiceThread` 对象 `mHandlerThread`，`AMS` 会把它传递个给其他服务，用于交互；
    - `UiHandler`：绑定了系统进程的主线程，用于处理 `ANR` 和 `Crash` 相关的信息！
    - `sKillThread`：绑定了子线程 `ServiceThread` 对象 `sKillThread`，用于处理杀进程的操作；
- 创建几个子线程
    - 一个线程 `mProcessCpuThread`，不断的更新 `cpu` 和电池的状态信息；
    - 两个 `ServiceThread` 子线程；


### 1.1.1 new ActivityStackSupervisor

`ActivityStackSupervisor` 是系统中所有的 `ActivityStack` 的管理对象！
```java
    public ActivityStackSupervisor(ActivityManagerService service) {
        mService = service;
        // 创建了一个 ActivityStackSupervisorHandler 的 Handler 对象！
        mHandler = new ActivityStackSupervisorHandler(mService.mHandler.getLooper());
        mActivityMetricsLogger = new ActivityMetricsLogger(this, mService.mContext);
        mResizeDockedStackTimeout = new ResizeDockedStackTimeout(service, this, mHandler);
    }
```

### 1.1.2 new ActivityStarter
`ActivityStarter` 对象负责启动 `activity`：
```java
    ActivityStarter(ActivityManagerService service, ActivityStackSupervisor supervisor) {
        mService = service;
        mSupervisor = supervisor;
        mInterceptor = new ActivityStartInterceptor(mService, mSupervisor);
    }
```

### 1.1.3 new RecentTasks

创建最近任务管理对象！
```java
    RecentTasks(ActivityManagerService service, ActivityStackSupervisor mStackSupervisor) {
        // 对应 /data/system 目录
        File systemDir = Environment.getDataSystemDirectory();
        mService = service;
        // 创建 TaskPersister 对象，用于管理那些重启后可恢复的任务！
        mTaskPersister = new TaskPersister(systemDir, mStackSupervisor, service, this);
        // 将最近任务对象传递给 StackSupervisor！
        mStackSupervisor.setRecentTasks(this);
    }
```
`TaskPersister` 对象用于管理那些重启后可恢复的任务，下面我们去看看：

#### 1.1.3.1 new TaskPersister

用于管理系统中所有的 `persisable` 类型的 `task`：
```java
    TaskPersister(File systemDir, ActivityStackSupervisor stackSupervisor,
            ActivityManagerService service, RecentTasks recentTasks) {

        // 删除目录：/data/system/recent_images 和其子目录！
        final File legacyImagesDir = new File(systemDir, IMAGES_DIRNAME);
        if (legacyImagesDir.exists()) {
            if (!FileUtils.deleteContents(legacyImagesDir) || !legacyImagesDir.delete()) {
                Slog.i(TAG, "Failure deleting legacy images directory: " + legacyImagesDir);
            }
        }
        
        // 删除目录：/data/system/recent_tasks 和其子目录！
        final File legacyTasksDir = new File(systemDir, TASKS_DIRNAME);
        if (legacyTasksDir.exists()) {
            if (!FileUtils.deleteContents(legacyTasksDir) || !legacyTasksDir.delete()) {
                Slog.i(TAG, "Failure deleting legacy tasks directory: " + legacyTasksDir);
            }
        }
        
        // 创建目录 ：/data/system_de，保存了 persisable 任务的 id；
        mTaskIdsDir = new File(Environment.getDataDirectory(), "system_de");
        mStackSupervisor = stackSupervisor;
        mService = service;
        mRecentTasks = recentTasks;

        // 创建一个线程对象，用于写数据！
        mLazyTaskWriterThread = new LazyTaskWriterThread("LazyTaskWriterThread");
    }

```
这里我来简单的说一下：

- 在 `/data/system_de` 目录下，会有一个以设备用户 `id` 命名的文件夹，内部会有一个名为 `persisted_taskIds.txt` 的文件，内部保存了所有的 `persisable` 类型的 `task` 的 `id`；

```java
lishuaiqi:/data/system_de # ls -al
drwxrwx---  3 system system 4096 2017-08-10 08:58 0
lishuaiqi:/data/system_de/0 # ls -al
-rw-------  1 system system   28 2017-08-14 01:30 persisted_taskIds.txt
lishuaiqi:/data/system_de/0 # cat persisted_taskIds.txt
278
279
320
```

- 在 `/data/system_de` 目录下，会有一个以设备用户 `id` 命名的文件夹，内部会有两个文件夹：`recent_images` 和 `recent_tasks`：

```java
lishuaiqi:/data/system_ce/0 # ls -al
drwx------ 2 system system  4096 2017-08-14 01:30 recent_images
drwx------ 2 system system  4096 2017-08-14 01:31 recent_tasks
```
其中，`recent_images` 用来保存 `task` 的截图信息，文件名的格式为：`[task_id]_task_thumbnail.png`
```java
lishuaiqi:/data/system_ce/0/recent_images # ls -al
-rw------- 1 system system  68019 2017-08-14 01:30 326_task_thumbnail.png
-rw------- 1 system system  18394 2017-08-14 01:30 327_task_thumbnail.png
```
其中，`recent_tasks` 用来保存 `task` 的信息，文件名的格式为：`[task_id]_task.xml`
```java
lishuaiqi:/data/system_ce/0/recent_tasks # ls -al
-rw------- 1 system system 1107 2017-08-14 16:58 279_task.xml
-rw------- 1 system system 1116 2017-08-19 19:06 320_task.xml
```

下面我们来看看 `[task_id]_task.xml` 文件中有那些奇葩的属性，这里以腾讯微信为例子：
```java
<task task_id="278" 
	  real_activity="com.tencent.mm/.ui.LauncherUI" 
	  real_activity_suspended="false" 
	  affinity="com.tencent.mm" 
	  root_has_reset="true" 
	  auto_remove_recents="false" 
	  asked_compat_mode="false" 
	  user_id="0" 
	  user_setup_complete="true" 
	  effective_uid="10109" 
	  task_type="0" 
	  first_active_time="1502678072409" 
	  last_active_time="1502678115558" 
	  last_time_moved="1502678078477" 
	  never_relinquish_identity="true" 
	  task_description_color="ff212121" 
	  task_description_colorBackground="fffafafa" 
	  task_thumbnailinfo_task_width="1080" 
	  task_thumbnailinfo_task_height="1920" 
	  task_thumbnailinfo_screen_orientation="1" 
	  task_affiliation_color="-14606047" 
	  task_affiliation="278" 
	  prev_affiliation="-1" 
	  next_affiliation="-1" 
	  calling_uid="10109" 
	  calling_package="com.tencent.mm" 
	  resize_mode="4" 
	  privileged="false" 
	  min_width="-1" 
	  min_height="-1">

	<intent action="android.intent.action.MAIN" 
	        component="com.tencent.mm/.ui.LauncherUI" 
			flags="10200000">
		<categories category="android.intent.category.LAUNCHER" />
	</intent>
</task>
```
可以看到，所有的 `persisable` 类型的 `task`，系统会将这些属性保存到对应的文件中，开机后可以恢复！

## 1.2 onStart
调用构造器，创建完 `AMS` 的对象之后，要做的就是启动 `AMS` 哦，在上一篇中，我们有看到，启动先调用 `LifeCircle` 的 `onStart` ，然后会调用了 `AMS` 的 `start` 方法：
```JAVA
    private void start() {
        // 移除所有的进程组！
        Process.removeAllProcessGroups();

        // 启动 ProcessCpuThread，监控 cpu 和电池的状态！
        mProcessCpuThread.start();
        
        // 将 BatteryStatsService 服务和 AppOpsService 注册到 ServiceManager 中，用于进程间通讯！
        mBatteryStatsService.publish(mContext);
        mAppOpsService.publish(mContext);
        Slog.d("AppOps", "AppOpsService published");
        
        // 将 AMS 添加进入 LocalServices 集合中，用于同一进程内的本地通信！
        LocalServices.addService(ActivityManagerInternal.class, new LocalService());
    }
```
这个方法很简单，就不多说了：

- 移除所有的进程组；并启动 `ProcessCpuThread`，监控 `cpu` 和电池的状态！
- 将 `BatteryStatsService` 服务和 `AppOpsService` 注册到 `ServiceManager` 中，用于 `Binder` 通讯！  

## 1.3 initPowerManagement
接下来，初始化 电池管理：
```JAVA
    public void initPowerManagement() {
        // 初始化电池管理！
        mStackSupervisor.initPowerManagement();
        mBatteryStatsService.initPowerManagement();

        mLocalPowerManager = LocalServices.getService(PowerManagerInternal.class);
        
        PowerManager pm = (PowerManager)mContext.getSystemService(Context.POWER_SERVICE);
        
        // 获得 WakeLock！
        mVoiceWakeLock = pm.newWakeLock(PowerManager.PARTIAL_WAKE_LOCK, "*voice*");
        mVoiceWakeLock.setReferenceCounted(false);
    }
```
这里主要是进一步的初始化电池管理！

## 1.4 setSystemProcess
接下来，是设置 SytemServer 系统进程的相关信息，我们来看看代码，这里要仔细的来看了！
```java
    public void setSystemProcess() {
        try {

            //【1】注册一些系统相关的服务
            ServiceManager.addService(Context.ACTIVITY_SERVICE, this, true);
            ServiceManager.addService(ProcessStats.SERVICE_NAME, mProcessStats);
            ServiceManager.addService("meminfo", new MemBinder(this));
            ServiceManager.addService("gfxinfo", new GraphicsBinder(this));
            ServiceManager.addService("dbinfo", new DbBinder(this));
            if (MONITOR_CPU_USAGE) {
                ServiceManager.addService("cpuinfo", new CpuBinder(this));
            }
            ServiceManager.addService("permission", new PermissionController(this));
            ServiceManager.addService("processinfo", new ProcessInfoService(this));

            // 从 PackageManagerService 中获取 framework-res.apk (包名为 android) 的 ApplicationInfo 信息!
            ApplicationInfo info = mContext.getPackageManager().getApplicationInfo("android", STOCK_PM_FLAGS | MATCH_SYSTEM_ONLY);

            //【2】保存 framework-res.apk 的信息到系统进程的主线程中！
            mSystemThread.installSystemApplicationInfo(info, getClass().getClassLoader());
            
            synchronized (this) {

                // 初始化系统进程的 ProcessRecord
                ProcessRecord app = newProcessRecordLocked(info, info.processName, false, 0);
                app.persistent = true; // 设置为常驻进程
                app.pid = MY_PID; // 设置 pid
                app.maxAdj = ProcessList.SYSTEM_ADJ; // 设置系统进程的 adj 为 -900，很难被杀死哦！
                
                // 使系统进程处于活跃状态
                app.makeActive(mSystemThread.getApplicationThread(), mProcessStats);
                synchronized (mPidsSelfLocked) {
                    // 将 systemServer 进程对象加入到 mPidsSelfLocked 中，以便管理！
                    mPidsSelfLocked.put(app.pid, app);
                }
                
                // 更新最近使用进程列表
                updateLruProcessLocked(app, false, null);
                // 用于更新进程的oom_adj值
                updateOomAdjLocked();
            }
        } catch (PackageManager.NameNotFoundException e) {
            throw new RuntimeException("Unable to find android system package", e);
        }
    }
```
上面代码中用到了 `mPidsSelfLocked` 变量，我们看下这个变量的定义：
```java
final SparseArray<ProcessRecord> mPidsSelfLocked = new SparseArray<ProcessRecord>();
```
`mPidsSelfLocked` 保存有 `pid` 组织起来的正在运行的进程，保存的是 `pid`，及 `pid` 对应的 `ProcessRecord`。

这里我们来总结一下，这方法主要做了什么：

- 注册一些系统相关的服务，我们可以通过 `adb shell dumpsys` 命令查看！
- 获取 `framework-res.apk` 安装包信息，保存到系统进程的 `ActivityThread` 对象中！
- 创建 `SystemServer` 系统进程的进程结构体，初始化属性，将其加入  `mPidsSelfLocked` ，用于管理！
- 更新进程的优先级和 `oomAdj`，这两个方法，我们后面单开一贴！！！

下面我们仔细的看看一些细节，有利于我们对 `AMS` 的结构有一个更深层次的理解！


### 1.4.1 ServiceManager.addService
第一步，注册一些重要的服务，包括 ActivityManagerService 自身！
```java
            // 注册一些系统相关的服务
            ServiceManager.addService(Context.ACTIVITY_SERVICE, this, true);
            ServiceManager.addService(ProcessStats.SERVICE_NAME, mProcessStats);
            ServiceManager.addService("meminfo", new MemBinder(this));
            ServiceManager.addService("gfxinfo", new GraphicsBinder(this));
            ServiceManager.addService("dbinfo", new DbBinder(this));
            if (MONITOR_CPU_USAGE) {
                ServiceManager.addService("cpuinfo", new CpuBinder(this));
            }
            ServiceManager.addService("permission", new PermissionController(this));
            ServiceManager.addService("processinfo", new ProcessInfoService(this));
```
这里注册了哪些服务呢？这里用到了几个静态变量：

> Context.ACTIVITY_SERVICE = "activity";
> ProcessStats.SERVICE_NAME = "procstats";

这里注册了那些服务呢，我们来看看：

| 服务名 |类名 | 功能  |
|---|---| ---- |
|`procstats`	|`ProcessStatsService`	|监控进程信息的服务|
|`cpuinfo`	|`CpuBinder`	|监控CPU信息的服务|
|`meminfo`	|`MemBinder`	|dump系统中每个进程的内存使用状况的服务|
|`gfxinfo`	|`GraphicsBinder`	|dump系统中每个进程使用图形加速卡状态的服务|
|`dbinfo`	|`DbBinder`	|dump 系统中每个进程的db状况的服务|
|`permission`	|`PermissionController`  |是检查Binder调用权限的服务|
|`processinfo`    |`ProcessInfoService` |进程信息|

这些服务我们先不细看，后面遇到了，再具体分析！

### 1.4.2 处理 framework-res.apk

接着，处理 `framework-res.apk` 安装包，在 `PMS` 的启动过程中，`PackageManagerService` 会扫描 `framework-res.apk` 的信息，并将信息封装成 `ApplicaitonInfo` 并保存在一个集合中：
```java
mSystemThread.installSystemApplicationInfo(info, getClass().getClassLoader());
```
`mSystemThread` 对象是 `ActvityThread` 实例，每一个进程都有一个 `ActivityThread`  对象，内部保存着进程的主线程对象，这里保存着 `SystemServer `进程的主线程对象！

我们继续看方法调用：

#### 1.4.2.1 ActivityThread.installSystemApplicaitonInfo
我们进入 `ActivityThread` 的 `installSystemApplicationInfo` 的方法：
```java
    public void installSystemApplicationInfo(ApplicationInfo info, ClassLoader classLoader) {
        synchronized (this) {

            //【1】调用 ContextImpl 对象的 installSystemApplicationInfo 方法！
            getSystemContext().installSystemApplicationInfo(info, classLoader);
            mProfiler = new Profiler();
        }
    }
```
`getSystemContext` 方法获得的是系统进程的 `Context` 对象，`Context` 的具体实现是 `ContextImpl` 类！

#### 1.4.2.2 ContextImpl.installSystemApplicaitonInfo

进入 `ContextImpl` 对象中：
```java
    void installSystemApplicationInfo(ApplicationInfo info, ClassLoader classLoader) {
    
        //【1】调用 LoadedApk.installSystemApplicationInfo 方法；
        mPackageInfo.installSystemApplicationInfo(info, classLoader);
    }
```
其中 `mPackageInfo` 是 `LoadedApk` 对象，这里的 `LoadedApk` 是属于 `SystemServer` 进程的：

#### 1.4.2.3 LoadedAPK.installSystemApplicaitonInfo
最后，将 `framework-res.apk` 的安装信息，保存到 `SystemServer` 进程的主线程的 `LoadedAPK` 对象中！
```java
    void installSystemApplicationInfo(ApplicationInfo info, ClassLoader classLoader) {
        assert info.packageName.equals("android");
        mApplicationInfo = info;
        mClassLoader = classLoader;
    }
```
每一个进程都会有一个 `LoadedApk` 对象，最后 `ApplicationInfo` 和 `ClassLoader` 设置到了 `LoadedApk` 对象中！

### 1.4.3 newProcessRecordLocked

接下来，创建系统进程对应的 `ProcessRecord` 对象，传入参数：

- `info`：`"framework-res.apk"` 的 `ApplicationInfo` 对象！
- `customProcess`：`info.processName`，所在进程名！
- `isolated`：传入 `false`，表示不是隔离进程！
- `isolatedUid`：传入 `0`，表示隔离进程的 `uid`；

`isolated` 和 `isolatedUid` 用来表示这进程是不是一个隔离进程！
```java
    final ProcessRecord newProcessRecordLocked(ApplicationInfo info, String customProcess,
            boolean isolated, int isolatedUid) {
        String proc = customProcess != null ? customProcess : info.processName;
        BatteryStatsImpl stats = mBatteryStatsService.getActiveStatistics();

        final int userId = UserHandle.getUserId(info.uid); // 获得所属的设备用户！
        int uid = info.uid;
        if (isolated) { // isolated 为 false, 不进入这个分支!
            if (isolatedUid == 0) {
                int stepsLeft = Process.LAST_ISOLATED_UID - Process.FIRST_ISOLATED_UID + 1;
                while (true) {
                    if (mNextIsolatedProcessUid < Process.FIRST_ISOLATED_UID
                            || mNextIsolatedProcessUid > Process.LAST_ISOLATED_UID) {
                        mNextIsolatedProcessUid = Process.FIRST_ISOLATED_UID;
                    }
                    uid = UserHandle.getUid(userId, mNextIsolatedProcessUid);
                    mNextIsolatedProcessUid++;
                    if (mIsolatedProcesses.indexOfKey(uid) < 0) {
                        // No process for this uid, use it.
                        break;
                    }
                    stepsLeft--;
                    if (stepsLeft <= 0) {
                        return null;
                    }
                }
            } else {
                // Special case for startIsolatedProcess (internal only), where
                // the uid of the isolated process is specified by the caller.
                uid = isolatedUid;
            }
        }

        //【1】创建 ProcessRecord，用于存储系统进程的信息
        final ProcessRecord r = new ProcessRecord(stats, info, proc, uid);
        if (!mBooted && !mBooting
                && userId == UserHandle.USER_SYSTEM
                && (info.flags & PERSISTENT_MASK) == PERSISTENT_MASK) {
            r.persistent = true;
        }

        //【2】保存系统进程 ProcessRecord 对象到 AMS 的结构体中！
        addProcessNameLocked(r);

        return r;
    }
```
接着，调用 `addProcessNameLocked` 方法：
#### 1.4.3.1 addProcessNameLocked

保存系统进程 `ProcessRecord` 对象到 `AMS` 的结构体中！
```java
    private final void addProcessNameLocked(ProcessRecord proc) {
        //【1】如果已经有了相同的 ProcessRecord 对象，移除旧的！
        ProcessRecord old = removeProcessNameLocked(proc.processName, proc.uid);
        if (old == proc && proc.persistent) {
            Slog.w(TAG, "Re-adding persistent process " + proc);
        } else if (old != null) {
            Slog.wtf(TAG, "Already have existing proc " + old + " when adding " + proc);
        }

        //【2】创建 UidRecord 用来保存活跃的 uid，如果已经存在，就不创建！
        UidRecord uidRec = mActiveUids.get(proc.uid); 

        if (uidRec == null) {
            uidRec = new UidRecord(proc.uid);

            // This is the first appearance of the uid, report it now!
            if (DEBUG_UID_OBSERVERS) Slog.i(TAG_UID_OBSERVERS,
                    "Creating new process uid: " + uidRec);
                    
            // 添加到 mActiveUids 对象！
            mActiveUids.put(proc.uid, uidRec);
            noteUidProcessState(uidRec.uid, uidRec.curProcState);
            enqueueUidChangeLocked(uidRec, -1, UidRecord.CHANGE_ACTIVE);
        }

        proc.uidRecord = uidRec;
        uidRec.numProcs++; // 所属该 uid 的进程计数加 1 （uid 代表一个应用程序）
        
        //【3】非隔离进程保存到 mProcessNames 集合中，隔离进程保存到 mIsolatedProcesses 中！
        mProcessNames.put(proc.processName, proc.uid, proc); 
        if (proc.isolated) {
            mIsolatedProcesses.put(proc.uid, proc);
        }
    }
```


### 1.4.4 ProcessRecord.makeActive
参数传递：

- `thread`：`mSystemThread.getApplicationThread()`，系统进程的 `ApplicationThread` 对象！
- `tracker`：`mProcessStats`，是 `ProcessStatsService` 服务对象，用来监听进程的状态！

</br>

```java
     public void makeActive(IApplicationThread _thread, ProcessStatsService tracker) {
        // 第一次进来，为null！
        if (thread == null) {

            //【1】将旧的 ProcessState 对象设置为非活跃状态！
            final ProcessState origBase = baseProcessTracker; 
            if (origBase != null) {
                origBase.setState(ProcessStats.STATE_NOTHING,
                        tracker.getMemFactorLocked(), SystemClock.uptimeMillis(), pkgList);
                origBase.makeInactive();
            }
            
            // 为系统进程创建新的 ProcessState 监听对象，并设置其为活跃状态！
            // info 为改应用程序所属的应用程序！
            baseProcessTracker = tracker.getProcessStateLocked(info.packageName, uid,
                    info.versionCode, processName);
            baseProcessTracker.makeActive();

            for (int i=0; i<pkgList.size(); i++) {
                ProcessStats.ProcessStateHolder holder = pkgList.valueAt(i);
                if (holder.state != null && holder.state != origBase) {
                    holder.state.makeInactive();
                }
                holder.state = tracker.getProcessStateLocked(pkgList.keyAt(i), uid,
                        info.versionCode, processName);
                if (holder.state != baseProcessTracker) {
                    holder.state.makeActive();
                }
            }
        }
        
        //【2】将系统进程的 ActivityThread 保存到 ProcessRecord.thread，完成引用关系！
        thread = _thread;
    }
```
以上是在 `SystemServer` 启动系统服务的第一阶段做的一些主要的工作，接下来，我们来看第二阶段！

# 2 第二阶段 - startCoreServices
接下来，看看第二阶段的代码段：
```java
    private void startCoreServices() {
        // 启动电池服务,必须是在 lightService 后启动 ；
        mSystemServiceManager.startService(BatteryService.class);

        //【1】启动应用统计服务，用于监控应用的使用频率等信息，并使得 AMS 持有其引用！
        mSystemServiceManager.startService(UsageStatsService.class);
        mActivityManagerService.setUsageStatsManager(
                LocalServices.getService(UsageStatsManagerInternal.class));

        // 启动 WebView 更新服务，这里不关注！
        mWebViewUpdateService = mSystemServiceManager.startService(WebViewUpdateService.class);
    }
```
这里我们看看 `setUsageStatsManager`

## 2.1 setUsageStatsManager
```JAVA
    public void setUsageStatsManager(UsageStatsManagerInternal usageStatsManager) {
        mUsageStatsService = usageStatsManager;
    }
```
这个方法主要的作用是：

- 设置应用统计服务的应用，用于监控应用的使用频率等信息！

# 3 第三阶段 - startOtherServices

这里就要进入开机的最后阶段了，`startOtherServices` 方法非常的长，这里只列举和 `ams` 相关的代码！
```java
    private void startOtherServices() {
        try {

            ... ... ... ...
            
            //【1】安装系统进程的 provider！
            traceBeginAndSlog("InstallSystemProviders");
            mActivityManagerService.installSystemProviders();
            Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);

            ... ... ... ...
            
            // 初始化 watch dog 实例！
            traceBeginAndSlog("InitWatchdog");
            final Watchdog watchdog = Watchdog.getInstance();
            watchdog.init(context, mActivityManagerService);
            Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);
            
            // 启动 InputManagerService 和 WindowManagerService 服务！
            traceBeginAndSlog("StartInputManagerService");
            inputManager = new InputManagerService(context);
            Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);

            traceBeginAndSlog("StartWindowManagerService");
            wm = WindowManagerService.main(context, inputManager,
                    mFactoryTestMode != FactoryTest.FACTORY_TEST_LOW_LEVEL,
                    !mFirstBoot, mOnlyCore);
            ServiceManager.addService(Context.WINDOW_SERVICE, wm);
            ServiceManager.addService(Context.INPUT_SERVICE, inputManager);
            Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);
            
            ... ... ... ...
            
            //【2】将 WMS 的引用实例保存到 ams 中！
            mActivityManagerService.setWindowManager(wm);

            inputManager.setWindowManagerCallbacks(wm.getInputMonitor());
            inputManager.start();

            ... ... ... ...
   
        } catch (RuntimeException e) {
            Slog.e("System", "******************************************");
            Slog.e("System", "************ Failure starting core service", e);
        }

        ... ... ... ...

        try {
            wm.displayReady();
        } catch (Throwable e) {
            reportWtf("making display ready", e);
        }

        ... ... ... ...  

        try {
            wm.systemReady();
        } catch (Throwable e) {
            reportWtf("making Window Manager Service ready", e);
        }
        Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);

        if (safeMode) {
            mActivityManagerService.showSafeModeOverlay();
        }

        // 更新显示相关的配置！
        Configuration config = wm.computeNewConfiguration();
        DisplayMetrics metrics = new DisplayMetrics();
        WindowManager w = (WindowManager)context.getSystemService(Context.WINDOW_SERVICE);
        w.getDefaultDisplay().getMetrics(metrics);
        context.getResources().updateConfiguration(config, metrics);

        // 重置系统主题，因为可能依赖显示配置！
        final Theme systemTheme = context.getTheme();
        if (systemTheme.getChangingConfigurations() != 0) {
            systemTheme.rebase();
        }

        ... ... ... ...

        //【3】进入 AMS 的 systemReady 方法，Runnable 任务中只要是启动一些其他的服务，这里我就省略了！
        mActivityManagerService.systemReady(new Runnable() {
            @Override
            public void run() {
                Slog.i(TAG, "Making services ready");
                ... ... ... ...

                try {
                    // 开始监听 native crash！
                    mActivityManagerService.startObservingNativeCrashes();
                } catch (Throwable e) {
                    reportWtf("observing native crashes", e);
                }
                Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);

                ... ... ... ...
                
                // 启动 watch dog！
                Watchdog.getInstance().start();

                ... ... ... ...
            }
        });
    }

```


## 3.1 AMS.installSystemProviders 

安装系统数据库：
```java
    public final void installSystemProviders() {
    
        List<ProviderInfo> providers;
        synchronized (this) {
        
            //【1】获取系统进程的 ProcessRecord, 之前 setSystemProcess() 时，创建了系统进程对应的 ProcessRecord
            ProcessRecord app = mProcessNames.get("system", Process.SYSTEM_UID);
            
            //【2】生成 ProviderInfo 的列表！
            providers = generateApplicationProvidersLocked(app);

            if (providers != null) {
                for (int i=providers.size()-1; i>=0; i--) {
                    ProviderInfo pi = (ProviderInfo)providers.get(i);
                    if ((pi.applicationInfo.flags&ApplicationInfo.FLAG_SYSTEM) == 0) {
                        Slog.w(TAG, "Not installing system proc provider " + pi.name
                                + ": not system .apk");
                                
                        //【2.1】移除一些非系统的 provider！
                        providers.remove(i);
                    }
                }
            }
        }
        if (providers != null) {

            //【3】调用系统进程的 ActivityThread 的 installSystemProviders 方法，安装系统进程的数据库！
            mSystemThread.installSystemProviders(providers);
        }

        mCoreSettingsObserver = new CoreSettingsObserver(this);
        mFontScaleSettingObserver = new FontScaleSettingObserver();

        //mUsageStatsService.monitorPackages();
    }

```
这里指定了进程名为 `system`，进程 `uid` 为 `Process.SYSTEM_UID`，一般来说，会有以下 `Provider` 会出现在返回结果中：

- `framework-res.apk` 中的 `Provider`, 定义在：`frameworks/base/core/res/AndroidManifest.xml`
- `SettingsProvider.apk` 中的 `Provider`, 定义在：`frameworks/base/packages/SettingsProvider/AndroidManifest.xml`

对于手机厂商，可以对系统 `apk` 进行定制，比如让系统 `apk` 共享系统 `uid`，这样的话，就不止以上两种 `provider` 了！

### 3.1.1 AMS.generateApplicationProvidersLocked

参数传递：

- `ProcessRecord app`：系统进程的 `ProcessRecord` 对象！

```java
    private final List<ProviderInfo> generateApplicationProvidersLocked(ProcessRecord app) {
        List<ProviderInfo> providers = null;
        try {
        
            //【1】通过 PackageManagerService 获得系统进程的 provider！
            providers = AppGlobals.getPackageManager()
                    .queryContentProviders(app.processName, app.uid,
                            STOCK_PM_FLAGS | PackageManager.GET_URI_PERMISSION_PATTERNS
                                    | MATCH_DEBUG_TRIAGED_MISSING)
                    .getList();

        } catch (RemoteException ex) {
        }
        
        if (DEBUG_MU) Slog.v(TAG_MU,
                "generateApplicationProvidersLocked, app.info.uid = " + app.uid);
                
        //【2】将 Provider 和 ProcessRecord 绑定！        
        int userId = app.userId;
        if (providers != null) {
            int N = providers.size();

            //【2.1】确保 ProcessRecord 中的 Provider 映射表的容量！
            app.pubProviders.ensureCapacity(N + app.pubProviders.size());
            
            // 遍历 ProviderInfo 列表
            for (int i=0; i<N; i++) {
                ProviderInfo cpi = (ProviderInfo) providers.get(i);
                // 判断是否是单例模式！
                boolean singleton = isSingleton(cpi.processName, cpi.applicationInfo,
                        cpi.name, cpi.flags);
                        
                // 如果是单例模式的 provider，且其 uid 不是 system，跳过该 provider！
                if (singleton && UserHandle.getUserId(app.uid) != UserHandle.USER_SYSTEM) {
                    providers.remove(i);
                    N--;
                    i--;
                    continue;
                }
  
                //【2.2】如果 mProviderMap 中不存在 ContentProviderRecord 对象，则新建一个！
                ComponentName comp = new ComponentName(cpi.packageName, cpi.name);
                ContentProviderRecord cpr = mProviderMap.getProviderByClass(comp, userId);
    
                if (cpr == null) {
                    cpr = new ContentProviderRecord(this, cpi, app.info, comp, singleton);
                    
                    //【2.3】添加到 AMS.mProciderMap 管理集合中；
                    mProviderMap.putProviderByClass(comp, cpr);
                }

                if (DEBUG_MU) Slog.v(TAG_MU,
                        "generateApplicationProvidersLocked, cpi.uid = " + cpr.uid);
                
                //【2.4】将当前的 provider 保存到系统进程的 ProcessRecord 的 pubProviders 中；
                app.pubProviders.put(cpi.name, cpr);

                if (!cpi.multiprocess || !"android".equals(cpi.packageName)) {
                    app.addPackage(cpi.applicationInfo.packageName, cpi.applicationInfo.versionCode,
                            mProcessStats);

                }

                notifyPackageUse(cpi.applicationInfo.packageName,
                                 PackageManager.NOTIFY_PACKAGE_USE_CONTENT_PROVIDER);
            }
        }
        return providers;
    }

```
我们通过一张类图，简单的了解一下 `AMS` 是如何管理 `ContentProvider` 的：


<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/23/AMS管理provider-20221024234421404.png" alt="AMS管理provider.png-66.2kB" style="zoom:50%;" />


- `AMS` 为每一个 `ContentProvider` 创建了一个 `ContentProviderRecord` 对象，`ContentProviderRecord` 中保存着 `provider` 的配置信息 `providerInfo`；`provider` 所属进程的 `uid`；`provider` 所属应用的 `ApplicaitonInfo` 对象！

- `AMS` 的 `mProviderMap` 对象管理着系统中所有的 `ContentProviderRecord` 对象，其内部保存着 `Authority` 或者 `CompnentName` 到指定 `ContentProviderRecord` 的映射！

- `AMS` 内部还维护着和 `ProcessRecord` 相关的集合，用于管理进程，有前台进程集合，内存常驻进程集合，最近使用的进程等等，当 `AMS` 将一个 `ContentProviderRecord` 与 `ProcessRecord` 关联时，会将它保存到 `ProcessRecord` 内部的一个 `pubProviders` 集合中！


### 3.1.2 ActivityThread.installSystemProviders

接着调用 `systemserver` 进程的 `ActivityThread` 的 `installSystemProvider` 方法：
```java
    public final void installSystemProviders(List<ProviderInfo> providers) {
        if (providers != null) {
    
            //【1】mInitialApplication 是 SystemServer 的进程的 Application 对象！
            installContentProviders(mInitialApplication, providers);
        }
    }
```

这里的 `mInitialApplication` 是系统进程的 `Application` 对象，接着调用了 `installContentProviders` 方法！

### 3.1.3 ActivityThread.installContentProviders

```java
    private void installContentProviders(Context context, List<ProviderInfo> providers) {

        final ArrayList<IActivityManager.ContentProviderHolder> results =
            new ArrayList<IActivityManager.ContentProviderHolder>();
            
        //【1】遍历系统进程的所有 provider，为每一个 provider 创建对应的 ContentProviderHolder 对象！
        for (ProviderInfo cpi : providers) {
            if (DEBUG_PROVIDER) {
                StringBuilder buf = new StringBuilder(128);
                buf.append("Pub ");
                buf.append(cpi.authority);
                buf.append(": ");
                buf.append(cpi.name);
                Log.i(TAG, buf.toString());
            }

            //【1.1】将每一个 ProviderInfo 封装为一个 ContentProviderHolder 对象！
            IActivityManager.ContentProviderHolder cph = installProvider(context, null, cpi,
                    false /*noisy*/, true /*noReleaseNeeded*/, true /*stable*/);
            if (cph != null) {
                cph.noReleaseNeeded = true;

                // 添加到集合中！
                results.add(cph);
            }
        }

        try {

            //【2】调用 AMS 的 publish 方法， 注册 ContentProvider
            ActivityManagerNative.getDefault().publishContentProviders(
                getApplicationThread(), results);
        } catch (RemoteException ex) {
        }
    }

```
所有 `ContentProvider` 的创建都需要经过 `installContentProvider()` 函数，这个地方调用了 `ActivityThread.installProvider` 方法，继续安装 `provider`!

#### 3.1.3.1 ActivityThread.installProvider

让我们来继续看看这部分的代码 `installProvider`，参数传递：

- `Context context`：传入 `context` 对象，这里的 `context` 是 `mInitialApplication` 对象，是系统进程的上下文运行环境；
- `IActivityManager.ContentProviderHolder holder`：传入 **`null`**；
- `ProviderInfo info`：传入 `ProviderInfo` 对象，封装着 `provider` 的信息；
- `boolean noisy`：传入 **`false`**；
- `boolean noReleaseNeeded`：传入 **`true`**；
- `boolean stable`：传入 **`true`**；

接着来看：

```java
    private IActivityManager.ContentProviderHolder installProvider(Context context,
            IActivityManager.ContentProviderHolder holder, ProviderInfo info,
            boolean noisy, boolean noReleaseNeeded, boolean stable) {

        ContentProvider localProvider = null;
        IContentProvider provider;

        if (holder == null || holder.provider == null) { // 进入这个分支!
            if (DEBUG_PROVIDER || noisy) {
                Slog.d(TAG, "Loading provider " + info.authority + ": "
                        + info.name);
            }

            Context c = null;
            ApplicationInfo ai = info.applicationInfo;

            //【1】通过包名，找到和 provider 匹配的 context！
            // 如果当前已有的运行环境 Context 不匹配的话，就会创建一个新的 Context！
            if (context.getPackageName().equals(ai.packageName)) {
                c = context;
            } else if (mInitialApplication != null &&
                    mInitialApplication.getPackageName().equals(ai.packageName)) {
                c = mInitialApplication;
            } else {
                try {
                    c = context.createPackageContext(ai.packageName,
                            Context.CONTEXT_INCLUDE_CODE);
                } catch (PackageManager.NameNotFoundException e) {
                    // Ignore
                }
            }

            if (c == null) {
                Slog.w(TAG, "Unable to get context for package " +
                      ai.packageName +
                      " while loading content provider " +
                      info.name);
                return null;
            }

            //【2】根据包名，通过反射创建新的 ContentProvider 对象！
            try {

                final java.lang.ClassLoader cl = c.getClassLoader();
                localProvider = (ContentProvider)cl.
                    loadClass(info.name).newInstance();
                provider = localProvider.getIContentProvider();

                if (provider == null) {
                    Slog.e(TAG, "Failed to instantiate class " +
                          info.name + " from sourceDir " +
                          info.applicationInfo.sourceDir);
                    return null;
                }

                if (DEBUG_PROVIDER) Slog.v(
                    TAG, "Instantiating local provider " + info.name);
                
                //【2.1】将创建的 provider 和 providerInfo 数据进行绑定；
                // 这个方法对 provider 会做初始化，
                localProvider.attachInfo(c, info);

            } catch (java.lang.Exception e) {
                if (!mInstrumentation.onException(null, e)) {
                    throw new RuntimeException(
                            "Unable to get provider " + info.name
                            + ": " + e.toString(), e);
                }
                return null;
            }

        } else {
            provider = holder.provider;
            if (DEBUG_PROVIDER) Slog.v(TAG, "Installing external provider " + info.authority + ": "
                    + info.name);
        }

        //【3】设置对应的 ContentProviderHolder 对象！
        IActivityManager.ContentProviderHolder retHolder;

        synchronized (mProviderMap) {
            if (DEBUG_PROVIDER) Slog.v(TAG, "Checking to add " + provider
                    + " / " + info.name);

            IBinder jBinder = provider.asBinder();
            if (localProvider != null) {
                ComponentName cname = new ComponentName(info.packageName, info.name);

                // 尝试获得对应的 ProviderClientRecord 对象！
                ProviderClientRecord pr = mLocalProvidersByName.get(cname);
                if (pr != null) {
                    if (DEBUG_PROVIDER) {
                        Slog.v(TAG, "installProvider: lost the race, "
                                + "using existing local provider");
                    }
                    provider = pr.mProvider;
                } else {

                    // 创建一个新的 ContentProviderHolder 对象!
                    holder = new IActivityManager.ContentProviderHolder(info);
                    holder.provider = provider;
                    holder.noReleaseNeeded = true;

                    // 创建一个新的 ProviderClientRecord 对象，并将映射关系保存起来！
                    pr = installProviderAuthoritiesLocked(provider, localProvider, holder);
                    mLocalProviders.put(jBinder, pr);
                    mLocalProvidersByName.put(cname, pr);
                }
                retHolder = pr.mHolder;

            } else {
                ProviderRefCount prc = mProviderRefCountMap.get(jBinder);
                if (prc != null) {
                    if (DEBUG_PROVIDER) {
                        Slog.v(TAG, "installProvider: lost the race, updating ref count");
                    }
                    if (!noReleaseNeeded) {
                        incProviderRefLocked(prc, stable);
                        try {
                            ActivityManagerNative.getDefault().removeContentProvider(
                                    holder.connection, stable);
                        } catch (RemoteException e) {
                            // do nothing content provider object is dead any way
                        }
                    }
                } else {
                    ProviderClientRecord client = installProviderAuthoritiesLocked(
                            provider, localProvider, holder);
                    if (noReleaseNeeded) {
                        prc = new ProviderRefCount(holder, client, 1000, 1000);
                    } else {
                        prc = stable
                                ? new ProviderRefCount(holder, client, 1, 0)
                                : new ProviderRefCount(holder, client, 0, 1);
                    }
                    mProviderRefCountMap.put(jBinder, prc);
                }
                retHolder = prc.holder;
            }
        }

        return retHolder;
    }

```
- 首先，找到 `ProviderInfo` 对应的运行环境 `Context`：
   - 对于 `framework-res.apk` 中定义的 `com.android.server.am.DumpHeapProvider` 而言，重新设置的 `Context` 就是 `mInitialApplication`，所以就直接使用；
   - 对于 `SettingProvider.apk` 中定义的 `SettingsProvider` 而言，它的包名为 `com.android.providers.settings`，不等于 `mInitialApplication` 的包名 `android`，所以，会通过 `Context.createPackageContext()` 函数创建一个新的 `Context` 实例！

- 接着，反射创建 `ContentProvider` 对象，然后，根据反射对象 `ContentProvider`，创建对应的 `ContentProviderHolder` 对象！

#### 3.1.3.2 AactivityManagerN.publishContentProviders

最终会调用 `ActivityManagerService` 中的 `publishContentProviders` 方法：
```java
    public final void publishContentProviders(IApplicationThread caller,
            List<ContentProviderHolder> providers) {
        if (providers == null) {
            return;
        }

        enforceNotIsolatedCaller("publishContentProviders");
        synchronized (this) {

            //【1】获得调用者所在进程的 ProcessRecord 对象！
            final ProcessRecord r = getRecordForAppLocked(caller);
            if (DEBUG_MU) Slog.v(TAG_MU, "ProcessRecord uid = " + r.uid);
            if (r == null) {
                throw new SecurityException(
                        "Unable to find app for caller " + caller
                      + " (pid=" + Binder.getCallingPid()
                      + ") when publishing content providers");
            }

            final long origId = Binder.clearCallingIdentity();

            //【2】遍历传入的 ContentProviderHolder 集合！
            final int N = providers.size();
            for (int i = 0; i < N; i++) {

                ContentProviderHolder src = providers.get(i);
                if (src == null || src.info == null || src.provider == null) {
                    continue;
                }

                // 从 ProcessRecord 中获得其对应的 ContentProviderRecord 对象！
                ContentProviderRecord dst = r.pubProviders.get(src.info.name);

                if (DEBUG_MU) Slog.v(TAG_MU, "ContentProviderRecord uid = " + dst.uid);
                
                if (dst != null) {

                    ComponentName comp = new ComponentName(dst.info.packageName, dst.info.name);

                    // 将其保存到 AMS 的 mProviderMap 中！
                    mProviderMap.putProviderByClass(comp, dst);
                    String names[] = dst.info.authority.split(";");
                    for (int j = 0; j < names.length; j++) {
                        mProviderMap.putProviderByName(names[j], dst);
                    }

                    int launchingCount = mLaunchingProviders.size();
                    int j;
                    boolean wasInLaunchingProviders = false;
                    for (j = 0; j < launchingCount; j++) {
                        if (mLaunchingProviders.get(j) == dst) {
                            // 从正在启动的 provider 列表中删除！
                            mLaunchingProviders.remove(j);
                            wasInLaunchingProviders = true;
                            j--;
                            launchingCount--;
                        }
                    }

                    if (wasInLaunchingProviders) {
                        mHandler.removeMessages(CONTENT_PROVIDER_PUBLISH_TIMEOUT_MSG, r);
                    }

                    synchronized (dst) {
                        dst.provider = src.provider;
                        dst.proc = r;
                        dst.notifyAll();
                    }

                    // 更新 ommAdj 值，用于 LMK！
                    updateOomAdjLocked(r);

                    // 更新 provider 的使用状态！
                    maybeUpdateProviderUsageStatsLocked(r, src.info.packageName,
                            src.info.authority);
                }
            }

            Binder.restoreCallingIdentity(origId);
        }
    }
```
关于 `ContentProvider` 的实现原理，我后续会有对应的博文分析，这里就不多说了！

### 3.1.3 阶段总结

我们总结一下：

- 首先，获得系统进程的 `providerInfo` 列表！
   - 通过 PMS，获得系统进程的 `providerInfo` 列表；
   - 确保系统进程 `ProcessRecord` 有足够的空间存储 `ContentProvider`；
   - 然后，对找到的 `ProviderInfo` 列表进行遍历, 如有需要, 则新建一个 `ContentProviderRecord` 对象, 将其添加到 `AMS.mProviderMap` 中以方便管理；同时, 也需要将其添加到 `PrcoessRecord.mPubProviders` 中。
   - 最后返回 `providerInfo` 列表；

- 将 `provider` 注册到系统进程！

## 3.2 ActivityManagerS.setWindowManager

接着是就是设置 `wms` 的引用对象！
```java
    void setWindowManager(WindowManagerService wm) {
        synchronized (mService) {
            //【1】设置 wms 引用实例
            mWindowManager = wm;
            //【2】获得 dms 的引用实力，并将 ams 注册进入 dms！
            mDisplayManager =
                    (DisplayManager)mService.mContext.getSystemService(Context.DISPLAY_SERVICE);
            mDisplayManager.registerDisplayListener(this, null);
            
            //【3】通过 dms 获得是当前安卓设备所有有效的逻辑显示器列表！
            Display[] displays = mDisplayManager.getDisplays();
            for (int displayNdx = displays.length - 1; displayNdx >= 0; --displayNdx) {
                final int displayId = displays[displayNdx].getDisplayId();
                
                // 为每一个显示器创建一个 ActivityDisplay 对象，存储该显示器的信息；
                ActivityDisplay activityDisplay = new ActivityDisplay(displayId);
                if (activityDisplay.mDisplay == null) {
                    throw new IllegalStateException("Default Display does not exist");
                }
                
                //【3.1】将其保存到 ams 的 mActivityDisplays 集合中，后续会用到！
                mActivityDisplays.put(displayId, activityDisplay);
                calculateDefaultMinimalSizeOfResizeableTasks(activityDisplay);
            }
            
            //【4】初始化 mHomeStack，mFocusedStack 和 mLastFocusedStack 这三个 stack！
            mHomeStack = mFocusedStack = mLastFocusedStack =
                    getStack(HOME_STACK_ID, CREATE_IF_NEEDED, ON_TOP);
            
            //【5】通过本地服务过得 ims 的调用接口；
            mInputManagerInternal = LocalServices.getService(InputManagerInternal.class);
        }
    }
```

这个方法逻辑很简单！这里重点看一下，在系统启动的时候，会初始化 `mFocusedStack` 和 `mLastFocusedStack` 为 `HOME_STACK_ID` 类型的 `stack`，这个是显而易见的，因为开机后，就会进入桌面，此时焦点 `stack` 一定是 `home stack`!


### 3.2.1 ActivityStackSupervisor.getStack

该方法的作用是返回一个可用的 `stack`，`createStaticStackIfNeeded` 表示有必要的话是否创建一个静态 `stack`！

```java
    ActivityStack getStack(int stackId, boolean createStaticStackIfNeeded, boolean createOnTop) {
        //【1】获得 stackId 对应的 ActivityContainer 如果其不为 null，返回其内部的 Stack！
        ActivityContainer activityContainer = mActivityContainers.get(stackId);
        if (activityContainer != null) {
            return activityContainer.mStack;
        }
        
        //【2】如果不能通过 ActivityContainer 来获得想要的 statck 话，就先判断是否需要创建一个 stack
        // 不需要创建一个新的 stack 或者 stack id 不是静态的，那就不能创建，返回一个空的 stack！
        if (!createStaticStackIfNeeded || !StackId.isStaticStack(stackId)) {
            return null;
        }
        
        //【3】创建一个 stack，这里的 Display.DEFAULT_DISPLAY 对应的是默认的显示器 ！
        return createStackOnDisplay(stackId, Display.DEFAULT_DISPLAY, createOnTop);
    }
```

这里我们知道，在刚开机的时候，是没有任何的 `ActivityContainer` 的，所以这里会进入 `createStackOnDisplay` 方法中继续创建！

### 3.2.2 ActivityStackSupervisor.createStackOnDisplay

```java
    ActivityStack createStackOnDisplay(int stackId, int displayId, boolean onTop) {
        //【1】获得 displayId 对应的逻辑显示器的 ActivityDisplay 对象！
        ActivityDisplay activityDisplay = mActivityDisplays.get(displayId);
        if (activityDisplay == null) { // 这里显然不会进入！
            return null;
        }
        
        //【2】根据 stackId 创建了一个 ActivityContainer 对象爱！
        ActivityContainer activityContainer = new ActivityContainer(stackId);

        //【3】将其加入到 mActivityContainers 中！
        mActivityContainers.put(stackId, activityContainer);

        //【4】将 activityContainer 绑定到对应的显示器对象 activityDisplay 中！
        activityContainer.attachToDisplayLocked(activityDisplay, onTop);
        return activityContainer.mStack;
    }
```
这里其实很简单，针对于指定的显示器 `ActivityDisplay`，创建对应的 `ActivityContainer`，然后 `ActivityContainer` 内部再去创建 `ActivityStack`!

#### 3.2.2.1 new ActivityContainer

我们来去看看 `ActivityContainer` 的创建：

```java
    class ActivityContainer extends android.app.IActivityContainer.Stub {
        final static int FORCE_NEW_TASK_FLAGS = FLAG_ACTIVITY_NEW_TASK |
                FLAG_ACTIVITY_MULTIPLE_TASK | Intent.FLAG_ACTIVITY_NO_ANIMATION;

        final int mStackId; // stack 的 id！
        IActivityContainerCallback mCallback = null;
        final ActivityStack mStack; // 对应的 ActivityStack 对象！
        ActivityRecord mParentActivity = null; // 父亲 activity！
        String mIdString; 

        boolean mVisible = true; // 是否是可见的，默认为 true！

        // 其所绑定到的 ActivityDisplay 对象，不为 null 说明绑定到了一个逻辑显示器，
        // 这样该 stack 中的 activity 就可以被显示出来！
        ActivityDisplay mActivityDisplay;

        final static int CONTAINER_STATE_HAS_SURFACE = 0;
        final static int CONTAINER_STATE_NO_SURFACE = 1;
        final static int CONTAINER_STATE_FINISHING = 2;
        
        // container 的状态，取值有 3 种！
        int mContainerState = CONTAINER_STATE_HAS_SURFACE;

        ActivityContainer(int stackId) {
            synchronized (mService) {
                mStackId = stackId;
                //【1】创建一个 ActivityStack
                mStack = new ActivityStack(this, mRecentTasks);
    
                mIdString = "ActivtyContainer{" + mStackId + "}";
                if (DEBUG_STACK) Slog.d(TAG_STACK, "Creating " + this);
            }
        }
        
        ... ... ...   
    
    }
```
我们再去看看 `ActivityStack` 的创建：

##### 3.2.2.2.1 new ActivityStack
```java
    ActivityStack(ActivityStackSupervisor.ActivityContainer activityContainer,
            RecentTasks recentTasks) {
        //【1】设置 ActivityContainer 引用关系；
        mActivityContainer = activityContainer;
        // ActivityContainer 是 ActivityStackSupervisor 内部类，这里是通过 getOuter 获得外部类的实例；
        mStackSupervisor = activityContainer.getOuter();
        mService = mStackSupervisor.mService; // ams 的引用实例；
        //【2】创建 ActivityStackHandler 用于处理 activity 的周期变化；
        mHandler = new ActivityStackHandler(mService.mHandler.getLooper());
        mWindowManager = mService.mWindowManager; // wms 实例，不多说了；
        mStackId = activityContainer.mStackId; // stack id；
        mCurrentUser = mService.mUserController.getCurrentUserIdLocked(); // 当前的设备用户对象；
        mRecentTasks = recentTasks; // 最近任务；

        // 如果 stack id 为 FREEFORM_WORKSPACE_STACK_ID，才会创建对应实例，安卓手机默认是为 null
        mTaskPositioner = mStackId == FREEFORM_WORKSPACE_STACK_ID
                ? new LaunchingTaskPositioner() : null; 
    }
```

这里就不多说了！

#### 3.2.2.2 ActivityContainer.attachToDisplayLocked

接下来就是很关键的一部，就是将 `ActivityContainer` 绑定到指定的 `ActivityDisplay` 对象上！

```java
        void attachToDisplayLocked(ActivityDisplay activityDisplay, boolean onTop) {
            if (DEBUG_STACK) Slog.d(TAG_STACK, "attachToDisplayLocked: " + this
                    + " to display=" + activityDisplay + " onTop=" + onTop);
            //【1】设置 mActivityDisplay 引用关系！
            mActivityDisplay = activityDisplay;
            // 
            mStack.attachDisplay(activityDisplay, onTop);
            activityDisplay.attachActivities(mStack, onTop);
        }
```

##### 3.2.2.2.1 ActivityStack.attachDisplay

```java
    void attachDisplay(ActivityStackSupervisor.ActivityDisplay activityDisplay, boolean onTop) {
        mDisplayId = activityDisplay.mDisplayId;
        mStacks = activityDisplay.mStacks;
        mBounds = mWindowManager.attachStack(mStackId, activityDisplay.mDisplayId, onTop);
        mFullscreen = mBounds == null;
        if (mTaskPositioner != null) {
            mTaskPositioner.setDisplay(activityDisplay.mDisplay);
            mTaskPositioner.configure(mBounds);
        }

        if (mStackId == DOCKED_STACK_ID) {
            // If we created a docked stack we want to resize it so it resizes all other stacks
            // in the system.
            mStackSupervisor.resizeDockedStackLocked(
                    mBounds, null, null, null, null, PRESERVE_WINDOWS);
        }
    }
```

##### 3.2.2.2.2 ActivityDisplay.attachActivities

```java
        void attachActivities(ActivityStack stack, boolean onTop) {
            if (DEBUG_STACK) Slog.v(TAG_STACK,
                    "attachActivities: attaching " + stack + " to displayId=" + mDisplayId
                    + " onTop=" + onTop);
            if (onTop) {
                mStacks.add(stack);
            } else {
                mStacks.add(0, stack);
            }
        }
```


其实，整个过程就是创建容器，建立引用关系的过程，通过分析，我们知道，安卓系统的所有的 `stack` 的创建都是这样的！！

## 3.3 ActivityManagerS.systemReady
`SystemServer` 在启动完所有服务之后，将调用 `AMS` 的 `systemReady()` 方法。这个方法是 `Android` 进入用户交互阶段前最后进行的准备工作。这里就即将进入开机的最后阶段了：

```java
    public void systemReady(final Runnable goingCallback) {
        synchronized(this) {

            //【3.3.1】最开始是不会进入这个分支的，即 mSystemReady 为 false，但是由于 systemReady 会被调用多次
            // 后续 mSystemReady 会被置为 true，这样执行 goingCallback 后就会退出了！
            if (mSystemReady) { 

                if (goingCallback != null) {
                    goingCallback.run();
                }
                return;
            }

            mLocalDeviceIdleController
                    = LocalServices.getService(DeviceIdleController.LocalService.class);

            // 通知 UserController、RecentTask 和 AppOpsService 服务，系统已经准备好了！
            mUserController.onSystemReady();
            mRecentTasks.onSystemReadyLocked();
            mAppOpsService.systemReady();

            //【2】mSystemReady 置为 true，表示系统进程已经准备完毕了！
            mSystemReady = true;
        }

        //【3.3.2】遍历 mPidsSelfLocked，找到已经启动但是没有带有 FLAG_PERSISTENT 标记的非 persistent 应用进程；
        // 然后杀掉它们，目的是在启动 Home 前准备一个干净的环境！
        ArrayList<ProcessRecord> procsToKill = null;
        synchronized(mPidsSelfLocked) {
            for (int i=mPidsSelfLocked.size()-1; i>=0; i--) {
                ProcessRecord proc = mPidsSelfLocked.valueAt(i);

                if (!isAllowedWhileBooting(proc.info)){
                    if (procsToKill == null) {
                        procsToKill = new ArrayList<ProcessRecord>();
                    }
                    procsToKill.add(proc);
                }
            }
        }

        synchronized(this) {
            if (procsToKill != null) {
                for (int i=procsToKill.size()-1; i>=0; i--) {
                    ProcessRecord proc = procsToKill.get(i);
                    Slog.i(TAG, "Removing system update proc: " + proc);
                    removeProcessLocked(proc, true, false, "system update done");
                }
            }

            //【3.1】表示系统进程已经准备完毕，可以启动其他进程了！
            mProcessesReady = true;
        }

        Slog.i(TAG, "System now ready");
        EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_AMS_READY,
            SystemClock.uptimeMillis());

        synchronized(this) {
            if (mFactoryTest == FactoryTest.FACTORY_TEST_LOW_LEVEL) {
               ... ... ... ...// 测试模式，非正常情况，先不看！
            }
        }
        
        //【3.3.3】读取设置信息！
        retrieveSettings();
        final int currentUserId;
        synchronized (this) {

            // 获得当前设备用户 id！
            currentUserId = mUserController.getCurrentUserIdLocked();

            //【3.3.4】从 /data/system/urigrants.xml 文件中读取 Uri 相关的权限信息！
            readGrantedUriPermissionsLocked();
        }

        //【5】执行传入的 callback 回调，主要是启动其他服务等等！
        if (goingCallback != null) goingCallback.run();

        mBatteryStatsService.noteEvent(BatteryStats.HistoryItem.EVENT_USER_RUNNING_START,
                Integer.toString(currentUserId), currentUserId);
        mBatteryStatsService.noteEvent(BatteryStats.HistoryItem.EVENT_USER_FOREGROUND_START,
                Integer.toString(currentUserId), currentUserId);

        // 启动当前的设备用户！
        mSystemServiceManager.startUser(currentUserId);

        synchronized (this) {

            //【3.3.5】启动那些 peristent 应用进程!
            startPersistentApps(PackageManager.MATCH_DIRECT_BOOT_AWARE);

            // Start up initial activity.
            mBooting = true;

            if (UserManager.isSplitSystemUser()) {
                ComponentName cName = new ComponentName(mContext, SystemUserHomeActivity.class);
                try {

                    AppGlobals.getPackageManager().setComponentEnabledSetting(cName,
                            PackageManager.COMPONENT_ENABLED_STATE_ENABLED, 0,
                            UserHandle.USER_SYSTEM);

                } catch (RemoteException e) {
                    throw e.rethrowAsRuntimeException();
                }
            }

            //【3.3.6】启动桌面 activity！
            startHomeActivityLocked(currentUserId, "systemReady");

            try {
                if (AppGlobals.getPackageManager().hasSystemUidErrors()) {
                    Slog.e(TAG, "UIDs on the system are inconsistent, you need to wipe your"
                            + " data partition or your device will be unstable.");
                    mUiHandler.obtainMessage(SHOW_UID_ERROR_UI_MSG).sendToTarget();
                }
            } catch (RemoteException e) {
            }

            if (!Build.isBuildConsistent()) {
                Slog.e(TAG, "Build fingerprint is not consistent, warning user");
                mUiHandler.obtainMessage(SHOW_FINGERPRINT_ERROR_UI_MSG).sendToTarget();
            }

            long ident = Binder.clearCallingIdentity();

            try {
                // 发送 ACTION_USER_STARTED 广播！
                Intent intent = new Intent(Intent.ACTION_USER_STARTED);
                intent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY
                        | Intent.FLAG_RECEIVER_FOREGROUND);
                intent.putExtra(Intent.EXTRA_USER_HANDLE, currentUserId);
                broadcastIntentLocked(null, null, intent,
                        null, null, 0, null, null, null, AppOpsManager.OP_NONE,
                        null, false, false, MY_PID, Process.SYSTEM_UID,
                        currentUserId);
                
                // 发送 ACTION_USER_STARTING 广播！
                intent = new Intent(Intent.ACTION_USER_STARTING);
                intent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY);
                intent.putExtra(Intent.EXTRA_USER_HANDLE, currentUserId);
    
                broadcastIntentLocked(null, null, intent,
                        null, new IIntentReceiver.Stub() {
                            @Override
                            public void performReceive(Intent intent, int resultCode, String data,
                                    Bundle extras, boolean ordered, boolean sticky, int sendingUser)
                                    throws RemoteException {
                            }
                        }, 0, null, null,
                        new String[] {INTERACT_ACROSS_USERS}, AppOpsManager.OP_NONE,
                        null, true, false, MY_PID, Process.SYSTEM_UID, UserHandle.USER_ALL);

            } catch (Throwable t) {
                Slog.wtf(TAG, "Failed sending first user broadcasts", t);
            } finally {
                Binder.restoreCallingIdentity(ident);
            }

            //【8】此时的桌面已经是 top activity，设置桌面 activity 为 resumed 状态！
            mStackSupervisor.resumeFocusedStackTopActivityLocked();
            // 发送设备用户已经切换的广播；
            mUserController.sendUserSwitchBroadcastsLocked(-1, currentUserId);
        }
    }
```

这里的代码逻辑比较复杂，里面有一些关键的点，我们来看看：

### 3.3.1 通知其他服务 - SystemReady
代码如下：
```java
            mUserController.onSystemReady();
            mRecentTasks.onSystemReadyLocked();
            mAppOpsService.systemReady();
            mSystemReady = true;
```
我们一个一个来看：
#### 3.3.1.1 UserController.onSystemReady
```java
    void onSystemReady() {
        updateCurrentProfileIdsLocked();
    }
```
调用了 `updateCurrentProfileIdsLocked`：
```java
    private void updateCurrentProfileIdsLocked() {
        final List<UserInfo> profiles = getUserManager().getProfiles(mCurrentUserId,
                false /* enabledOnly */);
        int[] currentProfileIds = new int[profiles.size()]; // profiles will not be null
        for (int i = 0; i < currentProfileIds.length; i++) {
            currentProfileIds[i] = profiles.get(i).id;
        }
        mCurrentProfileIds = currentProfileIds;

        synchronized (mUserProfileGroupIdsSelfLocked) {
            mUserProfileGroupIdsSelfLocked.clear();
            final List<UserInfo> users = getUserManager().getUsers(false);

            // 保存所有设备用户的信息，用于以后权限校验使用！
            for (int i = 0; i < users.size(); i++) {
                UserInfo user = users.get(i);
                if (user.profileGroupId != UserInfo.NO_PROFILE_GROUP_ID) {
                    // 保存在这里！
                    mUserProfileGroupIdsSelfLocked.put(user.id, user.profileGroupId);
                }
            }
        }
    }

```
这个方法的作用是：`Android` 的许多系统调用都需要检查用户的 `ID`，所以这里调用 `updateCurrnetProfileIdsLocked()` 方法来通过 `UserManagerService` 读取系统保持的 `Profile` 信息，装载系统中已经存在的用户信息。


#### 3.3.1.2 RecentTasks.onSystemReady

代码如下：
```java
    void onSystemReadyLocked() {
        clear();
        mTaskPersister.startPersisting();
    }
```
从 `Android5.0` 开始支持带有 `persistent` 标记的 `task`，这些 `task` 在关机时，信息保存在 `/data/system_ce/recent_tasks` 目录下的 `xxx_task.xml`（`xxx` 表示 `task id`）中，系统重启时，通过这些文件中保存的信息重建 `task`，和 `JobSchedulerService` 很类似哦！

```java
    void startPersisting() {
        if (!mLazyTaskWriterThread.isAlive()) {
            mLazyTaskWriterThread.start();
        }
    }
```
该方法中启动了一个 `mLazyTaskWriterThread` 的线程，恢复那些 `pesistable` 类型的 `task`，这里不多说了！

#### 3.3.1.3 AppOpsService.onSystemReady

这里调用了 `AppOpsService` 的 `systemReady`，代码如下：
```java
    public void systemReady() {
        synchronized (this) {
            boolean changed = false;
            //【1】更新系统中所有的 uid 的状态！
            for (int i = mUidStates.size() - 1; i >= 0; i--) {
                UidState uidState = mUidStates.valueAt(i);
                
                //【1.1】uid 对应的 package 为 null，移除该 uid！
                String[] packageNames = getPackagesForUid(uidState.uid);
                if (ArrayUtils.isEmpty(packageNames)) {
                    uidState.clear();
                    mUidStates.removeAt(i);
                    changed = true;
                    continue;
                }

                ArrayMap<String, Ops> pkgs = uidState.pkgOps;
                if (pkgs == null) {
                    continue;
                }
                 
                //【1.1】如果 uid 的某个 ops 发生变化，移除它！
                Iterator<Ops> it = pkgs.values().iterator();
                while (it.hasNext()) {
                    Ops ops = it.next();
                    int curUid = -1;
                    try {
                        curUid = AppGlobals.getPackageManager().getPackageUid(ops.packageName,
                                PackageManager.MATCH_UNINSTALLED_PACKAGES,
                                UserHandle.getUserId(ops.uidState.uid));
                    } catch (RemoteException ignored) {
                    }
                    if (curUid != ops.uidState.uid) {
                        Slog.i(TAG, "Pruning old package " + ops.packageName
                                + "/" + ops.uidState + ": new uid=" + curUid);
                        it.remove();
                        changed = true;
                    }
                }

                if (uidState.isDefault()) {
                    mUidStates.removeAt(i);
                }
            }
            //【1.3】如果 uid 的状态发生变化，将其持久化到本地文件中！
            if (changed) {
                scheduleFastWriteLocked();
            }
        }
        
        //【2】获得挂载服务，设置外置存储挂载策略！
        MountServiceInternal mountServiceInternal = LocalServices.getService(
                MountServiceInternal.class);
        mountServiceInternal.addExternalStoragePolicy(
                new MountServiceInternal.ExternalStorageMountPolicy() {
                    @Override
                    public int getMountMode(int uid, String packageName) {
                        if (Process.isIsolated(uid)) {
                            return Zygote.MOUNT_EXTERNAL_NONE;
                        }
                        if (noteOperation(AppOpsManager.OP_READ_EXTERNAL_STORAGE, uid,
                                packageName) != AppOpsManager.MODE_ALLOWED) {
                            return Zygote.MOUNT_EXTERNAL_NONE;
                        }
                        if (noteOperation(AppOpsManager.OP_WRITE_EXTERNAL_STORAGE, uid,
                                packageName) != AppOpsManager.MODE_ALLOWED) {
                            return Zygote.MOUNT_EXTERNAL_READ;
                        }
                        return Zygote.MOUNT_EXTERNAL_WRITE;
                    }

                    @Override
                    public boolean hasExternalStorage(int uid, String packageName) {
                        final int mountMode = getMountMode(uid, packageName);
                        return mountMode == Zygote.MOUNT_EXTERNAL_READ
                                || mountMode == Zygote.MOUNT_EXTERNAL_WRITE;
                    }
                });
    }
```
这个 `AppOpsService` 是谷歌原生的应用程序权限管理服务，在 `Android M ( Android 6.0 )` 加入的 `Application Permission Manager` 的功能就是基于 `AppOps` 实现的！

`AppOps` 全称是 `Application Operations`，类似我们平时常说的应用程序的操作（权限）管理。目前 `Google` 在每次版本更新时都会隐藏掉 `AppOps` 的入口。

注意：`AppOps` 虽然涵盖了 `App` 的权限管理，但是 `Google` 原生的设计并不仅仅是对“权限”的管理，而是对 `App` 的“动作”的管理。我们平时讲的权限管理多是针对具体的权限（`App` 开发者在 `Manifest` 里申请的权限），而 `AppOps` 所管理的是所有可能涉及用户隐私和安全的操作，包括 `access notification`, `keep weak lock`, `activate vpn`, `display toast` 等等，有些操作是不需要 `Manifest` 里申请权限的。

不多说了，这不多关注！

### 3.3.2 杀掉特定进程

刚刚有讲到，`AMS` 会杀掉已经启动并且没有带有 `FLAG_PERSISTENT` 标记的进程，那如何判断是否具有这个标记呢：

#### 3.3.2.1 isAllowedWhileBooting
```java
    boolean isAllowedWhileBooting(ApplicationInfo ai) {
        return (ai.flags&ApplicationInfo.FLAG_PERSISTENT) != 0;
    }

```
如果某个进程没有 `FLAG_PERSISTENT` ，这个方法会返回 `false`！

### 3.3.3 读取设置信息 - retrieveSettings

该方法的目的是从前面已经配置好的系统 `provider` 中互动一些设置属性！
```java
    private void retrieveSettings() {
        final ContentResolver resolver = mContext.getContentResolver();

        //【1】获得一些系统属性！
        final boolean freeformWindowManagement =
                mContext.getPackageManager().hasSystemFeature(FEATURE_FREEFORM_WINDOW_MANAGEMENT)
                        || Settings.Global.getInt(
                                resolver, DEVELOPMENT_ENABLE_FREEFORM_WINDOWS_SUPPORT, 0) != 0;
        // 是否支持画中画模式！
        final boolean supportsPictureInPicture =
                mContext.getPackageManager().hasSystemFeature(FEATURE_PICTURE_IN_PICTURE);

        // 是否支持分屏模式！
        final boolean supportsMultiWindow = ActivityManager.supportsMultiWindow();
        final String debugApp = Settings.Global.getString(resolver, DEBUG_APP);
        final boolean waitForDebugger = Settings.Global.getInt(resolver, WAIT_FOR_DEBUGGER, 0) != 0;

        final boolean alwaysFinishActivities =
                Settings.Global.getInt(resolver, ALWAYS_FINISH_ACTIVITIES, 0) != 0;
        final boolean lenientBackgroundCheck =
                Settings.Global.getInt(resolver, LENIENT_BACKGROUND_CHECK, 0) != 0;
        // 是否强制 RTL 显示！
        final boolean forceRtl = Settings.Global.getInt(resolver, DEVELOPMENT_FORCE_RTL, 0) != 0;
        // 是否强制 activity 显示尺寸可变化！
        final boolean forceResizable = Settings.Global.getInt(
                resolver, DEVELOPMENT_FORCE_RESIZABLE_ACTIVITIES, 0) != 0;
        final boolean supportsLeanbackOnly =
                mContext.getPackageManager().hasSystemFeature(FEATURE_LEANBACK_ONLY);

        SystemProperties.set(DEVELOPMENT_FORCE_RTL, forceRtl ? "1":"0");

        final Configuration configuration = new Configuration();
        Settings.System.getConfiguration(resolver, configuration);
        if (forceRtl) {
            // This will take care of setting the correct layout direction flags
            configuration.setLayoutDirection(configuration.locale);
        }

        //【2】根据获得的系统属性，初始化系统属性变量！
        synchronized (this) {
            mDebugApp = mOrigDebugApp = debugApp;
            mWaitForDebugger = mOrigWaitForDebugger = waitForDebugger;
            mAlwaysFinishActivities = alwaysFinishActivities;
            mLenientBackgroundCheck = lenientBackgroundCheck;
            mSupportsLeanbackOnly = supportsLeanbackOnly;
            mForceResizableActivities = forceResizable;
            mWindowManager.setForceResizableTasks(mForceResizableActivities);

            if (supportsMultiWindow || forceResizable) {
                mSupportsMultiWindow = true;
                mSupportsFreeformWindowManagement = freeformWindowManagement || forceResizable;
                mSupportsPictureInPicture = supportsPictureInPicture || forceResizable;
            } else {
                mSupportsMultiWindow = false;
                mSupportsFreeformWindowManagement = false;
                mSupportsPictureInPicture = false;
            }

            // This happens before any activities are started, so we can
            // change mConfiguration in-place.
            updateConfigurationLocked(configuration, null, true);
            if (DEBUG_CONFIGURATION) Slog.v(TAG_CONFIGURATION,
                    "Initial config: " + mConfiguration);

            // Load resources only after the current configuration has been set.
            final Resources res = mContext.getResources();

            // 是否有最近任务！
            mHasRecents = res.getBoolean(com.android.internal.R.bool.config_hasRecents);
            mThumbnailWidth = res.getDimensionPixelSize(
                    com.android.internal.R.dimen.thumbnail_width);
            mThumbnailHeight = res.getDimensionPixelSize(
                    com.android.internal.R.dimen.thumbnail_height);
            mDefaultPinnedStackBounds = Rect.unflattenFromString(res.getString(
                    com.android.internal.R.string.config_defaultPictureInPictureBounds));
                    
            mAppErrors.loadAppsNotReportingCrashesFromConfigLocked(res.getString(
                    com.android.internal.R.string.config_appsNotReportingCrashes));
            if ((mConfiguration.uiMode & UI_MODE_TYPE_TELEVISION) == UI_MODE_TYPE_TELEVISION) {
                mFullscreenThumbnailScale = (float) res
                    .getInteger(com.android.internal.R.integer.thumbnail_width_tv) /
                    (float) mConfiguration.screenWidthDp;
            } else {
                mFullscreenThumbnailScale = res.getFraction(
                    com.android.internal.R.fraction.thumbnail_fullscreen_scale, 1, 1);
            }
        }
    }
```
这里不多说了！

### 3.3.4 读取 Uri 权限信息


#### 3.3.4.1 读取 Uri 权限信息
```java
    private void readGrantedUriPermissionsLocked() {
        if (DEBUG_URI_PERMISSION) Slog.v(TAG_URI_PERMISSION, "readGrantedUriPermissions()");

        final long now = System.currentTimeMillis();

        FileInputStream fis = null;
        try {
            //【1】读取 /data/system/urigrants.xml 文件！
            fis = mGrantFile.openRead();
            final XmlPullParser in = Xml.newPullParser();
            in.setInput(fis, StandardCharsets.UTF_8.name());

            int type;
            while ((type = in.next()) != END_DOCUMENT) {
                final String tag = in.getName();
                if (type == START_TAG) {
                    if (TAG_URI_GRANT.equals(tag)) { // uri-grants 标签
                        final int sourceUserId;
                        final int targetUserId;
                        final int userHandle = readIntAttribute(in,
                                ATTR_USER_HANDLE, UserHandle.USER_NULL); // userHandle 属性
                        if (userHandle != UserHandle.USER_NULL) {
                            // For backwards compatibility.
                            sourceUserId = userHandle;
                            targetUserId = userHandle;
                        } else {
                            sourceUserId = readIntAttribute(in, ATTR_SOURCE_USER_ID); // sourceUserId 属性
                            targetUserId = readIntAttribute(in, ATTR_TARGET_USER_ID); // targetUserId 属性
                        }
                        final String sourcePkg = in.getAttributeValue(null, ATTR_SOURCE_PKG); // sourcePkg 属性
                        final String targetPkg = in.getAttributeValue(null, ATTR_TARGET_PKG); // targetPkg 属性
                        final Uri uri = Uri.parse(in.getAttributeValue(null, ATTR_URI)); // uri 属性
                        final boolean prefix = readBooleanAttribute(in, ATTR_PREFIX); // prefix 属性
                        final int modeFlags = readIntAttribute(in, ATTR_MODE_FLAGS); // modeFlags 属性
                        final long createdTime = readLongAttribute(in, ATTR_CREATED_TIME, now); // createdTime 属性

                        // Sanity check that provider still belongs to source package
                        // Both direct boot aware and unaware packages are fine as we
                        // will do filtering at query time to avoid multiple parsing.
                        // 根据 uri 获得其所属的 ContentProvider 对象！
                        final ProviderInfo pi = getProviderInfoLocked(
                                uri.getAuthority(), sourceUserId, MATCH_DIRECT_BOOT_AWARE
                                        | MATCH_DIRECT_BOOT_UNAWARE);
                        if (pi != null && sourcePkg.equals(pi.packageName)) {
                            int targetUid = -1;
                            try {
                                // 获得持有该 uri 权限的目标ingyon
                                targetUid = AppGlobals.getPackageManager().getPackageUid(
                                        targetPkg, MATCH_UNINSTALLED_PACKAGES, targetUserId);
                            } catch (RemoteException e) {
                            }
                            // 创建 UriPermission，封装该应用的 uri 权限！
                            if (targetUid != -1) {
                                final UriPermission perm = findOrCreateUriPermissionLocked(
                                        sourcePkg, targetPkg, targetUid,
                                        new GrantUri(sourceUserId, uri, prefix));
                                perm.initPersistedModes(modeFlags, createdTime);
                            }
                        } else {
                            Slog.w(TAG, "Persisted grant for " + uri + " had source " + sourcePkg
                                    + " but instead found " + pi);
                        }
                    }
                }
            }
        } catch (FileNotFoundException e) {
            // Missing grants is okay
        } catch (IOException e) {
            Slog.wtf(TAG, "Failed reading Uri grants", e);
        } catch (XmlPullParserException e) {
            Slog.wtf(TAG, "Failed reading Uri grants", e);
        } finally {
            IoUtils.closeQuietly(fis);
        }
    }

```

#### 3.3.4.2 ActivityManagerS.findOrCreateUriPermissionLocked
```java
    private UriPermission findOrCreateUriPermissionLocked(String sourcePkg,
            String targetPkg, int targetUid, GrantUri grantUri) {
        ArrayMap<GrantUri, UriPermission> targetUris = mGrantedUriPermissions.get(targetUid);
        if (targetUris == null) {
            targetUris = Maps.newArrayMap();
            mGrantedUriPermissions.put(targetUid, targetUris);
        }

        UriPermission perm = targetUris.get(grantUri);
        if (perm == null) {
            // 创建一个 UriPermission 对象！
            perm = new UriPermission(sourcePkg, targetPkg, targetUid, grantUri);
            targetUris.put(grantUri, perm);
        }

        return perm;
    }
```

mGrantedUriPermissions 用来保存系统中所有已经授予的 uri 权限！

下面去看看 UriPermission 对象！

```java
    UriPermission(String sourcePkg, String targetPkg, int targetUid, GrantUri uri) {
        this.targetUserId = UserHandle.getUserId(targetUid);
        this.sourcePkg = sourcePkg;
        this.targetPkg = targetPkg;
        this.targetUid = targetUid;
        this.uri = uri;
    }
```
接着调用了 initPersistedModes 初始化 persisted 相关属性，包括 persistableModeFlags，persistedModeFlags 和 persistedCreateTime！
```java

    void initPersistedModes(int modeFlags, long createdTime) {
        // 初始化 persistableModeFlags 和 persistedModeFlags 
        // 只包含 FLAG_GRANT_READ_URI_PERMISSION 和 FLAG_GRANT_WRITE_URI_PERMISSION！
        modeFlags &= (Intent.FLAG_GRANT_READ_URI_PERMISSION
                | Intent.FLAG_GRANT_WRITE_URI_PERMISSION);

        persistableModeFlags = modeFlags;
        persistedModeFlags = modeFlags;
        persistedCreateTime = createdTime;

        updateModeFlags(); // 更新 modeFlags!
    }
    
    private void updateModeFlags() {
        final int oldModeFlags = modeFlags;
        modeFlags = ownedModeFlags | globalModeFlags | persistableModeFlags | persistedModeFlags;

        if (Log.isLoggable(TAG, Log.VERBOSE) && (modeFlags != oldModeFlags)) {
            Slog.d(TAG,
                    "Permission for " + targetPkg + " to " + uri + " is changing from 0x"
                            + Integer.toHexString(oldModeFlags) + " to 0x"
                            + Integer.toHexString(modeFlags),
                    new Throwable());
        }
    }
```
关于 UriPermission 我们这里只涉及这么多！


### 3.3.5 启动 persistent 进程 - startPersistentApps

```java
    private void startPersistentApps(int matchFlags) {
        if (mFactoryTest == FactoryTest.FACTORY_TEST_LOW_LEVEL) return;

        synchronized (this) {
            try {
                //【1】获得 persistent 类型的 app 的信息！
                final List<ApplicationInfo> apps = AppGlobals.getPackageManager()
                        .getPersistentApplications(STOCK_PM_FLAGS | matchFlags).getList();
                for (ApplicationInfo app : apps) {
                    if (!"android".equals(app.packageName)) {

                        //【2】启动对应进程！
                        addAppLocked(app, false, null /* ABI override */);
                    }
                }
            } catch (RemoteException ex) {
            }
        }
    }
```
启动进程调用了 `addAppLocked`，我们继续看！

#### 3.3.5.1 ActivityManagerS.addAppLocked
```java
    final ProcessRecord addAppLocked(ApplicationInfo info, boolean isolated,
            String abiOverride) {
        //【1】创建 persistent 进程对应的 ProcessRecord 对象！
        ProcessRecord app;
        if (!isolated) {
            app = getProcessRecordLocked(info.processName, info.uid, true);
        } else {
            app = null;
        }

        if (app == null) {
            app = newProcessRecordLocked(info, null, isolated, 0);
            updateLruProcessLocked(app, false, null);
            updateOomAdjLocked();
        }

        //【2】要被启动的 package 不能处于 true 状态！
        try {
            AppGlobals.getPackageManager().setPackageStoppedState(
                    info.packageName, false, UserHandle.getUserId(app.uid));
        } catch (RemoteException e) {
        } catch (IllegalArgumentException e) {
            Slog.w(TAG, "Failed trying to unstop package "
                    + info.packageName + ": " + e);
        }
        
        //【3】设置进程的 persistent 和 maxAdj 属性！
        if ((info.flags & PERSISTENT_MASK) == PERSISTENT_MASK) {
            app.persistent = true;
            app.maxAdj = ProcessList.PERSISTENT_PROC_ADJ;
        }
        
        //【4】保存到 mPersistentStartingProcesses 类型的集合中进行管理，并启动进程！
        if (app.thread == null && mPersistentStartingProcesses.indexOf(app) < 0) {
            mPersistentStartingProcesses.add(app);
            startProcessLocked(app, "added application", app.processName, abiOverride,
                    null /* entryPoint */, null /* entryPointArgs */);
        }

        return app;
    }
```

关于进程的启动，请看我的另一系列的博文，这里就不多说了！

### 3.3.6 启动桌面进程 

接着是启动桌面所在的进程：

#### 3.3.6.1 startHomeActivityLocked 

```java
    boolean startHomeActivityLocked(int userId, String reason) {
        if (mFactoryTest == FactoryTest.FACTORY_TEST_LOW_LEVEL
                && mTopAction == null) {
            return false;
        }
        //【1】创建启动桌面的 Intent！
        Intent intent = getHomeIntent();

        //【2】获得桌面应用对应的 ActivityInfo 实例！
        ActivityInfo aInfo = resolveActivityInfo(intent, STOCK_PM_FLAGS, userId);
        if (aInfo != null) {
            intent.setComponent(new ComponentName(aInfo.applicationInfo.packageName, aInfo.name));
            aInfo = new ActivityInfo(aInfo);
            aInfo.applicationInfo = getAppInfoForUser(aInfo.applicationInfo, userId);

            //【2.1】获得桌面的进程 ProcessRecord，因为 ams 的初始化在开机，所以 app 一定为 null！
            ProcessRecord app = getProcessRecordLocked(aInfo.processName,
                    aInfo.applicationInfo.uid, true);

            if (app == null || app.instrumentationClass == null) {
                intent.setFlags(intent.getFlags() | Intent.FLAG_ACTIVITY_NEW_TASK);

                //【2.2】启动桌面！
                mActivityStarter.startHomeActivityLocked(intent, aInfo, reason);
            }
        } else {
            Slog.wtf(TAG, "No home screen found for " + intent, new Throwable());
        }

        return true;
    }
```

##### 3.3.6.1.1 ActivityManagerS.getHomeIntent
```java
    Intent getHomeIntent() {
        //【1】这里的 mTopAction 等于 Intent.ACTION_MAIN;
        Intent intent = new Intent(mTopAction, mTopData != null ? Uri.parse(mTopData) : null);
        intent.setComponent(mTopComponent);
        intent.addFlags(Intent.FLAG_DEBUG_TRIAGED_MISSING);

        if (mFactoryTest != FactoryTest.FACTORY_TEST_LOW_LEVEL) {
            intent.addCategory(Intent.CATEGORY_HOME);
        }
        return intent;
    }
```

`startHomeActivityLocked` 方法最后会来自桌面 `activity` 的 `onCreate` 方法，然后，`resumeFocusedStackTopActivityLocked` 又会拉起桌面 `activity` 的 `onResume` 方法：

```java
mStackSupervisor.resumeFocusedStackTopActivityLocked();
```
对于 `activity` 的启动，这里我就不多讲了，请看我的其他博文！

到这里，桌面就完全显示在用户面前的，然而，还没有结束，我们进入到 `ActivityThread` 中去：


#### 3.3.6.2 ActivityThread.handleResumeActivity

`handleResumeActivity` 负责拉起桌面 `activity` 的 `onResume` 方法，我们过滤掉一些不重要的代码：

```java
    final void handleResumeActivity(IBinder token,
            boolean clearHide, boolean isForward, boolean reallyResume, int seq, String reason) {
        ... ... ... ...

        // 拉起桌面 activity 的 onResume 方法！
        r = performResumeActivity(token, clearHide, reason);

        if (r != null) {
            final Activity a = r.activity;

            ... ... ... ...

            if (!r.onlyLocalRequest) {
                r.nextIdle = mNewActivities;
                mNewActivities = r;

                // 创建了一个 Idler 对象，加入到了主线程的 Looper 队列中！
                Looper.myQueue().addIdleHandler(new Idler());
            }
            r.onlyLocalRequest = false;

            if (reallyResume) {
                try {
                    ActivityManagerNative.getDefault().activityResumed(token);
                } catch (RemoteException ex) {
                    throw ex.rethrowFromSystemServer();
                }
            }

        } else {
            ... ... ... ...
            // 异常状态，不处理！
        }
    }
```
当桌面进程的主线程进入空闲状态时，`Idler.queueIdle` 方法会被执行：

```java
    private class Idler implements MessageQueue.IdleHandler {
        @Override
        public final boolean queueIdle() {
            ActivityClientRecord a = mNewActivities;
            boolean stopProfiling = false;
            if (mBoundApplication != null && mProfiler.profileFd != null
                    && mProfiler.autoStopProfiler) {
                stopProfiling = true;
            }
            if (a != null) {
                mNewActivities = null;
                IActivityManager am = ActivityManagerNative.getDefault();
                ActivityClientRecord prev;
                do {
                    if (localLOGV) Slog.v(
                        TAG, "Reporting idle of " + a +
                        " finished=" +
                        (a.activity != null && a.activity.mFinished));
                    if (a.activity != null && !a.activity.mFinished) {
                        try {
                            //【1】调用 ams 的 activityIdle 方法！
                            am.activityIdle(a.token, a.createdConfig, stopProfiling);
                            a.createdConfig = null;
                        } catch (RemoteException ex) {
                            throw ex.rethrowFromSystemServer();
                        }
                    }
                    prev = a;
                    a = a.nextIdle;
                    prev.nextIdle = null;
                } while (a != null);
            }
            if (stopProfiling) {
                mProfiler.stopProfiling();
            }
            ensureJitEnabled();
            return false;
        }
    }
```
下面，进入 `ActivityManagerService` 中去：

#### 3.3.6.3 ActivityManagerS.activityIdle

```java
    @Override
    public final void activityIdle(IBinder token, Configuration config, boolean stopProfiling) {
        final long origId = Binder.clearCallingIdentity();
        synchronized (this) {
            ActivityStack stack = ActivityRecord.getStackLocked(token);
            if (stack != null) {
                //【1】继续调用！
                ActivityRecord r =
                        mStackSupervisor.activityIdleInternalLocked(token, false, config);
                if (stopProfiling) {
                    if ((mProfileProc == r.app) && (mProfileFd != null)) {
                        try {
                            mProfileFd.close();
                        } catch (IOException e) {
                        }
                        clearProfilerLocked();
                    }
                }
            }
        }
        Binder.restoreCallingIdentity(origId);
    }
```
#### 3.3.6.4 ActivityManagerS.activityIdleInternalLocked

这里我们只关注和 `AMS`启动有关的代码！

```java
    final ActivityRecord activityIdleInternalLocked(final IBinder token, boolean fromTimeout,
            Configuration config) {
        if (DEBUG_ALL) Slog.v(TAG, "Activity idle: " + token);

        ArrayList<ActivityRecord> finishes = null;
        ArrayList<UserState> startingUsers = null;
        int NS = 0;
        int NF = 0;
        boolean booting = false;
        boolean activityRemoved = false;

        ActivityRecord r = ActivityRecord.forTokenLocked(token);
        if (r != null) {
            if (DEBUG_IDLE) Slog.d(TAG_IDLE, "activityIdleInternalLocked: Callers="
                    + Debug.getCallers(4));
            mHandler.removeMessages(IDLE_TIMEOUT_MSG, r);
            r.finishLaunchTickingLocked();
            if (fromTimeout) {
                reportActivityLaunchedLocked(fromTimeout, r, -1, -1);
            }

            if (config != null) {
                r.configuration = config;
            }

            r.idle = true;
            
            // 显然，桌面 activity 所在的 stack 就是焦点栈！
            if (isFocusedStack(r.task.stack) || fromTimeout) {
                //【1】继续调用！
                booting = checkFinishBootingLocked();
            }
        }

        ... ... ... ...

        return r;
    }
```

#### 3.3.6.5 ActivityStackS.checkFinishBootingLocked

```java
    private boolean checkFinishBootingLocked() {
        // 此时 mService.mBooting 为 true；
        final boolean booting = mService.mBooting;
        boolean enableScreen = false;
        mService.mBooting = false;
        if (!mService.mBooted) {
            mService.mBooted = true;
            enableScreen = true;
        }
        if (booting || enableScreen) {
            //【1】进入这里！
            mService.postFinishBooting(booting, enableScreen);
        }
        return booting;
    }
```
`postFinishBooting` 会通过 `MainHandler` 发送 `FINISH_BOOTING_MSG` 消息给子线程，子线程会调用 `AMS.finishBooting` 的方法！

#### 3.3.6.6 ActivityManagerS.finishBooting

该方法就是最关键的地方了，我们的开机广播就是在这个地方发送的！！！
```java
    final void finishBooting() {
        synchronized (this) {
            //【1】如果开机动画没有显示完，则退出，等开机动画显示完后才会调用！
            if (!mBootAnimationComplete) {
                mCallFinishBooting = true;
                return;
            }
            mCallFinishBooting = false;
        }

        // 检查系统 ABI，ABI 和系统 cpu 指令集有关！
        ArraySet<String> completedIsas = new ArraySet<String>();
        for (String abi : Build.SUPPORTED_ABIS) {
            Process.establishZygoteConnectionForAbi(abi);
            final String instructionSet = VMRuntime.getInstructionSet(abi);
            if (!completedIsas.contains(instructionSet)) {
                try {
                    mInstaller.markBootComplete(VMRuntime.getInstructionSet(abi));
                } catch (InstallerException e) {
                    Slog.w(TAG, "Unable to mark boot complete for abi: " + abi + " (" +
                            e.getMessage() +")");
                }
                completedIsas.add(instructionSet);
            }
        }
        
        // 注册接收者，监听重启 package 的广播，当系统发送了该广播后，该广播会传递所有需要重启的 package
        // 这里监听到该广播后，会重启所有的 package 的进程！
        IntentFilter pkgFilter = new IntentFilter();
        pkgFilter.addAction(Intent.ACTION_QUERY_PACKAGE_RESTART);
        pkgFilter.addDataScheme("package");
        mContext.registerReceiver(new BroadcastReceiver() {
            @Override
            public void onReceive(Context context, Intent intent) {
                String[] pkgs = intent.getStringArrayExtra(Intent.EXTRA_PACKAGES);
                if (pkgs != null) {
                    for (String pkg : pkgs) {
                        synchronized (ActivityManagerService.this) {
                            if (forceStopPackageLocked(pkg, -1, false, false, false, false, false,
                                    0, "query restart")) {
                                setResultCode(Activity.RESULT_OK);
                                return;
                            }
                        }
                    }
                }
            }
        }, pkgFilter);

        // 注册接收者，监听 ACTION_DELETE_DUMPHEAP 广播！
        IntentFilter dumpheapFilter = new IntentFilter();
        dumpheapFilter.addAction(DumpHeapActivity.ACTION_DELETE_DUMPHEAP);
        mContext.registerReceiver(new BroadcastReceiver() {

            @Override
            public void onReceive(Context context, Intent intent) {
                if (intent.getBooleanExtra(DumpHeapActivity.EXTRA_DELAY_DELETE, false)) {
                    mHandler.sendEmptyMessageDelayed(POST_DUMP_HEAP_NOTIFICATION_MSG, 5*60*1000);
                } else {
                    mHandler.sendEmptyMessage(POST_DUMP_HEAP_NOTIFICATION_MSG);
                }
            }
        }, dumpheapFilter);

        // 通知系统服务已经进入最后阶段：PHASE_BOOT_COMPLETED
        mSystemServiceManager.startBootPhase(SystemService.PHASE_BOOT_COMPLETED);

        synchronized (this) {

            // 启动那些之前无法启动的进程，比如系统进程未准备好，且不是常驻进程等等！
            final int NP = mProcessesOnHold.size();
            if (NP > 0) {
                ArrayList<ProcessRecord> procs =
                    new ArrayList<ProcessRecord>(mProcessesOnHold);
                for (int ip=0; ip<NP; ip++) {
                    if (DEBUG_PROCESSES) Slog.v(TAG_PROCESSES, "Starting process on hold: "
                            + procs.get(ip));
                    // 启动之前无法启动而被系统暂时监管的应用进程！
                    startProcessLocked(procs.get(ip), "on-hold", null);
                }
            }

            if (mFactoryTest != FactoryTest.FACTORY_TEST_LOW_LEVEL) {
                Message nmsg = mHandler.obtainMessage(CHECK_EXCESSIVE_WAKE_LOCKS_MSG);
                mHandler.sendMessageDelayed(nmsg, POWER_CHECK_DELAY);

                // 设置系统属性！
                SystemProperties.set("sys.boot_completed", "1");

                if (!"trigger_restart_min_framework".equals(SystemProperties.get("vold.decrypt"))
                    || "".equals(SystemProperties.get("vold.encrypt_progress"))) {
                    SystemProperties.set("dev.bootcomplete", "1");
                }
                
                //【2】解锁设备用户，发送开机相关的广播！
                mUserController.sendBootCompletedLocked(
                        new IIntentReceiver.Stub() {
                            @Override
                            public void performReceive(Intent intent, int resultCode,
                                    String data, Bundle extras, boolean ordered,
                                    boolean sticky, int sendingUser) {
                                synchronized (ActivityManagerService.this) {
                                    requestPssAllProcsLocked(SystemClock.uptimeMillis(),
                                            true, false);
                                }
                            }
                        });

                scheduleStartProfilesLocked();
            }
        }
    }
```

总体来看，逻辑不复杂，最后的一步很关键： 
```java
mUserController.sendBootCompletedLocked(...);
```

该方法会发送一系列的开机广播，并且会解除所有启动完成的设备用户的锁定状态！

广播的发送顺序是：

```java
ACTION_LOCKED_BOOT_COMPLETED -> ACTION_PRE_BOOT_COMPLETED -> ACTION_BOOT_COMPLETED
```

- **ACTION_LOCKED_BOOT_COMPLETED**

因为此时设备用户仍然处于锁定状态，该方法会首先发送 `ACTION_LOCKED_BOOT_COMPLETED` 广播，
这个广播的触发时机是：设备用户已经完成了启动，当时仍然处于锁定状态！

发生了该广播后，系统进程就会开始接触设备用户的锁定！！

- **ACTION_PRE_BOOT_COMPLETED**

当设备用户解除锁定后，就会发送这个广播，这个广播的发送时机是，系统升级后，设备的 `FINGERPRINT` 发生了变化！如果没有发生变化的话，会直接发送 `ACTION_BOOT_COMPLETED` 广播！


下面是核心代码：
```java
    void finishUserUnlocked(final UserState uss) {
        final int userId = uss.mHandle.getIdentifier();
        synchronized (mService) {
            ... ... ... ...

            if (uss.setState(STATE_RUNNING_UNLOCKING, STATE_RUNNING_UNLOCKED)) {
                ... ... ... ... ...
          
                final UserInfo info = getUserInfo(userId);
                 
                if (!Objects.equals(info.lastLoggedInFingerprint, Build.FINGERPRINT)) {
                    final boolean quiet;
                    if (info.isManagedProfile()) {
                        quiet = !uss.tokenProvided
                                || !mLockPatternUtils.isSeparateProfileChallengeEnabled(userId);
                    } else {
                        quiet = false;
                    }
                    // PreBootBroadcaster 的 sendNext 方法中会发送 ACTION_PRE_BOOT_COMPLETED！
                    new PreBootBroadcaster(mService, userId, null, quiet) {
                        @Override
                        public void onFinished() {
                            // 该方法会发送 ACTION_BOOT_COMPLETED 广播！
                            finishUserUnlockedCompleted(uss);
                        }
                    }.sendNext();
                } else {
                    // 该方法会发送 ACTION_BOOT_COMPLETED 广播！
                    finishUserUnlockedCompleted(uss);
                }
            }
        }
    }
```

- **ACTION_BOOT_COMPLETED**

`ACTION_PRE_BOOT_COMPLETED` 广播是最后发送的，也就是我们说的开机广播，发送的代码在 `finishUserUnlockedCompleted` 方法中！

## 4 总结

到这里，`AMS` 的启动过程就分析完了!


  [1]: 