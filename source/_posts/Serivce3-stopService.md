# Serivce 篇 3 - stopService 流程分析
title: Serivce 篇 3 - stopService 流程分析
date: 2016/04/03 20:46:25
catalog: true
categories:
- AndroidFramework源码分析
- Service服务
tags: Service服务
---

本文基于 Android 7.1.1 源码分析，转载请说明出处！

# 0 综述

我们通过 `startService` 启动的服务，需要通过 `stopService` 来停止：

```java
context.stopService(intent);
```

以前我们只是会调用，但是其底层的调用到底是什么样的呢？知其然知其所以然，今天我们就来学习下 `stopService` 的过程！

如果之前并不了解这块逻辑的话，那该如何去学习呢？ follow the funtion path！

# 1 发起端进程

## 1.1 ContextWrapper.stopService

```java
public class ContextWrapper extends Context {
    Context mBase;

    public ContextWrapper(Context base) {
        mBase = base;
    }
    
    @Override
    public boolean stopService(Intent name) {
        return mBase.stopService(name);
    }
    
}
```

`ContextWrapper` 提供了两个方法来启动 `Service`，其中一个是隐藏方法：`startServiceAsUser`！

`mBase` 是 `ContextImpl` 对象，继续看！

## 1.2 ContextImpl.stopService
```java
class ContextImpl extends Context {

    @Override
    public boolean stopService(Intent service) {
        
        // 这里判断调用者是否是系统进程；
        warnIfCallingFromSystemProcess();
        //【1】继续停止服务！
        return stopServiceCommon(service, mUser);
    }
}
```

这里调用了 `warnIfCallingFromSystemProcess`，判断是否是系统进程调用！
```java
    private void warnIfCallingFromSystemProcess() {
        if (Process.myUid() == Process.SYSTEM_UID) {
            Slog.w(TAG, "Calling a method in the system process without a qualified user: "
                    + Debug.getCallers(5));
        }
    }
```

`ContextImpl` 和 `ContextWrapper` 的具体关系，请来看另一博文：`Android` 系统的 `Context` 分析，这里我们不再详细说明！

`mUser`：表示的是当前的设备 `user`！

最终，调用了 `stopServiceCommon` 方法；

## 1.3 ContextImpl.stopServiceCommon
```java
    private boolean stopServiceCommon(Intent service, UserHandle user) {
        try {
            validateServiceIntent(service);
            service.prepareToLeaveProcess(this);
            
            //【1】进入到系统进程，停止服务！
            int res = ActivityManagerNative.getDefault().stopService(
                mMainThread.getApplicationThread(), service,
                service.resolveTypeIfNeeded(getContentResolver()), user.getIdentifier());
            if (res < 0) {
                throw new SecurityException(
                        "Not allowed to stop service " + service);
            }
            return res != 0;
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
```
validateServiceIntent 方法用用来校验启动用的 intent 是否安全：
```java
    private void validateServiceIntent(Intent service) {
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


接着，调用 `ActivityManagerNative.getDefault()` 方法，获得 `AMS` 的代理对象 `ActivityManagerProxy`！

`ActivityManagerProxy` 是 `ActivityManagerN` 的内部类，通过 `getDefault` 方法创建了对应的单例模式，保存在 `ActivityManagerNative` 的类变量 `getDefault` 中！

## 1.4 ActivityManagerP.stopService
```java
    public int stopService(IApplicationThread caller, Intent service,
            String resolvedType, int userId) throws RemoteException
    {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IActivityManager.descriptor);

        data.writeStrongBinder(caller != null ? caller.asBinder() : null);

        service.writeToParcel(data, 0);
        data.writeString(resolvedType);
        data.writeInt(userId);

        //【1】binder 通信，阻塞式通信！！
        mRemote.transact(STOP_SERVICE_TRANSACTION, data, reply, 0);

        reply.readException();
        int res = reply.readInt();
        reply.recycle();
        data.recycle();
        return res;
    }
```
通过 `binder` 进程间通信，进入系统进程，参数分析：

- `IApplicationThread caller`：调用者进程的 `ApplicationThread` 对象，实现了 `IApplicationThread` 接口；
- `Intent service`：启动的 `intent`；
- `String resolvedType`：这个 `intent` 的 `MIME` 类型；
- `String callingPackage`：启动者所属包名；
- `int userId`：设备用户 `id`；


# 2 系统进程

接下来，进入系统进程的 `ActivityManagerService` 中！

首先要进入 `ActivityManagerN.onTransact` 方法！
```java
        case STOP_SERVICE_TRANSACTION: {
            data.enforceInterface(IActivityManager.descriptor);
            IBinder b = data.readStrongBinder();

            IApplicationThread app = ApplicationThreadNative.asInterface(b);

            Intent service = Intent.CREATOR.createFromParcel(data);
            String resolvedType = data.readString();
            int userId = data.readInt();

            //【1】继续调用 stopService 方法！
            int res = stopService(app, service, resolvedType, userId);

            reply.writeNoException();
            reply.writeInt(res);
            return true;
        }
```

## 2.1 ActivityManagerS.stopService
```java
    @Override
    public int stopService(IApplicationThread caller, Intent service,
            String resolvedType, int userId) {
            
        // 隔离进程不能调用这个方法！
        enforceNotIsolatedCaller("stopService");

        // 不能用 intent 传递文件描述符！
        if (service != null && service.hasFileDescriptors() == true) {
            throw new IllegalArgumentException("File descriptors passed in Intent");
        }

        synchronized(this) {
            
            //【1】进入 ActiveServices 
            return mServices.stopServiceLocked(caller, service, resolvedType, userId);
        }
    }
```
`mServices` 是 `ActivityManagerService` 的一个内部管理对象，用于管理所有的 `Service`！

## 2.2 ActiveServices.stopServiceLocked

```java
    int stopServiceLocked(IApplicationThread caller, Intent service,
            String resolvedType, int userId) {
        if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "stopService: " + service
                + " type=" + resolvedType);
        
        // 获得调用者所在进程！
        final ProcessRecord callerApp = mAm.getRecordForAppLocked(caller);
        if (caller != null && callerApp == null) {
            throw new SecurityException(
                    "Unable to find app for caller " + caller
                    + " (pid=" + Binder.getCallingPid()
                    + ") when stopping service " + service);
        }

        //【1】查询要停止的服务的信息！
        ServiceLookupResult r = retrieveServiceLocked(service, resolvedType, null,
                Binder.getCallingPid(), Binder.getCallingUid(), userId, false, false, false);

        if (r != null) {
            if (r.record != null) {
                final long origId = Binder.clearCallingIdentity();
                try {
                
                    //【2】继续停止服务！
                    stopServiceLocked(r.record);
                } finally {
                    Binder.restoreCallingIdentity(origId);
                }
                return 1;
            }
            return -1;
        }

        return 0;
    }

```

### 2.2.1 ActiveServices.retrieveServiceLocked

根据 `intent` 来查询被启动服务的 `ServiceRecord` 对象，参数传递：

- `boolean createIfNeeded`：传入的是 `true`；
- `boolean callingFromFg`：表示是从前台调用，还是后台调用；
- `boolean isBindExternal`：表示 `bind` 的是否是 `isolated`, `external` 类型的服务，我们这里默认为 `false`；

```java
    private ServiceLookupResult retrieveServiceLocked(Intent service,
            String resolvedType, String callingPackage, int callingPid, int callingUid, int userId,
            boolean createIfNeeded, boolean callingFromFg, boolean isBindExternal) {
    
        ServiceRecord r = null;

        if (DEBUG_SERVICE)
            Slog.v(TAG_SERVICE, "retrieveServiceLocked: " + service
                + " type=" + resolvedType + " callingUid=" + callingUid);

        userId = mAm.mUserController.handleIncomingUser(callingPid, callingUid, userId, false,
                ActivityManagerService.ALLOW_NON_FULL_IN_PROFILE, "service", null);

        // 从 ActiveServices 的 mServiceMap 中获得设备用户 userId 下的所有服务列表！
        ServiceMap smap = getServiceMap(userId);
        
        // 如果 intent 有设置组件名，就从 smap.mServicesByName 中获取！
        final ComponentName comp = service.getComponent();
        if (comp != null) {
            r = smap.mServicesByName.get(comp);
        }

        // 如果 intent 没有设置组件名，就从 smap.mServicesByIntent 通过 filter 获得！
        if (r == null && !isBindExternal) {
            Intent.FilterComparison filter = new Intent.FilterComparison(service);
            r = smap.mServicesByIntent.get(filter);
        }
        
        // 对于 external 类型的服务，只能限制其应用程序自己 bind！
        if (r != null && (r.serviceInfo.flags & ServiceInfo.FLAG_EXTERNAL_SERVICE) != 0
                && !callingPackage.equals(r.packageName)) {

            r = null;
        }

        if (r == null) {
            try {

                // 通过 PMS，查询能够匹配 intent 的服务！
                ResolveInfo rInfo = AppGlobals.getPackageManager().resolveService(service,
                        resolvedType, ActivityManagerService.STOCK_PM_FLAGS
                                | PackageManager.MATCH_DEBUG_TRIAGED_MISSING,
                        userId);
                
                // 获得服务信息！
                ServiceInfo sInfo =
                    rInfo != null ? rInfo.serviceInfo : null;

                if (sInfo == null) {
                    Slog.w(TAG_SERVICE, "Unable to start service " + service + " U=" + userId +
                          ": not found");
                    return null;
                }

                ComponentName name = new ComponentName(
                        sInfo.applicationInfo.packageName, sInfo.name);
                
                // 如果服务类型是 external 的，那么 bind 时必须加 BIND_EXTERNAL_SERVICE 的标志！
                // 不然会抛出安全异常！
                if ((sInfo.flags & ServiceInfo.FLAG_EXTERNAL_SERVICE) != 0) {
                    if (isBindExternal) {
                        if (!sInfo.exported) {

                            // 对于 external  类型的服务，必须要设置 android:exported 的值为 true，不然抛异常！
                            throw new SecurityException("BIND_EXTERNAL_SERVICE failed, " + name +
                                    " is not exported");
                        }
                        
                        // 对于 external 类型的服务，必须运行在隔离进程中！
                        if ((sInfo.flags & ServiceInfo.FLAG_ISOLATED_PROCESS) == 0) {
                            throw new SecurityException("BIND_EXTERNAL_SERVICE failed, " + name +
                                    " is not an isolatedProcess");
                        }

                        // 获得调用者应用的 ApplicationInfo 信息
                        ApplicationInfo aInfo = AppGlobals.getPackageManager().getApplicationInfo(
                                callingPackage, ActivityManagerService.STOCK_PM_FLAGS, userId);

                        if (aInfo == null) {
                            throw new SecurityException("BIND_EXTERNAL_SERVICE failed, " +
                                    "could not resolve client package " + callingPackage);
                        }

                        sInfo = new ServiceInfo(sInfo);
                        sInfo.applicationInfo = new ApplicationInfo(sInfo.applicationInfo);
                        sInfo.applicationInfo.packageName = aInfo.packageName;
                        sInfo.applicationInfo.uid = aInfo.uid;
                        
                        // 设置 ComponentName 对象！
                        name = new ComponentName(aInfo.packageName, name.getClassName());
                        service.setComponent(name);

                    } else {
                        throw new SecurityException("BIND_EXTERNAL_SERVICE required for " +
                                name);
                    }
    
                } else if (isBindExternal) {
                    throw new SecurityException("BIND_EXTERNAL_SERVICE failed, " + name +
                            " is not an externalService");
                }

                // 处理多用户的情况！
                if (userId > 0) {
                    if (mAm.isSingleton(sInfo.processName, sInfo.applicationInfo,
                            sInfo.name, sInfo.flags)
                            && mAm.isValidSingletonCall(callingUid, sInfo.applicationInfo.uid)) {
                        userId = 0;
                        smap = getServiceMap(0);
                    }
                    sInfo = new ServiceInfo(sInfo);
                    sInfo.applicationInfo = mAm.getAppInfoForUser(sInfo.applicationInfo, userId);
                }
                
                // 根据 ComponentName 获得 ServiceRecord 对象！
                r = smap.mServicesByName.get(name);

                if (r == null && createIfNeeded) {

                    Intent.FilterComparison filter
                            = new Intent.FilterComparison(service.cloneFilter());
                            
                    // 创建服务重启对象，这是一个 Runnable 对象！！
                    ServiceRestarter res = new ServiceRestarter();

                    BatteryStatsImpl.Uid.Pkg.Serv ss = null;
                    BatteryStatsImpl stats = mAm.mBatteryStatsService.getActiveStatistics();

                    synchronized (stats) {
                        ss = stats.getServiceStatsLocked(
                                sInfo.applicationInfo.uid, sInfo.packageName,
                                sInfo.name);
                    }
                     
                    // 创建新的 ServiceRecord 对象！
                    r = new ServiceRecord(mAm, ss, name, filter, sInfo, callingFromFg, res);
                    res.setService(r);
                    
                    // 将新的 ServiceRecord 添加到 smap.mServicesByName 和 smap.mServicesByIntent 中！
                    smap.mServicesByName.put(name, r);
                    smap.mServicesByIntent.put(filter, r);

                    // 从 mPendingServices 中移除这个 ServiceRecord！
                    for (int i=mPendingServices.size()-1; i>=0; i--) {
                        ServiceRecord pr = mPendingServices.get(i);
                        if (pr.serviceInfo.applicationInfo.uid == sInfo.applicationInfo.uid
                                && pr.name.equals(name)) {
                            mPendingServices.remove(i);
                        }
                    }
                }
            } catch (RemoteException ex) {
                // pm is in same process, this will never happen.
            }
        }

        // 下面对查询的数据进行封装！
        if (r != null) {
            if (mAm.checkComponentPermission(r.permission,
                    callingPid, callingUid, r.appInfo.uid, r.exported)
                    != PackageManager.PERMISSION_GRANTED) { // 检查权限如果不授予！

                if (!r.exported) {
                    Slog.w(TAG, "Permission Denial: Accessing service " + r.name
                            + " from pid=" + callingPid
                            + ", uid=" + callingUid
                            + " that is not exported from uid " + r.appInfo.uid);
                    
                    // 异常返回，_record 为 null，无法启动！
                    return new ServiceLookupResult(null, "not exported from uid "
                            + r.appInfo.uid);
                }

                Slog.w(TAG, "Permission Denial: Accessing service " + r.name
                        + " from pid=" + callingPid
                        + ", uid=" + callingUid
                        + " requires " + r.permission);

                // 异常返回，_record 为 null，无法启动！
                return new ServiceLookupResult(null, r.permission);
            } else if (r.permission != null && callingPackage != null) {
                final int opCode = AppOpsManager.permissionToOpCode(r.permission);
                if (opCode != AppOpsManager.OP_NONE && mAm.mAppOpsService.noteOperation(
                        opCode, callingUid, callingPackage) != AppOpsManager.MODE_ALLOWED) {
                        
                    Slog.w(TAG, "Appop Denial: Accessing service " + r.name
                            + " from pid=" + callingPid
                            + ", uid=" + callingUid
                            + " requires appop " + AppOpsManager.opToName(opCode));
                            
                    // 异常返回，无法启动！
                    return null;
                }
            }

            if (!mAm.mIntentFirewall.checkService(r.name, service, callingUid, callingPid,
                    resolvedType, r.appInfo)) {

                // 异常返回，无法启动！
                return null;
            }
            
            // 返回最终的服务封装类！！
            return new ServiceLookupResult(r, null);
        }
        return null;
    }

```

这个方法的最终目的是返回要启动的服务的信息封装对象 ServiceLookupResult！


## 2.3 ActiveServices.stopServiceLocked
```java
    private void stopServiceLocked(ServiceRecord service) {
        
        //【1】如果服务是延迟启动的，但是还没有启动！
        if (service.delayed) {

            if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE, "Delaying stop of pending: " + service);
            
            // 那就设置 delayedStop 为 true，这样服务在延迟启动时会判断这个变量，如果为 true，就不会启动这个服务！
            service.delayedStop = true;
            return;
        }

        synchronized (service.stats.getBatteryStats()) {
            service.stats.stopRunningLocked();
        }
        
        //【2】因为要停止服务了，所以设 startRequested 为 false！
        service.startRequested = false;

        if (service.tracker != null) {
            
            // 停止监控！
            service.tracker.setStarted(false, mAm.mProcessStats.getMemFactorLocked(),
                    SystemClock.uptimeMillis());
        }
        
        //【3】设置 callStart 为 false！
        service.callStart = false;
        
        //【4】停止服务！！
        bringDownServiceIfNeededLocked(service, false, false);
    }
```

## 2.4 ActiveServices.bringDownServiceIfNeededLocked
```java
    private final void bringDownServiceIfNeededLocked(ServiceRecord r, boolean knowConn,
            boolean hasConn) {
        
        //【1】如果这个服务还被需要，就不能停止！
        if (isServiceNeeded(r, knowConn, hasConn)) {
            return;
        }

        // 如果该服务正在等待其进程启动，就不能停止！
        if (mPendingServices.contains(r)) {
            return;
        }

        //【2】继续停止服务！
        bringDownServiceLocked(r);
    }
```

如果这个服务仍然被需要，或者服务所在的进程正在启动，这两种情况下，不能停止服务！

### 2.4.1 ActiveServices.isServiceNeeded
```java
    private final boolean isServiceNeeded(ServiceRecord r, boolean knowConn, boolean hasConn) {
        if (r.startRequested) { // 显然 r.startRequested 已经被置为 false！
            return true;
        }

        // 如果仍然有自动创建的连接，那就不能 stop 该服务！
        if (!knowConn) {
            hasConn = r.hasAutoCreateConnections();
        }
        if (hasConn) {
            return true;
        }

        return false;
    }
```
判断一个服务是否仍被需要，有两种情况：

- 服务已经被请求启动，`stopService` 前面会被置为 `false`；
- 服务被应用通过自动创建的方式绑定，即 `bindService` 时，`flags` 为 `Context.BIND_AUTO_CREATE`；

继续来看！

## 2.5 ActiveServices.bringDownServiceLocked

我们来看看 bringDownServiceLocked 方法做了什么：

```java
    private final void bringDownServiceLocked(ServiceRecord r) {
        //Slog.i(TAG, "Bring down service:");
        //r.dump("  ");

        //【1】首先，遍历 r.connections 集合，终止所有的 bind 连接
        for (int conni=r.connections.size()-1; conni>=0; conni--) {

            ArrayList<ConnectionRecord> c = r.connections.valueAt(conni);
            for (int i=0; i<c.size(); i++) {
                ConnectionRecord cr = c.get(i);
                
                // 设置 serviceDead 为 true，表示服务要被 stop 掉，连接断开！
                cr.serviceDead = true;
                try {
                    
                    // 1、Binder 调用，进入应用进程触发 ServiceConnection 的 onServiceDisconnected，这是非阻塞的！
                    // 这里第二个参数，为 null，所以会触发 onServiceDisconnected！
                    cr.conn.connected(r.name, null);
                    
                } catch (Exception e) {
                    Slog.w(TAG, "Failure disconnecting service " + r.name +
                          " to connection " + c.get(i).conn.asBinder() +
                          " (in " + c.get(i).binding.client.processName + ")", e);
                }
            }
        }

        //【2】处理所有的 r.bindings 集合中的 IntentBindRecord 对象！
        if (r.app != null && r.app.thread != null) {
            for (int i=r.bindings.size()-1; i>=0; i--) {

                IntentBindRecord ibr = r.bindings.valueAt(i);
                
                if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "Bringing down binding " + ibr
                        + ": hasBound=" + ibr.hasBound);
                        
                if (ibr.hasBound) {
                    try {
                        
                        // 设置 unbind 超时处理
                        bumpServiceExecutingLocked(r, false, "bring down unbind");
                        mAm.updateOomAdjLocked(r.app);
                        
                        // 置 IntentBindRecord 对象的 hasBound 为 false，表示 bind 断开；
                        ibr.hasBound = false;
                        
                        //【2.1】通过 bind 通信，进入服务的应用进程，调用 AT.scheduleUnbindService 方法
                        // ，拉起服务的 onUnbind 方法，接触绑定！
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

        if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "Bringing down " + r + " " + r.intent);
        
        // 设置服务的销毁时间
        r.destroyTime = SystemClock.uptimeMillis();

        if (LOG_SERVICE_START_STOP) {
            EventLogTags.writeAmDestroyService(
                    r.userId, System.identityHashCode(r), (r.app != null) ? r.app.pid : -1);
        }

        final ServiceMap smap = getServiceMap(r.userId);

        // 从 ActiveServices 的 mServiceMap 中移除服务 ServiceRecord 对象！
        smap.mServicesByName.remove(r.name);
        smap.mServicesByIntent.remove(r.intent);

        // 重启参数置为 0
        r.totalRestartCount = 0;

        // 取消重启任务
        unscheduleServiceRestartLocked(r, 0, true);

        // 从 mPendingServices 列表中移除！
        for (int i=mPendingServices.size()-1; i>=0; i--) {
            if (mPendingServices.get(i) == r) {
                mPendingServices.remove(i);
                if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "Removed pending: " + r);
            }
        }
        
        // 取消前台的通知！
        cancelForegroudNotificationLocked(r);
        
        r.isForeground = false;
        r.foregroundId = 0;
        r.foregroundNoti = null;

        // 清理启动项集合！
        r.clearDeliveredStartsLocked();
        r.pendingStarts.clear();

        if (r.app != null) {

            synchronized (r.stats.getBatteryStats()) {
                r.stats.stopLaunchedLocked();
            }
            
            // 将服务从其所在进程的 app.services 集合中删除！
            r.app.services.remove(r);

            if (r.whitelistManager) {
                updateWhitelistManagerLocked(r.app);
            }

            if (r.app.thread != null) {
                updateServiceForegroundLocked(r.app, false);
                try {
                     
                    // 设置 destroy 超时处理！                   
                    bumpServiceExecutingLocked(r, false, "destroy");
                    
                    // 将其添加到 AS 的 mDestroyingServices 机和中，表示服务正在销毁！
                    mDestroyingServices.add(r);
                    r.destroying = true;

                    mAm.updateOomAdjLocked(r.app);
                    
                    // 3、通过 binder 通信，回调 AT 的 scheduleStopService 方法！
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

        // 清空服务的 bindings 集合！
        if (r.bindings.size() > 0) {
            r.bindings.clear();
        }

        // 将服务的重启任务对象置空！
        if (r.restarter instanceof ServiceRestarter) {
           ((ServiceRestarter)r.restarter).setService(null);
        }

        int memFactor = mAm.mProcessStats.getMemFactorLocked();
        long now = SystemClock.uptimeMillis();

        if (r.tracker != null) {
            r.tracker.setStarted(false, memFactor, now);
            r.tracker.setBound(false, memFactor, now);

            if (r.executeNesting == 0) {
                r.tracker.clearCurrentOwner(r, false);
                r.tracker = null;
            }
        }

        smap.ensureNotStartingBackground(r);
    }

```
这段代码很简单，主要逻辑如下：

- 首先进入绑定服务的应用进程，回调 ServiceConnection 的 onServiceDisconnected 方法，取消所有进程对该服务的绑定；
- 接着，进入服务所在的进程，拉起服务的 onUnbind 方法；
- 最后，拉起服务的 onDestroy 方法，销毁服务；

整个过程，还会清空和重置一些关键变量！！

下面我们重点分析一下上面的三个过程！

# 3 绑定者进程

系统进程会调用 IServiceConnection.Proxy 代理对象的 connected 方法，通过 binder 调用，进入绑定服务的应用进程，这个是异步调用：

```java
cr.conn.connected(r.name, null);
```
我们去看看！

## 3.1 InnerConnection.connected
```java
        private static class InnerConnection extends IServiceConnection.Stub {
            final WeakReference<LoadedApk.ServiceDispatcher> mDispatcher;

            InnerConnection(LoadedApk.ServiceDispatcher sd) {
                mDispatcher = new WeakReference<LoadedApk.ServiceDispatcher>(sd);
            }

            public void connected(ComponentName name, IBinder service) throws RemoteException {

                LoadedApk.ServiceDispatcher sd = mDispatcher.get();

                if (sd != null) {
                
                    // 调用 mDispatcher 对象的 connected，这里的 service 为 null；
                    sd.connected(name, service);
                }
            }
        }
```
继续来看：

## 3.2 ServiceDispatcher.connected
```java
        public void connected(ComponentName name, IBinder service) {
            if (mActivityThread != null) {
                mActivityThread.post(new RunConnection(name, service, 0));

            } else {
                doConnected(name, service);

            }
        }
```
看过 bindService 一文的朋友，应该知道，最后都会调用 doConnected 方法：


## 3.3 ServiceDispatcher.doConnected

参数 service 这里为 null！
```java
        public void doConnected(ComponentName name, IBinder service) {
    
            ServiceDispatcher.ConnectionInfo old;
            ServiceDispatcher.ConnectionInfo info;

            synchronized (this) {
                
                if (mForgotten) {
                    return;
                }
                
                // 获得已有的活跃的 connection 对象！
                old = mActiveConnections.get(name);
                if (old != null && old.binder == service) { // 因为 service 为 null，所以这里不会进入该分支！
                    return;
                }

                if (service != null) {

                    // A new service is being connected... set it all up.
                    info = new ConnectionInfo();
                    info.binder = service;
                    info.deathMonitor = new DeathMonitor(name, service);
                    try {
                        service.linkToDeath(info.deathMonitor, 0);
                        mActiveConnections.put(name, info);
                    } catch (RemoteException e) {

                        // This service was dead before we got it...  just
                        // don't do anything with it.
                        mActiveConnections.remove(name);
                        return;
                    }

                } else {

                    // service 为 null，说明是解除绑定，所以要从 mActiveConnections 中移除绑定对象！
                    mActiveConnections.remove(name);
                }

                // 取消死亡通知监控器
                if (old != null) {
                    old.binder.unlinkToDeath(old.deathMonitor, 0);
                }
            }

            // 之前是有连接的，所以会进入这分支！！
            if (old != null) {
            
                // 拉起服务的 onServiceDisconnected 方法！
                mConnection.onServiceDisconnected(name);
            }

            // 不进入这个分支！
            if (service != null) {
                mConnection.onServiceConnected(name, service);
            }
        }
```
这里，最后会进入应用的 ServiceConnection 对象！
```java
    private ServiceConnection mConnection = new ServiceConnection() {
        @Override
        public void onServiceDisconnected(ComponentName name) {
        
        }
    }
```
注意，onServiceDisconnected 方法是非阻塞的，即，系统进程不会等 onServiceDisconnected 执行完才继续执行！！

# 4 服务所在进程

我们先回到系统进程 bringDownServiceLocked 方法中：
```java
    // 拉起服务的 onUnbind 方法；
    r.app.thread.scheduleUnbindService(r,
            ibr.intent.getIntent());
            
    ... ... ... ...
    
    // 拉起服务的 onDestroy 方法；
    r.app.thread.scheduleStopService(r);
```
这两个方法，都通过 ApplicationThreadProxy 对象进行 bind 通信，我们下面来逐个分析：

## 4.1 ApplicationThread.scheduleUnbindService

参数 IBinder token 就是 ServiceRecord 对象！

```java
    public final void scheduleUnbindService(IBinder token, Intent intent)
            throws RemoteException {

        Parcel data = Parcel.obtain();
        data.writeInterfaceToken(IApplicationThread.descriptor);
        data.writeStrongBinder(token);
        intent.writeToParcel(data, 0);
        
        // 这里也是异步调用 IBinder.FLAG_ONEWAY！
        mRemote.transact(SCHEDULE_UNBIND_SERVICE_TRANSACTION, data, null, IBinder.FLAG_ONEWAY);

        data.recycle();
    }
```
可以看出 **scheduleUnbindService** 方法的 binder 通信是 **IBinder.FLAG_ONEWAY** 方式！

## 4.2 ApplicationThreadP.scheduleStopService

参数 IBinder token 就是 ServiceRecord 对象！

```java
    public final void scheduleStopService(IBinder token)
            throws RemoteException {

        Parcel data = Parcel.obtain();
        data.writeInterfaceToken(IApplicationThread.descriptor);
        data.writeStrongBinder(token);
        
        // 这里也是异步调用 IBinder.FLAG_ONEWAY！
        mRemote.transact(SCHEDULE_STOP_SERVICE_TRANSACTION, data, null, IBinder.FLAG_ONEWAY);

        data.recycle();
    }
```
可以看出 **scheduleStopService** 方法的 binder 通信是 **IBinder.FLAG_ONEWAY** 方式！


下面是**真正进入了应用程序进程**！

## 4.3 ApplicationThreadN.onTransact

```java
        case SCHEDULE_UNBIND_SERVICE_TRANSACTION: {

            data.enforceInterface(IApplicationThread.descriptor);
            IBinder token = data.readStrongBinder();
            Intent intent = Intent.CREATOR.createFromParcel(data);
            
            // 进入 ActivityThread.scheduleUnbindService !
            scheduleUnbindService(token, intent);

            return true;
        }

        case SCHEDULE_STOP_SERVICE_TRANSACTION:
        {
            data.enforceInterface(IApplicationThread.descriptor);
            IBinder token = data.readStrongBinder();
            
            // 进入 ActivityThread.scheduleStopService！
            scheduleStopService(token);

            return true;
        }
```

## 4.4 ApplicationThread

接着，进入 ApplicationThread！！

### 4.4.1 ApplicationThread.scheduleUnbindService
```java
        public final void scheduleUnbindService(IBinder token, Intent intent) {
            BindServiceData s = new BindServiceData();
            
            // ServiceRecord 对象！
            s.token = token;
            s.intent = intent;

            // 发送 msg：H.UNBIND_SERVICE
            sendMessage(H.UNBIND_SERVICE, s);
        }
```

### 4.4.2 ApplicationThread.scheduleStopService
```java
        public final void scheduleStopService(IBinder token) {

            // 发送 msg：H.STOP_SERVICE
            sendMessage(H.STOP_SERVICE, token);
        }
```

## 4.5 ActivityThread.H
```java
    private class H extends Handler {
    
        public void handleMessage(Message msg) {
                case UNBIND_SERVICE:
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "serviceUnbind");
                    
                    handleUnbindService((BindServiceData)msg.obj);
                    
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                    break;
                case STOP_SERVICE:
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "serviceStop");
                    
                    handleStopService((IBinder)msg.obj);
                    maybeSnapshot();
                    
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                    break;        
        }
    }
```
这里很简单，我们继续看：

### 4.5.1 ActivityThread.handleUnbindService

首先会拉起 Serivce 的 onUnbind 方法！

```java
    private void handleUnbindService(BindServiceData data) {

        // 从 AT 的托管集合 mServices 获得 Service 对象！
        Service s = mServices.get(data.token);
        if (s != null) {
            try {
                data.intent.setExtrasClassLoader(s.getClassLoader());
                data.intent.prepareToEnterProcess();
                
                // 拉起服务的 onUnbind，onUnbind 的返回值表示，是否重新 rebind 服务，默认返回 false！
                boolean doRebind = s.onUnbind(data.intent);

                try {
                    if (doRebind) {
                    
                        // 如果 doRebind 为 true，进入这里！
                        ActivityManagerNative.getDefault().unbindFinished(
                                data.token, data.intent, doRebind);
                    } else {
                    
                        // 默认为 false，进入这里！
                        ActivityManagerNative.getDefault().serviceDoneExecuting(
                                data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
                    }

                } catch (RemoteException ex) {
                    throw ex.rethrowFromSystemServer();
                }

            } catch (Exception e) {
                if (!mInstrumentation.onException(s, e)) {
                    throw new RuntimeException(
                            "Unable to unbind to service " + s
                            + " with " + data.intent + ": " + e.toString(), e);
                }
            }
        }
    }
```
如果 onUnbind 方法返回 true，并且服务被解绑后没有被销毁，下次被绑定时，会拉起 onRebind 方法！

### 4.5.2 ActivityThread.handleStopService

最后，拉起 Serivce 的 onDestroy 方法！

```java
    private void handleStopService(IBinder token) {
        
        // 将服务从 AT 的托管集合 mServices 中移除！
        Service s = mServices.remove(token);
        if (s != null) {
            try {
                if (localLOGV) Slog.v(TAG, "Destroying service " + s);
                
                // 拉起 onDestory 方法！
                s.onDestroy();

                Context context = s.getBaseContext();

                if (context instanceof ContextImpl) {
                    final String who = s.getClassName();
                    
                    // 执行清理操作！
                    ((ContextImpl) context).scheduleFinalCleanup(who, "Service");
                }

                QueuedWork.waitToFinish();

                try {

                    // 最后，执行 serviceDoneExecuting 方法！
                    ActivityManagerNative.getDefault().serviceDoneExecuting(
                            token, SERVICE_DONE_EXECUTING_STOP, 0, 0);

                } catch (RemoteException e) {
                    throw e.rethrowFromSystemServer();
                }

            } catch (Exception e) {
                if (!mInstrumentation.onException(s, e)) {
                    throw new RuntimeException(
                            "Unable to stop service " + s
                            + ": " + e.toString(), e);
                }
                Slog.i(TAG, "handleStopService: exception for " + token, e);
            }

        } else {
            Slog.i(TAG, "handleStopService: token=" + token + " not found.");
        }

        //Slog.i(TAG, "Running services: " + mServices);
    }
```

这里逻辑很清楚，我们继续来看，接下来，调用 ActivityManagerP 相关的方法！

## 4.6 ActivityManagerProxy
```java
    // 对于 onUnbind 返回 true 的情况!
    public void unbindFinished(IBinder token, Intent intent, boolean doRebind)
            throws RemoteException {
            
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IActivityManager.descriptor);
        data.writeStrongBinder(token);
        intent.writeToParcel(data, 0);
        data.writeInt(doRebind ? 1 : 0);
        
        // flags 为 0
        mRemote.transact(UNBIND_FINISHED_TRANSACTION, data, reply, 0);

        reply.readException();
        data.recycle();
        reply.recycle();
    }
    
    // 对于 onUnbind 返回默认 false 的情况
    // 和拉起 onDestory 方法后！
    public void serviceDoneExecuting(IBinder token, int type, int startId,
            int res) throws RemoteException {
            
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IActivityManager.descriptor);
        data.writeStrongBinder(token);
        data.writeInt(type);
        data.writeInt(startId);
        data.writeInt(res);
        
        // flags 为 IBinder.FLAG_ONEWAY，非阻塞式的
        mRemote.transact(SERVICE_DONE_EXECUTING_TRANSACTION, data, reply, IBinder.FLAG_ONEWAY);

        reply.readException();
        data.recycle();
        reply.recycle();
    }
```

发送了 **UNBIND_FINISHED_TRANSACTION** 和 **SERVICE_DONE_EXECUTING_TRANSACTION** 消息！

unbindFinished 的 binder 通信是同步的，serviceDoneExecuting 的 binder 通信是异步的！

# 5 系统进程

首先会进入 ActivityManagerNative 的 onTransact 方法中：
```java
        case UNBIND_FINISHED_TRANSACTION: {

            data.enforceInterface(IActivityManager.descriptor);
            IBinder token = data.readStrongBinder();
            Intent intent = Intent.CREATOR.createFromParcel(data);
            boolean doRebind = data.readInt() != 0;
            
            // 继续来看！
            unbindFinished(token, intent, doRebind);

            reply.writeNoException();
            return true;
        }

        case SERVICE_DONE_EXECUTING_TRANSACTION: {

            data.enforceInterface(IActivityManager.descriptor);
            IBinder token = data.readStrongBinder();
            int type = data.readInt();
            int startId = data.readInt();
            int res = data.readInt();
            
            // 继续来看！
            serviceDoneExecuting(token, type, startId, res);

            reply.writeNoException();
            return true;
        }
```
接下来，进入 AMS!

## 5.1 ActivityManagerS.unbindFinished
```java
    public void unbindFinished(IBinder token, Intent intent, boolean doRebind) {
        
        // 不能通过 intent 传递文件描述符！
        if (intent != null && intent.hasFileDescriptors() == true) {
            throw new IllegalArgumentException("File descriptors passed in Intent");
        }

        synchronized(this) {
            
            // 进入 ActiveServices！
            mServices.unbindFinishedLocked((ServiceRecord)token, intent, doRebind);
        }
    }
```
### 5.1.1 ActiveServices.unbindFinishedLocked
```java
    void unbindFinishedLocked(ServiceRecord r, Intent intent, boolean doRebind) {
        final long origId = Binder.clearCallingIdentity();

        try {
            if (r != null) {
                Intent.FilterComparison filter
                        = new Intent.FilterComparison(intent);
                
                // 获得绑定 Serivce 的 intent 对应的 IntentBindRecord 对象！
                IntentBindRecord b = r.bindings.get(filter);

                if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "unbindFinished in " + r
                        + " at " + b + ": apps="
                        + (b != null ? b.apps.size() : 0));
                
                // 这里的 inDestroying 的值为 false，因为还没有添加！
                boolean inDestroying = mDestroyingServices.contains(r);
                
                // 显然，这里不会为 null！
                if (b != null) {

                    if (b.apps.size() > 0 && !inDestroying) {

                        // Applications have already bound since the last
                        // unbind, so just rebind right here.
                        boolean inFg = false;
                        for (int i=b.apps.size()-1; i>=0; i--) {
                            ProcessRecord client = b.apps.valueAt(i).client;
                            if (client != null && client.setSchedGroup
                                    != ProcessList.SCHED_GROUP_BACKGROUND) {
                                inFg = true;
                                break;
                            }
                        }

                        try {
                            requestServiceBindingLocked(r, b, inFg, true);
                        } catch (TransactionTooLargeException e) {
                            // Don't pass this back to ActivityThread, it's unrelated.
                        }

                    } else {

                        // 将 doRebind 置为 true！
                        b.doRebind = true;
                    }
                }
                
                // 最后进入 serviceDoneExecutingLocked 方法！
                serviceDoneExecutingLocked(r, inDestroying, false);
            }
        } finally {

            Binder.restoreCallingIdentity(origId);
        }
    }
```

接着来看！

### 5.1.2 ActiveServices.serviceDoneExecutingLocked

参数传递：

- boolean inDestroying：传入 false；
- boolean finishing：传入 false；

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
                
                // 从所在进程的 executingServices 中删除该服务！
                r.app.execServicesFg = false;
                r.app.executingServices.remove(r);

                if (r.app.executingServices.size() == 0) {
                    if (DEBUG_SERVICE || DEBUG_SERVICE_EXECUTING) Slog.v(TAG_SERVICE_EXECUTING,
                            "No more executingServices of " + r.shortName);
                    
                    // 如果进程中没有在执行指定代码逻辑的服务了，就取消超市任务！
                    mAm.mHandler.removeMessages(ActivityManagerService.SERVICE_TIMEOUT_MSG, r.app);

                } else if (r.executeFg) {

                    // 如果进程还有在执行指定代码逻辑的服务，并且有服务在前台执行，那就要将进程的 execServicesFg 置为 ture
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
                
                // 更新 oomAdj 值！
                mAm.updateOomAdjLocked(r.app);
            }
    　　　　
    　　　　// 置服务的 executeFg 为 false；
            r.executeFg = false;
            if (r.tracker != null) {
                r.tracker.setExecuting(false, mAm.mProcessStats.getMemFactorLocked(),
                        SystemClock.uptimeMillis());
                if (finishing) {
                    r.tracker.clearCurrentOwner(r, false);
                    r.tracker = null;
                }
            }

            if (finishing) { // 不进入这个分支！
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
这里就不再详细说了！！

## 5.2 ActivityManagerS.serviceDoneExecuting

对于拉起 onDestroy 方法，最后会调用
```java
    public void serviceDoneExecuting(IBinder token, int type, int startId, int res) {
        synchronized(this) {
            if (!(token instanceof ServiceRecord)) {
                Slog.e(TAG, "serviceDoneExecuting: Invalid service token=" + token);
                throw new IllegalArgumentException("Invalid service token");
            }
            
            // 最后，进入 AS！
            mServices.serviceDoneExecutingLocked((ServiceRecord)token, type, startId, res);
        }
    }
```
接着，进入 ActiveServices！

### 5.2.1 ActiveServices.serviceDoneExecutingLocked

根据参数传递：

- **int type**：传入 ActivityThread.SERVICE_DONE_EXECUTING_STOP；
- **int startId**：传入 0；
- **int res**：传入 0；

```java
    void serviceDoneExecutingLocked(ServiceRecord r, int type, int startId, int res) {
        
        // 这里的 inDestroying 为 true，因为我们之前已经将该服务添加到 mDestroyingServices！
        boolean inDestroying = mDestroyingServices.contains(r);

        if (r != null) {
            if (type == ActivityThread.SERVICE_DONE_EXECUTING_START) {
                
                ... ... ... // 这里是和 startService 服务有关，我们这里不看！
                
            } else if (type == ActivityThread.SERVICE_DONE_EXECUTING_STOP) {

                // This is the final call from destroying the service...  we should
                // actually be getting rid of the service at this point.  Do some
                // validation of its state, and ensure it will be fully removed.
                if (!inDestroying) {

                    // Not sure what else to do with this...  if it is not actually in the
                    // destroying list, we don't need to make sure to remove it from it.
                    // If the app is null, then it was probably removed because the process died,
                    // otherwise wtf
                    if (r.app != null) {
                        Slog.w(TAG, "Service done with onDestroy, but not inDestroying: "
                                + r + ", app=" + r.app);
                    }
                } else if (r.executeNesting != 1) {

                    Slog.w(TAG, "Service done with onDestroy, but executeNesting="
                            + r.executeNesting + ": " + r);
                    // Fake it to keep from ANR due to orphaned entry.
                    r.executeNesting = 1;
                }
            }

            final long origId = Binder.clearCallingIdentity();
            
            // 最后，再次调用 serviceDoneExecutingLocked！
            serviceDoneExecutingLocked(r, inDestroying, inDestroying);
            Binder.restoreCallingIdentity(origId);
        } else {
            Slog.w(TAG, "Done executing unknown service from pid "
                    + Binder.getCallingPid());
        }
    }
```

### 5.2.2 ActiveServices.serviceDoneExecutingLocked

参数传递：

- boolean inDestroying：传入 true；
- boolean finishing：传入 true；

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
                
                // 从所在进程的 executingServices 中删除该服务！
                r.app.execServicesFg = false;
                r.app.executingServices.remove(r);

                if (r.app.executingServices.size() == 0) {
                    if (DEBUG_SERVICE || DEBUG_SERVICE_EXECUTING) Slog.v(TAG_SERVICE_EXECUTING,
                            "No more executingServices of " + r.shortName);
                    
                    // 如果进程中没有在执行指定代码逻辑的服务了，就取消超市任务！
                    mAm.mHandler.removeMessages(ActivityManagerService.SERVICE_TIMEOUT_MSG, r.app);

                } else if (r.executeFg) {

                    // 如果进程还有在执行指定代码逻辑的服务，并且有服务在前台执行，那就要将进程的 execServicesFg 置为 ture
                    for (int i=r.app.executingServices.size()-1; i>=0; i--) {
                        if (r.app.executingServices.valueAt(i).executeFg) {
                            r.app.execServicesFg = true;
                            break;
                        }
                    }
                }

                if (inDestroying) { // inDestroying 为 true 进入！

                    if (DEBUG_SERVICE) Slog.v(TAG_SERVICE,
                            "doneExecuting remove destroying " + r);
                            
                    // 从 mDestroyingServices 中删除 ServiceRecord！
                    mDestroyingServices.remove(r);
                    
                    // 清空其 bindings 集合！
                    r.bindings.clear();
                }
                
                // 更新 oomAdj 值！
                mAm.updateOomAdjLocked(r.app);
            }
    　　　　
    　　　　// 置服务的 executeFg 为 false；
            r.executeFg = false;

            if (r.tracker != null) {
                r.tracker.setExecuting(false, mAm.mProcessStats.getMemFactorLocked(),
                        SystemClock.uptimeMillis());
                        
                // 取消对服务的监控！
                if (finishing) {
                    r.tracker.clearCurrentOwner(r, false);
                    r.tracker = null;
                }
            }

            if (finishing) { // finishing 为 true ，进入这个分支！
                if (r.app != null && !r.app.persistent) {
                
                    // 如果服务所在的进程不是常驻进程，从进程的 services 中移除这个服务！
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

到这里，stopService 的整个过程，就分析完了！！

# 6 总结

我们来回顾一下，整个 stopService 的过程，都遇到了那些重要的数据结构！

