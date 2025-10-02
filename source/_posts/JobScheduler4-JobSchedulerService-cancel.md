# JobScheduler第 4 篇 - JobSchedulerService - cancel
title: JobScheduler第 4 篇 - JobSchedulerService - cancel
date: 2016/04/21 20:46:25
categories:
- AndroidFramework源码分析
- JobScheduler任务调度
tags: JobScheduler任务调度
---

基于 Android 7.1.1 源码分析
# 前提
接下来，我们来看看 JobServiceService 服务中和 cancel 相关的服务：
```java
    // 取消指定设备用户的所有的 job！
    void cancelJobsForUser(int userHandle) {
        List<JobStatus> jobsForUser;
        synchronized (mLock) {
            jobsForUser = mJobs.getJobsByUser(userHandle);
        }
        for (int i=0; i<jobsForUser.size(); i++) {
            JobStatus toRemove = jobsForUser.get(i);
            cancelJobImpl(toRemove, null);
        }
    }
    // 取消指定 package 和指定 uid 的所有的 job！
    void cancelJobsForPackageAndUid(String pkgName, int uid) {
        List<JobStatus> jobsForUid;
        synchronized (mLock) {
            jobsForUid = mJobs.getJobsByUid(uid);
        }
        for (int i = jobsForUid.size() - 1; i >= 0; i--) {
            final JobStatus job = jobsForUid.get(i);
            if (job.getSourcePackageName().equals(pkgName)) {
                cancelJobImpl(job, null);
            }
        }
    }
    
    // 取消指定 uid 的应用程序的所有的 job！
    public void cancelJobsForUid(int uid, boolean forceAll) {
        List<JobStatus> jobsForUid;
        synchronized (mLock) {
            jobsForUid = mJobs.getJobsByUid(uid);
        }
        for (int i=0; i<jobsForUid.size(); i++) {
            JobStatus toRemove = jobsForUid.get(i);
            if (!forceAll) {
                String packageName = toRemove.getServiceComponent().getPackageName();
                try {
                    if (ActivityManagerNative.getDefault().getAppStartMode(uid, packageName)
                            != ActivityManager.APP_START_MODE_DISABLED) {
                        continue;
                    }
                } catch (RemoteException e) {
                }
            }
            cancelJobImpl(toRemove, null);
        }
    }
    // 取消指定 uid 的应用程序的 id 为 jobId 的 job！
    public void cancelJob(int uid, int jobId) {
        JobStatus toCancel;
        synchronized (mLock) {
            toCancel = mJobs.getJobByUidAndJobId(uid, jobId);
        }
        if (toCancel != null) {
            cancelJobImpl(toCancel, null);
        }
    }
```
JobSchedulerService 提供了很多个接口来取消 job，但是他们都疯了似的调用了同一个方法，我们来继续看！
# 1 JSS.cancelJobImpl
同一个取消接口：
```java
    private void cancelJobImpl(JobStatus cancelled, JobStatus incomingJob) {
        if (DEBUG) Slog.d(TAG, "CANCEL: " + cancelled.toShortString());
        // 停止监视这个 job，writeBack 为 true!
        stopTrackingJob(cancelled, incomingJob, true /* writeBack */);
        synchronized (mLock) {
            // Remove from pending queue.
            // 从 mPendingJobs 中移除！
            if (mPendingJobs.remove(cancelled)) {
                mJobPackageTracker.noteNonpending(cancelled);
            }
            // Cancel if running.
            // 如果正在运行，就要停止这个 job
            stopJobOnServiceContextLocked(cancelled, JobParameters.REASON_CANCELED);
            reportActive();
        }
    }
```
这里我们一个一个来看：
## 1.1 JSS.stopTrackingJob

停止 stopTrackingJob 如下：
```java
    private boolean stopTrackingJob(JobStatus jobStatus, JobStatus incomingJob,
            boolean writeBack) {
        synchronized (mLock) {
            // Remove from store as well as controllers.
            // 先从 JobStore 中移除这个 job，因为 writeBack 为 true，则需要更新 jobxs.xml 文件！
            final boolean removed = mJobs.remove(jobStatus, writeBack);
            if (removed && mReadyToRock) {
                for (int i=0; i<mControllers.size(); i++) {
                    // 通知控制器，取消 track！
                    StateController controller = mControllers.get(i);
                    controller.maybeStopTrackingJobLocked(jobStatus, incomingJob, false);
                }
            }
            return removed;
        }
    }
```
先从 JobStore 中移除这个 job，因为 writeBack 为 true，则需要更新 jobxs.xml 文件，通知控制器，取消 track！
## 1.2 JSS.stopJobOnServiceContextLocked
如果这个 job 是在运行中，就要取消它！
```java
    private boolean stopJobOnServiceContextLocked(JobStatus job, int reason) {
        for (int i=0; i<mActiveServices.size(); i++) {
            JobServiceContext jsc = mActiveServices.get(i);
            final JobStatus executing = jsc.getRunningJob();
            if (executing != null && executing.matches(job.getUid(), job.getJobId())) {
                jsc.cancelExecutingJob(reason);
                return true;
            }
        }
        return false;
    }
```
每一个运行着的 job，都是和一个 JobServiceContext 绑定着的，这里是遍历所有的 JobServiceContext，找到要 cancel 的 job，调用 jobServiceContext 的 cancelExecutingJob 方法：
### 1.2.1 JSC.cancelExecutingJob

这个方法很简单，发送了 MSG_CANCEL 消息给 JobServiceHandler，reason 为 JobParameters.REASON_CANCELED
```java
    void cancelExecutingJob(int reason) {
        mCallbackHandler.obtainMessage(MSG_CANCEL, reason, 0 /* unused */).sendToTarget();
    }
```
我们继续看，进入 JobServiceHandler：
### 1.2.2 JSH.MSG_CANCEL

JobServiceHander 会处理 MSG_CANCEL 消息！
```java
    private class JobServiceHandler extends Handler {
        JobServiceHandler(Looper looper) {
            super(looper);
        }
        @Override
        public void handleMessage(Message message) {
                
                ... ... ... ...

                case MSG_CANCEL: // 接受到了 MSG_CANCEL.
                    if (mVerb == VERB_FINISHED) { 
                        if (DEBUG) {
                            Slog.d(TAG,
                                   "Trying to process cancel for torn-down context, ignoring.");
                        }
                        return;
                    }
                    mParams.setStopReason(message.arg1); // reason 为 JobParameters.REASON_CANCELED
                    if (message.arg1 == JobParameters.REASON_PREEMPT) { // 不进入，因为这不是优先级取代！
                        mPreferredUid = mRunningJob != null ? mRunningJob.getUid() :
                                NO_PREFERRED_UID;
                    }
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

        ... ... ... ... ...

    }
```
调用 handleCancelH 方法来取消 Job：
### 1.2.3 JSH.handleCancelH
```java
       /**
         * A job can be in various states when a cancel request comes in:
         *   VERB_BINDING   -> Cancelled before bind completed. Mark as cancelled and wait for
         *                    {@link #onServiceConnected(android.content.ComponentName, android.os.IBinder)}
         *      _STARTING   -> Mark as cancelled and wait for
         *                    {@link JobServiceContext#acknowledgeStartMessage(int, boolean)}
         *     _EXECUTING  -> call {@link #sendStopMessageH}}, but only if there are no callbacks
         *                      in the message queue.
         *     _ENDING     -> No point in doing anything here, so we ignore.
         */
        private void handleCancelH() {
            if (JobSchedulerService.DEBUG) {
                Slog.d(TAG, "Handling cancel for: " + mRunningJob.getJobId() + " "
                        + VERB_STRINGS[mVerb]);
            }
            switch (mVerb) {
                case VERB_BINDING:  // 如果 job 的状态是 VERB_BINDING 或者 VERB_STARTING，直接将 mCancelled 置为 true !
                case VERB_STARTING:
                    mCancelled.set(true);
                    break;
                case VERB_EXECUTING: // 如果 job 的状态是 VERB_EXECUTING，
                    if (hasMessages(MSG_CALLBACK)) { // 判断客户端是否已经调用过 jobFinished 方法，有直接 return!
                        // If the client has called jobFinished, ignore this cancel.
                        return;
                    }
                    sendStopMessageH(); // 
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
我们来看看 sendStopMessageH 方法：
### 1.2.4 JSH.sendStopMessageH
```java
        /**
         * Already running, need to stop. Will switch {@link #mVerb} from VERB_EXECUTING ->
         * VERB_STOPPING.
         */
        private void sendStopMessageH() {
            removeOpTimeOut();
            if (mVerb != VERB_EXECUTING) { // 如果 job 的状态已经不是 VERB_EXECUTING，那就清除资源，恢复初始化!
                Slog.e(TAG, "Sending onStopJob for a job that isn't started. " + mRunningJob);
                closeAndCleanupJobH(false /* reschedule */);
                return;
            }
            try {
                mVerb = VERB_STOPPING; // 否则，job 状态设置为 VERB_STOPPING，
                scheduleOpTimeOut(); 
                service.stopJob(mParams); // 调用 jobService 的 stopJob，停止服务!
            } catch (RemoteException e) {
                Slog.e(TAG, "Error sending onStopJob to client.", e);
                closeAndCleanupJobH(false /* reschedule */);
            }
        }
```
这里会根据 mVerb 中存储的 job 的动作状态，来做相应的处理：
如果 job 的状态已经不是 VERB_EXECUTING，那就清除资源，恢复初始化；
否则，job 状态设置为 VERB_STOPPING，调用 jobService 的 stopJob，停止服务!
#### 1.2.4.1 JobService.stopJob

通过 Binder 机制，停止 job：
```java
public abstract class JobService extends Service {
    
    ... ... ... ...    

    private static final String TAG = "JobService";
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
            JobService service = mService.get();
            if (service != null) {
                service.ensureHandler(); // 确保 Handler 已经创建
                Message m = Message.obtain(service.mHandler, MSG_STOP_JOB, jobParams); // 发送 MSG_STOP_JOB 到 JobHandler！
                m.sendToTarget();
            }
        }
    }

    ... ... ... ...
}
```
进入 JobHandler 方法：
#### 1.2.4.2 JobService.JobHandler

处理前面通过 Binder 发送过来的 MSG_STOP_JOB：
```java
    class JobHandler extends Handler {
        JobHandler(Looper looper) {
            super(looper);
        }
        @Override
        public void handleMessage(Message msg) {
            final JobParameters params = (JobParameters) msg.obj;
            switch (msg.what) {
                ... ... ... ...
                case MSG_STOP_JOB:
                    try {
                        // 这里是获得 JobService 的 onStopJob 的返回值!
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

        ... ... ... ...    
 
        private void ackStopMessage(JobParameters params, boolean reschedule) {
            final IJobCallback callback = params.getCallback();
            final int jobId = params.getJobId();
            if (callback != null) {
                try {
                    // 将被停止的 job 的 id 和 onStopJob 的返回值发给 JobServiceContext！
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
JobService 的 onStopJob 的返回值如果为 true，那么，就表示 JobService 需要重新拉起他！前面我们分析的时候，知道，callback 就是 JobServiceContext
#### 1.2.4.3 JSC.acknowledgeStopMessage

接着发送 MSG_CALLBACK 给 JobServiceHandler：
```java
    @Override
    public void acknowledgeStopMessage(int jobId, boolean reschedule) {
        if (!verifyCallingUid()) {
            return;
        }
        mCallbackHandler.obtainMessage(MSG_CALLBACK, jobId, reschedule ? 1 : 0)
                .sendToTarget();
    }
```
我们进入 JobServiceHandler 方法里面看看：
#### 1.2.4.4 JSC.JobServiceHandler
```java
    private class JobServiceHandler extends Handler {
        @Override
        public void handleMessage(Message message) {
            switch (message.what) {
                ... ... ... ...
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
                            mVerb == VERB_STOPPING) { // 进入这里！
                        final boolean reschedule = message.arg2 == 1;
                        handleFinishedH(reschedule);
                    } else {
                        if (DEBUG) {
                            Slog.d(TAG, "Unrecognised callback: " + mRunningJob);
                        }
                    }
                    break;
                ... ... ... ...
                default:
                    Slog.e(TAG, "Unrecognised message: " + message);
            }
        }
        
        ... ... ... ...
        
        /**
         * VERB_EXECUTING  -> Client called jobFinished(), clean up and notify done.
         *     _STOPPING   -> Successful finish, clean up and notify done.
         *     _STARTING   -> Error
         *     _PENDING    -> Error
         */
        private void handleFinishedH(boolean reschedule) {
            switch (mVerb) {
                case VERB_EXECUTING:
                case VERB_STOPPING:
                    closeAndCleanupJobH(reschedule);
                    break;
                default:
                    Slog.e(TAG, "Got an execution complete message for a job that wasn't being" +
                            "executed. Was " + VERB_STRINGS[mVerb] + ".");
            }
        }
        
        ... ... ... ...
    }
```
可以看到，最后会调用 JSH 的 closeAndCleanupJobH 方法：
#### 1.2.4.5 JSH.closeAndCleanupJobH

如果 job 的状态已经不是 VERB_EXECUTING，那就清除资源，恢复初始化；这个方法很简单，就是将 JobServiceContext 中的属性值恢复初始化，表示没有任何 job 在运行！
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
                mContext.unbindService(JobServiceContext.this); // 取消绑定 jobSerivce
                mWakeLock = null;
                mRunningJob = null; // mRunningJob 的值置为 null!
                mParams = null; // JobParamters 置为 null
                mVerb = VERB_FINISHED; // mVerb 的状态值为 VERB_FINISHED
                mCancelled.set(false); // mCancelled 置为 false
                service = null; // 应用的 JobService 代理对象置为 null
                mAvailable = true; // mAvailable 置为 true，表示这个 JobService 可以分配给其他 job
            }
            removeOpTimeOut();
            removeMessages(MSG_CALLBACK); // 清除处理过的 MSG
            removeMessages(MSG_SERVICE_BOUND);
            removeMessages(MSG_CANCEL);
            removeMessages(MSG_SHUTDOWN_EXECUTION);
            // 回调 JobSchedulerService 的 onJobCompleted 方法!
            mCompletedListener.onJobCompleted(completedJob, reschedule);
        }
```
恢复初始化，为下次做准备!
##### 1.2.4.5.1 JSC.onServiceDisconnected

取消绑定 JobService 会调用 onServiceDisconnected
```java
    /** If the client service crashes we reschedule this job and clean up. */
    @Override
    public void onServiceDisconnected(ComponentName name) {
        mCallbackHandler.obtainMessage(MSG_SHUTDOWN_EXECUTION).sendToTarget();
    }
```
这里的会再回到 JobServiceHandler 中：
```java
                case MSG_SHUTDOWN_EXECUTION:
                    closeAndCleanupJobH(true /* needsReschedule */);
                    break;
```
做第二次 clean，防止第一次不成功，其实第一次 clean 成功的话，mVerb 的值为 VERB_FINISHED，第二次 clean 会自动退出的！
##### 1.2.4.5.2 JSS.onJobCompleted

我们进入 JSS 的 onJobCompleted 的方法中：
```java
    @Override
    public void onJobCompleted(JobStatus jobStatus, boolean needsReschedule) {
        if (DEBUG) {
            Slog.d(TAG, "Completed " + jobStatus + ", reschedule=" + needsReschedule);
        }
        // Do not write back immediately if this is a periodic job. The job may get lost if system
        // shuts down before it is added back.
        // 再次停止 track 这个 job，这里 stopTrackingJob 的返回值为 false！
        if (!stopTrackingJob(jobStatus, null, !jobStatus.getJob().isPeriodic())) {
            if (DEBUG) {
                Slog.d(TAG, "Could not find job to remove. Was job removed while executing?");
            }
            // We still want to check for jobs to execute, because this job may have
            // scheduled a new job under the same job id, and now we can run it.
            // 发送 MSG_CHECK_JOB_GREEDY，继续执行其他的 job，然后直接 return
            mHandler.obtainMessage(MSG_CHECK_JOB_GREEDY).sendToTarget();
            return;
        }
        // Note: there is a small window of time in here where, when rescheduling a job,
        // we will stop monitoring its content providers.  This should be fixed by stopping
        // the old job after scheduling the new one, but since we have no lock held here
        // that may cause ordering problems if the app removes jobStatus while in here.
        if (needsReschedule) {
            JobStatus rescheduled = getRescheduleJobForFailure(jobStatus);
            startTrackingJob(rescheduled, jobStatus);
        } else if (jobStatus.getJob().isPeriodic()) {
            JobStatus rescheduledPeriodic = getRescheduleJobForPeriodic(jobStatus);
            startTrackingJob(rescheduledPeriodic, jobStatus);
        }
        reportActive();
        mHandler.obtainMessage(MSG_CHECK_JOB_GREEDY).sendToTarget();
    }
```
这里首先调用了 stopTrackingJob 将这个 job 从 JobStore 和 controller 中移除，因为之前已经移除过了，所以这个 stopTrackingJob 的返回值为 false
```java
    /**
     * Called when we want to remove a JobStatus object that we've finished executing. Returns the
     * object removed.
     */
    private boolean stopTrackingJob(JobStatus jobStatus, JobStatus incomingJob,
            boolean writeBack) {
        synchronized (mLock) {
            // Remove from store as well as controllers.
            // 尝试从 JobStore 中移除这个 job，注意，在之前已经移除过了，所以，这里的 removed 为 false！
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
可以看出，对于用户主动 cancel 的任务，无论 onStopJob 的返回值是什么，都不会 reschedule 了！




