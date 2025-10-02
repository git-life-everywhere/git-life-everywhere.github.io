# Serivce 篇 7 - Service restart 流程分析
title: Serivce 篇 7 - Service restart 流程分析
date: 2016/09/21 20:46:25
catalog: true
categories:
- AndroidFramework源码分析
- Service服务
tags: Service服务
---

本文基于 Android 7.1.1 源码分析，转载请说明出处！

# 0 综述

当服务被 kill 掉后，会根据 onStartCommand 方法的返回值，来决定是否对服务进行重启，我们先来回顾下返回值类型：

- **START_STICKY**：
- **START_STICKY_COMPATIBILITY**：
  - 删除本次启动的 startId 对应的启动项；
  - 设置服务的 stopIfKilled 为 false；
- **START_NOT_STICKY**： 
  - 删除本次启动的服务的 startId 对应的启动项；
  - 如果最后一次启动的启动项的 id(lastStartId) 等于本次启动的启动项的 id；
     - 设置服务的 stopIfKilled 为 true；
- **START_REDELIVER_INTENT**： 
  - 不删除本次启动的 startId 对应的启动项；
  - 如果本次启动的启动项不为 null；
     - 设置启动项的 deliveryCount 为 0；
     - 设置启动项的 doneExecutingCount 加 1；
     - 设置服务的 stopIfKilled 为 true；

下面，我们来分析下服务的重启过程，有两种情况会出现服务的重启：

第一种是，在拉起 Service 的 onCreate 方法时出现异常，这时如果服务没有被 destroy 掉，那就会尝试重新启动服务！

第二种是，服务所在的进程被 kill 掉了，这是会尝试重启服务！


# 1 第一种情况

## 1.1 ActiveServices.realStartServiceLocked
```java
    private final void realStartServiceLocked(ServiceRecord r,
            ProcessRecord app, boolean execInFg) throws RemoteException {

        ... ... ... ...

        try {

            synchronized (r.stats.getBatteryStats()) {
                r.stats.startLaunchedLocked();
            }

            mAm.notifyPackageUse(r.serviceInfo.packageName,
                                 PackageManager.NOTIFY_PACKAGE_USE_SERVICE);
            // 更新服务的 state！
            app.forceProcessStateUpTo(ActivityManager.PROCESS_STATE_SERVICE);
            // 拉起服务的 onCreate 方法
            app.thread.scheduleCreateService(r, r.serviceInfo,
                    mAm.compatibilityInfoForPackageLocked(r.serviceInfo.applicationInfo),
                    app.repProcState);
                    
            r.postNotification();
            created = true;
        } catch (DeadObjectException e) {
            Slog.w(TAG, "Application dead when creating service " + r);
            mAm.appDiedLocked(app);
            throw e;
        } finally {
            if (!created) { // created 为 false，表示拉起失败！
                final boolean inDestroying = mDestroyingServices.contains(r);
                serviceDoneExecutingLocked(r, inDestroying, inDestroying);

                if (newService) {
                    app.services.remove(r);
                    r.app = null;
                }

                //【1】如果服务没有被销毁掉，那就尝试重新启动服务！
                if (!inDestroying) {
                    scheduleServiceRestartLocked(r, false);
                }
            }
        }
        
        ... ... ... ...
    
    }
```

我们继续来看！

## 1.2 ActiveServices.scheduleServiceRestartLocked

参数传递：

- `ServiceRecord r`：要被重启的服务；
- `boolean allowCancel`：这里传入 `false`；

```java
private final boolean scheduleServiceRestartLocked(ServiceRecord r,
            boolean allowCancel) {

        // 变量 cancel 用于表示是否放弃重启服务！
        boolean canceled = false;
        
        //【1】如果系统在关机，那就不重启！
        if (mAm.isShuttingDownLocked()) {

            Slog.w(TAG, "Not scheduling restart of crashed service " + r.shortName
                    + " - system is shutting down");

            return false;
        }

        ServiceMap smap = getServiceMap(r.userId);
        
        //【1】再次和服务集合中的服务匹配校验，如果不是同一个，那就停止重启！
        if (smap.mServicesByName.get(r.name) != r) {
            ServiceRecord cur = smap.mServicesByName.get(r.name);

            Slog.wtf(TAG, "Attempting to schedule restart of " + r
                    + " when found in map: " + cur);

            return false;
        }

        final long now = SystemClock.uptimeMillis();
        
        //【1.1】对于非常驻类型的服务，进入这个分支！    
        if ((r.serviceInfo.applicationInfo.flags
                &ApplicationInfo.FLAG_PERSISTENT) == 0) {

            // minDuration 是被系统重启的时间间隔
            long minDuration = SERVICE_RESTART_DURATION;
            // resetTime 是重置这个重启时间的间隔
            long resetTime = SERVICE_RESET_RUN_DURATION;

            // 便利该服务的 deliveredStarts 集合，获得所有已经被分发的启动项！！
            // 对于那些之前已经分发但是没有完成的启动项，要重新加入到 pendingStarts 队列中！
            final int N = r.deliveredStarts.size();
            if (N > 0) {
                for (int i=N-1; i>=0; i--) {
                    ServiceRecord.StartItem si = r.deliveredStarts.get(i);
                    si.removeUriPermissionsLocked();

                    if (si.intent == null) {

                    } else if (!allowCancel || (si.deliveryCount < ServiceRecord.MAX_DELIVERY_COUNT
                            && si.doneExecutingCount < ServiceRecord.MAX_DONE_EXECUTING_COUNT)) {

                        // 如果不允许取消，显然这里 allowCancel 为 false；
                        // 或者该启动项的分发次数小于最大值 3 次，且处理次数小于最大值 6 次
                        // 才会把该启动项重新添加到 pendingStarts 列表中！
                        r.pendingStarts.add(0, si);
                        
                        long dur = SystemClock.uptimeMillis() - si.deliveredTime;
                        dur *= 2;

                        if (minDuration < dur) minDuration = dur;
                        if (resetTime < dur) resetTime = dur;

                    } else {

                        Slog.w(TAG, "Canceling start item " + si.intent + " in service "
                                + r.name);
                        
                        // 置 canceled 为 true，表示放弃重启服务！
                        canceled = true;

                    }
                }

                r.deliveredStarts.clear();
            }
            
            // 服务的重启次数加一；
            r.totalRestartCount++;
 
            // 计算服务的重启延迟时间；
            if (r.restartDelay == 0) {

                // 如果没有重启过，restartDelay 为 0；
                // 那么重启延迟时间为 minDuration；
                r.restartCount++;
                r.restartDelay = minDuration;

            } else {

                // 如果已经重启过了，进入以下分支；
                // restartTime 是上一次重启的时间；
                if (now > (r.restartTime+resetTime)) {

                    // 并且当前时间距离上次重启时间超过了 resetTime，
                    // 则把 restartCount 重置为 1，restartDelay 设为 minDuration！
                    r.restartCount = 1;
                    r.restartDelay = minDuration;

                } else {

                    // 并且当前时间距离上次重启时间没有超过 resetTime，则调大 restartDelay！
                    r.restartDelay *= SERVICE_RESTART_DURATION_FACTOR;
                    if (r.restartDelay < minDuration) {
                        r.restartDelay = minDuration;
                    }

                }
            }

            // 计算服务的下一次重启时间
            r.nextRestartTime = now + r.restartDelay;

            // 设置所有服务的重启时间差最小为 10 秒！
            // 一个服务重启后，下一个服务至少 10 后才能被重启！
            boolean repeat;
            do {
                repeat = false;
                for (int i=mRestartingServices.size()-1; i>=0; i--) {
                    ServiceRecord r2 = mRestartingServices.get(i);
                    
                    if (r2 != r && r.nextRestartTime
                            >= (r2.nextRestartTime-SERVICE_MIN_RESTART_TIME_BETWEEN)
                            && r.nextRestartTime
                            < (r2.nextRestartTime+SERVICE_MIN_RESTART_TIME_BETWEEN)) {

                        r.nextRestartTime = r2.nextRestartTime + SERVICE_MIN_RESTART_TIME_BETWEEN;
                        r.restartDelay = r.nextRestartTime - now;
                        repeat = true;
                        break;
                    }
                }
            } while (repeat);

        } else {

            //【1.2】常驻进程的服务要被立刻重启！
            r.totalRestartCount++;
            r.restartCount = 0;
            r.restartDelay = 0;
            r.nextRestartTime = now;
        }
        
        //【2】如果 mRestartingServices 列表中服务该服务，将该服务添加进去！
        if (!mRestartingServices.contains(r)) {
            r.createdFromFg = false;
            mRestartingServices.add(r);
            r.makeRestarting(mAm.mProcessStats.getMemFactorLocked(), now);
        }
        
        //【3】取消前台的 notification，这个是通过 startForegroud 的方式设置的！
        cancelForegroudNotificationLocked(r);

        //【4】这里是最关键的地方，执行重启服务的操作！！
        // 这里会使用 AMS.MainHandler 来执行重启任务
        mAm.mHandler.removeCallbacks(r.restarter);
        mAm.mHandler.postAtTime(r.restarter, r.nextRestartTime);
        
        // 计算下次重启时间！
        r.nextRestartTime = SystemClock.uptimeMillis() + r.restartDelay;

        Slog.w(TAG, "Scheduling restart of crashed service "
                + r.shortName + " in " + r.restartDelay + "ms");
        EventLog.writeEvent(EventLogTags.AM_SCHEDULE_SERVICE_RESTART,
                r.userId, r.shortName, r.restartDelay);

        return canceled;
    }
```
这里涉及了几个常量：

- ActiveServices 
```java
    // 两个服务的重启间隔的最小值！
    static final int SERVICE_MIN_RESTART_TIME_BETWEEN = 10*1000;
    
    // 服务延迟重启的时间间隔
    static final int SERVICE_RESTART_DURATION = 1*1000;
    
    // 重置服务的重启时间的时间间隔
    static final int SERVICE_RESET_RUN_DURATION = 60*1000;
    
    // 用于扩大服务重启的时间间隔的乘法因子
    static final int SERVICE_RESTART_DURATION_FACTOR = 4;
```
- ServiceRecord
```java
    // 放弃重启操作前，启动项分发的最大次数
    static final int MAX_DELIVERY_COUNT = 3;

    // 放弃重启操作前，启动项执行的最大次数
    static final int MAX_DONE_EXECUTING_COUNT = 6;
```

还记得 ServiceRestarter，在启动服务前，会创建一个 ServiceRestarter 对象，用于重启服务！


我们看到，如果服务之前重启过，并且当前时间距离上次重启时间没有超过 resetTime，则调大 restartDelay！

这是为了防止 service 被在内存不足的情况下被频繁重启，第一次内存不足时杀掉 service，1s 后重启该 service，重启后又消耗了一部分内存造成内存再次不足再次杀掉 service，这时1s后就不应该重启了，要往后推迟一段时间再尝试重启。

# 3 ServiceRestarter.run
```java
    private class ServiceRestarter implements Runnable {
        private ServiceRecord mService;

        void setService(ServiceRecord service) {
            mService = service;
        }

        public void run() {
            synchronized(mAm) {
                
                //【1】重启服务！
                performServiceRestartLocked(mService);
            }
        }
    }
```
继续看！


# 4 ActiveServices.performServiceRestartLocked

```java
    final void performServiceRestartLocked(ServiceRecord r) {
        // 如果服务没有在重启列表中，就不重启！
        if (!mRestartingServices.contains(r)) {
            return;
        }
        
        // 如果服务不再需要，就不重启！
        if (!isServiceNeeded(r, false, false)) {
            Slog.wtf(TAG, "Restarting service that is not needed: " + r);
            return;
        }

        try {
            
            //【1】进入 bringUpServiceLocked 方法，执行重启服务操作！
            bringUpServiceLocked(r, r.intent.getIntent().getFlags(), r.createdFromFg, true, false);
        } catch (TransactionTooLargeException e) {
            // Ignore, it's been logged and nothing upstack cares.
        }
    }
```
这里和前面的 startService 的逻辑是一样的！但是我们重点要看的是，对于 onStartCommand 方法的返回值，在重启的时候，系统是如何处理的！！


# 5 onStartCommand 返回值的不同处理

## 5.1 START_STICKY（默认返回值）

对于 Service.START_STICKY 的返回值，服务所在进程被 kill 后，系统会重新启动该服务以及所在进程！

onStartCommand 方法在返回 Service.START_STICKY 后，系统会做如下处理：

- 从 r.deliveredStarts 中删除 startId 对应的启动项
- 设置 r.stopIfKilled = false
- 设置 r.callStart = true

对于这种返回值，服务的 r.deliveredStarts 和 r.pendingStarts 中的启动项都为空，所以在 scheduleServiceRestartLocked 方法中，不会遍历 deliveredStarts 集合，所以 scheduleServiceRestartLocked 的返回值 canceled 为 false；

sr.startRequested 在开始启动时就被设置成了 true！这样，scheduleServiceRestartLocked 方法返回后，就不会进入这个分支：
```java
// sr.startRequested: true sr.stopIfKilled: false canceled: false，不进入该分支！
if (sr.startRequested && (sr.stopIfKilled || canceled)) {

    ... ... ... 

    bringDownServiceLocked(sr);
}
```
接下来，就是服务的重启了，这时就进入 bringUpServiceLocked 方法，因为对于 Service.START_STICKY 的返回值：
```java
if(r.startRequested && r.callStart && r.pendingStarts.size() == 0) {
}
```
这个的条件是满足的，那就会创建一个 Intent 为 null 启动项，添加到 r.pendingStarts 中，如果你在 onStartCommand 方法中，对参数 Intent 做一个非空判断，你会发现，这个 Intent 为 null！！
```java
    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        if (intent == null) {
            Log.d(TAG, "the intent is null!");
        }
        return Service.START_STICKY;
    }
```

## 5.2 START_NOT_STICKY

对于 Service.START_NOT_STICKY 的返回值，服务所在进程被 kill 后，系统不会重新启动该服务以及所在进程！

onStartCommand 方法在返回 Service.START_NOT_STICKY 后，系统会做如下处理：

- 从 r.deliveredStarts 中删除 startId 对应的启动项；
- 如果 r.getLastStartId() == startId，设置 r.stopIfKilled = true；（**和 START_STICKY 的不同之处**）
- r.callStart = true；

对于这种返回值，服务的 r.deliveredStarts 和 r.pendingStarts 中的启动项也都为空，所以在 scheduleServiceRestartLocked 方法中，不会遍历 deliveredStarts 集合，所以 scheduleServiceRestartLocked 的返回值 canceled 同样也为 false；

而 sr.startRequested 在开始启动时就被设置成了 true！这样，scheduleServiceRestartLocked 返回后，根据结果，就会进入这个分支：
```java
// sr.startRequested: true sr.stopIfKilled: true canceled: false
if(r.startRequested && r.callStart && r.pendingStarts.size() == 0) {
   
    // 同样的 sr.pendingStarts.size() == 0 满足条件，进入该分支！
    if (sr.pendingStarts.size() == 0) {
        sr.startRequested = false;
        if (sr.tracker != null) {
            sr.tracker.setStarted(false, mAm.mProcessStats.getMemFactorLocked(),
                    SystemClock.uptimeMillis());
        }
        // 如果之前服务没有自动创建的连接，就停止服务！
        if (!sr.hasAutoCreateConnections()) {
             bringDownServiceLocked(sr);
        }
    }
}
```
如果我们之前只是通过 startService 的方式，启动了服务，那么，这里就会调用 bringDownServiceLocked 停止服务，所以，**单纯的 startService，如果 onStartCommand 返回值为 Service.START_NOT_STICKY，那么服务是不会重启的**！

但是如果还通过 bindSerivce（Service.BIND_AUTO_CREATE）的方式创建了连接，那么还是会重启服务！


## 5.2 START_REDELIVER_INTENT

对于 Service.START_REDELIVER_INTENT 的返回值，服务所在进程被 kill 后，系统会重新启动该服务以及所在进程，并且会将最后启动（startService）分发过的 Intent 再次通过 onStartCommand 方法传递给服务！

onStartCommand 方法在返回 START_REDELIVER_INTENT 后，系统会做如下处理:

- 从 r.deliveredStarts 中获得 startId 对应的启动项 StartItem si ，【不删除】；
- 如果 si != null：
	- si.deliveryCount = 0;
    - si.doneExecutingCount++;（这样设置，在 7.2.3 的时候，flags 就会被设置 START_FLAG_REDELIVER 的标签）
    - r.stopIfKilled = true;
- r.callStart = true;

因为没有删除 r.deliveredStarts 中已经被分发的启动项，所以也就意味着，如果是多次 startSerivce，那么 r.deliveredStarts 会将所有的启动项都保留下来！

对于这种返回值，服务的 r.deliveredStarts 保存了所有的被分发的启动项，而 r.pendingStarts 中的启动项为空，我们先到 scheduleServiceRestartLocked 方法中去！

在 scheduleServiceRestartLocked 方法中，会将服务的 r.deliveredStarts 中所有的启动项，添加到 r.pendingStarts 中，用于重启后分发！

scheduleServiceRestartLocked 方法返回后，由于 sr.startRequested 为 true，sr.stopIfKilled 为 true， 且 canceled 为 false 满足条件，则会进入以下分支：
```java
// sr.startRequested: true sr.stopIfKilled: true canceled: false
if(r.startRequested && r.callStart && r.pendingStarts.size() == 0) {
   
    // 然而 sr.pendingStarts.size() ！= 0 ，不进入分支！
    if (sr.pendingStarts.size() == 0) {
    }
}
```
但是由于此时，sr.pendingStarts.size() 是不为 0 的，所以就不会停止服务，而是重启服务！因为所有的启动项 si.doneExecutingCount 都大于 0，所以 onStartCommand 方法的参数 flags 会设置 Service.START_FLAG_REDELIVERY 标志位！

当 onStartCommand 方法被拉起后，传入的 Intent 就是之前 startService 时传入的 Intent，如果之前多次调用了 startService 方法，那么当服务被重启后，onStartCommand 方法会被拉起多次，传入的 Intent 则是按照之前多次调用 startService 的 startId 顺序进行传入！


第二种是，当服务所在的进程因内存不足而被 `kill` 掉，会触发 `AMS.cleanUpApplicationRecordLocked` 方法，用于清理进程的资源：

```java
    private final boolean cleanUpApplicationRecordLocked(ProcessRecord app,
            boolean restarting, boolean allowRestart, int index, boolean replacingPid) {
        Slog.d(TAG, "cleanUpApplicationRecord -- " + app.pid);
    
        ... ... ... ...
        
        //【1】这里会杀掉该进程中的所有服务，allownRestart 方式是否允许服务重启！
        mServices.killServicesLocked(app, allowRestart);
        
        ... ... ... ...
        
    }
```


# 1 ActiveServices.killServicesLocked

参数传递：

- `ProcessRecord app`：被 `kill` 掉的进程！
- `boolean allowRestart`：表示是否重启服务，当进程被杀掉后，传入 `true`！

```java
    final void killServicesLocked(ProcessRecord app, boolean allowRestart) {

        // 默认不进入该分支，我们不看！
        if (false) {
            ... ... ...
        }

        // 清楚这个进程持有的所有的连接对象 ConnectionRecord！
        for (int i = app.connections.size() - 1; i >= 0; i--) {
            ConnectionRecord r = app.connections.valueAt(i);
            
            // 这里和调用 unbindService 方法很类似，主要是解除该进程对其他服务的绑定！
            removeConnectionLocked(r, app, null);

        }
        updateServiceConnectionActivitiesLocked(app);
        app.connections.clear();

        app.whitelistManager = false;

        //【1】遍历被 kill 掉的进程中的所有服务!
        for (int i = app.services.size() - 1; i >= 0; i--) {
            ServiceRecord sr = app.services.valueAt(i);
            synchronized (sr.stats.getBatteryStats()) {
                sr.stats.stopLaunchedLocked();
            }
            
            // 如果该服务所在进程不是当前进程，并且进程仍在运行，且不是常驻进程
            // 那就从服务所在进程中移除该服务！
            if (sr.app != app && sr.app != null && !sr.app.persistent) {
                sr.app.services.remove(sr);
            }

            sr.app = null;
            sr.isolatedProc = null;
            sr.executeNesting = 0;
            sr.forceClearTracker();

            if (mDestroyingServices.remove(sr)) {
                if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "killServices remove destroying " + sr);
            }
 
            // 在 ServiceRecord.bindings 中保存了绑定该服务的所有 intent 和 IntentBindRecord 的映射关系！
            final int numClients = sr.bindings.size();
            
            for (int bindingi=numClients-1; bindingi>=0; bindingi--) {
                IntentBindRecord b = sr.bindings.valueAt(bindingi);
                
                if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "Killing binding " + b
                        + ": shouldUnbind=" + b.hasBound);
                    
                // 初始化 IntentBindRecord 一切属性！  
                b.binder = null;
                b.requested = b.received = b.hasBound = false;

                // If this binding is coming from a cached process and is asking to keep
                // the service created, then we'll kill the cached process as well -- we
                // don't want to be thrashing around restarting processes that are only
                // there to be cached.
                // 遍历 IntentBindRecord 对象的 apps 哈希表，该哈希表保存了使用该 Intent 绑定该服务的所有进程
                // ProcessRecord 和进程绑定信息 AppBindRecord 的映射关系！
                for (int appi=b.apps.size()-1; appi>=0; appi--) {
                    final ProcessRecord proc = b.apps.keyAt(appi);

                    // 如果进程被杀了或者没有启动，就跳过处理！
                    if (proc.killedByAm || proc.thread == null) {
                        continue;
                    }

                    // Only do this for processes that have an auto-create binding;
                    // otherwise the binding can be left, because it won't cause the
                    // service to restart.
                    // 判断该进程持有的所有连接对象 ConnectionRecord 是否是通过 BIND_FLAG_AUTO 方式创建的！
                    final AppBindRecord abind = b.apps.valueAt(appi);
                    boolean hasCreate = false;

                    for (int conni=abind.connections.size()-1; conni>=0; conni--) {
                        ConnectionRecord conn = abind.connections.valueAt(conni);

                        if ((conn.flags&(Context.BIND_AUTO_CREATE|Context.BIND_ALLOW_OOM_MANAGEMENT
                                |Context.BIND_WAIVE_PRIORITY)) == Context.BIND_AUTO_CREATE) {
                                
                            // 设 hasCreate 为 true，表示存在自动创建的绑定！
                            hasCreate = true;
                            break;
                        }
                    }

                    if (!hasCreate) {

                        // 不存在自动创建的连接，就跳过这个服务！
                        continue;
                    }

                    // 如果存在自动创建的连接，如果该进程不是常驻进程，并且该进程在运行中，
                    // 该进程的 pid 不等于 0，也不等于系统进程 pid，且进程的状态不低于 PROCESS_STATE_LAST_ACTIVITY
                    // 那就要尝试杀掉进程，解除绑定！
                    if (false && proc != null && !proc.persistent && proc.thread != null
                            && proc.pid != 0 && proc.pid != ActivityManagerService.MY_PID
                            && proc.setProcState >= ActivityManager.PROCESS_STATE_LAST_ACTIVITY) {

                        proc.kill("bound to service " + sr.name.flattenToShortString()
                                + " in dying proc " + (app != null ? app.processName : "??"), true);
                    }
                }
            }
        }

        ServiceMap smap = getServiceMap(app.userId);

        //【2】开始处理被 kill 的进程中的服务！
        for (int i=app.services.size()-1; i>=0; i--) {
            ServiceRecord sr = app.services.valueAt(i);

            // 如果该进程不是常驻进程，就要从进程的 services 中移除该服务！
            if (!app.persistent) {
                app.services.removeAt(i);
            }

            // 如果当前的服务和服务集合中的服务无法匹配，就跳过不处理！
            final ServiceRecord curRec = smap.mServicesByName.get(sr.name);
            if (curRec != sr) {
                if (curRec != null) {
                    Slog.wtf(TAG, "Service " + sr + " in process " + app
                            + " not same as in map: " + curRec);
                }
                continue;
            }

            if (allowRestart && sr.crashCount >= 2 && (sr.serviceInfo.applicationInfo.flags
                    &ApplicationInfo.FLAG_PERSISTENT) == 0) {
                // 如果允许重启，且服务的 crash 次数大于 2，并且服务不是常驻服务，停止服务！
                    
                Slog.w(TAG, "Service crashed " + sr.crashCount
                        + " times, stopping: " + sr);

                EventLog.writeEvent(EventLogTags.AM_SERVICE_CRASHED_TOO_MUCH,
                        sr.userId, sr.crashCount, sr.shortName, app.pid);

                bringDownServiceLocked(sr);

            } else if (!allowRestart
                    || !mAm.mUserController.isUserRunningLocked(sr.userId, 0)) {
                
                // 如果不允许重启或者服务所在的设备用户不再运行状态，就停止服务！
                bringDownServiceLocked(sr);

            } else {

                //【2.1】尝试重启服务！
                boolean canceled = scheduleServiceRestartLocked(sr, true);

                // 根据尝试重启的结果，看是否需要停止服务！
                if (sr.startRequested && (sr.stopIfKilled || canceled)) {
                    if (sr.pendingStarts.size() == 0) {
                        sr.startRequested = false;
                        if (sr.tracker != null) {
                            sr.tracker.setStarted(false, mAm.mProcessStats.getMemFactorLocked(),
                                    SystemClock.uptimeMillis());
                        }
                        
                        // 如果之前服务没有自动创建的连接，就停止服务！
                        if (!sr.hasAutoCreateConnections()) {
                            bringDownServiceLocked(sr);
                        }
                    }
                }
            }
        }
        
        // 清理完成后！
        // 如果不允许重启，就把服务从 ServiceRecord 从 mRestartingServices 和 mPendingServices 中删除！
        if (!allowRestart) {
            app.services.clear();

            for (int i=mRestartingServices.size()-1; i>=0; i--) {

                ServiceRecord r = mRestartingServices.get(i);
                if (r.processName.equals(app.processName) &&
                        r.serviceInfo.applicationInfo.uid == app.info.uid) {

                    mRestartingServices.remove(i);
                    clearRestartingIfNeededLocked(r);
                }
            }
            for (int i=mPendingServices.size()-1; i>=0; i--) {
                ServiceRecord r = mPendingServices.get(i);
                if (r.processName.equals(app.processName) &&
                        r.serviceInfo.applicationInfo.uid == app.info.uid) {

                    mPendingServices.remove(i);
                }
            }
        }

        // 从 mDestroyingServices 中删除该服务！
        int i = mDestroyingServices.size();
        while (i > 0) {
            i--;
            ServiceRecord sr = mDestroyingServices.get(i);
            if (sr.app == app) {

                sr.forceClearTracker();
                mDestroyingServices.remove(i);
                if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "killServices remove destroying " + sr);
            }
        }

        app.executingServices.clear();
    }

```
这里会调用 `scheduleServiceRestartLocked` 方法来重启服务，我们继续看！

# 10 总结

## 10.1 相关 Log
```java
01-04 19:28:23.824  1765  3962 W ActiveServices:  scheduleServiceRestartLocked  N 0 now 7648574 r ServiceRecord{9463951 u0 com.cooqi.servicedemo/.service.AService}
01-04 19:28:23.824  1765  3962 W ActiveServices:  scheduleServiceRestartLocked  r.totalRestartCount 2 r ServiceRecord{9463951 u0 com.cooqi.servicedemo/.service.AService}
01-04 19:28:23.824  1765  3962 W ActiveServices: r.name ComponentInfo{com.cooqi.servicedemo/com.cooqi.servicedemo.service.AService} N 0 minDuration 1000 resetTime 60000 now 7648574 r.restartDelay 0 r.restartTime+resetTime 7605288 allowCancel true
01-04 19:28:23.825  1765  3962 W ActiveServices: r.name ComponentInfo{com.cooqi.servicedemo/com.cooqi.servicedemo.service.AService} N 0 minDuration 1000 resetTime 60000 now 7648574 r.restartDelay 1000 r.restartTime+resetTime 7605288 r.nextRestartTime 7649574 allowCancel true
01-04 19:28:23.825  1765  3962 W ActiveServices: r ServiceRecord{9463951 u0 com.cooqi.servicedemo/.service.AService} r.restartDelay 1000 r.nextRestartTime 7649574
01-04 19:28:23.825  1765  3962 W ActiveServices: r ServiceRecord{9463951 u0 com.cooqi.servicedemo/.service.AService} r.restartDelay 1000 r.nextRestartTime 7649575
01-04 19:28:23.825  1765  3962 W ActiveServices: Scheduling restart of crashed service com.cooqi.servicedemo/.service.AService in 1000ms
01-04 19:28:23.825  1765  3962 W ActiveServices: Restarting list - i 0 r2.nextRestartTime 7649575 r2.name ComponentInfo{com.cooqi.servicedemo/com.cooqi.servicedemo.service.AService}
01-04 19:28:23.826  1765  3962 V ActiveServices: scheduleServiceRestartLocked r ServiceRecord{9463951 u0 com.cooqi.servicedemo/.service.AService} call by com.android.server.am.ActiveServices.killServicesLocked:3066 com.android.server.am.ActivityManagerService.cleanUpApplicationRecordLocked:19081 com.android.server.am.ActivityManagerService.handleAppDiedLocked:5813 com.android.server.am.ActivityManagerService.appDiedLocked:5988 com.android.server.am.ActivityManagerService$AppDeathRecipient.binderDied:1656 android.os.BinderProxy.sendDeathNotice:688 <bottom of call stack> <bottom of call stack>
01-04 19:28:24.827  1765  1788 V ActiveServices: Bringing up ServiceRecord{9463951 u0 com.cooqi.servicedemo/.service.AService} android.content.Intent$FilterComparison@7df9497b
01-04 19:28:24.828  1765  1788 V ActiveServices_MU: bringUpServiceLocked: appInfo.uid=10106 app=null
01-04 19:28:24.943  1765  1804 V ActiveServices_MU: realStartServiceLocked, ServiceRecord.uid = 10106, ProcessRecord.uid = 10106
01-04 19:28:24.943  1765  1804 V ActiveServices: >>> EXECUTING create of ServiceRecord{9463951 u0 com.cooqi.servicedemo/.service.AService} in app ProcessRecord{ea46ad6 9319:com.cooqi.servicedemo/u0a106}
01-04 19:28:24.943  1765  1804 V ActiveServices: bumpServiceExecutingLocked r.executeNesting 0
01-04 19:28:24.943  1765  1804 V ActiveServices: bumpServiceExecutingLocked r.app.executingServices.size() 1
01-04 19:28:24.962  1765  1804 V ActiveServices: Sending arguments to: ServiceRecord{9463951 u0 com.cooqi.servicedemo/.service.AService} android.content.Intent$FilterComparison@7df9497b args=null
01-04 19:28:24.962  1765  1804 V ActiveServices: >>> EXECUTING start of ServiceRecord{9463951 u0 com.cooqi.servicedemo/.service.AService} in app ProcessRecord{ea46ad6 9319:com.cooqi.servicedemo/u0a106}
01-04 19:28:24.962  1765  1804 V ActiveServices: bumpServiceExecutingLocked r.executeNesting 1
01-04 19:28:25.112  9319  9319 D ServiceTest: AService: onCreate
01-04 19:28:25.112  1765  3964 V ActiveServices: serviceDoneExecutingLocked ServiceRecord= ServiceRecord{9463951 u0 com.cooqi.servicedemo/.service.AService} type= 0 startId= 0 res= 0
01-04 19:28:25.112  9319  9319 D ServiceTest: AService: onStartCommand
01-04 19:28:25.113  1765  3964 V ActiveServices: <<< DONE EXECUTING ServiceRecord{9463951 u0 com.cooqi.servicedemo/.service.AService}: nesting=2, inDestroying=false, app=ProcessRecord{ea46ad6 9319:com.cooqi.servicedemo/u0a106}
01-04 19:28:25.113  1765  3964 V ActiveServices: serviceDoneExecutingLocked ServiceRecord= ServiceRecord{9463951 u0 com.cooqi.servicedemo/.service.AService} type= 1 startId= 6 res= 1
01-04 19:28:25.113  1765  3964 V ActiveServices: <<< DONE EXECUTING ServiceRecord{9463951 u0 com.cooqi.servicedemo/.service.AService}: nesting=1, inDestroying=false, app=ProcessRecord{ea46ad6 9319:com.cooqi.servicedemo/u0a106}
01-04 19:28:25.113  1765  3964 V ActiveServices: Nesting at 0 of com.cooqi.servicedemo/.service.AService
01-04 19:28:25.113  1765  3964 V ActiveServices: r.app.executingServices.size(): 0
01-04 19:28:25.113  1765  3964 V ActiveServices: No more executingServices of com.cooqi.servicedemo/.service.AService
```
