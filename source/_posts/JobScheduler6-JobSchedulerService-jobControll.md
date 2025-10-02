# JobScheduler第 6 篇 - JobSchedulerService - job controll
title: JobScheduler第 6 篇 - JobSchedulerService - job controll
date: 2016/04/25 20:46:25 
categories: 
- AndroidFramework源码分析
- JobScheduler任务调度
tags: JobScheduler任务调度
---

基于 Android 7.1.1 源码分析，分析 job controller 的实现机制！

# 前言
我们知道，当我们把 job schedule 进入 JobSchedulerService 中后，JobSchdulerService 会拉起它，之前我们分析了用户主动 cancel 和 jobFinished job 时，具体的函数调用，而对于 job 除了受的到用户的主动操作之外，还有 controller 的控制，接下来，我们来看看那这部分代码！

Job 的 controll 都是在 controller 中，controller 的初始化是在 JobSchedulerService 中：
```java
    public JobSchedulerService(Context context) {
        super(context);
        mHandler = new JobHandler(context.getMainLooper());
        mConstants = new Constants(mHandler);
        mJobSchedulerStub = new JobSchedulerStub();
        mJobs = JobStore.initAndGet(this);

        // Create the controllers.
        mControllers = new ArrayList<StateController>();
        mControllers.add(ConnectivityController.get(this));
        mControllers.add(TimeController.get(this));
        mControllers.add(IdleController.get(this));
        mControllers.add(BatteryController.get(this));
        mControllers.add(AppIdleController.get(this));
        mControllers.add(ContentObserverController.get(this));
        mControllers.add(DeviceIdleJobsController.get(this));
    }
```
每一个 Controller 都采用了单例模式，保证了有且仅有一个 Controller，每个 Controller 内部都有一个集合用来管理其所控制的左右 job！

接下来，我们一个一个看：



# 1 ConnectivityController
ConnectivityController 主要是用来根据网络条件来控制 job！
我们先来看构造器：
```java
    private ConnectivityController(StateChangedListener stateChangedListener, Context context,
            Object lock) {
        super(stateChangedListener, context, lock);

        mConnManager = mContext.getSystemService(ConnectivityManager.class);
        mNetPolicyManager = mContext.getSystemService(NetworkPolicyManager.class);

        // 这里很简单，注册了一个广播接收者，监听 ConnectivityManager.CONNECTIVITY_ACTION 广播!
        final IntentFilter intentFilter = new IntentFilter(ConnectivityManager.CONNECTIVITY_ACTION);
        mContext.registerReceiverAsUser(
                mConnectivityReceiver, UserHandle.SYSTEM, intentFilter, null, null);

        // 向网络策略管理服务注册监听器！
        mNetPolicyManager.registerListener(mNetPolicyListener);
    }
```
ConnectivityController 对 job 的控制主要依靠这两个监听器：
## 1.1 CC.ConnectivityReceiver

我们先来看看第一个：
```java
    private BroadcastReceiver mConnectivityReceiver = new BroadcastReceiver() {
        @Override
        public void onReceive(Context context, Intent intent) {
            updateTrackedJobs(-1);
        }
    };
```
这个很简单，当网络断开或连接，会调用 updateTrackedJobs 方法！ 
## 1.2 CC.NetPolicyListener
接着看看网络策略监听器，当网络策略发生变化后，会回调接口：
```java
    private INetworkPolicyListener mNetPolicyListener = new INetworkPolicyListener.Stub() {
        @Override
        public void onUidRulesChanged(int uid, int uidRules) {
            updateTrackedJobs(uid);
        }
    
        @Override
        public void onMeteredIfacesChanged(String[] meteredIfaces) {
            updateTrackedJobs(-1);
        }

        @Override
        public void onRestrictBackgroundChanged(boolean restrictBackground) {
            updateTrackedJobs(-1);
        }

        @Override
        public void onRestrictBackgroundWhitelistChanged(int uid, boolean whitelisted) {
            updateTrackedJobs(uid);
        }

        @Override
        public void onRestrictBackgroundBlacklistChanged(int uid, boolean blacklisted) {
            updateTrackedJobs(uid);
        }
    };
```
可以发现，最终都调用了 updateTrackedJobs 方法：
## 1.3 CC.updateTrackedJobs

通过传入的 uid 的值，来更新具体的 job 状态！
```java
    private void updateTrackedJobs(int uid) {
        synchronized (mLock) {
            boolean changed = false;
            for (int i = 0; i < mTrackedJobs.size(); i++) {

                // 遍历 job 监控列表
                final JobStatus js = mTrackedJobs.get(i);

                // 关键点！
                if (uid == -1 || uid == js.getSourceUid()) {
                    changed |= updateConstraintsSatisfied(js);
                }
            }

            if (changed) { // changed 为 ture 表示约束条件改变！
                mStateChangedListener.onControllerStateChanged();
            }
        }
    }
```
更新策略分析：

- 如果 uid 的值为 -1，那就更新这个控制器监视的所有 job；
- 如果 uid 的值为指定的 uid，那就更新属于这个 uid 的所有 job；

这里调用了 updateConstraintsSatisfied 方法判断是否有 job 需要更新：

```java
    private boolean updateConstraintsSatisfied(JobStatus jobStatus) {
        final boolean ignoreBlocked = (jobStatus.getFlags() & JobInfo.FLAG_WILL_BE_FOREGROUND) != 0;

        // 获得这个 job 所属 uid （应用程序）的网络信息对象：NetworkInfo！
        final NetworkInfo info = mConnManager.getActiveNetworkInfoForUid(jobStatus.getSourceUid(),
                ignoreBlocked);

        final boolean connected = (info != null) && info.isConnected();
        final boolean unmetered = connected && !info.isMetered();
        final boolean notRoaming = connected && !info.isRoaming();
        boolean changed = false;

        // 根据网络信息来更新 job 的状态，动态更新 satisfiedConstraints 的对应二进制位！
        changed |= jobStatus.setConnectivityConstraintSatisfied(connected);
        changed |= jobStatus.setUnmeteredConstraintSatisfied(unmetered);
        changed |= jobStatus.setNotRoamingConstraintSatisfied(notRoaming);

        return changed;
    }
```

这个方法获得这个 job 所属 uid （应用程序）的网络信息对象， 根据网络信息来更新 job！利用 |= 方法，只要有一个约束条件发生了变化，即发生了变化！
最后，调用 mStateChangedListener 的 onControllerStateChanged 方法来通知有 job 变化了！

ConnectivityController 的 startTrack 和 stopTrack 也很简单：
```java
    @Override
    public void maybeStartTrackingJobLocked(JobStatus jobStatus, JobStatus lastJob) {
        if (jobStatus.hasConnectivityConstraint() || jobStatus.hasUnmeteredConstraint()
                || jobStatus.hasNotRoamingConstraint()) {

            // 在监控 job 之前，检查一下约束条件，并更新 job 的状态！
            updateConstraintsSatisfied(jobStatus); 
            
            // 添加到监控列表！
            mTrackedJobs.add(jobStatus);
        }
    }
    @Override
    public void maybeStopTrackingJobLocked(JobStatus jobStatus, JobStatus incomingJob,
            boolean forUpdate) {
        if (jobStatus.hasConnectivityConstraint() || jobStatus.hasUnmeteredConstraint()
                || jobStatus.hasNotRoamingConstraint()) {
           
            // 从监控列表中移除！
            mTrackedJobs.remove(jobStatus);
        }
    }
```
先到这里，不多说了！



# 2 TimeController
接下来，看看时间控制器，一个 job 的超时和延迟处理，都是由这个控制器处理的：
```java
    private TimeController(StateChangedListener stateChangedListener, Context context, Object lock) {
        super(stateChangedListener, context, lock);
        // 初始化为无限大！
        mNextJobExpiredElapsedMillis = Long.MAX_VALUE;
        mNextDelayExpiredElapsedMillis = Long.MAX_VALUE;
    }
```
可以看到，构造器中的代码很简单，重点不在这里啦！
## 2.1 maybeStartTrackingJobLocked

在  maybeStartTrackingJobLocked，参数传递：

- JobStatus job：这次要执行的 job
- JobStatus lastJob：被取代的 job，可以为 null！
```java
    @Override
    public void maybeStartTrackingJobLocked(JobStatus job, JobStatus lastJob) {

        // 判断 job 是否设置了延迟时间和超时时间！
        if (job.hasTimingDelayConstraint() || job.hasDeadlineConstraint()) {
            maybeStopTrackingJobLocked(job, null, false);

            boolean isInsert = false;
            ListIterator<JobStatus> it = mTrackedJobs.listIterator(mTrackedJobs.size());
            
            // 遍历 track 集合，根据超时处理的时间升序排列，插入新的 job！
            while (it.hasPrevious()) {
                JobStatus ts = it.previous();
                if (ts.getLatestRunTimeElapsed() < job.getLatestRunTimeElapsed()) {
                    // Insert
                    isInsert = true;
                    break;
                }
            }

            if (isInsert) {
                it.next();
            }
            it.add(job);

            // 设置延时处理和超时处理的 alarm！
            maybeUpdateAlarmsLocked(
                    job.hasTimingDelayConstraint() ? job.getEarliestRunTime() : Long.MAX_VALUE,
                    job.hasDeadlineConstraint() ? job.getLatestRunTimeElapsed() : Long.MAX_VALUE,
                    job.getSourceUid());
        }
    }
```
可以看到：

- 如果 job 设置了延时触发条件，那么需要传入一个延迟时间值，这个时间值在这里通过 job.getEarliestRunTime() 获取！
- 如果 job 设置了超时处理触发条件，那么需要传入一个超时时间值，这个时间值在这里通过 job.getLatestRunTimeElapsed() 获取！
- 如果没有设置，默认为 Long.MAX_VALUE！

注意：TimeController 的监控集合是以每个 job 的超时时间点排序的！

## 2.2 maybeUpdateAlarmsLocked

接着进入 maybeUpdateAlarmsLocked 方法中，参数传递：

- long delayExpiredElapsed：延迟触发时间！
- long deadlineExpiredElapsed：超时处理时间！

```java
    private void maybeUpdateAlarmsLocked(long delayExpiredElapsed, long deadlineExpiredElapsed,
            int uid) {
            
        // mNextDelayExpiredElapsedMillis 的值为 Long.MAX_VALUE
        if (delayExpiredElapsed < mNextDelayExpiredElapsedMillis) {
            setDelayExpiredAlarmLocked(delayExpiredElapsed, uid);
        }
        
        // mNextJobExpiredElapsedMillis 的值为 Long.MAX_VALUE
        if (deadlineExpiredElapsed < mNextJobExpiredElapsedMillis) {
            setDeadlineExpiredAlarmLocked(deadlineExpiredElapsed, uid);
        }
    }
```
这里调用了两个方法，来设定 alarm：
## 2.3 setXXXXExpiredAlarmLocked
```java
    // 延时处理闹钟设置！
    private void setDelayExpiredAlarmLocked(long alarmTimeElapsedMillis, int uid) {
    
        // 校验时间，最小等于从开机到现在的时间！
        alarmTimeElapsedMillis = maybeAdjustAlarmTime(alarmTimeElapsedMillis);

        mNextDelayExpiredElapsedMillis = alarmTimeElapsedMillis;
        updateAlarmWithListenerLocked(DELAY_TAG, mNextDelayExpiredListener,
                mNextDelayExpiredElapsedMillis, uid);
    }
    
    // 超时处理闹钟设置！
    private void setDeadlineExpiredAlarmLocked(long alarmTimeElapsedMillis, int uid) {
    
        // 校验时间，最小等于从开机到现在的时间！
        alarmTimeElapsedMillis = maybeAdjustAlarmTime(alarmTimeElapsedMillis);
        mNextJobExpiredElapsedMillis = alarmTimeElapsedMillis;

        updateAlarmWithListenerLocked(DEADLINE_TAG, mDeadlineExpiredListener,
                mNextJobExpiredElapsedMillis, uid);
    }
```
这里的代码很有意思：
首先返回一个正确的延迟或者超时处理时间点，然后根据这个时间点设置一个 alarm，时间点到了，会触发这个 alarm，linster  的回调会被执行！
我们继续看：
### 2.3.1 maybeAdjustAlarmTime   
```java
   private long maybeAdjustAlarmTime(long proposedAlarmTimeElapsedMillis) {
   
        // 获得从开机到现在的时间！
        final long earliestWakeupTimeElapsed = SystemClock.elapsedRealtime();
        
        // 取 job 的时间和开机时间的最大值！ 
        if (proposedAlarmTimeElapsedMillis < earliestWakeupTimeElapsed) {
            return earliestWakeupTimeElapsed;
        }
        return proposedAlarmTimeElapsedMillis;
    }
```
这个方法是用来和开机时间比较的，返回一个最大值，确保时间正确性！
### 2.3.2 updateAlarmWithListenerLocked

这里是设置一个 alarm，在指定时间触发 alarm，执行 job，参数传递：

- String tag：表示 alarm 的 tag，可取值为 DELAY_TAG 或者 DEADLINE_TAG；
- OnAlarmListener listener：OnAlarmListener 对象！
- long alarmTimeElapsed：触发时间！
- int uid： job 的 uid 

```java
    private void updateAlarmWithListenerLocked(String tag, OnAlarmListener listener,
            long alarmTimeElapsed, int uid) {

        // 获得 alarmService 代理对象！    
        ensureAlarmServiceLocked();
        if (alarmTimeElapsed == Long.MAX_VALUE) {
            mAlarmService.cancel(listener);

        } else {
            if (DEBUG) {
                Slog.d(TAG, "Setting " + tag + " for: " + alarmTimeElapsed);
            }

            // important point！
            mAlarmService.set(AlarmManager.ELAPSED_REALTIME_WAKEUP, alarmTimeElapsed,
                    AlarmManager.WINDOW_HEURISTIC, 0, tag, listener, null, new WorkSource(uid));
        }
    }
```
这里会在时间点 alarmTimeElapsed 到达时，触发 alarm，执行监听器 OnAlarmListener listener 的 onAlarm 方法：

#### 2.3.2.1 mNextDelayExpiredListener

我们先来看看延迟处理的 Listener：
```java
    private final OnAlarmListener mNextDelayExpiredListener = new OnAlarmListener() {
        @Override
        public void onAlarm() {
            if (DEBUG) {
                Slog.d(TAG, "Delay-expired alarm fired");
            }
            
            // 时间点到后会触发 checkExpiredDelaysAndResetAlarm 方法！
            checkExpiredDelaysAndResetAlarm();
        }
    };
```
##### 2.3.2.1.1 checkExpiredDelaysAndResetAlarm
这里调用了 checkExpiredDelaysAndResetAlarm 方法：
```java
    private void checkExpiredDelaysAndResetAlarm() {
        synchronized (mLock) {
        
            // 获得从开机到现在的时间
            final long nowElapsedMillis = SystemClock.elapsedRealtime();
            long nextDelayTime = Long.MAX_VALUE;
            int nextDelayUid = 0;
            boolean ready = false; // 默认为 false，表示是否有 job 准备执行！

            Iterator<JobStatus> it = mTrackedJobs.iterator(); // 遍历 track 集合！
            while (it.hasNext()) {
                final JobStatus job = it.next();

                if (!job.hasTimingDelayConstraint()) { 
                    continue;
                }

                // 获得每个 job 的延迟处理时间；
                final long jobDelayTime = job.getEarliestRunTime();

                if (jobDelayTime <= nowElapsedMillis) { // 说明延时触发时间点到了！
                   
                    // 设置该 job 的对应 TimingDelay 二进制位为 1，表示满足条件！
                    job.setTimingDelayConstraintSatisfied(true); 
                    
                    if (canStopTrackingJobLocked(job)) {
                        it.remove();
                    }
                    
                    if (job.isReady()) { // 如果 job 准备好了，就准备执行 job！
                        ready = true;
                    }
                    
                } else if (!job.isConstraintSatisfied(JobStatus.CONSTRAINT_TIMING_DELAY)) {

                    // 这里是为了在剩余的，本次未触发的 job 里，找到延时触发时间最短的 job，并以它的延迟触发时间，
                    // 作为下次 alarm 的触发时间点！
                    if (nextDelayTime > jobDelayTime) {
                        nextDelayTime = jobDelayTime;
                        nextDelayUid = job.getSourceUid();
                    }
                }
            }

            if (ready) { // 如果是 true，通知 jobScheduler！
                mStateChangedListener.onControllerStateChanged();
            }
            
            // 设置下一轮的 alarm!
            setDelayExpiredAlarmLocked(nextDelayTime, nextDelayUid);
        }
    }
```
当 lisener 的回调被执行时，这个时候，job 的延时处理时间会小于等于系统从开机到现在的时间，所以会进入第一个 if 分支！

对于本次没有触发，且没有被取消的 job，找到延时处理时间最短的 job，并以它的延迟触发时间，作为下次 alarm 的触发时间点！

这里调用了 `job.isReady()`，判断一个 job 是否准备好：
```
    public boolean isReady() {
    
        // 判断 job 的 deadline 条件是否满足，只对非周期 job 才会有；
        final boolean deadlineSatisfied = (!job.isPeriodic() && hasDeadlineConstraint()
                && (satisfiedConstraints & CONSTRAINT_DEADLINE) != 0);
        
        // 判断应用处于非空闲状态的条件是否满足！
        final boolean notIdle = (satisfiedConstraints & CONSTRAINT_APP_NOT_IDLE) != 0;
        
        // 判断处于非 dozing 模式的是否满足！
        final boolean notDozing = (satisfiedConstraints & CONSTRAINT_DEVICE_NOT_DOZING) != 0
                || (job.getFlags() & JobInfo.FLAG_WILL_BE_FOREGROUND) != 0;
                
        // 最后判断这个 job 是否 ready！
        return (isConstraintsSatisfied() || deadlineSatisfied) && notIdle && notDozing;
    }
```
可以看到，一个 job 处于 ready 状态，需要同时满足如下的几个条件：

- job 的所有约束条件都满足，或者对于非周期 job，deadline 已经满足了！
- job 设置的应用处于非空闲状态的条件满足；
- job 设置的设备处于非doze的条件满足；


#### 2.3.2.2 mDeadlineExpiredListener

我们先来看看超时处理的 Listener：
```java
    private final OnAlarmListener mDeadlineExpiredListener = new OnAlarmListener() {
        @Override
        public void onAlarm() {
            if (DEBUG) {
                Slog.d(TAG, "Deadline-expired alarm fired");
            }
            
            // 时间点到后会触发 checkExpiredDeadlinesAndResetAlarm 方法！
            checkExpiredDeadlinesAndResetAlarm();
        }
    };
```

这里调用了 checkExpiredDeadlinesAndResetAlarm：
##### 2.3.2.2.1 checkExpiredDeadlinesAndResetAlarm
```java
    private void checkExpiredDeadlinesAndResetAlarm() {
        synchronized (mLock) {
            long nextExpiryTime = Long.MAX_VALUE;
            int nextExpiryUid = 0;
            final long nowElapsedMillis = SystemClock.elapsedRealtime();
            Iterator<JobStatus> it = mTrackedJobs.iterator();

            while (it.hasNext()) { // 遍历集合
                JobStatus job = it.next();
                if (!job.hasDeadlineConstraint()) {
                    continue;
                }

                final long jobDeadline = job.getLatestRunTimeElapsed(); // 获得每个 job 的超时触发时间点！

                if (jobDeadline <= nowElapsedMillis) { // 超时时间触发点到了！
                    if (job.hasTimingDelayConstraint()) { // 如果 job 还有设置了延迟时间触发条件，将相应的 TimingDelay 二进制位为 1！
                        job.setTimingDelayConstraintSatisfied(true);
                    }
                    job.setDeadlineConstraintSatisfied(true); // 设置相应的 Deadline 二进制位为 1！
                    
                    mStateChangedListener.onRunJobNow(job); // 立刻执行 Job！
                    
                    it.remove(); // 移除，不再监控！
                    
                } else { 
                    // 找到下一个没有满足的 job ，以它的超时处理时间点作为下次 alarm 的触发点，退出循环！
                    nextExpiryTime = jobDeadline;
                    nextExpiryUid = job.getSourceUid();
                    break;
                    
                }
            }

            // 设置下一次的 alarm！
            setDeadlineExpiredAlarmLocked(nextExpiryTime, nextExpiryUid);
        }
    }
```
最后，调用了 JSS 的 onRunJobNow 方法，这里我们后面再看！


# 3 IdleController
我们来看看 IdleController 这个空闲状态控制器的构造器：
```java
    private IdleController(StateChangedListener stateChangedListener, Context context,
                Object lock) {
        super(stateChangedListener, context, lock);
        initIdleStateTracking();
    }
```
调用了这个方法，initIdleStateTracking：
```java
    /**
     * Idle state tracking, and messaging with the task manager when
     * significant state changes occur
     */
    private void initIdleStateTracking() {
        // 设备真正进入空闲的时间间隔：4260000 毫秒
        mInactivityIdleThreshold = mContext.getResources().getInteger(
                com.android.internal.R.integer.config_jobSchedulerInactivityIdleThreshold);
        mIdleWindowSlop = mContext.getResources().getInteger(
                com.android.internal.R.integer.config_jobSchedulerIdleWindowSlop);
        
        // 创建了一个设备空闲监控器，并启动监控！        
        mIdleTracker = new IdlenessTracker();
        mIdleTracker.startTracking();
    }
```
这里创建了一个 IdlenessTracker。
## 3.1 IdlenessTracker

这个 Tracker 是一个广播接收者！
```java
    class IdlenessTracker extends BroadcastReceiver {
        private AlarmManager mAlarm;
        private PendingIntent mIdleTriggerIntent;
        boolean mIdle;
        boolean mScreenOn;
        public IdlenessTracker() {
        
            // 设置一个 alarm，用来发送 ActivityManagerService.ACTION_TRIGGER_IDLE 广播！
            mAlarm = (AlarmManager) mContext.getSystemService(Context.ALARM_SERVICE);

            Intent intent = new Intent(ActivityManagerService.ACTION_TRIGGER_IDLE)
                    .setPackage("android")
                    .setFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY);
                    
            mIdleTriggerIntent = PendingIntent.getBroadcast(mContext, 0, intent, 0);
            // At boot we presume that the user has just "interacted" with the
            // device in some meaningful way.
            mIdle = false;
            mScreenOn = true;
        }

        // 设备是否处于空闲的标志！
        public boolean isIdle() {
            return mIdle;
        }

        public void startTracking() {
            IntentFilter filter = new IntentFilter();

            // 监听屏幕的亮暗状态！
            filter.addAction(Intent.ACTION_SCREEN_ON);
            filter.addAction(Intent.ACTION_SCREEN_OFF);

            // 监听设备的睡眠状态
            filter.addAction(Intent.ACTION_DREAMING_STARTED);
            filter.addAction(Intent.ACTION_DREAMING_STOPPED);

            // 监听设备进入空闲状态
            filter.addAction(ActivityManagerService.ACTION_TRIGGER_IDLE);
            mContext.registerReceiver(this, filter);
        }

        @Override
        public void onReceive(Context context, Intent intent) {
            final String action = intent.getAction();
            
            // 监听亮屏广播 或者 退出休眠的广播。
            if (action.equals(Intent.ACTION_SCREEN_ON) 
                    || action.equals(Intent.ACTION_DREAMING_STOPPED)) {
                if (DEBUG) {
                    Slog.v(TAG,"exiting idle : " + action);
                }
                mScreenOn = true; // 表示亮屏
                
                //cancel the alarm
                mAlarm.cancel(mIdleTriggerIntent); // 取消闹钟！
                if (mIdle) {
                    mIdle = false; // 表示设备进入了非空闲！
                    reportNewIdleState(mIdle);
                }

            } else if (action.equals(Intent.ACTION_SCREEN_OFF) // 熄屏广播 和 进入休眠的广播。
                    || action.equals(Intent.ACTION_DREAMING_STARTED)) {

                // 因为接到广播和设备真正处于空闲是有一定时间间隔的，所以这里设置了一个闹钟，
                // 用于在指定时间后发送广播：ActivityManagerService.ACTION_TRIGGER_IDLE

                final long nowElapsed = SystemClock.elapsedRealtime();
                final long when = nowElapsed + mInactivityIdleThreshold;
                if (DEBUG) {
                    Slog.v(TAG, "Scheduling idle : " + action + " now:" + nowElapsed + " when="
                            + when);
                }
                mScreenOn = false; // 表示灭屏！
                mAlarm.setWindow(AlarmManager.ELAPSED_REALTIME_WAKEUP,
                        when, mIdleWindowSlop, mIdleTriggerIntent);

                
            } else if (action.equals(ActivityManagerService.ACTION_TRIGGER_IDLE)) { // 表示设备真正进入了空闲！
                // idle time starts now. Do not set mIdle if screen is on.
                if (!mIdle && !mScreenOn) {
                    if (DEBUG) {
                        Slog.v(TAG, "Idle trigger fired @ " + SystemClock.elapsedRealtime());
                    }
                    mIdle = true; // 表示设备进入了空闲状态
                    reportNewIdleState(mIdle);
                }
            }
        }
    }
```
总结下：
收到亮屏广播或者退出休眠的广播，设备处于非空闲，取消 alarm，如果设备之前是空闲的，那就通知 jobScheduler！
收到熄屏广播或者进入出休眠的广播，设备可能进入空闲状态，设置 alarm，在一定时间后发送广播；接受这个广播，如果设备之前是非空闲的，那就通知 jobScheduler！

接下来，进入 reportNewIdleState 方法中来看看：
```java
    void reportNewIdleState(boolean isIdle) {
        synchronized (mLock) {
            for (JobStatus task : mTrackedTasks) {
                // 设置 job 的设备空闲状态条件！
                task.setIdleConstraintSatisfied(isIdle);
            }
        }
        
        // 通知 JSS，设备的空闲状态发生了变化！
        mStateChangedListener.onControllerStateChanged();
    }
```
这个方法很简单，修改自己所监控的 Job 的 JobStatus 的 Idle 二进制位，并通知 JobSchedulerService ，JobSchdulerService 会根据这个状态来动态控制 jobServcie！

IdleController 的 StartTrackingJob 和 StopTrackingJob 都很简单，请看下面：
```java
    /**
     * StateController interface
     */
    @Override
    public void maybeStartTrackingJobLocked(JobStatus taskStatus, JobStatus lastJob) {
        if (taskStatus.hasIdleConstraint()) { // 判断 job 是否设置空闲约束！
            mTrackedTasks.add(taskStatus);

            // 监控之前，进行 job 的设备状态初始化！
            taskStatus.setIdleConstraintSatisfied(mIdleTracker.isIdle()); 
        }
    }
    @Override
    public void maybeStopTrackingJobLocked(JobStatus taskStatus, JobStatus incomingJob, boolean forUpdate) {
        
        mTrackedTasks.remove(taskStatus);
    }
```
不多说了！




# 4 BatteryController
我们来看看电量控制器：
```java
    private BatteryController(StateChangedListener stateChangedListener, Context context,
            Object lock) {
        super(stateChangedListener, context, lock);

        // 这里创建了一个充电监视器
        mChargeTracker = new ChargingTracker();
        mChargeTracker.startTracking();
    }
```
## 4.1 ChargingTracker

我们去看看：
```java
    public class ChargingTracker extends BroadcastReceiver {
        /**
         * Track whether we're "charging", where charging means that we're ready to commit to
         * doing work.
         */
        private boolean mCharging;
        /** Keep track of whether the battery is charged enough that we want to do work. */
        private boolean mBatteryHealthy;
        public ChargingTracker() {
        }
        public void startTracking() {
            IntentFilter filter = new IntentFilter();
            // Battery health.

            // 监听电量
            filter.addAction(Intent.ACTION_BATTERY_LOW);
            filter.addAction(Intent.ACTION_BATTERY_OKAY);
            // Charging/not charging.

            // 监听充电的广播
            filter.addAction(BatteryManager.ACTION_CHARGING);
            filter.addAction(BatteryManager.ACTION_DISCHARGING);
            mContext.registerReceiver(this, filter);

            // Initialise tracker state.
            BatteryManagerInternal batteryManagerInternal =
                    LocalServices.getService(BatteryManagerInternal.class);

            // 初始化电量的状态
            mBatteryHealthy = !batteryManagerInternal.getBatteryLevelLow();
            mCharging = batteryManagerInternal.isPowered(BatteryManager.BATTERY_PLUGGED_ANY);
        }

        // 判断当前的电池环境是否稳定
        boolean isOnStablePower() {
            return mCharging && mBatteryHealthy;
        }

        @Override
        public void onReceive(Context context, Intent intent) {
            // 处理广播！
            onReceiveInternal(intent);
        }

        @VisibleForTesting
        public void onReceiveInternal(Intent intent) {
            final String action = intent.getAction();
            if (Intent.ACTION_BATTERY_LOW.equals(action)) {
                if (DEBUG) {
                    Slog.d(TAG, "Battery life too low to do work. @ "
                            + SystemClock.elapsedRealtime());
                }
                // If we get this action, the battery is discharging => it isn't plugged in so
                // there's no work to cancel. We track this variable for the case where it is
                // charging, but hasn't been for long enough to be healthy.
                // 电量太低，mBatteryHealthy 置为 false！
                mBatteryHealthy = false;

            } else if (Intent.ACTION_BATTERY_OKAY.equals(action)) {
                if (DEBUG) {
                    Slog.d(TAG, "Battery life healthy enough to do work. @ "
                            + SystemClock.elapsedRealtime());
                }
                // 电量充足，mBatteryHealthy 置为 true，通知 job 状态需要改变！
                mBatteryHealthy = true;
                maybeReportNewChargingState();

            } else if (BatteryManager.ACTION_CHARGING.equals(action)) {
                if (DEBUG) {
                    Slog.d(TAG, "Received charging intent, fired @ "
                            + SystemClock.elapsedRealtime());
                }
                // 正在充电，mCharging 置为 true，通知 job 状态需要改变！
                mCharging = true;
                maybeReportNewChargingState();

            } else if (BatteryManager.ACTION_DISCHARGING.equals(action)) {
                if (DEBUG) {
                    Slog.d(TAG, "Disconnected from power.");
                }
                // 充电断开，mCharging 置为 false，通知 job 状态需要改变！
                mCharging = false;
                maybeReportNewChargingState();

            }
        }
    }
```
### 4.1.1 maybeReportNewChargingState
最后调用了 maybeReportNewChargingState 方法：
```java
    private void maybeReportNewChargingState() {

        // 获得本次电池的状态
        final boolean stablePower = mChargeTracker.isOnStablePower();

        if (DEBUG) {
            Slog.d(TAG, "maybeReportNewChargingState: " + stablePower);
        }

        boolean reportChange = false;
        synchronized (mLock) {
            for (JobStatus ts : mTrackedTasks) {
                // 对于这个 controller 监视的所有 JobStatus，判断这次是否满足约束条件！
                boolean previous = ts.setChargingConstraintSatisfied(stablePower);
                if (previous != stablePower) {
                    // 这里一次和上一次不一样，需要通知 JobSchedulerService，发生变化
                    reportChange = true;
                }
            }
        }
        // Let the scheduler know that state has changed. This may or may not result in an
        // execution.
        if (reportChange) { // 通知 jobScheduler，发生变化
            mStateChangedListener.onControllerStateChanged();
        }
        // Also tell the scheduler that any ready jobs should be flushed.
        if (stablePower) { // 通知 jobScheduler，如果电池状态满足，执行任务
            mStateChangedListener.onRunJobNow(null);
        }
    }
```
这里很简单，就不说了！



       
# 5 AppIdleController
这个控制器的主要作用是，判断 app 是否是闲置的，如果是闲置的，那么他的 job 将会被执行，如何判断闲置呢？没有被启动，或者数小时或者数天之前在启动过！
我们先来看看 AppIdleController 的构造器：
```java
    private AppIdleController(JobSchedulerService service, Context context, Object lock) {
        super(service, context, lock);
        mJobSchedulerService = service;
        mUsageStatsInternal = LocalServices.getService(UsageStatsManagerInternal.class);
        mAppIdleParoleOn = true;
        // 将 AppIdleStateChangeListener 注册进入 UsageStatsManagerInternal 服务中
        mUsageStatsInternal.addAppIdleStateChangeListener(new AppIdleStateChangeListener());
    }
```
我们继续看：
## 5.1 AppIdleStateChangeListener

这里有一个监听器： AppIdleStateChangeListener：
```java
    private class AppIdleStateChangeListener
            extends UsageStatsManagerInternal.AppIdleStateChangeListener {

        @Override
        public void onAppIdleStateChanged(String packageName, int userId, boolean idle) {
            boolean changed = false;
            synchronized (mLock) {
                if (mAppIdleParoleOn) {
                    return;
                }
                // 创建 PackageUpdateFunc，对 userId 下的包名为 pacakgeName 的应用的所有 job，进行状态检查和更新
                PackageUpdateFunc update = new PackageUpdateFunc(userId, packageName, idle);
                mJobSchedulerService.getJobStore().forEachJob(update);
                if (update.mChanged) {
                    changed = true;
                }
            }
            if (changed) { // 通知 jobScheduler，发生变化
                mStateChangedListener.onControllerStateChanged();
            }
        }

        @Override
        public void onParoleStateChanged(boolean isParoleOn) {
            if (DEBUG) {
                Slog.d(LOG_TAG, "Parole on: " + isParoleOn);
            }
            setAppIdleParoleOn(isParoleOn);
        }
    }
```
这里的监听器本质上只是一个回调接口，具体的 controll 策略是在 UsageStatsManagerInternal 服务中，这里不多讲（因为特么我还没看）
### 5.1.1 PackageUpdateFunc

首先来看看这个函数类：
```java
    final static class PackageUpdateFunc implements JobStore.JobStatusFunctor {
        final int mUserId;
        final String mPackage;
        final boolean mIdle;
        boolean mChanged; // 用来保存 app 状态是否发生变化
        
        PackageUpdateFunc(int userId, String pkg, boolean idle) {
            mUserId = userId;
            mPackage = pkg;
            mIdle = idle;
        }
        @Override public void process(JobStatus jobStatus) {
            if (jobStatus.getSourcePackageName().equals(mPackage) // 判断当前的 job 是否属于这个应用程序
                    && jobStatus.getSourceUserId() == mUserId) {
                if (jobStatus.setAppNotIdleConstraintSatisfied(!mIdle)) {
                    if (DEBUG) {
                        Slog.d(LOG_TAG, "App Idle state changed, setting idle state of "
                                + mPackage + " to " + mIdle);
                    }
                    mChanged = true;
                }
            }
        }
    };
```
PackageUpdateFunc 会遍历 JobStore，查找指定 app 的 Job，改变 job 的属性值！
### 5.1.2 setAppIdleParoleOn

这个方法的作用是（特么我没看懂，这里的解释空下）
```java
    void setAppIdleParoleOn(boolean isAppIdleParoleOn) {
        // Flag if any app's idle state has changed
        boolean changed = false;
        synchronized (mLock) {
            if (mAppIdleParoleOn == isAppIdleParoleOn) { // 如果状态没有变化，不处理！
                return;
            }
            mAppIdleParoleOn = isAppIdleParoleOn;
            // 创建 GlobalUpdateFunc 来更新 Job 状态！
            GlobalUpdateFunc update = new GlobalUpdateFunc();
            mJobSchedulerService.getJobStore().forEachJob(update);
            if (update.mChanged) {
                changed = true;
            }
        }
        if (changed) {
            mStateChangedListener.onControllerStateChanged();
        }
    }
```
我们再去看看  GlobalUpdateFunc：
```java
    final class GlobalUpdateFunc implements JobStore.JobStatusFunctor {
        boolean mChanged;
        @Override public void process(JobStatus jobStatus) {
            String packageName = jobStatus.getSourcePackageName();
            final boolean appIdle = !mAppIdleParoleOn && mUsageStatsInternal.isAppIdle(packageName,
                    jobStatus.getSourceUid(), jobStatus.getSourceUserId());
            if (DEBUG) {
                Slog.d(LOG_TAG, "Setting idle state of " + packageName + " to " + appIdle);
            }
            // 改变 job 状态！如果改变成功了，该方法返回的是 true！
            if (jobStatus.setAppNotIdleConstraintSatisfied(!appIdle)) { 
                mChanged = true;
            }
        }
    };
```
如果，有 job 的状态发生了变化，那么 GlobalUpdateFunc 的 mChanged 的值为 true，然后通知 JobScheduler 即可！

# 6 ContentObserverController
通过监听数据库的变化来 controll job：
```ajva
    private ContentObserverController(StateChangedListener stateChangedListener, Context context,
                Object lock) {
        super(stateChangedListener, context, lock);
        mHandler = new Handler(context.getMainLooper());
    }
```
可以看到 ，构造器很简单，那真正的 controll 在哪里呢，首先我们来一个内部变量：
```ajva
    final private List<JobStatus> mTrackedTasks = new ArrayList<JobStatus>();
    /**
     * Per-userid {@link JobInfo.TriggerContentUri} keyed ContentObserver cache.
     */
    SparseArray<ArrayMap<JobInfo.TriggerContentUri, ObserverInstance>> mObservers = new SparseArray<>();
```
mTrackedTasks：表示这个 controller 监视的所有的 job：
mObservers：key 值为 userId，表示的是设备用户，value 值为 ArrayMap<JobInfo.TriggerContentUri, ObserverInstance>，数据库uri和它的观察者！

还记得，在 JobScheduler 中我们开始  track 每一个 job 吗？我们去这个方法里面去看看：
## 6.1 maybeStartTrackingJobLocked

参数传递：

- JobStatus taskStatus：即将要执行的 job
- JobStatus lastJob：被取代的旧的 job
```java
    @Override
    public void maybeStartTrackingJobLocked(JobStatus taskStatus, JobStatus lastJob) {
        // 判断新的 job  hasContentTriggerConstraint 是否为 true！
        if (taskStatus.hasContentTriggerConstraint()) {
            if (taskStatus.contentObserverJobInstance == null) {
                // 给 jobStatus 创建一个 JobInstance 对象！
                taskStatus.contentObserverJobInstance = new JobInstance(taskStatus);
            }
            if (DEBUG) {
                Slog.i(TAG, "Tracking content-trigger job " + taskStatus);
            }
            // 添加到追踪列表中！
            mTrackedTasks.add(taskStatus);
            boolean havePendingUris = false;
            // If there is a previous job associated with the new job, propagate over
            // any pending content URI trigger reports.
            if (taskStatus.contentObserverJobInstance.mChangedAuthorities != null) {
                havePendingUris = true;
            }
            // If we have previously reported changed authorities/uris, then we failed
            // to complete the job with them so will re-record them to report again.
            // 对于有些 job ，我们之前有监视过其依赖的数据库的变化；但是，我们并没有完成这个 job，
            // 如果这个 job 被 reschedule，那么之前的 uris/authorities 仍然要被监控！
            if (taskStatus.changedAuthorities != null) {
                havePendingUris = true; // 用户标记是否有正在处理的 uris
                if (taskStatus.contentObserverJobInstance.mChangedAuthorities == null) {
                    taskStatus.contentObserverJobInstance.mChangedAuthorities
                            = new ArraySet<>();
                }
                for (String auth : taskStatus.changedAuthorities) {
                    taskStatus.contentObserverJobInstance.mChangedAuthorities.add(auth);
                }
                if (taskStatus.changedUris != null) {
                    if (taskStatus.contentObserverJobInstance.mChangedUris == null) {
                        taskStatus.contentObserverJobInstance.mChangedUris = new ArraySet<>();
                    }
                    for (Uri uri : taskStatus.changedUris) {
                        taskStatus.contentObserverJobInstance.mChangedUris.add(uri);
                    }
                }
                taskStatus.changedAuthorities = null;
                taskStatus.changedUris = null;
            }
            taskStatus.changedAuthorities = null;
            taskStatus.changedUris = null;
            taskStatus.setContentTriggerConstraintSatisfied(havePendingUris);
        }
        // 如果当前的这个 job 有相同 jobId 的被取消的上一次 job !
        if (lastJob != null && lastJob.contentObserverJobInstance != null) {
            // And now we can detach the instance state from the last job.
            lastJob.contentObserverJobInstance.detachLocked();
            lastJob.contentObserverJobInstance = null;
        }
    }
```
对于有被替代的相同uid和jobId的旧的 job，会调用 jobInstance 的 detachLocked 方法来做一些释放资源的操作，这个我们在后面来看！

可以看到，每一个 JobStatus 都会有一个 ContentObserverController.JobInstance 类型的对象：
### 6.1.1 JobInstance

参数传递：

- JobStatus jobStatus
```java
    final class JobInstance {
        final ArrayList<ObserverInstance> mMyObservers = new ArrayList<>(); // 这个 job 所拥有的观察者实例集合
        final JobStatus mJobStatus; // 所属的 jobStatus
        final Runnable mExecuteRunner;
        final Runnable mTimeoutRunner;
        // 表示哪些数据库发生了变化，最大等于 build 时传入的 uris 个数！
        ArraySet<Uri> mChangedUris;
        ArraySet<String> mChangedAuthorities;
        boolean mTriggerPending;
        // This constructor must be called with the master job scheduler lock held.
        JobInstance(JobStatus jobStatus) {
            mJobStatus = jobStatus;
            mExecuteRunner = new TriggerRunnable(this);
            mTimeoutRunner = new TriggerRunnable(this);
            // 获得这个 job 所依赖的所有的数据库 uri，在 JobInfo.mTriggerContentUris 中！
            // 以及这个 job 所属的 userId！
            final JobInfo.TriggerContentUri[] uris = jobStatus.getJob().getTriggerContentUris();
            final int sourceUserId = jobStatus.getSourceUserId();
             
            // 创建 useId 下的观察者缓存，用来提高查询效率！
            ArrayMap<JobInfo.TriggerContentUri, ObserverInstance> observersOfUser =
                    mObservers.get(sourceUserId);
            if (observersOfUser == null) {
                observersOfUser = new ArrayMap<>();
                mObservers.put(sourceUserId, observersOfUser);
            }

            if (uris != null) {
                for (JobInfo.TriggerContentUri uri : uris) {
                    ObserverInstance obs = observersOfUser.get(uri);
                    if (obs == null) {
                        // 为每一个 uri 都创建一个观察者实例！
                        obs = new ObserverInstance(mHandler, uri, jobStatus.getSourceUserId());
                        // 添加到 ContentObserverController 的缓存集合里
                        observersOfUser.put(uri, obs);
                        final boolean andDescendants = (uri.getFlags() &
                                JobInfo.TriggerContentUri.FLAG_NOTIFY_FOR_DESCENDANTS) != 0;

                        if (DEBUG) {
                            Slog.v(TAG, "New observer " + obs + " for " + uri.getUri()
                                    + " andDescendants=" + andDescendants
                                    + " sourceUserId=" + sourceUserId);
                        }
                        // 注册这个内容观察者，当内容发生变化后，会触发回调！
                        mContext.getContentResolver().registerContentObserver(
                                uri.getUri(),
                                andDescendants,
                                obs,
                                sourceUserId
                        );
                    } else {
                        if (DEBUG) {
                            final boolean andDescendants = (uri.getFlags() &
                                    JobInfo.TriggerContentUri.FLAG_NOTIFY_FOR_DESCENDANTS) != 0;
                            Slog.v(TAG, "Reusing existing observer " + obs + " for " + uri.getUri()
                                    + " andDescendants=" + andDescendants);
                        }
                    }
                    // 将这个 JobInstance 添加到这个 ContextObsercer 中！
                    obs.mJobs.add(this);
                    // 添加到 jobInstance 的 mMyObservers 中！
                    mMyObservers.add(obs);
                }
            }
        }
        ... ... ... ...
    }
```
这里创建了一个 ObserverInstance 对象，用来监听数据库的变动，当 uri 所指向的数据库发生变化后，他会通知所有的和他关联的 job Instance：
### 6.1.2 ObserverInstance
```java
    final class ObserverInstance extends ContentObserver {
        final JobInfo.TriggerContentUri mUri;
        final @UserIdInt int mUserId;
        // 表示的是这个数据库关联的所有的 job 实例！
        final ArraySet<JobInstance> mJobs = new ArraySet<>();
        public ObserverInstance(Handler handler, JobInfo.TriggerContentUri uri,
                @UserIdInt int userId) {
            super(handler);
            mUri = uri;
            mUserId = userId;
        }
        @Override
        public void onChange(boolean selfChange, Uri uri) {
            if (DEBUG) {
                Slog.i(TAG,"onChange(self=" + selfChange + ") for "  + uri
                        + " when mUri=" + mUri + " mUserId=" + mUserId);
            }
            synchronized (mLock) {
                final int N = mJobs.size();
                // 当 uri 所指向的数据库发生变化后，他会通知所有的和他关联的 job Instance！
                // 将改变的数据库的 uri 和 authority 保存到对应的 JobInstance 中！
                for (int i=0; i<N; i++) {
                    JobInstance inst = mJobs.valueAt(i);
                    if (inst.mChangedUris == null) {
                        inst.mChangedUris = new ArraySet<>();
                    }
                    if (inst.mChangedUris.size() < MAX_URIS_REPORTED) {
                        inst.mChangedUris.add(uri);
                    }
                    if (inst.mChangedAuthorities == null) {
                        inst.mChangedAuthorities = new ArraySet<>();
                    }
                    inst.mChangedAuthorities.add(uri.getAuthority());
                    // 调用 JobStance 的 scheduleLocked 方法！
                    inst.scheduleLocked();
                }
            }
        }
    }
```
接下来，进入到 JobInstance 中，当观察者发现数据库改变了，他最终会调用相关联的所有的 JobInstance 的 scheduleLocked 方法：
#### 6.1.2.1 JI.scheduleLocked

开始执行触发任务：
```java
        void scheduleLocked() {
            if (!mTriggerPending) {
                mTriggerPending = true;
                mHandler.postDelayed(mTimeoutRunner, mJobStatus.getTriggerContentMaxDelay());
            }
            mHandler.removeCallbacks(mExecuteRunner);
            // URIS_URGENT_THRESHOLD 表示一个 job 最大可以依赖的数据库个数！
            if (mChangedUris.size() >= URIS_URGENT_THRESHOLD) {
                // If we start getting near the limit, GO NOW!
                mHandler.post(mExecuteRunner);
            } else {
                mHandler.postDelayed(mExecuteRunner, mJobStatus.getTriggerContentUpdateDelay());
            }
        }
```
mTriggerPending 默认为 fasle。用来标记：Job 触发是否正在进行！
可以看出：
第一次数据库改变，执行触发时，会延迟处理，延迟时间是：JobStatus.getTriggerContentMaxDelay()；
如果在第一次触发还未结束，又进行了多次触发，延时处理，延时时间是：JobStatus.getTriggerContentUpdateDelay()；
如果这个 job 依赖的数据库 uri 的个数到达的最大限制，就立刻触发，优先处理资源紧缺的 job 触发！

这里会在主线程执行 mTimeoutRunner 和  mExecuteRunner，mExecuteRunner  是 TriggerRunnable 对象：
```java
    static final class TriggerRunnable implements Runnable {
        final JobInstance mInstance;
        TriggerRunnable(JobInstance instance) {
            mInstance = instance;
        }
        @Override public void run() {
            // 执行 JobInstance 的 trigger 方法！
            mInstance.trigger();
        }
    }
```
我们进入到 trigger 方法中，去看看：
#### 6.1.2.2 JI.trigger

这个方法就是最终的通知改变的方法：
```java
        void trigger() {
            boolean reportChange = false;
            synchronized (mLock) {
                if (mTriggerPending) {
                    // 判断是否需要改变 job 状态
                    if (mJobStatus.setContentTriggerConstraintSatisfied(true)) {
                        reportChange = true;
                    }
                    unscheduleLocked(); // 结束本次执行操作！
                }
            }
            // Let the scheduler know that state has changed. This may or may not result in an
            // execution.
            if (reportChange) { // 当 reportChange 为 true 后，就通知 JobScheduler！
                mStateChangedListener.onControllerStateChanged();
            }
        }
```
再来看看 unscheduleLocked 方法，本次执行结束，取消任务，标记位置 mTriggerPending 置为 false！
```java
        void unscheduleLocked() {
            if (mTriggerPending) {
                mHandler.removeCallbacks(mExecuteRunner);
                mHandler.removeCallbacks(mTimeoutRunner);
                mTriggerPending = false;
            }
        }
```
这里逻辑已经很清楚了，就不多说了！
## 6.2 maybeStopTrackingJobLocked

接下来，我们来看看停止监视相关的代码：
```java
    @Override
    public void maybeStopTrackingJobLocked(JobStatus taskStatus, JobStatus incomingJob,
            boolean forUpdate) {
        if (taskStatus.hasContentTriggerConstraint()) {
            if (taskStatus.contentObserverJobInstance != null) {
                // 停止相应的 jobInstance 的执行
                taskStatus.contentObserverJobInstance.unscheduleLocked();
                if (incomingJob != null) { // 如果是有相同 userId 和 jobId 的新 job。即 incomingJob 不为 null！
                    if (taskStatus.contentObserverJobInstance != null
                            && taskStatus.contentObserverJobInstance.mChangedAuthorities != null) {
                        // We are stopping this job, but it is going to be replaced by this given
                        // incoming job.  We want to propagate our state over to it, so we don't
                        // lose any content changes that had happend since the last one started.
                        // If there is a previous job associated with the new job, propagate over
                        // any pending content URI trigger reports.
                        if (incomingJob.contentObserverJobInstance == null) {
                            // 给新 job 创建对应的 jobInstance
                            incomingJob.contentObserverJobInstance = new JobInstance(incomingJob);
                        }
                        // 用旧的 jobInstance 来初始化新的 JobInstance 的 uris/authorities
                        incomingJob.contentObserverJobInstance.mChangedAuthorities
                                = taskStatus.contentObserverJobInstance.mChangedAuthorities;
                        incomingJob.contentObserverJobInstance.mChangedUris
                                = taskStatus.contentObserverJobInstance.mChangedUris;
                        // 清空旧得 jobInstance 的 uris/authorities
                        taskStatus.contentObserverJobInstance.mChangedAuthorities = null;
                        taskStatus.contentObserverJobInstance.mChangedUris = null;
                    }
                    // We won't detach the content observers here, because we want to
                    // allow them to continue monitoring so we don't miss anything...  and
                    // since we are giving an incomingJob here, we know this will be
                    // immediately followed by a start tracking of that job.
                } else {
                    // But here there is no incomingJob, so nothing coming up, so time to detach.
                    taskStatus.contentObserverJobInstance.detachLocked();
                    taskStatus.contentObserverJobInstance = null;
                }
            }
            if (DEBUG) {
                Slog.i(TAG, "No longer tracking job " + taskStatus);
            }
            // 从监视队列中删除这个 job！
            mTrackedTasks.remove(taskStatus);
        }
```
这段代码的逻辑很简单：

- 如果 incomingJob 不为 null，说明有新任务会取代旧任务！
- 如果 incomingJob 为 null，说明是取消当前的这个任务！

这里调用了 JobInstance 的 detachLocked 方法：
### 6.2.1 JI.detachLocked

遍历 JobInstance 所依赖的所有观察者，解除依赖；取消依赖为 0 的观察者的注册；从 ContentObserverController 的缓存中删除这个观察者！
```java
        void detachLocked() {
            final int N = mMyObservers.size();
            for (int i=0; i<N; i++) {
                // 从这个 jobInstance 相关联的所有 ObserverInstance 移除对自身的关联！
                final ObserverInstance obs = mMyObservers.get(i);
                obs.mJobs.remove(this);
                 
                // 如果这个观察者依赖的 JobInstance 为 0！
                if (obs.mJobs.size() == 0) {
                    if (DEBUG) {
                        Slog.i(TAG, "Unregistering observer " + obs + " for " + obs.mUri.getUri());
                    }
                    // 取消这个观察者的注册
                    mContext.getContentResolver().unregisterContentObserver(obs);
                    ArrayMap<JobInfo.TriggerContentUri, ObserverInstance> observerOfUser =
                            mObservers.get(obs.mUserId);

                    if (observerOfUser !=  null) {
                        // 并将这个观测者从 ContentObserverController 的缓存中删除！
                        observerOfUser.remove(obs.mUri);
                    }
                }
            }
        }
```
不多说了！
## 6.3 prepareForExecutionLocked

这个是执行 job 前的准备操作！
```java
    @Override
    public void prepareForExecutionLocked(JobStatus taskStatus) {
        if (taskStatus.hasContentTriggerConstraint()) {
            if (taskStatus.contentObserverJobInstance != null) {
                taskStatus.changedUris = taskStatus.contentObserverJobInstance.mChangedUris;
                taskStatus.changedAuthorities = taskStatus.contentObserverJobInstance.mChangedAuthorities;

                taskStatus.contentObserverJobInstance.mChangedUris = null;
                taskStatus.contentObserverJobInstance.mChangedAuthorities = null;
            }
        }
    }
```
这个很简单吧，不说了！
# 7 DeviceIdleJobsController
这个是 Doze 模式的控制器，我们先来看看构造器：
```java
    private DeviceIdleJobsController(JobSchedulerService jobSchedulerService, Context context,
            Object lock) {
        super(jobSchedulerService, context, lock);
        mJobSchedulerService = jobSchedulerService;

        // 获得 PowerManagerService 对象！
        mPowerManager = (PowerManager) mContext.getSystemService(Context.POWER_SERVICE);

        // 获得 DOZE 模式的 DeviceIdleController 服务对象！
        mLocalDeviceIdleController =
                LocalServices.getService(DeviceIdleController.LocalService.class);

        // 这里注册了一个广播接收者！
        final IntentFilter filter = new IntentFilter();
        filter.addAction(PowerManager.ACTION_DEVICE_IDLE_MODE_CHANGED);
        filter.addAction(PowerManager.ACTION_LIGHT_DEVICE_IDLE_MODE_CHANGED);
        filter.addAction(PowerManager.ACTION_POWER_SAVE_WHITELIST_CHANGED);
        mContext.registerReceiverAsUser(
                mBroadcastReceiver, UserHandle.ALL, filter, null, null);
    }
```
注册了一个广播接收者，来接受这三个广播：

- PowerManager.ACTION_DEVICE_IDLE_MODE_CHANGED；
- PowerManager.ACTION_LIGHT_DEVICE_IDLE_MODE_CHANGED；
- PowerManager.ACTION_POWER_SAVE_WHITELIST_CHANGED；

## 7.1 BroadcastReceiver

我们来看看这里注册的这个广播接收者：
```java
    private final BroadcastReceiver mBroadcastReceiver = new BroadcastReceiver() {
        @Override
        public void onReceive(Context context, Intent intent) {

            final String action = intent.getAction();
            // 设备 idle 状态改变的广播
            if (PowerManager.ACTION_LIGHT_DEVICE_IDLE_MODE_CHANGED.equals(action)
                    || PowerManager.ACTION_DEVICE_IDLE_MODE_CHANGED.equals(action)) {
                    
                updateIdleMode(mPowerManager != null
                        ? (mPowerManager.isDeviceIdleMode()
                                || mPowerManager.isLightDeviceIdleMode())
                        : false);
            // doze 模式白名单改变的广播
            } else if (PowerManager.ACTION_POWER_SAVE_WHITELIST_CHANGED.equals(action)) {
                updateWhitelist();

            }
        }
    };
```
这里在接收到指定的广播后，会调用指定的方法，我们一个一个看：
```java
    // 更新 doze 模式的白名单！
    void updateWhitelist() {
        synchronized (mLock) {
            if (mLocalDeviceIdleController != null) {
            
                // 将最新的白名单保存到 mDeviceIdleWhitelistAppIds 数组中！
                mDeviceIdleWhitelistAppIds =
                        mLocalDeviceIdleController.getPowerSaveWhitelistUserAppIds();
                if (LOG_DEBUG) {
                    Slog.d(LOG_TAG, "Got whitelist " + Arrays.toString(mDeviceIdleWhitelistAppIds));
                }
            }
        }
    }
```
这个方法很简单，更新 mDeviceIdleWhitelistAppIds 白名单数组！

接着来看 updateIdleMode 方法，当受到设备状态改变的广播后，需要通知 JSS：
```java
    void updateIdleMode(boolean enabled) {
        boolean changed = false;
        // Need the whitelist to be ready when going into idle
        if (mDeviceIdleWhitelistAppIds == null) {
        
            // 如果白名单为 null，就先去更新 doze 白名单！
            updateWhitelist();
        }
        synchronized (mLock) {
            if (mDeviceIdleMode != enabled) { // 如果这次的状态和上一次不一样，那就要触发 job！
                changed = true; // changed 置为 true
            }
            mDeviceIdleMode = enabled;
            if (LOG_DEBUG) Slog.d(LOG_TAG, "mDeviceIdleMode=" + mDeviceIdleMode);
            
            mJobSchedulerService.getJobStore().forEachJob(mUpdateFunctor); //　更新 job 的状态！
        }

        // Inform the job scheduler service about idle mode changes
        if (changed) {
            mStateChangedListener.onDeviceIdleStateChanged(enabled); // 回调通知 jobScheduler，设备状态改变了！
        }
    }
```
这里会执行一个 JobStatusFunctor 的操作：
```java
    final JobStore.JobStatusFunctor mUpdateFunctor = new JobStore.JobStatusFunctor() {
        
        // 对 JobStore 中的每个 job 执行 process 操作！
        @Override public void process(JobStatus jobStatus) {
            updateTaskStateLocked(jobStatus);
        }
    };
```
接着来看：
```java
    private void updateTaskStateLocked(JobStatus task) {
    
        // 判断这个 job 的 uid 是否在 doze 模式的白名单中！
        final boolean whitelisted = isWhitelistedLocked(task);
        
        // 是否满足 doze 模式的约束条件
        final boolean enableTask = !mDeviceIdleMode || whitelisted;
        
        // 更新 job 的状态！
        task.setDeviceNotDozingConstraintSatisfied(enableTask, whitelisted);
    }
```
这里最后会调用：setDeviceNotDozingConstraintSatisfied 方法！

# 8 附录
## 8.1 JobStatus.setConstraintSatisfied

在 JobStatus 中，有着两个变量：

```java
    // Constraints.
    final int requiredConstraints;
    int satisfiedConstraints = 0;
```

这里解释一下：

- requiredConstraints：这是一个 final 类型的变量，是在 JobInfo 创建的时候，设置的所有的约束条件，一旦设置，就不可变了！
- satisfiedConstraints：它的每个二进制位是可变的，用来动态记录 job 的约束条件是否发生了变化！

在上面，我们看到，每个控制器都有调用这样的方法，这个方法，很有意思，我们看一下，参数传递：

- int constraint：某一个约束条件！
- boolean state：这一次是否满足条件！

```java
    boolean setConstraintSatisfied(int constraint, boolean state) {
        // 判断上一次是否满足约束条件！
        boolean old = (satisfiedConstraints&constraint) != 0;
        // 如果相等，说明约束条件没有变化，无需改变 job 的状态，返回 false！
        if (old == state) {
            return false;
        }
        // 如果不相等，说明约束条件变化了，那就要返回 true！
        // 并根据 state 的值来修改 satisfiedConstraints 的 constraint 位！
        satisfiedConstraints = (satisfiedConstraints&~constraint) | (state ? constraint : 0);
        return true;
    }
```

返回值说明：

- 返回 false：说明约束条件保持不变，无需修改  satisfiedConstraints 的指定二进制位！
- 返回 true：说明约束条件发生了变化，可能是从满足变为不满足，也可能是从不满足变为满足，最终的状态动态记录在 satisfiedConstraints 中！

注意：
satisfiedConstraints&~constraint 的作用是：将 constraint 位置为 0！
(satisfiedConstraints&~constraint) | (state ? constraint : 0) 的作用是：根据 state 的值，将 satisfiedConstraints 的二进制位 constraint 置为 1/0；

## 8.1 JobStatus.isConstraintSatisfied
这个方法的作用很简单，判断此时 jobStatus 是否满足指定条件：
```java
    boolean isConstraintSatisfied(int constraint) {
        return (satisfiedConstraints&constraint) != 0;
    }
```

## 8.3 JobSchedulerService

前面可以看到，每个控制器，最后都调用了 JSS 的如下方法：

- ConnectivityController：                  onControllerStateChanged、onRunJobNow
- TimeController：                          onControllerStateChanged、onRunJobNow
- IdleController：                          onControllerStateChanged
- BatteryController：                       onControllerStateChanged、onRunJobNow
- AppIdleController：                       onControllerStateChanged
- ContentObserverController：               onControllerStateChanged
- DeviceIdleJobsController：                onDeviceIdleStateChanged

接下来，我们看下这几个方法：
```java
    @Override
    public void onControllerStateChanged() {// 控制器状态变化，
        mHandler.obtainMessage(MSG_CHECK_JOB).sendToTarget();
    }

    @Override
    public void onRunJobNow(JobStatus jobStatus) {
        mHandler.obtainMessage(MSG_JOB_EXPIRED, jobStatus).sendToTarget();
    }
```
这两个方法很简单，区别在于：

- MSG_CHECK_JOB：让 JobSchedulerService 去，遍历 JobStore 执行相应的 job！
- MSG_JOB_EXPIRED：立刻执行这个 job！

接着：

```java
    @Override
    public void onDeviceIdleStateChanged(boolean deviceIdle) {
        synchronized (mLock) {
            if (deviceIdle) {
                // When becoming idle, make sure no jobs are actively running,
                // except those using the idle exemption flag.

                for (int i=0; i<mActiveServices.size(); i++) {
                    JobServiceContext jsc = mActiveServices.get(i);
                    final JobStatus executing = jsc.getRunningJob();
                    if (executing != null
                            && (executing.getFlags() & JobInfo.FLAG_WILL_BE_FOREGROUND) == 0) {
                        jsc.cancelExecutingJob(JobParameters.REASON_DEVICE_IDLE); // 取消执行 job
                    }
                }
            } else {

                // When coming out of idle, allow thing to start back up.
                if (mReadyToRock) {
                    if (mLocalDeviceIdleController != null) {
                        if (!mReportedActive) {
                            mReportedActive = true;
                            mLocalDeviceIdleController.setJobsActive(true);
                        }
                    }
                }
                mHandler.obtainMessage(MSG_CHECK_JOB).sendToTarget();
            }
        }
    }
```
先到这里吧！

## 8.3 Controller 和对应的条件

我们来总结一下，Controller 和其对应的约束条件：
























































