# Process篇 5 - 进程的 oomAdj 调度算法
title: Process篇 5 - 进程的 oomAdj 调度算法
date: 2016/05/15 20:46:25
catalog: true
categories:
- AndroidFramework源码分析
- Process进程
tags: Process进程
---

[toc]

基于 Android 7.1.1 源码，总结进程状态调整的调度策略！


# 1 进程 oomAdj 的调度方法


和进程的 oomAdj 调度相关的主要有`3 个方法：

- **updateOomAdjLocked**：用于更新进程的 adj，该方法会依次调用 computeOomAdjLocked 和 applyOomAdjLocked;
- **computeOomAdjLocked**：计算进程的 adj，返回计算后 RawAdj 值;
- **applyOomAdjLocked**：应用 adj，当需要杀掉目标进程则返回 false；否则返回 true。

当在一些特定的场景下，系统会调用 updateOomAdjLocked 来更新指定进程的 oomAdj，该方法会先调用 computeOomAdjLocked 计算出一个新的 oomAdj，然后在调用 applyOomAdjLocked 来设置指定进程的 oomAdj！

updateOomAdjLocked 有多个重载函数：

```java
// 更新指定进程的 oomAdj
boolean updateOomAdjLocked(ProcessRecord app) 
boolean updateOomAdjLocked(ProcessRecord app, int cachedAdj, ProcessRecord TOP_APP, boolean doingAll, long now)

// 更新所有进程的 oomAdj
void updateOomAdjLocked()
```
**其中，五个参数的 updateOomAdjLocked 方法是私有方法，只提供给其他的两个方法调用！！**

```java
boolean updateOomAdjLocked(ProcessRecord app, int cachedAdj, ProcessRecord TOP_APP, boolean doingAll, long now)
```

# 2 调度时机

接下里，我们看看 updateOomAdjLocked 的调度时机：

## 2.1 Activity 调整

和 Activity 相关的调整代码主要是位于 ActivityStackSupervisor.java 和 ActivityStack.java 中：

### 2.1.1 realStartActivityLocked

ActivityStackSupervisor.realStartActivityLocked：启动 Activity，以下是关键代码段：

```java
        //【1】将 activity 所在的进程添加到 lru process 列表中！
        mService.updateLruProcessLocked(app, true, null);
        //【2】更新所有进程的 oomAdj！
        mService.updateOomAdjLocked();
```

在启动之前会调用 updateOomAdjLocked 更新所有进程的 oomAdj！

### 2.1.2 resumeTopActivityInnerLocked

ActivityStack.resumeTopActivityInnerLocked: 恢复栈顶 Activity，以下是关键代码段：

```java
            //【1】next 表示的是下一个要被 resume 的 activity
            next.state = ActivityState.RESUMED;
            mResumedActivity = next;
            next.task.touchActiveTime();
            mRecentTasks.addLocked(next.task);
            //【2】更新 lru process 列表！
            mService.updateLruProcessLocked(next.app, true, null);
            //【3】更新 lru activity 列表！
            updateLRUListLocked(next);
            //【4】更新所有进程的 oomAdj 值
            mService.updateOomAdjLocked();
```
更新所有进程的 oomAdj！

### 2.1.3 finishCurrentActivityLocked

ActivityStack.finishCurrentActivityLocked: 结束 Activity，以下是关键代码段：

```java
        if (mode == FINISH_AFTER_VISIBLE && (r.visible || r.nowVisible)
                && next != null && !next.nowVisible) {
            if (!mStackSupervisor.mStoppingActivities.contains(r)) {
                addToStopping(r, false /* immediate */);
            }
            if (DEBUG_STATES) Slog.v(TAG_STATES,
                    "Moving to STOPPING: "+ r + " (finish requested)");
            // 将 activity 的状态改为 stopping！
            r.state = ActivityState.STOPPING;
            //【1】如果 oomAdj 为 true，那就需要调整所有进程的 `oomAdj`
            if (oomAdj) {
                mService.updateOomAdjLocked();
            }
            return r;
        }
```

更新所有进程的 oomAdj！

### 2.1.4 destroyActivityLocked

ActivityStack.destroyActivityLocked: 摧毁 Activity，以下是关键代码段：

```java
        if (hadApp) {
            if (removeFromApp) {
                r.app.activities.remove(r);
                if (mService.mHeavyWeightProcess == r.app && r.app.activities.size() <= 0) {
                    mService.mHeavyWeightProcess = null;
                    mService.mHandler.sendEmptyMessage(
                            ActivityManagerService.CANCEL_HEAVY_NOTIFICATION_MSG);
                }
                //【1】如果被 destroy 的 activity 所在的进程已经没有任何 activity 了，那就需要调整 oomAdj！
                if (r.app.activities.isEmpty()) {
                    //【1.1】通知所有的服务，该进程没有任何 activity
                    mService.mServices.updateServiceConnectionActivitiesLocked(r.app);
                    //【1.2】该进程没有了 activity，需要更新 lru list 和 oomAdj 的值！
                    mService.updateLruProcessLocked(r.app, false, null);
                    //【1.2】更新所有进程的 adj
                    mService.updateOomAdjLocked();
                }
            }
            ... ... ... ...
        }
```
会更新所有进程的 oomAdj！

## 2.2 Service 调整

和 Service 相关的调整代码主要是位于 ActiveServices.java 中：

### 2.2.1 realStartServiceLocked

**ActiveServices.realStartServiceLocked**: 启动服务，以下是关键代码段：

```java
        final boolean newService = app.services.add(r);
        bumpServiceExecutingLocked(r, execInFg, "create"); // 设置拉起 onCreate 方法超时处理！
        mAm.updateLruProcessLocked(app, false, null);
        mAm.updateOomAdjLocked(); //【1】更新所有进程的 oomAdj
```

### 2.2.2 bindServiceLocked

**ActiveServices.bindServiceLocked**: 绑定服务，以下是关键代码段：

```java
            if (s.app != null) { // 这里的 s.app 表示服务所在的进程！
                if ((flags&Context.BIND_TREAT_LIKE_ACTIVITY) != 0) {
                    s.app.treatLikeActivity = true;
                }
                if (s.whitelistManager) {
                    s.app.whitelistManager = true;
                }
                //【1】当我们 bind 一个服务的时候，该服务的优先级会被提升！
                mAm.updateLruProcessLocked(s.app, s.app.hasClientActivities
                        || s.app.treatLikeActivity, b.client);
                //【2】更新服务所在进程的 oomAdj
                mAm.updateOomAdjLocked(s.app);
            }
```
只更新服务所在进程的 oomAdj!

### 2.2.3 unbindServiceLocked

**ActiveServices.unbindServiceLocked**: 解绑服务，以下是关键代码段：

```java
                if (r.binding.service.app != null) { // r.binding.service.app 在这里表示的是服务所在的进程！
                    if (r.binding.service.app.whitelistManager) {
                        updateWhitelistManagerLocked(r.binding.service.app);
                    }
                    //【1】当我们 unbind 一个服务的时候，该服务的优先级会被降低！
                    if ((r.flags&Context.BIND_TREAT_LIKE_ACTIVITY) != 0) {
                        r.binding.service.app.treatLikeActivity = true;
                        mAm.updateLruProcessLocked(r.binding.service.app,
                                r.binding.service.app.hasClientActivities
                                || r.binding.service.app.treatLikeActivity, null);
                    }
                    //【2】更新服务所在进程的 oomAdj！
                    mAm.updateOomAdjLocked(r.binding.service.app);
                }
```
只更新服务所在进程的 oomAdj！！

### 2.2.4 bringDownServiceLocked

**ActiveServices.bringDownServiceLocked**: 结束服务，以下是关键代码段：

```java
        // 当我们要结束一个服务的时候，我们需要接触所有的绑定操作！
        if (r.app != null && r.app.thread != null) {
            for (int i=r.bindings.size()-1; i>=0; i--) {
                IntentBindRecord ibr = r.bindings.valueAt(i);
                if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "Bringing down binding " + ibr
                        + ": hasBound=" + ibr.hasBound);
                // 如果有绑定操作，接触绑定！
                if (ibr.hasBound) {
                    try {
                        bumpServiceExecutingLocked(r, false, "bring down unbind");

                        //【1】更新服务所在进程的 oomAdj！
                        mAm.updateOomAdjLocked(r.app);
                        ibr.hasBound = false;

                        //【2】拉起服务的 onUnBind 方法！
                        r.app.thread.scheduleUnbindService(r,
                                ibr.intent.getIntent());
                    } catch (Exception e) {
                        Slog.w(TAG, "Exception when unbinding service "
                                + r.shortName, e);
                        serviceProcessGoneLocked(r);
                    }
                }
            }
        }
        
        ... ... ... ...
        
        if (r.app != null) {
            synchronized (r.stats.getBatteryStats()) {
                r.stats.stopLaunchedLocked();
            }
            r.app.services.remove(r);
            if (r.whitelistManager) {
                updateWhitelistManagerLocked(r.app);
            }
            if (r.app.thread != null) {
                updateServiceForegroundLocked(r.app, false);
                try {
                    //【1】设置拉起 onDestroy 方法的超时处理！
                    bumpServiceExecutingLocked(r, false, "destroy");
                    mDestroyingServices.add(r);
                    r.destroying = true;
                    //【2】更新服务所在进程的 oomAdj 值！
                    mAm.updateOomAdjLocked(r.app);
                    
                    //【3】拉起服务的 onDestroy 服务！
                    r.app.thread.scheduleStopService(r);
                } catch (Exception e) {
                    Slog.w(TAG, "Exception when destroying service "
                            + r.shortName, e);
                    serviceProcessGoneLocked(r);
                }
            } else {
                if (DEBUG_SERVICE) Slog.v(
                    TAG_SERVICE, "Removed service that has no process: " + r);
            }
        } else {
            if (DEBUG_SERVICE) Slog.v(
                TAG_SERVICE, "Removed service that is not running: " + r);
        }
```
只更新服务所在进程的 oomAdj!

### 2.2.5 sendServiceArgsLocked

**ActiveServices.sendServiceArgsLocked**: 发送启动参数，该方法会拉起服务的 onStart 方法，以下是关键代码段：

```java
        bumpServiceExecutingLocked(r, execInFg, "start"); // 设置拉起 onStart 方法超时处理
        //【1】当 oomAdjusted 为 fasle，才会执行 oomAdj 的更新！
        if (!oomAdjusted) {
            oomAdjusted = true;
            mAm.updateOomAdjLocked(r.app); // 更新服务所在进程的 oomAdj！
        }
```
sendServiceArgsLocked 在很多地方都有调用：

- **realStartServiceLocked** 方法在拉起 onCreate 方法之前，会对所有进程的 oomAdj 做一次调整；接着调用 sendServiceArgsLocked，但是由于其第三个参数 oomAdjusted 传入的是 true，所以不会再次 oomAdj 更新！

- **cleanUpRemovedTaskLocked** 方法是当我们从最近任务中移除一个任务时候，我们会移除该任务所对应的 Service，那么如果 Service 没有设置 ServiceInfo.FLAG_STOP_WITH_TASK 标志位的话，并且如果服务所在进程没有死亡，那么系统会重新拉起服务：

```java                                                                                           
        for (int i = services.size() - 1; i >= 0; i--) {
            ServiceRecord sr = services.get(i);
            if (sr.startRequested) {
                if ((sr.serviceInfo.flags&ServiceInfo.FLAG_STOP_WITH_TASK) != 0) {
                    Slog.i(TAG, "Stopping service " + sr.shortName + ": remove task");
                    stopServiceLocked(sr);
                } else {
                    //【1】创建启动项！
                    sr.pendingStarts.add(new ServiceRecord.StartItem(sr, true,
                            sr.makeNextStartId(), baseIntent, null));
                    if (sr.app != null && sr.app.thread != null) {
                        try {
                            //【2】拉起服务的 onStart 方法！
                            sendServiceArgsLocked(sr, true, false);
                        } catch (TransactionTooLargeException e) {
                            // Ignore, keep going.
                        }
                    }
                }
            }
        }
```
这里，第三个参数 oomAdjusted 传入的是 false，所以会发生 oomAdj 更新！

- **bringUpServiceLocked** 方法中如果服务所在进程已经被启动，并且服务的 onCreate 方法已经被拉起，那就会直接 sendServiceArgsLocked 拉起服务的 onStart 方法！

```java
        if (r.app != null && r.app.thread != null) {
            sendServiceArgsLocked(r, execInFg, false);
            return null;
        }
```

这里，第三个参数 oomAdjusted 也传入的是 false，所以也会发生 oomAdj 更新！

### 2.2.6 serviceDoneExecutingLocked

**ActiveServices.serviceDoneExecutingLocked** 以下是关键代码段：

```java
        if (r.executeNesting <= 0) {
            if (r.app != null) {
                if (DEBUG_SERVICE) Slog.v(TAG_SERVICE,
                        "Nesting at 0 of " + r.shortName);
                r.app.execServicesFg = false;
                r.app.executingServices.remove(r);
                if (r.app.executingServices.size() == 0) {
                    if (DEBUG_SERVICE || DEBUG_SERVICE_EXECUTING) Slog.v(TAG_SERVICE_EXECUTING,
                            "No more executingServices of " + r.shortName);
                    mAm.mHandler.removeMessages(ActivityManagerService.SERVICE_TIMEOUT_MSG, r.app);
                } else if (r.executeFg) {
                    // Need to re-evaluate whether the app still needs to be in the foreground.
                    for (int i=r.app.executingServices.size()-1; i>=0; i--) {
                        if (r.app.executingServices.valueAt(i).executeFg) {
                            r.app.execServicesFg = true;
                            break;
                        }
                    }
                }
                if (inDestroying) {
                    if (DEBUG_SERVICE) Slog.v(TAG_SERVICE,
                            "doneExecuting remove destroying " + r);
                    mDestroyingServices.remove(r);
                    r.bindings.clear();
                }
                //【1】更新服务所在进程的 oomAdj！
                mAm.updateOomAdjLocked(r.app);
            }
```

serviceDoneExecutingLocked 方法的调用时机主要是在：

- 执行启动 Service 等等一些操作时抛出异
- 服务所在进程死亡时；
- bind 操作完成后；
- unbind 操作完成后等等！

### 2.2.7 removeConnectionLocked

**ActiveServices.removeConnectionLocked**：当我们要移除对一个服务的绑定的时候，会触发该方法，以下是关键代码段：

```java
            if (s.app != null && s.app.thread != null && b.intent.apps.size() == 0
                    && b.intent.hasBound) {
                try {
                    bumpServiceExecutingLocked(s, false, "unbind");
                    if (b.client != s.app && (c.flags&Context.BIND_WAIVE_PRIORITY) == 0
                            && s.app.setProcState <= ActivityManager.PROCESS_STATE_RECEIVER) {
                        mAm.updateLruProcessLocked(s.app, false, null);
                    }
                    //【1】更新进程的 oomAdj 的值！
                    mAm.updateOomAdjLocked(s.app);
                    b.intent.hasBound = false;
                    b.intent.doRebind = false;
                    s.app.thread.scheduleUnbindService(s, b.intent.intent.getIntent());
                } catch (Exception e) {
                    Slog.w(TAG, "Exception when unbinding service " + s.shortName, e);
                    serviceProcessGoneLocked(s);
                }
            }
```
removeConnectionLocked 的调用时机如下：

- **killServicesLocked**：当我们要杀掉某个应用进程的时候，我们会解除该进程对其他进程中服务的绑定：
```java
        // Clean up any connections this application has to other services.
        for (int i = app.connections.size() - 1; i >= 0; i--) {
            ConnectionRecord r = app.connections.valueAt(i);
            removeConnectionLocked(r, app, null);
        }
```

- **cleanUpActivityServicesLocked**：
```java
        if (r.connections != null) {
            Iterator<ConnectionRecord> it = r.connections.iterator();
            while (it.hasNext()) {
                ConnectionRecord c = it.next();
                // 移除该绑定关系！
                mService.mServices.removeConnectionLocked(c, null, r);
            }
            r.connections = null;
        }
```
- unbindServiceLocked: 当我们要接触绑定某个服务的时候，会调用 removeConnectionLocked，断开连接！

### 2.5.8 setServiceForegroundLocked

ActiveServices.setServiceForegroundLocked：用于将一个服务设置为前台，关键代码段：

```java
            ServiceRecord r = findServiceLocked(className, token, userId);
            if (r != null) {
                if (id != 0) {
                    ... ... ...
                    if (r.app != null) {
                        //【1】设置服务为前台
                        updateServiceForegroundLocked(r.app, true);
                    }
                    ... ... ...
                } else {
                    if (r.isForeground) {
                        r.isForeground = false;
                        if (r.app != null) {
                            mAm.updateLruProcessLocked(r.app, false, null);
                            //【2】设置服务为前台
                            updateServiceForegroundLocked(r.app, true);
                        }
                    }
                    ... ... ... ...
                }
            }
```
在 updateServiceForegroundLocked 方法中，会调用：AMS.updateProcessForegroundLocked 方法：

```java
    private void updateServiceForegroundLocked(ProcessRecord proc, boolean oomAdj) {
        ... ... ... ...
        //【1】更新前台进程的优先级；
        mAm.updateProcessForegroundLocked(proc, anyForeground, oomAdj);
    }
```
AMS.updateProcessForegroundLocked 中会根据第三个参数是否为 true 来设置进程的 oomAdj：

```java
    final void updateProcessForegroundLocked(ProcessRecord proc, boolean isForeground,
            boolean oomAdj) {
            ... ... ... ...
            if (oomAdj) {
                //【1】调整 oomAdj！
                updateOomAdjLocked();
            }
        }
    }
```
更新所有进程的 oomAdj 的值！

这里我们来说一下 updateProcessForegroundLocked，我们设置一个服务为前台，实际上是设置其所在的进程为前台进程，updateProcessForegroundLocked 的调用有多处，但只有在 setServiceForegroundLocked 方法的调用链中，boolean oomAdj 为 true！！


## 2.3 broadcast 调整

和 broadcast 相关的调整代码主要是在 BroadcastQueue.java 中：

### 2.3.1 processNextBroadcast

BroadcastQueue.processNextBroadcast: 发送下一个广播，以下是关键代码段：
```java
        if (mOrderedBroadcasts.size() == 0) {
            mService.scheduleAppGcsLocked();
            //【1】在处理完最后一个有序广播队列中的广播后，会更新所有进程的 oomAdj！
            if (looped) {
                mService.updateOomAdjLocked();
            }
            return;
        }
```
只针对有序发送的广播！

### 2.3.2 processCurBroadcastLocked

BroadcastQueue.processCurBroadcastLocked: 处理当前广播，以下是关键代码段：

```java
        r.receiver = app.thread.asBinder();
        r.curApp = app;
        app.curReceiver = r;
        //【1】更新进程的状态！
        app.forceProcessStateUpTo(ActivityManager.PROCESS_STATE_RECEIVER);
        //【2】更新 LRU 进程列表！
        mService.updateLruProcessLocked(app, false, null);
        //【3】更新所有进程的 oomAdj 值！
        mService.updateOomAdjLocked();
```
只针对有序发送的广播！

### 2.3.3 deliverToRegisteredReceiverLocked

BroadcastQueue.deliverToRegisteredReceiverLocked: 分发已注册的广播，以下是关键代码段：

```java
        if (ordered) { // 只针对有序发送的广播！
            r.receiver = filter.receiverList.receiver.asBinder();
            r.curFilter = filter;
            filter.receiverList.curBroadcast = r;
            r.state = BroadcastRecord.CALL_IN_RECEIVE;
            if (filter.receiverList.app != null) {
                //【1】r.curApp 表示要接受该广播的目标进程！
                r.curApp = filter.receiverList.app;
                filter.receiverList.app.curReceiver = r;
                //【2】更新目标进程的 oomAdj！
                mService.updateOomAdjLocked(r.curApp);
            }
        }
```
只针对有序发送的广播，只针对指定的目标进程！

## 2.4 ContentProvider 调整

和 ContentProvider 相关的调整代码主要是在 ActivityManagerService.java 中：

### 2.4.1 removeContentProviderXXX

ActivityManagerService.removeContentProvider: 移除 provider，关键代码段：

```java
                //【1】如果该 provider 没有其他人在使用，就会返回 true！
                if (decProviderCountLocked(conn, null, null, stable)) {
                    //【1】更新所有进程的 oomAdj 值！
                    updateOomAdjLocked();
                }
```
更新所有进程的 oomAdj 值！

ActivityManagerService.removeContentProviderExternalUnchecked: 移除 provider，关键代码段：

```java
            //【1】当 contentProvier 有扩展进程的引用，并且引用均被移除，更新所有进程的 oomAdj 值！
            if (localCpr.hasExternalProcessHandles()) {
                if (localCpr.removeExternalProcessHandleLocked(token)) {
                    //【1.1】更新所有进程的 oomAdj 值！
                    updateOomAdjLocked();
                } else {
                    Slog.e(TAG, "Attmpt to remove content provider " + localCpr
                            + " with no external reference for token: "
                            + token + ".");
                }
            } else {
                Slog.e(TAG, "Attmpt to remove content provider: " + localCpr
                        + " with no external references.");
            }
```
更新所有进程的 oomAdj 值！

### 2.4.2 publishContentProviders

ActivityManagerService.publishContentProviders: 发布 provider，关键代码段：

```java
            final int N = providers.size();
            for (int i = 0; i < N; i++) {
                ContentProviderHolder src = providers.get(i);
                if (src == null || src.info == null || src.provider == null) {
                    continue;
                }
                ContentProviderRecord dst = r.pubProviders.get(src.info.name);
                if (DEBUG_MU) Slog.v(TAG_MU, "ContentProviderRecord uid = " + dst.uid);
                if (dst != null) {
                    ComponentName comp = new ComponentName(dst.info.packageName, dst.info.name);
                    mProviderMap.putProviderByClass(comp, dst);
                    String names[] = dst.info.authority.split(";");
                    for (int j = 0; j < names.length; j++) {
                        mProviderMap.putProviderByName(names[j], dst);
                    }

                    ... ... ... ...// 处理 mLaunchingProviders

                    synchronized (dst) {
                        dst.provider = src.provider;
                        dst.proc = r;
                        dst.notifyAll();
                    }
                    //【1】更新指定进程的 oomAdj！
                    updateOomAdjLocked(r);
                    maybeUpdateProviderUsageStatsLocked(r, src.info.packageName,
                            src.info.authority);
                }
            }
```
更新指定进程的 oomAdj！

### 2.4.3 getContentProviderImpl

ActivityManagerService.getContentProviderImpl: 获取 provider，下面是关键代码段！

```java
                ... ... ... ...
                //【1】更新 provider 所在进程的 adj！
                boolean success = updateOomAdjLocked(cpr.proc);
                ... ... ... ...
```
provider 所在进程的 adj！

关于 ContentProvider，我会单独开一个系列来分析


## 2.5 Process 调整
和 Process 相关的调整代码主要是在 ActivityManagerService.java 中:


### 2.5.1 setSystemProcess

ActivityManagerService.setSystemProcess: 创建并设置系统进程，以下是关键代码段：

```java
            synchronized (this) {
                // 创建系统进程对应的 ProcessRecord，并设置其为 persistent 类型的进程！
                ProcessRecord app = newProcessRecordLocked(info, info.processName, false, 0);
                app.persistent = true;
                app.pid = MY_PID;
                // 设置系统进程的 maxAdj 为 SYSTEM_ADJ！
                app.maxAdj = ProcessList.SYSTEM_ADJ;
                app.makeActive(mSystemThread.getApplicationThread(), mProcessStats);
                synchronized (mPidsSelfLocked) {
                    mPidsSelfLocked.put(app.pid, app);
                }
                // 加入到 lru process 列表中！
                updateLruProcessLocked(app, false, null);
                // 更新所有进程的 adj 值！
                updateOomAdjLocked();
            }
```
更新所有进程的 `adj` 值！

### 2.5.2 addAppLocked

ActivityManagerService.addAppLocked: 创建 persistent 进程，以下是关键代码段：

```java
        //【1】当该进程第一次或者重新创建时，会更新所有进程的 oomAdj！
        if (app == null) {
            app = newProcessRecordLocked(info, null, isolated, 0);
            //【1.1】更新 lru 进程列表；
            updateLruProcessLocked(app, false, null);
            //【1.2】更新进程 oomAdj；
            updateOomAdjLocked();
        }
```

更新所有进程的 oomAdj！

### 2.5.3 attachApplicationLocked

ActivityManagerService.attachApplicationLocked: 进程创建后 attach 到 system_server 的过程，以下是关键代码段：

```java
        ... ... ... ...
        if (badApp) {
            app.kill("error during init", true);
            handleAppDiedLocked(app, false, true);
            return false;
        }
        
        //【1】didSomething 表示我们在启动进程后是否执行了启动组件的操作
        // 如果有 didSomething 就为 true，那就不需要在这里更新了，因为组件启动过程会有更新的操作！
        if (!didSomething) {
            updateOomAdjLocked();
        }
```

更新所有进程的 adj 值！

### 2.5.4 trimApplications

ActivityManagerService.trimApplications: 回收应用程序内存，以下是关键代码段：

```java
            // 处理所有要被移除的进程，回收资源！
            for (i = mRemovedProcesses.size() - 1; i >= 0; i --) {
                final ProcessRecord app = mRemovedProcesses.get(i);
                if (app.activities.size() == 0
                        && app.curReceiver == null && app.services.size() == 0) {
                    if (app.pid > 0 && app.pid != MY_PID) {
                        app.kill("empty", false);
                    } else {
                        try {
                            app.thread.scheduleExit();
                        } catch (Exception e) {
                        }
                    }
                    cleanUpApplicationRecordLocked(app, false, true, -1, false /*replacingPid*/);
                    mRemovedProcesses.remove(i);
                    //【1】如果该进程是 persistent，那就要 addAppLocked！
                    if (app.persistent) {
                        addAppLocked(app.info, false, null /* ABI override */);
                    }
                }
            }
            //【2】更新所有进程的 oomAdj！
            updateOomAdjLocked();
```
更新所有进程的 oomAdj！

### 2.5.5 appDiedLocked

ActivityManagerService.appDiedLocked：进程死亡，以下是关键代码段：

```java
       // 表示进程被重启了！
       if (app.pid == pid && app.thread != null &&
                app.thread.asBinder() == thread.asBinder()) {
            boolean doLowMem = app.instrumentationClass == null;
            boolean doOomAdj = doLowMem;
            if (!app.killedByAm) {
                Slog.i(TAG, "Process " + app.processName + " (pid " + pid
                        + ") has died");
                mAllowLowerMemLevel = true;
            } else {
                // Note that we always want to do oom adj to update our state with the
                // new number of procs.
                mAllowLowerMemLevel = false;
                doLowMem = false;
            }
            EventLog.writeEvent(EventLogTags.AM_PROC_DIED, app.userId, app.pid, app.processName);
            if (DEBUG_CLEANUP) Slog.v(TAG_CLEANUP,
                "Dying app: " + app + ", pid: " + pid + ", thread: " + thread.asBinder());
            handleAppDiedLocked(app, false, true);

            if (doOomAdj) {
                //【1】更新所有进程的 oomAdj！
                updateOomAdjLocked();
            }
            if (doLowMem) {
                doLowMemReportIfNeededLocked(app);
            }
        } else if (app.pid != pid) {
        }
```
更新所有进程的 oomAdj！

### 2.5.6 killAllBackgroundProcesses

ActivityManagerService.killAllBackgroundProcesses: 杀死所有后台进程，以下是关键代码段：

```java
            synchronized (this) {
                final ArrayList<ProcessRecord> procs = new ArrayList<>();
                final int NP = mProcessNames.getMap().size();
                for (int ip = 0; ip < NP; ip++) {
                    final SparseArray<ProcessRecord> apps = mProcessNames.getMap().valueAt(ip);
                    final int NA = apps.size();
                    for (int ia = 0; ia < NA; ia++) {
                        final ProcessRecord app = apps.valueAt(ia);
                        //【1】跳过 persistent 进程！
                        if (app.persistent) {
                            continue;
                        }
                        //【2】杀死 removed 为 ture 的进程
                        if (app.removed) {
                            procs.add(app);
                        //【3】杀死 adj >= CACHED_APP_MIN_ADJ 的进程！
                        } else if (app.setAdj >= ProcessList.CACHED_APP_MIN_ADJ) {
                            app.removed = true;
                            procs.add(app);
                        }
                    }
                }
                //【4】杀死上述两种进程！
                final int N = procs.size();
                for (int i = 0; i < N; i++) {
                    removeProcessLocked(procs.get(i), false, true, "kill all background");
                }
                mAllowLowerMemLevel = true;

                //【5】更新了所有进程
                updateOomAdjLocked();
                doLowMemReportIfNeededLocked(null);
            }
```
继续分析！

### 2.5.7 killPackageProcessesLocked

`ActivityManagerService.killPackageProcessesLocked`: 杀掉指定包名相关的进程，以下是关键代码段：

```java
        ... ... ...// 收集需要被杀掉的进程，这里就列出了！
        int N = procs.size();
        for (int i=0; i<N; i++) {
            removeProcessLocked(procs.get(i), callerWillRestart, allowRestart, reason);
        }
        // 调整所有进程的 oomAdj！
        updateOomAdjLocked();
```
调整所有进程的 oomAdj！


### 2.5.8 setProcessForeground

`ActivityManagerService.setProcessForeground`：设置进程为前台进程，下面是关键代码片段：

```java
            // 这里的 isForeground 是 setProcessForeground 的参数，表示是否为前台进程！
            synchronized (mPidsSelfLocked) {
                // 获得 pid 对应的进程！
                ProcessRecord pr = mPidsSelfLocked.get(pid);
                if (pr == null && isForeground) {
                    Slog.w(TAG, "setProcessForeground called on unknown pid: " + pid);
                    return;
                }
                // 获得该进程对应的前台 token，如果不为 null，说明该进程已经是前台进程
                // 那就要先移除旧的 token！
                ForegroundToken oldToken = mForegroundProcesses.get(pid);
                if (oldToken != null) {
                    oldToken.token.unlinkToDeath(oldToken, 0); // 解除绑定死亡仆告对象！
                    mForegroundProcesses.remove(pid);
                    if (pr != null) {
                        pr.forcingToForeground = null;
                    }
                    changed = true;
                }
                
                // 如果 isForeground 为 true，且 token 不为 null，这里的
                if (isForeground && token != null) {
                    ForegroundToken newToken = new ForegroundToken() {
                        @Override
                        public void binderDied() {
                            foregroundTokenDied(this); // 当 binder 对象死亡后，该方法会被调用！
                        }
                    };
                    newToken.pid = pid; // 设置 newToken 的 pid 和 token！
                    newToken.token = token;
                    try {
                        token.linkToDeath(newToken, 0); // 给 token 绑定死亡仆告对象！
                        mForegroundProcesses.put(pid, newToken); // 将该进程加入到 mForegroundProcesses 列表中！
                        pr.forcingToForeground = token;
                        changed = true;
                    } catch (RemoteException e) {
                    }
                }
            }
            if (changed) { // 当我们设置了一个进程为前台进程，我们需要更新所有进程的 oomAdj！
                updateOomAdjLocked();
            }
```
调整所有进程的 `oomAdj`！

首先我说下 `ForegroundToken`，表示一个前台句柄，其内部有一个 `token` 对象，该对象是一个 `Binder` 对象！当我们设置了一个进程为前台进程后，每个进程都会有一个 `token` 对象！

其使用场景是，弹出 `Toast`，当一个进程中会弹出 `Toast` 后，该进程会被设置为前台进程，就会触发 `setProcessForeground` 方法！`token` 对象定义是在 `NotificationManagerService` 中：

```java
final IBinder mForegroundToken = new Binder();
```

再说一下：`ForegroundToken.binderDied` 方法，当 `token` 死亡后，其会被调用，从而触发 `foregroundTokenDied`!

```java
    void foregroundTokenDied(ForegroundToken token) {
        synchronized (ActivityManagerService.this) {
            synchronized (mPidsSelfLocked) {
                ForegroundToken cur
                    = mForegroundProcesses.get(token.pid);
                if (cur != token) {
                    return;
                }
                mForegroundProcesses.remove(token.pid);
                ProcessRecord pr = mPidsSelfLocked.get(token.pid);
                if (pr == null) {
                    return;
                }
                pr.forcingToForeground = null;
                // 更新前台进程信息！
                updateProcessForegroundLocked(pr, false, false);
            }
            // 更新进程的 oomAdj
            updateOomAdjLocked();
        }
    }
```
调整所有进程的 `oomAdj`！


### 2.5.9 setHasTopUi

`ActivityManagerService.setHasTopUi`：设置进程是否具有 `top ui`！

```java
    @Override
    public void setHasTopUi(boolean hasTopUi) throws RemoteException {
        // 检查 permission.INTERNAL_SYSTEM_WINDOW 权限！
        if (checkCallingPermission(permission.INTERNAL_SYSTEM_WINDOW) != PERMISSION_GRANTED) {
            throw new SecurityException(msg);
        }
        final int pid = Binder.getCallingPid();
        final long origId = Binder.clearCallingIdentity();
        try {
            synchronized (this) {
                boolean changed = false;
                // 获得 pid 对应的进程！
                ProcessRecord pr;
                synchronized (mPidsSelfLocked) {
                    pr = mPidsSelfLocked.get(pid);
                    if (pr == null) {
                        Slog.w(TAG, "setHasTopUi called on unknown pid: " + pid);
                        return;
                    }
                    // 判断进程的 hasTopUi 是否发生变化！
                    if (pr.hasTopUi != hasTopUi) {
                        Slog.i(TAG, "Setting hasTopUi=" + hasTopUi + " for pid=" + pid);
                        pr.hasTopUi = hasTopUi;
                        changed = true;
                    }
                }
                // 发生了变化，那就更新该进程的 oomAdj！
                if (changed) {
                    updateOomAdjLocked(pr);
                }
            }
        } finally {
            Binder.restoreCallingIdentity(origId);
        }
    }
```
只更新特定进程的 `oomAdj`！

该方法主要是在状态栏使用，具体的代码在 `StatusBarWindowManager.apply` 方法中，以下是关键代码！

```java
    private void apply(State state) {
        ... ... ...
        if (mHasTopUi != mHasTopUiChanged) {
            try {
                mActivityManager.setHasTopUi(mHasTopUiChanged);
            } catch (RemoteException e) {
                Log.e(TAG, "Failed to call setHasTopUi", e);
            }
            mHasTopUi = mHasTopUiChanged;
        }
    }
```

### 2.5.10 updateSleepIfNeededLocked

`ActivityManagerService.updateSleepIfNeededLocked`：更新睡眠状态！

```java
    void updateSleepIfNeededLocked() {
        if (mSleeping && !shouldSleepLocked()) { // 退出休眠时！
            mSleeping = false;
            startTimeTrackingFocusedActivityLocked();
            mTopProcessState = ActivityManager.PROCESS_STATE_TOP;
            mStackSupervisor.comeOutOfSleepIfNeededLocked();
            // 更新服务的 oomAdj！
            updateOomAdjLocked();

        } else if (!mSleeping && shouldSleepLocked()) { // 进入休眠时！
            mSleeping = true;
            if (mCurAppTimeTracker != null) {
                mCurAppTimeTracker.stop();
            }
            mTopProcessState = ActivityManager.PROCESS_STATE_TOP_SLEEPING;
            mStackSupervisor.goingToSleepLocked();

            // 更新服务的 oomAdj！
            updateOomAdjLocked();

            // Initialize the wake times of all processes.
            checkExcessivePowerUsageLocked(false);
            mHandler.removeMessages(CHECK_EXCESSIVE_WAKE_LOCKS_MSG);
            Message nmsg = mHandler.obtainMessage(CHECK_EXCESSIVE_WAKE_LOCKS_MSG);
            mHandler.sendMessageDelayed(nmsg, POWER_CHECK_DELAY);
        }
    }
```
更新所有服务的 `oomAdj`！


## 2.6 总结

下面我用思维导图总结下，`updateOomAdjLocked` 的调用时机，当然了，随着版本的更替，代码逻辑和调用机制势必会发生变化，但是，**已有的架构和逻辑依然具有很强的参考性**，也能帮助我们在新的版本上快速建立代价整体结构的认识！！



# 3 oomAdj 算法分析

接下来，我们就要分析下 `oomAdj` 调度的核心算法了，我们从 `updateOomAdjLocked` 方法入手，我们先从最简单的一参函数看起：

## 3.1 [1]ActivityManagerS.updateOomAdjLocked - 更新指定进程 oomAdj

```java
    final boolean updateOomAdjLocked(ProcessRecord app) {
        // 获得 top activity 以及其所在的进程 top process！
        final ActivityRecord TOP_ACT = resumedAppLocked();
        final ProcessRecord TOP_APP = TOP_ACT != null ? TOP_ACT.app : null;

        // 该进程是否是缓存进程！
        final boolean wasCached = app.cached;

        mAdjSeq++; // adj 序列计数加 1

        //【1】计算 cahce 状态的 adj
        // 如果我们知道，我们的目标进程是处于缓存状态，那么我们返回 app.curRawAdj，否则
        // 我们不能确定其 adj 的值，设置其为 UNKNOWN_ADJ
        final int cachedAdj = app.curRawAdj >= ProcessList.CACHED_APP_MIN_ADJ
                ? app.curRawAdj : ProcessList.UNKNOWN_ADJ;
        
        //【2】更新指定进程的 oomAdj 的值，调用 5 个参数的 updateOomAdjLocked 方法，具体见[3.2]；       
        boolean success = updateOomAdjLocked(app, cachedAdj, TOP_APP, false,
                SystemClock.uptimeMillis());

        //【3】如果进程的从非缓存状态变为了缓存状态，或者相反，我们也要更新 lru 列表中其他进程的 adj！
        if (wasCached != app.cached || app.curRawAdj == ProcessList.UNKNOWN_ADJ) {
            updateOomAdjLocked();
        }

        return success;
    }
```
更新指定进程的 `oomAdj`，被更新的进程的 `app.adjSeq` 值会等于 `mAdjSeq`，这个我们后续再看！

对于 `cachedAdj` 的取值范围：

```java
    static final int UNKNOWN_ADJ = 1001;
    static final int CACHED_APP_MIN_ADJ = 900;
```
该方法主要功能：

- 执行五参 `updateOomAdjLocked`，更新指定进程的 `adj`；
- 当 `app` 经过更新 `adj` 操作后，其 `cached` 状态改变，或者 `curRawAdj=UNKNOWN_ADJ`，则执行空参 `updateOomAdjLocked`，更新所有进程的`adj`；


继续来看：

### 3.1.1 ActivityManagerS.resumedAppLocked

```java
    private final ActivityRecord resumedAppLocked() {
        // 获得当前的 top activity！
        ActivityRecord act = mStackSupervisor.resumedAppLocked();
        String pkg;
        int uid;

        //　获得 top activity 的 uid 和 package！
        if (act != null) {
            pkg = act.packageName;
            uid = act.info.applicationInfo.uid;
        } else {
            pkg = null;
            uid = -1;
        }

        // Has the UID or resumed package name changed?
        // 更新一下　mCurResumedUid 和 mCurResumedPackage
        if (uid != mCurResumedUid || (pkg != mCurResumedPackage
                && (pkg == null || !pkg.equals(mCurResumedPackage)))) {
            if (mCurResumedPackage != null) {
                mBatteryStatsService.noteEvent(BatteryStats.HistoryItem.EVENT_TOP_FINISH,
                        mCurResumedPackage, mCurResumedUid);
            }
            mCurResumedPackage = pkg;
            mCurResumedUid = uid;
            
            // 当前的 current resume pacakge 发生了变化，通知 BatteryStats
            if (mCurResumedPackage != null) {
                mBatteryStatsService.noteEvent(BatteryStats.HistoryItem.EVENT_TOP_START,
                        mCurResumedPackage, mCurResumedUid);
            }
        }
        return act;
    }
```


#### 3.1.1.1 ActivityStackSupervisor.resumedAppLocked

```java
    ActivityRecord resumedAppLocked() {
        //【1】获得焦点 stack!
        ActivityStack stack = mFocusedStack;
        if (stack == null) {
            return null;
        }
        //【2】获得 resume activity！
        ActivityRecord resumedActivity = stack.mResumedActivity;
        if (resumedActivity == null || resumedActivity.app == null) {
            resumedActivity = stack.mPausingActivity;
            if (resumedActivity == null || resumedActivity.app == null) {
                resumedActivity = stack.topRunningActivityLocked();
            }
        }
        return resumedActivity;
    }
```
这里我们可以看到 `resumedActivity` 的获取逻辑！

- 首先会焦点 `stack` 获取 `mResumedActivity`;
- 如果 `resumedActivity` 不存在，那就获取 `mPausingActivity`;
- 如果 `mPausingActivity` 不存在，那就获取 `top activity`！


## 3.3 [0]ActivityManagerS.updateOomAdjLocked - 更新 LRU 列表中的所有进程

刚才我们看了 `updateOomAdjLocked` 的一参和多参数方法，用来更新指定的进程的 `adj`，到那时如果该进程的状态在 `cache` 和非 `cache` 之前切换了，那就会更新所有进程的 `adj` 状态！

这个是最核心的一个方法，方法体很长，我们耐心点来看！！

```java
    final void updateOomAdjLocked() {

        //【1】获取 top Activity 以及其所在的 top process!
        final ActivityRecord TOP_ACT = resumedAppLocked();
        final ProcessRecord TOP_APP = TOP_ACT != null ? TOP_ACT.app : null;
        
        final long now = SystemClock.uptimeMillis();
        final long nowElapsed = SystemClock.elapsedRealtime();
        // MAX_EMPTY_TIME 表示空进程的存活时长，为 30min！
        final long oldTime = now - ProcessList.MAX_EMPTY_TIME;
        
        //【2】获得 LRU 进程列表的大小！
        final int N = mLruProcesses.size();

        if (false) {
            RuntimeException e = new RuntimeException();
            e.fillInStackTrace();
            Slog.i(TAG, "updateOomAdj: top=" + TOP_ACT, e);
        }

        //【3】重置当前所有 active uid 内部记录的进程状态：ActivityManager.PROCESS_STATE_CACHED_EMPTY：16
        for (int i=mActiveUids.size()-1; i>=0; i--) {
            final UidRecord uidRec = mActiveUids.valueAt(i);
            if (false && DEBUG_UID_OBSERVERS) Slog.i(TAG_UID_OBSERVERS,
                    "Starting update of " + uidRec);
            uidRec.reset();
        }

        //【4】给所有的 task 排序，更新 task 的 mLayerRank 变量！
        mStackSupervisor.rankTaskLayersIfNeeded();

        mAdjSeq++; // 更新本次计算序列号 +1

        mNewNumServiceProcs = 0; // 表示最新的服务进程数（更新 adj 后）；
        mNewNumAServiceProcs = 0; // 表示最新的 A service 进程数（更新 adj 后）；

        //【5】根据系统空进程和缓存进程的总限制量，计算空进程和缓存进程限制数！！
        final int emptyProcessLimit;
        final int cachedProcessLimit;
        if (mProcessLimit <= 0) {
            emptyProcessLimit = cachedProcessLimit = 0;
            
        } else if (mProcessLimit == 1) {
            emptyProcessLimit = 1;
            cachedProcessLimit = 0;
            
        } else {
            // 调用 ProcessList.computeEmptyProcessLimit 计算空进程限制数，计算方法为：totalProcessLimit(参数) / 2
            emptyProcessLimit = ProcessList.computeEmptyProcessLimit(mProcessLimit);

            // 可以得到：
            // 缓存进程限制数 = 空进程限制数 = 总进程限制数 / 2；
            cachedProcessLimit = mProcessLimit - emptyProcessLimit;
            
        }

        // CACHED_APP_MAX_ADJ 为 906，而 CACHED_APP_MIN_ADJ 为 900，所以计算出的 numSlots 为 3
        // 就是说，我们将 cache adj 划分为 3 个范围，每个范围可以容纳定量的 empty 和 cache 进程！
        int numSlots = (ProcessList.CACHED_APP_MAX_ADJ
                - ProcessList.CACHED_APP_MIN_ADJ + 1) / 2;
        
        // 计算当前空进程的数量：等于所有进程数 N - 非 cache 进程数 - 被隐藏的 cache 进程数！ 
        int numEmptyProcs = N - mNumNonCachedProcs - mNumCachedHiddenProcs;
        if (numEmptyProcs > cachedProcessLimit) {

            // 空进程的数量不能超过缓存进程限制数，最大只能和其相等，这是一定的！
            numEmptyProcs = cachedProcessLimit;
        }
        
        // 计算空进程分配因子（每个 slot 可以容纳的空进程数量），最高为 1；
        int emptyFactor = numEmptyProcs / numSlots;
        if (emptyFactor < 1) emptyFactor = 1;
        
        // 计算缓存进程分配因子（计算出每个 slot 中，可容纳后台进程的数量），最高为 1；
        int cachedFactor = (mNumCachedHiddenProcs > 0 ? mNumCachedHiddenProcs : 1) / numSlots;
        if (cachedFactor < 1) cachedFactor = 1;

        int stepCached = 0;
        int stepEmpty = 0;

        int numCached = 0;
        int numEmpty = 0;
        int numTrimming = 0;

        mNumNonCachedProcs = 0; // 用于记录非 cache/empty 进程数，初始化为 0；
        mNumCachedHiddenProcs = 0; // 用于记录 cache 进程数，初四化为 0；

        // 用于为 cache 进程和 empty 进程分配 adj！
        int curCachedAdj = ProcessList.CACHED_APP_MIN_ADJ; // 900；
        int nextCachedAdj = curCachedAdj + 1;

        int curEmptyAdj = ProcessList.CACHED_APP_MIN_ADJ; // 900；
        int nextEmptyAdj = curEmptyAdj + 2;

        //【8】逆序遍历 mLruProcesses 进程列表，更新每一个进程的 adj！
        for (int i=N-1; i>=0; i--) {
            ProcessRecord app = mLruProcesses.get(i);
            if (!app.killedByAm && app.thread != null) {
                app.procStateChanged = false;

                //【8.1】这里调用了 5 个参数的 computeOomAdjLocked 方法，更新指定的进程的 adj
                // 注意这里的 doingAll 参数传入的是 true，具体分析见 3.3.1 节!
                computeOomAdjLocked(app, ProcessList.UNKNOWN_ADJ, TOP_APP, true, now);

                // 如果计算后，仍然没有分配给进程一个合适的 adj，即：app.curAdj >= ProcessList.UNKNOWN_ADJ
                // 那就在这里分配！
                // （computeOomAdjLocked 可能无法计算 cache 进程和 empty 进程的 adj，所以在下面处理！）
                if (app.curAdj >= ProcessList.UNKNOWN_ADJ) {

                    // 根据进程的状态进行处理：
                    switch (app.curProcState) {
                        case ActivityManager.PROCESS_STATE_CACHED_ACTIVITY:
                        case ActivityManager.PROCESS_STATE_CACHED_ACTIVITY_CLIENT:

                            // 如果进程是持有 activity 的 cache 进程，或者
                            // 给其分配一个 cache adj！
                            // 从 900 到 906，逐个分配，最大不超过 906！
                            app.curRawAdj = curCachedAdj;
                            app.curAdj = app.modifyRawOomAdj(curCachedAdj); // 此时是单纯的加 1；

                            if (DEBUG_LRU && false) Slog.d(TAG_LRU, "Assigning activity LRU #" + i
                                    + " adj: " + app.curAdj + " (curCachedAdj=" + curCachedAdj
                                    + ")");

                            if (curCachedAdj != nextCachedAdj) {
                                stepCached++;

                                if (stepCached >= cachedFactor) { // 判断是否到达分配因子！
                                    stepCached = 0; // stepCached 置为 0，进行下一个 slot 的分配！
                                    curCachedAdj = nextCachedAdj;
                                    nextCachedAdj += 2;

                                    if (nextCachedAdj > ProcessList.CACHED_APP_MAX_ADJ) { // 控制 adj 的不超过最大值！
                                        nextCachedAdj = ProcessList.CACHED_APP_MAX_ADJ;
                                    }
                                }
                            }
                            break;

                        default:
                            // 否则，将其视为 empty 进程对待，给其分配一个 empty adj！
                            // 从 900 到 906，逐个分配，最大不超过 906！
                            app.curRawAdj = curEmptyAdj;
                            app.curAdj = app.modifyRawOomAdj(curEmptyAdj);

                            if (DEBUG_LRU && false) Slog.d(TAG_LRU, "Assigning empty LRU #" + i
                                    + " adj: " + app.curAdj + " (curEmptyAdj=" + curEmptyAdj
                                    + ")");

                            if (curEmptyAdj != nextEmptyAdj) {
                                stepEmpty++;
                                if (stepEmpty >= emptyFactor) {
                                    stepEmpty = 0;
                                    curEmptyAdj = nextEmptyAdj;
                                    nextEmptyAdj += 2;
                                    if (nextEmptyAdj > ProcessList.CACHED_APP_MAX_ADJ) {
                                        nextEmptyAdj = ProcessList.CACHED_APP_MAX_ADJ;
                                    }
                                }
                            }
                            break;
                    }
                }
                
                //【8.2】分配好后，调用 applyOomAdjLocked 更新进程的 adj，具体分析见 3.3.2 节!
                applyOomAdjLocked(app, true, now, nowElapsed);

                //【8.3】根据进程的类型，统计不同类型的进程数量!
                switch (app.curProcState) {
                    case ActivityManager.PROCESS_STATE_CACHED_ACTIVITY:
                    case ActivityManager.PROCESS_STATE_CACHED_ACTIVITY_CLIENT:

                        // 如果进程是持有 activity 的 cache 进程或者是客户端进程，
                        // 属于 cache 进程，mNumCachedHiddenProcs 加 1!
                        mNumCachedHiddenProcs++;
                        numCached++;
                        
                        // 注意，如果当前 cache 进程数量超过了限制量，那就杀掉超过限制的所有进程！
                        // 但是依然统计数量！
                        if (numCached > cachedProcessLimit) {
                            app.kill("cached #" + numCached, true);
                        }
                        break;

                    case ActivityManager.PROCESS_STATE_CACHED_EMPTY:
                
                        // 如果进程是 empty cache 进程，下面的处理！
                        if (numEmpty > ProcessList.TRIM_EMPTY_APPS
                                && app.lastActivityTime < oldTime) {
                            // 如果空进程的数量超过了空进程上限数，且该空进程的空闲时间超过了 30 min
                            // 那就杀掉该进程！
                            app.kill("empty for "
                                    + ((oldTime + ProcessList.MAX_EMPTY_TIME - app.lastActivityTime)
                                    / 1000) + "s", true);
                        } else {
                            // 否则，numEmpty 加 1，如果此时空进程数超过了空进程限制数，杀掉该进程！
                            // 但是依然统计数量！
                            numEmpty++;
                            if (numEmpty > emptyProcessLimit) {
                                app.kill("empty #" + numEmpty, true);
                            }
                        }
                        break;

                    default:
                        // 其他类型，默认属于非 cache/empty 进程，mNumNonCachedProcs 加 1；
                        mNumNonCachedProcs++;
                        break;
                }

                //【8.4】进一步处理进程的状态！
                if (app.isolated && app.services.size() <= 0) {

                    // 如果这是一个隔离进程，且内部不再运行任何服务，kill 该进程！
                    app.kill("isolated not needed", true);
                } else {
                
                    // 否则，保留该进程，更新其对应的 uid 中记录的进程状态！
                    final UidRecord uidRec = app.uidRecord;
                    if (uidRec != null && uidRec.curProcState > app.curProcState) {
                        uidRec.curProcState = app.curProcState;
                    }
                }

                //【8.5】进一步处理进程的状态！
                if (app.curProcState >= ActivityManager.PROCESS_STATE_HOME
                        && !app.killedByAm) {

                    // 记录需要回收内存的，进程状态优先级不高于 PROCESS_STATE_HOME 的后台进程数量！
                    // PROCESS_STATE_HOME 表示持有 Home 的后台进程的状态！
                    numTrimming++;
                }
            }
        }

        //【9】mNewNumServiceProcs 用于保存更新 adj 后，最新的服务进程数，更新 mNumServiceProcs！
        mNumServiceProcs = mNewNumServiceProcs;

        //【10】计算统计到的 cache 和 empty 进程总数！
        final int numCachedAndEmpty = numCached + numEmpty;
        
        //【11】根据统计的 cache 和 empty 进程总数，计算内存回收等级，用于内存回收！
        int memFactor;
        
        // TRIM_CACHED_APPS 表示触发内存回收的 cache 进程上限： 5；
        // TRIM_EMPTY_APPS 表示触发内存回收的 empty 进程上限：16；
        if (numCached <= ProcessList.TRIM_CACHED_APPS
                && numEmpty <= ProcessList.TRIM_EMPTY_APPS) {
        
            if (numCachedAndEmpty <= ProcessList.TRIM_CRITICAL_THRESHOLD) { // 个数为 3；
                // 总数小于 3 时，内存回收等级为 critical，取值为 3！
                memFactor = ProcessStats.ADJ_MEM_FACTOR_CRITICAL;
                
            } else if (numCachedAndEmpty <= ProcessList.TRIM_LOW_THRESHOLD) { // 个数为 5；
                // 总数小于 3 时，内存回收等级为 low，取值为 2；
                memFactor = ProcessStats.ADJ_MEM_FACTOR_LOW; 
                
            } else {
                // 否则，内存回收等级为 moderate，取值为 1；
                memFactor = ProcessStats.ADJ_MEM_FACTOR_MODERATE;
            }
        } else {
            // 取值为 0，cachce 和 empty 进程足够时，内存回收等级为 normal
            memFactor = ProcessStats.ADJ_MEM_FACTOR_NORMAL; 
        }

        if (DEBUG_OOM_ADJ) Slog.d(TAG_OOM_ADJ, "oom: memFactor=" + memFactor
                + " last=" + mLastMemoryLevel + " allowLow=" + mAllowLowerMemLevel
                + " numProcs=" + mLruProcesses.size() + " last=" + mLastNumProcesses);
    
        // 如果新的内存回收等级比上一次的大，判读是否使用新的内存回收等级！
        // 一般情况下，内存回收等级变高时（即允许尽可能多地回收），是不允许降级的
        // 但 mAllowLowerMemLevel  为false，或进程数量变多时，可以降级！
        if (memFactor > mLastMemoryLevel) {
            if (!mAllowLowerMemLevel || mLruProcesses.size() >= mLastNumProcesses) {

                // 使用上一次的低内存级别！
                memFactor = mLastMemoryLevel;
                if (DEBUG_OOM_ADJ) Slog.d(TAG_OOM_ADJ, "Keeping last mem factor!");
            }
        }
        if (memFactor != mLastMemoryLevel) {
            EventLogTags.writeAmMemFactor(memFactor, mLastMemoryLevel);
        }
        
        // 更新内存级别和进程数量！
        mLastMemoryLevel = memFactor;
        mLastNumProcesses = mLruProcesses.size();
        
        // 将最新的内存回收等级保存到 ProcessStats 中，如果和上一次的不同，allChanged 为 true！
        boolean allChanged = mProcessStats.setMemFactorLocked(memFactor, !isSleepingLocked(), now);
        final int trackerMemFactor = mProcessStats.getMemFactorLocked();
        
        // 如果最新的内存回收等级不等于 normal，那么所有进程都要进行内存回收工作！
        if (memFactor != ProcessStats.ADJ_MEM_FACTOR_NORMAL) {
            if (mLowRamStartTime == 0) { // 记录内存回收时间
                mLowRamStartTime = now;
            }

            int step = 0;
            int fgTrimLevel;

            // 计算前台进程的内存回收等级；
            switch (memFactor) {
                case ProcessStats.ADJ_MEM_FACTOR_CRITICAL:
                    fgTrimLevel = ComponentCallbacks2.TRIM_MEMORY_RUNNING_CRITICAL; // critical 级别；
                    break;
                case ProcessStats.ADJ_MEM_FACTOR_LOW:
                    fgTrimLevel = ComponentCallbacks2.TRIM_MEMORY_RUNNING_LOW; // low 级别；
                    break;
                default:
                    fgTrimLevel = ComponentCallbacks2.TRIM_MEMORY_RUNNING_MODERATE; // moderate 级别；
                    break;
            }

            // 计算回收因子，numTrimming 表示所有需要回收内存的后台进程！
            // 其实就是分阶段的回收！
            int factor = numTrimming / 3;
            int minFactor = 2;

            // 如果 home 进程不会 null，previous 进程不为 null，最小因子均加 1；
            if (mHomeProcess != null) minFactor++;
            if (mPreviousProcess != null) minFactor++;
            
            // 回收因子不能低于最小因子！
            if (factor < minFactor) factor = minFactor;

            // 初始默认的回收级别为 complete！
            int curLevel = ComponentCallbacks2.TRIM_MEMORY_COMPLETE;
            
            // 再次逆序遍历 mLruProcesses 列表，根据进程的状态进行不同的回收处理！
            for (int i=N-1; i>=0; i--) {
            
                // 开始处理进程！
                ProcessRecord app = mLruProcesses.get(i);
                
                // 如果内存回收等级变化了，或者进程的状态发生变化，更新进程的监控信息！
                if (allChanged || app.procStateChanged) {
                    setProcessTrackerStateLocked(app, trackerMemFactor, now);
                    app.procStateChanged = false;
                }

                if (app.curProcState >= ActivityManager.PROCESS_STATE_HOME
                        && !app.killedByAm) {
                // 如果进程的状态优先级不高于 PROCESS_STATE_HOME：12，并且没有被 AMS 后台杀死，进入该分支！

                    if (app.trimMemoryLevel < curLevel && app.thread != null) {
                        // 如果进程的内存回收级别低于 curLevel，并且进程还在运行，执行内存回收！
                        try {
                            if (DEBUG_SWITCH || DEBUG_OOM_ADJ) Slog.v(TAG_OOM_ADJ,
                                    "Trimming memory of " + app.processName + " to " + curLevel);
                            // 回收内存！
                            app.thread.scheduleTrimMemory(curLevel);
                        } catch (RemoteException e) {
                        }

                        if (false) { // 这里默认是不进入的！
                            if (curLevel >= ComponentCallbacks2.TRIM_MEMORY_COMPLETE
                                    && app != mHomeProcess && app != mPreviousProcess) {
                                mStackSupervisor.scheduleDestroyAllActivities(app, "trim");
                            }
                        }
                    }
                    
                    // 更新进程的内存回收级别！
                    app.trimMemoryLevel = curLevel;
                    
                    // 记录处理的进度，当到达回收因子的时候，step 归 0，准备进入下一个回收因子阶段！
                    step++;
                    if (step >= factor) {
                        step = 0;
                        switch (curLevel) {
                            case ComponentCallbacks2.TRIM_MEMORY_COMPLETE: 
                                // 如果是 COMPLETE:80，设置 curLevel 为 MODERATE:60
                                curLevel = ComponentCallbacks2.TRIM_MEMORY_MODERATE;
                                break;
                                
                            case ComponentCallbacks2.TRIM_MEMORY_MODERATE:
                                // 如果是 COMPLETE:60，设置 curLevel 为 BACKGROUND:40
                                curLevel = ComponentCallbacks2.TRIM_MEMORY_BACKGROUND;
                                break;
                                
                        }
                    }
                    
                } else if (app.curProcState == ActivityManager.PROCESS_STATE_HEAVY_WEIGHT) {
                // 如果进程的状态属于 height weight 类型（9）的进程，进入该分支！
                
                    if (app.trimMemoryLevel < ComponentCallbacks2.TRIM_MEMORY_BACKGROUND
                            && app.thread != null) {
                    // 如果进程的内存回收级别低于 BACKGROUND:40，并且进程还在运行，执行内存回收！
                        try {
                            if (DEBUG_SWITCH || DEBUG_OOM_ADJ) Slog.v(TAG_OOM_ADJ,
                                    "Trimming memory of heavy-weight " + app.processName
                                    + " to " + ComponentCallbacks2.TRIM_MEMORY_BACKGROUND);
                            // 回收内存！
                            app.thread.scheduleTrimMemory(
                                    ComponentCallbacks2.TRIM_MEMORY_BACKGROUND);
                        } catch (RemoteException e) {
                        }
                    }
                    // 更新进程的内存回收登记为 BACKGROUND:40
                    app.trimMemoryLevel = ComponentCallbacks2.TRIM_MEMORY_BACKGROUND;
                    
                } else {
                // 其他情况，进入这里：
                
                    if ((app.curProcState >= ActivityManager.PROCESS_STATE_IMPORTANT_BACKGROUND
                            || app.systemNoUi) && app.pendingUiClean) {
                    // 如果该进程的进程状态优先级低于 important background: 7，或者其是 system 进程但是不显示 ui
                    // 且 pendingUiClean 为 true，这种进程的回收登记很特殊，为 TRIM_MEMORY_UI_HIDDEN: 20
                    
                        final int level = ComponentCallbacks2.TRIM_MEMORY_UI_HIDDEN;
                        if (app.trimMemoryLevel < level && app.thread != null) {
                            // 如果进程的内存回收级别低于 level，并且进程还在运行，执行内存回收！
                            try {
                                if (DEBUG_SWITCH || DEBUG_OOM_ADJ) Slog.v(TAG_OOM_ADJ,
                                        "Trimming memory of bg-ui " + app.processName
                                        + " to " + level);
                                // 回收内存！
                                app.thread.scheduleTrimMemory(level);
                            } catch (RemoteException e) {
                            }
                        }
                        // 设置 pendingUiClean 为 false，表示 ui 资源已经被回收了
                        app.pendingUiClean = false;
                    }
                    
                    // 如果 TRIM_MEMORY_UI_HIDDEN 等级不够无法回收，则以前台进程的回收级别再次回收！！
                    if (app.trimMemoryLevel < fgTrimLevel && app.thread != null) {
                        try {
                            if (DEBUG_SWITCH || DEBUG_OOM_ADJ) Slog.v(TAG_OOM_ADJ,
                                    "Trimming memory of fg " + app.processName
                                    + " to " + fgTrimLevel);
                            // 回收内存！
                            app.thread.scheduleTrimMemory(fgTrimLevel);
                        } catch (RemoteException e) {
                        }
                    }
                    // 更新进程的内存回收登记！
                    app.trimMemoryLevel = fgTrimLevel;
                }
            }

        } else { // 如果最新的内存回收等级等于 normal，那么进入该分支！
            if (mLowRamStartTime != 0) {
                mLowRamTimeSinceLastIdle += now - mLowRamStartTime;
                mLowRamStartTime = 0;
            }
            
            // 逆序遍历 mLruProcesses！
            for (int i=N-1; i>=0; i--) {
                ProcessRecord app = mLruProcesses.get(i);
                
                // 同样的，更新进程信息！
                if (allChanged || app.procStateChanged) {
                    setProcessTrackerStateLocked(app, trackerMemFactor, now);
                    app.procStateChanged = false;
                }
                
                // 如果该进程的进程状态优先级低于 important background: 6，或者其是 system 进程但是不显示 ui
                // 且 pendingUiClean 为 true，才会回收内存！！
                if ((app.curProcState >= ActivityManager.PROCESS_STATE_IMPORTANT_BACKGROUND
                        || app.systemNoUi) && app.pendingUiClean) {
                        
                    // 回收等级是 TRIM_MEMORY_UI_HIDDEN: 20
                    if (app.trimMemoryLevel < ComponentCallbacks2.TRIM_MEMORY_UI_HIDDEN
                            && app.thread != null) {
                        try {
                            if (DEBUG_SWITCH || DEBUG_OOM_ADJ) Slog.v(TAG_OOM_ADJ,
                                    "Trimming memory of ui hidden " + app.processName
                                    + " to " + ComponentCallbacks2.TRIM_MEMORY_UI_HIDDEN);
                                    
                            // 回收内存！
                            app.thread.scheduleTrimMemory(
                                    ComponentCallbacks2.TRIM_MEMORY_UI_HIDDEN);
                        } catch (RemoteException e) {
                        }
                    }
                    // 设置 pendingUiClean 为 false；
                    app.pendingUiClean = false;
                }
                // 这里很特殊，设置 trimMemoryLevel 为 0；
                app.trimMemoryLevel = 0;
            }
        }

        // 如果允许销毁后台 Activity
        // 可以通过：开发者选项 ——》不保留活动，开启！
        if (mAlwaysFinishActivities) {
            mStackSupervisor.scheduleDestroyAllActivities(null, "always-finish");
        }

        // allChanged 为 true，表示发生了内存回收，这里是请求 PSS 内存！
        if (allChanged) {
            requestPssAllProcsLocked(now, false, mProcessStats.isMemFactorLowered());
        }

        // 更新活跃的 uid 的状态！
        for (int i=mActiveUids.size()-1; i>=0; i--) {
            final UidRecord uidRec = mActiveUids.valueAt(i);
            int uidChange = UidRecord.CHANGE_PROCSTATE;

            // 如果 setProcState 和 curProcState 不相等，说明所属进程状态发生了变化！
            if (uidRec.setProcState != uidRec.curProcState) {
            
                if (DEBUG_UID_OBSERVERS) Slog.i(TAG_UID_OBSERVERS,
                        "Changes in " + uidRec + ": proc state from " + uidRec.setProcState
                        + " to " + uidRec.curProcState);
                
                if (ActivityManager.isProcStateBackground(uidRec.curProcState)) {
                    if (!ActivityManager.isProcStateBackground(uidRec.setProcState)) {
                        // 如果 uid 所属进程当前的状态是后台进程，但是之前的状态不是后台进程！
                        // 更新 uid 的 lastBackgroundTime 变量！

                        uidRec.lastBackgroundTime = nowElapsed;

                        if (!mHandler.hasMessages(IDLE_UIDS_MSG)) {

                            // 发送 uid 处于 idle 状态的消息！
                            mHandler.sendEmptyMessageDelayed(IDLE_UIDS_MSG, BACKGROUND_SETTLE_TIME);
                        }
                    }
                } else {

                    // 如果 uid 所属进程当前的状态是前台进程，那么该 uid 处于 active 状态！
                    if (uidRec.idle) {
                        uidChange = UidRecord.CHANGE_ACTIVE;
                        uidRec.idle = false;
                    }
                    uidRec.lastBackgroundTime = 0;
                }
                
                // 更新 setProcState 为 curProcState！
                uidRec.setProcState = uidRec.curProcState;
                enqueueUidChangeLocked(uidRec, -1, uidChange);
                noteUidProcessState(uidRec.uid, uidRec.curProcState);
            }
        }

        // 更新进程的状态！
        if (mProcessStats.shouldWriteNowLocked(now)) {
            mHandler.post(new Runnable() {
                @Override public void run() {
                    synchronized (ActivityManagerService.this) {
                        mProcessStats.writeStateAsyncLocked();
                    }
                }
            });
        }

        if (DEBUG_OOM_ADJ) { // debug 相关，不处理！
            final long duration = SystemClock.uptimeMillis() - now;
            if (false) {
                Slog.d(TAG_OOM_ADJ, "Did OOM ADJ in " + duration + "ms",
                        new RuntimeException("here").fillInStackTrace());
            } else {
                Slog.d(TAG_OOM_ADJ, "Did OOM ADJ in " + duration + "ms");
            }
        }
    }

```

### 3.2.1 变量总结

下面我们来总结，空参 `updateOomAdjLocked` 中遇到的一些变量：




### 3.2.2 过程总结


## 3.3 [5]ActivityManagerS.updateOomAdjLocked

最后，我们来看 `5` 参的 `updateOomAdjLocked` 方法更新指定进程的 `oomAdj`，参数传递：

- **ProcessRecord app**：要更新 `oomAdj` 的进程；
- **int cachedAdj**：进程处于 `cache` 状态的 `adj` 值；
- **ProcessRecord TOP_APP**：`top activity` 所在进程；
- **boolean doingAll**：是否对所有进程都更新 `adj` 的值！
- **long now**：更新的时间点！

```java
    private final boolean updateOomAdjLocked(ProcessRecord app, int cachedAdj,
            ProcessRecord TOP_APP, boolean doingAll, long now) {
        if (app.thread == null) { // 进程没有启动！
            return false;
        }
        
        //【1】计算 oomAdj 的值，doingAll 表示是否是对所有的进程都更新
        // 如果是的话，那就会统计进程数量！
        computeOomAdjLocked(app, cachedAdj, TOP_APP, doingAll, now);

        //【2】应用 oomAdj 的值；
        return applyOomAdjLocked(app, doingAll, now, SystemClock.elapsedRealtime());
    }
```
我们看到，更新指定进程的 `oomAdj` 的会先调用 `computeOomAdjLocked` 计算 `oomAdj`，在调用 `applyOomAdjLocked` 应用计算的  `oomAdj`！！

### 3.3.1 ActivityManagerS.computeOomAdjLocked

该方法用于计算除了 `cachedProcess` 和 `emptyProcess` 进程以外的进程的 `oom_adj` 值！

该方法的判断分支很多，我们按照模块划分下：


#### 3.3.1.1 已经更新/进程未启动的情况

```java
    private final int computeOomAdjLocked(ProcessRecord app, int cachedAdj, ProcessRecord TOP_APP,
            boolean doingAll, long now) {
        //【1】如果该进程的 app.adjSeq 和 mAdjSeq 相等，说明已经计算过了，直接返回计算的结果！
        if (mAdjSeq == app.adjSeq) {
            return app.curRawAdj;
        }

        //【2】如果进程的 thread 为 null，说明进程没有启动，
        // 设置进程的调度组为 SCHED_GROUP_BACKGROUND，进程状态为 SCHED_GROUP_BACKGROUND
        // curAdj 和 curRawAdj 均为 CACHED_APP_MAX_ADJ；                                              
        if (app.thread == null) {
            app.adjSeq = mAdjSeq; 
            app.curSchedGroup = ProcessList.SCHED_GROUP_BACKGROUND;
            app.curProcState = ActivityManager.PROCESS_STATE_CACHED_EMPTY;
            return (app.curAdj=app.curRawAdj=ProcessList.CACHED_APP_MAX_ADJ);
        }
        
        ... ... ... ... // 见下面[3.1.2.1.2]！
```
- 对于已经更新过的进程，通过 `mAdjSeq == app.adjSeq` 进行判断，当系统每次调用 `updateOomAdjLocked` 方法更新系统中进程的 `adj` 的时候，`mAdjSeq` 的值就会加一，同时，本次更新过的进程的 `adjSeq` 也会等于 `mAdjSeq`！

- 对于**进程还没有启动的情况**，我们默认设置初始的值给对应的属性！

  - `app.curSchedGroup = ProcessList.SCHED_GROUP_BACKGROUND` : 
  - `app.curProcState = ActivityManager.PROCESS_STATE_CACHED_EMPTY` : `16`
  - `app.curAdj = app.curRawAdj = ProcessList.CACHED_APP_MAX_ADJ` : `906`


#### 3.3.1.2 系统进程

下面我们来看看系统进程的处理！

```java
        ... ... ... ... // 见上面[3.1.2.1.1]！

        // 初始化一些 debug 的变量，后续会使用！
        app.adjTypeCode = ActivityManager.RunningAppProcessInfo.REASON_UNKNOWN;
        app.adjSource = null;
        app.adjTarget = null;
        
        app.empty = false; // 进程是否会为空；
        app.cached = false; // 进程是否被缓存；

        final int activitiesSize = app.activities.size(); // 该进程中的 activity 数！

        //【3】如果进程的 maxAdj <= FOREGROUND_APP_ADJ(0)，说明该进程是系统进程，那么一定是在前台！
        if (app.maxAdj <= ProcessList.FOREGROUND_APP_ADJ) {
            app.adjType = "fixed";
            app.adjSeq = mAdjSeq;
            app.curRawAdj = app.maxAdj;
            app.foregroundActivities = false;

            // 设置进程的调度组为 SCHED_GROUP_DEFAULT；
            app.curSchedGroup = ProcessList.SCHED_GROUP_DEFAULT;

            // 设置进程的状态为 PROCESS_STATE_PERSISTENT；
            app.curProcState = ActivityManager.PROCESS_STATE_PERSISTENT;

            // 判断系统进程当前是否正在显示 ui，如果正在显示，那么我们需要等到用户离开 ui 后再回收内存！
            app.systemNoUi = true;

            if (app == TOP_APP) { // 如果该系统进程是 top activity 所在进程，systemNoUi 为 false；
                app.systemNoUi = false;
                app.curSchedGroup = ProcessList.SCHED_GROUP_TOP_APP;
                app.adjType = "pers-top-activity";

            } else if (app.hasTopUi) { // 如果该系统进程 hasTopUi 为 true，说明其持有 top-level ui，systemNoUi 为 false；
                app.systemNoUi = false;
                app.curSchedGroup = ProcessList.SCHED_GROUP_TOP_APP;
                app.adjType = "pers-top-ui";

            } else if (activitiesSize > 0) { // 如果系统进程中的 activity 数目大于 0
                for (int j = 0; j < activitiesSize; j++) {
                    final ActivityRecord r = app.activities.get(j);
                    if (r.visible) { // 如果有可见的 activity，那么 systemNoUi 为 false；
                        app.systemNoUi = false;
                    }
                }
            }

            // 如果系统进程持有 ui 界面，那其进程状态为 PROCESS_STATE_PERSISTENT_UI；
            if (!app.systemNoUi) {
                app.curProcState = ActivityManager.PROCESS_STATE_PERSISTENT_UI;
            }

            // 那么其 curAdj 为 maxAdj 的值，这里直接 return！
            return (app.curAdj = app.maxAdj);
        }

        ... ... ... ... // 见下面[3.1.2.1.3]！
```

对于系统进程，其一定是要在前台的，这里首先判断了最大的 `maxAdj` 的取值！`FOREGROUND_APP_ADJ` 的取值为 `0`，表示前台进程的 `adj`！

**`maxAdj` 在进程对象刚创建的时候，会被初始化为 `UNKNOWN_ADJ`**！

```java
        maxAdj = ProcessList.UNKNOWN_ADJ;
        curRawAdj = setRawAdj = ProcessList.INVALID_ADJ;
        curAdj = setAdj = verifiedAdj = ProcessList.INVALID_ADJ;
```
**1、在设置系统进程 `setSystemProcess` 的时候，会被设置为如下值：**
```java
        // 为系统进程创建 ProcessRecord 对象！
        ProcessRecord app = newProcessRecordLocked(info, info.processName, false, 0);
        app.persistent = true; // 表示为常驻进程！
        app.pid = MY_PID;
        app.maxAdj = ProcessList.SYSTEM_ADJ; // 设置 maxAdj！
        app.makeActive(mSystemThread.getApplicationThread(), mProcessStats);
```

**2、在启动 `persistent app` 的时候，会调用 `addAppLocked` 方法，设置为如下值：**
```java
        if ((info.flags & PERSISTENT_MASK) == PERSISTENT_MASK) {
            app.persistent = true;
            app.maxAdj = ProcessList.PERSISTENT_PROC_ADJ;
        }
```
**这两个值对应的 `adj` 优先级都高于 `FOREGROUND_APP_ADJ`，都属于 `persistent adj`，都会进入该分支**！

这一段的逻辑如下：

如果进程的 `maxAdj` 比 `FOREGROUND_APP_ADJ` 小，说明其是常驻进程或者系统进程，那么就做如下判断：

- **`app == TOP_APP`**

当前的 `top activity` 所在进程就是该进程，那么有：

```java
app.systemNoUi = false;
app.curSchedGroup = ProcessList.SCHED_GROUP_TOP_APP;
app.curProcState = ActivityManager.PROCESS_STATE_PERSISTENT_UI;
app.foregroundActivities = false;
```
- **`app.hasTopUi`**

当前进程不是 `top process`，但是显示 `top-level` 级别的 `ui`，那么有：

```java
app.systemNoUi = false;
app.curSchedGroup = ProcessList.SCHED_GROUP_TOP_APP;
app.curProcState = ActivityManager.PROCESS_STATE_PERSISTENT_UI;
app.foregroundActivities = false;
```

- **`activitiesSize > 0`**

如果即不是`top process`，也没有显示`top-level ui`，那就要判断下内部的是否持有可见的`activity`，如果有那么`systemNoUi`为

`false`，`curProcState = ActivityManager.PROCESS_STATE_PERSISTENT_UI`;

```java
app.systemNoUi = false; / true
app.curSchedGroup = ProcessList.SCHED_GROUP_TOP_APP;
app.curProcState = ActivityManager.PROCESS_STATE_PERSISTENT_UI; / ActivityManager.PROCESS_STATE_PERSISTENT;
app.foregroundActivities = false;
```

对于 `persistent` 进程的情况，最后返回的是 `app.curAdj = app.curRawAdj = app.maxAdj`!

#### 3.3.1.3 前台进程

对于其他的进程来说，`maxAdj` 只会在初始化的时候被设置为 `ProcessList.UNKNOWN_ADJ`; 显然，他们是需要进入下面的分支的，**判断其是否属于前台进程**：

```java
        ... ... ... ... // 见上面[3.1.2.1.2]！

        app.systemNoUi = false;

        final int PROCESS_STATE_CUR_TOP = mTopProcessState; // 用于保存当前 top process 的状态！

        //【4】接下来就开始决定非系统前台进程的重要性了，从高级别开始一直到低级别，逐级分配 oomAdj！
        // adj 用来保存计算出的 oomAdj，schedGroup 用来保存计算出的调度组，procState 用来保存计算出的进程状态！
        int adj;
        int schedGroup;
        int procState;
        boolean foregroundActivities = false;
        BroadcastQueue queue;

        if (app == TOP_APP) {

            // 该进程就是 top activity 所在的进程，此时进程处于前台！
            adj = ProcessList.FOREGROUND_APP_ADJ; // 0；
            
            schedGroup = ProcessList.SCHED_GROUP_TOP_APP;
            app.adjType = "top-activity";
            foregroundActivities = true;
            
            procState = PROCESS_STATE_CUR_TOP;
            
        } else if (app.instrumentationClass != null) {

            // 如果进程中有正在运行的 instrumentation，用于测试，此时进程处于前台！
            adj = ProcessList.FOREGROUND_APP_ADJ; // 0
            
            schedGroup = ProcessList.SCHED_GROUP_DEFAULT;
            app.adjType = "instrumentation";
            
            procState = ActivityManager.PROCESS_STATE_FOREGROUND_SERVICE; // 4
            
        } else if ((queue = isReceivingBroadcast(app)) != null) {

            // 如果进程中有正在运行的 BroadcastReceiver 接收处理广播，此时进程处于前台！
            // 然后根据广播所处的队列类型设置调度组！
            adj = ProcessList.FOREGROUND_APP_ADJ; // 0
            
            schedGroup = (queue == mFgBroadcastQueue)
                    ? ProcessList.SCHED_GROUP_DEFAULT : ProcessList.SCHED_GROUP_BACKGROUND;
            app.adjType = "broadcast";
            
            procState = ActivityManager.PROCESS_STATE_RECEIVER; // 11
            
        } else if (app.executingServices.size() > 0) {
        
            // 如果进程中有服务正在执行，此时进程处于前台，然后根据执行操作的前台后台，设置不同调度组；
            adj = ProcessList.FOREGROUND_APP_ADJ; // 0
            
            schedGroup = app.execServicesFg ?
                    ProcessList.SCHED_GROUP_DEFAULT : ProcessList.SCHED_GROUP_BACKGROUND;
            app.adjType = "exec-service";
            
            procState = ActivityManager.PROCESS_STATE_SERVICE; // 10
            //Slog.i(TAG, "EXEC " + (app.execServicesFg ? "FG" : "BG") + ": " + app);
            
        } else {

            // 其他类型的进程进入该分支，进程状态为 PROCESS_STATE_CACHED_EMPTY，调度组为 SCHED_GROUP_BACKGROUND！
            // 这种情况下我们无法知道实际的 adj，我们暂时使用 cache adj，后续我们会继续调整！
            schedGroup = ProcessList.SCHED_GROUP_BACKGROUND;
            
            adj = cachedAdj;
            procState = ActivityManager.PROCESS_STATE_CACHED_EMPTY; // 16
            
            app.cached = true;
            app.empty = true;
            app.adjType = "cch-empty";
        }
        
        ... ... ... ... // 见下面[3.1.2.1.4]！
```
首先来说一下：`mTopProcessState`，其表示 `top process` 的状态，其默认取值为：
```java
int mTopProcessState = ActivityManager.PROCESS_STATE_TOP;
```

当我们的系统进入睡眠状态的时候，会更新其状态，具体的方法在 `updateSleepIfNeededLocked` 方法：

- 当系统不睡眠时候，设置 `mTopProcessState = ActivityManager.PROCESS_STATE_TOP`；
- 当系统睡眠时候，设置 `mTopProcessState = ActivityManager.PROCESS_STATE_TOP_SLEEPING`；

我们来看看，**那些进程能归类于前台进程**：

- **1、当前进程是 `top process`**，进行如下处理：

```java
adj = ProcessList.FOREGROUND_APP_ADJ; // 取值 0
schedGroup = ProcessList.SCHED_GROUP_TOP_APP;
foregroundActivities = true;

procState = PROCESS_STATE_CUR_TOP; // 进程状态 2 或者 5！
```
这里的 `PROCESS_STATE_CUR_TOP` 就是 `mTopProcessState`，且 `foregroundActivities` 只有在该条件下才为 `true`；

- **4、当前进程不是 `top process`，但是其内部运行着 `instrumentation` 用于测试的话：** 

```java
adj = ProcessList.FOREGROUND_APP_ADJ; // 取值 0
schedGroup = ProcessList.SCHED_GROUP_DEFAULT;
foregroundActivities = false;

procState = ActivityManager.PROCESS_STATE_FOREGROUND_SERVICE; // 进程状态 4；
```
继续来看：

</br>

- **3、当前进程不是 `top process`，内部没有运行着 `instrumentation`，但是其内部有正在接受处理广播的 `BroadcastReceiver` 的话：** 

```java
adj = ProcessList.FOREGROUND_APP_ADJ; // 取值 0
schedGroup = (queue == mFgBroadcastQueue) ? ProcessList.SCHED_GROUP_DEFAULT : ProcessList.SCHED_GROUP_BACKGROUND;
foregroundActivities = false;

procState = ActivityManager.PROCESS_STATE_RECEIVER; // 进程状态 11；
```

如果广播所在的队列是后台队列，所属的调度组为：`SCHED_GROUP_BACKGROUND`，如果是前台，调度组为 `SCHED_GROUP_DEFAULT`；

</br>

-  **4、当前进程不是 `top process`，内部没有运行着 `instrumentation`，也没有接受处理广播的 `BroadcastReceiver`，但是有正在执行的 `Service` 的话：** 

```java
adj = ProcessList.FOREGROUND_APP_ADJ; // 取值 0
schedGroup = app.execServicesFg ? ProcessList.SCHED_GROUP_DEFAULT : ProcessList.SCHED_GROUP_BACKGROUND;
foregroundActivities = false;

procState = ActivityManager.PROCESS_STATE_SERVICE; // 进程状态 10；
```

如果后台执行，所属的调度组为：`SCHED_GROUP_BACKGROUND`，如果是前台，调度组为 `SCHED_GROUP_DEFAULT`；

</br>

- 5、其他不属于以上的情况，说明其不是前台进程，有如下处理：

```java
schedGroup = ProcessList.SCHED_GROUP_BACKGROUND;
adj = cachedAdj; // 我们使用 cachedAdj ，或者为 unknowAdj；

procState = ActivityManager.PROCESS_STATE_CACHED_EMPTY; // 进程状态为 16；
foregroundActivities = false;
app.cached = true;
app.empty = true;
```

对于这种情况，我们将其当作 `cache` 进程处理，后续会进行调整！

对于前台进程，其 `oom_adj` 均被赋值为 `FOREGROUND_APP_ADJ`，即从 `LowMemoryKiller` 的角度来看，它们的重要性是一致的。 
但这些进程的 `procState` 不同，于是从 `ActivityManagerService` 主动回收内存的角度来看，它们的重要性不同。

这里我们关注一个变量 `foregroundActivities`，这里只有在 `app == TOP_APP` 的情况下为 `true`，表示持有前台 `activity`!

我们继续来看！

#### 3.3.1.4 处理非前台 activity 所在进程

处理完了前台进程，接下来，处理非前台 `activity` 的情况，这里可以看作上面部分的延续，以下几种情况，会进入下面的逻辑，进一步的调整其 `adj`!

- **如果该进程是前台进程，但不是 `top process`，并且内部持有 `activity`**，其 `adj` 为 `FOREGROUND_APP_ADJ`！
- **如果该进程不是前台进程，并且内部持有 `activity`**，其 `adj` 为 `cachedAdj`！

这里的 `top process` 是 `top activity` 所在的进程！！

```java
        ... ... ... ... // 见上面[3.1.2.1.3]！

        //【5】处理完了前台进程，接下来，处理非前台 activity 的情况！
        if (!foregroundActivities && activitiesSize > 0) {

            // minLayer 默认等于 PERCEPTIBLE_APP_ADJ（200） - VISIBLE_APP_ADJ（100）- 1，等于 99！
            int minLayer = ProcessList.VISIBLE_APP_LAYER_MAX;

            for (int j = 0; j < activitiesSize; j++) { // 遍历！
                final ActivityRecord r = app.activities.get(j);
                if (r.app != app) {

                    Log.e(TAG, "Found activity " + r + " in proc activity list using " + r.app
                            + " instead of expected " + app);
                            
                    if (r.app == null || (r.app.uid == app.uid)) {
                        // 处理数据异常的问题！
                        r.app = app;
                    } else {
                        continue;
                    }
                }

                if (r.visible) {
                    //【5.1】如果 activity 是可见的，
                    // 那提高 adj 最高为 VISIBLE_APP_ADJ，提高 procState 最高为 PROCESS_STATE_CUR_TOP！
                    if (adj > ProcessList.VISIBLE_APP_ADJ) { // 100
                        adj = ProcessList.VISIBLE_APP_ADJ;
                        app.adjType = "visible";
                    }

                    if (procState > PROCESS_STATE_CUR_TOP) { // top state: 2/5
                        procState = PROCESS_STATE_CUR_TOP;
                    }

                    schedGroup = ProcessList.SCHED_GROUP_DEFAULT; // 调整调度组为 default！

                    app.cached = false;
                    app.empty = false;

                    foregroundActivities = true; // 设置 foregroundActivities 为 true；

                    // 如果其所在 task 如果不为 null，需要根据 task.mLayerRank
                    // 调整 minLayer 的值！
                    if (r.task != null && minLayer > 0) {
                        final int layer = r.task.mLayerRank;
                        if (layer >= 0 && minLayer > layer) {

                            // minLayer 为其所在 task.mLayerRank 值！
                            minLayer = layer;
                        }
                    }
                    break;

                } else if (r.state == ActivityState.PAUSING || r.state == ActivityState.PAUSED) {

                    //【5.2】如果 activity 正在暂停，或者已经暂停，
                    // 调整 adj 至少为 PERCEPTIBLE_APP_ADJ，调整 procState 至少为 top 进程的状态；
                    if (adj > ProcessList.PERCEPTIBLE_APP_ADJ) { // 200
                        adj = ProcessList.PERCEPTIBLE_APP_ADJ;
                        app.adjType = "pausing";
                    }

                    if (procState > PROCESS_STATE_CUR_TOP) { // top state: 2/5
                        procState = PROCESS_STATE_CUR_TOP;
                    }

                    schedGroup = ProcessList.SCHED_GROUP_DEFAULT;

                    app.cached = false;
                    app.empty = false;

                    foregroundActivities = true; // 设置 foregroundActivities 为 true；

                } else if (r.state == ActivityState.STOPPING) {

                    //【5.3】如果 activity 正在停止， 调整 adj 至少为 PERCEPTIBLE_APP_ADJ
                    if (adj > ProcessList.PERCEPTIBLE_APP_ADJ) { // 200
                        adj = ProcessList.PERCEPTIBLE_APP_ADJ;
                        app.adjType = "stopping";
                    }
                    
                    // 如果此时 activity 没有开始 finish，那就调整 procState 最低为 PROCESS_STATE_LAST_ACTIVITY！
                    if (!r.finishing) {
                        if (procState > ActivityManager.PROCESS_STATE_LAST_ACTIVITY) { // 13
                            procState = ActivityManager.PROCESS_STATE_LAST_ACTIVITY; 
                        }
                    }

                    app.cached = false;
                    app.empty = false;

                    foregroundActivities = true; // 设置 foregroundActivities 为 true；

                } else {

                    //【5.4】不属于以上情况的话，只是含有 cached activity 的进程！
                    // 这里只会设置进程状态 procState 最低为 PROCESS_STATE_CACHED_ACTIVITY！
                    if (procState > ActivityManager.PROCESS_STATE_CACHED_ACTIVITY) { // 14
                        procState = ActivityManager.PROCESS_STATE_CACHED_ACTIVITY; 
                        app.adjType = "cch-act";
                    }
                }
            }

            // 同时，不同可见进程的 oom_adj 有一定的差异，我们根据其所在的 task 的 mLayerRank 来动态调整其 adj！
            // 如果此时 adj 为 VISIBLE_APP_ADJ（100），那就给 adj 加上 minLayer！
            // minLayer 
            if (adj == ProcessList.VISIBLE_APP_ADJ) {
                adj += minLayer;
            }
        }
        
        ... ... ... ... // 见下面[3.1.2.1.5]！
```

**如果进程中没有 `top activity`，但是内部运行着 `activity`**，还需要进入如下的判断：

- **1、如果 r.visible，表示内部有 activity 是可见的：**
   - 如果 `adj > ProcessList.VISIBLE_APP_ADJ`：**100**，那就调整到 `ProcessList.VISIBLE_APP_ADJ`；
   - 如果 `procState > PROCESS_STATE_CUR_TOP`，那就调整到 `top process state`；
   - 设置 `foregroundActivities` 为 `true`；

**然后结束处理**；

</br>

- **2、如果 r.state == ActivityState.PAUSING || r.state == ActivityState.PAUSED，表示内部有 activity 是正在暂停，或者已经暂停：**

  - 如果 `adj > ProcessList.VISIBLE_APP_ADJ`：**100**，那就调整到 `ProcessList.VISIBLE_APP_ADJ`；
  - 如果 `procState > PROCESS_STATE_CUR_TOP`，那就调整到 `top process state`；
  - 设置 `foregroundActivities` 为 `true`；

**然后继续处理其他 `activity`**

</br>

- **3、如果 r.state == ActivityState.STOPPING，表示内部有 activity 是正在停止：**

  - 如果 `adj > ProcessList.PERCEPTIBLE_APP_ADJ`：**200**，那就调整到 `ProcessList.PERCEPTIBLE_APP_ADJ`；
  - 如果此时 `activity` 没有开始 `finish`，并且 `procState > ActivityManager.PROCESS_STATE_LAST_ACTIVITY`：**13**，那就调整 `procState` 为 `PROCESS_STATE_LAST_ACTIVITY`；
  - 设置 `foregroundActivities` 为 `true`，**然后继续处理其他 `activity`**；

</br>

- **4、其他情况：**
  - 如果 `procState > ActivityManager.PROCESS_STATE_CACHED_ACTIVITY`: **14**，那就设置 `procState` 为 `PROCESS_STATE_CACHED_ACTIVITY`，然后继续处理其他 `activity`!

</br>
    
- **5、最后，如果 `adj` 等于 `VISIBLE_APP_ADJ：100`**，在其之上加入 `minLayer` 调整，这样的话，最后的 `adj` 介于 `100` 和 `200` 之间！

</br>

**我们得到：**

- 对于 **如果该进程是前台进程，但不是 `top process` 这种情况，其 `adj` 为 `FOREGROUND_APP_ADJ：0`，不会做 `adj` 调整**，**但是其 `procState` 会发生变化**，其变化依据是依赖于进程内部的 `activity` 的状态，根据【3.3.1.3】节我们知道，其  `procState` 的取值如下：
```java
procState = PROCESS_STATE_CUR_TOP; // 进程状态 2 或者 5！
procState = ActivityManager.PROCESS_STATE_FOREGROUND_SERVICE; // 进程状态 4；
procState = ActivityManager.PROCESS_STATE_RECEIVER; // 进程状态 11；
procState = ActivityManager.PROCESS_STATE_SERVICE; // 进程状态 10；
```

可以看到，如果其内部持有非 `top activity`，这里的调整，顶多会将其调整到 `PROCESS_STATE_CUR_TOP` 级别！


- **而对于非前台进程，在【3.3.1.3】节，会被置为如下的状态**：
```java
adj = cachedAdj; // 我们使用 cachedAdj ，或者为 unknowAdj；
procState = ActivityManager.PROCESS_STATE_CACHED_EMPTY; // 进程状态为 16；
foregroundActivities = false;
```

可以看到，如果其内持有非 `top activity`，会根据其 `activity` 的状态，其 `adj` 被调整为：
```java
// 内部有非 top activity，且其 visible；
ProcessList.VISIBLE_APP_ADJ // 100

// 内部有非 top activity，其没 visible，但是 pausing 或者 paused，或者 stoping；
ProcessList.PERCEPTIBLE_APP_ADJ // 200
```
其 `procState` 被调整为：

```java
// 内部有非 top activity，其 visible / pausing / pauesed，且其 procState > PROCESS_STATE_CUR_TOP（满足）
ActivityManager.PROCESS_STATE_CUR_TOP // 2 或者 5

// 内部有非 top activity，其 stopping，但没 finishing，且其 procState > ActivityManager.PROCESS_STATE_LAST_ACTIVITY（满足）
ActivityManager.PROCESS_STATE_LAST_ACTIVITY // 13

// 内部有非 top activity，其没 visible / pausing / pauesed / stopping，且其 procState > PROCESS_STATE_CACHED_ACTIVITY（满足）
ActivityManager.PROCESS_STATE_CACHED_ACTIVITY // 14
```

</br>

我们接着来看！！


#### 3.3.1.5 调整可感知进程

接下来处理可感知的进程，可以看到，能够进入该分支的需要满足下面某一个条件：

- `adj` 的值大于 `PERCEPTIBLE_APP_ADJ（200）`：
- `procState` 的值大于 `PROCESS_STATE_FOREGROUND_SERVICE（4）`：

```
        ... ... ... ... // 见上面[3.1.2.1.4]！
        
        //【5】接下来，处理 adj 大于 PERCEPTIBLE_APP_ADJ（200） 
        // 或者 procState 大于 PROCESS_STATE_FOREGROUND_SERVICE（4）的情况！
        if (adj > ProcessList.PERCEPTIBLE_APP_ADJ
                || procState > ActivityManager.PROCESS_STATE_FOREGROUND_SERVICE) {

            if (app.foregroundServices) {

                //【5.1】如果该进程中有前台服务，那么用户是可以感知到这种进程的，所以是可见的！
                // 设置 adj 最低为 PERCEPTIBLE_APP_ADJ（200）
                // 设置 procState 最低为 PROCESS_STATE_FOREGROUND_SERVICE（4）
                adj = ProcessList.PERCEPTIBLE_APP_ADJ;
                procState = ActivityManager.PROCESS_STATE_FOREGROUND_SERVICE;
                app.cached = false;
                app.adjType = "fg-service";
                schedGroup = ProcessList.SCHED_GROUP_DEFAULT;

            } else if (app.forcingToForeground != null) {

                //【5.2】如果进程 forcingToForeground 不为 null，说明进程被强制设置到前台，同样可见！
                // 设置 adj 最低为 PERCEPTIBLE_APP_ADJ（200）
                // 设置 procState 最低为 PROCESS_STATE_IMPORTANT_FOREGROUND（6）
                adj = ProcessList.PERCEPTIBLE_APP_ADJ;
                procState = ActivityManager.PROCESS_STATE_IMPORTANT_FOREGROUND;
                app.cached = false;
                app.adjType = "force-fg";
                app.adjSource = app.forcingToForeground;
                schedGroup = ProcessList.SCHED_GROUP_DEFAULT;

            }
        }
        
        ... ... ... ... // 见下面[3.1.2.1.6]！
```
对于系统进程和前台进程 `adj` 的优先级均是高于 `ProcessList.FOREGROUND_APP_ADJ`，所以第一个条件是不满足的！

我们来看看**那些进程属于可感知的进程**：

- **`app.foregroundServices` 为 `true`，表示服务被 `start` 后，调用了 `startForeground` 方法**
此时我们来看看进程的属性：

```java
adj = ProcessList.PERCEPTIBLE_APP_ADJ; // 200;
procState = ActivityManager.PROCESS_STATE_FOREGROUND_SERVICE; // 4
schedGroup = ProcessList.SCHED_GROUP_DEFAULT;
```

在 `Service` 的被启动后，可以调用 `startForeground`，让服务进程变为可感知的
    

- **`app.foregroundServices` 为 `false`，但是 `app.forcingToForeground != null`，表示调用了 `setProcessForeground` 方法**
此时我们来看看进程的属性：

```java
adj = ProcessList.PERCEPTIBLE_APP_ADJ; // 200
procState = ActivityManager.PROCESS_STATE_IMPORTANT_FOREGROUND; // 6
schedGroup = ProcessList.SCHED_GROUP_DEFAULT;
```

当我们在进程中调用了 `setProcessForeground` 方法后，该进程的 `app.forcingToForeground` 不为 `null`，这样进程就会变为可感知的！！

</br>

**注意**

**这里要重点看第二个条件：`procState > ActivityManager.PROCESS_STATE_FOREGROUND_SERVICE : 4`**

只要满足该条件的进程，也可以进入该分支，即使对于前台进程，其 `adj == FOREGROUND_APP_ADJ`，也可能会进入该分支，比如：

- 不是 `top process`，其内部没有非`top activity`，但是其内部有正在接受处理广播的 `BroadcastReceiver`，其 `procState ==  PROCESS_STATE_RECEIVER：11`!

- 不是 `top process`，其内部没有非`top activity`，但是其内部有正在执行的 `Service`，其 `procState == PROCESS_STATE_SERVICE：10`!

如果其被显示设置成了前台的话，`adj` 和 `procState` 也会发生调整！

接着来看：

    
#### 3.3.1.6 heavy weight 进程

下面是处理 `heavy weight` 类型的进程！

```
        ... ... ... ... // 见上面[3.1.2.1.5]

        //【6】接下来，处理 heavy weight 进程！
        if (app == mHeavyWeightProcess) {
        
            //【6.1】设置进程 adj 最低为 HEAVY_WEIGHT_APP_ADJ
            if (adj > ProcessList.HEAVY_WEIGHT_APP_ADJ) { // 400
                adj = ProcessList.HEAVY_WEIGHT_APP_ADJ;
                schedGroup = ProcessList.SCHED_GROUP_BACKGROUND;
                app.cached = false;
                app.adjType = "heavy";
            }
            
            //【6.2】设置进程状态 procState 最低为 PROCESS_STATE_HEAVY_WEIGHT
            if (procState > ActivityManager.PROCESS_STATE_HEAVY_WEIGHT) { // 9
                procState = ActivityManager.PROCESS_STATE_HEAVY_WEIGHT;
            }
        }
        
        ... ... ... ... // 见下面[3.1.2.1.7]
```

`ams` 通过 `mHeavyWeightProcess` 来保存系统中的 `heavy weight` 进程！

- 如果 `app == mHeavyWeightProcess`，说明该进程是 `heavy weight` 类型的进程，我们来看看其属性设置：

   - **adj = ProcessList.HEAVY_WEIGHT_APP_ADJ;**
   - **procState = ActivityManager.PROCESS_STATE_HEAVY_WEIGHT;**
   - schedGroup = ProcessList.SCHED_GROUP_BACKGROUND;

**注意**，对于 `adj` 的调整，只有 `cacheAdj` 或者 `unKnowAdj` 的情况下，如果满足条件，才会被调整为 `HEAVY_WEIGHT_APP_ADJ`；

而对于 `stateProc`，只要 `procState > ActivityManager.PROCESS_STATE_HEAVY_WEIGHT：9`，并且该进程是 `heavy weight` 进程，那就会发生调整，这个可以参见【3.3.1.6】节！
   
接着来看：

#### 3.3.1.7 home 进程

下面是处理 `home` 进程！

```java
        ... ... ... ... // 见上面[3.1.2.1.6]

        //【7】接下来，处理 home 进程！
        if (app == mHomeProcess) {

            //【7.1】设置进程 adj 最低为 HOME_APP_ADJ
            if (adj > ProcessList.HOME_APP_ADJ) { // 600
                adj = ProcessList.HOME_APP_ADJ;
                schedGroup = ProcessList.SCHED_GROUP_BACKGROUND;
                app.cached = false;
                app.adjType = "home";
            }

            //【7.2】设置进程状态 procState 最低为 PROCESS_STATE_HOME！
            if (procState > ActivityManager.PROCESS_STATE_HOME) { // 12
                procState = ActivityManager.PROCESS_STATE_HOME;
            }
        }
 
       ... ... ... ... // 见下面[3.1.2.1.8]       
```

- 如果 `app == mHomeProcess`，说明该进程是 `home` 所在的进程，我们来看看其属性设置：

   - **adj = ProcessList.HOME_APP_ADJ; // 600**
   - **procState = ActivityManager.PROCESS_STATE_HOME; // 12**
   - schedGroup = ProcessList.SCHED_GROUP_BACKGROUND;

**注意：**

- 对于 `home process` 的情况，只有 `cacheAdj` 或者 `unKnowAdj` 的情况下，才会被调整 `adj` 和 `procState`！

- 而对于 `foreground process`，`visible process`，`perceptible process`和 `heavy weight process` 来说，他们的 `adj` 和 `procState`都远远高于 `home process`，所以是不会调整的！
   
接着来看：

#### 3.3.1.8 持有 activity 的 previous 进程

下面是处理持有 `activity` 的 `previous` 进程：

```java
        ... ... ... ... // 见上面[3.1.2.1.7]

        //【8】接下来，处理用户之前所在的，持有 activity 的 previous 进程！
        // 因为这是前一个显示 ui 给用户的进程，我们尽量不杀死它，这样能够给用户一个好的用户体验！
        if (app == mPreviousProcess && app.activities.size() > 0) {

            //【8.1】设置进程 adj 最低为 PREVIOUS_APP_ADJ！
            if (adj > ProcessList.PREVIOUS_APP_ADJ) { // 700
                adj = ProcessList.PREVIOUS_APP_ADJ;
                schedGroup = ProcessList.SCHED_GROUP_BACKGROUND;
                app.cached = false;
                app.adjType = "previous";
            }

            //【8.2】设置进程状态 procState 最低为 PROCESS_STATE_LAST_ACTIVITY！
            if (procState > ActivityManager.PROCESS_STATE_LAST_ACTIVITY) { // 13
                procState = ActivityManager.PROCESS_STATE_LAST_ACTIVITY;
            }
        }

        if (false) Slog.i(TAG, "OOM " + app + ": initial adj=" + adj
                + " reason=" + app.adjType);
                
       ... ... ... ... // 见下面[3.1.2.1.9]      
```

**持有 `activity` 的 `previous` 进程**，就是上一个显示 `activity` 的进程！

- **`app == mPreviousProcess && app.activities.size() > 0`，表示该进程是用户所在的上一个进程，并且该进程内部运行着 `activity`：**
下面我们来看看这类进程的属性设置：

 - **adj = ProcessList.PREVIOUS_APP_ADJ**;
 - **procState = ActivityManager.PROCESS_STATE_LAST_ACTIVITY**;
 - schedGroup = ProcessList.SCHED_GROUP_BACKGROUND;

**注意：**

- 对于 `previous process` 的情况，只有 `cacheAdj` 或者 `unKnowAdj` 的情况下，才会被调整 `adj` 和 `procState`！

- 而对于 `foreground process`，`visible process`，`perceptible process`和 `heavy weight process`， `home process` 来说，他们的 `adj` 和 `procState`都远远高于 `previous process`，所以是不会调整的！

接着来看：

#### 3.3.1.9 处于 back-up 的进程

下面是处理处于 `back-up` 的进程：

```java
        ... ... ... ... // 见上面[3.1.2.1.8]

        // 设置进程的 adjSeq 为此时的 mAdjSeq，通过比较 adjSeq 和 mAdjSeq 就可以知道进程是否已经进行了 oomAdj 更新！
        app.adjSeq = mAdjSeq;

        // 如果进程中的 services 或者 providers 被其他进程依赖，那么该进程的 oomAdj 也会发生变化，所以后续还要调整！
        app.curRawAdj = adj; 
        app.hasStartedServices = false; // hasStartedServices 表示该进程中是否有被启动的 service

        //【9】接下来，处理正在执行备份操作的进程，对比这类进程，我们要避免杀死他们；
        if (mBackupTarget != null && app == mBackupTarget.app) {

            //【9.1】设置进程 adj 最低为 BACKUP_APP_ADJ：300
            if (adj > ProcessList.BACKUP_APP_ADJ) { // 300

                if (DEBUG_BACKUP) Slog.v(TAG_BACKUP, "oom BACKUP_APP_ADJ for " + app);
                adj = ProcessList.BACKUP_APP_ADJ;
                
                // 同时，设置进程 procState 最低为 PROCESS_STATE_IMPORTANT_BACKGROUND
                if (procState > ActivityManager.PROCESS_STATE_IMPORTANT_BACKGROUND) { // 7
                    procState = ActivityManager.PROCESS_STATE_IMPORTANT_BACKGROUND;
                }

                app.adjType = "backup";
                app.cached = false;
            }

            //【9.2】设置进程状态 procState 最低为 PROCESS_STATE_BACKUP：8！
            if (procState > ActivityManager.PROCESS_STATE_BACKUP) {
                procState = ActivityManager.PROCESS_STATE_BACKUP;
            }
        }

       ... ... ... ... // 见下面[3.1.2.1.10]  
```
对于正在备份的进程有：

- `mBackupTarget != null && app == mBackupTarget.app` 条件满足，那就做如下调整：

如果 `adj > ProcessList.BACKUP_APP_ADJ` 的话：
```java
adj = ProcessList.BACKUP_APP_ADJ; // 优先级最低为 BACKUP_APP_ADJ
procState = ActivityManager.PROCESS_STATE_IMPORTANT_BACKGROUND; // 优先级最低为 PROCESS_STATE_IMPORTANT_BACKGROUND
```
否则，只调整 `procState` 优先级最低为 `PROCESS_STATE_BACKUP` 

**注意：**

- **正在备份的进程和前面遇到的进程并不冲突**，比如，一个前台进程也有可能正在做备份操作，所以这里将其对应的调整放在了其他进程之后！

我们继续来看：

#### 3.3.1.10 处理持有 services 的进程

下面是处理处于持有 `services` 的进程，当进程中持有 `services` 并且别其他进程绑定后，该进程的 `adj` 和 `state` 会发生变化！

```java
       ... ... ... ... // 见上面[3.1.2.1.9]  
       
        boolean mayBeTop = false;

        //【9】处理该进程中的 serivce，判断是否有其他进程依赖这些 service，如果有，那么该进程的 adj 会发生变化！
        // 循环触发的条件是：adj > ProcessList.FOREGROUND_APP_ADJ / schedGroup == ProcessList.SCHED_GROUP_BACKGROUND
        //  / procState > ActivityManager.PROCESS_STATE_TOP
        for (int is = app.services.size()-1;
                is >= 0 && (adj > ProcessList.FOREGROUND_APP_ADJ
                        || schedGroup == ProcessList.SCHED_GROUP_BACKGROUND
                        || procState > ActivityManager.PROCESS_STATE_TOP);
                is--) {

//-----------------------------------------------------------------------------------------------------

            //【9.1】处理每一个 ServiceRecord 对象！
            ServiceRecord s = app.services.valueAt(is);
            
            //【9.2】s.startRequested 为 true 表示服务是通过 startService 方式启动了！
            if (s.startRequested) {
                // 设置 hasStartedServices 为 true！
                app.hasStartedServices = true;
                // 同时设置 procState 至少为 PROCESS_STATE_SERVICE！
                if (procState > ActivityManager.PROCESS_STATE_SERVICE) { // 10！
                    procState = ActivityManager.PROCESS_STATE_SERVICE;
                }

                if (app.hasShownUi && app != mHomeProcess) {

                    // 如果进程自启动后，显示过 ui ，并且不是 home 进程，这里并没有做调整，设置了一个标记，用于 debug！
                    if (adj > ProcessList.SERVICE_ADJ) {
                        app.adjType = "cch-started-ui-services";
                    }

                } else {

                    // 如果该服务的在 30 mins 内活跃过，那么我们会保持其进程在后台之前！
                    // 调整 adj 最低为 SERVICE_ADJ，可以看到 adj 大于 500 的进程均会受此判断的影响！
                    if (now < (s.lastActivity + ActiveServices.MAX_SERVICE_INACTIVITY)) {
                        if (adj > ProcessList.SERVICE_ADJ) { // 500
                            adj = ProcessList.SERVICE_ADJ;
                            app.adjType = "started-services";
                            app.cached = false;
                        }
                    }

                   
                    // 如果该服务没有活跃的时间已经超过了 30 mins，那就不会更新其 adj！
                    if (adj > ProcessList.SERVICE_ADJ) {
                        app.adjType = "cch-started-services"; // 增加 debug 的描述信息！
                    }
                }
            }

//----------------------------------------------------------------------------------------

            //【9.3】如果该服务被 bind 了，处理该服务的所有绑定信息！
            for (int conni = s.connections.size()-1;
                    conni >= 0 && (adj > ProcessList.FOREGROUND_APP_ADJ
                            || schedGroup == ProcessList.SCHED_GROUP_BACKGROUND
                            || procState > ActivityManager.PROCESS_STATE_TOP);
                    conni--) {

                ArrayList<ConnectionRecord> clist = s.connections.valueAt(conni);

                for (int i = 0;
                        i < clist.size() && (adj > ProcessList.FOREGROUND_APP_ADJ
                                || schedGroup == ProcessList.SCHED_GROUP_BACKGROUND
                                || procState > ActivityManager.PROCESS_STATE_TOP);
                        i++) {
                    
                    // 获得连接的 client 客户端！
                    ConnectionRecord cr = clist.get(i);

                    //【9.3.1】如果 client 的宿主进程就是当前进程，跳过！
                    if (cr.binding.client == app) {
                        continue;
                    }

                    //【9.3.1】接下来，根据 bind 时候设置的 flags 的不同，进行不同的处理！
                    
///--------------------------------------------------------------------------------------

                    //【9.3.1.1】如果 bind 的时候没有设置 BIND_WAIVE_PRIORITY，进入下面的分支！
                    // BIND_WAIVE_PRIORITY 表示 client 会不会影响服务进程的优先级，为 1 表示不会影响，就不会进入 if 分支
                    // 为 0 表示会影响！
                    if ((cr.flags & Context.BIND_WAIVE_PRIORITY) == 0) {
                        
                        ProcessRecord client = cr.binding.client;

                        // 计算绑定者进程 client 的 adj，这里依然是调用 computeOomAdjLocked 方法，不多说！
                        // 计算绑定者进程 client 的状态！ 
                        int clientAdj = computeOomAdjLocked(client, cachedAdj,
                                TOP_APP, doingAll, now);
                        int clientProcState = client.curProcState;
                        
                        // 当绑定者进程 client 的状态 clientProcState >= PROCESS_STATE_CACHED_ACTIVITY：14
                        // 设置其为 PROCESS_STATE_CACHED_EMPTY：16
                        if (clientProcState >= ActivityManager.PROCESS_STATE_CACHED_ACTIVITY) {
                            clientProcState = ActivityManager.PROCESS_STATE_CACHED_EMPTY;
                        }

                        String adjType = null;
                        
                        // 如果 bind 的时候还设置了 BIND_ALLOW_OOM_MANAGEMENT 标志位，表明
                        // 如果遇到 OOM 需要杀死进程，被绑定的服务进程会被 OOM 杀掉，那就不调整该进程的 adj！！
                        if ((cr.flags & Context.BIND_ALLOW_OOM_MANAGEMENT) != 0) {

                            if (app.hasShownUi && app != mHomeProcess) {
                                // 如果当前进程已经显示 ui 且不是 home 进程，
                                // 此时设置 clientAdj 和 clientProcState 均为当前进程的 adj 和 procState！
                                // 不再考虑 client 的影响！
                                if (adj > clientAdj) {
                                    // 用一个标签标记，用于debug
                                    adjType = "cch-bound-ui-services";
                                }

                                app.cached = false;

                                // 设置 client 的 adj 和 procState；
                                clientAdj = adj;
                                clientProcState = procState;

                            } else {
                                // 如果当前进程没有显示 ui，并且在距离上次该服务活跃的时间超过 30mins
                                // 此时只设置 clientAdj 为当前进程的 adj！
                                // 不再考虑 client 的影响！
                                if (now >= (s.lastActivity
                                        + ActiveServices.MAX_SERVICE_INACTIVITY)) {

                                    if (adj > clientAdj) {
                                        adjType = "cch-bound-services";
                                    }

                                    clientAdj = adj;
                                }
                            }
                        }

                        // 如果 client 的 adj 和当前进程的 adj 一样，即不需要考虑客户端进程了
                        // 如果当前进程的 adj > 绑定者进程的 clientAdj，那就需要调整当前进程的 adj，进入以下分支！
                        if (adj > clientAdj) {
                            if (app.hasShownUi && app != mHomeProcess
                                    && clientAdj > ProcessList.PERCEPTIBLE_APP_ADJ) {

                                // 如果进程显示 ui，并且不是 home process，
                                // 并且绑定者进程的 adj 优先级低于可感知 ProcessList.PERCEPTIBLE_APP_ADJ: 200
                                // 这种情况也不需要考虑客户端进程的影响！
                                adjType = "cch-bound-ui-services";

                            } else {

                                // 不满足上述条件，进入下面的判断，此时该进程的 adj 会收到 client 进程的 adj 的影响！
                                if ((cr.flags & (Context.BIND_ABOVE_CLIENT
                                        | Context.BIND_IMPORTANT)) != 0) {

                                    // 如果绑定设置了 BIND_IMPORTANT 和 BIND_ABOVE_CLIENT 标记
                                    // BIND_ABOVE_CLIENT 表示客户端处于前台时，绑定的 service 进程也变为前台进程
                                    // 那么该进程的 adj 满足一下关系：
                                    // adj 会被调整到和 client 一样但是不超过 PERSISTENT_SERVICE_ADJ！
                                    adj = clientAdj >= ProcessList.PERSISTENT_SERVICE_ADJ
                                            ? clientAdj : ProcessList.PERSISTENT_SERVICE_ADJ;

                                } else if ((cr.flags & Context.BIND_NOT_VISIBLE) != 0
                                        && clientAdj < ProcessList.PERCEPTIBLE_APP_ADJ
                                        && adj > ProcessList.PERCEPTIBLE_APP_ADJ) {

                                    // BIND_NOT_VISIBLE 表示服务进程不能变为 visible 进程
                                    // 如果绑定设置了 BIND_NOT_VISIBLE 标记，且绑定者 adj 优先级
                                    // 高于可感知，而当前进程的 adj 优先级低于可感知
                                    // 那么该进程的 adj 会被提升到可感知 PERCEPTIBLE_APP_ADJ！
                                    adj = ProcessList.PERCEPTIBLE_APP_ADJ;

                                } else if (clientAdj >= ProcessList.PERCEPTIBLE_APP_ADJ) {

                                    // 其他情况，如果 clientAdj 优先级不高于可感知
                                    // 那么当前进程的 adj 和 client 一样！
                                    adj = clientAdj;
 
                                } else {

                                    // 其他情况，如果当前进程的 adj 大于可感知 VISIBLE_APP_ADJ
                                    // 那么 adj 取绑定者进程 clientAdj 和 VISIBLE_APP_ADJ 的最大值，即优先级最低的那个！
                                    if (adj > ProcessList.VISIBLE_APP_ADJ) {
                                        adj = Math.max(clientAdj, ProcessList.VISIBLE_APP_ADJ);
                                    }
                                }

                                // 如果绑定者进程没有处于 cache 状态，那么当前进程也不能处于 cache！
                                if (!client.cached) {
                                    app.cached = false;
                                }

                                adjType = "service";
                            }
                        }
                        
                        // 接下来，处理 BIND_NOT_FOREGROUND 标志位的情况！
                        if ((cr.flags & Context.BIND_NOT_FOREGROUND) == 0) {
                            //【9.3.1.2】如果 bind 的时候没有设置 BIND_NOT_FOREGROUND 标志位，
                            // 那么被绑定的服务进程优先级可以被提升到 FOREGROUND 级别！
                            
                            // 如果 client 的调度组大于该进程的调度组，调整该进程的调度组
                            if (client.curSchedGroup > schedGroup) {
                                if ((cr.flags & Context.BIND_IMPORTANT) != 0) {
                                    schedGroup = client.curSchedGroup;
                                } else {
                                    schedGroup = ProcessList.SCHED_GROUP_DEFAULT;
                                }
                            }

                            // 根据 client 进程的状态，设置 clientProcState 的值，后面会根据 clientProcState 为参考
                            // 调整该进程的 procState！
                            // 如果 clientProcState 优先级不低于 PROCESS_STATE_TOP，说明其处于 top 状态，进入 IF 分支
                            if (clientProcState <= ActivityManager.PROCESS_STATE_TOP) {
                                if (clientProcState == ActivityManager.PROCESS_STATE_TOP) {
                                
                                    // 如果 clientProcState 等于 PROCESS_STATE_TOP，说明 client 进程就是 top 进程！
                                    // 设置 mayBeTop 为 true，后续会统一做调整，这里不做调整！
                                    // 设置 clientProcState 为 PROCESS_STATE_CACHED_EMPTY，不做参考！
                                    mayBeTop = true;
                                    clientProcState = ActivityManager.PROCESS_STATE_CACHED_EMPTY;

                                } else {
    
                                    // 如果 clientProcState 小于 PROCESS_STATE_TOP，说明 client 进程的优先级比 top process
                                    // 还要高，比如 persistent 进程，这种情况不会将当前进程调整为 top，而是需要
                                    // 调整 clientProcState 的值，作为参考！
                                    if ((cr.flags & Context.BIND_FOREGROUND_SERVICE) != 0) {
                                        clientProcState =
                                                ActivityManager.PROCESS_STATE_BOUND_FOREGROUND_SERVICE;

                                    } else if (mWakefulness
                                                    == PowerManagerInternal.WAKEFULNESS_AWAKE &&
                                            (cr.flags & Context.BIND_FOREGROUND_SERVICE_WHILE_AWAKE)
                                                    != 0) {
                                        clientProcState =
                                                ActivityManager.PROCESS_STATE_BOUND_FOREGROUND_SERVICE;

                                    } else {
                                        clientProcState =
                                                ActivityManager.PROCESS_STATE_IMPORTANT_FOREGROUND;

                                    }
                                }
                            }
                            
                        } else {
                            //【9.3.1.2】如果 bind 的时候设置了 BIND_NOT_FOREGROUND 标志位，
                            // 那么被绑定的服务进程优先级不能被提升到 FOREGROUND 级别！
                            // 设置 clientProcState 最高为 PROCESS_STATE_IMPORTANT_BACKGROUND！
                            if (clientProcState <
                                    ActivityManager.PROCESS_STATE_IMPORTANT_BACKGROUND) {

                                clientProcState =
                                        ActivityManager.PROCESS_STATE_IMPORTANT_BACKGROUND;
                            }
                        }
                        
                        // 如果当前进程的 state 优先级低于 client 进程的 state，设置二者相等！
                        if (procState > clientProcState) {
                            procState = clientProcState;
                        }
                        
                        // 如果当前进程的状态高于 PROCESS_STATE_IMPORTANT_BACKGROUND
                        // 并且绑定时候设置了 BIND_SHOWING_UI 标志位，那就设置 pendingUiClean 为 true
                        if (procState < ActivityManager.PROCESS_STATE_IMPORTANT_BACKGROUND
                                && (cr.flags & Context.BIND_SHOWING_UI) != 0) {
                            app.pendingUiClean = true;
                        }
                        
                        // debug 调试相关，暂不处理！
                        if (adjType != null) {
                            app.adjType = adjType;
                            app.adjTypeCode = ActivityManager.RunningAppProcessInfo
                                    .REASON_SERVICE_IN_USE;
                            app.adjSource = cr.binding.client;
                            app.adjSourceProcState = clientProcState;
                            app.adjTarget = s.name;
                        }
                    }

///------------------------------------------------------------------------------------------------------------

                    //【9.2.1.2】如果 bind 的时候设置了 BIND_TREAT_LIKE_ACTIVITY，设置其 treatLikeActivity 为 true！
                    if ((cr.flags & Context.BIND_TREAT_LIKE_ACTIVITY) != 0) {
                        app.treatLikeActivity = true;
                    }

///-----------------------------------------------------------------------------------------------------------

                    final ActivityRecord a = cr.activity; // 获得绑定该 service 的 activity！
                    
                    //【9.2.1.3】如果 bind 的时候设置了 BIND_ADJUST_WITH_ACTIVITY，
                    // 表示系统根据绑定 service 的 activity 的重要程度来调整这个 service 的优先级！
                    if ((cr.flags&Context.BIND_ADJUST_WITH_ACTIVITY) != 0) {
    
                        // 如果存在绑定者 activity，并且进程此时的 adj 大于 FOREGROUND_APP_ADJ
                        // 同时 activity 是可见的，或者 activity 处于 resume 或者 pause 状态，
                        // 那就需要调整其 adj 为 FOREGROUND_APP_ADJ！
                        if (a != null && adj > ProcessList.FOREGROUND_APP_ADJ &&
                            (a.visible || a.state == ActivityState.RESUMED ||
                             a.state == ActivityState.PAUSING)) {

                            adj = ProcessList.FOREGROUND_APP_ADJ; // 那就需要调整其 adj 为 FOREGROUND_APP_ADJ！
                            
                            // 接着判断 flags 是否设置了 BIND_NOT_FOREGROUND 标志位，
                            // 该 flags 表示被绑定的 service 永远不会有运行于前台的优先级，如果没有设置，进入该分支！
                            if ((cr.flags & Context.BIND_NOT_FOREGROUND) == 0) {
                            
                                // 接着判断 flags 是否设置了 BIND_IMPORTANT 标志位，
                                // 该标志位表示服务对客户端是非常重要的，会将服务提升至前台进程优先级！
                                // 调整其所属的调度组；
                                if ((cr.flags & Context.BIND_IMPORTANT) != 0) {
                                    schedGroup = ProcessList.SCHED_GROUP_TOP_APP_BOUND;

                                } else {
                                    schedGroup = ProcessList.SCHED_GROUP_DEFAULT;

                                }
                            }
                            app.cached = false;
                            app.adjType = "service";
                            app.adjTypeCode = ActivityManager.RunningAppProcessInfo
                                    .REASON_SERVICE_IN_USE;
                            app.adjSource = a;
                            app.adjSourceProcState = procState;
                            app.adjTarget = s.name;
                        }
                    }
                }
            }
        }

       ... ... ... ... // 见下面[3.1.2.1.11]  
```
**1、进入条件：**

-   `is >= 0 && (adj > ProcessList.FOREGROUND_APP_ADJ || schedGroup == ProcessList.SCHED_GROUP_BACKGROUND
                        || procState > ActivityManager.PROCESS_STATE_TOP`
**第一个条件**：该进程中运行着服务！
**第二个条件**：
           该进程的 `adj > ProcessList.FOREGROUND_APP_ADJ`，只要优先级低于前台进程，都可以满足；
           或者 `schedGroup == ProcessList.SCHED_GROUP_BACKGROUND`，只要调度组为后台类型，就可以满足；
           或者 `procState > ActivityManager.PROCESS_STATE_TOP`，只要进程状态优先级低于 `top state`；


**2、处理 `Service` 的逻辑如下**：

- 处理进程中通过 `startService` 启动的服务：
- 处理进程中被其他进程 `bind` 的服务，针对 `bind` 时候传入的 `flags` 进行不同的处理：
    - 如果设置了 `Context.BIND_WAIVE_PRIORITY`：
    - 如果设置了 `Context.BIND_TREAT_LIKE_ACTIVITY`：
    - 如果设置了 `Context.BIND_ADJUST_WITH_ACTIVITY`：

#### 3.3.1.11 处理持有 providers 的进程

最后，来看看对持有 `ContentProvider` 进程的处理， 由于 `ContentProvider` 也是有 `client` 的，所以客户端的绑定也会影响其进程！

```java
       ... ... ... ... // 见上面[3.1.2.1.10]  

        //【10】处理该进程中的 providers，判断是否有其他进程依赖这些 providers，如果有，那么该进程的 adj 会发生变化！
        for (int provi = app.pubProviders.size()-1;
                provi >= 0 && (adj > ProcessList.FOREGROUND_APP_ADJ
                        || schedGroup == ProcessList.SCHED_GROUP_BACKGROUND
                        || procState > ActivityManager.PROCESS_STATE_TOP);
                provi--) {
                
            //【10.1】app.pubProviders 内部用于保存该进程中所有的 provider！
            ContentProviderRecord cpr = app.pubProviders.valueAt(provi);
            for (int i = cpr.connections.size()-1;
                    i >= 0 && (adj > ProcessList.FOREGROUND_APP_ADJ
                            || schedGroup == ProcessList.SCHED_GROUP_BACKGROUND
                            || procState > ActivityManager.PROCESS_STATE_TOP);
                    i--) {

                ContentProviderConnection conn = cpr.connections.get(i);
                ProcessRecord client = conn.client;

                //【10.1.1】跳过那些同一个进程中的绑定！
                if (client == app) {
                    continue;
                }

                //【10.1.2】计算 client 进程的 adj 和 procState！
                int clientAdj = computeOomAdjLocked(client, cachedAdj, TOP_APP, doingAll, now);
                int clientProcState = client.curProcState;
    
                //【10.1.3】如果 client 的 procState 优先级比 PROCESS_STATE_CACHED_ACTIVITY 还低，调整其最低为：
                // PROCESS_STATE_CACHED_EMPTY，即如果 client 进程属于缓存状态，我们一律将其视为空进程；
                if (clientProcState >= ActivityManager.PROCESS_STATE_CACHED_ACTIVITY) {
                    clientProcState = ActivityManager.PROCESS_STATE_CACHED_EMPTY;
                }

                //【10.1.4】处理该进程的 adj 的优先级比 client 进程的 adj 低，那就需要调整该进程的 adj 了！
                if (adj > clientAdj) {
                    // 这里和 Service 的处理是一样的！
                    // 如果该进程已经显示过了 ui，并且不是 home proces，且其 client 进程的 adj 优先级低于可感知
                    // 那就不考虑 client 进程对当前进程的影响，不调整其 adj！
                    if (app.hasShownUi && app != mHomeProcess
                            && clientAdj > ProcessList.PERCEPTIBLE_APP_ADJ) {
                        app.adjType = "cch-ui-provider";

                    } else {
                        // 否则，将当前进程的 adj 调整为 client 进程的 adj，但是不超过 FOREGROUND_APP_ADJ！
                        adj = clientAdj > ProcessList.FOREGROUND_APP_ADJ
                                ? clientAdj : ProcessList.FOREGROUND_APP_ADJ;
                        app.adjType = "provider";

                    }

                    app.cached &= client.cached;

                    // debug 相关，不处理！
                    app.adjTypeCode = ActivityManager.RunningAppProcessInfo
                            .REASON_PROVIDER_IN_USE;
                    app.adjSource = client;
                    app.adjSourceProcState = clientProcState;
                    app.adjTarget = cpr.name;
                }

                //【10.1.5】调整该进程的 procState！！
                // 如果 client 进程的 procState 优先级不高于 top process 的状态，我们要对 clientProcState 做一下调整！
                if (clientProcState <= ActivityManager.PROCESS_STATE_TOP) {

                    // 要先调整下 client 进程的 procState！
                    if (clientProcState == ActivityManager.PROCESS_STATE_TOP) {
                        // 如果 client 进程是 top 级别的，设置 mayBeTop 为 true，此时我们不会立刻做调整，后续会做调整！
                        // 设置 clientProcState 为 PROCESS_STATE_CACHED_EMPTY，不做参考！ 
                        mayBeTop = true;
                        clientProcState = ActivityManager.PROCESS_STATE_CACHED_EMPTY;
                        
                    } else {
                        // 如果 clientProcState 小于 PROCESS_STATE_TOP，说明 client 进程的优先级比 top process
                        // 还要高，比如 persistent 进程，这种情况不会将当前进程调整为 top，而是需要
                        // 调整 clientProcState 的值，作为参考，这里是设置其为 PROCESS_STATE_BOUND_FOREGROUND_SERVICE
                        clientProcState =
                                ActivityManager.PROCESS_STATE_BOUND_FOREGROUND_SERVICE;
                    }
                }

                // 如果此时该进程的 procState 优先级低于 client 进程的 procState，提高其等于 client 进程的 procState
                if (procState > clientProcState) {
                    procState = clientProcState;
                }

                //【10.1.6】调整该进程的调度组 schedGroup！       
                if (client.curSchedGroup > schedGroup) {
                    schedGroup = ProcessList.SCHED_GROUP_DEFAULT;
                }
            }

            //【10.2】如何当前的 provider 依赖于一些 external (non-framework) process，
            // 那么我们要保证当前进程的 adj 至少为 FOREGROUND_APP_ADJ
            if (cpr.hasExternalProcessHandles()) {

                if (adj > ProcessList.FOREGROUND_APP_ADJ) {
                    adj = ProcessList.FOREGROUND_APP_ADJ;
                    schedGroup = ProcessList.SCHED_GROUP_DEFAULT;
                    app.cached = false;
                    app.adjType = "provider";
                    app.adjTarget = cpr.name;
                }
                if (procState > ActivityManager.PROCESS_STATE_IMPORTANT_FOREGROUND) {
                    procState = ActivityManager.PROCESS_STATE_IMPORTANT_FOREGROUND;
                }
            }
        }

        //【11】如果该进程中之前有 provider 活跃过，并且距离上一次 provider 活时间超过了 20 秒
        // 对该进程的 adj，schedGroup 和 procState 做进一步调整；
        if (app.lastProviderTime > 0 && (app.lastProviderTime + CONTENT_PROVIDER_RETAIN_TIME) > now) {
            if (adj > ProcessList.PREVIOUS_APP_ADJ) {
                adj = ProcessList.PREVIOUS_APP_ADJ;
                schedGroup = ProcessList.SCHED_GROUP_BACKGROUND;
                app.cached = false;
                app.adjType = "provider";
            }

            if (procState > ActivityManager.PROCESS_STATE_LAST_ACTIVITY) {
                procState = ActivityManager.PROCESS_STATE_LAST_ACTIVITY;
            }
        }

        ... ... ...// 见下面[3.1.2.1.12]  
```
这个方法的流程很长，我们总结一下整个的计算过程：

- 整个的计算过程是从优先级高的 `adj` 开始，逐级向下遍历，逐渐的调整进程的 `oomAdj` 和 `procState` 的状态，知道找到一个合适的 `oomAdj` 和 `procState` 值！


#### 3.3.1.12 最后调整阶段

```java
        ... ... ...// 见上面[3.1.2.1.11]  

        //【12】处理 top process 绑定了当前进程的 Service 或者 Provider 的情况！
        if (mayBeTop && procState > ActivityManager.PROCESS_STATE_TOP) {
        
            // top 进程绑定了该进程 Service 或者 provider，我们应该将该进程也设置为 top 状态
            // 但是如果此时该进程因为一些原因正在后台运行，我们会将其设置到特定的状态！
            switch (procState) {
                case ActivityManager.PROCESS_STATE_IMPORTANT_FOREGROUND: // 很重要，可以感知到；
                case ActivityManager.PROCESS_STATE_IMPORTANT_BACKGROUND: // 很重要，感知不到；
                case ActivityManager.PROCESS_STATE_SERVICE: // 后台服务；
                    // 上面这三这种状态都是会长时间保持的状态，将其设置为 PROCESS_STATE_BOUND_FOREGROUND_SERVICE！
                    // 在所有的后台进程之上
                    procState = ActivityManager.PROCESS_STATE_BOUND_FOREGROUND_SERVICE;
                    break;

                default:
                    // 其他情况，该进程的状态为 top
                    procState = ActivityManager.PROCESS_STATE_TOP;
                    break;
            }
        }

        //【13】如果该进程的 procState 优先级低于 PROCESS_STATE_CACHED_EMPTY，表示当前进程是一个 cache 进程！
        if (procState >= ActivityManager.PROCESS_STATE_CACHED_EMPTY) {
            if (app.hasClientActivities) { 

                // hasClientActivities 为 true，表示该进程内部有 activity 作为客户端绑定其他的服务！ 
                // 调整 procState 为 PROCESS_STATE_CACHED_ACTIVITY_CLIENT
                procState = ActivityManager.PROCESS_STATE_CACHED_ACTIVITY_CLIENT;
                app.adjType = "cch-client-act";

            } else if (app.treatLikeActivity) {

                // treatLikeActivity 为 true，表示该进程需要被当作有 activity 的进程对待！
                // 设置 procState 为 PROCESS_STATE_CACHED_ACTIVITY
                procState = ActivityManager.PROCESS_STATE_CACHED_ACTIVITY;
                app.adjType = "cch-as-act";
            }
        }

        //【14】如果该进程的 adj 等于 SERVICE_ADJ，这边会对进程根据 Service A list 和 Service B list 做一个划分！
        if (adj == ProcessList.SERVICE_ADJ) {
        
            // doingAll 为 true 表示本次 update 更新的是所有进程的 adj，只有在空参数 update 调用的时候会为 true！
            if (doingAll) { 
            
                // 先判断该进程是否是 Service B list 进程
                app.serviceb = mNewNumAServiceProcs > (mNumServiceProcs / 3);
                
                mNewNumServiceProcs++; // 计算最新的服务进程数， + 1；

                //Slog.i(TAG, "ADJ " + app + " serviceb=" + app.serviceb);
    
                if (!app.serviceb) {
                
                    // This service isn't far enough down on the LRU list to
                    // normally be a B service, but if we are low on RAM and it
                    // is large we want to force it down since we would prefer to
                    // keep launcher over it.
                    // 如果不是 Service B list，但内存回收等级过高，也被视为 Service B list
                    if (mLastMemoryLevel > ProcessStats.ADJ_MEM_FACTOR_NORMAL
                            && app.lastPss >= mProcessList.getCachedRestoreThresholdKb()) {
                        app.serviceHighRam = true;
                        app.serviceb = true;
                        //Slog.i(TAG, "ADJ " + app + " high ram!");

                    } else {
                        // Service A list 类型的进程数 + 1；
                        mNewNumAServiceProcs++;
                        //Slog.i(TAG, "ADJ " + app + " not high ram!");

                    }

                } else {
                    app.serviceHighRam = false;
                }
            }

            if (app.serviceb) { 
                // 如果 serviceb 为 true，则该进程是 Service B list 中的！
                // 设置其 adj 为 SERVICE_B_ADJ！
                adj = ProcessList.SERVICE_B_ADJ;
            }
        }
        
        // 设置进程的 curRawAdj 为最终的计算结果！
        app.curRawAdj = adj;

        //Slog.i(TAG, "OOM ADJ " + app + ": pid=" + app.pid +
        //      " adj=" + adj + " curAdj=" + app.curAdj + " maxAdj=" + app.maxAdj);
        if (adj > app.maxAdj) {
            adj = app.maxAdj;
            if (app.maxAdj <= ProcessList.PERCEPTIBLE_APP_ADJ) {
                schedGroup = ProcessList.SCHED_GROUP_DEFAULT;
            }
        }

        // Do final modification to adj.  Everything we do between here and applying
        // the final setAdj must be done in this function, because we will also use
        // it when computing the final cached adj later.  Note that we don't need to
        // worry about this for max adj above, since max adj will always be used to
        // keep it out of the cached vaues.
        // 对上面计算出的 adj 做第二次修正计算，其返回值是进程的最终 adj！！
        app.curAdj = app.modifyRawOomAdj(adj);
        
        // 设置其他的一些变量属性！
        app.curSchedGroup = schedGroup;
        app.curProcState = procState;
        app.foregroundActivities = foregroundActivities;

        return app.curRawAdj;
    }
```

这里调用了 `modifyRawOomAdj` 方法，根据 `BIND_ABOVE_CLIENT`

```java
    int modifyRawOomAdj(int adj) {
        // hasAboveClient 为 true，表示该进程通过 BIND_ABOVE_CLIENT 方式绑定了 Service！
        // BIND_ABOVE_CLIENT 表示该进程的优先级已经超过了 Activity，也就是说当资源不够的时候 Activity 要比 Service 先死；
        // 所以需要降低该进程的 adj 优先级！
        if (hasAboveClient) {
            if (adj < ProcessList.FOREGROUND_APP_ADJ) {
                // System process will not get dropped, ever
    
            } else if (adj < ProcessList.VISIBLE_APP_ADJ) {
                adj = ProcessList.VISIBLE_APP_ADJ;
                
            } else if (adj < ProcessList.PERCEPTIBLE_APP_ADJ) {
                adj = ProcessList.PERCEPTIBLE_APP_ADJ;
                
            } else if (adj < ProcessList.CACHED_APP_MIN_ADJ) {
                adj = ProcessList.CACHED_APP_MIN_ADJ;
                
            } else if (adj < ProcessList.CACHED_APP_MAX_ADJ) {
                adj++;
            }
        }
        // 一般情况是直接返回第一次计算后的值的！
        return adj;
    }
```
`LRU list` 中，从后往前数，前 `1/3` 的 `service` 进程就是 `AService` 进程，其余的就是 `BService` 进程；
                

### 3.3.2  ActivityManagerS.applyOomAdjLocked

最后，`applyOomAdjLocked` 方法用来应用计算出的 `adj`！

```java
    private final boolean applyOomAdjLocked(ProcessRecord app, boolean doingAll, long now,
            long nowElapsed) {
        boolean success = true;

        //【1】更新 setRawAdj 的值！
        if (app.curRawAdj != app.setRawAdj) {
            app.setRawAdj = app.curRawAdj; // 更新 setRawAdj！
        }

        int changes = 0; // 用于记录进程的状态是否发生了变化！

        //【2】更新 setAdj 的值！
        if (app.curAdj != app.setAdj) {
            ProcessList.setOomAdj(app.pid, app.info.uid, app.curAdj); // 设置进程的最新 adj！
            if (DEBUG_SWITCH || DEBUG_OOM_ADJ) Slog.v(TAG_OOM_ADJ,
                    "Set " + app.pid + " " + app.processName + " adj " + app.curAdj + ": "
                    + app.adjType);
            app.setAdj = app.curAdj;  // 更新 setAdj！
            app.verifiedAdj = ProcessList.INVALID_ADJ;
        }

        //【3】更新 setSchedGroup 调度组信息！
        if (app.setSchedGroup != app.curSchedGroup) {
            int oldSchedGroup = app.setSchedGroup;
            app.setSchedGroup = app.curSchedGroup; // 更新 setSchedGroup！

            if (DEBUG_SWITCH || DEBUG_OOM_ADJ) Slog.v(TAG_OOM_ADJ,
                    "Setting sched group of " + app.processName
                    + " to " + app.curSchedGroup);
    
            if (app.waitingToKill != null && app.curReceiver == null
                    && app.setSchedGroup == ProcessList.SCHED_GROUP_BACKGROUND) {

                //【3.1】杀掉那些之前延迟 kill，现在不持有 receiver 且属于后台的进程！
                app.kill(app.waitingToKill, true);
                success = false;
    
            } else {

                //【3.2】根据调度组 curSchedGroup，设置进程组信息！
                int processGroup;
                switch (app.curSchedGroup) {
                    case ProcessList.SCHED_GROUP_BACKGROUND:
                        // 后台调度组: 6
                        processGroup = Process.THREAD_GROUP_BG_NONINTERACTIVE;
                        break;

                    case ProcessList.SCHED_GROUP_TOP_APP:
                    case ProcessList.SCHED_GROUP_TOP_APP_BOUND:
                        // 前台调度组：5
                        processGroup = Process.THREAD_GROUP_TOP_APP;
                        break;
        
                    default:
                        // 默认调度组：-1
                        processGroup = Process.THREAD_GROUP_DEFAULT;
                        break;

                }

                long oldId = Binder.clearCallingIdentity();

                try {
                    //【3.3】设置进程的调度组！
                    Process.setProcessGroup(app.pid, processGroup);
                    
                    //【3.4】根据调度组，设置
                    if (app.curSchedGroup == ProcessList.SCHED_GROUP_TOP_APP) {

                        // do nothing if we already switched to RT
                        //【3.3.1】如果之前进程的调度组不是 top app 级别，而现在的调度组是 top app 级别，进入这里！
                        if (oldSchedGroup != ProcessList.SCHED_GROUP_TOP_APP) {
                        
                            // Switch VR thread for app to SCHED_FIFO
                            if (mInVrMode && app.vrThreadTid != 0) {
                                try {
                                    // 如果进程启动了 VR    
                                    Process.setThreadScheduler(app.vrThreadTid,
                                        Process.SCHED_FIFO | Process.SCHED_RESET_ON_FORK, 1);
                                } catch (IllegalArgumentException e) {
                                    // thread died, ignore
                                }
                            }

                            if (mUseFifoUiScheduling) {

                                // Switch UI pipeline for app to SCHED_FIFO
                                app.savedPriority = Process.getThreadPriority(app.pid);

                                try {
                                    Process.setThreadScheduler(app.pid,
                                        Process.SCHED_FIFO | Process.SCHED_RESET_ON_FORK, 1);
                                } catch (IllegalArgumentException e) {
                                    // thread died, ignore
                                }

                                if (app.renderThreadTid != 0) {
                                    try {
                                        Process.setThreadScheduler(app.renderThreadTid,
                                            Process.SCHED_FIFO | Process.SCHED_RESET_ON_FORK, 1);
                                    } catch (IllegalArgumentException e) {
                                        // thread died, ignore
                                    }
                                    if (DEBUG_OOM_ADJ) {
                                        Slog.d("UI_FIFO", "Set RenderThread (TID " +
                                            app.renderThreadTid + ") to FIFO");
                                    }
                                } else {
                                    if (DEBUG_OOM_ADJ) {
                                        Slog.d("UI_FIFO", "Not setting RenderThread TID");
                                    }
                                }
                            } else {
                                // Boost priority for top app UI and render threads
                                Process.setThreadPriority(app.pid, -10);
                                if (app.renderThreadTid != 0) {
                                    try {
                                        Process.setThreadPriority(app.renderThreadTid, -10);
                                    } catch (IllegalArgumentException e) {
                                        // thread died, ignore
                                    }
                                }
                            }
                        }

                    } else if (oldSchedGroup == ProcessList.SCHED_GROUP_TOP_APP &&
                               app.curSchedGroup != ProcessList.SCHED_GROUP_TOP_APP) {
                    //【3.3.2】如果之前进程的调度组是 top app 级别，而现在的调度组不是 top app 级别，则进入这里！

                        // Reset VR thread to SCHED_OTHER
                        // Safe to do even if we're not in VR mode
                        // 进程的 vrThreadTid 不等于 0，说明我们在 VR 模式！
                        if (app.vrThreadTid != 0) {
                            Process.setThreadScheduler(app.vrThreadTid, Process.SCHED_OTHER, 0);
                        }

                        if (mUseFifoUiScheduling) {
                            // Reset UI pipeline to SCHED_OTHER
                            Process.setThreadScheduler(app.pid, Process.SCHED_OTHER, 0);
                            Process.setThreadPriority(app.pid, app.savedPriority);
                            if (app.renderThreadTid != 0) {
                                Process.setThreadScheduler(app.renderThreadTid,
                                    Process.SCHED_OTHER, 0);
                                Process.setThreadPriority(app.renderThreadTid, -4);
                            }

                        } else {
                            // Reset priority for top app UI and render threads
                            Process.setThreadPriority(app.pid, 0);
                            if (app.renderThreadTid != 0) {
                                Process.setThreadPriority(app.renderThreadTid, 0);
                            }
                        }
                    }
                } catch (Exception e) {
                    Slog.w(TAG, "Failed setting process group of " + app.pid
                            + " to " + app.curSchedGroup);
                    e.printStackTrace();
                } finally {
                    Binder.restoreCallingIdentity(oldId);
                }
            }
        }
        
        //【4】接下来是记录进程的信息和状态的变化！
        // 进程内部的 acitivity 的状态发生了变化！ProcessChangeItem.CHANGE_ACTIVITIES
        if (app.repForegroundActivities != app.foregroundActivities) {
            app.repForegroundActivities = app.foregroundActivities; // 更新 repForegroundActivities！
            changes |= ProcessChangeItem.CHANGE_ACTIVITIES;
        }

        //【5】进程状态发生了变化，通知上层应用进程！！
        if (app.repProcState != app.curProcState) {
            app.repProcState = app.curProcState; // 更新 repProcState！
            changes |= ProcessChangeItem.CHANGE_PROCESS_STATE;
            if (app.thread != null) {
                try {
                    if (false) {
                        //RuntimeException h = new RuntimeException("here");
                        Slog.i(TAG, "Sending new process state " + app.repProcState
                                + " to " + app /*, h*/);
                    }

                    //【5.1】通知上层进程！
                    app.thread.setProcessState(app.repProcState);
                } catch (RemoteException e) {
                }
            }
        }

        //【5】如果进程在先后状态下的内存状态不同，那就要更新下下次申请 PSS 的时间！
        if (app.setProcState == ActivityManager.PROCESS_STATE_NONEXISTENT
                || ProcessList.procStatesDifferForMem(app.curProcState, app.setProcState)) {
                
            if (false && mTestPssMode && app.setProcState >= 0 && app.lastStateTime <= (now - 200)) { // 用于 debug！
                long start = SystemClock.uptimeMillis();
                
                // 记录 PPS 物理内存！
                long pss = Debug.getPss(app.pid, mTmpLong, null);
                recordPssSampleLocked(app, app.curProcState, pss, mTmpLong[0], mTmpLong[1], now);

                // 从 mPendingPssProcesses 列表中删除当前进程！
                mPendingPssProcesses.remove(app);
                Slog.i(TAG, "Recorded pss for " + app + " state " + app.setProcState
                        + " to " + app.curProcState + ": "
                        + (SystemClock.uptimeMillis()-start) + "ms");
            }

            app.lastStateTime = now;

            //【5.1】更新下次申请 pss 内存的时间！
            app.nextPssTime = ProcessList.computeNextPssTime(app.curProcState, true,
                    mTestPssMode, isSleepingLocked(), now);

            if (DEBUG_PSS) Slog.d(TAG_PSS, "Process state change from "
                    + ProcessList.makeProcStateString(app.setProcState) + " to "
                    + ProcessList.makeProcStateString(app.curProcState) + " next pss in "
                    + (app.nextPssTime-now) + ": " + app);

        } else {
            //【5.2】如果当前时间已经超过了 nextPssTime 下一次申请 PSS 的时间
            // 那就要请求 PSS 物理内存！
            if (now > app.nextPssTime || (now > (app.lastPssTime + ProcessList.PSS_MAX_INTERVAL)
                    && now > (app.lastStateTime + ProcessList.minTimeFromStateChange(
                    mTestPssMode)))) {

                // 请求物理内存！
                requestPssLocked(app, app.setProcState);
                app.nextPssTime = ProcessList.computeNextPssTime(app.curProcState, false,
                        mTestPssMode, isSleepingLocked(), now);
            } else if (false && DEBUG_PSS) Slog.d(TAG_PSS,
                    "Not requesting PSS of " + app + ": next=" + (app.nextPssTime-now));

        }
        
        //【6】更新 setProcState 属性！
        if (app.setProcState != app.curProcState) {
            if (DEBUG_SWITCH || DEBUG_OOM_ADJ) Slog.v(TAG_OOM_ADJ,
                    "Proc state change of " + app.processName
                            + " to " + app.curProcState);

            // 计算进程状态变化前后的重要性！
            boolean setImportant = app.setProcState < ActivityManager.PROCESS_STATE_SERVICE;
            boolean curImportant = app.curProcState < ActivityManager.PROCESS_STATE_SERVICE;

            if (setImportant && !curImportant) {

                // 如果进程变的不重要了，需要记录下其 wake time，用于后续的 kill 操作！
                BatteryStatsImpl stats = mBatteryStatsService.getActiveStatistics();
                synchronized (stats) {
                    app.lastWakeTime = stats.getProcessWakeTime(app.info.uid,
                            app.pid, nowElapsed);
                }
                app.lastCpuTime = app.curCpuTime;

            }

            // 更新进程的使用情况！
            maybeUpdateUsageStatsLocked(app, nowElapsed);
            
            // 更新 setProcState！
            app.setProcState = app.curProcState;

            if (app.setProcState >= ActivityManager.PROCESS_STATE_HOME) {
                app.notCachedSinceIdle = false;
            }

            if (!doingAll) {
                setProcessTrackerStateLocked(app, mProcessStats.getMemFactorLocked(), now);
            } else {
                app.procStateChanged = true;
            }
            
        } else if (app.reportedInteraction && (nowElapsed - app.interactionEventTime)
                > USAGE_STATS_INTERACTION_INTERVAL) {

            // For apps that sit around for a long time in the interactive state, we need
            // to report this at least once a day so they don't go idle.
            maybeUpdateUsageStatsLocked(app, nowElapsed);
        }

        //【7】如果 changes 不等于 0，说明进程的状态信息发生了变化！
        // 我们需要将这次的改变封装为 mPendingProcessChanges，通知那些监控进程变化的服务！
        if (changes  != 0) {

            if (DEBUG_PROCESS_OBSERVERS) Slog.i(TAG_PROCESS_OBSERVERS,
                    "Changes in " + app + ": " + changes);
                    
            // 判断该进程是否已经有一个 ProcessChangeItem！
            int i = mPendingProcessChanges.size()-1;
            ProcessChangeItem item = null;

            while (i >= 0) {
                item = mPendingProcessChanges.get(i);
                if (item.pid == app.pid) {
                    if (DEBUG_PROCESS_OBSERVERS) Slog.i(TAG_PROCESS_OBSERVERS,
                            "Re-using existing item: " + item);
                    break;
                }
                i--;
            }

            // i < 0 说明该进程没有已存在的等待处理的 ProcessChangeItem 项，那就创建一个新的！
            if (i < 0) {
                final int NA = mAvailProcessChanges.size();
                // 尝试从 mAvailProcessChanges 中回收复用一个 ProcessChangeItem 项！
                // 不能回收，就创建一个新的！
                if (NA > 0) {
                    item = mAvailProcessChanges.remove(NA-1);
                    if (DEBUG_PROCESS_OBSERVERS) Slog.i(TAG_PROCESS_OBSERVERS,
                            "Retrieving available item: " + item);
                } else {
                    item = new ProcessChangeItem();
                    if (DEBUG_PROCESS_OBSERVERS) Slog.i(TAG_PROCESS_OBSERVERS,
                            "Allocating new item: " + item);
                }
                // 初始化变量！
                item.changes = 0;
                item.pid = app.pid;
                item.uid = app.info.uid;

                if (mPendingProcessChanges.size() == 0) {
                    if (DEBUG_PROCESS_OBSERVERS) Slog.i(TAG_PROCESS_OBSERVERS,
                            "*** Enqueueing dispatch processes changed!");
                    mUiHandler.obtainMessage(DISPATCH_PROCESSES_CHANGED_UI_MSG).sendToTarget();
                }

                // 添加到 mPendingProcessChanges 列表中！
                mPendingProcessChanges.add(item);
            }
            
            // 设置 ProcessChangeItem 的属性！
            item.changes |= changes;
            item.processState = app.repProcState;
            item.foregroundActivities = app.repForegroundActivities;
            
            if (DEBUG_PROCESS_OBSERVERS) Slog.i(TAG_PROCESS_OBSERVERS,
                    "Item " + Integer.toHexString(System.identityHashCode(item))
                    + " " + app.toShortString() + ": changes=" + item.changes
                    + " procState=" + item.processState
                    + " foreground=" + item.foregroundActivities
                    + " type=" + app.adjType + " source=" + app.adjSource
                    + " target=" + app.adjTarget);
        }

        return success;
    }

```



# 4 总结

首先，我们知道对于进程的优先级，`Android` 有 `2` 套分级机制：

 - `ActivityManagerService` 中对进程优先级的分级，以 `PROCESS_STATE_` 开头，定义在 `ActivityManager.java` 中;
 - `LowMemoryKiller` 通过 `oomAdj` 对进程优先级的分级;






    //开始逆序处理LRU中的每一个进程
    // 对应重要性大于home的进程而言，重要性越高，内存回收等级越低
    // 对于重要性小于home的进程，排在LRU表越靠后，即越重要回收等级越高
    // 这么安排的理由有两个：1、此时越不重要的进程，其中运行的组件越少，能够回收的内存不多，不需要高回收等级
    // 2、越不重要的进程越有可能被LMK kill掉，没必要以高等级回收内存


Android 并不是 kill 掉所有 Empty 进程后，才 kill 后台进程。 
它是将 CACHED_APP_MIN_ADJ 和 CACHED_APP_MAX_ADJ 之间的范围，分成 3 个 slot。 

然后在每个 slot 中，分别分配一定量的后台进程和 Empty 进程。 
在单独的 slot 中，会先 kill 掉 empty 进程，后 kill 掉后台进程。 
只有当一个 slot 中的进程 kill 完毕后，才会 kill 掉下一个 slot 中的进程。 
我们将从后面的代码中，得到对应的分析依据，这里先有个印象即可。


  
  http://blog.csdn.net/Gaugamela/article/details/54176460
  
  
  http://www.cnblogs.com/tiger-wang-ms/p/6445213.html
 