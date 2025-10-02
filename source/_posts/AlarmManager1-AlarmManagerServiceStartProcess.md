# AlarmManager第 1 篇 - AlarmManagerService启动
title: AlarmManager第 1 篇 - AlarmManagerService启动
date: 2017/05/23 20:46:25 
categories: 
- AndroidFramework源码分析
- AlarmManager闹钟管理
tags: AlarmManager闹钟管理
copyright: true
---

基于 Android7.1.1 源码，分析 AlarmManagerService 的启动流程，这里我们重点关注和分析 java 层的逻辑实现！

```java
    private void startOtherServices() {
        ... ... ...
        traceBeginAndSlog("StartAlarmManagerService");

        //【1】启动 AlarmManagerService
        mSystemServiceManager.startService(AlarmManagerService.class);
        Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);
        ... ... ...
    }
```

首先会创建 AlarmManagerService 实例对象，然后会调用其 onStart 方法！

# 1 Constructor

```java
    public AlarmManagerService(Context context) {
        super(context);
        // 创建了常量实例，用于存储其所需的一些常量！
        mConstants = new Constants(mHandler);
    }
```
AlarmManagerService 的构造器很简单，只是创建了一个 Constants 实例对象！

## 1.1 new Constants

Constants 用来保存 alarm manger 需要的一些常量数据，其内部有一些和属性值！

该类中所有的时间单位都是毫秒，这些常量的值和系统全局设置 Settings.Global 中的值保持同步，任何访问该类或该类中的字段都要持有 AlarmManagerService.mLock 锁！

首先，是一些  Settings.Global 中的 key 值，用于将数据保存到 Settings 中！
```java
        private static final String KEY_MIN_FUTURITY = "min_futurity";
        private static final String KEY_MIN_INTERVAL = "min_interval";
        private static final String KEY_ALLOW_WHILE_IDLE_SHORT_TIME = "allow_while_idle_short_time";
        private static final String KEY_ALLOW_WHILE_IDLE_LONG_TIME = "allow_while_idle_long_time";
        private static final String KEY_ALLOW_WHILE_IDLE_WHITELIST_DURATION
                = "allow_while_idle_whitelist_duration";
        private static final String KEY_LISTENER_TIMEOUT = "listener_timeout";
```
接着是一些 default 值，用于初始化成员变量！
```java
        private static final long DEFAULT_MIN_FUTURITY = 5 * 1000; // 默认的最小触发时间： 5s
        private static final long DEFAULT_MIN_INTERVAL = 60 * 1000; // 默认最小时间间隔； 60s!
        private static final long DEFAULT_ALLOW_WHILE_IDLE_SHORT_TIME = DEFAULT_MIN_FUTURITY;
        private static final long DEFAULT_ALLOW_WHILE_IDLE_LONG_TIME = 9*60*1000;
        private static final long DEFAULT_ALLOW_WHILE_IDLE_WHITELIST_DURATION = 10*1000;
    
        private static final long DEFAULT_LISTENER_TIMEOUT = 5 * 1000; // 默认的监听超时时间：5s
```
最后是一些 alarm manager 会用到的一些成员变量：
```java
        // Minimum futurity of a new alarm
        public long MIN_FUTURITY = DEFAULT_MIN_FUTURITY;

        // alarm 重复执行的最小时间间隔！
        public long MIN_INTERVAL = DEFAULT_MIN_INTERVAL;

        // Minimum time between ALLOW_WHILE_IDLE alarms when system is not idle.
        public long ALLOW_WHILE_IDLE_SHORT_TIME = DEFAULT_ALLOW_WHILE_IDLE_SHORT_TIME;

        // Minimum time between ALLOW_WHILE_IDLE alarms when system is idling.
        public long ALLOW_WHILE_IDLE_LONG_TIME = DEFAULT_ALLOW_WHILE_IDLE_LONG_TIME;

        // BroadcastOptions.setTemporaryAppWhitelistDuration() to use for FLAG_ALLOW_WHILE_IDLE.
        public long ALLOW_WHILE_IDLE_WHITELIST_DURATION
                = DEFAULT_ALLOW_WHILE_IDLE_WHITELIST_DURATION;

        // Direct alarm listener callback timeout
        // 闹钟监听回调超时时间，初始化为 5s;
        public long LISTENER_TIMEOUT = DEFAULT_LISTENER_TIMEOUT;

        private ContentResolver mResolver; // 用于访问设置的全局数据库！
        private final KeyValueListParser mParser = new KeyValueListParser(',');
        private long mLastAllowWhileIdleWhitelistDuration = -1;
```


接着，去看看 Constants 的构造器！

```java
    public Constants(Handler handler) {
        super(handler);
        // 更新 idle 相关的时间值
        updateAllowWhileIdleMinTimeLocked();
        updateAllowWhileIdleWhitelistDurationLocked();
    }
```

## 1.2 Constructor.updateAllowWhileIdleMinTimeLocked

这里调用了两个方法：

```java
        public void updateAllowWhileIdleMinTimeLocked() {
            //【1】更新 allow while idle 类型的 alarm 的最小触发时间点！
            mAllowWhileIdleMinTime = mPendingIdleUntil != null
                    ? ALLOW_WHILE_IDLE_LONG_TIME : ALLOW_WHILE_IDLE_SHORT_TIME;
        }
```
如果此时已经进入了 idle 状态，那么触发的最小时间间隔为：ALLOW_WHILE_IDLE_LONG_TIME，否则为 ALLOW_WHILE_IDLE_SHORT_TIME！

## 1.3 Constructor.updateAllowWhileIdleWhitelistDurationLocked
```java
        public void updateAllowWhileIdleWhitelistDurationLocked() {
            //【2】更新 mLastAllowWhileIdleWhitelistDuration 变量！
            if (mLastAllowWhileIdleWhitelistDuration != ALLOW_WHILE_IDLE_WHITELIST_DURATION) {
                mLastAllowWhileIdleWhitelistDuration = ALLOW_WHILE_IDLE_WHITELIST_DURATION;
                BroadcastOptions opts = BroadcastOptions.makeBasic();
                opts.setTemporaryAppWhitelistDuration(ALLOW_WHILE_IDLE_WHITELIST_DURATION);
                mIdleOptions = opts.toBundle();
            }
        }
```
继续来看，该方法用于设置 system 更新 idle whilde list 的时间间隔，目前是 10 × 1000 毫秒！

# 2 onStart

接着是重点 onStart 方法：

```java
    @Override
    public void onStart() {
        //【2.1】native 层的初始化操作，打开 alarm 驱动，并返回驱动文件句柄！
        mNativeData = init();
        mNextWakeup = mNextNonWakeup = 0;

        //【2.2】设置时区给 kernel，因为 kernel 重启后不会保存时区信息！
        setTimeZoneImpl(SystemProperties.get(TIMEZONE_PROPERTY));

        PowerManager pm = (PowerManager) getContext().getSystemService(Context.POWER_SERVICE);
        mWakeLock = pm.newWakeLock(PowerManager.PARTIAL_WAKE_LOCK, "*alarm*");
        
        // 创建时间发生改变的广播！
        mTimeTickSender = PendingIntent.getBroadcastAsUser(getContext(), 0,
                new Intent(Intent.ACTION_TIME_TICK).addFlags(
                        Intent.FLAG_RECEIVER_REGISTERED_ONLY
                        | Intent.FLAG_RECEIVER_FOREGROUND), 0,
                        UserHandle.ALL);
                        
        // 创建日期发生改变的广播！
        Intent intent = new Intent(Intent.ACTION_DATE_CHANGED);
        intent.addFlags(Intent.FLAG_RECEIVER_REPLACE_PENDING);
        mDateChangeSender = PendingIntent.getBroadcastAsUser(getContext(), 0, intent,
                Intent.FLAG_RECEIVER_REGISTERED_ONLY_BEFORE_BOOT, UserHandle.ALL);
        
        //【2.2】创建一个 ClockReceiver 广播接收者，用于接收 Intent.ACTION_DATE_CHANGED 和 Intent.ACTION_TIME_TICK 广播！
        mClockReceiver = new ClockReceiver();
        // 设置时间改变和日期改变的 Alarm！
        mClockReceiver.scheduleTimeTickEvent();
        mClockReceiver.scheduleDateChangedEvent();
        
        //【2.3】创建监听熄屏亮屏广播的接收者！
        mInteractiveStateReceiver = new InteractiveStateReceiver();
        mUninstallReceiver = new UninstallReceiver();
        
        //【2.4】如果成功打开了驱动文件句柄，那就创建一个 AlarmThread，监控驱动文件！
        // 如果无法打开 Alarm 驱动，那就是用 AlarmHandler 对象！
        if (mNativeData != 0) {
            AlarmThread waitThread = new AlarmThread();
            waitThread.start();
        } else {
            Slog.w(TAG, "Failed to open alarm driver. Falling back to a handler.");
        }

        try {
            // 注册 uid 观察者！
            ActivityManagerNative.getDefault().registerUidObserver(new UidObserver(),
                    ActivityManager.UID_OBSERVER_IDLE);
        } catch (RemoteException e) {
            // ignored; both services live in system_server
        }

        // 将 AlarmManagerService 注册到 SystemService 中！
        publishBinderService(Context.ALARM_SERVICE, mService);
        publishLocalService(LocalService.class, new LocalService());
    }
```

下面我们来具体分析下：

## 2.1 init -> android_server_AlarmManagerService_init

```java
private native long init();
```
可以看到，这是一个 native 方法，方法的定义在：android/frameworks/base/services/core/jni/com_android_server_AlarmManagerService.cpp 中！

最终会通过 systemCall 进入 native 层，调用 android_server_AlarmManagerService_init 方法！

我们来看看 init 最终做了什么：
```java
static jlong android_server_AlarmManagerService_init(JNIEnv*, jobject)
{
    //【2.1.1】初始化 alarm 驱动！
    jlong ret = init_alarm_driver();
    if (ret) {
        return ret;
    }
    
    //【2.1.2】初始化定时器 fd！
    return init_timerfd();
}
```
这里我们不过多关注！

## 2.2 setTimeZoneImpl

更新时区信息，把当前时区保存到内核中！
```java
    void setTimeZoneImpl(String tz) {
        if (TextUtils.isEmpty(tz)) {
            return;
        }

        // 获取最新的时区信息！ 
        TimeZone zone = TimeZone.getTimeZone(tz);

        // 防止重复写入
        boolean timeZoneWasChanged = false;
        synchronized (this) {
            // 通过系统属性获得已经保存的时区信息；
            String current = SystemProperties.get(TIMEZONE_PROPERTY);
            
            // 如果 current 不等于 zone.getID()。说明时区信息发生了变化！
            if (current == null || !current.equals(zone.getID())) {
                if (localLOGV) {
                    Slog.v(TAG, "timezone changed: " + current + ", new=" + zone.getID());
                }
                timeZoneWasChanged = true;
                // 更新系统属性，应用程序会读取该属性！
                SystemProperties.set(TIMEZONE_PROPERTY, zone.getID());
            }

            // 更新内核时区信息  
             // 内核跟踪时间偏移为GMT以西的分钟数 
            int gmtOffset = zone.getOffset(System.currentTimeMillis());
            setKernelTimezone(mNativeData, -(gmtOffset / 60000));
        }

        TimeZone.setDefault(null);

        // 如果时区发生了变化，那就发送 Intent.ACTION_TIMEZONE_CHANGED 的广播！
        if (timeZoneWasChanged) {
            Intent intent = new Intent(Intent.ACTION_TIMEZONE_CHANGED);
            intent.addFlags(Intent.FLAG_RECEIVER_REPLACE_PENDING);
            intent.putExtra("time-zone", zone.getID());
            getContext().sendBroadcastAsUser(intent, UserHandle.ALL);
        }
    }
```
这里的 setKernelTimezone 是一个 native 方法，这里我们先不看！

```java
private native int setKernelTimezone(long nativeData, int minuteswest);
```
更新 kernel 时区的信息的原因是，kernel 在重启后，并不会保存时区信息！

## 2.3 ClockReceiver

我们来看看 ClockReceiver，是一个动态注册的 BroadcastReceiver，用于接收 Intent.ACTION_TIME_TICK 时间改变和 Intent.ACTION_DATE_CHANGED 日期改变的广播！

```java
    class ClockReceiver extends BroadcastReceiver {
    
        public ClockReceiver() {
            // 监听 Intent.ACTION_TIME_TICK 和 Intent.ACTION_DATE_CHANGED 的广播！
            IntentFilter filter = new IntentFilter();
            filter.addAction(Intent.ACTION_TIME_TICK);
            filter.addAction(Intent.ACTION_DATE_CHANGED);
            getContext().registerReceiver(this, filter);
        }
        
        @Override
        public void onReceive(Context context, Intent intent) {
            if (intent.getAction().equals(Intent.ACTION_TIME_TICK)) {
                if (DEBUG_BATCH) {
                    Slog.v(TAG, "Received TIME_TICK alarm; rescheduling");
                }
                // 接收到 Intent.ACTION_TIME_TICK 广播后，执行 scheduleTimeTickEvent！
                scheduleTimeTickEvent();

            } else if (intent.getAction().equals(Intent.ACTION_DATE_CHANGED)) {
                // 因为 kernel 无法对 DST 进行监控，基于当前时区的 gmt 偏移 + 用户空间跟踪保存的信息来重置时区信息！
                TimeZone zone = TimeZone.getTimeZone(SystemProperties.get(TIMEZONE_PROPERTY));
                int gmtOffset = zone.getOffset(System.currentTimeMillis());
                setKernelTimezone(mNativeData, -(gmtOffset / 60000));

                // 接收到 Intent.ACTION_DATE_CHANGED 广播后，执行 scheduleDateChangedEvent！
                scheduleDateChangedEvent();
            }
        }
        
        ... ... ... ...
    }
```
可以看到：
- 当收到 Intent.ACTION_TIME_TICK 后，触发 scheduleTimeTickEvent 方法；
- 当收到 Intent.ACTION_DATE_CHANGED，触发 scheduleDateChangedEvent 方法：

### 2.3.1 ClockReceiver.scheduleTimeTickEvent

我们来看下 scheduleTimeTickEvent，该方法每一分钟，就会给所有接收 ACTION_TIME_TICK 广播的接收者发送广播，注意这里也包括自身！

```java
        public void scheduleTimeTickEvent() {
            //【1】获得当前时间对应的毫秒值！
            final long currentTime = System.currentTimeMillis();
            //【2】计算当前时间过 1 分钟后的毫秒值！
            final long nextTime = 60000 * ((currentTime / 60000) + 1);

            // Schedule this event for the amount of time that it would take to get to
            // the top of the next minute.
            final long tickEventDelay = nextTime - currentTime;

            final WorkSource workSource = null; // Let system take blame for time tick events.

            //【3】设置 alarm！
            setImpl(ELAPSED_REALTIME, SystemClock.elapsedRealtime() + tickEventDelay, 0,
                    0, mTimeTickSender, null, null, AlarmManager.FLAG_STANDALONE, workSource,
                    null, Process.myUid(), "android");
        }
```
可以看出 scheduleTimeTickEvent 方法会设置一个 alarm，距离当前时间的下一秒，这个 alarm 会发送 Intent.ACTION_TIME_TICK 广播！

然后 ClockReceiver 会接收到这个广播，然后继续设置下一秒的，发送 Intent.ACTION_TIME_TICK 广播的 alarm，如此循环下去！

### 2.3.2 ClockReceiver.scheduleDateChangedEvent

我们来看下 scheduleDateChangedEvent：

```java
        public void scheduleDateChangedEvent() {
            //【1】计算下一天的 0 点整所对应的 alarm！
            Calendar calendar = Calendar.getInstance();
            calendar.setTimeInMillis(System.currentTimeMillis());
            calendar.set(Calendar.HOUR_OF_DAY, 0);
            calendar.set(Calendar.MINUTE, 0);
            calendar.set(Calendar.SECOND, 0);
            calendar.set(Calendar.MILLISECOND, 0);
            calendar.add(Calendar.DAY_OF_MONTH, 1);

            final WorkSource workSource = null; // Let system take blame for date change events.

            //【2】设置 alarm！
            setImpl(RTC, calendar.getTimeInMillis(), 0, 0, mDateChangeSender, null, null,
                    AlarmManager.FLAG_STANDALONE, workSource, null,
                    Process.myUid(), "android");
        }
```

可以看出 scheduleDateChangedEvent 方法会设置一个 alarm，触发事件是下一天的 0 点整，这个 alarm 会发送 Intent.ACTION_DATE_CHANGED 广播！

然后 ClockReceiver 会接收到这个广播，然后继续设置下一天 0 点整的，发送 Intent.Intent.ACTION_DATE_CHANGED 广播的 alarm，如此循环下去！

我们可以看到这里涉及到了一个方法 **setImpl**，这个方法我们后续会分析，这里先简单认为其设置了一个 alarm！

## 2.4 InteractiveStateReceiver

InteractiveStateReceiver 用于监控熄屏亮屏的广播！

```java
    class InteractiveStateReceiver extends BroadcastReceiver {
        public InteractiveStateReceiver() {
            IntentFilter filter = new IntentFilter();
            filter.addAction(Intent.ACTION_SCREEN_OFF);
            filter.addAction(Intent.ACTION_SCREEN_ON);
            filter.setPriority(IntentFilter.SYSTEM_HIGH_PRIORITY);
            getContext().registerReceiver(this, filter);
        }

        @Override
        public void onReceive(Context context, Intent intent) {
            synchronized (mLock) {
                //【2.2.1】处理熄屏亮屏广播！
                interactiveStateChangedLocked(Intent.ACTION_SCREEN_ON.equals(intent.getAction()));
            }
        }
    }
```
监听的 Alarm：

- Intent.ACTION_SCREEN_OFF
- Intent.ACTION_SCREEN_ON

继续来看：

### 2.4.1 interactiveStateChangedLocked

这里说下参数 interactive，表示接收到的是否亮屏广播：

```java
Intent.ACTION_SCREEN_ON.equals(intent.getAction())
```
成员变量 mInteractive 用来保存上一次的熄屏亮屏状态！

接下来，我们来看看 interactiveStateChangedLocked 方法：

```java
    void interactiveStateChangedLocked(boolean interactive) {
        if (mInteractive != interactive) { // 不相等，说明熄屏亮屏状态发生了变化！
            mInteractive = interactive; // 更新 mInteractive
            
            // 获得自开机后，经过的时间，包括深度睡眠的时间！
            final long nowELAPSED = SystemClock.elapsedRealtime();

            if (interactive) { 
            // 如果从 screen off 变为了 screen on，说明此时是亮屏状态！
                if (mPendingNonWakeupAlarms.size() > 0) {
                    final long thisDelayTime = nowELAPSED - mStartCurrentDelayTime;
                    mTotalDelayTime += thisDelayTime;
                    if (mMaxDelayTime < thisDelayTime) {
                        mMaxDelayTime = thisDelayTime;
                    }
                    // 立刻分发那些等待中的 no wake up 类型的 alarm！
                    deliverAlarmsLocked(mPendingNonWakeupAlarms, nowELAPSED);
                    mPendingNonWakeupAlarms.clear();
                }

                // 如果 mNonInteractiveStartTime 大于 0，那么就计算上次处于熄屏的时间间隔，保存到 mNonInteractiveTime！
                if (mNonInteractiveStartTime > 0) {
                    long dur = nowELAPSED - mNonInteractiveStartTime;
                    if (dur > mNonInteractiveTime) {
                        mNonInteractiveTime = dur;
                    }
                }
            } else {
            // 如果从 screen on 变为了 screen off，说明此时是熄屏状态，记录熄屏开始时间！
                mNonInteractiveStartTime = nowELAPSED;
            }
        }
    }
```


# 3 onBootPhase

在启动的最后阶段，system ready 的时候，会调用 AlarmManagerService 的 onBootPhase 方法！

```java
    @Override
    public void onBootPhase(int phase) {
        if (phase == PHASE_SYSTEM_SERVICES_READY) {
            //【3.1】调用 Constants 的 start 方法！
            mConstants.start(getContext().getContentResolver());
            // 获得 AppOps 管理对象!
            mAppOps = (AppOpsManager) getContext().getSystemService(Context.APP_OPS_SERVICE);
            // 获得 DeviceIdleController 对象！
            mLocalDeviceIdleController
                    = LocalServices.getService(DeviceIdleController.LocalService.class);
        }
    }
```
我们来去看看 Constants.start 方法：

## 3.1 Constants.start

```java
        public void start(ContentResolver resolver) {
            //【1】获得系统数据库的 ContentResolver！
            mResolver = resolver;
            //【2】监听 alarm_manager_constants 数据库的变化！
            mResolver.registerContentObserver(Settings.Global.getUriFor(
                    Settings.Global.ALARM_MANAGER_CONSTANTS), false, this);
            //【3】启动时，默认更新一次 Constants；
            updateConstants();
        }
```
当 Settings.Global.ALARM_MANAGER_CONSTANTS 的数据发生变化后，会触发 Constants 的回调：

```java
        @Override
        public void onChange(boolean selfChange, Uri uri) {
            //【3.2】调用 updateConstants 更新
            updateConstants();
        }
```

## 3.2 Constants.updateConstants

更新 Constants 中的属性值！
```java
        private void updateConstants() {
            synchronized (mLock) {
                //【1】读取 Settings 的 Global 数据库中的 alarm_manager_constants 项数据！
                // 更新 Constants 内部变量！
                try {
                    mParser.setString(Settings.Global.getString(mResolver,
                            Settings.Global.ALARM_MANAGER_CONSTANTS));
                } catch (IllegalArgumentException e) {
                    // Failed to parse the settings string, log this and move on
                    // with defaults.
                    Slog.e(TAG, "Bad device idle settings", e);
                }

                MIN_FUTURITY = mParser.getLong(KEY_MIN_FUTURITY, DEFAULT_MIN_FUTURITY);
                MIN_INTERVAL = mParser.getLong(KEY_MIN_INTERVAL, DEFAULT_MIN_INTERVAL);
                ALLOW_WHILE_IDLE_SHORT_TIME = mParser.getLong(KEY_ALLOW_WHILE_IDLE_SHORT_TIME,
                        DEFAULT_ALLOW_WHILE_IDLE_SHORT_TIME);
                ALLOW_WHILE_IDLE_LONG_TIME = mParser.getLong(KEY_ALLOW_WHILE_IDLE_LONG_TIME,
                        DEFAULT_ALLOW_WHILE_IDLE_LONG_TIME);
                ALLOW_WHILE_IDLE_WHITELIST_DURATION = mParser.getLong(
                        KEY_ALLOW_WHILE_IDLE_WHITELIST_DURATION,
                        DEFAULT_ALLOW_WHILE_IDLE_WHITELIST_DURATION);
                LISTENER_TIMEOUT = mParser.getLong(KEY_LISTENER_TIMEOUT,
                        DEFAULT_LISTENER_TIMEOUT);

                updateAllowWhileIdleMinTimeLocked();
                updateAllowWhileIdleWhitelistDurationLocked();
            }
        }
```











Alarm 是 AlarmManagerService 的一个内部类 Alarm，所有应用在设置 Alarm 的时候，都会在 setImplLocked 函数中将 Alarm 格式化为内部类 Alarm 的格式，定义截选如下：

    private static class Alarm {

    public int type;

    public int count;

    public long when;

    public long repeatInterval;

    public PendingIntent operation;

    public int uid;

    public int pid;

其中记录了逻辑闹钟的一些关键信息。 

type域：记录着逻辑闹钟的闹钟类型，比如RTC_WAKEUP、ELAPSED_REALTIME_WAKEUP等；

count域：是个辅助域，它和repeatInterval域一起工作。当repeatInterval大于0时，这个域可被用于计算下一次重复激发alarm的时间。

when域：记录闹钟的激发时间。这个域和type域相关，详细情况见后文；

repeatInterval域：表示重复激发闹钟的时间间隔，如果闹钟只需激发一次，则此域为0，如果闹钟需要重复激发，此域为以毫秒为单位的时间间隔；

operation域：记录闹钟激发时应该执行的动作，详细情况见后文；

uid域：记录设置闹钟的进程的uid；

pid域：记录设置闹钟的进程的pid。