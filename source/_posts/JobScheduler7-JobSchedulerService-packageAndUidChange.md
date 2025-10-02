# JobScheduler第 7 篇 - JobSchedulerService - package and uid change
title: JobScheduler第 7 篇 - JobSchedulerService - package and uid change
date: 2016/04/27 20:46:25 
categories: 
- AndroidFramework源码分析
- JobScheduler任务调度
tags: JobScheduler任务调度
---

基于 Android 7.1.1 源码分析：
# 前言
我们想象这样的场景，如果有一个应用，它 schedule 了一些 job， 这些 job 可能正在运行，可能在 pending！这个时候，用户卸载了这个应用，那这个应用对应的 job 该何去何从？这里就要涉及到 package change 对 job 的影响了！

同时，进程优先级的变化，其实也会影响进程内的 job 的优先级！

我们回到 JSS 的启动和初始化：
```java
    @Override
    public void onBootPhase(int phase) {
        if (PHASE_SYSTEM_SERVICES_READY == phase) {
            mConstants.start(getContext().getContentResolver());
            
            // 注册广播，监听应用程序包的操作！
            final IntentFilter filter = new IntentFilter();
            filter.addAction(Intent.ACTION_PACKAGE_REMOVED);
            filter.addAction(Intent.ACTION_PACKAGE_CHANGED);
            filter.addAction(Intent.ACTION_PACKAGE_RESTARTED);
            filter.addAction(Intent.ACTION_QUERY_PACKAGE_RESTART);
            filter.addDataScheme("package");
            getContext().registerReceiverAsUser(
                    mBroadcastReceiver, UserHandle.ALL, filter, null, null);

            // 注册广播，监听设备用户的操作！
            final IntentFilter userFilter = new IntentFilter(Intent.ACTION_USER_REMOVED);
            getContext().registerReceiverAsUser(
                    mBroadcastReceiver, UserHandle.ALL, userFilter, null, null);

            mPowerManager = (PowerManager)getContext().getSystemService(Context.POWER_SERVICE);
            try {
            
                // 注册 uid 监控者
                ActivityManagerNative.getDefault().registerUidObserver(mUidObserver,
                        ActivityManager.UID_OBSERVER_PROCSTATE | ActivityManager.UID_OBSERVER_GONE
                        | ActivityManager.UID_OBSERVER_IDLE);
            } catch (RemoteException e) {
                // ignored; both services live in system_server
            }
        } else if (phase == PHASE_THIRD_PARTY_APPS_CAN_START) {
            ... ... ... ...
        }
    }
```
可以看到，这段代码，注册了两个广播监听者，下面我们一一来看！

这里有个小细节，如果想监听程序的安装和删除的广播，需要静态或者动态配置以下属性：
```xml
<data android:scheme="package">
</data>
```
或者：
```java
filter.addDataScheme("package");
```

# 1 Package Change
我们回到 JobSchedulerService 服务中去，在 JSS 中，有一个广播接收者，代码如下：
```java
    /**
     * Cleans up outstanding jobs when a package is removed. Even if it's being replaced later we
     * still clean up. On reinstall the package will have a new uid.
     */
    private final BroadcastReceiver mBroadcastReceiver = new BroadcastReceiver() {
        @Override
        public void onReceive(Context context, Intent intent) {
            final String action = intent.getAction();
            if (DEBUG) {
                Slog.d(TAG, "Receieved: " + action);
            }

            if (Intent.ACTION_PACKAGE_CHANGED.equals(action)) { // 应用程序包改变的广播！
                // Purge the app's jobs if the whole package was just disabled.  When this is
                // the case the component name will be a bare package name.
                final String pkgName = getPackageName(intent);
                final int pkgUid = intent.getIntExtra(Intent.EXTRA_UID, -1);

                if (pkgName != null && pkgUid != -1) {
                    // 获得改变的应用程序组件。
                    final String[] changedComponents = intent.getStringArrayExtra(
                            Intent.EXTRA_CHANGED_COMPONENT_NAME_LIST);
                            
                    if (changedComponents != null) {
                        for (String component : changedComponents) {
                            if (component.equals(pkgName)) {
                                if (DEBUG) {
                                    Slog.d(TAG, "Package state change: " + pkgName);
                                }
                                try {
                                    final int userId = UserHandle.getUserId(pkgUid);
                                    IPackageManager pm = AppGlobals.getPackageManager();

                                    // 获得这个 package 的状态！
                                    final int state = pm.getApplicationEnabledSetting(pkgName, userId);
                                    if (state == COMPONENT_ENABLED_STATE_DISABLED
                                            || state ==  COMPONENT_ENABLED_STATE_DISABLED_USER) {
                                        if (DEBUG) {
                                            Slog.d(TAG, "Removing jobs for package " + pkgName
                                                    + " in user " + userId);
                                        }

                                        // 第二个参数表示，强制取消！
                                        cancelJobsForUid(pkgUid, true);
                                    }
                                } catch (RemoteException|IllegalArgumentException e) {
                                    ... ... ... ... ...
                                }
                                break;
                            }
                        }
                    }
                } else {
                    Slog.w(TAG, "PACKAGE_CHANGED for " + pkgName + " / uid " + pkgUid);
                }

            } else if (Intent.ACTION_PACKAGE_REMOVED.equals(action)) { // 应用程序被移除的广播！
                // If this is an outright uninstall rather than the first half of an
                // app update sequence, cancel the jobs associated with the app.
                if (!intent.getBooleanExtra(Intent.EXTRA_REPLACING, false)) {
                    int uidRemoved = intent.getIntExtra(Intent.EXTRA_UID, -1);
                    if (DEBUG) {
                        Slog.d(TAG, "Removing jobs for uid: " + uidRemoved);
                    }
                    
                    cancelJobsForUid(uidRemoved, true);
                }

            } else if (Intent.ACTION_USER_REMOVED.equals(action)) { // 设备用户被移除的广播！
                final int userId = intent.getIntExtra(Intent.EXTRA_USER_HANDLE, 0);
                if (DEBUG) {
                    Slog.d(TAG, "Removing jobs for user: " + userId);
                }

                cancelJobsForUser(userId);

            } else if (Intent.ACTION_QUERY_PACKAGE_RESTART.equals(action)) { // package 被启用的广播！
                // Has this package scheduled any jobs, such that we will take action
                // if it were to be force-stopped?
                final int pkgUid = intent.getIntExtra(Intent.EXTRA_UID, -1);
                final String pkgName = intent.getData().getSchemeSpecificPart();
                if (pkgUid != -1) {
                    List<JobStatus> jobsForUid;
                    synchronized (mLock) {
                        jobsForUid = mJobs.getJobsByUid(pkgUid);
                    }
                    for (int i = jobsForUid.size() - 1; i >= 0; i--) {
                        if (jobsForUid.get(i).getSourcePackageName().equals(pkgName)) {
                            if (DEBUG) {
                                Slog.d(TAG, "Restart query: package " + pkgName + " at uid "
                                        + pkgUid + " has jobs");
                            }
                            setResultCode(Activity.RESULT_OK);
                            break;
                        }
                    }
                }

            } else if (Intent.ACTION_PACKAGE_RESTARTED.equals(action)) { // package 被重启 的广播！
                // possible force-stop
                final int pkgUid = intent.getIntExtra(Intent.EXTRA_UID, -1);
                final String pkgName = intent.getData().getSchemeSpecificPart();
                if (pkgUid != -1) {
                    if (DEBUG) {
                        Slog.d(TAG, "Removing jobs for pkg " + pkgName + " at uid " + pkgUid);
                    }
                    cancelJobsForPackageAndUid(pkgName, pkgUid);
                }
            }

        }
    };
```
我们来总结一下：

监听的广播有下面的这些：

- package 相关
  - Intent.ACTION_PACKAGE_CHANGED：应用程序包发生改变的广播！
  - Intent.ACTION_PACKAGE_REMOVED：删除应用程序包的广播！
  - Intent.ACTION_QUERY_PACKAGE_RESTART：
  - Intent.ACTION_PACKAGE_RESTARTED：

- user 相关
  - Intent.ACTION_USER_REMOVED：设备用户被移除的广播！

但不管怎样，最后都会到了 cancel 相关的代码了！

# 2 Uid Change
接着来，是监听进程 uid 的变化，依赖于 JSS 内部的一个观察者：mUidObserver：
```java
    final private IUidObserver mUidObserver = new IUidObserver.Stub() {
    
         // 传入的参数是 uid 和 uid 所属进程的状态 !
        @Override public void onUidStateChanged(int uid, int procState) throws RemoteException { // 进程状态变化！
            updateUidState(uid, procState);
        }

        @Override public void onUidGone(int uid) throws RemoteException { // 进程被杀了！
            updateUidState(uid, ActivityManager.PROCESS_STATE_CACHED_EMPTY);
        }

        @Override public void onUidActive(int uid) throws RemoteException { // 进程活跃！
        }

        @Override public void onUidIdle(int uid) throws RemoteException { // 进程空闲！
            cancelJobsForUid(uid, false);
        }
    };
```
这里面调用了一个 方法：
```
    void updateUidState(int uid, int procState) {

        synchronized (mLock) {
            if (procState == ActivityManager.PROCESS_STATE_TOP) {
                // Only use this if we are exactly the top app.  All others can live
                // with just the foreground priority.  This means that persistent processes
                // can never be the top app priority...  that is fine.
                mUidPriorityOverride.put(uid, JobInfo.PRIORITY_TOP_APP);

            } else if (procState <= ActivityManager.PROCESS_STATE_FOREGROUND_SERVICE) {
                mUidPriorityOverride.put(uid, JobInfo.PRIORITY_FOREGROUND_APP);

            } else {
                mUidPriorityOverride.delete(uid);

            }
        }
    }
```
可以看到，这里当应用的进程的状态发生变化时，需要对其进程的状态和uid进行记录：
```java
    /**
     * Which uids are currently in the foreground.
     */
    final SparseIntArray mUidPriorityOverride = new SparseIntArray();
```
这个稀疏数组用来保存进程的 uid 和进程的状态的，当某个应用的进程变为前台进程，每次变化就会被保存到这个集合中！

还记得在 schedule 篇中，我们讲到，对于同一个  uid 和 jobId 的 job，如果优先级不同，会发生优先级高的替换优先级低的 job 的情况，这里会用到如下的方法：

```java
    private int adjustJobPriority(int curPriority, JobStatus job) {
        if (curPriority < JobInfo.PRIORITY_TOP_APP) {
            float factor = mJobPackageTracker.getLoadFactor(job);

            if (factor >= mConstants.HEAVY_USE_FACTOR) {
                curPriority += JobInfo.PRIORITY_ADJ_ALWAYS_RUNNING;
                
            } else if (factor >= mConstants.MODERATE_USE_FACTOR) {
                curPriority += JobInfo.PRIORITY_ADJ_OFTEN_RUNNING;
                
            }
        }
        return curPriority;
    }

    private int evaluateJobPriorityLocked(JobStatus job) {
        // 获得 job 的优先级，默认为：PRIORITY_DEFAULT(0)
        int priority = job.getPriority();
        if (priority >= JobInfo.PRIORITY_FOREGROUND_APP) {
            return adjustJobPriority(priority, job);

        }
        // 这里是判断当前 job 的 uid 所属进程是否是前台进程。
        int override = mUidPriorityOverride.get(job.getSourceUid(), 0);
        if (override != 0) {
            return adjustJobPriority(override, job);

        }
        return adjustJobPriority(priority, job);
    }
```

这个方式是为了给 job 计算合适的优先级！

这里就不细说哦，可以去看 schedule 篇！


