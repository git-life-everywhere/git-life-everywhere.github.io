# Doze模式第 1 篇 - DeviceIdleController 的启动

title: Doze模式第 1 篇 - DeviceIdleController 的启动
date: 2017/10/01 20:46:25 
categories:
- AndroidFramework源码分析
- Doze假寐模式
tags: Doze假寐模式
---


[toc]

基于 Android 7.1.1 源码，分析 doze 模式的原理！

# 0 综述
下面是 Android devoloper 网站对于 doze 模式的介绍：

![image.png-61.3kB](https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/23/image.png)

从 Android 6.0（API 级别 23）开始，Android 引入了两个省电功能，可通过管理应用在设备未连接至电源时的行为方式为用户延长电池寿命。

- 低电耗模式（doze）通过在设备长时间处于闲置状态时推迟应用的后台 CPU 和网络操作来减少电池消耗。

- 应用待机模式可推迟用户近期未与之交互的应用的后台 Activity 的网络操作。

假设一个用户停止充电 (on battery: 利用电池供电)，关闭屏幕 (screen off)。手机处于静止状态 (stationary)。保持以上条件一段时间之后，系统就会进入 Doze 模式。一旦进入 Doze 模式。系统就降低 (延缓) 应用对网络的訪问、以及对 CPU 的占用，来节省电池电量。

这一系列的文章先来分析下低电耗模式!

在低电耗 doze 模式下，应用会受到以下限制：

- 暂停访问 **Network**。
- 系统将忽略 **Wake Locks**。
- 标准 **AlarmManager** 闹铃（包括 setExact() 和 setWindow()）推迟到 doze 模式的下一个 maintenance window 时间窗。
    - 如果您需要设置在低电耗模式下触发的闹铃，请使用 setAndAllowWhileIdle() 或 setExactAndAllowWhileIdle()。
    - 一般情况下，使用 setAlarmClock() 设置的闹铃将继续触发，但系统会在这些闹铃触发之前不久退出低电耗模式。
- 系统不执行 **WiFi** 扫描。
- 系统不允许运行 **Sync** 同步适配器。
- 系统不允许运行 **JobScheduler**。

doze 模式的核心实现在 DeviceIdleController 中，下面我们来看看 DeviceIdleController 的启动！

```java
    private void startOtherServices() {
        final Context context = mSystemContext;
        ... ... ...
        if (mFactoryTestMode != FactoryTest.FACTORY_TEST_LOW_LEVEL) {
            if (!disableNonCoreServices) {
                traceBeginAndSlog("StartLockSettingsService");
                try {
                    mSystemServiceManager.startService(LOCK_SETTINGS_SERVICE_CLASS);
                    lockSettings = ILockSettings.Stub.asInterface(
                            ServiceManager.getService("lock_settings"));
                } catch (Throwable e) {
                    reportWtf("starting LockSettingsService service", e);
                }
                Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);

                if (!SystemProperties.get(PERSISTENT_DATA_BLOCK_PROP).equals("")) {
                    mSystemServiceManager.startService(PersistentDataBlockService.class);
                }
                //【1】启动 DeviceIdleController 服务！
                mSystemServiceManager.startService(DeviceIdleController.class);

                // Always start the Device Policy Manager, so that the API is compatible with
                // API8.
                mSystemServiceManager.startService(DevicePolicyManagerService.Lifecycle.class);
            }

            ... ... ...
        }
        ... ... ... ...
    }
```
启动 DeviceIdleController 的时机是在 SystemServer.startOtherServices 方法中！

# 1 new DeviceIdleController

```java
    public DeviceIdleController(Context context) {
        super(context);
        mConfigFile = new AtomicFile(new File(getSystemDir(), "deviceidle.xml"));
        //【1.1】处理耗时操作！
        mHandler = new MyHandler(BackgroundThread.getHandler().getLooper());
    }
```
配置文件：

位于 /data/system/deviceidle.xml 中！

## 1.1 new MyHandler

```java
    final class MyHandler extends Handler {
        MyHandler(Looper looper) {
            super(looper);
        }

        @Override public void handleMessage(Message msg) {
            if (DEBUG) Slog.d(TAG, "handleMessage(" + msg.what + ")");
            switch (msg.what) {
                case MSG_WRITE_CONFIG: { //【1】更新持久化文件！
                    handleWriteConfigFile();
                } break;
                
                case MSG_REPORT_IDLE_ON: //【2】进入 device idle 状态
                case MSG_REPORT_IDLE_ON_LIGHT: {
                    ... ... ...
                } break;
                
                case MSG_REPORT_IDLE_OFF: { //【3】进入时间窗 window 
                    ... ... ...
                } break;
                
                case MSG_REPORT_ACTIVE: {//【4】退出 doze 模式，处于 active 状态
                    ... ... ...
                } break;
                
                case MSG_TEMP_APP_WHITELIST_TIMEOUT: { //【5】device idle 临时白名单超时移除操作！
                    int uid = msg.arg1; 
                    checkTempAppWhitelistTimeout(uid);
                } break;
                
                case MSG_REPORT_MAINTENANCE_ACTIVITY: {
                    boolean active = (msg.arg1 == 1);
                    final int size = mMaintenanceActivityListeners.beginBroadcast();
                    try {
                        for (int i = 0; i < size; i++) {
                            try {
                                mMaintenanceActivityListeners.getBroadcastItem(i)
                                        .onMaintenanceActivityChanged(active);
                            } catch (RemoteException ignored) {
                            }
                        }
                    } finally {
                        mMaintenanceActivityListeners.finishBroadcast();
                    }
                } break;
                
                case MSG_FINISH_IDLE_OP: {
                    decActiveIdleOps();
                } break;
            }
        }
    }
```
我们看到，MyHandler 绑定到了一个后台线程，他会在后台线程中做一些比较耗时的操作！

# 2 DeviceIdleController.onStart

接着是进入 onStart 方法！

```java
    @Override
    public void onStart() {
        final PackageManager pm = getContext().getPackageManager();

        synchronized (this) {
            //【1】获得 deep mode 和 light mode 的属性配置！
            mLightEnabled = mDeepEnabled = getContext().getResources().getBoolean(
                    com.android.internal.R.bool.config_enableAutoPowerModes);
                    
            //【2】从 SystemConfig 中获得解析到了 doze 模式白名单！
            // 获取在 power save 模式下可以运行，但在 device Idle (doze) 模式下不能运行的系统应用名单！
            SystemConfig sysConfig = SystemConfig.getInstance();
            ArraySet<String> allowPowerExceptIdle = sysConfig.getAllowInPowerSaveExceptIdle();
            for (int i=0; i<allowPowerExceptIdle.size(); i++) {
                String pkg = allowPowerExceptIdle.valueAt(i);
                try {
                    ApplicationInfo ai = pm.getApplicationInfo(pkg,
                            PackageManager.MATCH_SYSTEM_ONLY);
                    int appid = UserHandle.getAppId(ai.uid);
                    mPowerSaveWhitelistAppsExceptIdle.put(ai.packageName, appid);
                    mPowerSaveWhitelistSystemAppIdsExceptIdle.put(appid, true);
                } catch (PackageManager.NameNotFoundException e) {
                }
            }

            //【3】获取在 power save 模式下能够执行的系统应用名单！
            ArraySet<String> allowPower = sysConfig.getAllowInPowerSave();
            for (int i=0; i<allowPower.size(); i++) {
                String pkg = allowPower.valueAt(i);
                try {
                    ApplicationInfo ai = pm.getApplicationInfo(pkg,
                            PackageManager.MATCH_SYSTEM_ONLY);
                    int appid = UserHandle.getAppId(ai.uid);
                    //【3.1】在 power save 模式下能够执行的系统应用，也不能在 doze 模式下运行，
                    // 所以也加入了上面的集合！
                    mPowerSaveWhitelistAppsExceptIdle.put(ai.packageName, appid);
                    mPowerSaveWhitelistSystemAppIdsExceptIdle.put(appid, true);

                    mPowerSaveWhitelistApps.put(ai.packageName, appid);
                    mPowerSaveWhitelistSystemAppIds.put(appid, true);
                } catch (PackageManager.NameNotFoundException e) {
                }
            }

            //【*2.1】创建 Constants 实例，继承 ContentObserver，用于监控数据库变化，
            // 同时也定义了 Doze 模式中的一些时间间隔常量！
            mConstants = new Constants(mHandler, getContext().getContentResolver());
            
            //【*2.2】读取持久化的配置 deviceidle.xml，并将当中定义的 package 
            // 增加到 mPowerSaveWhitelistUserApps 中！
            readConfigFileLocked();

            //【*2.3】更新名单！
            updateWhitelistAppIdsLocked();
            
            //【4】初始化状态变量！
            mNetworkConnected = true;
            mScreenOn = true;
            mCharging = true;
            mState = STATE_ACTIVE;
            mLightState = LIGHT_STATE_ACTIVE;
            mInactiveTimeout = mConstants.INACTIVE_TIMEOUT;
        }

        //【2.4】创建 BinderService 对象，作为 DeviceIdleController 服务桩对象，
        // 并添加到 ServiceManager 中，用于和其他进程通信！
        mBinderService = new BinderService();
        publishBinderService(Context.DEVICE_IDLE_CONTROLLER, mBinderService);

        //【2.5】创建对应的 LocalService，方便于进程内部通信！
        publishLocalService(LocalService.class, new LocalService());
    }
```
**SystemConfig** 我们之前在分析 PMS 的启动的时候有涉及到，SystemConfig 会用来解析系统的配置信息！

这里涉及到了两个内部变量！

DeviceIdleController 提供了 2 中模式： mLightEnabled 和 mDeepEnabled！

- **deep idle 模式**：会禁止 NetWork、Wakelock，还会禁止 Alarm。

- **light idle 模式**：会禁止 NetWork、Wakelock，但是不会禁止 Alarm。

通过 config_enableAutoPowerModes 属性进行初始化，默认是 mLightEnabled = mDeepEnabled = false！

我们再来看下这里涉及到的重要集合：

- **mPowerSaveWhitelistAppsExceptIdle**：保存在 power save 模式下可以后台运行，但在 doze 模式下不能后台运行的系统应用名单，packageName 和 appId 的映射！

- **mPowerSaveWhitelistSystemAppIdsExceptIdle**：保存在 power save 模式下后台运行运行，但在 doze 模式下不能后台运行的系统应用名单，appId 和 true 的映射！

- **mPowerSaveWhitelistApps**：保存在 power save 模式下可以后台运行的系统应用程序，packageName 和 appId 的映射！
  
- **mPowerSaveWhitelistSystemAppIds**：保存在 power save 模式下可以后台运行的系统应用程序，appId 和 true 的映射！


## 2.1 new Constants

```java
        public Constants(Handler handler, ContentResolver resolver) {
            super(handler);
            mResolver = resolver;
            mHasWatch = getContext().getPackageManager().hasSystemFeature(
                    PackageManager.FEATURE_WATCH);
            //【1】监控数据的变化！
            mResolver.registerContentObserver(Settings.Global.getUriFor(
                    mHasWatch ? Settings.Global.DEVICE_IDLE_CONSTANTS_WATCH
                              : Settings.Global.DEVICE_IDLE_CONSTANTS),
                    false, this);
            //【2.1.1】更新常量值！
            updateConstants();
        }
```
可以看到 Constants 会监控数据库的变化，这里先做了一个 feature 的判断：PackageManager.FEATURE_WATCH，判断当前设备是否是可穿戴设备：watch！

因为这里我们关注的是手机设备，所以 Constants 监控的数据库是：Settings.Global.DEVICE_IDLE_CONSTANTS（device_idle_constants）

### 2.1.1 Constants.updateConstants

updateConstants 会从 device_idle_constants 数据库中读取属性值，初始化一些关键的常量信息，如果读取不到，会初始化为默认值！

```java
        private void updateConstants() {
            synchronized (DeviceIdleController.this) {
                try {
                    //【1】读取数据库的值！
                    mParser.setString(Settings.Global.getString(mResolver,
                            mHasWatch ? Settings.Global.DEVICE_IDLE_CONSTANTS_WATCH
                                      : Settings.Global.DEVICE_IDLE_CONSTANTS));
                } catch (IllegalArgumentException e) {
                    Slog.e(TAG, "Bad device idle settings", e);
                }
                //【2】初始化关键性常量！
                LIGHT_IDLE_AFTER_INACTIVE_TIMEOUT = mParser.getLong(
                        KEY_LIGHT_IDLE_AFTER_INACTIVE_TIMEOUT,
                        !COMPRESS_TIME ? 5 * 60 * 1000L : 15 * 1000L);
                LIGHT_PRE_IDLE_TIMEOUT = mParser.getLong(KEY_LIGHT_PRE_IDLE_TIMEOUT,
                        !COMPRESS_TIME ? 10 * 60 * 1000L : 30 * 1000L);
                LIGHT_IDLE_TIMEOUT = mParser.getLong(KEY_LIGHT_IDLE_TIMEOUT,
                        !COMPRESS_TIME ? 5 * 60 * 1000L : 15 * 1000L);
                LIGHT_IDLE_FACTOR = mParser.getFloat(KEY_LIGHT_IDLE_FACTOR,
                        2f);
                LIGHT_MAX_IDLE_TIMEOUT = mParser.getLong(KEY_LIGHT_MAX_IDLE_TIMEOUT,
                        !COMPRESS_TIME ? 15 * 60 * 1000L : 60 * 1000L);
                LIGHT_IDLE_MAINTENANCE_MIN_BUDGET = mParser.getLong(
                        KEY_LIGHT_IDLE_MAINTENANCE_MIN_BUDGET,
                        !COMPRESS_TIME ? 1 * 60 * 1000L : 15 * 1000L);
                LIGHT_IDLE_MAINTENANCE_MAX_BUDGET = mParser.getLong(
                        KEY_LIGHT_IDLE_MAINTENANCE_MAX_BUDGET,
                        !COMPRESS_TIME ? 5 * 60 * 1000L : 30 * 1000L);
                MIN_LIGHT_MAINTENANCE_TIME = mParser.getLong(
                        KEY_MIN_LIGHT_MAINTENANCE_TIME,
                        !COMPRESS_TIME ? 5 * 1000L : 1 * 1000L);
                MIN_DEEP_MAINTENANCE_TIME = mParser.getLong(
                        KEY_MIN_DEEP_MAINTENANCE_TIME,
                        !COMPRESS_TIME ? 30 * 1000L : 5 * 1000L);
                long inactiveTimeoutDefault = (mHasWatch ? 15 : 30) * 60 * 1000L;
                INACTIVE_TIMEOUT = mParser.getLong(KEY_INACTIVE_TIMEOUT,
                        !COMPRESS_TIME ? inactiveTimeoutDefault : (inactiveTimeoutDefault / 10));
                SENSING_TIMEOUT = mParser.getLong(KEY_SENSING_TIMEOUT,
                        !DEBUG ? 4 * 60 * 1000L : 60 * 1000L);
                LOCATING_TIMEOUT = mParser.getLong(KEY_LOCATING_TIMEOUT,
                        !DEBUG ? 30 * 1000L : 15 * 1000L);
                LOCATION_ACCURACY = mParser.getFloat(KEY_LOCATION_ACCURACY, 20);
                MOTION_INACTIVE_TIMEOUT = mParser.getLong(KEY_MOTION_INACTIVE_TIMEOUT,
                        !COMPRESS_TIME ? 10 * 60 * 1000L : 60 * 1000L);
                long idleAfterInactiveTimeout = (mHasWatch ? 15 : 30) * 60 * 1000L;
                IDLE_AFTER_INACTIVE_TIMEOUT = mParser.getLong(KEY_IDLE_AFTER_INACTIVE_TIMEOUT,
                        !COMPRESS_TIME ? idleAfterInactiveTimeout
                                       : (idleAfterInactiveTimeout / 10));
                IDLE_PENDING_TIMEOUT = mParser.getLong(KEY_IDLE_PENDING_TIMEOUT,
                        !COMPRESS_TIME ? 5 * 60 * 1000L : 30 * 1000L);
                MAX_IDLE_PENDING_TIMEOUT = mParser.getLong(KEY_MAX_IDLE_PENDING_TIMEOUT,
                        !COMPRESS_TIME ? 10 * 60 * 1000L : 60 * 1000L);
                IDLE_PENDING_FACTOR = mParser.getFloat(KEY_IDLE_PENDING_FACTOR,
                        2f);
                IDLE_TIMEOUT = mParser.getLong(KEY_IDLE_TIMEOUT,
                        !COMPRESS_TIME ? 60 * 60 * 1000L : 6 * 60 * 1000L);
                MAX_IDLE_TIMEOUT = mParser.getLong(KEY_MAX_IDLE_TIMEOUT,
                        !COMPRESS_TIME ? 6 * 60 * 60 * 1000L : 30 * 60 * 1000L);
                IDLE_FACTOR = mParser.getFloat(KEY_IDLE_FACTOR,
                        2f);
                MIN_TIME_TO_ALARM = mParser.getLong(KEY_MIN_TIME_TO_ALARM,
                        !COMPRESS_TIME ? 60 * 60 * 1000L : 6 * 60 * 1000L);
                MAX_TEMP_APP_WHITELIST_DURATION = mParser.getLong(
                        KEY_MAX_TEMP_APP_WHITELIST_DURATION, 5 * 60 * 1000L);
                MMS_TEMP_APP_WHITELIST_DURATION = mParser.getLong(
                        KEY_MMS_TEMP_APP_WHITELIST_DURATION, 60 * 1000L);
                SMS_TEMP_APP_WHITELIST_DURATION = mParser.getLong(
                        KEY_SMS_TEMP_APP_WHITELIST_DURATION, 20 * 1000L);
                NOTIFICATION_WHITELIST_DURATION = mParser.getLong(
                        KEY_NOTIFICATION_WHITELIST_DURATION, 30 * 1000L);
            }
        }
```
这里的 COMPRESS_TIME 为 false！

```java
    private static final boolean COMPRESS_TIME = false;
```

### 2.1.2 Constants 常量

从上面可以看到，doze 模式下涉及到很多和时间相关的值，这里我们来解释一下：

#### 2.1.2.1 Light Idle 常量

- **LIGHT_IDLE_AFTER_INACTIVE_TIMEOUT**
light idle 时间属性，取值为 5 mins，当设备进入灭屏不充电的状态时，需要持续的时间!

```java
    LIGHT_IDLE_AFTER_INACTIVE_TIMEOUT = mParser.getLong(
            KEY_LIGHT_IDLE_AFTER_INACTIVE_TIMEOUT,
            !COMPRESS_TIME ? 5 * 60 * 1000L : 15 * 1000L);
```

- **LIGHT_PRE_IDLE_TIMEOUT**
light idle 时间属性，取值为 10 mins，在进入 LIGHT_STATE_PRE_IDLE 阶段之前，如果判断设备有活跃操作执行的话，我们会设置一个 10mins 以后的 Alarm，等待这些操作执行完成，再进入 LIGHT_STATE_PRE_IDLE 阶段的处理！ 

```java                        
    LIGHT_PRE_IDLE_TIMEOUT = mParser.getLong(KEY_LIGHT_PRE_IDLE_TIMEOUT,
            !COMPRESS_TIME ? 10 * 60 * 1000L : 30 * 1000L);
```

- **LIGHT_IDLE_TIMEOUT**
light idle 时间属性，取值为 5 mins，表示 light idle 的最小持续时间，也是第一次进入 light idle 的持续时间！

```java
    LIGHT_IDLE_TIMEOUT = mParser.getLong(KEY_LIGHT_IDLE_TIMEOUT,
            !COMPRESS_TIME ? 5 * 60 * 1000L : 15 * 1000L);
```

- **LIGHT_IDLE_FACTOR**
light idle 时间属性，取值为 2f, light idle 持续时间因子！

```java
    LIGHT_IDLE_FACTOR = mParser.getFloat(KEY_LIGHT_IDLE_FACTOR, 2f);
```

- **LIGHT_MAX_IDLE_TIMEOUT**
light idle 时间属性，取值为 15 mins，表示 light idle 的最大持续时间！

```java
    LIGHT_MAX_IDLE_TIMEOUT = mParser.getLong(KEY_LIGHT_MAX_IDLE_TIMEOUT,
            !COMPRESS_TIME ? 15 * 60 * 1000L : 60 * 1000L);
```

- **LIGHT_IDLE_MAINTENANCE_MIN_BUDGET**
light idle 时间属性，取值为 1mins, light idle 的最小时间窗！

```java
    LIGHT_IDLE_MAINTENANCE_MIN_BUDGET = mParser.getLong(
            KEY_LIGHT_IDLE_MAINTENANCE_MIN_BUDGET,
            !COMPRESS_TIME ? 1 * 60 * 1000L : 15 * 1000L);
```

- **LIGHT_IDLE_MAINTENANCE_MAX_BUDGET**
light idle 时间属性，取值为 5mins, light idle 的最大时间窗！
```java                
    LIGHT_IDLE_MAINTENANCE_MAX_BUDGET = mParser.getLong(
            KEY_LIGHT_IDLE_MAINTENANCE_MAX_BUDGET,
            !COMPRESS_TIME ? 5 * 60 * 1000L : 30 * 1000L);
```

- **MIN_LIGHT_MAINTENANCE_TIME**
light idle 时间属性，取值为 5s, 用于判断是否提前退出时间窗！
```java                        
    MIN_LIGHT_MAINTENANCE_TIME = mParser.getLong(
            KEY_MIN_LIGHT_MAINTENANCE_TIME,
            !COMPRESS_TIME ? 5 * 1000L : 1 * 1000L);
```

#### 2.1.2.2 Deep Idle 常量

- **MIN_DEEP_MAINTENANCE_TIME**
deep idle 时间属性，取值为 30s, 用于判断是否提前退出时间窗！
```java
    MIN_DEEP_MAINTENANCE_TIME = mParser.getLong(
            KEY_MIN_DEEP_MAINTENANCE_TIME,
            !COMPRESS_TIME ? 30 * 1000L : 5 * 1000L);
```

- **INACTIVE_TIMEOUT**
deep idle 时间属性，取值为 30 mins，当设备的角度发生变化后，会回到 STATE_ACTIVE 状态，当下次满足灭屏不充电时，需要持续该时间才能进入 STATE_INACTIVE 状态；第一次进入满足灭屏不充电时，时间值也由 INACTIVE_TIMEOUT 设置  ！！

```java
    long inactiveTimeoutDefault = (mHasWatch ? 15 : 30) * 60 * 1000L;
    INACTIVE_TIMEOUT = mParser.getLong(KEY_INACTIVE_TIMEOUT,
            !COMPRESS_TIME ? inactiveTimeoutDefault : (inactiveTimeoutDefault / 10));
```

- **SENSING_TIMEOUT**      
deep idle 时间属性，取值为 4 mins！
```java
    SENSING_TIMEOUT = mParser.getLong(KEY_SENSING_TIMEOUT,
            !DEBUG ? 4 * 60 * 1000L : 60 * 1000L);
```

- **LOCATING_TIMEOUT**
deep idle 时间属性，取值为 30s，当处于定位阶段时，要持续的时间！
```java                        
    LOCATING_TIMEOUT = mParser.getLong(KEY_LOCATING_TIMEOUT,
            !DEBUG ? 30 * 1000L : 15 * 1000L);
```
- **LOCATION_ACCURACY**
deep idle 定位属性，取值为 20meter，定位精度！
```java                        
    LOCATION_ACCURACY = mParser.getFloat(KEY_LOCATION_ACCURACY, 20);
```

- **MOTION_INACTIVE_TIMEOUT**
deep idle 时间属性，取值为 10mins，当 sensor 被触发后，会回到 STATE_ACTIVE 状态，当下次满足灭屏不充电时，持续的时间就由 30mins 缩短为 10mins！

```java
    MOTION_INACTIVE_TIMEOUT = mParser.getLong(KEY_MOTION_INACTIVE_TIMEOUT,
            !COMPRESS_TIME ? 10 * 60 * 1000L : 60 * 1000L);
```

- **IDLE_AFTER_INACTIVE_TIMEOUT**
deep idle 时间属性，取值为 30 mins，当设备在 STATE_INACTIVE 阶段是会监听设备运动，需要保持不运动 30mins 才能进入 STATE_IDLE_PENDING 阶段！!

```                        
    long idleAfterInactiveTimeout = (mHasWatch ? 15 : 30) * 60 * 1000L;
    IDLE_AFTER_INACTIVE_TIMEOUT = mParser.getLong(KEY_IDLE_AFTER_INACTIVE_TIMEOUT,
            !COMPRESS_TIME ? idleAfterInactiveTimeout
                           : (idleAfterInactiveTimeout / 10));
```

- **IDLE_PENDING_TIMEOUT**
deep idle 时间属性，取值为 5 mins，在 deep idle 状态下的初始时间窗！

```java                                      
    IDLE_PENDING_TIMEOUT = mParser.getLong(KEY_IDLE_PENDING_TIMEOUT,
            !COMPRESS_TIME ? 5 * 60 * 1000L : 30 * 1000L);
```

- **MAX_IDLE_PENDING_TIMEOUT**
deep idle 时间属性，取值为 10 mins，在 deep idle 状态下的最大时间窗！
```java
    MAX_IDLE_PENDING_TIMEOUT = mParser.getLong(KEY_MAX_IDLE_PENDING_TIMEOUT,
            !COMPRESS_TIME ? 10 * 60 * 1000L : 60 * 1000L);
```

- **IDLE_PENDING_FACTOR**
deep idle 因子属性，取值为 2f，用于调整 deep idle 的时间窗！

```java
    IDLE_PENDING_FACTOR = mParser.getFloat(KEY_IDLE_PENDING_FACTOR,
            2f);
```

- **IDLE_TIMEOUT**
deep idle 时间属性，取值为 60 mins，deep idle 状态的初始持续时间！
```java                        
    IDLE_TIMEOUT = mParser.getLong(KEY_IDLE_TIMEOUT,
            !COMPRESS_TIME ? 60 * 60 * 1000L : 6 * 60 * 1000L);
```

- **MAX_IDLE_TIMEOUT**
deep idle 时间属性，取值为 6 hours，deep idle 状态的最大持续时间！
```java     
    MAX_IDLE_TIMEOUT = mParser.getLong(KEY_MAX_IDLE_TIMEOUT,
            !COMPRESS_TIME ? 6 * 60 * 60 * 1000L : 30 * 60 * 1000L);
```

- **IDLE_FACTOR**
deep idle 因子属性，取值为 2f，用于调整 deep idle 的时间窗！
```java                        
    IDLE_FACTOR = mParser.getFloat(KEY_IDLE_FACTOR, 2f);
```

- **MIN_TIME_TO_ALARM**
deep idle 时间属性，取值为 60mins，在每次改变 deep 状态时，会判断是否有 Alarm 在该时间内触发，如果有，不会进入 deep idle 状态！

```java                        
    MIN_TIME_TO_ALARM = mParser.getLong(KEY_MIN_TIME_TO_ALARM,
            !COMPRESS_TIME ? 60 * 60 * 1000L : 6 * 60 * 1000L);
```

#### 2.1.2.3 其他相关变量

- **MAX_TEMP_APP_WHITELIST_DURATION**
doze 模式的临时缓存白名单的最大有效时间，取值为 5mins！
```java                        
    MAX_TEMP_APP_WHITELIST_DURATION = mParser.getLong(
            KEY_MAX_TEMP_APP_WHITELIST_DURATION, 5 * 60 * 1000L);
```

- **MMS_TEMP_APP_WHITELIST_DURATION**
MMS 应用的 doze 模式的临时缓存白名单的有效时间，60s

```java                          
    MMS_TEMP_APP_WHITELIST_DURATION = mParser.getLong(
            KEY_MMS_TEMP_APP_WHITELIST_DURATION, 60 * 1000L);
```
- **SMS_TEMP_APP_WHITELIST_DURATION**
SMS 应用的 doze 模式的临时缓存白名单的有效时间，20s
```java                         
    SMS_TEMP_APP_WHITELIST_DURATION = mParser.getLong(
            KEY_SMS_TEMP_APP_WHITELIST_DURATION, 20 * 1000L);
```
- **NOTIFICATION_WHITELIST_DURATION**
通知的白名单时间间隔，30s

```java                          
    NOTIFICATION_WHITELIST_DURATION = mParser.getLong(
            KEY_NOTIFICATION_WHITELIST_DURATION, 30 * 1000L);
```

对于这些变量的使用，我们后续再分析！

## 2.2 DeviceIdleController.readConfigFileLocked

```java
   void readConfigFileLocked() {
        if (DEBUG) Slog.d(TAG, "Reading config from " + mConfigFile.getBaseFile());
        //【1】清空 mPowerSaveWhitelistUserApps 集合！
        mPowerSaveWhitelistUserApps.clear();
        FileInputStream stream;
        try {
            stream = mConfigFile.openRead(); //【2】创建 /data/system/deviceidle.xml 输入流！
        } catch (FileNotFoundException e) {
            return;
        }
        try {
            XmlPullParser parser = Xml.newPullParser();
            parser.setInput(stream, StandardCharsets.UTF_8.name());
            //【2.2.1】读取文件！
            readConfigFileLocked(parser);
        } catch (XmlPullParserException e) {
        } finally {
            try {
                stream.close();
            } catch (IOException e) {
            }
        }
    }
```
继续调用另一个 readConfigFileLocked 方法！
```java
    private void readConfigFileLocked(XmlPullParser parser) {
        final PackageManager pm = getContext().getPackageManager();

        try {
            int type;
            while ((type = parser.next()) != XmlPullParser.START_TAG
                    && type != XmlPullParser.END_DOCUMENT) {
                ;
            }

            if (type != XmlPullParser.START_TAG) {
                throw new IllegalStateException("no start tag found");
            }

            int outerDepth = parser.getDepth();
            while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
                    && (type != XmlPullParser.END_TAG || parser.getDepth() > outerDepth)) {
                if (type == XmlPullParser.END_TAG || type == XmlPullParser.TEXT) {
                    continue;
                }

                String tagName = parser.getName();
                if (tagName.equals("wl")) {
                    String name = parser.getAttributeValue(null, "n");
                    if (name != null) {
                        try {
                            ApplicationInfo ai = pm.getApplicationInfo(name,
                                    PackageManager.MATCH_UNINSTALLED_PACKAGES);
                            //【1】将用户应用程序的包名和 appId 加入到 mPowerSaveWhitelistUserApps 中；
                            mPowerSaveWhitelistUserApps.put(ai.packageName,
                                    UserHandle.getAppId(ai.uid));
                        } catch (PackageManager.NameNotFoundException e) {
                        }
                    }
                } else {
                    Slog.w(TAG, "Unknown element under <config>: "
                            + parser.getName());
                    XmlUtils.skipCurrentTag(parser);
                }
            }

        } catch (IllegalStateException e) {
            Slog.w(TAG, "Failed parsing config " + e);
        } catch (NullPointerException e) {
            Slog.w(TAG, "Failed parsing config " + e);
        } catch (NumberFormatException e) {
            Slog.w(TAG, "Failed parsing config " + e);
        } catch (XmlPullParserException e) {
            Slog.w(TAG, "Failed parsing config " + e);
        } catch (IOException e) {
            Slog.w(TAG, "Failed parsing config " + e);
        } catch (IndexOutOfBoundsException e) {
            Slog.w(TAG, "Failed parsing config " + e);
        }
    }
```
可以看到，readConfigFileLocked 方法最终解析 /data/system/deviceidle.xml 文件。

该文件中保存的是在 device mode 模式下，能够后台运行的用户应用白名单！

最终的解析结果，保存到了 **mPowerSaveWhitelistUserApps** 集合中，packageName 和 appId 的映射！

## 2.3 DeviceIdleController.updateWhitelistAppIdsLocked

我们先来回顾下前面涉及到的几个集合：

- **mPowerSaveWhitelistAppsExceptIdle**：保存在 power save 模式下可以后台运行，但在 doze 模式下不能后台运行的系统应用名单，packageName 和 appId 的映射！

- **mPowerSaveWhitelistSystemAppIdsExceptIdle**：保存在 power save 模式下可以后台运行，但在 doze 模式下不能后台运行的系统应用名单，appId 和 true 的映射！

- **mPowerSaveWhitelistApps**：保存在 power save 模式下可以后台运行的系统应用，packageName 和 appId 的映射！
  
- **mPowerSaveWhitelistSystemAppIds**：保存在 power save 模式下可以后台运行的系统应用，appId 和 true 的映射！

```java
    private void updateWhitelistAppIdsLocked() {
        //【1】合并白名单！
        mPowerSaveWhitelistExceptIdleAppIdArray = buildAppIdArray(mPowerSaveWhitelistAppsExceptIdle,
                mPowerSaveWhitelistUserApps, mPowerSaveWhitelistExceptIdleAppIds);

        mPowerSaveWhitelistAllAppIdArray = buildAppIdArray(mPowerSaveWhitelistApps,
                mPowerSaveWhitelistUserApps, mPowerSaveWhitelistAllAppIds);

        mPowerSaveWhitelistUserAppIdArray = buildAppIdArray(null,
                mPowerSaveWhitelistUserApps, mPowerSaveWhitelistUserAppIds);
        
        //【2】将白名单设置到 PowerManager 和 AlarmManager 中！
        if (mLocalPowerManager != null) {
            if (DEBUG) {
                Slog.d(TAG, "Setting wakelock whitelist to "
                        + Arrays.toString(mPowerSaveWhitelistAllAppIdArray));
            }
            mLocalPowerManager.setDeviceIdleWhitelist(mPowerSaveWhitelistAllAppIdArray);
        }
        if (mLocalAlarmManager != null) {
            if (DEBUG) {
                Slog.d(TAG, "Setting alarm whitelist to "
                        + Arrays.toString(mPowerSaveWhitelistUserAppIdArray));
            }
            mLocalAlarmManager.setDeviceIdleUserWhitelist(mPowerSaveWhitelistUserAppIdArray);
        }
    }
```
1、这里又设计到了几个重要集合：

- **mPowerSaveWhitelistExceptIdleAppIds**：

保存在 power save 模式下可以后台运行，但在 doze 模式下不能后台运行的系统和用户应用，packageName 和 appId 的映射；

集合来自 mPowerSaveWhitelistAppsExceptIdle 和 mPowerSaveWhitelistUserApps 的合并；

- **mPowerSaveWhitelistExceptIdleAppIdArray**：

保存在 power save 模式下可以后台运行，但在 doze 模式下不能后台运行的系统和用户应用 appId；

集合来自 mPowerSaveWhitelistAppsExceptIdle 和 mPowerSaveWhitelistUserApps 的合并；

- **mPowerSaveWhitelistAllAppIds**：

保存在 power save 模式下可以后台运行的系统和用户应用 (appId 和 true 的映射)；

集合来自 mPowerSaveWhitelistApps 和 mPowerSaveWhitelistUserApps 的合并；
                    
- **mPowerSaveWhitelistAllAppIdArray**：

保存在 power save 模式下可以后台运行的系统和用户应用 appId；

集合来自 mPowerSaveWhitelistApps 和 mPowerSaveWhitelistUserApps 的合并；

- **mPowerSaveWhitelistUserAppIds**：

保存在 power save 模式下可以后台运行的用户应用 (appId 和 true 的映射)；

集合来自 mPowerSaveWhitelistUserApps；
                    
- **mPowerSaveWhitelistUserAppIdArray**：

保存在 power save 模式下可以后台运行的用户应用 appId；

集合来自 mPowerSaveWhitelistUserApps；

</br>

2、传递名单

接着，将 mPowerSaveWhitelistAllAppIdArray 传递给了 PowerManagerService；

然后，将 mPowerSaveWhitelistUserAppIdArray 传递给了 AlarmManagerService；

### 2.3.1 DeviceIdleController.buildAppIdArray
```java
    private static int[] buildAppIdArray(ArrayMap<String, Integer> systemApps,
            ArrayMap<String, Integer> userApps, SparseBooleanArray outAppIds) {
        outAppIds.clear();
        //【1】将 systemApps 和 userApps 合并到 outAppIds 中，appId 和 true 值的映射！
        if (systemApps != null) {
            for (int i = 0; i < systemApps.size(); i++) {
                outAppIds.put(systemApps.valueAt(i), true);
            }
        }
        if (userApps != null) {
            for (int i = 0; i < userApps.size(); i++) {
                outAppIds.put(userApps.valueAt(i), true);
            }
        }
        //【2】将 outAppIds 中的所有 appId 保存到数组 appids 中，并返回！
        int size = outAppIds.size();
        int[] appids = new int[size];
        for (int i = 0; i < size; i++) {
            appids[i] = outAppIds.keyAt(i);
        }
        return appids;
    }
```


## 2.4 new BinderService

创建了一个 BinderService 对象，继承了 IDeviceIdleController.Stub，作为服务端桩对象，用于其它进程访问 DeviceIdleController 内部接口！！

```java
    private final class BinderService extends IDeviceIdleController.Stub {
        //【1】添加应用到 power save 应用白名单中！
        @Override public void addPowerSaveWhitelistApp(String name) {
            getContext().enforceCallingOrSelfPermission(android.Manifest.permission.DEVICE_POWER,
                    null);
            long ident = Binder.clearCallingIdentity();
            try {
                addPowerSaveWhitelistAppInternal(name);
            } finally {
                Binder.restoreCallingIdentity(ident);
            }
        }
        //【2】移除应用从 power save 应用白名单中！
        @Override public void removePowerSaveWhitelistApp(String name) {
            getContext().enforceCallingOrSelfPermission(android.Manifest.permission.DEVICE_POWER,
                    null);
            long ident = Binder.clearCallingIdentity();
            try {
                removePowerSaveWhitelistAppInternal(name);
            } finally {
                Binder.restoreCallingIdentity(ident);
            }
        }
        //【3】获取在 power save 模式下可以后台运行但是在 device idle 状态下不能后台运行的白名单！
        @Override public String[] getSystemPowerWhitelistExceptIdle() {
            return getSystemPowerWhitelistExceptIdleInternal();
        }

        @Override public String[] getSystemPowerWhitelist() {
            return getSystemPowerWhitelistInternal();
        }

        @Override public String[] getUserPowerWhitelist() {
            return getUserPowerWhitelistInternal();
        }

        @Override public String[] getFullPowerWhitelistExceptIdle() {
            return getFullPowerWhitelistExceptIdleInternal();
        }

        @Override public String[] getFullPowerWhitelist() {
            return getFullPowerWhitelistInternal();
        }

        @Override public int[] getAppIdWhitelistExceptIdle() {
            return getAppIdWhitelistExceptIdleInternal();
        }

        @Override public int[] getAppIdWhitelist() {
            return getAppIdWhitelistInternal();
        }

        @Override public int[] getAppIdUserWhitelist() {
            return getAppIdUserWhitelistInternal();
        }

        @Override public int[] getAppIdTempWhitelist() {
            return getAppIdTempWhitelistInternal();
        }

        @Override public boolean isPowerSaveWhitelistExceptIdleApp(String name) {
            return isPowerSaveWhitelistExceptIdleAppInternal(name);
        }

        @Override public boolean isPowerSaveWhitelistApp(String name) {
            return isPowerSaveWhitelistAppInternal(name);
        }

        @Override public void addPowerSaveTempWhitelistApp(String packageName, long duration,
                int userId, String reason) throws RemoteException {
            addPowerSaveTempWhitelistAppChecked(packageName, duration, userId, reason);
        }

        @Override public long addPowerSaveTempWhitelistAppForMms(String packageName,
                int userId, String reason) throws RemoteException {
            long duration = mConstants.MMS_TEMP_APP_WHITELIST_DURATION;
            addPowerSaveTempWhitelistAppChecked(packageName, duration, userId, reason);
            return duration;
        }

        @Override public long addPowerSaveTempWhitelistAppForSms(String packageName,
                int userId, String reason) throws RemoteException {
            long duration = mConstants.SMS_TEMP_APP_WHITELIST_DURATION;
            addPowerSaveTempWhitelistAppChecked(packageName, duration, userId, reason);
            return duration;
        }

        @Override public void exitIdle(String reason) {
            getContext().enforceCallingOrSelfPermission(Manifest.permission.DEVICE_POWER,
                    null);
            long ident = Binder.clearCallingIdentity();
            try {
                exitIdleInternal(reason);
            } finally {
                Binder.restoreCallingIdentity(ident);
            }
        }

        @Override public boolean registerMaintenanceActivityListener(
                IMaintenanceActivityListener listener) {
            return DeviceIdleController.this.registerMaintenanceActivityListener(listener);
        }

        @Override public void unregisterMaintenanceActivityListener(
                IMaintenanceActivityListener listener) {
            DeviceIdleController.this.unregisterMaintenanceActivityListener(listener);
        }

        @Override protected void dump(FileDescriptor fd, PrintWriter pw, String[] args) {
            DeviceIdleController.this.dump(fd, pw, args);
        }

        @Override public void onShellCommand(FileDescriptor in, FileDescriptor out,
                FileDescriptor err, String[] args, ResultReceiver resultReceiver) {
            (new Shell()).exec(this, in, out, err, args, resultReceiver);
        }
    }
```

这里的接口，我们先不详细看！

最后，将桩对象注册到了 ServiceManager 中！

```java
publishBinderService(Context.DEVICE_IDLE_CONTROLLER, mBinderService);
```

## 2.5 new LocalService

创建了本地服务对象，方便进程内部通信！
```java
    public final class LocalService {
        public void addPowerSaveTempWhitelistAppDirect(int appId, long duration, boolean sync,
                String reason) {
            addPowerSaveTempWhitelistAppDirectInternal(0, appId, duration, sync, reason);
        }

        public long getNotificationWhitelistDuration() {
            return mConstants.NOTIFICATION_WHITELIST_DURATION;
        }

        public void setNetworkPolicyTempWhitelistCallback(Runnable callback) {
            setNetworkPolicyTempWhitelistCallbackInternal(callback);
        }

        public void setJobsActive(boolean active) {
            DeviceIdleController.this.setJobsActive(active);
        }

        // Up-call from alarm manager.
        public void setAlarmsActive(boolean active) {
            DeviceIdleController.this.setAlarmsActive(active);
        }

        public int[] getPowerSaveWhitelistUserAppIds() {
            return DeviceIdleController.this.getPowerSaveWhitelistUserAppIds();
        }
    }

```

# 3 DeviceIdleController.onBootPhase
```java
    @Override
    public void onBootPhase(int phase) {
        if (phase == PHASE_SYSTEM_SERVICES_READY) {
            synchronized (this) {
                //【1】获得了一些重要的系统服务：AlarmManager，BatteryStats，PowerManager，ConnectivityService
                // NetworkPolicyManager，DisplayManager 和 SensorManager；
                mAlarmManager = (AlarmManager) getContext().getSystemService(Context.ALARM_SERVICE);
                mBatteryStats = BatteryStatsService.getService();
                mLocalPowerManager = getLocalService(PowerManagerInternal.class);
                mPowerManager = getContext().getSystemService(PowerManager.class);
                mActiveIdleWakeLock = mPowerManager.newWakeLock(PowerManager.PARTIAL_WAKE_LOCK,
                        "deviceidle_maint");
                mActiveIdleWakeLock.setReferenceCounted(false);
                mConnectivityService = (ConnectivityService)ServiceManager.getService(
                        Context.CONNECTIVITY_SERVICE);
                mLocalAlarmManager = getLocalService(AlarmManagerService.LocalService.class);
                mNetworkPolicyManager = INetworkPolicyManager.Stub.asInterface(
                        ServiceManager.getService(Context.NETWORK_POLICY_SERVICE));
                mDisplayManager = (DisplayManager) getContext().getSystemService(
                        Context.DISPLAY_SERVICE);

                // 获得 SensorManager，用于获得系统的传感器信息！
                mSensorManager = (SensorManager) getContext().getSystemService(Context.SENSOR_SERVICE);
                int sigMotionSensorId = getContext().getResources().getInteger(
                        com.android.internal.R.integer.config_autoPowerModeAnyMotionSensor);
                // 获得运动传感器对象，监听手机位置变化！
                if (sigMotionSensorId > 0) {
                    mMotionSensor = mSensorManager.getDefaultSensor(sigMotionSensorId, true);
                }
                if (mMotionSensor == null && getContext().getResources().getBoolean(
                        com.android.internal.R.bool.config_autoPowerModePreferWristTilt)) {
                    mMotionSensor = mSensorManager.getDefaultSensor(
                            Sensor.TYPE_WRIST_TILT_GESTURE, true);
                }
                if (mMotionSensor == null) {
                    mMotionSensor = mSensorManager.getDefaultSensor(
                            Sensor.TYPE_SIGNIFICANT_MOTION, true);
                }

                if (getContext().getResources().getBoolean(
                        com.android.internal.R.bool.config_autoPowerModePrefetchLocation)) {
                    mLocationManager = (LocationManager) getContext().getSystemService(
                            Context.LOCATION_SERVICE);
                    mLocationRequest = new LocationRequest()
                        .setQuality(LocationRequest.ACCURACY_FINE)
                        .setInterval(0)
                        .setFastestInterval(0)
                        .setNumUpdates(1);
                }
                // 通过配置文件。得到角度变化的阈值！
                // 创建一个 AnyMotionDetector，同一时候将 DeviceIdleController 注冊到当中
                // 当 AnyMotionDetector 检測到手机变化角度超过阈值时。就会回调 DeviceIdleController 的接口
                float angleThreshold = getContext().getResources().getInteger(
                        com.android.internal.R.integer.config_autoPowerModeThresholdAngle) / 100f;
                mAnyMotionDetector = new AnyMotionDetector(
                        (PowerManager) getContext().getSystemService(Context.POWER_SERVICE),
                        mHandler, mSensorManager, this, angleThreshold);
                
                // 创建了两个广播对象，一个用于通知 deep doze 模式变化；另一个用于通知 light doze 模式变化！
                mIdleIntent = new Intent(PowerManager.ACTION_DEVICE_IDLE_MODE_CHANGED);
                mIdleIntent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY
                        | Intent.FLAG_RECEIVER_FOREGROUND);
                mLightIdleIntent = new Intent(PowerManager.ACTION_LIGHT_DEVICE_IDLE_MODE_CHANGED);
                mLightIdleIntent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY
                        | Intent.FLAG_RECEIVER_FOREGROUND);

                //【3.1】监听电池变化的广播！
                IntentFilter filter = new IntentFilter();
                filter.addAction(Intent.ACTION_BATTERY_CHANGED);
                getContext().registerReceiver(mReceiver, filter);

                //【3】监听 package 变化的广播！
                filter = new IntentFilter();
                filter.addAction(Intent.ACTION_PACKAGE_REMOVED);
                filter.addDataScheme("package");
                getContext().registerReceiver(mReceiver, filter);

                //【4】监听网络变化的广播！
                filter = new IntentFilter();
                filter.addAction(ConnectivityManager.CONNECTIVITY_ACTION);
                getContext().registerReceiver(mReceiver, filter);

                //【5】再次将 mPowerSaveWhitelistAllAppIdArray 传递给了 PowerManagerService；
                // 将 mPowerSaveWhitelistUserAppIdArray 传递给了 AlarmManagerService；
                mLocalPowerManager.setDeviceIdleWhitelist(mPowerSaveWhitelistAllAppIdArray);
                mLocalAlarmManager.setDeviceIdleUserWhitelist(mPowerSaveWhitelistUserAppIdArray);

                //【3.2】注册 DisplayListener 监听熄屏亮屏状态，同时更新屏幕状态！
                mDisplayManager.registerDisplayListener(mDisplayListener, null);
                updateDisplayLocked();
            }
            //【3.3】更新网络连接状态！
            updateConnectivityState(null);
        }
    }
```
这里我们来看看 DisplayListener 对象！

## 3.1 new BroadcastReceiver

```java
    private final BroadcastReceiver mReceiver = new BroadcastReceiver() {
        @Override public void onReceive(Context context, Intent intent) {
            switch (intent.getAction()) {
                case ConnectivityManager.CONNECTIVITY_ACTION: {
                    updateConnectivityState(intent); // 网络变化
                } break;
                case Intent.ACTION_BATTERY_CHANGED: {
                    synchronized (DeviceIdleController.this) {
                        int plugged = intent.getIntExtra("plugged", 0);
                        updateChargingLocked(plugged != 0); // 电量变化
                    }
                } break;
                case Intent.ACTION_PACKAGE_REMOVED: { // 升级包移除！
                    if (!intent.getBooleanExtra(Intent.EXTRA_REPLACING, false)) {
                        Uri data = intent.getData();
                        String ssp;
                        if (data != null && (ssp = data.getSchemeSpecificPart()) != null) {
                            //【3.1.1】更新用户应用白名单！
                            removePowerSaveWhitelistAppInternal(ssp);
                        }
                    }
                } break;
            }
        }
    };
```
创建了一个 BroadcastReceiver 对象，监听 CONNECTIVITY_ACTION，ACTION_BATTERY_CHANGED，ACTION_PACKAGE_REMOVED 广播！

关于监听到变化后的动态处理，这里我们先不分析！

### 3.1.1 DeviceIdleController.removePowerSaveWhitelistAppInternal

```java
    public boolean removePowerSaveWhitelistAppInternal(String name) {
        synchronized (this) {
            if (mPowerSaveWhitelistUserApps.remove(name) != null) {
                //【1】发送 ACTION_POWER_SAVE_WHITELIST_CHANGED 广播！
                reportPowerSaveWhitelistChangedLocked();
                //【2.3.1】更新名单列表，同时
                updateWhitelistAppIdsLocked();
                //【3】更新本地持久化文件！
                writeConfigFileLocked();
                return true;
            }
        }
        return false;
    }
```
reportPowerSaveWhitelistChangedLocked 方法会发送 PowerManager.ACTION_POWER_SAVE_WHITELIST_CHANGED 广播！
```java
    private void reportPowerSaveWhitelistChangedLocked() {
        Intent intent = new Intent(PowerManager.ACTION_POWER_SAVE_WHITELIST_CHANGED);
        intent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY);
        getContext().sendBroadcastAsUser(intent, UserHandle.SYSTEM);
    }

```
writeConfigFileLocked 方法会发送 MSG_WRITE_CONFIG 消息给 MyHandler。MyHandler 会处理消息，调用 handleWriteConfigFile -> writeConfigFileLocked(XmlSerializer out) 方法，更新本地名单！
```java
    void writeConfigFileLocked() {
        mHandler.removeMessages(MSG_WRITE_CONFIG);
        mHandler.sendEmptyMessageDelayed(MSG_WRITE_CONFIG, 5000);
    }
```
继续看！

## 3.2 register DisplayListener

mDisplayListener 用于监听屏幕的状态！
```java
    private final DisplayManager.DisplayListener mDisplayListener
            = new DisplayManager.DisplayListener() {
        @Override public void onDisplayAdded(int displayId) {
        }

        @Override public void onDisplayRemoved(int displayId) {
        }

        @Override public void onDisplayChanged(int displayId) {
            if (displayId == Display.DEFAULT_DISPLAY) {
                synchronized (DeviceIdleController.this) {
                    //【3.1.1】当屏幕状态发生变化后，onDisplayChanged 方法被触发！
                    // 执行了 DeviceIdleController.updateDisplayLocked 方法！
                    updateDisplayLocked();
                }
            }
        }
    };
```


### 3.2.1 DeviceIdleController.updateDisplayLocked

updateDisplayLocked 用于更新屏幕的状态！
```java
    void updateDisplayLocked() {
        //【1】获取屏幕当前的状态，并判断是否是亮屏状态！
        mCurDisplay = mDisplayManager.getDisplay(Display.DEFAULT_DISPLAY);

        boolean screenOn = mCurDisplay.getState() == Display.STATE_ON;
        if (DEBUG) Slog.d(TAG, "updateDisplayLocked: screenOn=" + screenOn);
        if (!screenOn && mScreenOn) {
            //【2.1】如果是从亮屏转为熄屏，设置 mScreenOn 为 false！
            mScreenOn = false;
            if (!mForceIdle) {
                becomeInactiveIfAppropriateLocked(); // 进入 Doze 模式；
            }
        } else if (screenOn) {
            //【2.2】如果是从熄屏转亮屏，设置 mScreenOn 为 true！
            mScreenOn = true;
            if (!mForceIdle) {
                becomeActiveLocked("screen", Process.myUid()); // 退出 Doze 模式；
            }
        }
    }
```
mForceIdle 表示是否强制进入 idle 状态，默认为 false 的，目前唯一的开启方式是通过 adb shell，执行 dumpsys 命令，触发 force-idle，force-inactive 相关指令，强制进入 idle 状态！！

这里看到，当熄屏后，会调用 becomeInactiveIfAppropriateLocked 方法，进入 doze 模式；当亮屏后，会调用 becomeActiveLocked 方法，退出 doze！

## 3.3 DeviceIdleController.updateConnectivityState

更新网络状态！
```java
    void updateConnectivityState(Intent connIntent) {
        ConnectivityService cm;
        synchronized (this) {
            cm = mConnectivityService;
        }
        if (cm == null) {
            return;
        }
        NetworkInfo ni = cm.getActiveNetworkInfo();
        synchronized (this) {
            boolean conn;
            if (ni == null) {
                //【1】网络断开，conn 为 false；
                conn = false;
            } else {
                //【2】获得网络的连接状态；
                if (connIntent == null) {
                    conn = ni.isConnected();
                } else {
                    final int networkType =
                            connIntent.getIntExtra(ConnectivityManager.EXTRA_NETWORK_TYPE,
                                    ConnectivityManager.TYPE_NONE);
                    if (ni.getType() != networkType) {
                        return;
                    }
                    conn = !connIntent.getBooleanExtra(ConnectivityManager.EXTRA_NO_CONNECTIVITY,
                            false);
                }
            }
            //【3】处理连接状态，如果当前状态和之前的状态发生了变化，更新 mNetworkConnected 的值
            if (conn != mNetworkConnected) {
                mNetworkConnected = conn;
                //【4】如果本次状态是处于连接中，并且 light idle 正在等待网络，那就继续处理状态！！
                if (conn && mLightState == LIGHT_STATE_WAITING_FOR_NETWORK) {
                    stepLightIdleStateLocked("network");
                }
            }
        }
    }
```

可以看到 DeviceIdleController 的启动流程还是很简单的！

# 4 启动总结

我们通过一张图来看看 DeviceIdleController 的整个启动过程！


![DeviceIdleController启动流程.png-75kB](https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/23/DeviceIdleController启动流程.png)


其他优秀博客！！

https://blog.csdn.net/kc58236582/article/details/54923406
https://www.cnblogs.com/gavanwanggw/p/7327832.html



  

 
