# Service 篇 2 - startService 流程分析
title: Service 篇 2 - startService 流程分析
date: 2016/03/01 20:46:25
catalog: true
categories:
- AndroidFramework源码分析
- Service服务
tags: Service服务
---

基于 `Android 7.1.1` 源码分析 `startService` 的流程，本文为作者原创，转载请说明出处！


# 0 综述

我们在应用中经常会启动 `Service`：

```java
    startService(intent);
```

这个方法最终会拉起 `Service` 的 `onStartCommand` 方法：
```java
    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        OppoLog.d(TAG, "onStartCommand");
        return super.onStartCommand(intent, flags, startId);
    }
```
该方法有三个参数：

- **Intent intent**: 启动服务的 `Intent`！
- **int flags**: 启动时的额外参数，取值可以为 `0`，`START_FLAG_REDELIVERY` 和 `START_FLAG_RETRY`！
- **int startId**: 当前服务的唯一 `ID`，和 `StopSelfResult(int startId)` 配合使用！

**1、**下面我们来看看 `onStartCommand` 的 `flag` 参数：

</br>

- **START_FLAG_REDELIVERY** 
   - **取值**：`x0001`
   - **解释**：如果 `Service` 的 `onStartCommand` 返回值是 `START_REDELIVER_INTENT`，当服务被杀掉，服务会重启，这时 `flag` 会被传入这个值！

</br>

- **START_FLAG_RETRY**
   - **取值**：`0x0002`
   - **解释**：当 `onStartCommand` 方法调用后一直没有返回时，会尝试重新去调用 `onStartCommand` 方法，这时 `flag` 会被传入这个值！

**2、**而 `onStartCommand` 有如下的返回值，这里先简单的介绍下：

</br>

- **START_STICKY_COMPATIBILITY** 
   - **取值**：`0`
- **START_STICKY**
   - **取值**：`1`
   - **说明**：
      - 如果 `Service` 进程启动后被杀掉了，服务被启动的状态会被保留并抛弃本次启动的 `Intent`，然后系统会尝试重新创建和启动一个新的服务实例!
      - 如果在重启期间，没有任何新的启动项 `Intent` 传递给 `Service`，那么会传递一个空的 `Intent` 对象，所以在 `onStartCommand` 方法中要做 `Intent` 非空的判断！ 
   - **用途**：
       - 比较适用于不执行命令、但无限期运行并等待作业的媒体播放器或类似服务。

</br>

- **START_NOT_STICKY**
   - **取值**：`2`
   - **说明**：
       - 如果 `Service` 进程启动后被杀掉了，并且没有新的启动 `Intent` 分发给服务，那么会移除服务被启动的状态，也不会重新创建服务，直到下一次显示地通过 `startService` 启动服务！
       - 这种情况下，`onStartCommand` 不会传入空的 `Intent` 对象（因为在没有正在等待分发的 Intent 的情况下，该服务不会重启）！
   - **用途**：
       - 该模式比较适用于启动后需要做一些工作，但是在内存不够的情况下可以被停止，然后通过显示地再次启动去继续工作的情况！
       - 一个简单的例子：从服务器获得数据的服务，通过设置一个 `alarm`，间隔一定时间启动服务`onStartCommand`，然后设置新的 `alarm`，如果服务进程被杀掉，那服务不会被触发，知道下一次 `alarm` 触发！

</br>

- **START_REDELIVER_INTENT** 
   - **取值**： `3`
   - **说明**：
       - 如果 `Service` 进程启动后被杀掉了，该 `Service` 将会被重启，并且会将最后启动（`startService`）分发过的 `Intent` 再次通过 `onStartCommand` 方法传递给 `Service` ，该 `Intent` 将会被保留用于下一次的重启分发，除非 `Service` 调用 `stopSelf(int startId)` 方法停止运行！
       - 这种情况下，`onStartCommand` 也不会传入空的 `Intent` 对象，因为服务只有在没有完成处理所有分发给它的 `Intent` 的情况下或重启，一旦重启，就会将最后启动（`startService`）分发过的 `Intent` 再次通过 `onStartCommand` 方法传递给服务！
   - **用途**：
       - 这个模式适用于主动执行应该立即恢复的作业（例如下载文件）的服务。

</br>

- **START_CONTINUATION_MASK** 0xf


这里是 `Android` 文档中的对这些参数的做的简单说明，下面我们来分析 startService 的过程！

# 1 启动端进程

## 1.1 ContextWrapper.startService
```java
    @Override
    public ComponentName startService(Intent service) {
        return mBase.startService(service);
    }
    
    /** @hide */
    @Override
    public ComponentName startServiceAsUser(Intent service, UserHandle user) {
        return mBase.startServiceAsUser(service, user);
    }
```

`ContextWrapper` 提供了两个方法来启动 `Service`，其中一个是隐藏方法：`startServiceAsUser`！

`mBase` 是 `ContextImpl` 对象，继续看！

## 1.2 ContextImpl.startService
```java
class ContextImpl extends Context {

    @Override
    public ComponentName startService(Intent service) {
        warnIfCallingFromSystemProcess();
        return startServiceCommon(service, mUser);
    }
    
    @Override
    public ComponentName startServiceAsUser(Intent service, UserHandle user) {
        return startServiceCommon(service, user);
    }
}
```

`ContextImpl` 和 `ContextWrapper` 的具体关系，请看博文，这里我们不再详细说明！

`mUser` 表示的是当前的设备 `user`！

最终，调用了 `startServiceCommon` 方法；

## 1.3 ContextImpl.startServiceCommon
```java
    private ComponentName startServiceCommon(Intent service, UserHandle user) {
        try {
            
            // 如果系统版本不低于 L，那就要抛出异常，提示必须是显示启动！
            validateServiceIntent(service);
            service.prepareToLeaveProcess(this);
            
            //【1】启动指定的 Service！
            ComponentName cn = ActivityManagerNative.getDefault().startService(
                mMainThread.getApplicationThread(), service, service.resolveTypeIfNeeded(
                            getContentResolver()), getOpPackageName(), user.getIdentifier());

            if (cn != null) {
                if (cn.getPackageName().equals("!")) {
                    throw new SecurityException(
                            "Not allowed to start service " + service
                            + " without permission " + cn.getClassName());
                } else if (cn.getPackageName().equals("!!")) {
                    throw new SecurityException(
                            "Unable to start service " + service
                            + ": " + cn.getClassName());
                }
            }
            return cn;
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
```
`validateServiceIntent` 方法用用来校验启动用的 `intent` 是否安全：

### 1.3.1 ContextImpl.validateServiceIntent

```java
    private void c(Intent service) {
        if (service.getComponent() == null && service.getPackage() == null) {
            if (getApplicationInfo().targetSdkVersion >= Build.VERSION_CODES.LOLLIPOP) {
                IllegalArgumentException ex = new IllegalArgumentException(
                        "Service Intent must be explicit: " + service);
                throw ex;
            } else {
                Log.w(TAG, "Implicit intents with startService are not safe: " + service
                        + " " + Debug.getCallers(2, 3));
            }
        }
    }
```
如果 `intent` 既没有设置 `component` 也没有设置 `package`，那就要判断一下系统的版本了: 

  -  如果系统版本不低于 `L`，那就要抛出异常，提示必须是显示启动；
  -  如果系统版本低于 `L`，那只提示隐式启动不安全；

这里可以看出，`Intent` 必须要设置 `component` 或者 `package` 中的一个！


### 1.3.2 ContextImpl.getOpPackageName

通过 `getOpPackageName` 来获得启动者所在的包名：
```java
    /** @hide */
    @Override
    public String getOpPackageName() {
        return mOpPackageName != null ? mOpPackageName : getBasePackageName();
    }
```
这里的 `mOpPackageName` 是在创建 `ContextImpl` 的时候初始化的！

## 1.4 ActivityManagerP.startService

接着，通过 `ActivityManagerP.startService` 方法，启动服务！
```java
    public ComponentName startService(IApplicationThread caller, Intent service,
         String resolvedType, String callingPackage, int userId) throws RemoteException
    {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IActivityManager.descriptor);

        data.writeStrongBinder(caller != null ? caller.asBinder() : null);

        service.writeToParcel(data, 0);
        data.writeString(resolvedType);
        data.writeString(callingPackage);
        data.writeInt(userId);
        mRemote.transact(START_SERVICE_TRANSACTION, data, reply, 0);
        reply.readException();

        ComponentName res = ComponentName.readFromParcel(reply);

        data.recycle();
        reply.recycle();

        return res;
    }

```
通过 `binder` 进程间通信，进入系统进程，参数分析：

- `IApplicationThread caller`： 调用者进程的 `ApplicationThread` 对象，实现了 `IApplicationThread` 接口；
- `Intent service`： 启动的 `intent`
- `String resolvedType`： 这个 `intent` 的 `MIME` 类型；
- `String callingPackage`： 启动者所属包名；
- `int userId`： 设备用户 `id`；


# 2 系统进程

接下来，进入系统进程的 `ActivityManagerService` 中！

首先要进入 `ActivityManagerN.onTransact` 方法！
```java
        case START_SERVICE_TRANSACTION: {
            data.enforceInterface(IActivityManager.descriptor);
            
            IBinder b = data.readStrongBinder();
            IApplicationThread app = ApplicationThreadNative.asInterface(b);

            Intent service = Intent.CREATOR.createFromParcel(data);
            String resolvedType = data.readString();
            String callingPackage = data.readString();
            int userId = data.readInt();
            
            //【1】继续 startService！
            ComponentName cn = startService(app, service, resolvedType, callingPackage, userId);

            reply.writeNoException();
            ComponentName.writeToParcel(cn, reply);
            return true;
        }
```

## 2.1 ActivityManagerS.startService
```java
    @Override
    public ComponentName startService(IApplicationThread caller, Intent service,
            String resolvedType, String callingPackage, int userId)
            throws TransactionTooLargeException {
        
        // 用来校验启动者进程是否是隔离的，如果是隔离进程，抛出异常！！
        enforceNotIsolatedCaller("startService");

        // 不能通过 intent 传递文件描述符，否则抛出非法异常！
        if (service != null && service.hasFileDescriptors() == true) {
            throw new IllegalArgumentException("File descriptors passed in Intent");
        }
        
        // 调用者 package 为 null，抛出异常！
        if (callingPackage == null) {
            throw new IllegalArgumentException("callingPackage cannot be null");
        }

        if (DEBUG_SERVICE) Slog.v(TAG_SERVICE,
                "startService: " + service + " type=" + resolvedType);

        synchronized(this) {

            // 获得调用者的 uid 和 pid！
            final int callingPid = Binder.getCallingPid();
            final int callingUid = Binder.getCallingUid();
            final long origId = Binder.clearCallingIdentity();
            
            //【1】进入 ActiveServices，继续启动！
            ComponentName res = mServices.startServiceLocked(caller, service,
                    resolvedType, callingPid, callingUid, callingPackage, userId);

            Binder.restoreCallingIdentity(origId);
            return res;
        }
    }
```
`mServices` 是 `ActivityManagerService` 的一个内部管理对象，用于管理所有的 `Service`！

## 2.2 ActiveServices.startServiceLoced

参数和前面保持一致！
```java
    ComponentName startServiceLocked(IApplicationThread caller, Intent service, String resolvedType,
            int callingPid, int callingUid, String callingPackage, final int userId)
            throws TransactionTooLargeException {

        if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE, "startService: " + service
                + " type=" + resolvedType + " args=" + service.getExtras());
        
        //【1】判断是否是前台调用！
        final boolean callerFg;
        if (caller != null) {
            
            // 获得调用者进程的 ProcessRecord 对象！
            final ProcessRecord callerApp = mAm.getRecordForAppLocked(caller);
            if (callerApp == null) {
                throw new SecurityException(
                        "Unable to find app for caller " + caller
                        + " (pid=" + Binder.getCallingPid()
                        + ") when starting service " + service);

            }
            callerFg = callerApp.setSchedGroup != ProcessList.SCHED_GROUP_BACKGROUND;
        } else {
            callerFg = true;
        }

        //【2】检索要启动的服务的信息！
        ServiceLookupResult res =
            retrieveServiceLocked(service, resolvedType, callingPackage,
                    callingPid, callingUid, userId, true, callerFg, false);

        if (res == null) {
            return null;
        }

        if (res.record == null) {
            return new ComponentName("!", res.permission != null
                    ? res.permission : "private to package");
        }

        //【3】获得要启动的服务的数据对象：ServiceRecord！
        ServiceRecord r = res.record;
        
        // 如果服务所属的设备用户不存在，直接返回！
        if (!mAm.mUserController.exists(r.userId)) {
            Slog.w(TAG, "Trying to start service with non-existent user! " + r.userId);
            return null;
        }

        //【4】如果服务还没有被请求启动，要先判断服务是否允许在后台启动，如果不允许就直接返回！
        if (!r.startRequested) {
            final long token = Binder.clearCallingIdentity();
            try {

                final int allowed = mAm.checkAllowBackgroundLocked(
                        r.appInfo.uid, r.packageName, callingPid, true);

                if (allowed != ActivityManager.APP_START_MODE_NORMAL) {
                    Slog.w(TAG, "Background start not allowed: service "
                            + service + " to " + r.name.flattenToShortString()
                            + " from pid=" + callingPid + " uid=" + callingUid
                            + " pkg=" + callingPackage);

                    return null;
                }

            } finally {
                Binder.restoreCallingIdentity(token);
            }
        }

        NeededUriGrants neededGrants = mAm.checkGrantUriPermissionFromIntentLocked(
                callingUid, r.packageName, service, service.getFlags(), null, r.userId);

        //【5】如果启动者属于前台应用，并且启动服务组件是需要校验权限，就需要弹出权限校验界面！
        // 只有权限校验成功，才会继续启动！
        if (Build.PERMISSIONS_REVIEW_REQUIRED) {
            if (!requestStartTargetPermissionsReviewIfNeededLocked(r, callingPackage,
                    callingUid, service, callerFg, userId)) {
                return null;
            }
        }

        //【6】取消服务重启的任务！
        if (unscheduleServiceRestartLocked(r, callingUid, false)) {
            if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "START SERVICE WHILE RESTART PENDING: " + r);
        }

        //【7】设置服务 ServiceRecord 的属性值；
        r.lastActivity = SystemClock.uptimeMillis();
        // 设置 startRequested 为 true，表示被请求启动！
        r.startRequested = true;
        // 设置 delayedStop 为 false；表示服务没有被延迟停止，只对后台服务有效！
        r.delayedStop = false;
        
        // 将本次启动需要的数据，包括 intent，封装成 StartItem，保存到 pendingStarts 列表中！
        r.pendingStarts.add(new ServiceRecord.StartItem(r, false, r.makeNextStartId(),
                service, neededGrants));
        
        // 获得 userId 下的所有 Service 的 Map 集合！
        final ServiceMap smap = getServiceMap(r.userId);

        // 表示是否将服务添加到后台启动列表中；
        boolean addToStarting = false;

        //【8】对于后台启动的服务，需要做一些条件判断，看是延迟启动，还是立刻后台启动！
        if (!callerFg && r.app == null
                && mAm.mUserController.hasStartedUserState(r.userId)) {
            
            // 获得被启动的服务所在进程的 ProcessRecord 数据对象！
            ProcessRecord proc = mAm.getProcessRecordLocked(r.processName, r.appInfo.uid, false);
            
            // 下面判断当前启动服务的进程状态来确定是否需要延时启动这个服务！
            // 对于非前台的启动，尝试延迟启动服务！ 
            if (proc == null || proc.curProcState > ActivityManager.PROCESS_STATE_RECEIVER) {
                if (DEBUG_DELAYED_SERVICE) Slog.v(TAG_SERVICE, "Potential start delay of "
                        + r + " in " + proc);

                if (r.delayed) {

                    // 如果该服务已经延迟启动，说明他已经被加入到了 smap.mDelayedStartList 列表中等待启动，直接返回！
                    if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE, "Continuing to delay: " + r);
                    return r.name;
                }
                
                // 当同一时间正在后台启动的服务数超过了最大后台启动服务数，延迟启动！
                if (smap.mStartingBackground.size() >= mMaxStartingBackground) {
                    Slog.i(TAG_SERVICE, "Delaying start of: " + r);
                    
                    // 就要将该服务加入 smap.mDelayedStartList 中；
                    smap.mDelayedStartList.add(r);
                    // 设置其 r.delayed 为 true！
                    r.delayed = true;

                    return r.name;
                }
                
                if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE, "Not delaying: " + r);
                
                addToStarting = true;

            } else if (proc.curProcState >= ActivityManager.PROCESS_STATE_SERVICE) {

                addToStarting = true;

                if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE,
                        "Not delaying, but counting as bg: " + r);

            } else if (DEBUG_DELAYED_STARTS) { // 和 debug 相关，只是输出一些 log 信息！

                StringBuilder sb = new StringBuilder(128);
                sb.append("Not potential delay (state=").append(proc.curProcState)
                        .append(' ').append(proc.adjType);
                String reason = proc.makeAdjReason();
                if (reason != null) {
                    sb.append(' ');
                    sb.append(reason);
                }
                sb.append("): ");
                sb.append(r.toString());
                Slog.v(TAG_SERVICE, sb.toString());
            }

        } else if (DEBUG_DELAYED_STARTS) {
        
            // 和 debug 相关，只是输出一些 log 信息！
            if (callerFg) {
                Slog.v(TAG_SERVICE, "Not potential delay (callerFg=" + callerFg + " uid="
                        + callingUid + " pid=" + callingPid + "): " + r);

            } else if (r.app != null) {
                Slog.v(TAG_SERVICE, "Not potential delay (cur app=" + r.app + "): " + r);

            } else {
                Slog.v(TAG_SERVICE,
                        "Not potential delay (user " + r.userId + " not started): " + r);

            }
        }

        //【9】进一步调用！
        return startServiceInnerLocked(smap, service, r, callerFg, addToStarting);
    }

```
该方法的主要流程：

- 判断是前台启动，还是后台启动；
- 检索需要启动的服务的信息；
- 如果不满足启动条件就取消本次启动；
- 创建本次启动对应的启动项；
- 对于后台启动的服务，需要判读是否延迟启动；
- 进一步启动服务！

如何判断后台启动的服务是否需要延迟启动呢，依据如下：

这个地方会对进程的状态做一个判断：

- 如果是前台进程的调度，就直接进行启动；
- 如果是后台进程的调度，就要先判断一下，是否延迟执行；

对于如何判断进程是否是前台进程，还是后台进程，我会在另外一篇文章中说明！！

### 2.2.1 ActiveServices.unscheduleServiceRestartLocked

这里有一个方法，取消上一次的重启任务，参数传入：

- boolean force：传入 false！
```java
    private final boolean unscheduleServiceRestartLocked(ServiceRecord r, int callingUid,
            boolean force) {

        if (!force && r.restartDelay == 0) {
            return false;
        }

        //【1】将服务从重启列表中移除！
        boolean removed = mRestartingServices.remove(r);
        if (removed || callingUid != r.appInfo.uid) {
            
            // 清空 restartCount，restartDelay，restartTime！
            r.resetRestartCounter();
        }
        if (removed) {

            // 让 restartTracker 对 ServiceRecord 重新监控！
            clearRestartingIfNeededLocked(r);
        }
        //【2】移除重启任务！
        mAm.mHandler.removeCallbacks(r.restarter);

        return true;
    }
```
这里有一个数据结构：r.restarter，他是在创建 ServiceRecord 时传入的，他是一个 ServiceRestarter 对象！
```java
    private class ServiceRestarter implements Runnable {

        private ServiceRecord mService;

        void setService(ServiceRecord service) {
            mService = service;
        }

        public void run() {
            synchronized(mAm) {
                
                //【1】执行重启操作！
                performServiceRestartLocked(mService);
            }
        }
    }
```

我们再去看看 performServiceRestartLocked 方法，是如何实现服务的重启的！
```java
    final void performServiceRestartLocked(ServiceRecord r) {
    
        //【1】mRestartingServices 没有当前服务，无法执行重启！
        if (!mRestartingServices.contains(r)) {
            return;
        }
        
        //【2】不需要重启服务！
        if (!isServiceNeeded(r, false, false)) {
            Slog.wtf(TAG, "Restarting service that is not needed: " + r);
            return;
        }
        try {
        
            //【3】再次拉起服务！！
            bringUpServiceLocked(r, r.intent.getIntent().getFlags(), r.createdFromFg, true, false);

        } catch (TransactionTooLargeException e) {
            // Ignore, it's been logged and nothing upstack cares.
        }
    }

```

用于执行重启操作，这里我们只是简单地看看！

## 2.3 ActiveServices.startServiceInnerLocked

接下来，进一步地启动服务，如果 addToStarting 为 true，表示该服务是后台启动的，需要将其添加到 mStartingBackground 集合中！

```java
    ComponentName startServiceInnerLocked(ServiceMap smap, Intent service, ServiceRecord r,
            boolean callerFg, boolean addToStarting) throws TransactionTooLargeException {

        ServiceState stracker = r.getTracker();
        if (stracker != null) {
            
            // 通知服务的 tracker 对象，开始监控服务!
            stracker.setStarted(true, mAm.mProcessStats.getMemFactorLocked(), r.lastActivity);
        }
        
        // 表示是否被启动了，在启动前初始化为 false！
        r.callStart = false;

        synchronized (r.stats.getBatteryStats()) {
            r.stats.startRunningLocked();
        }
         
        //【1】拉起服务，这个方法会拉起服务的 onCreate 和 onStartCommand 方法！
        String error = bringUpServiceLocked(r, service.getFlags(), callerFg, false, false);

        if (error != null) {
            return new ComponentName("!!", error);
        }
        
        //【2】如果服务已经被请求启动（startRequested 为 true），且是后台启动（addToStarting 为 true）
        if (r.startRequested && addToStarting) {
            
            boolean first = smap.mStartingBackground.size() == 0;

            // 将服务添加到 mStartingBackground 列表中，表示正在后台启动！
            smap.mStartingBackground.add(r);

            // 设置后台启动服务的超时时间！
            r.startingBgTimeout = SystemClock.uptimeMillis() + BG_START_TIMEOUT;

            if (DEBUG_DELAYED_SERVICE) {

                RuntimeException here = new RuntimeException("here");
                here.fillInStackTrace();
                Slog.v(TAG_SERVICE, "Starting background (first=" + first + "): " + r, here);

            } else if (DEBUG_DELAYED_STARTS) {
                Slog.v(TAG_SERVICE, "Starting background (first=" + first + "): " + r);
            }
            
            //【2.1】如果 first 为 true，表示这是第一个后台启动的服务！那就需要初始化延迟启动的任务调度！
            if (first) {
                smap.rescheduleDelayedStarts();
            }

        } else if (callerFg) {
            //【2.2】如果是前台启动该服务，就取消该服务之前的后台运行任务！
            smap.ensureNotStartingBackground(r);
        }

        return r.name;
    }
```
对于前台启动的方式：`callerFg` 为 `true`，`addToStarting` 为 `false`，就会执行 `ensureNotStartingBackground` 取消服务后台启动！
对于后台启动的方式：`callerFg` 为 `false`，`addToStarting` 为 `true`，就会将服务添加到 `mStartingBackground` 集合中！

### 2.3.1 ServiceMap.rescheduleDelayedStart

对于后台启动的方式，如果 `mStartingBackground.size() == 0`，那就需要对延迟启动的调度做一次初始化，我们先来看看 `rescheduleDelayedStarts` 方法！

```java
        void rescheduleDelayedStarts() {
            // 移除 MSG_BG_START_TIMEOUT 后台启动超时消息！
            removeMessages(MSG_BG_START_TIMEOUT);
            final long now = SystemClock.uptimeMillis();
            
            //【1】对于后台启动的服务，如果后台启动超时，就从移除该服务！ 为延迟后台启动的服务腾出位置！
            for (int i=0, N=mStartingBackground.size(); i<N; i++) {
                ServiceRecord r = mStartingBackground.get(i);
                if (r.startingBgTimeout <= now) {
                    Slog.i(TAG, "Waited long enough for: " + r);
                    mStartingBackground.remove(i);
                    N--;
                    i--;
                }
            }
            
            //【2】处理后台延迟启动的服务，如果此时延迟启动列表不为空，且正在后台启动的服务数不超过最大后台启动数
            // 那这个时候就要处理后台延迟服务的启动了！
            while (mDelayedStartList.size() > 0
                    && mStartingBackground.size() < mMaxStartingBackground) {
                
                //【2.1】从 mDelayedStartList 移除该服务，保存到 r 中！
                ServiceRecord r = mDelayedStartList.remove(0);
                
                if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE,
                        "REM FR DELAY LIST (exec next): " + r);
                if (r.pendingStarts.size() <= 0) {
                    Slog.w(TAG, "**** NO PENDING STARTS! " + r + " startReq=" + r.startRequested
                            + " delayedStop=" + r.delayedStop);
                }

                if (DEBUG_DELAYED_SERVICE) {
                    if (mDelayedStartList.size() > 0) {
                        Slog.v(TAG_SERVICE, "Remaining delayed list:");
                        for (int i=0; i<mDelayedStartList.size(); i++) {
                            Slog.v(TAG_SERVICE, "  #" + i + ": " + mDelayedStartList.get(i));
                        }
                    }
                }
                
                // 将 r.delayed 设为 false；
                r.delayed = false;

                try {
                    
                    //【2.2】后台启动该服务（callerFg 为 false，addStarting 为 true）
                    // 服务会被添加到 mStartingBackground 列表中！
                    startServiceInnerLocked(this, r.pendingStarts.get(0).intent, r, false, true);

                } catch (TransactionTooLargeException e) {
                }
            }
            
            //【3】设置后台启动的超时处理！
            if (mStartingBackground.size() > 0) {
        
                // 设置后台超时消息的发送的时间 when 为 startingBgTimeout 和 now 中的最大值！
                ServiceRecord next = mStartingBackground.get(0);
                long when = next.startingBgTimeout > now ? next.startingBgTimeout : now;

                if (DEBUG_DELAYED_SERVICE) Slog.v(TAG_SERVICE, "Top bg start is " + next
                        + ", can delay others up to " + when);

                Message msg = obtainMessage(MSG_BG_START_TIMEOUT);
                sendMessageAtTime(msg, when);
            }
            
            //【4】发送那些等待后台服务启动的后台广播！
            if (mStartingBackground.size() < mMaxStartingBackground) {
                mAm.backgroundServicesFinishedLocked(mUserId);
            }
        }
    }
```
我们看到，对于延迟后台启动的服务，最后又会调用 `startServiceInnerLocked` 后台启动，然后，该服务会被添加到 `mStartingBackground` 列表中，表示正在启动，这个我们后面会分析到！

该方法的主要作用是：

- 移除 `mStartingBackground` 中已经启动超时的服务；
- 如果后台启动的服务数小于最大允许后台启动的服务数，且 `mDelayedStartList` 中有延迟启动的服务，就立刻启动并移除 `mDelayedStartList` 中的服务，然后将其添加到 `mStartingBackground` 列表中！
- 设置后台启动的超时处理！
- 发送那些等待后台服务启动的后台广播！

</br>

这里要简单的说一下 **ServiceMap**：

`ServiceMap` 是 `Handler` 的子类，其内部封装了指定设备用户所有的服务信息对象，还有后台启动的服务对象列表 `mStartingBackground`，以及延迟后台启动的服务对象列表 `mDelayedStartList`！

其本身是 `Handler`，用于处理 `MSG_BG_START_TIMEOUT`消息：
```java
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case MSG_BG_START_TIMEOUT: {
                    synchronized (mAm) {
                        //【1】
                        rescheduleDelayedStarts();
                    }
                } break;
            }
        }
```
`ServiceMap` 收到该消息后，会再次触发 `rescheduleDelayedStart` 方法，该内部又会发送 `MSG_BG_START_TIMEOUT` 消息，这样就会不断的循环发送 `MSG_BG_START_TIMEOUT` 消息，不断的处理  `mDelayedStartList` 和 `mStartingBackground`
列表，保证所有的后台延迟启动的服务能够即使启动，同时设置后台启动的超时处理！

### 2.3.2 ServiceMap.ensureNotStartingBackground

对于前台启动的方式，就需要更新 `mStartingBackground` 和 `mDelayedStartList` 集合中的元素了，尝试将当前的服务从这两个集合中删除，因为这里是前台启动，我们继续看：
```java
        void ensureNotStartingBackground(ServiceRecord r) {

            //【1】从 mStartingBackground 中删除当前服务；
            if (mStartingBackground.remove(r)) {
                if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE,
                        "No longer background starting: " + r);

                //【1.1】再次启动后台延迟服务的调度！
                rescheduleDelayedStarts();
            }

            //【2】从 mDelayedStartList 中删除当前服务；
            if (mDelayedStartList.remove(r)) {
                if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE, "No longer delaying start: " + r);
            }
        }
```

首先是尝试从 `mStartingBackground` 中删除当前服务，如果删除成功，就要再次调用 `rescheduleDelayedStarts` 方法，尝试后台启动被延迟后台启动的服务，并再次发送 `MSG_BG_START_TIMEOUT` 消息！

### 2.3.3 阶段总结

对于立刻后台启动的服务 `r`，启动时，都会被添加到 `mStartingBackground` 列表中，并且设置 `r.startingBgTimeout` 后台启动超时时间为 `now + 15s`；每个后台启动的服务 `r`，当其 `r.startingBgTimeout` 时间到了后，会从 `mStartingBackground` 列表中删除；

对于延迟后台启动的服务 `r`，都会添加到 `mDelayedStartList` 列表中，并且设置 `r.delayed` 为 `true`，每个延迟后台启动的服务的启动都依赖于一个条件：`mStartingBackground.size() < mMaxStartingBackground`；

通过上面的分析，我们可以知道 `rescheduleDelayedStarts` 和 `ensureNotStartingBackground` 作用是什么了:

就是不断处理 `mStartingBackground` 中的服务，不断的根据 `mStartingBackground` 中服务的 `r.startingBgTimeout` 发送 `MSG_BG_START_TIMEOUT` 消息，不断的处理该消息，删除对应的服务，使得 `mStartingBackground.size() < mMaxStartingBackground`，为 `mDelayedStartList` 中的服务腾出位置，保证这些延迟后台启动的服务能够后台启动！

## 2.4 ActiveServices.bringUpServiceLocked

下面我们来看看，系统进程是如何拉起 Serivce 的，传入参数分析：

- **ServiceRecord r**：服务的 ServiceRecord 对象；
- **int intentFlags**：启动服务的 intent 的 flag，通过 getFlags 获得；
- **boolean execInFg**：传入 callFg，表示本次启动是前台调用还是后台调用；
- **boolean whileRestarting**：表示是否是正在重启，传入 false；
- **boolean permissionsReviewRequired**：false；

```java
    private String bringUpServiceLocked(ServiceRecord r, int intentFlags, boolean execInFg,
            boolean whileRestarting, boolean permissionsReviewRequired)
            throws TransactionTooLargeException {

        //【1】如果 r.app 和 r.app.thread 都不为 null。说明不仅服务所在进程已经被启动，服务也已经被创建（onCreate）；
        // 那就直接拉起 onStartCommand() 方法！这里我们先假设服务所在的进程还没有被启动！
        if (r.app != null && r.app.thread != null) {

            sendServiceArgsLocked(r, execInFg, false);
            return null;
        }

        //【2】如果服务正在等待重启，就退出！
        if (!whileRestarting && r.restartDelay > 0) {
            return null;
        }

        if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "Bringing up " + r + " " + r.intent);

        //【3】如果服务是被启动了，所以要从 mRestartingServices 列表中移除它，并清除内部的启动计数！
        if (mRestartingServices.remove(r)) {
            r.resetRestartCounter();
            clearRestartingIfNeededLocked(r);
        }

        //【4】如果这个服务被延迟后台启动，即 delayed 为 true，那就从 mDelayedStartList 中移除它，
        // 并置 r.delayed 为 false；
        if (r.delayed) {
            if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE, "REM FR DELAY LIST (bring up): " + r);
            getServiceMap(r.userId).mDelayedStartList.remove(r);
            r.delayed = false;
        }

        // 如果这个服务所在的设备用户没有被启动，那就不允许启动这个服务！
        if (!mAm.mUserController.hasStartedUserState(r.userId)) {
            String msg = "Unable to launch app "
                    + r.appInfo.packageName + "/"
                    + r.appInfo.uid + " for service "
                    + r.intent.getIntent() + ": user " + r.userId + " is stopped";
            Slog.w(TAG, msg);
            
            // 停止这个服务！
            bringDownServiceLocked(r);
            return msg;
        }

        try {
            // 服务将要被启动，所以要设置其 package 的停止状态为 false；
            AppGlobals.getPackageManager().setPackageStoppedState(
                    r.packageName, false, r.userId);

        } catch (RemoteException e) {
        } catch (IllegalArgumentException e) {
            Slog.w(TAG, "Failed trying to unstop package "
                    + r.packageName + ": " + e);
        }

        // 判断这个服务是否属于隔离进程；
        final boolean isolated = (r.serviceInfo.flags&ServiceInfo.FLAG_ISOLATED_PROCESS) != 0;

        //【5】获得要启动的服务的目标进程名和进程对象；
        final String procName = r.processName;
        ProcessRecord app;

        if (!isolated) {

            // 对于非隔离进程，先获得所在进程的 ProcessRecord 对象；
            app = mAm.getProcessRecordLocked(procName, r.appInfo.uid, false);
            
            if (DEBUG_MU) Slog.v(TAG_MU, "bringUpServiceLocked: appInfo.uid=" + r.appInfo.uid
                        + " app=" + app);

            if (app != null && app.thread != null) { // 如果进程已经启动；
                try {

                    app.addPackage(r.appInfo.packageName, r.appInfo.versionCode, mAm.mProcessStats);

                    //【5.1】启动指定服务！
                    realStartServiceLocked(r, app, execInFg);

                    return null;
                } catch (TransactionTooLargeException e) {
                    throw e;
                } catch (RemoteException e) {
                    Slog.w(TAG, "Exception when starting service " + r.shortName, e);
                }

            }

        } else {

            // 对于隔离进程，获得服务之前所在的进程！
            app = r.isolatedProc;
        }

        // 如果服务所在的进程没有启动！
        if (app == null && !permissionsReviewRequired) {
        
            //【6】首先要启动服务所在的进程，这里请去看进程启动和创建的博文！
            if ((app=mAm.startProcessLocked(procName, r.appInfo, true, intentFlags,
                    "service", r.name, false, isolated, false)) == null) {
                String msg = "Unable to launch app "
                        + r.appInfo.packageName + "/"
                        + r.appInfo.uid + " for service "
                        + r.intent.getIntent() + ": process is bad";
                Slog.w(TAG, msg);

                bringDownServiceLocked(r); // 进程启动失败；

                return msg;
            }
            
            // 如果服务要运行在隔离进程中，就把创建的 ProcessRecord 保存到 r.isolatedProc 中！
            if (isolated) {
                r.isolatedProc = app;
            }
        }
        
        //【7】将这个服务加入到 mPendingServices 中，表示该服务正在其所在的进程启动；
        if (!mPendingServices.contains(r)) {
            mPendingServices.add(r);
        }

        if (r.delayedStop) {

            // Oh and hey we've already been asked to stop!
            r.delayedStop = false;
            if (r.startRequested) {
                if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE,
                        "Applying delayed stop (in bring up): " + r);
                stopServiceLocked(r);
            }
        }

        return null;
    }
```

我们来总结一下，这个方法的流程：

> 第一阶段： 

- 如果服务所在的进程已经启动，就调用 `sendServiceArgsLocked`，通过 `ApplicationThread` 拉起 `onStartCommand` 的方法，**结束流程**；
- 如果服务所在的进程没有被启动，进入下面的流程：

> 第二阶段： 

- 如果服务正在等待重启，不处理，**结束流程**；
- 否则，进入下面的流程：

> 第三阶段： 

- 当前服务要被启动了，就从等待重启 `mRestartingServices` 中移除它；
- 如果这个服务是被延迟启动（`delayed` 的值为 `true`）的，就从 `mDelayedStartList` 中移除它，并置其 `delayed` 为 `false`；
- 当前服务将要被启动，所以要设置其 `package` 的停止状态为 `false`；
- 判断这个进程是否是隔离进程：
  - 如果是非隔离进程，根据进程名和 `uid`，获得其所在进程的 `ProcessRecord` 对象，如果进程已经启动，就启动指定服务，**结束流程**；
  - 如果是隔离进程，获得服务之前所在的隔离进程的 `ProcessRecord` 对象；

> 第四阶段： 

- 根据获得的 `ProcessRecord` 对象，判断，如果之前服务所在进程没有启动，就需要先启动进程；
- 将服务添加到 `mPendingServices` 中，表示正在启动该服务；

之前我们有说，我们假设要启动的服务所在的进程没有被启动，那么就要先启动目标进程，具体的启动过程，大家可以去看我的其他几篇博客：
**Android 进程的创建和启动**

这里我们直接用之前的结果，进入 `AMS.attachApplicationLocked` 方法！

## 2.5 ActivityManagerS.attachApplicationLocked

当应用进程启动后，会通过 Binder 通信，调用 AMS.attachApplicationLocked，我们去看看：
```java
    private final boolean attachApplicationLocked(IApplicationThread thread,
            int pid) {

            ... ... ... ...
            
            // 这里会回到应用进程中，创建 Applicaiton 对象，并调用其 onCreate 方法，这里我们先不说！
            thread.bindApplication(processName, appInfo, providers, app.instrumentationClass,
                    profilerInfo, app.instrumentationArguments, app.instrumentationWatcher,
                    app.instrumentationUiAutomationConnection, testMode,
                    mBinderTransactionTrackingEnabled, enableTrackAllocation,
                    isRestrictedBackupMode || !normalMode, app.persistent,
                    new Configuration(mConfiguration), app.compat,
                    getCommonServicesLocked(app.isolated),
                    mCoreSettingsObserver.getCoreSettingsLocked());
            
            // 更新进程 LRU 队列！
            updateLruProcessLocked(app, false, null);

            app.lastRequestedGc = app.lastLowMemory = SystemClock.uptimeMillis();
        } catch (Exception e) {
            app.resetPackageList(mProcessStats);
            app.unlinkDeathRecipient();

            // 每当 bind 操作失败，则重新启动进程, 此处有可能会导致进程无限重启。
            startProcessLocked(app, "bind fail", processName);
            return false;
        }

        // 将当前已经被启动的新进程的 ProcessRecord 从正在启动的进程集合 mPersistentStartingProcesses
        // 和等待启动的进程集合 mProcessesOnHold 中移除！
        mPersistentStartingProcesses.remove(app);
        mProcessesOnHold.remove(app);

        boolean badApp = false;
        boolean didSomething = false;

        ... ... ... ...

        if (!badApp) {
            try {

                 // 检查是否有 sevvce 组件要在这个新进程中运行！
                didSomething |= mServices.attachApplicationLocked(app, processName);
            } catch (Exception e) {
                Slog.wtf(TAG, "Exception thrown starting services in " + app, e);
                badApp = true;
            }
        }

        ... ... ... ...

        if (badApp) { // badApp 为 true，就杀掉这个进程
            app.kill("error during init", true);
            handleAppDiedLocked(app, false, true);
            
            return false;
        }

        if (!didSomething) { // 检查后发现，没有任何组件要运行在新进程，更新 OomAdj 的值！
            updateOomAdjLocked();
        }

        return true;
    }
```
`bindApplication` 方法会回到应用进程中，创建 `Applicaiton` 对象，并调用其 `onCreate` 方法，这里我们不看！

这里会调用 `ActiveServices` 的 `attachApplicationLocked` 方法！


## 2.6 ActiveServices.attachApplicationLocked

当应用进程启动成功，并且 Application 对象已经创建，其 onCreate 方法已经调用后，就要启动指定的服务了！
```java
    boolean attachApplicationLocked(ProcessRecord proc, String processName)
            throws RemoteException {

        boolean didSomething = false;
        
        //【1】启动该进程中所有的服务！
        if (mPendingServices.size() > 0) {

            ServiceRecord sr = null;
            try {

                // 遍历 mPendingServices 集合中的所有服务，启动属于这个进程的所有服务！
                for (int i=0; i<mPendingServices.size(); i++) {
                    sr = mPendingServices.get(i);

                    if (proc != sr.isolatedProc && (proc.uid != sr.appInfo.uid
                            || !processName.equals(sr.processName))) {
                        continue;
                    }

                    mPendingServices.remove(i);

                    i--;
                    
                    // 将服务的包名，版本号以及 AMS 的进程状态对象保存到进程对象中！
                    proc.addPackage(sr.appInfo.packageName, sr.appInfo.versionCode,
                            mAm.mProcessStats);
                    
                    //【1】启动服务，这里又回到了之前的方法中！
                    realStartServiceLocked(sr, proc, sr.createdFromFg);

                    didSomething = true;

                    if (!isServiceNeeded(sr, false, false)) { // 如果已经不需要启动了，就取消启动！

                        bringDownServiceLocked(sr);
                    }
                }
            } catch (RemoteException e) {
                Slog.w(TAG, "Exception in new application when starting service "
                        + sr.shortName, e);
                throw e;
            }
        }

        //【2】接着启动那些需要重启的服务，重启的服务都会保存到 mRestartingServices 服务中！
        // 因为重启的时间可能还没有到，所以这里并不是立刻启动它们，而是启动了一个任务，交给了 AMS 的 MainHandler 去做！
        if (mRestartingServices.size() > 0) {
            ServiceRecord sr;
            for (int i=0; i<mRestartingServices.size(); i++) {

                sr = mRestartingServices.get(i);
                if (proc != sr.isolatedProc && (proc.uid != sr.appInfo.uid
                        || !processName.equals(sr.processName))) {
                    continue;
                }
                
                // sr.restarter 是一个 ServiceRestarter 对象，是一个 Runnable，这里会执行重启！
                mAm.mHandler.removeCallbacks(sr.restarter);
                mAm.mHandler.post(sr.restarter);
            }
        }

        return didSomething;
    }
```
这里有一个方法 `isServiceNeeded` 是否需要启动服务：
```java
    private final boolean isServiceNeeded(ServiceRecord r, boolean knowConn, boolean hasConn) {
        if (r.startRequested) {
            return true;
        }

        if (!knowConn) {
            hasConn = r.hasAutoCreateConnections();
        }
        if (hasConn) {
            return true;
        }

        return false;
    }
```
判断依据很简单:

- `r.startRequested` 为 `true`，说明服务被请求启动，那么就需要；
- `r.hasAutoCreateConnections` 为 `true`；说明有应用已经和该服务建立了自动创建的连接，那么就需要；

我们发现无论进程是否已经被创建，最终都会调用 `realStartServiceLocked` 方法！

## 2.7 AS.realStartServiceLocked

通过调用 realStartServiceLocked 方法来启动进程中的指定服务：
```java
    private final void realStartServiceLocked(ServiceRecord r,
            ProcessRecord app, boolean execInFg) throws RemoteException {

        if (app.thread == null) {
            throw new RemoteException();
        }

        if (DEBUG_MU)
            Slog.v(TAG_MU, "realStartServiceLocked, ServiceRecord.uid = " + r.appInfo.uid
                    + ", ProcessRecord.uid = " + app.uid);
        
        // 设置服务的进程对象；
        r.app = app;
        // 设置服务的启动和活跃时间；
        r.restartTime = r.lastActivity = SystemClock.uptimeMillis();
        
        // 将被启动服务的 ServiceRecord 对象添加到所属进程的 app.services 中！
        final boolean newService = app.services.add(r);
        
        //【1】针对 onCreate 方法，设置超时处理！
        bumpServiceExecutingLocked(r, execInFg, "create");
 
        // 更新 LruProcess 和 OomAdj！
        mAm.updateLruProcessLocked(app, false, null);
        mAm.updateOomAdjLocked();

        boolean created = false;

        try {

            if (LOG_SERVICE_START_STOP) { // log 相关！
                String nameTerm;
                int lastPeriod = r.shortName.lastIndexOf('.');
                nameTerm = lastPeriod >= 0 ? r.shortName.substring(lastPeriod) : r.shortName;
                EventLogTags.writeAmCreateService(
                        r.userId, System.identityHashCode(r), nameTerm, r.app.uid, r.app.pid);
            }

            synchronized (r.stats.getBatteryStats()) {
                r.stats.startLaunchedLocked();
            }

            mAm.notifyPackageUse(r.serviceInfo.packageName,
                                 PackageManager.NOTIFY_PACKAGE_USE_SERVICE);

            // 将进程的状态更新为 ActivityManager.PROCESS_STATE_SERVICE！
            app.forceProcessStateUpTo(ActivityManager.PROCESS_STATE_SERVICE);
            
            //【2】通过 binder 通信，拉起服务的 onCreate 方法！
            app.thread.scheduleCreateService(r, r.serviceInfo,
                    mAm.compatibilityInfoForPackageLocked(r.serviceInfo.applicationInfo),
                    app.repProcState);
            
            // 如果服务通过 setForeground 方法设置了通知，就显示通知！
            r.postNotification();

            created = true;

        } catch (DeadObjectException e) {
            Slog.w(TAG, "Application dead when creating service " + r);
            mAm.appDiedLocked(app); // 如果在服务启动时，应用进程死了，调用 appDiedLocked 通知死亡仆告对象！
            throw e;

        } finally {
            if (!created) { // 如果启动失败，并且不死一个被销毁的服务，那就尝试重启他！

                final boolean inDestroying = mDestroyingServices.contains(r);
                serviceDoneExecutingLocked(r, inDestroying, inDestroying);

                if (newService) {
                    app.services.remove(r);
                    r.app = null;
                }

                if (!inDestroying) {
                    scheduleServiceRestartLocked(r, false);
                }
            }
        }

        if (r.whitelistManager) {
            app.whitelistManager = true;
        }
        
        // 拉起 onBind 相关的方法，但是这里是不会触发的，因为 r.bindings 集合只有在 bindService 才会有元素！
        // 但是，如果之前 bindService 的标志位设置的不是 Context.BIND_AUTO_CREATE，而是其他参数，那么其 r.bindings
        // 是有元素的，这里才会调用服务的 onBind 方法，为之前的 bindService 返回代理对象！
        requestServiceBindingsLocked(r, execInFg);

        updateServiceClientActivitiesLocked(app, null, true);

        // 如果服务被请求启动，但是其内部的启动项为 0，那就创建一个启动项，保证 Service 的 onStartCommand 可以被调用；
        if (r.startRequested && r.callStart && r.pendingStarts.size() == 0) {
            r.pendingStarts.add(new ServiceRecord.StartItem(r, false, r.makeNextStartId(),
                    null, null));
        }
        
        //【2】通过 binder 通信，拉起服务的 onStartCommand 方法！
        sendServiceArgsLocked(r, execInFg, true);

        // 如果是延迟启动的话，在这里将其移除！
        if (r.delayed) {
            if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE, "REM FR DELAY LIST (new proc): " + r);
            getServiceMap(r.userId).mDelayedStartList.remove(r);
            r.delayed = false;
        }

        if (r.delayedStop) {
            // Oh and hey we've already been asked to stop!
            r.delayedStop = false;
            if (r.startRequested) {
                if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE,
                        "Applying delayed stop (from start): " + r);
                stopServiceLocked(r);

            }
        }
    }

```
**这里要注意**：

方法 `requestServiceBindingsLocked`，一般是不会触发的，因为 `r.bindings` 集合只有在 `bindService` 的情况才会被添加元素！但是，如果之前 `bindService` 的标志位没有设置 `Context.BIND_AUTO_CREATE`，而是其他标志位，那么其 `r.bindings` 是有元素的，这里就会调用服务的 `onBind` 方法，为之前的 `bindService` 返回代理对象！

### 2.7.1 AS.bumpServiceExecutingLocked

针对 `onCreate` 方法，设置超时处理，参数 `fg` 表示这个服务是前台执行，还是后台执行的！
```java
    private final void bumpServiceExecutingLocked(ServiceRecord r, boolean fg, String why) {

        if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, ">>> EXECUTING "
                + why + " of " + r + " in app " + r.app);
        else if (DEBUG_SERVICE_EXECUTING) Slog.v(TAG_SERVICE_EXECUTING, ">>> EXECUTING "
                + why + " of " + r.shortName);

        long now = SystemClock.uptimeMillis();

        if (r.executeNesting == 0) { // executeNesting 用来记录是否有超时处理操作！

            r.executeFg = fg;
            ServiceState stracker = r.getTracker();
            if (stracker != null) {

                // 对该服务进行内存监控处理！
                stracker.setExecuting(true, mAm.mProcessStats.getMemFactorLocked(), now);
            }
            
            if (r.app != null) {
                
                // 将服务添加到 ProcessRecord 的 executingServices 集合中，表示服务正在执行某段代码！
                r.app.executingServices.add(r);
                r.app.execServicesFg |= fg;
                
                // 如果正在执行指定函数的服务个数为 1，设置超时处理！
                if (r.app.executingServices.size() == 1) {

                    scheduleServiceTimeoutLocked(r.app);
                }

            }
        } else if (r.app != null && fg && !r.app.execServicesFg) {

            r.app.execServicesFg = true;
            scheduleServiceTimeoutLocked(r.app);
        }

        r.executeFg |= fg;
        r.executeNesting++;
        
        // 记录启动时间！
        r.executingStart = now;
    }
```
我们来看看超市处理的操作！

#### 2.7.1.1 AS.scheduleServiceTimeoutLocked
```java
    void scheduleServiceTimeoutLocked(ProcessRecord proc) {
        
        // 如果进程没有正在执行某段代码的服务就返回！
        if (proc.executingServices.size() == 0 || proc.thread == null) {
            return;
        }
        
        long now = SystemClock.uptimeMillis();
        Message msg = mAm.mHandler.obtainMessage(
                ActivityManagerService.SERVICE_TIMEOUT_MSG);
        msg.obj = proc;

        mAm.mHandler.sendMessageAtTime(msg,
                proc.execServicesFg ? (now+SERVICE_TIMEOUT) : (now+ SERVICE_BACKGROUND_TIMEOUT));
    }
```
可以看到：

- 对于前台服务：超时时间是 20s
- 对于后台服务：超市时间是 200s

最后发送 SERVICE_TIMEOUT_MSG 消息到 AMS 的 MainHandler 中！

#### 2.7.1.2 ActivityServiceS.MainHandler
```java
    final class MainHandler extends Handler {
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {

            ... ... ... ...
            
            case SERVICE_TIMEOUT_MSG: {
                if (mDidDexOpt) { // 表示是否延迟执行 odex
                    mDidDexOpt = false;
                    Message nmsg = mHandler.obtainMessage(SERVICE_TIMEOUT_MSG);
                    nmsg.obj = msg.obj;
                    mHandler.sendMessageDelayed(nmsg, ActiveServices.SERVICE_TIMEOUT);
                    return;
                }
                
                // 调用 serviceTimeout，设置超时处理！
                mServices.serviceTimeout((ProcessRecord)msg.obj);
            } break;

            ... ... ... ...

            }
        }
    }
```
接着进入 ActiveServices 中！

#### 2.7.1.3 ActiveServices.serviceTimeout
```java
    void serviceTimeout(ProcessRecord proc) {
        String anrMessage = null;

        synchronized(mAm) {
            if (proc.executingServices.size() == 0 || proc.thread == null) {
                return;
            }
            final long now = SystemClock.uptimeMillis();
            final long maxTime =  now -
                    (proc.execServicesFg ? SERVICE_TIMEOUT : SERVICE_BACKGROUND_TIMEOUT);

            ServiceRecord timeout = null;
            long nextTime = 0;
            
            // 从正在执行代码逻辑的服务列表中，获得判断是否超时的服务！
            for (int i=proc.executingServices.size()-1; i>=0; i--) {
                ServiceRecord sr = proc.executingServices.valueAt(i);
                
                if (sr.executingStart < maxTime) {

                    // 服务启动已经超时，timeout 不为 null；
                    timeout = sr;
                    break;
                }
                if (sr.executingStart > nextTime) {
                    nextTime = sr.executingStart;
                }
            }

            if (timeout != null && mAm.mLruProcesses.contains(proc)) {
                Slog.w(TAG, "Timeout executing service: " + timeout);
                
                // 启动服务超时了，进入这个分支!
                StringWriter sw = new StringWriter();
                PrintWriter pw = new FastPrintWriter(sw, false, 1024);
                pw.println(timeout);
                timeout.dump(pw, "    ");
                pw.close();
                mLastAnrDump = sw.toString();
                mAm.mHandler.removeCallbacks(mLastAnrDumpClearer);
                mAm.mHandler.postDelayed(mLastAnrDumpClearer, LAST_ANR_LIFETIME_DURATION_MSECS);
                anrMessage = "executing service " + timeout.shortName;

            } else {
            
                // 继续发送超时消息，进行下一轮超时判断！
                Message msg = mAm.mHandler.obtainMessage(
                        ActivityManagerService.SERVICE_TIMEOUT_MSG);

                msg.obj = proc;
                mAm.mHandler.sendMessageAtTime(msg, proc.execServicesFg
                        ? (nextTime+SERVICE_TIMEOUT) : (nextTime + SERVICE_BACKGROUND_TIMEOUT));
            }
        }

        if (anrMessage != null) {
            
            // 捕捉到 ANR！
            mAm.mAppErrors.appNotResponding(proc, null, null, false, anrMessage);
        }
    }

```
我们可以看到，判断启动服务是否超时：不断的给 AMS 的 MainHandler 发送  SERVICE_TIMEOUT_MSG 消息，不断的判断时间是否超时！

如果超时的话，会触发 AppErrors.appNotResponding 方法，这个我们以后再看！



### 2.7.2 ApplicationThreadP.scheduleCreateService

拉起其 onCreate 方法，参数说明：

- IBinder token：服务的 ServiceRecord 对象，实现了 IBinder 接口！
- erviceInfo info：服务的组件信息；
- CompatibilityInfo compatInfo：
- int processState：服务所在进程的状态；
```java
class ApplicationThreadProxy implements IApplicationThread {

    public final void scheduleCreateService(IBinder token, ServiceInfo info,
            CompatibilityInfo compatInfo, int processState) throws RemoteException {

        Parcel data = Parcel.obtain();
        data.writeInterfaceToken(IApplicationThread.descriptor);
        data.writeStrongBinder(token);
        info.writeToParcel(data, 0);
        compatInfo.writeToParcel(data, 0);
        data.writeInt(processState);

        try {
        
            // 通过 binder 通信！
            mRemote.transact(SCHEDULE_CREATE_SERVICE_TRANSACTION, data, null,
                    IBinder.FLAG_ONEWAY);

        } catch (TransactionTooLargeException e) {
            Log.e("CREATE_SERVICE", "Binder failure starting service; service=" + info);
            throw e;
        }

        data.recycle();
    }
    
}

```
binder 通信，进入应用进程！


### 2.7.3 AS.sendServiceArgsLocked

拉起其 onStartCommand 方法：

参数传递：

- ServiceRecord r：被启动的服务的数据对象，它实现了 Binder 接口，可以跨进程传递；
- boolean execInFg：callFg，表示本次启动是前台调用还是后台调用；
- oomAdjusted：传入 false；

```java
    private final void sendServiceArgsLocked(ServiceRecord r, boolean execInFg,
            boolean oomAdjusted) throws TransactionTooLargeException {
        
        // 还记得 pendingStarts 吗，在前面启动的时候，
        // 会将 intent 等等的数据封装成 StartItem，保存到 pendingStarts 中；
        final int N = r.pendingStarts.size();

        if (N == 0) { // 如果 N 为 null，说明没有启动操作；
            return;
        }

        while (r.pendingStarts.size() > 0) { // 遍历要启动服务 ServiceRecord 的 pendingStarts！
            Exception caughtException = null;
            ServiceRecord.StartItem si = null;

            try {
                // 将要处理的启动项从 pendingStarts 中移除！
                si = r.pendingStarts.remove(0);

                if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "Sending arguments to: "
                        + r + " " + r.intent + " args=" + si.intent);

                if (si.intent == null && N > 1) {
                    // 如果有启动项的 intent 为 null，且启动项目总数大于 1，那就会忽视掉这个 intent！
                     continue;
                }

                // 将要处理的启动项添加到 deliveredStarts 中，表示启动项已经分发处理！
                si.deliveredTime = SystemClock.uptimeMillis();
                r.deliveredStarts.add(si);

                // 增加启动项目的 deliveryCount 计数，表示这个启动项的处理次数！
                si.deliveryCount++;

                if (si.neededGrants != null) {
                    mAm.grantUriPermissionUncheckedFromIntentLocked(si.neededGrants,
                            si.getUriPermissionsLocked());
                }
                
                // 为 onStartCommand 方法设置超时任务！
                bumpServiceExecutingLocked(r, execInFg, "start");

                if (!oomAdjusted) { // 更新服务所在进程的 abj 值；
                    oomAdjusted = true;
                    mAm.updateOomAdjLocked(r.app);
                }

                int flags = 0;
                if (si.deliveryCount > 1) {
                    flags |= Service.START_FLAG_RETRY;
                }
                if (si.doneExecutingCount > 0) {
                    flags |= Service.START_FLAG_REDELIVERY;
                }
                
                // thread 是 ApplicationThreadProxy 对象；
                r.app.thread.scheduleServiceArgs(r, si.taskRemoved, si.id, flags, si.intent);

            } catch (TransactionTooLargeException e) {

                if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "Transaction too large: intent="
                        + si.intent);
                caughtException = e;

            } catch (RemoteException e) {

                // Remote process gone...  we'll let the normal cleanup take care of this.
                if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "Crashed while sending args: " + r);
                caughtException = e;

            } catch (Exception e) {
                Slog.w(TAG, "Unexpected exception", e);
                caughtException = e;

            }

            if (caughtException != null) {

                // Keep nesting count correct
                final boolean inDestroying = mDestroyingServices.contains(r);
                serviceDoneExecutingLocked(r, inDestroying, inDestroying);
                if (caughtException instanceof TransactionTooLargeException) {
                    throw (TransactionTooLargeException)caughtException;
                }
                break;
            }
        }
    }
```

可以看到，如果服务所在的进程已经被启动了，那就直接调用 `sendServiceArgsLocked` 方法，通过服务所属进程在系统进程中的 `Binder` 对象 `ApplicationThreadProxy`，通过 `Binder` 间通信，调用应用进程的 `ApplicationThread` 的 `scheduleServiceArgs` 对象，拉起指定服务的 `onStartCommand` 方法；

#### 2.7.3.1 ApplicationThreadP.scheduleServiceArgs

拉起 onStartCommand 方法，参数说明：

- `IBinder token`：服务的 `ServiceRecord` 对象，实现了 `IBinder` 接口！
- `boolean taskRemoved`：启动项的 `taskRemoved`；
- `int startId`：启动项的 `id`，用来表示本次启动的 `id` 值！
- `int flags`：
- `Intent args`：启动项用的 `intent`，用来启动 `Service`！

```java
class ApplicationThreadProxy implements IApplicationThread {

    public final void scheduleServiceArgs(IBinder token, boolean taskRemoved, int startId,
            int flags, Intent args) throws RemoteException {

        Parcel data = Parcel.obtain();
        data.writeInterfaceToken(IApplicationThread.descriptor);
        data.writeStrongBinder(token);
        data.writeInt(taskRemoved ? 1 : 0);
        data.writeInt(startId);
        data.writeInt(flags);

        if (args != null) {
            data.writeInt(1);
            args.writeToParcel(data, 0);
        } else {
            data.writeInt(0);
        }
        
        // binder 通信，进入到应用进程
        mRemote.transact(SCHEDULE_SERVICE_ARGS_TRANSACTION, data, null,
                IBinder.FLAG_ONEWAY);

        data.recycle();
    }

}
```
binder 通信，进入应用进程！

# 3 被启动者进程

在之前的进程启动分析中，当进程被拉起后，应用进程会创建进程的 `ActivityThread` 对象，用于托管进程的主线程，组件信息，`ApplicationThread` 对象等等的信息！


首先会进入 `ApplicationThreadNative.onTransact` 方法！
```java
/** {@hide} */
public abstract class ApplicationThreadNative extends Binder
        implements IApplicationThread {
        
    @Override
    public boolean onTransact(int code, Parcel data, Parcel reply, int flags)
            throws RemoteException {
        switch (code) {
        
            // 拉起 onCreate 方法！
            case SCHEDULE_CREATE_SERVICE_TRANSACTION: {
                data.enforceInterface(IApplicationThread.descriptor);

                IBinder token = data.readStrongBinder();
                ServiceInfo info = ServiceInfo.CREATOR.createFromParcel(data);
                CompatibilityInfo compatInfo = CompatibilityInfo.CREATOR.createFromParcel(data);
                int processState = data.readInt();
                
                // 继续来看！
                scheduleCreateService(token, info, compatInfo, processState);

                return true;
            }
        
            // 拉起 onStartCommand 方法！
            case SCHEDULE_SERVICE_ARGS_TRANSACTION:
            {
                data.enforceInterface(IApplicationThread.descriptor);

                IBinder token = data.readStrongBinder();
                boolean taskRemoved = data.readInt() != 0;
                int startId = data.readInt();
                int fl = data.readInt();
                Intent args;
                if (data.readInt() != 0) {
                    args = Intent.CREATOR.createFromParcel(data);
                } else {
                    args = null;
                }

                // 继续来看！
                scheduleServiceArgs(token, taskRemoved, startId, fl, args);

                return true;
            }
    
    }

}
```
接下来，进入 binder 通信服务端 ApplicationThread！

## 3.1 ApplicationThread.scheduleCreateService
```java
private class ApplicationThread extends ApplicationThreadNative {

        public final void scheduleCreateService(IBinder token,
                ServiceInfo info, CompatibilityInfo compatInfo, int processState) {

            updateProcessState(processState, false);
            CreateServiceData s = new CreateServiceData();
            s.token = token;
            s.info = info;
            s.compatInfo = compatInfo;
            
            // 发送 H.CREATE_SERVICE 到主线程的 Handler!
            sendMessage(H.CREATE_SERVICE, s);
        }

}
```
可以看到，最后会调用 sendMessage 方法，将消息  H.CREATE_SERVICE 发送给进程的主线程！

## 3.2 ApplicationThread.scheduleServiceArgs
```java
private class ApplicationThread extends ApplicationThreadNative {

        public final void scheduleServiceArgs(IBinder token, boolean taskRemoved, int startId,
            int flags ,Intent args) {

            ServiceArgsData s = new ServiceArgsData();
            s.token = token;
            s.taskRemoved = taskRemoved;
            s.startId = startId;
            s.flags = flags;
            s.args = args;

            // 发送 H.SERVICE_ARGS 到主线程的 Handler!
            sendMessage(H.SERVICE_ARGS, s);
        }

}
```
可以看到，最后会调用 sendMessage 方法，将消息  H.SERVICE_ARGS 发送给进程的主线程！

## 3.3 ApplicationThread.H
```java
private class H extends Handler {

    public void handleMessage(Message msg) {
    
        case CREATE_SERVICE:
            Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER,
                                        ("serviceCreate: " + String.valueOf(msg.obj)));
            handleCreateService((CreateServiceData)msg.obj);
              
            Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
            break;
              
        case SERVICE_ARGS:
            Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, 
                                        ("serviceStart: " + String.valueOf(msg.obj)));
            handleServiceArgs((ServiceArgsData)msg.obj);
            
            Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
            break;              
    }
    
}
```
这里省略了其他消息的处理！

最后分别调用：handleCreateService 和 handleServiceArgs 拉起指定的方法！

### 3.3.1 ApplicationThread.handleCreateService
```java
    private void handleCreateService(CreateServiceData data) {
        
        // 如果此时准备要 GC，那就跳过本次 GC！
        unscheduleGcIdler();
        
        // 获得应用程序的加载信息，先会从缓存中获取，找不到才创建！
        LoadedApk packageInfo = getPackageInfoNoCheck(
                data.info.applicationInfo, data.compatInfo);

        Service service = null;
        try {
        
            // 通过反射创建要启动的服务的实例！
            java.lang.ClassLoader cl = packageInfo.getClassLoader();
            service = (Service) cl.loadClass(data.info.name).newInstance();

        } catch (Exception e) {
            if (!mInstrumentation.onException(service, e)) {
                throw new RuntimeException(
                    "Unable to instantiate service " + data.info.name
                    + ": " + e.toString(), e);
            }
        }

        try {
            if (localLOGV) Slog.v(TAG, "Creating service " + data.info.name);
            
            // 创建该 Serivce Context 对象！
            ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
            context.setOuterContext(service);
            
            // 返回该进程的 Application 对象！
            Application app = packageInfo.makeApplication(false, mInstrumentation);

            // 将服务实例和 context，Application 进行绑定！
            service.attach(context, this, data.info.name, data.token, app,
                    ActivityManagerNative.getDefault());

            // 调用了服务的 onCreate 方法！
            service.onCreate();
            
            // 将创建的服务实例保存到 AT 的托管集合中！
            mServices.put(data.token, service);

            try {
 
                // 通知 AMS，操作完成；
                ActivityManagerNative.getDefault().serviceDoneExecuting(
                        data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);

            } catch (RemoteException e) {
                throw e.rethrowFromSystemServer();
            }

        } catch (Exception e) {

            if (!mInstrumentation.onException(service, e)) {
                throw new RuntimeException(
                    "Unable to create service " + data.info.name
                    + ": " + e.toString(), e);
            }
        }
    }

```
接着，进入了 Service 的 onCreate 方法中：


#### 3.3.1.1 service.onCreate
```java
public abstract class Service extends ContextWrapper implements ComponentCallbacks2 {

    /**
     * @hide
     */
    public final void attach(
            Context context,
            ActivityThread thread, String className, IBinder token,
            Application application, Object activityManager) {

        attachBaseContext(context);
        mThread = thread;           // NOTE:  unused - remove?
        mClassName = className;
        
        // 这个 mToken 在系统进程对应着的是服务的 ServiceRecord 对象
        mToken = token;
        mApplication = application;
        mActivityManager = (IActivityManager)activityManager;
        mStartCompatibility = getApplicationInfo().targetSdkVersion
                < Build.VERSION_CODES.ECLAIR;
    }
    
    public void onCreate() {
        // 由于用户实现具体的调用内容！
    }

    
}
```
注意这个 mToken， 它在系统进程对应着的是服务的 ServiceRecord 对象，之前在系统进程通过 ApplicationThreadP 传递对象的时候，会将 ServiceRecord 通过 binder 通信传递过来！


### 3.3.2 ApplicationThread.handleServiceArgs
```java
    private void handleServiceArgs(ServiceArgsData data) {

        // 获得之前创建的服务对象！
        Service s = mServices.get(data.token);
        if (s != null) {
            try {
                if (data.args != null) {
                    data.args.setExtrasClassLoader(s.getClassLoader());
                    data.args.prepareToEnterProcess();
                }

                int res;
                
                if (!data.taskRemoved) { // 如果 data.taskRemoved 为 false，就拉起 onStartCommand 方法！

                    // 拉起服务的 onStartCommand 方法！
                    res = s.onStartCommand(data.args, data.flags, data.startId);

                } else {
                    // 否则，拉起 onTaskRemoved 方法！
                    s.onTaskRemoved(data.args);
                    res = Service.START_TASK_REMOVED_COMPLETE;

                }

                QueuedWork.waitToFinish();

                try {

                    // 通知 AMS，操作完成，并传递 onStartCommand 的返回结果；
                    ActivityManagerNative.getDefault().serviceDoneExecuting(
                            data.token, SERVICE_DONE_EXECUTING_START, data.startId, res);
                            
                } catch (RemoteException e) {
                    throw e.rethrowFromSystemServer();
                }

                ensureJitEnabled();

            } catch (Exception e) {

                if (!mInstrumentation.onException(s, e)) {
                    throw new RuntimeException(
                            "Unable to start service " + s
                            + " with " + data.args + ": " + e.toString(), e);
                }
            }
        }
    }
```

#### 3.3.2.1 service.onStartCommand
```java
    public @StartResult int onStartCommand(Intent intent, @StartArgFlags int flags, int startId) {
        onStart(intent, startId);
        return mStartCompatibility ? START_STICKY_COMPATIBILITY : START_STICKY;
    }
```
最后进入了 Service 的 onStartCommand 方法！


## 3.4 ActivityManagerP.serviceDoneExecuting

当服务的指定方法被拉起后，会通知 AMS，拉起操作执行成功，代码如下：
```java                
    // 通知 AMS，拉起 onCreate 操作完成；
    ActivityManagerNative.getDefault().serviceDoneExecuting(
                data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
                
    // 通知 AMS，拉起 onStart 操作完成；
    ActivityManagerNative.getDefault().serviceDoneExecuting(
                data.token, SERVICE_DONE_EXECUTING_START, data.startId, res);
```

这里进入 ActivityManagerP：
```java
    public void serviceDoneExecuting(IBinder token, int type, int startId,
            int res) throws RemoteException {

        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IActivityManager.descriptor);
        data.writeStrongBinder(token);
        data.writeInt(type);
        data.writeInt(startId);
        data.writeInt(res);
        mRemote.transact(SERVICE_DONE_EXECUTING_TRANSACTION, data, reply, IBinder.FLAG_ONEWAY);
        reply.readException();
        data.recycle();
        reply.recycle();
    }

```

这里再次进入系统进程，我们去看看发生了什么！！

# 4 系统进程

接下来，进入系统进程，处理反馈结果，首先要进入 ActivityManagerN.onTransact 方法！
```java
        case SERVICE_DONE_EXECUTING_TRANSACTION: {
            data.enforceInterface(IActivityManager.descriptor);
            
            
            IBinder token = data.readStrongBinder();
            
            int type = data.readInt();
            int startId = data.readInt();
            int res = data.readInt();
            
            // 进入 AMS!
            serviceDoneExecuting(token, type, startId, res);
            reply.writeNoException();
            return true;
        }
```

## 4.1 ActivityManagerS.serviceDoneExecuting
```java
    public void serviceDoneExecuting(IBinder token, int type, int startId, int res) {
        synchronized(this) {
        
            // 如果 token 不是 ServiceRecord 的实例，就会抛出异常！
            if (!(token instanceof ServiceRecord)) {
                Slog.e(TAG, "serviceDoneExecuting: Invalid service token=" + token);
                throw new IllegalArgumentException("Invalid service token");
            }
            
            // 进入 ActiveServices！
            mServices.serviceDoneExecutingLocked((ServiceRecord)token, type, startId, res);
        }
    }
```

继续看！
## 4.2 ActiveServices.serviceDoneExecutingLocked

- 对于拉起 onCreate  方法，type 的值为 SERVICE_DONE_EXECUTING_ANON；
- 对于拉起 onStartCommand 方法，type 的值为 SERVICE_DONE_EXECUTING_START；

```java
    void serviceDoneExecutingLocked(ServiceRecord r, int type, int startId, int res) {
    
        // 判断这个服务是否正在被销毁！
        boolean inDestroying = mDestroyingServices.contains(r);
        if (r != null) {
        
            // 这里处理拉起 onStartCommand 方法后的，onStart 方法的返回值！
            if (type == ActivityThread.SERVICE_DONE_EXECUTING_START) {

                // 将 r.callStart 置为 true！
                r.callStart = true;

                switch (res) {

                    case Service.START_STICKY_COMPATIBILITY:
                    case Service.START_STICKY: {

                        // 从服务的 deliveredStarts 集合中删除本次启动对应的启动项！
                        r.findDeliveredStart(startId, true);

                        // 表示服务被杀死后不会被停止，会被重启！
                        r.stopIfKilled = false;

                        break;
                    }

                    case Service.START_NOT_STICKY: {

                        // 从服务的 deliveredStarts 集合中删除本次启动对应的启动项！
                        r.findDeliveredStart(startId, true);

                        // 判断服务最后一次启动的 id 是否为本次 id！
                        if (r.getLastStartId() == startId) {

                            // 表示服务被杀死后会被停止，不会被重启！
                            r.stopIfKilled = true;
                        }

                        break;
                    }

                    case Service.START_REDELIVER_INTENT: {

                        // 第二个参数为 false，并没有删除 startId 对应的启动项目，这里只是返回了！
                        // 这个启动项 Intent 被保留下来了！
                        ServiceRecord.StartItem si = r.findDeliveredStart(startId, false);
                        if (si != null) {
                            si.deliveryCount = 0;
                            si.doneExecutingCount++;

                            // Don't stop if killed.
                            r.stopIfKilled = true;
                        }

                        break;
                    }

                    case Service.START_TASK_REMOVED_COMPLETE: {
    
                        // Special processing for onTaskRemoved().  Don't
                        // impact normal onStartCommand() processing.
                        r.findDeliveredStart(startId, true);
    
                        break;
                    }

                    default:
                        throw new IllegalArgumentException(
                                "Unknown service start result: " + res);

                }
                if (res == Service.START_STICKY_COMPATIBILITY) {
                    r.callStart = false;
                }

            } else if (type == ActivityThread.SERVICE_DONE_EXECUTING_STOP) {
                // 这里是处理 destroy 服务的情况，这里先不看！

            }
            
            // 对于拉起 onCreate 方法的返回 type，直接进入这里！！
            final long origId = Binder.clearCallingIdentity();

            serviceDoneExecutingLocked(r, inDestroying, inDestroying);

            Binder.restoreCallingIdentity(origId);
        } else {
            Slog.w(TAG, "Done executing unknown service from pid "
                    + Binder.getCallingPid());
        }
    }
```

findDeliveredStart 方法用于从 ServiceRecord 的 deliveredStarts 集合中返回指定 id 的启动项，如果 remove 为 true，也要移除指定 id 的启动项！

```java
    public StartItem findDeliveredStart(int id, boolean remove) {
        final int N = deliveredStarts.size();
        for (int i=0; i<N; i++) {
            StartItem si = deliveredStarts.get(i);
            if (si.id == id) {
                if (remove) deliveredStarts.remove(i);
                return si;
            }
        }
        
        return null;
    }
```

这里我们来看看 `Service` 的 `onStartCommand` 方法的返回值的处理总结：

- **START_STICKY**：
- **START_STICKY_COMPATIBILITY**：
  - 删除本次启动的 `startId` 对应的启动项；
  - 设置服务的 `stopIfKilled` 为 `false`；
- **START_NOT_STICKY**： 
  - 删除本次启动的服务的 `startId` 对应的启动项；
  - 如果最后一次启动的启动项的 `id` (lastStartId) 等于本次启动的启动项的 `id`；
     - 设置服务的 `stopIfKilled` 为 `true`；
- **START_REDELIVER_INTENT**： 
  - 不删除本次启动的 `startId` 对应的启动项；
  - 如果本次启动的启动项不为 null；
     - 设置启动项的 `deliveryCount` 为 `0`；
     - 设置启动项的 `doneExecutingCount` 加 `1`；
     - 设置服务的 `stopIfKilled` 为 `true`；

那么，如果服务被启动后，因为一些原因，所在的进程被杀掉了，那么系统会根据服务以上的属性对服务进行重启！

这里的 `r.stopIfKilled` 属性在 `AS.killServicesLocked` 的方法中会处理，用于判断服务在被 `kill` 掉后，是否被重启起！对于 `Service` 的重启，我会在另外一篇博文中分析，尽情期待！

## 4.3 ActiveServices.serviceDoneExecutingLocked

我们继续看，参数传递： 

- `boolean inDestroying`：服务正在被销毁！
- `boolean finishing`：服务正在完成！

因为这里是刚刚启动服务，所以传入均为 false；

```java
    private void serviceDoneExecutingLocked(ServiceRecord r, boolean inDestroying,
            boolean finishing) {

        if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "<<< DONE EXECUTING " + r
                + ": nesting=" + r.executeNesting
                + ", inDestroying=" + inDestroying + ", app=" + r.app);
        else if (DEBUG_SERVICE_EXECUTING) Slog.v(TAG_SERVICE_EXECUTING,
                "<<< DONE EXECUTING " + r.shortName);

        r.executeNesting--;

        if (r.executeNesting <= 0) {
            if (r.app != null) {

                if (DEBUG_SERVICE) Slog.v(TAG_SERVICE,
                        "Nesting at 0 of " + r.shortName);

                r.app.execServicesFg = false;
                
                // 从 r.app.executingServices 移除该服务！
                r.app.executingServices.remove(r);
                
                // 如果该进程中所有的服务都执行成功了，进入该分支！
                if (r.app.executingServices.size() == 0) {
                    if (DEBUG_SERVICE || DEBUG_SERVICE_EXECUTING) 
                         Slog.v(TAG_SERVICE_EXECUTING,
                            "No more executingServices of " + r.shortName);

                    // 移除服务启动超时的消息！
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

                if (inDestroying) { // 不进入！
                    if (DEBUG_SERVICE) Slog.v(TAG_SERVICE,
                            "doneExecuting remove destroying " + r);
                    mDestroyingServices.remove(r);

                    r.bindings.clear();

                }
                
                // 更新 oomAdj！
                mAm.updateOomAdjLocked(r.app);
            }

            r.executeFg = false;
            if (r.tracker != null) {
                r.tracker.setExecuting(false, mAm.mProcessStats.getMemFactorLocked(),
                        SystemClock.uptimeMillis());
                if (finishing) {
                    r.tracker.clearCurrentOwner(r, false);
                    r.tracker = null;
                }
            }

            if (finishing) { // 不进入！
                if (r.app != null && !r.app.persistent) {
                    r.app.services.remove(r);
                    if (r.whitelistManager) {
                        updateWhitelistManagerLocked(r.app);
                    }
                }
                r.app = null;
            }
        }
    }

```
这里会让 AMS 的 MainHandler 不再处理超时的消息！


# 5 总结

到这里，startService 的启动流程就分析完了，我们来回顾一下！！

## 6.1 数据结构总结

我们先来看看，在 start 流程中，会涉及到的一些重要的数据结构，以及他们之间的关系！

### 6.1.1 **ProcessRecord**

启动者和被启动者所在进程的信息对象；
```java 
    int curProcState // 表示当前进程的状态，可以通过其来判断是否是前台进程调度，还是后台进调度
    int setSchedGroup // 进程的最新状态
    
    ArraySet<ServiceRecord> services // 运行在这个进程中的所有服务
    ArraySet<ServiceRecord> executingServices // 当前正在执行某段代码逻辑的服务列表，比如正在拉起 onCreate 等
```
属性解释：


### 6.1.2 **ActivieServices**

用来保存和管理系统中所有活跃的 Service！
```java   
    SparseArray<ServiceMap> mServiceMap // 所有设备用户下的 Service 集合，key 为 userId，value 是 ServiceMap 类型的实例；
    
    ArrayList<ServiceRecord> mRestartingServices  // 因为 CRASH 需要重启的  ServiceRecord 集合！
    ArrayList<ServiceRecord> mDestroyingServices // 进程被销毁的 ServiceRecord 集合！
	ArrayList<ServiceRecord> mPendingServices // 所有正在启动的 ServiceRecord 集合
```
属性解释：
			
### 6.1.3 **ServiceMap**

是 Handler 的子类，用来保存制定 userId 目录下的所有 ServiceRecord！ 

```java
    int mUserId： // 指定的 uid

    ArrayMap<ComponentName, ServiceRecord> mServicesByName：// 通过组件名为 key，存储对应的 ServiceRecord
    ArrayMap<Intent.FilterComparison, ServiceRecord> mServicesByIntent；// 通过 intent 为 key，存储对应的 ServiceRecord
    
    ArrayList<ServiceRecord> mDelayedStartList: // 延迟启动的 ServiceRecord 列表
    ArrayList<ServiceRecord> mStartingBackground：// 正在后台启动的 ServiceRecord 列表 
```
属性解释：

### 6.1.4 **ServiceRecord**

要启动的 Service 的数据对象！

```java
        ServiceInfo serviceInfo // 该服务的解析信息，来自 PMS
        ApplicationInfo appInfo // 该服务所属应用的信息，来自 PMS
        int userId // 该服务所属的设备用户
        Intent.FilterComparison intent // 用来匹配该服务的 intent 封装类，等价于 intent
        String permission // 访问该服务需要的权限

        long createTime // 该服务创建的时间
        boolean createdFromFg // 表示服务是通过前台进程调度创建的还是后台进程
        long lastActivity // 服务的活跃时间；

        boolean startRequested // 服务是否被请求启动，开始启动时会置为 true；
        boolean delayedStop // 服务是否被延迟停止，开始启动时会置为 false；
		boolean delayed // 服务是否延迟启动，默认为 false；
		
		long restartDelay // 服务重启延迟时间！
		int restartCount // 服务重启次数！
        long restartTime // 服务重启时间！
		
		ArrayList<StartItem> pendingStarts // 待处理的启动项列表；
		ArrayList<StartItem> deliveredStarts // 已经被处理的启动项列表；
		
		Runnable restarter // 这是一个 ServiceRestarter 对象，用于处理重启任务！
		
		boolean callStart
		ProcessRecord app // 服务所在的进程
		
		boolean executeFg // 表示该服务是否是前台执行
        int executeNesting // 
        long executingStart // 表示服务开始执行的时间
```
属性解释：

- **r.pendingStarts**：AS 会把启动相关的信息封装成一个 StartItem，保存到 pendingStarts 中！
- **r.delayed：**于后台启动的方式，如果 ServiceMap.mStartingBackground 的大小超过 mMaxStartingBackground 的话，会被置为 true，同时，将 ServiceRecord 添加到  ServiceMap.mDelayedStartList 中！

### 6.1.5  **StartItem**

这是一个启动项，封装一次 `startService` 的数据

```java
        final ServiceRecord sr; // 要启动的服务
        final boolean taskRemoved; //  
        final int id; // 启动项 id ，最小从 1 开始
        final Intent intent; // 启动用的 Intent
        final ActivityManagerService.NeededUriGrants neededGrants;
        long deliveredTime;
        int deliveryCount;
        int doneExecutingCount;
        UriPermissionOwner uriPermissions;
```
属性解释：

### 6.1.6 **ServiceRestarter**

这是 `Runnable` 对象，用于处理服务的重启！

```java
        ServiceRecord mService // 对应的服务！
```
属性解释：

### 6.1.7 **CreateServiceData**

`ActivityThread` 的内部类，用于封装拉起 `onCreate` 方法的数据：
```java
        IBinder token; // ServiceRecord 对象
        ServiceInfo info;
        CompatibilityInfo compatInfo;
        Intent intent;
```

属性解释：

### 6.1.8 **ServiceArgsData**

`ActivityThread` 的内部类，用于封装拉起 `onStartCommand` 方法的数据：

```java
        IBinder token; // ServiceRecord 对象
        boolean taskRemoved;
        int startId; // 本次启动的 id
        int flags; // 如果
        Intent args; // 启动的 Intent
```
属性解释：

## 6.2 生命周期总结

startService 方法周期很简单：
```java
Serice.onCreate()(once) -> Service.onStartCommand()(more)
```

## 6.3 流程总结

下面是整个过程的 UML 序列图：

（我先填个坑，毕竟画图太累）

我们来看一下，这里面的一些细节：

- 如果多次调用 `startService`，只有第一次会拉起服务的 `onCreate` 方法，因为 `r.app` 和 `r.app.thread` 为 `null`，`r.app` 是在创建 `Serivce（onCreate）`时被赋值的，多次调用 `StartService`，会直接进入 `sendServiceArgsLocked` 方法，拉起服务的 `onStartCommand` 方法；
- `onCreate` 方法的 `startId` 为 `0`，`onStartCommand` 方法的 `startId` 从 `1` 开始，一次递增！
- 如果服务启动后，因为内存不足等情况，被杀掉了，那么系统会重启该服务，如果在重启之前，用户通过 `startService` 再次启动了服务，那么就会取消之前的重启任务！（`r.restartDelay` 是在设置服务重启任务时设置的，我们后面再看）
- `onStartCommand` 方法的返回值有多个，这些返回值会影响 `Serivce` 的重启，请看 `Service reStart` 流程处理的分析，这里就不再多说了！







