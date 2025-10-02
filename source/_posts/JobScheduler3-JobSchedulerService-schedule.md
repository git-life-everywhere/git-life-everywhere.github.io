# JobScheduler第 3 篇 - JobSchedulerService - schedule
title: JobScheduler第 3 篇 - JobSchedulerService - schedule
date: 2016/04/18 20:46:25
categories: 
- AndroidFramework源码分析
- JobScheduler任务调度 
tags: JobScheduler任务调度  
---


基于 Android 7.1.1 源码分析，本文为作者原创，转载请说明出处，谢谢。
# 前言
我们先从基本的方法开始，也就是 schedule 方法，方法参数传递：

- JobInfo job：需要 schedule 的任务！
- int uId：调用方的 uid！

```java
    public int schedule(JobInfo job, int uId) {
        return scheduleAsPackage(job, uId, null, -1, null);
    }

    public int scheduleAsPackage(JobInfo job, int uId, String packageName, int userId, String tag) {

        // 创建新的 jobStatus
        JobStatus jobStatus = JobStatus.createFromJobInfo(job, uId, packageName, userId, tag);
        try {
            if (ActivityManagerNative.getDefault().getAppStartMode(uId,
                    job.getService().getPackageName()) == ActivityManager.APP_START_MODE_DISABLED) {
                Slog.w(TAG, "Not scheduling job " + uId + ":" + job.toString()
                        + " -- package not allowed to start");
                return JobScheduler.RESULT_FAILURE;
            }
        } catch (RemoteException e) {
        }

        if (DEBUG) Slog.d(TAG, "SCHEDULE: " + jobStatus.toShortString());
        JobStatus toCancel;
        synchronized (mLock) {
            // 判断应用设置的任务数量是否超过上限：100
            if (ENFORCE_MAX_JOBS && packageName == null) {
                if (mJobs.countJobsForUid(uId) > MAX_JOBS_PER_APP) {
                    Slog.w(TAG, "Too many jobs for uid " + uId);
                    throw new IllegalStateException("Apps may not schedule more than "
                                + MAX_JOBS_PER_APP + " distinct jobs");
                }
            }

            // 如果同一个id，之前已经注册了一个任务，取消上一个任务
            toCancel = mJobs.getJobByUidAndJobId(uId, job.getId());
            if (toCancel != null) {
                cancelJobImpl(toCancel, jobStatus);
            }

            // 开始追踪该任务
            startTrackingJob(jobStatus, toCancel);
        }

        // 向 system_server 进程的主线程发送 message：MSG_CHECK_JOB
        mHandler.obtainMessage(MSG_CHECK_JOB).sendToTarget();
        return JobScheduler.RESULT_SUCCESS;
    }
```
这里的 ENFORCE_MAX_JOBS 是一个 boolean 值，表示是否限制每个应用的 job 数！

>    private static final int MAX_JOBS_PER_APP = 100;

这里 MAX_JOBS_PER_APP 表示每个应用最大的 Job 数！

这里总结一下 schedule 方法的主要流程：

- 根据传入的 JobInfo，创建 JobStatus；
- 如果已经注册过一个相同 uid，相同 jobId 的 job，就取消之前注册过的；
- 将新 job 加入到 JobStore 中，并通知 controller 开始监控新 schedule 的 job！
- 发送 MSG_CHECK_JOB 消息给 JobHandler，执行 job！

我们是使用 schedule 方法将我们需要的任务注册到系统中的，传入的参数主要是：

- JobInfo：封装任务的基本信息！
- uId：调用者的 uid！

我们接着来看，先调用 JobStatus 的 createFromJobInfo ，创建 JobStatus：

# 1 JobStatus.createFromJobInfo
参数传递：job, uId, null, -1, null：
```java
    public static JobStatus createFromJobInfo(JobInfo job, int callingUid, String sourcePackageName, int sourceUserId, String tag) {
        // 从开机到现在的时间
        final long elapsedNow = SystemClock.elapsedRealtime();
        final long earliestRunTimeElapsedMillis, latestRunTimeElapsedMillis;

        // 判断这个任务是否是周期性的
        if (job.isPeriodic()) {
            latestRunTimeElapsedMillis = elapsedNow + job.getIntervalMillis();
            earliestRunTimeElapsedMillis = latestRunTimeElapsedMillis - job.getFlexMillis();
        } else {
            earliestRunTimeElapsedMillis = job.hasEarlyConstraint() ?
                    elapsedNow + job.getMinLatencyMillis() : NO_EARLIEST_RUNTIME;
            latestRunTimeElapsedMillis = job.hasLateConstraint() ?
                    elapsedNow + job.getMaxExecutionDelayMillis() : NO_LATEST_RUNTIME;
        }
        
        // 创建对应的 JobStatus 对象！
        return new JobStatus(job, callingUid, sourcePackageName, sourceUserId, tag, 0,
                earliestRunTimeElapsedMillis, latestRunTimeElapsedMillis);
    }
```


这里通过 JobInfo 创建了对应的 JobStatus 对象，参数传递：
job, uId, null, -1, null，0，earliestRunTimeElapsedMillis，latestRunTimeElapsedMillis。
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
我们后面会详细的开一篇，来讲讲这些参数的，这里先不看！
# 2 JobStore.countJobsForUid
这里是统计 uid 对用的应用有多少个任务：
```java
    public int countJobsForUid(int uid) {
        return mJobSet.countJobsForUid(uid);
    }
```
JobStore 有一个集合，用来存放所有的 Job！

> final JobSet mJobSet;

我们来看看这个 JobSet
```java
    static class JobSet {
        // Key is the getUid() originator of the jobs in each sheaf
        // mJobs 的 key 是应用的 uid！
        private SparseArray<ArraySet<JobStatus>> mJobs;
        public JobSet() {
            mJobs = new SparseArray<ArraySet<JobStatus>>();
        }
        ... ... ... ...
        // We only want to count the jobs that this uid has scheduled on its own
        // behalf, not those that the app has scheduled on someone else's behalf.
        // 统计 uid 对应的应用注册的任务数！
        public int countJobsForUid(int uid) {
            int total = 0;
            ArraySet<JobStatus> jobs = mJobs.get(uid);
            if (jobs != null) {
                for (int i = jobs.size() - 1; i >= 0; i--) {
                    JobStatus job = jobs.valueAt(i);
                    if (job.getUid() == job.getSourceUid()) {
                        total++;
                    }
                }
            }
            return total;
        }
        ... ... ... ...
    }
```
JobSet 有很多的其他方法：getJobsByXXX，add，remove，getXXX，等等的方法，这里我们先不看！

# 3 JobStore.getJobByUidAndJobId
接着是根据 uid 和 jobId，来获得一个任务，这里的目的是判断是否之前已经注册过了一个相同的任务：
```java
    public JobStatus getJobByUidAndJobId(int uid, int jobId) {
        return mJobSet.get(uid, jobId);
    }
```
还是调用的是 JobSet.get 方法：
```java
        public JobStatus get(int uid, int jobId) {
            ArraySet<JobStatus> jobs = mJobs.get(uid);
            if (jobs != null) {
                for (int i = jobs.size() - 1; i >= 0; i--) {
                    JobStatus job = jobs.valueAt(i);
                    if (job.getJobId() == jobId) {
                        return job;
                    }
                }
            }
            return null;
        }
```
这个很简单，不详细说了！

# 4 JSS.cancelJobImpl
如果之前已经注册过一个任务了，需要先取消掉之前的任务！
```java
    private void cancelJobImpl(JobStatus cancelled, JobStatus incomingJob) {
        if (DEBUG) Slog.d(TAG, "CANCEL: " + cancelled.toShortString());
        // 停止追踪任务!
        stopTrackingJob(cancelled, incomingJob, true /* writeBack */);

        synchronized (mLock) {
            
            // 如果这个任务在等待队列中，移除它。
            if (mPendingJobs.remove(cancelled)) {
                mJobPackageTracker.noteNonpending(cancelled);
            }
            
            // 如果正在运行，取消这个任务！
            stopJobOnServiceContextLocked(cancelled, JobParameters.REASON_CANCELED);
            
            // 更新 jss 的状态！
            reportActive();
        }
    }
```
这里主要做的是：停止对这个任务的监视，同时，将这个任务移除等待队列，如果这个任务正在运行，那么就要取消它，我们一个一个来看！

## 4.1 JSS.stopTrackingJob
我们先来看第一个方法，参数传递：

- JobStatus jobStatus：要被取消的任务；
- JobStatus incomingJob：本次要注册的任务；
- boolean writeBack：true；

```java
    private boolean stopTrackingJob(JobStatus jobStatus, JobStatus incomingJob,
            boolean writeBack) {
        synchronized (mLock) {

            // 从 JobStore 和 controller 的监控队列中移除这个 job！
            final boolean removed = mJobs.remove(jobStatus, writeBack);
            if (removed && mReadyToRock) {
                for (int i=0; i<mControllers.size(); i++) {
                    StateController controller = mControllers.get(i);
                    // 停止 job 监控！
                    controller.maybeStopTrackingJobLocked(jobStatus, incomingJob, false);
                }
            }
            return removed;
        }
    }
```
这里的 mJobs 是 JobStore 对象！

### 4.1.1 JobStore.remove

我们来看看这个移除操作：
```java
    public boolean remove(JobStatus jobStatus, boolean writeBack) {
    
        // 先从 mJobSet 中移除要删除的 JobStatus！
        boolean removed = mJobSet.remove(jobStatus);
        if (!removed) {
            if (DEBUG) {
                Slog.d(TAG, "Couldn't remove job: didn't exist: " + jobStatus);
            }
            return false;
        }
        // 如果 writeBack 是 true，并且 jobStatus 是 isPersisted 的，那就要更新 Jobs.xml 文件！
        if (writeBack && jobStatus.isPersisted()) {
            maybeWriteStatusToDiskAsync();
        }
        return removed;
    }
```
从 JobSet 集合中移除 Job，并且，如果需要写入操作，并且这个 Job 是设备重启后仍然需要保留的，那就要调用 remove 方法，将更新后的 JobSet 写入到 system/job/jobs.xml，因为所有 Persisted 为 true 的 Job ，都是会被写入到这个文件中去的，用于重启恢复！！

我们来看 maybeWriteStatusToDiskAsync 方法：
```java
    private void maybeWriteStatusToDiskAsync() {
        mDirtyOperations++;
        if (mDirtyOperations >= MAX_OPS_BEFORE_WRITE) {
            if (DEBUG) {
                Slog.v(TAG, "Writing jobs to disk.");
            }
            mIoHandler.post(new WriteJobsMapToDiskRunnable());
        }
    }
    ... ... ... ... ... ...
 
    private class WriteJobsMapToDiskRunnable implements Runnable {
        @Override
        public void run() {
            final long startElapsed = SystemClock.elapsedRealtime();
            final List<JobStatus> storeCopy = new ArrayList<JobStatus>();
            synchronized (mLock) {
                // Clone the jobs so we can release the lock before writing.
                // 这里是将 mJobSet 拷贝一份到 storeCopy 中。
                mJobSet.forEachJob(new JobStatusFunctor() {
                    @Override
                    public void process(JobStatus job) {
                        if (job.isPersisted()) {
                            storeCopy.add(new JobStatus(job));
                        }
                    }
                });
            }
            // 将更新后的拷贝写入 jobs.xml。
            writeJobsMapImpl(storeCopy);
            if (JobSchedulerService.DEBUG) {
                Slog.v(TAG, "Finished writing, took " + (SystemClock.elapsedRealtime()
                        - startElapsed) + "ms");
            }
        }

        ... ... ... ... 
    
    }
```
我们接着看！

### 4.1.2 SC.maybeStopTrackingJobLocked

接着，就是遍历控制器集合，让控制器停止对该任务的监视，参数传递：

- JobStatus jobStatus：要被删除的 Job！
- JobStatus incomingJob：同一个 uid，同一个 jobId 的要被注册执行的 Job！
- boolean forUpdate：false

```java
    /**
     * Remove task - this will happen if the task is cancelled, completed, etc.
     */
    public abstract void maybeStopTrackingJobLocked(JobStatus jobStatus, JobStatus incomingJob,
            boolean forUpdate);
```
StateController 是一个抽象类，具体的实现，我们在 JobSchedulerService 的构造器中有看到：
```java
        mControllers.add(ConnectivityController.get(this));
        mControllers.add(TimeController.get(this));
        mControllers.add(IdleController.get(this));
        mControllers.add(BatteryController.get(this));
        mControllers.add(AppIdleController.get(this));
        mControllers.add(ContentObserverController.get(this));
        mControllers.add(DeviceIdleJobsController.get(this));
```
这里有很多的控制器：
#### 4.1.2.1 ConnectivityController
我们先来看看 ConnectivityController 类的方法：
```java
    @Override
    public void maybeStopTrackingJobLocked(JobStatus jobStatus, JobStatus incomingJob,
            boolean forUpdate) {
        if (jobStatus.hasConnectivityConstraint() || jobStatus.hasUnmeteredConstraint()
                || jobStatus.hasNotRoamingConstraint()) {
            mTrackedJobs.remove(jobStatus);
        }
    }
```
其他控制器的方法很类似的哦！
## 4.2 JobPackageTracker.noteNonpending

这个就是这样的
```java
    public void noteNonpending(JobStatus job) {
        final long now = SystemClock.uptimeMillis();
        mCurDataSet.decPending(job.getSourceUid(), job.getSourcePackageName(), now);
        rebatchIfNeeded(now);
    }
```
这里我们先不看！
## 4.3 JSS.stopJobOnServiceContextLocked

如果 Job 有在运行，那就停止它，参数传递：cancelled, JobParameters.REASON_CANCELED
```java
    private boolean stopJobOnServiceContextLocked(JobStatus job, int reason) {
        // 遍历 mActiveServices 集合；
        for (int i=0; i<mActiveServices.size(); i++) {
            JobServiceContext jsc = mActiveServices.get(i);
            final JobStatus executing = jsc.getRunningJob();
            // 找到匹配的 jobStatus 对象；
            if (executing != null && executing.matches(job.getUid(), job.getJobId())) {
                jsc.cancelExecutingJob(reason);
                return true;
            }
        }
        return false;
    }
```
这里调用了 JobServiceContext 的 cancelExecutingJob 这个方法，取消正在运行中的 job！
### 4.3.1 JobServiceContext.cancelExecutingJob
我们进入 JobServiceContext 文件中来看看：
```java
    /** Called externally when a job that was scheduled for execution should be cancelled. */
    void cancelExecutingJob(int reason) {

        // 发送一个 MSG_CANCEL 消息给 JobServiceHandler！
        mCallbackHandler.obtainMessage(MSG_CANCEL, reason, 0 /* unused */).sendToTarget();
    }
```
其中：mCallbackHandler = new JobServiceHandler(looper)，这里的 looper 是 system_server 的主线程的 looper，这个在第二篇初始化和启动就可以看到！

这里我们向 JobServiceHandler 发送了一个 MSG_CANCEL 的消息，我们去看看：
```java
   private class JobServiceHandler！ extends Handler {
        @Override
        public void handleMessage(Message message) {
            switch (message.what) {

                ... ... ... ...

                // 收到 MSG_CANCEL 的消息！
                case MSG_CANCEL:
                    // 如果这个 Job，已经完成了，就不处理！
                    if (mVerb == VERB_FINISHED) {
                        if (DEBUG) {
                            Slog.d(TAG, "Trying to process cancel for torn-down context, ignoring.");
                        }
                        return;
                    }

                    // 保存停止的原因！
                    mParams.setStopReason(message.arg1);
                    if (message.arg1 == JobParameters.REASON_PREEMPT) {
                        // 如果原因是因为优先级取代，就用 mPreferredUid 保存被取代的 job 的 uid！
                        mPreferredUid = mRunningJob != null ? mRunningJob.getUid() :
                                NO_PREFERRED_UID;
                    }
                    
                    // 执行取消任务～
                    handleCancelH();
                    break;

                case MSG_TIMEOUT:
                    handleOpTimeoutH();
                    break;

                case MSG_SHUTDOWN_EXECUTION:
                    closeAndCleanupJobH(true /* needsReschedule */);
                    break;

                default:
                    Slog.e(TAG, "Unrecognised message: " + message);
            }
        }

    ... ... ... ...

  }
```
我们进入到 handleCancelH 方法中来看看：
```java
        private void handleCancelH() {
            if (JobSchedulerService.DEBUG) {
                Slog.d(TAG, "Handling cancel for: " + mRunningJob.getJobId() + " "
                        + VERB_STRINGS[mVerb]);
            }
            switch (mVerb) { // 判断当前的 job 的状态！
                case VERB_BINDING:
                case VERB_STARTING:
                    // 将变量 mCancelled 置为 true；
                    mCancelled.set(true);
                    break;
                    
                case VERB_EXECUTING:
                    // 如果当前的 job 是正在执行状态，就先判断应用是否调用过 jobFinished 主动结束任务
                    // 如果有，就取消这次 cancel！
                    if (hasMessages(MSG_CALLBACK)) {
                        // If the client has called jobFinished, ignore this cancel.
                        return;
                    }

                    // 发送停止的消息！
                    sendStopMessageH();
                    break;
                    
                case VERB_STOPPING:
                    // Nada.
                    break;
                default:
                    Slog.e(TAG, "Cancelling a job without a valid verb: " + mVerb);
                    break;
            }
        }
```
mVerb 使用来保存 Job 的状态的，一个 Job 在 JobServiceContext 中会有如下的几种状态：

    static final int VERB_BINDING = 0;
    static final int VERB_STARTING = 1;
    static final int VERB_EXECUTING = 2;
    static final int VERB_STOPPING = 3;
    static final int VERB_FINISHED = 4;

这里我先不看！
## 4.4 JSS.reportActive
最后，调用 reportActive 方法，通知 JSS 的状态！
```java
    void reportActive() {
        // active is true if pending queue contains jobs OR some job is running.
        boolean active = mPendingJobs.size() > 0;
        if (mPendingJobs.size() <= 0) {
            for (int i=0; i<mActiveServices.size(); i++) {
                final JobServiceContext jsc = mActiveServices.get(i);
                final JobStatus job = jsc.getRunningJob();
                if (job != null
                        && (job.getJob().getFlags() & JobInfo.FLAG_WILL_BE_FOREGROUND) == 0
                        && !job.dozeWhitelisted) {
                    // We will report active if we have a job running and it is not an exception
                    // due to being in the foreground or whitelisted.
                    active = true;
                    break;
                }
            }
        }

        // 当状态发生改变时，通知 DeviceIdleController 当前 JSS 的状态！
        if (mReportedActive != active) {
            mReportedActive = active;
            if (mLocalDeviceIdleController != null) {
                mLocalDeviceIdleController.setJobsActive(active);
            }
        }
    }

```
这个 reportActive 是将 JSS 的状态通知给 DeviceIdleController，用于 DOZE 模式，这里简单介绍下 DOZE 模式：

当用户在连续的一段时间内没有使用手机，就延缓终端中 APP 后台的CPU和网络活动，以达到减少电量消耗的目的，这时就进入 DOZE 模式，进入 DOZE 模式后的系统有如下约束：

- 暂停网络访问。 
- 系统忽略所有的 WakeLock。 
- 标准的 AlarmManager alarms 被延缓到下一个 maintenance window。 
   - 但使用 AlarmManager 的 setAndAllowWhileIdle、setExactAndAllowWhileIdle 和 setAlarmClock 时，alarms 定义的事件仍会启动，在这些 alarms 启动前，系统会短暂地退出Doze模式。 
- 系统不再进行 WiFi 扫描。 
- 系统不允许 sync adapters 运行。 
- **系统不允许 JobScheduler 运行**。

注意到最后一条了吧，这个就是 reportActive 的意义，我们继续看：

# 5 JSS.startTrackingJob
接下来，就是开始 track 任务，参数传递：

- JobStatus jobStatus：本次注册的 job！
- JobStatus lastJob：同一个 uid，同一个 jobId，上次注册的已经被取消的任务，可以为 null！

```java
    private void startTrackingJob(JobStatus jobStatus, JobStatus lastJob) {
        synchronized (mLock) {

            // 添加到 JobStore 中!
            final boolean update = mJobs.add(jobStatus);

            if (mReadyToRock) {
                // 遍历控制器，执行监控操作！
                for (int i = 0; i < mControllers.size(); i++) {
                    StateController controller = mControllers.get(i);
                    
                    if (update) { // 发生了相同 uid 和 jobId 的取代操作，先停止对旧的 job 的监控！
                        controller.maybeStopTrackingJobLocked(jobStatus, null, true);
                    }
                    
                    // 监控新的 jobStatus！
                    controller.maybeStartTrackingJobLocked(jobStatus, lastJob);
                }
            }
        }
    }
```
第一步，可以看到，先将这一次要注册执行的任务，加入到 JobStore 中：
## 5.1 JobStore.add
这里是将要好注册执行的 Job 添加到 JobStore 中，这里和上面有些类似：
```java
    public boolean add(JobStatus jobStatus) {
        // 取代之前已经被添加过相同 uid 和 jobId 的 jobStatus！
        boolean replaced = mJobSet.remove(jobStatus);
        // 将新的 job 添加到 JobSets 集合中！
        mJobSet.add(jobStatus);

        if (jobStatus.isPersisted()) { // 如果是 persist 类型的 job，还需要写到 Job.xml 文件中去！
            maybeWriteStatusToDiskAsync();
        }

        if (DEBUG) {
            Slog.d(TAG, "Added job status to store: " + jobStatus);
        }

        return replaced; // 返回是否有取代发生！
    }
```
返回值表示是否有取代发生，在 add 的过程中，会判断是否有相同 uid 和 jobId 的 jobStatus 存在，如果有，就要取代之前的！

如果这个 Job 是 isPersisted 的，那就需要更新 Jobs.xml 文件！
## 5.2 SC.maybeStopTrackingJobLocked
这里先要停止 track 这个任务，参数传递：

- JobStatus jobStatus：要被注册执行的新的 Job！
- JobStatus incomingJob：null
- boolean forUpdate：true
```java
    /**
     * Remove task - this will happen if the task is cancelled, completed, etc.
     */
    public abstract void maybeStopTrackingJobLocked(JobStatus jobStatus, JobStatus incomingJob,
            boolean forUpdate);
```
具体的实现代码是在每个 Controller 中实现的，我们来看一个最简单的 Controller：
### 5.2.1 ConnectivityController
代码如下，逻辑很简单：
```java
    @Override
    public void maybeStopTrackingJobLocked(JobStatus jobStatus, JobStatus incomingJob,
            boolean forUpdate) {
        if (jobStatus.hasConnectivityConstraint() || jobStatus.hasUnmeteredConstraint()
                || jobStatus.hasNotRoamingConstraint()) {
            // 从监控列表中移除这个 job！
            mTrackedJobs.remove(jobStatus);
        }
    }
```
接着看：
## 5.3 SC.maybeStartTrackingJobLocked
接着，调用这个方法，来开始 track 任务，方法参数：

- JobStatus jobStatus：这次要注册的任务
- JobStatus lastJob：同一个 uid，同一个 jobId 已经被取消掉的上一个任务，可以为 null！

```java
    public abstract void maybeStartTrackingJobLocked(JobStatus jobStatus, JobStatus lastJob);
```
这里同样是一个抽象接口，具体的实现是子类：
### 5.3.1 ConnectivityController
让我们来看看 ConnectivityController 对象的这个方法：
```
    @Override
    public void maybeStartTrackingJobLocked(JobStatus jobStatus, JobStatus lastJob) {
        if (jobStatus.hasConnectivityConstraint() || jobStatus.hasUnmeteredConstraint()
                || jobStatus.hasNotRoamingConstraint()) {
            // 判断当前的 job 约束条件是否满足，如果满足，就可以立即执行！
            updateConstraintsSatisfied(jobStatus);
            // 将这个 Job 添加到 ConnectivityController 的跟踪列表中！
            mTrackedJobs.add(jobStatus);
        }
    }
```
首先，判断 Job 是否有设置和 Connectivity 网络相关的属性：

- jobStatus.hasConnectivityConstraint()
- jobStatus.hasUnmeteredConstraint()
- jobStatus.hasNotRoamingConstraint()

调用了 updateConstraintsSatisfied 方法来判断当前的约束条件是否满足，同时更新 JobStatus 的状态！
```java
    private boolean updateConstraintsSatisfied (JobStatus jobStatus) {
        final boolean ignoreBlocked = (jobStatus.getFlags() & JobInfo.FLAG_WILL_BE_FOREGROUND) != 0;
        // 获得当前的网络状态信息：NetworkInfo
        final NetworkInfo info = mConnManager.getActiveNetworkInfoForUid(jobStatus.getSourceUid(),
                ignoreBlocked);
        // 判断约束条件是否满足；
        final boolean connected = (info != null) && info.isConnected();
        final boolean unmetered = connected && !info.isMetered();
        final boolean notRoaming = connected && !info.isRoaming();

        boolean changed = false;
        
        // 根据约束条件来更新 JobStatus 的信息！ 
        changed |= jobStatus.setConnectivityConstraintSatisfied(connected);
        changed |= jobStatus.setUnmeteredConstraintSatisfied(unmetered);
        changed |= jobStatus.setNotRoamingConstraintSatisfied(notRoaming);
        return changed;
    }
```
返回的 changed 表示：约束条件是否变化！

最后将 jobStatus 加入到指定的 Controller 的监控列表中！

# 6 [JSS.JobHandler].MSG_CHECK_JOB

接着，就是向 JobHandler 发送了 MSG_CHECK_JOB 的消息，我们来看看：
```
    private class JobHandler extends Handler {
        public JobHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message message) {
            synchronized (mLock) {
                if (!mReadyToRock) {
                    return;
                }
            }

            switch (message.what) {
                case MSG_JOB_EXPIRED:
                    ... ... ... ...
                    break;

                case MSG_CHECK_JOB: // 接收到 MSG_CHECK_JOB 的消息！
                    synchronized (mLock) {
                        if (mReportedActive) { // 为 true，表示 JSS 通知了设备管理器，自己处于活跃状态！
                            // if jobs are currently being run, queue all ready jobs for execution.
                            queueReadyJobsForExecutionLockedH();
                        } else {
                            // Check the list of jobs and run some of them if we feel inclined.
                            maybeQueueReadyJobsForExecutionLockedH();
                        }
                    }
                    break;

                case MSG_CHECK_JOB_GREEDY:
                    synchronized (mLock) {
                        queueReadyJobsForExecutionLockedH();
                    }
                    break;

                case MSG_STOP_JOB:
                    cancelJobImpl((JobStatus)message.obj, null);
                    break;
            }

            // 处理 mPendingJobs 中的任务！
            maybeRunPendingJobsH();

            // Don't remove JOB_EXPIRED in case one came along while processing the queue.
            removeMessages(MSG_CHECK_JOB); 
        }
        ... ... ... ... ...
}
```
mReportActive 为 true 的情况：
有等待中的 job （mPendingJobs.size 大于 0）或者有正在执行中的 job （JobServiceContext.getRunningJob 不为 null）

接着，收到了 MSG_CHECK_JOB 的消息，开始遍历队列，执行准备好的任务！
## 6.1 queueReadyJobsForExecutionLockedH
当 mReport 下面是这个方法的代码：
```java
        private void queueReadyJobsForExecutionLockedH() {
            if (DEBUG) {
                Slog.d(TAG, "queuing all ready jobs for execution:");
            }

            noteJobsNonpending(mPendingJobs);
            mPendingJobs.clear();
            mJobs.forEachJob(mReadyQueueFunctor);
            mReadyQueueFunctor.postProcess();

            if (DEBUG) {
                final int queuedJobs = mPendingJobs.size();
                if (queuedJobs == 0) {
                    Slog.d(TAG, "No jobs pending.");
                } else {
                    Slog.d(TAG, queuedJobs + " jobs queued.");
                }
            }
        }
```
这个方法和 maybeQueueReadyJobsForExecutionLockedH 很类似，我们先来看下面的这个方法：

## 6.2 maybeQueueReadyJobsForExecutionLockedH
```java
        private void maybeQueueReadyJobsForExecutionLockedH() {
            if (DEBUG) Slog.d(TAG, "Maybe queuing ready jobs...");
            noteJobsNonpending(mPendingJobs);

            // 先清空 mPendingJobs 集合!
            mPendingJobs.clear();

            // 将准备好的 Job 加入到 JobHandler 内部的 runnableJobs 集合中。
            mJobs.forEachJob(mMaybeQueueFunctor);
            // 将 runnableJobs 加入到 mPendingJobs 集合中!
            mMaybeQueueFunctor.postProcess();
        }
```
### 6.2.1 mPendingJobs.clear
首先清除了：mPendingJobs 集合，为这次执行做准备：
> mPendingJobs.clear();

接着，这里传入了 mMaybeQueueFunctor 对象！
```java
    public void forEachJob(JobStatusFunctor functor) {
        mJobSet.forEachJob(functor);
    }
```
进入了 JobSet：
```java
        public void forEachJob(JobStatusFunctor functor) {
            for (int uidIndex = mJobs.size() - 1; uidIndex >= 0; uidIndex--) {
            
                // 遍历 JobStore 中的所有 job！
                ArraySet<JobStatus> jobs = mJobs.valueAt(uidIndex);
                for (int i = jobs.size() - 1; i >= 0; i--) {
                
                    // 对每个 uid 的应用程序的 Job，执行下面操作：
                    functor.process(jobs.valueAt(i));
                }
            }
        }
```

可以看出，这对所有的 Job，都调用了 MaybeReadyJobQueueFunctor 的 process 方法：
### 6.2.2 MaybeReadyJobQueueFunctor.process
```java
        class MaybeReadyJobQueueFunctor implements JobStatusFunctor {
            int chargingCount;
            int idleCount;
            int backoffCount;
            int connectivityCount;
            int contentCount;
            
            List<JobStatus> runnableJobs; // 符合条件的即将运行的 Job
            
            public MaybeReadyJobQueueFunctor() {
                reset();
            }

            // Functor method invoked for each job via JobStore.forEachJob()
            @Override
            public void process(JobStatus job) {
            
                // 判断这个 Job 是否是准备了！
                if (isReadyToBeExecutedLocked(job)) {
                    try {
                        if (ActivityManagerNative.getDefault().getAppStartMode(job.getUid(),
                                job.getJob().getService().getPackageName())
                                == ActivityManager.APP_START_MODE_DISABLED) {
                            Slog.w(TAG, "Aborting job " + job.getUid() + ":"
                                    + job.getJob().toString() + " -- package not allowed to start");
                            mHandler.obtainMessage(MSG_STOP_JOB, job).sendToTarget();
                            return;
                        }
                    } catch (RemoteException e) {
                    }
                    if (job.getNumFailures() > 0) {
                        backoffCount++;
                    }
                    if (job.hasIdleConstraint()) {
                        idleCount++;
                    }
                    if (job.hasConnectivityConstraint() || job.hasUnmeteredConstraint()
                            || job.hasNotRoamingConstraint()) {
                        connectivityCount++;
                    }
                    if (job.hasChargingConstraint()) {
                        chargingCount++;
                    }
                    if (job.hasContentTriggerConstraint()) {
                        contentCount++;
                    }
                    if (runnableJobs == null) {
                        runnableJobs = new ArrayList<>();
                    }

                    // 将满足条件的 Job，先加入到 runnableJobs 集合！
                    runnableJobs.add(job);
                } else if (areJobConstraintsNotSatisfiedLocked(job)) {
                    stopJobOnServiceContextLocked(job,
                            JobParameters.REASON_CONSTRAINTS_NOT_SATISFIED);
                }
            }
```

这里调用了 isReadyToBeExecutedLocked 方法来判断，这个 job 是否已经准备好了：

```java
        private boolean isReadyToBeExecutedLocked(JobStatus job) {
            final boolean jobReady = job.isReady(); // job 是否准备好了
            final boolean jobPending = mPendingJobs.contains(job); // job 是否已经在等待队列中
            final boolean jobActive = isCurrentlyActiveLocked(job); // job 是否已经在 JobSerivceContext 中运行了！
            final int userId = job.getUserId();
            final boolean userStarted = ArrayUtils.contains(mStartedUsers, userId); // job 所在的设备用户是否运行！

            final boolean componentPresent; // 表示 job 对应的应用程序的 JobService 组件是否存在！

            try {
                componentPresent = (AppGlobals.getPackageManager().getServiceInfo(
                        job.getServiceComponent(), PackageManager.MATCH_DEBUG_TRIAGED_MISSING,
                        userId) != null);
            } catch (RemoteException e) {
                throw e.rethrowAsRuntimeException();
            }
            if (DEBUG) {
                Slog.v(TAG, "isReadyToBeExecutedLocked: " + job.toShortString()
                        + " ready=" + jobReady + " pending=" + jobPending
                        + " active=" + jobActive + " userStarted=" + userStarted
                        + " componentPresent=" + componentPresent);
            }
            return userStarted && componentPresent && jobReady && !jobPending && !jobActive;
        }
```
可以看出，一个准备好的 Job 要满足这些条件：

- It's ready.
- It's not pending.  
- It's not already running on a JSC.
- The user that requested the job is running.
- The component is enabled and runnable.

最后，调用 MaybeQueueFunctor.postProcess 方法：
### 6.2.3 MaybeReadyJobQueueFunctor.postProcess
```java
            public void postProcess() {
                
                // 判断一下条件！
                if (backoffCount > 0 ||
                        idleCount >= mConstants.MIN_IDLE_COUNT ||
                        connectivityCount >= mConstants.MIN_CONNECTIVITY_COUNT ||
                        chargingCount >= mConstants.MIN_CHARGING_COUNT ||
                        contentCount >= mConstants.MIN_CONTENT_COUNT ||
                        (runnableJobs != null
                                && runnableJobs.size() >= mConstants.MIN_READY_JOBS_COUNT)) {
                    if (DEBUG) {
                        Slog.d(TAG, "maybeQueueReadyJobsForExecutionLockedH: Running jobs.");
                    }
                    
                    noteJobsPending(runnableJobs);

                    // 将 runnableJobs 加入到 mPendingJobs 集合中!
                    mPendingJobs.addAll(runnableJobs);
                } else {
                    if (DEBUG) {
                        Slog.d(TAG, "maybeQueueReadyJobsForExecutionLockedH: Not running anything.");
                    }
                }
                // Be ready for next time
                reset();
            }
```
该功能:

- 先将 JobStore 的 JobSet 中满足条件的 job 对应的 JobStatus 加入 runnableJobs 队列；
- 再将 runnableJobs 中满足触发条件的 JobStatus 加入到 mPendingJobs 队列；

## 6.3 maybeRunPendingJobsH

接着，就是处理 mPendingJobs 中的 Job：
```java
        private void maybeRunPendingJobsH() {
            synchronized (mLock) {
                if (DEBUG) {
                    Slog.d(TAG, "pending queue: " + mPendingJobs.size() + " jobs.");
                }
                
                // 分配 Job 给 JSC!
                assignJobsToContextsLocked();
                
                // 更新 JSS 的状态！
                reportActive();
            }
        }
```
结合就是调用 JSS 的 assignJobsToContextLocked 方法：
### 6.3.1 JSS.assignJobsToContextsLocked

这个方法其实从名字上就可以看出，就是把 Job 分配给 JobServiceContext 方法，我们去那个方法里面看一下：
```java
    private void assignJobsToContextsLocked() {
        if (DEBUG) {
            Slog.d(TAG, printPendingQueue());
        }

        // 根据当前系统的内存级别，设定最大的活跃 Job 数，保存到 mMaxActiveJobs！
        int memLevel;
        try {
            memLevel = ActivityManagerNative.getDefault().getMemoryTrimLevel();
        } catch (RemoteException e) {
            memLevel = ProcessStats.ADJ_MEM_FACTOR_NORMAL;
        }
        switch (memLevel) {
            case ProcessStats.ADJ_MEM_FACTOR_MODERATE:
                mMaxActiveJobs = mConstants.BG_MODERATE_JOB_COUNT;
                break;
            case ProcessStats.ADJ_MEM_FACTOR_LOW:
                mMaxActiveJobs = mConstants.BG_LOW_JOB_COUNT;
                break;
            case ProcessStats.ADJ_MEM_FACTOR_CRITICAL:
                mMaxActiveJobs = mConstants.BG_CRITICAL_JOB_COUNT;
                break;
            default:
                mMaxActiveJobs = mConstants.BG_NORMAL_JOB_COUNT;
                break;
        }

        JobStatus[] contextIdToJobMap = mTmpAssignContextIdToJobMap; // 大小为 16，和 mActiveServices 相对应！
        boolean[] act = mTmpAssignAct;

        int[] preferredUidForContext = mTmpAssignPreferredUidForContext; // 大小为 16，和 mActiveServices 相对应！
        int numActive = 0;
        int numForeground = 0;

        for (int i = 0; i < MAX_JOB_CONTEXTS_COUNT; i++) {

            // 获得每一个 JobSchedulerContext 和其内部运行着的 JobStatus（可为 null）
            final JobServiceContext js = mActiveServices.get(i);
            final JobStatus status = js.getRunningJob();

            // 初始化 contextIdToJobMap，里面保存当前运行着的 Job，和空闲的用于增加的 Job 位置!
            // 然后，分别计算活跃 job 和前台 job 的个数！ 
            if ((contextIdToJobMap[i] = status) != null) {
                numActive++;
                if (status.lastEvaluatedPriority >= JobInfo.PRIORITY_TOP_APP) {
                    numForeground++;
                }
            }

            act[i] = false;
            // 如果当前的 JSC 之前有发生过 job 优先级取代，将上一次的被取代的 job 的 uid 保存到 
            preferredUidForContext[i] = js.getPreferredUid();
        }

        if (DEBUG) {
            Slog.d(TAG, printContextIdToJobMap(contextIdToJobMap, "running jobs initial"));
        }
        
        // 遍历 mPendingJos 任务集合，尽可能为每一个 job 找到合适的 JobSchedulerContext!
        for (int i=0; i<mPendingJobs.size(); i++) {
            JobStatus nextPending = mPendingJobs.get(i);

            // 判断当前的 Job 是不是在 contextIdToJobMap 中了，即他是不是已经在运行了!
            int jobRunningContext = findJobContextIdFromMap(nextPending, contextIdToJobMap);
            if (jobRunningContext != -1) {
                continue;
            }

            // 计算 Job 的优先级!
            final int priority = evaluateJobPriorityLocked(nextPending);
            nextPending.lastEvaluatedPriority = priority;

            // 遍历 contextIdToJobMap
            // 给这个即将被执行的 Job 找一个合适的 context，至少要满足两个中的一个要求：1、可利用；2、优先级最低！
            int minPriority = Integer.MAX_VALUE;
            int minPriorityContextId = -1;

            for (int j=0; j < MAX_JOB_CONTEXTS_COUNT; j++) {
                JobStatus job = contextIdToJobMap[j];
                int preferredUid = preferredUidForContext[j];
                if (job == null) { 
                    if ((numActive < mMaxActiveJobs ||
                            (priority >= JobInfo.PRIORITY_TOP_APP &&
                                    numForeground < mConstants.FG_JOB_COUNT)) &&
                            (preferredUid == nextPending.getUid() ||
                                    preferredUid == JobServiceContext.NO_PREFERRED_UID)) {

                        // 如果 context 原有的 job 为空，并且同时满足以下 2 个条件：
                        // 1、当前活跃 job 数小于最大活跃数，或者 job 优先级不低于 PRIORITY_TOP_APP 且前台 job 数小于 mConstants.FG_JOB_COUNT
                        // 2、JobSchedulerContext 中保存的之前被取代的 job 的 uid 等于当前的 job 的 id 或者等于 NO_PREFERRED_UID
                        // 这个 JobSchedulerContext 是合适的，退出循环，处理下一个 job！
                        minPriorityContextId = j;
                        break;
                    }

                    continue;
                }

                // 如果 context 原有的 job 不为空，且 uid 和即将执行的 job 不一样，那么这个 JobSchedulerContext 不合适!
                if (job.getUid() != nextPending.getUid()) { 
                    continue;
                }
                
                // 如果 context 原有的 job 不为空，且 uid 相同，已有 job 优先级大于等于即将执行的 job，那么这个 JobSchedulerContext 不合适!
                if (evaluateJobPriorityLocked(job) >= nextPending.lastEvaluatedPriority) {
                    continue;
                }
                
                // 如果存在 JobSchedulerContext 原有的 job 不为空，且 uid 相同，已有 job 优先级低于即将执行的 job，那么就选择优先级最低的那个！
                if (minPriority > nextPending.lastEvaluatedPriority) { 
                    minPriority = nextPending.lastEvaluatedPriority;
                    // 用来存储优先级最低的 JobSchedulerContext 所在的数组下标！
                    minPriorityContextId = j;
                }
            }

            // 找到了合适的 JobSchedulerContext 的下标!
            if (minPriorityContextId != -1) { 

                // 先将这个要被执行的 Job 放入 contextIdToJobMap 的 minPriorityContextId 位置！
                contextIdToJobMap[minPriorityContextId] = nextPending;

                // 对应的 act 位置为 true，表示可以运行!
                act[minPriorityContextId] = true;
                // 活跃 job 数加 1；
                numActive++; 
                if (priority >= JobInfo.PRIORITY_TOP_APP) {
                    // 如果 job 的权限不低于 PRIORITY_TOP_APP，前台 job 数加 1；
                    numForeground++;
                }
            }
        }

        if (DEBUG) {
            Slog.d(TAG, printContextIdToJobMap(contextIdToJobMap, "running jobs final"));
        }
        
        mJobPackageTracker.noteConcurrency(numActive, numForeground);

        // 执行所有已经分配了 JobSchedulerContext 的 job !
        for (int i=0; i < MAX_JOB_CONTEXTS_COUNT; i++) {
            boolean preservePreferredUid = false;

            if (act[i]) { // art 为 true，表示可运行！
                JobStatus js = mActiveServices.get(i).getRunningJob();
                if (js != null) {
                    if (DEBUG) {
                        Slog.d(TAG, "preempting job: " + mActiveServices.get(i).getRunningJob());
                    }
                    
                    // preferredUid will be set to uid of currently running job.
                    // 如果这个 context 已经在运行了一个优先级低的 job，那就要取消它！
                    mActiveServices.get(i).preemptExecutingJob();
                    preservePreferredUid = true;
                } else {

                    // 从 contextIdToJobMap 中获得即将执行的 Job!
                    final JobStatus pendingJob = contextIdToJobMap[i];
                    if (DEBUG) {
                        Slog.d(TAG, "About to run job on context "
                                + String.valueOf(i) + ", job: " + pendingJob);
                    }

                    // 通知控制器!
                    for (int ic=0; ic<mControllers.size(); ic++) {
                        mControllers.get(ic).prepareForExecutionLocked(pendingJob);
                    }

                    // 使用对应的 JobServiceContext 来执行 Job
                    if (!mActiveServices.get(i).executeRunnableJob(pendingJob)) {
                        Slog.d(TAG, "Error executing " + pendingJob);
                    }

                    // Job 已经开始执行了，从 mPendingJobs 中移除这个 Job!
                    if (mPendingJobs.remove(pendingJob)) {
                        mJobPackageTracker.noteNonpending(pendingJob);
                    }
                }
            }

            // 如果 preservePreferredUid 为 false，说明本次执行，没有发生 job 优先级取代
            // 那就要调用 JobServiceContext 的 clearPreferredUid 清除上一次取代（如果有）保存的 uid！
            if (!preservePreferredUid) { 
                mActiveServices.get(i).clearPreferredUid();
            }
        }
    }
```
接着我们继续来看：
#### 6.3.1.1 JSS.findJobContextIdFromMap

这个方法的作用很简单，就是判断 job 是否已经在 map 集合中了！
```java
    int findJobContextIdFromMap(JobStatus jobStatus, JobStatus[] map) {
        for (int i=0; i<map.length; i++) {
            // 在 contextIdToJobMap 匹配 JobId 和 uid 相同的一个 job！
            if (map[i] != null && map[i].matches(jobStatus.getUid(), jobStatus.getJobId())) {
                return i;
            }
        }
        return -1;
    }
```
#### 6.3.1.2 JSS.evaluateJobPriorityLocked
这个方法是为 job 计算优先级：
```java
    private int adjustJobPriority(int curPriority, JobStatus job) {
    
        // 如果 curPriority 小于 PRIORITY_TOP_APP （40）
        if (curPriority < JobInfo.PRIORITY_TOP_APP) {
            // 获得计算因子，根据计算因子，计算优先级！
            float factor = mJobPackageTracker.getLoadFactor(job);
            if (factor >= mConstants.HEAVY_USE_FACTOR) {
                curPriority += JobInfo.PRIORITY_ADJ_ALWAYS_RUNNING;
            } else if (factor >= mConstants.MODERATE_USE_FACTOR) {
                curPriority += JobInfo.PRIORITY_ADJ_OFTEN_RUNNING;
            }
        }
        // 返回 curPriority；
        return curPriority;
    }

    private int evaluateJobPriorityLocked(JobStatus job) {
        // 获得 job 的优先级
        int priority = job.getPriority();

        // 如果 job 的优先级大于等于 PRIORITY_FOREGROUND_APP（30）
        if (priority >= JobInfo.PRIORITY_FOREGROUND_APP) {
            // 直接利用 job 的 priority 计算优先级！
            return adjustJobPriority(priority, job);
        }
        
        // 如果 job 的优先级小于 PRIORITY_FOREGROUND_APP（30）
        // 就先判断当前的 job 的 uid 是否是前台 uid！
        int override = mUidPriorityOverride.get(job.getSourceUid(), 0);
        if (override != 0) {
            // 如果当前 job 的 uid 是前台 uid，那就以这个 uid 对应的优先级计算当前 job 的优先级！
            return adjustJobPriority(override, job);
        }
        
        // 直接利用 job 的 priority 计算优先级！
        return adjustJobPriority(priority, job);
    }
```

这里的 mUidPriorityOverride 用来保存当前**所有在前台的 uid，以及 uid 所有的 job 的优先级**，下面是 Jss 中定义的一些优先级：
```java
    public static final int PRIORITY_ADJ_ALWAYS_RUNNING = -80;
    public static final int PRIORITY_ADJ_OFTEN_RUNNING = -40;
    public static final int PRIORITY_DEFAULT = 0;
    public static final int PRIORITY_SYNC_EXPEDITED = 10;
    public static final int PRIORITY_SYNC_INITIALIZATION = 20;
    public static final int PRIORITY_FOREGROUND_APP = 30;
    public static final int PRIORITY_TOP_APP = 40;
```
值越小，则优先级越高！

接着源码继续分析：
#### 6.3.1.3 JSC.preemptExecutingJob

这里调用了 JobServiceContext 的 preemptExecutingJob 方法来取消相同 uid 但是优先级更低的任务，之前有说过，如果 Context 已经在执行一个优先级 job，并且这个 job 和即将被执行的 job 属于同一个 uid，那么要取消优先级低的！
```java
    void preemptExecutingJob() {
        Message m = mCallbackHandler.obtainMessage(MSG_CANCEL);

        // 参数为 JobParameters.REASON_PREEMPT，表示要取代！
        m.arg1 = JobParameters.REASON_PREEMPT;
        m.sendToTarget();
    }
```
发送一个 MSG_CANCEL 消息给 JSC 内部的 JobServiceHandler ，附带一个参数，表示原因：JobParameters.REASON_PREEMPT！
##### JobServiceHandler.MSG_CANCEL

我们接着来看：
```java
    private class JobServiceHandler extends Handler {
        JobServiceHandler(Looper looper) {
            super(looper);
        }
        @Override
        public void handleMessage(Message message) {
            switch (message.what) {
                ... ... ... ...
                case MSG_CANCEL:
                    if (mVerb == VERB_FINISHED) { // 这个 mVerb 表示的是，这个 Context 的 job 的当前状态!
                        if (DEBUG) {
                            Slog.d(TAG,
                                   "Trying to process cancel for torn-down context, ignoring.");
                        }
                        return;
                    }
                    mParams.setStopReason(message.arg1);
                    if (message.arg1 == JobParameters.REASON_PREEMPT) {
                        // 如果是取消的原因是优先级取代，就用 mPreferredUid 保存 uid！
                        mPreferredUid = mRunningJob != null ? mRunningJob.getUid() :
                                NO_PREFERRED_UID;
                    }
                    // 执行取消操作!
                    handleCancelH();
                    break;
                case MSG_TIMEOUT:
                    handleOpTimeoutH();
                    break;
                case MSG_SHUTDOWN_EXECUTION:
                    closeAndCleanupJobH(true /* needsReschedule */);
                    break;
                default:
                    Slog.e(TAG, "Unrecognised message: " + message);
            }
        }
       
        ... ... ... ... ... ...

        private void handleCancelH() {
            if (JobSchedulerService.DEBUG) {
                Slog.d(TAG, "Handling cancel for: " + mRunningJob.getJobId() + " "
                        + VERB_STRINGS[mVerb]);
            }
            
            // 这里因为是取消的是这个 context 里面正在执行的 job，所以 mVerb 的值为 VERB_EXECUTING!
            switch (mVerb) { 
                case VERB_BINDING:
                case VERB_STARTING:
                    mCancelled.set(true);
                    break;
                    
                case VERB_EXECUTING:
                    // 如果 client 已经调用了 jobFinished 方法结束了 job，那就 return!
                    if (hasMessages(MSG_CALLBACK)) {
                        // If the client has called jobFinished, ignore this cancel.
                        return;
                    }
                    sendStopMessageH(); // 否则，就发送停止的消息!
                    break;
                    
                case VERB_STOPPING:
                    // Nada.
                    break;
                    
                default:
                    Slog.e(TAG, "Cancelling a job without a valid verb: " + mVerb);
                    break;
            }
        }
        ... ... ... ...
    }
```
如果有优先级低的 job 正在运行，那就 stop job，这时 mVerb 会从：VERB_EXECUTING -> VERB_STOPPING.
```
        /**
         * Already running, need to stop. Will switch {@link #mVerb} from VERB_EXECUTING ->
         * VERB_STOPPING.
         */
        private void sendStopMessageH() {
            removeOpTimeOut();
            if (mVerb != VERB_EXECUTING) {
                Slog.e(TAG, "Sending onStopJob for a job that isn't started. " + mRunningJob);
                closeAndCleanupJobH(false /* reschedule */);
                return;
            }
            try {
                // mVerb 状态变为 VERB_STOPPING！
                mVerb = VERB_STOPPING;
                scheduleOpTimeOut();

                // binder 通信，调用 JobService 的 stopJob 方法
                service.stopJob(mParams); 
            } catch (RemoteException e) {
                Slog.e(TAG, "Error sending onStopJob to client.", e);
                closeAndCleanupJobH(false /* reschedule */);
            }
        }
```
这里的 service 是一个 IJobService 对象，本质是一个 Binder 对象，对应着应用程序进程中的 JobService！
##### JobService.stopJob

注意：这里对于 Binder 机制来说：应用程序的 JobService 所在进程是 Binder 服务端，JobServiceContext 所在的进程 system_server 是 Binder 客户端！也就是说，应用程序定义的 JobService 是被 JobSerivceContext 来 bind 的，所以，你会发现，你无法 override JobService 的 onbind 方法！
```java
    static final class JobInterface extends IJobService.Stub {
        final WeakReference<JobService> mService;
        JobInterface(JobService service) {
            mService = new WeakReference<>(service);
        }

        @Override
        public void startJob(JobParameters jobParams) throws RemoteException {
            JobService service = mService.get();
            if (service != null) {
                service.ensureHandler();
                Message m = Message.obtain(service.mHandler, MSG_EXECUTE_JOB, jobParams);
                m.sendToTarget();
            }
        }

        @Override
        public void stopJob(JobParameters jobParams) throws RemoteException { 
            // 通过 binder 机制来来调用指定应用的 JobService 的 stopJob 方法！
            JobService service = mService.get();
            if (service != null) {
                service.ensureHandler();
                // 发送到 JobService 内部的 JobHandler 对象中！
                Message m = Message.obtain(service.mHandler, MSG_STOP_JOB, jobParams);
                m.sendToTarget();
            }
        }
    }
```
这里发送了 MSG_STOP_JOB 消息给 JobService.JobHandler，我们去 JobHandler 内部去看看：
##### JobHandler.MSG_STOP_JOB

JobHandler 位于应用程序的主线程：
```java
    class JobHandler extends Handler {
        JobHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {

            // 这里获得传递过来的 JobParameters 
            final JobParameters params = (JobParameters) msg.obj;
            switch (msg.what) {

                ... ... ...

                case MSG_STOP_JOB:
                    try {

                        // 调用 JobService onStopJob 方法，把参数传递过去，这里是不是很熟悉，就不多说!
                        // onStopJob 会返回 true / false， true 表示还会 reschedule 这个 job!
                        boolean ret = JobService.this.onStopJob(params);
                        ackStopMessage(params, ret);
                    } catch (Exception e) {
                        Log.e(TAG, "Application unable to handle onStopJob.", e);
                        throw new RuntimeException(e);
                    }
                    break;

                ... ... ... ...

                default:
                    Log.e(TAG, "Unrecognised message received.");
                    break;
            }
        }
        
        ... ... ... ... ...
        
        private void ackStopMessage(JobParameters params, boolean reschedule) {

            // 这里获得了 IJobCallback 对象，这里显示是 Binder 机制，服务端是 JobServiceContext
            final IJobCallback callback = params.getCallback();
            final int jobId = params.getJobId();
            if (callback != null) {
                try {

                    // 发送消息给 JobServiceContext
                    callback.acknowledgeStopMessage(jobId, reschedule);
                } catch(RemoteException e) {
                    Log.e(TAG, "System unreachable for stopping job.");
                }
            } else {
                if (Log.isLoggable(TAG, Log.DEBUG)) {
                    Log.d(TAG, "Attempting to ack a job that has already been processed.");
                }
            }
        }
    }
```
这里的 IJobCallback 又是使用了 Binder 机制，Binder 客户端是应用程序的 JobService 所在进程，Binder 服务端是 JobServiceContext 所在的进程，最后调用的是 JobServiceContext.acknowledgeStopMessage 方法：
##### JobServiceContext.acknowledgeStopMessage

通过 Binder 机制，将回调信息发回给 JobServiceContext：
```java
    @Override
    public void acknowledgeStopMessage(int jobId, boolean reschedule) {
        if (!verifyCallingUid()) {
            return;
        }

        // 发送 MSG_CALLBACK 给 JobServiceHandler
        mCallbackHandler.obtainMessage(MSG_CALLBACK, jobId, reschedule ? 1 : 0)
                .sendToTarget();
    }
```
然后又会发送 MSG_CALLBACK 给 JobServiceContext.JobServiceHandler，这里我们只看关键代码：
##### JobServiceHandler.MSG_CALLBACK
JobServiceHandler 同样的也是位于主线程：
```java
                case MSG_CALLBACK:
                    if (DEBUG) {
                        Slog.d(TAG, "MSG_CALLBACK of : " + mRunningJob
                                + " v:" + VERB_STRINGS[mVerb]);
                    }
                    removeOpTimeOut();
                    if (mVerb == VERB_STARTING) {
                        final boolean workOngoing = message.arg2 == 1;
                        handleStartedH(workOngoing);
                        
                    } else if (mVerb == VERB_EXECUTING ||
                            mVerb == VERB_STOPPING) { // 从前面跟代码，可以看出 mVerb 的值为 VERB_STOPPING.
                        final boolean reschedule = message.arg2 == 1;
                        
                        handleFinishedH(reschedule);
                        
                    } else {
                        if (DEBUG) {
                            Slog.d(TAG, "Unrecognised callback: " + mRunningJob);
                        }
                    }
                    break;
```
接着，调用 JobServiceContext 的 handleFinishedH 方法：
##### JobServiceContext.handleFinishedH
```java
        private void handleFinishedH(boolean reschedule) {
            switch (mVerb) {
                case VERB_EXECUTING:
                case VERB_STOPPING: // 进入这个分支！
                    closeAndCleanupJobH(reschedule); // 调用了 closeAndCleanupJobH
                    break;
、
                default:
                    Slog.e(TAG, "Got an execution complete message for a job that wasn't being" +
                            "executed. Was " + VERB_STRINGS[mVerb] + ".");
            }
        }
```
接着进入 closeAndCleanupJobH 方法，参数 reschedule 表示是否再次执行这个 job！
```java
        private void closeAndCleanupJobH(boolean reschedule) {
            final JobStatus completedJob;
            synchronized (mLock) {
                if (mVerb == VERB_FINISHED) {
                    return;
                }
                completedJob = mRunningJob;
                mJobPackageTracker.noteInactive(completedJob);
                try {
                    mBatteryStats.noteJobFinish(mRunningJob.getBatteryName(),
                            mRunningJob.getSourceUid());
                } catch (RemoteException e) {
                    // Whatever.
                }
                if (mWakeLock != null) {
                    mWakeLock.release();
                }

                mContext.unbindService(JobServiceContext.this); // 取消绑定 JobService
                mWakeLock = null;
                mRunningJob = null;
                mParams = null;
                mVerb = VERB_FINISHED; // mVerb 状态置为 VERB_FINISHED；
                mCancelled.set(false);
                service = null;
                mAvailable = true;
            }

            removeOpTimeOut(); // 移除已经处理的消息
            removeMessages(MSG_CALLBACK);
            removeMessages(MSG_SERVICE_BOUND);
            removeMessages(MSG_CANCEL);
            removeMessages(MSG_SHUTDOWN_EXECUTION);

            // 调用了 mCompletedListener 的 onJobCompleted 方法!
            mCompletedListener.onJobCompleted(completedJob, reschedule);
        }
    }
```
这个方法很简单，就是对当前的 JobSchedulerContext 进行初始化操作，为执行下一个 job 做准备！

这里 mCompletedListener 大家去看 JobServiceContext 的初始化，也就是第二篇，其实就是 JobSchedulerService：
##### JobSchedulerService.onJobCompleted
```java
    @Override
    public void onJobCompleted(JobStatus jobStatus, boolean needsReschedule) {
        if (DEBUG) {
            Slog.d(TAG, "Completed " + jobStatus + ", reschedule=" + needsReschedule);
        }

        // 停止 track 这个 job，如果是周期性 job 不更新本地的 jobs.xml 文件!
        if (!stopTrackingJob(jobStatus, null, !jobStatus.getJob().isPeriodic())) {
            if (DEBUG) {
                Slog.d(TAG, "Could not find job to remove. Was job removed while executing?");
            }
            // We still want to check for jobs to execute, because this job may have
            // scheduled a new job under the same job id, and now we can run it.
            mHandler.obtainMessage(MSG_CHECK_JOB_GREEDY).sendToTarget();
            return;
        }
        
        // 重新 schedule 这个 job，注意这时候
        if (needsReschedule) {
            JobStatus rescheduled = getRescheduleJobForFailure(jobStatus);
            startTrackingJob(rescheduled, jobStatus);

        } else if (jobStatus.getJob().isPeriodic()) {
            JobStatus rescheduledPeriodic = getRescheduleJobForPeriodic(jobStatus);
            startTrackingJob(rescheduledPeriodic, jobStatus);

        }
        reportActive();

        // 发送 MSG_CHECK_JOB_GREEDY 给 JobSchedulerService.JobHandler
        mHandler.obtainMessage(MSG_CHECK_JOB_GREEDY).sendToTarget();
    }
```

这里首先调用了 stopTrackingJob 将这个 job 从 JobStore 和 controller 中移除：
```java
    private boolean stopTrackingJob(JobStatus jobStatus, JobStatus incomingJob,
            boolean writeBack) {
        synchronized (mLock) {

            // 从 JobStore 中移除这个 job，如果 writeback 为 true，还要更新本地的 job.xml 文件!
            final boolean removed = mJobs.remove(jobStatus, writeBack);

            if (removed && mReadyToRock) {
                for (int i=0; i<mControllers.size(); i++) {
                    StateController controller = mControllers.get(i);

                    // 从 Controller 的跟踪队列中移除！
                    controller.maybeStopTrackingJobLocked(jobStatus, incomingJob, false);
                }
            }
            return removed;
        }
    }
```
最后是发 MSG_CHECK_JOB_GREEDY 给 JobHandler：
##### JobHandler.MSG_CHECK_JOB_GREEDY
```java
    private class JobHandler extends Handler {
        @Override
        public void handleMessage(Message message) {
            synchronized (mLock) {
                if (!mReadyToRock) {
                    return;
                }
            }
            switch (message.what) {
                ... ... ... ...
                case MSG_CHECK_JOB_GREEDY:
                    synchronized (mLock) {

                        // 作用和 maybeQueueReadyJobsForExecutionLockedH 一样的都是更新 mPendingJobs 集合！
                        queueReadyJobsForExecutionLockedH();
                    }
                    break;
                ... ... ... ...
            }

            // 再次执行 mPendingJobs 中的 job
            maybeRunPendingJobsH();
            // Don't remove JOB_EXPIRED in case one came along while processing the queue.
            removeMessages(MSG_CHECK_JOB);
        }
        ... ... ... ...
    }
```
最后继续 maybeRunPendingJobsH，这里又回到了上面的第 6 节了，就不多说了！
#### 6.3.1.4 JSC.prepareForExecutionLocked
如果不需要优先级取代，取消优先级低的 job，那就直接执行当前的新 job，在执行那个前，调用 prepareForExecutionLocked 做准备工作。

这个方法其实很简单，就是一个抽象类的方法：
```java
    /**
     * Optionally implement logic here to prepare the job to be executed.
     */
    public void prepareForExecutionLocked(JobStatus jobStatus) {
    }
```
表示通知 StateController，做好准备，具体实现是在 Controller 中，我们先看看 ConnectivityController，其他类似：

#### 6.3.1.5 JSC.executeRunnableJob

这里就是调用 JobSchedulerContext 方法来执行 Job：
```java
    boolean executeRunnableJob(JobStatus job) {
        synchronized (mLock) {
            if (!mAvailable) {
                Slog.e(TAG, "Starting new runnable but context is unavailable > Error.");
                return false;
            }
            
            mPreferredUid = NO_PREFERRED_UID;
            // 保存到 mRunningJob 中
            mRunningJob = job;

            final boolean isDeadlineExpired =
                    job.hasDeadlineConstraint() &&
                            (job.getLatestRunTimeElapsed() < SystemClock.elapsedRealtime());
                            
            Uri[] triggeredUris = null;
            if (job.changedUris != null) {
                triggeredUris = new Uri[job.changedUris.size()];
                job.changedUris.toArray(triggeredUris);
            }
            
            String[] triggeredAuthorities = null;
            if (job.changedAuthorities != null) {
                triggeredAuthorities = new String[job.changedAuthorities.size()];
                job.changedAuthorities.toArray(triggeredAuthorities);
            }
            
            // 创建 job 需要的 JobParamters
            mParams = new JobParameters(this, job.getJobId(), job.getExtras(), isDeadlineExpired,
                    triggeredUris, triggeredAuthorities);
                    
            mExecutionStartTimeElapsed = SystemClock.elapsedRealtime();
            
            // mVerb 的值变为 VERB_BINDING!
            mVerb = VERB_BINDING;

            scheduleOpTimeOut();

            // 这里很关键，bind 应用程序中注册的 JobService，并返回结果！
            final Intent intent = new Intent().setComponent(job.getServiceComponent());
            boolean binding = mContext.bindServiceAsUser(intent, this,
                    Context.BIND_AUTO_CREATE | Context.BIND_NOT_FOREGROUND,
                    new UserHandle(job.getUserId()));
 
            if (!binding) { // 如果 bind 失败，异常处理！
                if (DEBUG) {
                    Slog.d(TAG, job.getServiceComponent().getShortClassName() + " unavailable.");
                }
                mRunningJob = null;
                mParams = null;
                mExecutionStartTimeElapsed = 0L;
                mVerb = VERB_FINISHED;
                removeOpTimeOut();
                return false;
            }
            // 记录信息!
            try {
                mBatteryStats.noteJobStart(job.getBatteryName(), job.getSourceUid());
            } catch (RemoteException e) {
                // Whatever.
            }
            mJobPackageTracker.noteActive(job);
            mAvailable = false;
            return true;
        }
    }
```
这便是由 system_server 进程的主线程来执行 bind Service 的方式来拉起的进程，当服务启动后回调到发起端的 onServiceConnected。

##### 6.3.1.5.1 JSC.onServiceConnected

bind 成功后，JobServiceContext 的 onServiceConnected 方法会执行：
```java
    @Override
    public void onServiceConnected(ComponentName name, IBinder service) {
        JobStatus runningJob;
        synchronized (mLock) {
            runningJob = mRunningJob;
        }

        // 异常检测，如果 runningJob 为 null 或者 JobService 组件不匹配，发送 MSG_SHUTDOWN_EXECUTION 终止执行！
        if (runningJob == null || !name.equals(runningJob.getServiceComponent())) {
            mCallbackHandler.obtainMessage(MSG_SHUTDOWN_EXECUTION).sendToTarget();
            return;
        }

        // 获得了应用程序的 JobService 的代理对象！
        this.service = IJobService.Stub.asInterface(service);
        // 申请 wakeLock 锁！
        final PowerManager pm =
                (PowerManager) mContext.getSystemService(Context.POWER_SERVICE);
        PowerManager.WakeLock wl = pm.newWakeLock(PowerManager.PARTIAL_WAKE_LOCK,
                runningJob.getTag());
        wl.setWorkSource(new WorkSource(runningJob.getSourceUid()));
        wl.setReferenceCounted(false);
        wl.acquire();

        synchronized (mLock) {
    
            if (mWakeLock != null) {
                Slog.w(TAG, "Bound new job " + runningJob + " but live wakelock " + mWakeLock
                        + " tag=" + mWakeLock.getTag());
                mWakeLock.release();
            }
            mWakeLock = wl;
        }

        // 发送 MSG_SERVICE_BOUND 给 JobServiceHandler 中！
        mCallbackHandler.obtainMessage(MSG_SERVICE_BOUND).sendToTarget();
    }
```
可以看到，这里会发送消息到 JobServiceHandler 中：
##### 6.3.1.5.2 JSC.JobServiceHandler

JobServiceHandler 方法也是在主线程中！
```java
    private class JobServiceHandler extends Handler {
        JobServiceHandler(Looper looper) {
            super(looper);
        }
        @Override
        public void handleMessage(Message message) {
            switch (message.what) {
                case MSG_SERVICE_BOUND: // 绑定成功！
                    removeOpTimeOut();
                    handleServiceBoundH();
                    break;

                ... ... ... ...

                case MSG_SHUTDOWN_EXECUTION: // 绑定是出现异常的消息！
                    closeAndCleanupJobH(true /* needsReschedule */);
                    break;
                default:
                    Slog.e(TAG, "Unrecognised message: " + message);
            }
        }
     
        ... ... ... ...
    
    }
```
接着我们来分别看一下：
###### 6.3.1.5.2.1 JSS.handleServiceBoundH

收到这个消息后，调用 handleServiceBoundH 方法：
```java
        /** Start the job on the service. */
        private void handleServiceBoundH() {
            if (DEBUG) {
                Slog.d(TAG, "MSG_SERVICE_BOUND for " + mRunningJob.toShortString());
            }
            if (mVerb != VERB_BINDING) { // 状态异常
                Slog.e(TAG, "Sending onStartJob for a job that isn't pending. "
                        + VERB_STRINGS[mVerb]);
                closeAndCleanupJobH(false /* reschedule */);
                return;
            }

            // 如果 mCancelled.get() 为 true，表示 job 被取消了，那就执行舒适化 JSC，并 return！
            if (mCancelled.get()) { 
                if (DEBUG) {
                    Slog.d(TAG, "Job cancelled while waiting for bind to complete. "
                            + mRunningJob);
                }
                closeAndCleanupJobH(true /* reschedule */);
                return;
            }

            try {

                // mVerb 的值设置为 VERB_STARTING！
                mVerb = VERB_STARTING;
                scheduleOpTimeOut();

                // 这里调用了 JobService 的 startJob 方法
                service.startJob(mParams);
            } catch (RemoteException e) {
                Slog.e(TAG, "Error sending onStart message to '" +
                        mRunningJob.getServiceComponent().getShortClassName() + "' ", e);
            }
        }
```
这里就不细看了！

# 7 总结

## 7.1 schedule 流程
终于跟完了代码。我们来总结一下，整个流程：

- 根据传入的 JobInfo，创建 JobStatus；

- 如果已经注册过一个相同 uid 和 jobId 的 job，就 cancel 之前注册过的 job；
  - 停止监控这个 job：
     - 从 JobStore 和 controller 的监控队列中移除这个 job，并停止 controller 对这个 job 的监控；
  - 如果这个任务在 mPendingJobs 队列中，移除它；
  - 如果正在运行，取消这个任务；
  - 更新 JSS 的状态！

- 将新 job 加入到 JobStore 中，并通知 controller 开始监控新 schedule 的 job！
  - 如果 JobStore 中已经存在相同 uid 和 jobId 的 job，就取代这个 job ，并停止对这个旧 job 的监控！

- 发送 MSG_CHECK_JOB 消息给 JobHandler，进行执行 job 前的检查！
  - 清空 mPendingJobs，为本次执行做准备，将 JobStore 中所有准备好的 job，加入到 runnableJobs 可执行列表中，再将 runnableJobs 列表中的所有 job 添加到 mPendingJobs 集合中！
  - 给 mPendingJobs 集合中的 job 分配 JobSchedulerContext！


## 7.2 JSC 和 JS 的关系
先空着，后续补上！


## 7.3 JSC 的 Job 分配算法
这里我们来总结一些 job 的分配算法：