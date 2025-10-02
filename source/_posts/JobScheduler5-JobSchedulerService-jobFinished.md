# JobScheduler第 5 篇 - JobSchedulerService - jobFinished
title: JobScheduler第 5 篇 - JobSchedulerService - jobFinished
date: 2016/04/23 20:46:25 
categories: 
- AndroidFramework源码分析
- JobScheduler任务调度
tags: JobScheduler任务调度
---

基于 Android 7.1.1 源码分析
# 前言
当任务完成时，应用需要手动调用 jobFinished 方法，这个方法是属于 JobService 的：
```java
    public final void jobFinished(JobParameters params, boolean needsReschedule) {
        ensureHandler();
        Message m = Message.obtain(mHandler, MSG_JOB_FINISHED, params);
        m.arg2 = needsReschedule ? 1 : 0;
        m.sendToTarget();
    }
```
其实，参数很简单，这个消息 MSG_JOB_FINISHED 会发送到 JobHandler 中！
# 1 JS.JobHandler
进入 JobHandler 中来看看：
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
          
                case MSG_JOB_FINISHED:
                    final boolean needsReschedule = (msg.arg2 == 1);
                    // callback 是 JobServiceContext 
                    IJobCallback callback = params.getCallback();
                    if (callback != null) {
                        try {
                            callback.jobFinished(params.getJobId(), needsReschedule);
                        } catch (RemoteException e) {
                            Log.e(TAG, "Error reporting job finish to system: binder has gone" + " away.");
                        }
                    } else {
                        Log.e(TAG, "finishJob() called for a nonexistent job id.");
                    }
                    break;

                default:
                    Log.e(TAG, "Unrecognised message received.");
                    break;
            }
        }
    }
```
这里调用了 JObServiceContext 的 jobFinished 方法！
# 2 JSC.jobFinished
我们来看看 JobServiceContext 的 jobFinished 方法：
```java
    @Override
    public void jobFinished(int jobId, boolean reschedule) {
        if (!verifyCallingUid()) {
            return;
        }

        mCallbackHandler.obtainMessage(MSG_CALLBACK, jobId, reschedule ? 1 : 0)
                .sendToTarget();
    }
```
这里先调用了 JobServiceContext 的 verifyCallingUid，校验 uid！
```java
    private boolean verifyCallingUid() {
        synchronized (mLock) {
            if (mRunningJob == null || Binder.getCallingUid() != mRunningJob.getUid()) {
                if (DEBUG) {
                    Slog.d(TAG, "Stale callback received, ignoring.");
                }
                return false;
            }
            return true;
        }
    }
```
接着发送了一个 MSG_CALLBACK 的消息给了 JobServiceHandler！
# 3 JSC.JobServiceHandler
接着进入了 JobServiceHandler，来看主要代码：
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
                    } else if (mVerb == VERB_EXECUTING || // 此时 mVerb 的状态为 VERB_EXECUTING
                            mVerb == VERB_STOPPING) { 
                        final boolean reschedule = message.arg2 == 1;
                        handleFinishedH(reschedule);
                    } else {
                        if (DEBUG) {
                            Slog.d(TAG, "Unrecognised callback: " + mRunningJob);
                        }
                    }
                    break;
```
调用了 handleFinishedH：
```java
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
```
这里就很简单了，调用了 closeAndCleanupJobH 来解除 bind 和将 JObServiceContext 恢复初始化，为下一次 bind 做准备！
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
                mContext.unbindService(JobServiceContext.this);
                mWakeLock = null;
                mRunningJob = null;
                mParams = null;
                mVerb = VERB_FINISHED; // 状态变为 VERB_FINISHED
                mCancelled.set(false);
                service = null;
                mAvailable = true;
            }

            removeOpTimeOut();
            removeMessages(MSG_CALLBACK);
            removeMessages(MSG_SERVICE_BOUND);
            removeMessages(MSG_CANCEL);
            removeMessages(MSG_SHUTDOWN_EXECUTION);

            // 通知 JobSchedulerService，这个 job 已经 finished 了
            mCompletedListener.onJobCompleted(completedJob, reschedule);
        }
    }
```
接着，进入了 JobSchedulerService 方法中：
# 4 JSS.onJobCompleted
调用了 onJobCompleted 方法；
```java
    @Override
    public void onJobCompleted(JobStatus jobStatus, boolean needsReschedule) {
        if (DEBUG) {
            Slog.d(TAG, "Completed " + jobStatus + ", reschedule=" + needsReschedule);
        }

        // 停止 track job，会将 job 从 JobStore 中删除，如果不是周期性的 job，还要更新本地的 jobs.xml 文件！
        if (!stopTrackingJob(jobStatus, null, !jobStatus.getJob().isPeriodic())) {
            if (DEBUG) {
                Slog.d(TAG, "Could not find job to remove. Was job removed while executing?");
            }
            
            // 停止失败，说明 job 之前已经被移除，继续发起下一轮的执行！
            mHandler.obtainMessage(MSG_CHECK_JOB_GREEDY).sendToTarget();
            return;
        }
        
        // 停止成功
        if (needsReschedule) {// 需要 reschedule 这个 job，如果需要，就 startTrackingJob
            JobStatus rescheduled = getRescheduleJobForFailure(jobStatus);
            startTrackingJob(rescheduled, jobStatus);
            
        } else if (jobStatus.getJob().isPeriodic()) { // 如果不需要 reschedule，但是这个 job 是周期性的，也要 startTrackingJob
            JobStatus rescheduledPeriodic = getRescheduleJobForPeriodic(jobStatus);
            startTrackingJob(rescheduledPeriodic, jobStatus);

        }
        
        // 汇报 JSS 运行状态给 DeviceIdleController！
        reportActive();
        
        // 继续开始新一轮的 job 执行！
        mHandler.obtainMessage(MSG_CHECK_JOB_GREEDY).sendToTarget();
    }
```


## 4.1 JSS.getRescheduleJobForFailure
这里调用了 getRescheduleJobForPeriodic 的方法：
```java
    private JobStatus getRescheduleJobForFailure(JobStatus failureToReschedule) {
        // 获得自开机后，经过的时间，包括深度睡眠的时间;
        final long elapsedNowMillis = SystemClock.elapsedRealtime();
        // 获得这个 jobFinished 的 jobStatus 对应的 JobInfo;
        final JobInfo job = failureToReschedule.getJob();

        final long initialBackoffMillis = job.getInitialBackoffMillis();
        final int backoffAttempts = failureToReschedule.getNumFailures() + 1;
        long delayMillis;

        switch (job.getBackoffPolicy()) {
            case JobInfo.BACKOFF_POLICY_LINEAR:
                delayMillis = initialBackoffMillis * backoffAttempts;
                break;

            case JobInfo.BACKOFF_POLICY_EXPONENTIAL:
                delayMillis =
                        (long) Math.scalb(initialBackoffMillis, backoffAttempts - 1);
                break;

            default:
                if (DEBUG) {
                    Slog.v(TAG, "Unrecognised back-off policy, defaulting to exponential.");
                }
        }
        delayMillis =
                Math.min(delayMillis, JobInfo.MAX_BACKOFF_DELAY_MILLIS);

        JobStatus newJob = new JobStatus(failureToReschedule, elapsedNowMillis + delayMillis,
                JobStatus.NO_LATEST_RUNTIME, backoffAttempts);

        for (int ic=0; ic<mControllers.size(); ic++) {
            StateController controller = mControllers.get(ic);
            controller.rescheduleForFailure(newJob, failureToReschedule);
        }

        return newJob;
    }
```


## 4.2 JSS.getRescheduleJobForPeriodic
```java
    private JobStatus getRescheduleJobForPeriodic(JobStatus periodicToReschedule) {
        final long elapsedNow = SystemClock.elapsedRealtime();
        // Compute how much of the period is remaining.
        long runEarly = 0L;

        // If this periodic was rescheduled it won't have a deadline.
        if (periodicToReschedule.hasDeadlineConstraint()) {
            runEarly = Math.max(periodicToReschedule.getLatestRunTimeElapsed() - elapsedNow, 0L);
        }
        long flex = periodicToReschedule.getJob().getFlexMillis();
        long period = periodicToReschedule.getJob().getIntervalMillis();
        long newLatestRuntimeElapsed = elapsedNow + runEarly + period;
        long newEarliestRunTimeElapsed = newLatestRuntimeElapsed - flex;

        if (DEBUG) {
            Slog.v(TAG, "Rescheduling executed periodic. New execution window [" +
                    newEarliestRunTimeElapsed/1000 + ", " + newLatestRuntimeElapsed/1000 + "]s");
        }
        return new JobStatus(periodicToReschedule, newEarliestRunTimeElapsed,
                newLatestRuntimeElapsed, 0 /* backoffAttempt */);
    }
```

































