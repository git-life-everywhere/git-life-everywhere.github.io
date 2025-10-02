# JobScheduler第 8 篇 - JobInfo JobStatus 和 JobParameters
title: JobScheduler第 8 篇 - JobInfo JobStatus 和 JobParameters
date: 2016/04/29 20:46:25
categories: 
- AndroidFramework源码分析
- JobScheduler任务调度
tags: JobScheduler任务调度 
---

基于 Android 7.1.1 源码分析，本文为原创，转载请说明出处，谢谢！

# 前言
这一篇我们来分析一下 JSS 中的一些对象，看看我们能对 job 做哪些属性的设置：

- JobInfo 
- JobStatus
- JobParameters

# 1 JobInfo
我们先来看看 JobInfo 的简化结构：

```java
public class JobInfo implements Parcelable {

    public static final class Builder {
    
        public JobInfo build() {
            return job;
        }
    }
}
```
这里使用了设计模式中的建造者模式，通过 build 方法，返回最终的 JobInfo！

## 1.1 builder
我们先来看看 buidler 中的方法和参数，我们用一张表格来说明一下：

|属性|属性默认值|属性解释|
|:--|:--|:--|
|int mJobId||Job 的 Id|
|ComponentName mJobService|-|Job 对应的 JobService 组件|
|PersistableBundle mExtras|PersistableBundle.EMPTY|job 重启保留需要恢复的数据|
|int mPriority|PRIORITY_DEFAULT|job 优先级|
|int mFlags|-|-|
|boolean mRequiresCharging|-|约束条件：充电|
|boolean mRequiresDeviceIdle|-|约束条件：doze 模式|
|int mNetworkType|-|约束条件：网络类型|
|ArrayList<TriggerContentUri> mTriggerContentUris|-|约束条件：数据库 Uri|
|long mTriggerContentUpdateDelay|-1|数据库改变后，job 的延迟执行时间|
|long mTriggerContentMaxDelay|-1|数据库改变后，job 的最大延迟执行时间|
|boolean mIsPersisted||job 是否重启保留，需要RECEIVE_BOOT_COMPLETED 权限|
|long mMinLatencyMillis|-|job 延迟执行时间|
|long mMaxExecutionDelayMillis|-|job 最大延迟执行时间，一旦达到该时间，无论条件是否满足，任务都会执行|
|boolean mIsPeriodic|-|job 是否是周期性的|
|**boolean mHasEarlyConstraint**|-|job 是否设置了延迟执行时间（周期和非周期都有）|
|**boolean mHasLateConstraint**|-|job 是否设置了最大延迟执行时间（周期和非周期都有）|
|long mIntervalMillis|-|job 执行的周期时间间隔|
|long mFlexMillis|-|job 执行周期末的一个时间窗，任何任务在这个时间窗中都可能被执行|
|long mInitialBackoffMillis|DEFAULT_INITIAL_BACKOFF_MILLIS|job 第一次尝试重试的等待间隔，单位为毫秒|
|int mBackoffPolicy|DEFAULT_BACKOFF_POLICY|job 对应的退避策略|
|boolean mBackoffPolicySet|False|job 是否设置了设置退避 / 重试策略，类似网络原理中的冲突退避，当一个任务的调度失败时需要重试，所采取的策略。|

接着来看属性和其对应的方法：

|属性|设置方法|
|:--|:--|
|int mJobId|Builder 构造器|
|ComponentName mJobService|Builder 构造器|
|PersistableBundle mExtras|setExtras|
|int mPriority|setPriority|
|int mFlags|setFlags|
|boolean mRequiresCharging|setRequiresCharging|
|boolean mRequiresDeviceIdle|setRequiresDeviceIdle|
|int mNetworkType|setRequiredNetworkType|
|ArrayList<TriggerContentUri> mTriggerContentUris|addTriggerContentUri|
|long mTriggerContentUpdateDelay|setTriggerContentUpdateDelay|
|long mTriggerContentMaxDelay|setTriggerContentMaxDelay|
|boolean mIsPersisted|setPersisted|
|long mMinLatencyMillis|setMinimumLatency|
|long mMaxExecutionDelayMillis|setOverrideDeadline|
|boolean mIsPeriodic|setPeriodic|
|boolean mHasEarlyConstraint|setMinimumLatency、setPeriodic|
|boolean mHasLateConstraint|setOverrideDeadline、setPeriodic|
|long mIntervalMillis|setPeriodic|
|long mFlexMillis|setPeriodic|
|long mInitialBackoffMillis|setBackoffCriteria|
|int mBackoffPolicy|setBackoffCriteria|
|boolean mBackoffPolicySet|setBackoffCriteria|

接下来我们分析一下细节：

> 网络类型的取值：

   - NETWORK_TYPE_NONE：默认值。表示与网络状态无关
   - NETWORK_TYPE_ANY：必须连接网络
   - NETWORK_TYPE_NOT_ROAMING：必须连接非漫游的网络
   - NETWORK_TYPE_UNMETERED：必须连接非计费的网络

> 退避 | 重试策略：

   - long mInitialBackoffMillis：第一次尝试重试的等待间隔，单位为毫秒，预设的取值有下面 2 种：
       - DEFAULT_INITIAL_BACKOFF_MILLIS：30 秒
       - MAX_BACKOFF_DELAY_MILLIS：5 小时
   - int mBackoffPolicy：退避策略
       - BACKOFF_POLICY_LINEAR：0
            - 对应的重试时间计算公式是:
            - **current_time + initial_backoff_millis * num_failures, num_failures >= 1**
       - BACKOFF_POLICY_EXPONENTIAL：1
            - 对应的重试时间计算公式是： 
            - **current_time + initial_backoff_millis * 2 ^ (num_failures - 1), num_failures >= 1**

> 优先级的取值：

 - PRIORITY_DEFAULT：0
 - PRIORITY_SYNC_EXPEDITED：10
 - PRIORITY_SYNC_INITIALIZATION：20
 - PRIORITY_FOREGROUND_APP：30
 - PRIORITY_TOP_APP：40
 - PRIORITY_ADJ_OFTEN_RUNNING：-40
 - PRIORITY_ADJ_ALWAYS_RUNNING：-80

> 周期时间的取值：

 - MIN_PERIOD_MILLIS：15 * 60 * 1000L，最小的执行周期时间，15 分钟
 - MIN_FLEX_MILLIS：5 * 60 * 1000L，最小的时间窗，5分钟

接下来，我们来分析下 build 方法：
```java
        public JobInfo build() {
            
            // 至少要设置一个约束条件！
            if (!mHasEarlyConstraint && !mHasLateConstraint && !mRequiresCharging &&
                    !mRequiresDeviceIdle && mNetworkType == NETWORK_TYPE_NONE &&
                    mTriggerContentUris == null) {
                throw new IllegalArgumentException("You're trying to build a job with no " +
                        "constraints, this is not allowed.");
            }

            mExtras = new PersistableBundle(mExtras);  // Make our own copy.

            // 周期性的 job 不能设置延迟执行时间和最大延迟执行时间，也不能设置数据库变化的约束条件！
            // 重启保留的 job 不能设置数据库变化的约束条件！
            // 设置了退避 | 重试策略的 job 不能设置 DOZE 约束条件！
            if (mIsPeriodic && (mMaxExecutionDelayMillis != 0L)) {
                throw new IllegalArgumentException("Can't call setOverrideDeadline() on a " +
                        "periodic job.");
            }
            if (mIsPeriodic && (mMinLatencyMillis != 0L)) {
                throw new IllegalArgumentException("Can't call setMinimumLatency() on a " +
                        "periodic job");
            }
            if (mIsPeriodic && (mTriggerContentUris != null)) {
                throw new IllegalArgumentException("Can't call addTriggerContentUri() on a " +
                        "periodic job");
            }
            if (mIsPersisted && (mTriggerContentUris != null)) {
                throw new IllegalArgumentException("Can't call addTriggerContentUri() on a " +
                        "persisted job");
            }
            if (mBackoffPolicySet && mRequiresDeviceIdle) {
                throw new IllegalArgumentException("An idle mode job will not respect any" +
                        " back-off policy, so calling setBackoffCriteria with" +
                        " setRequiresDeviceIdle is an error.");
            }
            
            // 创建 JobInfo 对象！
            JobInfo job = new JobInfo(this);

            if (job.isPeriodic()) {
                if (job.intervalMillis != job.getIntervalMillis()) {
                    StringBuilder builder = new StringBuilder();
                    builder.append("Specified interval for ")
                            .append(String.valueOf(mJobId))
                            .append(" is ");
                    formatDuration(mIntervalMillis, builder);
                    builder.append(". Clamped to ");
                    formatDuration(job.getIntervalMillis(), builder);
                    Log.w(TAG, builder.toString());
                }
                if (job.flexMillis != job.getFlexMillis()) {
                    StringBuilder builder = new StringBuilder();
                    builder.append("Specified flex for ")
                            .append(String.valueOf(mJobId))
                            .append(" is ");
                    formatDuration(mFlexMillis, builder);
                    builder.append(". Clamped to ");
                    formatDuration(job.getFlexMillis(), builder);
                    Log.w(TAG, builder.toString());
                }
            }

            return job;
        }
```
这里我们来总结一下：

- 至少要设置一个约束条件！
- 周期性的 job 不能设置延迟执行时间和最大延迟执行时间，也不能设置数据库变化的约束条件！
- 重启保留的 job 不能设置数据库变化的约束条件！
- 设置了退避 | 重试策略的 job 不能设置 DOZE 约束条件！

## 1.2 JobInfo
我们来看看构造器：
```java
    private JobInfo(JobInfo.Builder b) {
        jobId = b.mJobId;
        extras = b.mExtras;
        service = b.mJobService;
        requireCharging = b.mRequiresCharging;
        requireDeviceIdle = b.mRequiresDeviceIdle;
        triggerContentUris = b.mTriggerContentUris != null
                ? b.mTriggerContentUris.toArray(new TriggerContentUri[b.mTriggerContentUris.size()])
                : null;
        triggerContentUpdateDelay = b.mTriggerContentUpdateDelay;
        triggerContentMaxDelay = b.mTriggerContentMaxDelay;
        networkType = b.mNetworkType;
        minLatencyMillis = b.mMinLatencyMillis;
        maxExecutionDelayMillis = b.mMaxExecutionDelayMillis;
        isPeriodic = b.mIsPeriodic;
        isPersisted = b.mIsPersisted;
        intervalMillis = b.mIntervalMillis;
        flexMillis = b.mFlexMillis;
        initialBackoffMillis = b.mInitialBackoffMillis;
        backoffPolicy = b.mBackoffPolicy;
        hasEarlyConstraint = b.mHasEarlyConstraint;
        hasLateConstraint = b.mHasLateConstraint;
        priority = b.mPriority;
        flags = b.mFlags;
    }
```
可以看出，参数的意义和 builder 是一样的，我们来看看：

|属性|获取方法|
|:--|:--|
|int jobId |getId|
|ComponentName service|getService|
|PersistableBundle extras|getExtras|
|int priority|getPriority|
|int flags|getFlags|
|boolean requiresCharging|isRequireCharging|
|boolean requiresDeviceIdle|isRequireDeviceIdle|
|int networkType|getNetworkType|
|ArrayList<TriggerContentUri> triggerContentUris|getTriggerContentUris|
|long triggerContentUpdateDelay|getTriggerContentUpdateDelay|
|long triggerContentMaxDelay|getTriggerContentMaxDelay|
|boolean isPersisted|isPersisted|
|long minLatencyMillis|getMinLatencyMillis|
|long maxExecutionDelayMillis|getMaxExecutionDelayMillis|
|boolean isPeriodic|isPeriodic|
|boolean hasEarlyConstraint|hasEarlyConstraint|
|boolean hasLateConstraint|hasLateConstraint|
|long intervalMillis|getIntervalMillis|
|long flexMillis|getFlexMillis|
|long initialBackoffMillis|getInitialBackoffMillis|
|int backoffPolicy|getBackoffPolicy|

我们接着看：
# 2 JobStatus
我们 schedule 一个 jobInfo 后，JSS 会创建对应的 JobStatus，我们先来看看他的构造器：
```java
    // 通过一个 JobStatus 创建！
    public JobStatus(JobStatus jobStatus) {
    
        this(jobStatus.getJob(), jobStatus.getUid(),
                jobStatus.getSourcePackageName(), jobStatus.getSourceUserId(),
                jobStatus.getSourceTag(), jobStatus.getNumFailures(),
                jobStatus.getEarliestRunTime(), jobStatus.getLatestRunTimeElapsed());
    }

    // 从本地文件 Jobs.xml 中读取数据，创建 JobStatus
    public JobStatus(JobInfo job, int callingUid, String sourcePackageName, int sourceUserId,
            String sourceTag, long earliestRunTimeElapsedMillis, long latestRunTimeElapsedMillis) {
            
        this(job, callingUid, sourcePackageName, sourceUserId, sourceTag, 0,
                earliestRunTimeElapsedMillis, latestRunTimeElapsedMillis);
    }

    // 根据参数创建一个 JobStatus
    public JobStatus(JobStatus rescheduling, long newEarliestRuntimeElapsedMillis,
                      long newLatestRuntimeElapsedMillis, int backoffAttempt) {
                      
        this(rescheduling.job, rescheduling.getUid(),
                rescheduling.getSourcePackageName(), rescheduling.getSourceUserId(),
                rescheduling.getSourceTag(), backoffAttempt, newEarliestRuntimeElapsedMillis,
                newLatestRuntimeElapsedMillis);
    }
```
最终都会调用这个构造器方法，参数传递：

- JobInfo job：jobInfo 对象
- int callingUid：job 所属进程的 uid
- String sourcePackageName：job 所属应用的包名
- int sourceUserId：job所属的设备用户
- String tag：
- int numFailures：失败次数
- long earliestRunTimeElapsedMillis： 最早执行时间
- long latestRunTimeElapsedMillis：最晚执行时间

```java
    private JobStatus(JobInfo job, int callingUid, String sourcePackageName,
            int sourceUserId, String tag, int numFailures, long earliestRunTimeElapsedMillis,
            long latestRunTimeElapsedMillis) {

        this.job = job;
        this.callingUid = callingUid;

        int tempSourceUid = -1;
        if (sourceUserId != -1 && sourcePackageName != null) {
            try {
                tempSourceUid = AppGlobals.getPackageManager().getPackageUid(sourcePackageName, 0,
                        sourceUserId);
            } catch (RemoteException ex) {
                // Can't happen, PackageManager runs in the same process.
            }
        }

        if (tempSourceUid == -1) {
            this.sourceUid = callingUid;
            this.sourceUserId = UserHandle.getUserId(callingUid);
            this.sourcePackageName = job.getService().getPackageName();
            this.sourceTag = null;

        } else {
            this.sourceUid = tempSourceUid;
            this.sourceUserId = sourceUserId;
            this.sourcePackageName = sourcePackageName;
            this.sourceTag = tag;

        }

        this.batteryName = this.sourceTag != null
                ? this.sourceTag + ":" + job.getService().getPackageName()
                : job.getService().flattenToShortString();
        this.tag = "*job*/" + this.batteryName;

        this.earliestRunTimeElapsedMillis = earliestRunTimeElapsedMillis;
        this.latestRunTimeElapsedMillis = latestRunTimeElapsedMillis;
        this.numFailures = numFailures;

        // requiredConstraints 使用二进制位记录 job 所依赖的约束条件，依赖某个条件，对应的二进制位置为 1！
        int requiredConstraints = 0;

        if (job.getNetworkType() == JobInfo.NETWORK_TYPE_ANY) {
            requiredConstraints |= CONSTRAINT_CONNECTIVITY;
        }
        if (job.getNetworkType() == JobInfo.NETWORK_TYPE_UNMETERED) {
            requiredConstraints |= CONSTRAINT_UNMETERED;
        }
        if (job.getNetworkType() == JobInfo.NETWORK_TYPE_NOT_ROAMING) {
            requiredConstraints |= CONSTRAINT_NOT_ROAMING;
        }
        if (job.isRequireCharging()) {
            requiredConstraints |= CONSTRAINT_CHARGING;
        }
        if (earliestRunTimeElapsedMillis != NO_EARLIEST_RUNTIME) {
            requiredConstraints |= CONSTRAINT_TIMING_DELAY;
        }
        if (latestRunTimeElapsedMillis != NO_LATEST_RUNTIME) {
            requiredConstraints |= CONSTRAINT_DEADLINE;
        }
        if (job.isRequireDeviceIdle()) {
            requiredConstraints |= CONSTRAINT_IDLE;
        }
        if (job.getTriggerContentUris() != null) {
            requiredConstraints |= CONSTRAINT_CONTENT_TRIGGER;
        }

        this.requiredConstraints = requiredConstraints;
    }
```

我们可以看到，构造函数很简单，没有多么复杂的，我们来总结一下，JobStatus 中的参数：

|参数|参数解释|
|:--|:--|
|JobInfo job|JobInfo 对象|
|int callingUid	|调用者进程 uid|
|String batteryName	||
|String sourcePackageName|	job 所属应用的包名|
|int sourceUserId|	job 所属设备用户|
|int sourceUid|	job 所属应用的 uid|
|String sourceTag	||
|String tag	||
|long earliestRunTimeElapsedMillis|	job 最早执行的时间|
|long latestRunTimeElapsedMillis	|job 最晚执行的时间|
|int numFailures	|job 执行失败的次数|
|int requiredConstraints	|每个二进制位表示 job 所依赖的约束条件|
|int satisfiedConstraints = 0	|每个二进制位表示 job 对应的约束条件是否满足|
|boolean dozeWhitelisted|	job 是否在 doze 白名单中|
|ArraySet<Uri> changedUris|	job 依赖的数据库的 Uri|
|ArraySet<String> changedAuthorities|	job 依赖的数据库的 Uri|
|int lastEvaluatedPriority|	job 的优先级|
|int overrideState = 0	|shell 命令用到|


我们继续来看：

> 约束条件的表示

- 用户可以设置的约束条件：
   - CONSTRAINT_CHARGING：1<<0，充电
   - CONSTRAINT_TIMING_DELAY：1<<1，延迟处理
   - CONSTRAINT_DEADLINE：1<<2，超时处理
   - CONSTRAINT_IDLE：1<<3，doze 模式
   - CONSTRAINT_UNMETERED：1<<4，必须连接非漫游的网络
   - CONSTRAINT_CONNECTIVITY：1<<5，必须连接网络
   - CONSTRAINT_APP_NOT_IDLE：1<<6，app 处于非空闲状态
   - CONSTRAINT_CONTENT_TRIGGER：1<<7，数据库变化
   - CONSTRAINT_NOT_ROAMING：1<<9，必须连接非计费的网络

- 用户无法设置的约束条件：
   - CONSTRAINT_DEVICE_NOT_DOZING：1<<8，不受 doze 模式限制，这个是由 DeviceIdleJobsController 获得 doze 模式白名单的更新，动态的对 uid 在白名单中的 job 进行设置的！

> 最早执行时间和最晚执行时间（）

- 对于周期性 job
   -  最早执行时间： 
   -  **earliestRunTimeElapsedMillis = latestRunTimeElapsedMillis - job.getFlexMillis()**
   -  最晚执行时间： 
   -  **latestRunTimeElapsedMillis = elapsedNow + job.getIntervalMillis()**
- 对于非周期性 job
   -  最早执行时间：
       - 设置了延迟执行时间：
       - **earliestRunTimeElapsedMillis = elapsedNow + job.getMinLatencyMillis()**
       - 未设置延迟执行时间： 
       - **earliestRunTimeElapsedMillis = NO_EARLIEST_RUNTIME = 0L**
   -  最晚执行时间：
       - 设置了最大延迟执行时间：
       - **latestRunTimeElapsedMillis = elapsedNow + job.getMaxExecutionDelayMillis()**
       - 未设置最大延迟执行时间： 
       - **latestRunTimeElapsedMillis = NO_LATEST_RUNTIME = Long.MAX_VALUE**

   

# 3 JobParameters
JobParameters 是 job 执行时的参数，我们来看看他的参数：

- int jobId：job 的 id！
- PersistableBundle extras：重启恢复的数据！
- IBinder callback：JobSchedulerContext 的 binder 代理对象！
- boolean overrideDeadlineExpired：job 是否已经超时！
- Uri[] mTriggeredContentUris：监控的数据库的 Uri！
- String[] mTriggeredContentAuthorities：监控的数据库的 Authorities！
- int stopReason：job 停止的原因！
   - int REASON_CANCELED = 0，默认为 cancel
   - int REASON_CONSTRAINTS_NOT_SATISFIED = 1
   - int REASON_PREEMPT = 2
   - int REASON_TIMEOUT = 3
   - int REASON_DEVICE_IDLE = 4 