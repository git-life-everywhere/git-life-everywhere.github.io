# AlarmManager第 4 篇 - AlarmManagerService属性总结
title: AlarmManager第 4 篇 - AlarmManagerService属性总结
date: 2017/09/27 20:46:25 
categories: 
- AndroidFramework源码分析
- AlarmManager闹钟管理
tags: AlarmManager闹钟管理
---

首先我们来看看 AlarmManagerService 中的一些成员变量：

1、首先是一些二进制标志位：

```java
    //【1】alarm 的类型，用二进制表示！
    private static final int RTC_WAKEUP_MASK = 1 << RTC_WAKEUP; // 1 << 0
    private static final int RTC_MASK = 1 << RTC;  // 1 << 1
    private static final int ELAPSED_REALTIME_WAKEUP_MASK = 1 << ELAPSED_REALTIME_WAKEUP; // // 1 << 2
    private static final int ELAPSED_REALTIME_MASK = 1 << ELAPSED_REALTIME;  // 1 << 3
    
    // 时间是否改变
    static final int TIME_CHANGED_MASK = 1 << 16;
    
    // 该 alarm 是否是 wake up 类型的
    static final int IS_WAKEUP_MASK = RTC_WAKEUP_MASK|ELAPSED_REALTIME_WAKEUP_MASK;

    // Mask for testing whether a given alarm type is wakeup vs non-wakeup
    static final int TYPE_NONWAKEUP_MASK = 0x1; // low bit => non-wakeup
```
RTC_WAKEUP，RTC，ELAPSED_REALTIME_WAKEUP，ELAPSED_REALTIME 定义在 AlarmManager 中，其代表了 4 种 alarm 类型！


```java
    // 这种 alarm 的时间计算方法为 System.currentTimeMillis()，不包括休眠时间！
    // 该 alarm 触发会唤醒设备！
    public static final int RTC_WAKEUP = 0;

    // 这种 alarm 的时间计算方法为 System.currentTimeMillis()，不包括休眠时间！
    // 这种类型的 alarm 不会唤醒设备，如果其在设备休眠时候触发，直到设备被唤醒后，才被被分发出去！
    public static final int RTC = 1;

    // 这种 alarm 的时间计算方法为 SystemClock.elapsedRealtime()，包括休眠时间！
    // 这种类型的 alarm 触发时会唤醒设备；
    public static final int ELAPSED_REALTIME_WAKEUP = 2;

    // 这种 alarm 的时间计算方法为 SystemClock.elapsedRealtime()，包括休眠时间！
    // 这种类型的 alarm 不会唤醒设备，如果其在设备休眠时候触发，直到设备下次被唤醒后，才被被分发出去！
    public static final int ELAPSED_REALTIME = 3;
```
2、和 debug 相关的 alarm，这里不关注！

```java
    static final String TAG = "AlarmManager";
    static final boolean localLOGV = false;
    static final boolean DEBUG_BATCH = localLOGV || false;
    static final boolean DEBUG_VALIDATE = localLOGV || false;
    static final boolean DEBUG_ALARM_CLOCK = localLOGV || false;
    static final boolean DEBUG_LISTENER_CALLBACK = localLOGV || false;
    
    final LocalLog mLog = new LocalLog(TAG);
    
    boolean mLastWakeLockUnimportantForLogging;
    
    // 用于 dump wake up 状态信息！
    static final boolean WAKEUP_STATS = false;
    
```

3、广播以及关闭广播接受者

```java
    // 后台广播，用于分发 alarm！
    private final Intent mBackgroundIntent
            = new Intent().addFlags(Intent.FLAG_FROM_BACKGROUND);
            
    // 下一个 alarm 时间发生改变的广播！
    private static final Intent NEXT_ALARM_CLOCK_CHANGED_INTENT =
            new Intent(AlarmManager.ACTION_NEXT_ALARM_CLOCK_CHANGED)
                    .addFlags(Intent.FLAG_RECEIVER_REPLACE_PENDING);
                    
    // 用于监听时间改变和日期改变的广播！
    ClockReceiver mClockReceiver;
    
    // 用于监听熄屏亮屏广播
    InteractiveStateReceiver mInteractiveStateReceiver;

    // 用于监听 package 相关的广播！
    private UninstallReceiver mUninstallReceiver;
    
    PendingIntent mTimeTickSender; // 时间改变的广播；
    PendingIntent mDateChangeSender; // 日期改变的广播；
```

4、时间相关

```java
    private long mNextWakeup; // 下一个包含 wakeup 的 batch 的开始时间 
    private long mNextNonWakeup; // 下一个包含 no wake up 的 batch 的开始时间  
    private long mLastWakeupSet;
    private long mLastWakeup;

    boolean mInteractive = true; // 熄屏亮屏状态，亮屏为 true

    long mNonInteractiveStartTime; // 熄灭屏幕的开始时间；
    long mNonInteractiveTime; // 熄灭屏幕的最长时间间隔；
    
    long mLastAlarmDeliveryTime; // 上一个 alarm 的分发时间！
    long mStartCurrentDelayTime; // 需要延迟执行的非 wake up 类型的 alarm 开始延迟的时间！
    long mNextNonWakeupDeliveryTime; // 下一次非 wake up 类型的 alarm 分发时间！
    
    long mLastTimeChangeClockTime; // 上一次时间发生改变的时间点，取值为 System.currentTimeMillis()；
    long mLastTimeChangeRealtime; // 上一次时间发生改变的时间点，取值为 SystemClock.elapsedRealtime()；
    
    long mAllowWhileIdleMinTime; // 可以开始执行 flag 为 ALLOW_WHILE_IDLE 的 alarm 的最小时间间隔！
    
    long mTotalDelayTime = 0;
    long mMaxDelayTime = 0; //
    
    int mNumTimeChanged; // 时间改变的次数！
    int mNumDelayedAlarms = 0;
```

5、集合
```java
    // 延迟执行的 no wake up 类型的 alarm 列表！
    ArrayList<Alarm> mPendingNonWakeupAlarms = new ArrayList<>();
    
    // 正在执行中的 alarm 列表！
    ArrayList<InFlight> mInFlight = new ArrayList<>();
    
    /**
     * For each uid, this is the last time we dispatched an "allow while idle" alarm,
     * used to determine the earliest we can dispatch the next such alarm.
     */
    final SparseLongArray mLastAllowWhileIdleDispatch = new SparseLongArray();
    
    final ArrayList<IdleDispatchEntry> mAllowWhileIdleDispatches = new ArrayList();
    
    private final SparseArray<AlarmManager.AlarmClockInfo> mNextAlarmClockForUser =
            new SparseArray<>();
            
    private final SparseArray<AlarmManager.AlarmClockInfo> mTmpSparseAlarmClockArray =
            new SparseArray<>();
            
    private final SparseBooleanArray mPendingSendNextAlarmClockChangedForUser =
            new SparseBooleanArray();
            
    /**
     * The current set of user whitelisted apps for device idle mode, meaning these are allowed
     * to freely schedule alarms.
     */
    int[] mDeviceIdleUserWhitelist = new int[0];
    
    ArrayList<Alarm> mPendingWhileIdleAlarms = new ArrayList<>();
    
    final SparseArray<ArrayMap<String, BroadcastStats>> mBroadcastStats
            = new SparseArray<ArrayMap<String, BroadcastStats>>();
            
    // May only use on mHandler's thread, locking not required.
    private final SparseArray<AlarmManager.AlarmClockInfo> mHandlerSparseAlarmClockArray =
            new SparseArray<>();
            
    final LinkedList<WakeupEvent> mRecentWakeups = new LinkedList<WakeupEvent>();
    
    final HashMap<String, PriorityClass> mPriorities = new HashMap<>();
    
    // // alarm批处理对象列表  
    final ArrayList<Batch> mAlarmBatches = new ArrayList<>();
```

6、比较器

```java
    // 时间比较器！
    static final IncreasingTimeOrder sIncreasingTimeOrder = new IncreasingTimeOrder();
    
    static final BatchTimeOrder sBatchOrder = new BatchTimeOrder();
    
    final Comparator<Alarm> mAlarmDispatchComparator = new Comparator<Alarm>() {
        @Override
        public int compare(Alarm lhs, Alarm rhs) {
            // priority class trumps everything.  TICK < WAKEUP < NORMAL
            if (lhs.priorityClass.priority < rhs.priorityClass.priority) {
                return -1;
            } else if (lhs.priorityClass.priority > rhs.priorityClass.priority) {
                return 1;
            }

            // within each class, sort by nominal delivery time
            if (lhs.whenElapsed < rhs.whenElapsed) {
                return -1;
            } else if (lhs.whenElapsed > rhs.whenElapsed) {
                return 1;
            }

            // same priority class + same target delivery time
            return 0;
        }
    };
```

```java
    static final boolean RECORD_ALARMS_IN_HISTORY = true; // 是否将 alarm 保存到历史信息中！
    static final boolean RECORD_DEVICE_IDLE_ALARMS = false;
    
    // 消息 id，AlarmHandler 处理！
    static final int ALARM_EVENT = 1;
    
    // 系统属性用于获取时区信息，更新 kernel 的时区信息！
    static final String TIMEZONE_PROPERTY = "persist.sys.timezone";

    // appOps 管理对象！
    AppOpsManager mAppOps;
    // doze 模式
    DeviceIdleController.LocalService mLocalDeviceIdleController;

    // 锁对象，访问 mConstants 的时候，必须持有锁！
    final Object mLock = new Object();

    // alarm 驱动指针！
    long mNativeData;
    
    // 用于标记当前正在分发处理的 alarm 个数！
    int mBroadcastRefCount = 0;
    
    PowerManager.WakeLock mWakeLock;
    
    final AlarmHandler mHandler = new AlarmHandler();
    
    // alarm 分发监控器
    final DeliveryTracker mDeliveryTracker = new DeliveryTracker();
    
    Random mRandom;

    /**
     * Broadcast options to use for FLAG_ALLOW_WHILE_IDLE.
     */
    Bundle mIdleOptions;

    private boolean mNextAlarmClockMayChange; // 下一个 alarm 的触发时间是否会改变！

    final Constants mConstants; // 常量类！

    // Alarm delivery ordering bookkeeping
    static final int PRIO_TICK = 0;
    static final int PRIO_WAKEUP = 1;
    static final int PRIO_NORMAL = 2;

    int mCurrentSeq = 0;

    final long RECENT_WAKEUP_PERIOD = 1000L * 60 * 60 * 24; // one day
    
    // minimum recurrence period or alarm futurity for us to be able to fuzz it
    static final long MIN_FUZZABLE_INTERVAL = 10000;

    // set to null if in idle mode; while in this mode, any alarms we don't want
    // to run during this time are placed in mPendingWhileIdleAlarms
    Alarm mPendingIdleUntil = null;

    // 下一个请求退出 idle 模式的 alarm  
    Alarm mNextWakeFromIdle = null;
```

以上的只是对主要的变量进行了解释和说明，其具体的作用，我们后续在分析 alarm manager 机制的时候会分析！