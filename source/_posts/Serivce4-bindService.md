# Serivce 篇 4 - bindService 流程分析
title: Serivce 篇 4 - bindService 流程分析
date: 2016/05/31 20:46:25
catalog: true
categories:
- AndroidFramework源码分析
- Service服务
tags: Service服务
---

本文基于 Android 7.1.1 源码分析，转载请说明出处！

# 0 综述

我们在应用中经常会启动 Service：

```java
context.bindService(intent, mConnection, Service.BIND_AUTO_CREATE);
```

以前我们只是会调用，但是其底层的调用到底是什么样的呢？知其然知其所以然，今天我们就来学习下 bindService 的过程！

如果之前并不了解这块逻辑的话，那该如何去学习呢？ follow the funtion path！


# 1 绑定者进程

## 1.1 ContextWrapper.bindService
```java
public class ContextWrapper extends Context {
    Context mBase;

    public ContextWrapper(Context base) {
        mBase = base;
    }
    
    @Override
    public boolean bindService(Intent service, ServiceConnection conn,
            int flags) {
        return mBase.bindService(service, conn, flags);
    }

    /** @hide */
    @Override
    public boolean bindServiceAsUser(Intent service, ServiceConnection conn, int flags,
            UserHandle user) {
        return mBase.bindServiceAsUser(service, conn, flags, user);
    }

}
```

ContextWrapper 提供了两个方法来绑定 Service，其中一个是隐藏方法：bindServiceAsUser！

bindService 的第三个参数 flags 一般都会传 0 或 BIND_AUTO_CREATE，**跨进程调用 bindService 会引起组件间的依赖**，比如 A 进程的 Activity 中 bindService 调用 B 进程 service，则 B 进程的 service 的 oom_adj 值依赖于 A 进程 Activity 的 oom_adj 值，这个我们后面在说，这些 flags 也可以通过按位或的方式，进行搭配使用！！

接着来说 flags，这里的 flags 可以取值如下，Android 7.1.1 内置了如下的几个标志位：

| 标志位 | 解释 |
| ---- | ---- |
| **0** | 无意义 |
| **BIND_AUTO_CREATE** (0x0001) | 绑定服务时候，如果服务尚未创建，服务会自动创建，在 API LEVEL 14 以前的版本不支持这个标志，使用 Context.BIND_WAIVE_PRIORITY 可以达到同样效果 |
| **BIND_DEBUG_UNBIND** (0x0002) | 通常用于 Debug，在 unbindService 时候，会将服务信息保存并打印出来，这个标记很容易造成内存泄漏，应该准备用于 debugging |
| **BIND_NOT_FOREGROUND** (0x0002)| 表示不允许绑定操作将服务所在进程的优先级提升到前台进程优先级 |
| **BIND_ABOVE_CLIENT** (0x0008)| 设置服务的进程优先级高于客户端的优先级，只有当需要服务晚于客户端被销毁这种情况才这样设置。 |
|**BIND_ALLOW_OOM_MANAGEMENT** (0x0010)|保持服务受默认的服务管理器管理，当内存不足时候，会销毁服务|
|**BIND_WAIVE_PRIORITY** (0x0020)| 绑定操作不会影响到服务所在进程的优先级 |
|**BIND_IMPORTANT** (0x0040)| 标识服务对客户端是非常重要的，会将服务提升至前台进程优先级，通常情况下，即时客户端是前台优先级，服务最多也只能被提升至可见进程优先级 |
|**BIND_ADJUST_WITH_ACTIVITY** (0x0080)| 如果绑定来自 Activity，服务优先级的提高取决于Activity的进程优先级，使用这个标识后，会无视其他标识 |
|**BIND_ALLOW_WHITELIST_MANAGEMENT** (0x01000000)|   |
|**BIND_FOREGROUND_SERVICE_WHILE_AWAKE** (0x02000000)|   |
|**BIND_FOREGROUND_SERVICE** (0x04000000)|    |
|**BIND_TREAT_LIKE_ACTIVITY** (0x08000000)|    |
|**BIND_VISIBLE** (0x10000000)|   |
|**BIND_SHOWING_UI** (0x20000000)| 这种绑定方式会让目标服务所在进程显示出 UI 界面，当 UI 界面退出后会触发 UI_HIDDEN 类型的内存回收操作 |
|**BIND_NOT_VISIBLE** (0x40000000)|   |
|**BIND_EXTERNAL_SERVICE** (0x80000000)| **1、**通过这种方式绑定的服务是属于隔离的外部服务，服务将会被绑定到调用者应用的包内，而不是服务所在的应用包内！<br>**2、**使用这个标签进行绑定，服务的代码将会在调用者应用的包名和 uid 下执行！因为服务所在的进程是一个隔离进程，隔离进程是不能直接访问应用程序的数据的！|

具体标志位对于 Service 进程有什么影响，请看进程的优先级调度！！！

这里有一点，对于 BIND_AUTO_CREATE，如果服务没有被创建，会自动创建，而对于其他的 flags，是不会自动创建服务的！所以说我们从 Service 的生命周期的角度来说，可以把这些 flags 分为两大类，自动创建和非自动创建类！！

mBase 是 ContextImpl 对象，继续看！

## 1.2 ContextImpl.bindService

mMainThread 是应用进程主线程的 Handler！
```java
class ContextImpl extends Context {

    @Override
    public boolean bindService(Intent service, ServiceConnection conn,
            int flags) {
        warnIfCallingFromSystemProcess();

        return bindServiceCommon(service, conn, flags, mMainThread.getHandler(),
                Process.myUserHandle());
    }

    /** @hide */
    @Override
    public boolean bindServiceAsUser(Intent service, ServiceConnection conn, int flags,
            UserHandle user) {

        return bindServiceCommon(service, conn, flags, mMainThread.getHandler(), user);
    }

    /** @hide */
    @Override
    public boolean bindServiceAsUser(Intent service, ServiceConnection conn, int flags,
            Handler handler, UserHandle user) {
        if (handler == null) {
            throw new IllegalArgumentException("handler must not be null.");
        }

        return bindServiceCommon(service, conn, flags, handler, user);
    }
}
```

ContextImpl 和 ContextWrapper 的具体关系，请来看另一博文：Android 系统的 Context 分析，这里我们不再详细说明！

mUser：表示的是当前的设备 user！

最终，调用了 bindServiceCommon 方法；

## 1.3 ContextImpl.bindServiceCommon
```java
    private boolean bindServiceCommon(Intent service, ServiceConnection conn, int flags, Handler
            handler, UserHandle user) {

        IServiceConnection sd;

        if (conn == null) {
            throw new IllegalArgumentException("connection is null");
        }
        if (mPackageInfo != null) {

            // 获得本次连接的 InnerConnection 对象！
            sd = mPackageInfo.getServiceDispatcher(conn, getOuterContext(), handler, flags);

        } else {
            throw new RuntimeException("Not supported in system context");
        }

        validateServiceIntent(service);

        try {

            // 获得 ActivityClientRecord 对象！
            IBinder token = getActivityToken();

            // 这里的意思是对于 Android 版本小于 android.os.Build.VERSION_CODES.ICE_CREAM_SANDWICH
            // BIND_AUTO_CREATE 是不使用的，用 BIND_WAIVE_PRIORITY替换！
            if (token == null && (flags&BIND_AUTO_CREATE) == 0 && mPackageInfo != null
                    && mPackageInfo.getApplicationInfo().targetSdkVersion
                    < android.os.Build.VERSION_CODES.ICE_CREAM_SANDWICH) {

                flags |= BIND_WAIVE_PRIORITY;
            }

            service.prepareToLeaveProcess(this);
            
            // bind 通信，进入 ActivityManagerService！
            int res = ActivityManagerNative.getDefault().bindService(
                mMainThread.getApplicationThread(), getActivityToken(), service,
                service.resolveTypeIfNeeded(getContentResolver()),
                sd, flags, getOpPackageName(), user.getIdentifier());

            if (res < 0) {
                throw new SecurityException(
                        "Not allowed to bind to service " + service);
            }

            return res != 0;

        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }

```
getActivityToken 方法获得的是 ActivityClientRecord 对象，具体为什么是它，请去看 startActivity 相关的博文！！

### 1.3.1 LoadedApk.getServiceDispatcher

mPackageInfo 是 LoadedApk 类的实例，每一个进程都会有一个，这里调用了 LoadedApk 的 getServiceDispatcher 方法来获得一个 IServiceConnection 类型的对象，这里我们只需要知道，ServiceConnection 是应用进程中的创建的连接对象，ServiceDispatcher 用来托管 ServiceConnection 对象和 ServiceDispatcher.InnerConnection 对象的映射关系，InnerConnection 能够进行 binder 通信！！

这里会涉及一个集合：
```java
    private final ArrayMap<Context, ArrayMap<ServiceConnection, LoadedApk.ServiceDispatcher>> mServices
        = new ArrayMap<Context, ArrayMap<ServiceConnection, LoadedApk.ServiceDispatcher>>();
```
这个是 LoadedApk 的内部实例集合，每一个进程在创建后，都会有一个 LoadedApk 对象， mServices 集合的 key 值为 Context 对象，表示上下文运行环境，大家可以理解为特定的组件，value 是一个 ArrayMap 集合，封装了特定的组件所持有的所有服务连接对象 ServiceConnection 和服务分发对象 ServiceDispatcher！

```java
    public final IServiceConnection getServiceDispatcher(ServiceConnection c,
            Context context, Handler handler, int flags) {

        synchronized (mServices) {

            LoadedApk.ServiceDispatcher sd = null;

            ArrayMap<ServiceConnection, LoadedApk.ServiceDispatcher> map = mServices.get(context);
            
            // 如果有的话，就直接返回！
            if (map != null) {
                sd = map.get(c);
            }

            if (sd == null) {

                //【1】不然就创建新的 ServiceDispatcher，然后返回，内部会创建 InnerConnection 对象！
                sd = new ServiceDispatcher(c, context, handler, flags);

                if (map == null) {
                    map = new ArrayMap<ServiceConnection, LoadedApk.ServiceDispatcher>();
                    mServices.put(context, map);
                }

                //【2】添加到对应的 map 集合中！
                map.put(c, sd);

            } else {
                sd.validate(context, handler);
            }
            
            // 返回 InnerConnection 对象，其继承了 IServiceConnection.Stub！
            return sd.getIServiceConnection();
        }
    }
```
我们来看看 ServiceDispatcher 对象的创建：
```java
        ServiceDispatcher(ServiceConnection conn,
                Context context, Handler activityThread, int flags) {

            //【1】创建 InnerConnection 对象，用于跨进程调用！
            mIServiceConnection = new InnerConnection(this);

            // 初始化内部 ServiceConnection 对象！
            mConnection = conn;
            mContext = context;
            mActivityThread = activityThread;
            mLocation = new ServiceConnectionLeaked(null);
            mLocation.fillInStackTrace();
            mFlags = flags;
        }
```

接着再看看 InnerConnection 对象的创建！
```java
        InnerConnection(LoadedApk.ServiceDispatcher sd) {
           
            //【1】持有 ServiceDispatcher 的弱引用！
            mDispatcher = new WeakReference<LoadedApk.ServiceDispatcher>(sd);
        }
```
这里就看这么多！！

我们来他通过一张图看看绑定者进程的数据结构之间的关系：

![绑定者进程中的数据结构关系.png-164.8kB][3]

其中，mActiveConnections 表示的是处于活跃中的连接信息，这个我们后面再看！


接着，调用 ActivityManagerNative.getDefault() 方法，获得 AMS 的代理对象 ActivityManagerProxy，调用代理的 bindService 方法！

## 1.5 ActivityManagerP.bindService

```java
    public int bindService(IApplicationThread caller, IBinder token,
            Intent service, String resolvedType, IServiceConnection connection,
            int flags,  String callingPackage, int userId) throws RemoteException {

        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IActivityManager.descriptor);
        
        // 这里的 caller 是 ApplicationThread 对象！
        data.writeStrongBinder(caller != null ? caller.asBinder() : null);
        data.writeStrongBinder(token);

        service.writeToParcel(data, 0);
        data.writeString(resolvedType);

        // 这里的 connection 是 InnerConnection 对象！
        data.writeStrongBinder(connection.asBinder());

        data.writeInt(flags);
        data.writeString(callingPackage);
        data.writeInt(userId);

        mRemote.transact(BIND_SERVICE_TRANSACTION, data, reply, 0);

        reply.readException();
        int res = reply.readInt();
        data.recycle();
        reply.recycle();

        return res;
    }
```
通过 binder 进程间通信，进入系统进程，参数分析：

- IApplicationThread caller：调用者进程的 ApplicationThread 对象，实现了 IApplicationThread 接口；
- IBinder token：应用端的 mActivityToken；
- Intent service：启动的 intent
- String resolvedType：这个 intent 的 MIME 类型；
- IServiceConnection connection：服务连接对象 InnerConnection；
- String callingPackage：启动者所属包名；
- int userId：设备用户 id；


下面会进入系统进程！

# 2 系统进程

首先要进入 ActivityManagerN.onTransact 方法！
```java
        case BIND_SERVICE_TRANSACTION: {

            data.enforceInterface(IActivityManager.descriptor);
            
            //【1】获得 ApplicationThreadProxy 对象！
            IBinder b = data.readStrongBinder();
            IApplicationThread app = ApplicationThreadNative.asInterface(b);
            
            //【2】Activity 句柄！
            IBinder token = data.readStrongBinder();

            Intent service = Intent.CREATOR.createFromParcel(data);
            String resolvedType = data.readString();
            b = data.readStrongBinder();
            int fl = data.readInt();
            String callingPackage = data.readString();
            int userId = data.readInt();
            
            //【3】获得 InnerConnection 的代理对象！
            IServiceConnection conn = IServiceConnection.Stub.asInterface(b);
            
            // 进入 AMS!
            int res = bindService(app, token, service, resolvedType, conn, fl,
                    callingPackage, userId);

            reply.writeNoException();
            reply.writeInt(res);
            return true;
        }
```
继续看！

## 2.1 ActivityManagerS.bindService
```java
    public int bindService(IApplicationThread caller, IBinder token, Intent service,
            String resolvedType, IServiceConnection connection, int flags, String callingPackage,
            int userId) throws TransactionTooLargeException {

        enforceNotIsolatedCaller("bindService");

        if (service != null && service.hasFileDescriptors() == true) {
            throw new IllegalArgumentException("File descriptors passed in Intent");
        }

        if (callingPackage == null) {
            throw new IllegalArgumentException("callingPackage cannot be null");
        }

        synchronized(this) {
        
            // 进入 ActiveServices 中！！
            return mServices.bindServiceLocked(caller, token, service,
                    resolvedType, connection, flags, callingPackage, userId);
        }
    }
```

## 2.2 ActiveServices.bindServiceLocked

这个方法的逻辑比较长，我们来仔细看看！
```java
    int bindServiceLocked(IApplicationThread caller, IBinder token, Intent service,
            String resolvedType, final IServiceConnection connection, int flags,
            String callingPackage, final int userId) throws TransactionTooLargeException {

        if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "bindService: " + service
                + " type=" + resolvedType + " conn=" + connection.asBinder()
                + " flags=0x" + Integer.toHexString(flags));
        
        // 获得调用者进程 ProcessRecord 对象！
        final ProcessRecord callerApp = mAm.getRecordForAppLocked(caller);
        if (callerApp == null) {
            throw new SecurityException(
                    "Unable to find app for caller " + caller
                    + " (pid=" + Binder.getCallingPid()
                    + ") when binding service " + service);
        }

        // 如果 token 不为 null 的话，就获得其对应的 ActivityRecord 对象！
        ActivityRecord activity = null;
        if (token != null) {
            activity = ActivityRecord.isInStackLocked(token);
            if (activity == null) {
                Slog.w(TAG, "Binding with unknown activity: " + token);
                return 0;
            }
        }

        int clientLabel = 0;
        PendingIntent clientIntent = null;
        
        // 判断是否是系统进程！
        final boolean isCallerSystem = callerApp.info.uid == Process.SYSTEM_UID;

        if (isCallerSystem) {
        
            service.setDefusable(true);
            clientIntent = service.getParcelableExtra(Intent.EXTRA_CLIENT_INTENT);

            if (clientIntent != null) {
                clientLabel = service.getIntExtra(Intent.EXTRA_CLIENT_LABEL, 0);
                if (clientLabel != 0) {

                    service = service.cloneFilter();
                }
            }
        }

        // 如果设置了 Context.BIND_TREAT_LIKE_ACTIVITY 的标签，
        // 必须有权限 android.Manifest.permission.MANAGE_ACTIVITY_STACKS；
        if ((flags&Context.BIND_TREAT_LIKE_ACTIVITY) != 0) {
            mAm.enforceCallingPermission(android.Manifest.permission.MANAGE_ACTIVITY_STACKS,
                    "BIND_TREAT_LIKE_ACTIVITY");
        }

        // 非系统进程不能设置 BIND_ALLOW_WHITELIST_MANAGEMENT 的 flags！
        if ((flags & Context.BIND_ALLOW_WHITELIST_MANAGEMENT) != 0 && !isCallerSystem) {
            throw new SecurityException(
                    "Non-system caller " + caller + " (pid=" + Binder.getCallingPid()
                    + ") set BIND_ALLOW_WHITELIST_MANAGEMENT when binding service " + service);
        }

        // callerFg 表示的是前台调用或者是后台调用！
        final boolean callerFg = callerApp.setSchedGroup != ProcessList.SCHED_GROUP_BACKGROUND;
        
        // 表示是否 bind 的是 isolated， external 类型的服务！
        final boolean isBindExternal = (flags & Context.BIND_EXTERNAL_SERVICE) != 0;

        //【1】检索要 bind 的服务信息！
        ServiceLookupResult res =
            retrieveServiceLocked(service, resolvedType, callingPackage, Binder.getCallingPid(),
                    Binder.getCallingUid(), userId, true, callerFg, isBindExternal);

        if (res == null) {
            return 0;
        }
        if (res.record == null) {
            return -1;
        }
        
        // 获得服务的信息对象 ServiceRecord！
        ServiceRecord s = res.record;

        boolean permissionsReviewRequired = false;

        // 如果编译时，指定需要检测权限，那就先要通过权限的检测！
        if (Build.PERMISSIONS_REVIEW_REQUIRED) {
            if (mAm.getPackageManagerInternalLocked().isPermissionsReviewRequired(
                    s.packageName, s.userId)) {

                permissionsReviewRequired = true;

                // 对于前台调用才要校验权限！
                if (!callerFg) {
                    Slog.w(TAG, "u" + s.userId + " Binding to a service in package"
                            + s.packageName + " requires a permissions review");
                    return 0;
                }

                final ServiceRecord serviceRecord = s;
                final Intent serviceIntent = service;

                // 校验权限后的回调！
                RemoteCallback callback = new RemoteCallback(
                        new RemoteCallback.OnResultListener() {
                    @Override
                    public void onResult(Bundle result) {
                        synchronized(mAm) {
                            final long identity = Binder.clearCallingIdentity();
                            try {
                                if (!mPendingServices.contains(serviceRecord)) {
                                    return;
                                }

                                if (!mAm.getPackageManagerInternalLocked()
                                        .isPermissionsReviewRequired(
                                                serviceRecord.packageName,
                                                serviceRecord.userId)) {
                                    try {
                                        
                                        //
                                        bringUpServiceLocked(serviceRecord,
                                                serviceIntent.getFlags(),
                                                callerFg, false, false);
                                    } catch (RemoteException e) {
                                        /* ignore - local call */
                                    }
                                } else {
                                    unbindServiceLocked(connection);
                                }
                            } finally {
                                Binder.restoreCallingIdentity(identity);
                            }
                        }
                    }
                });
                
                // 创建 intent，启动 PackageInstaller，来校验权限！
                final Intent intent = new Intent(Intent.ACTION_REVIEW_PERMISSIONS);
                intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK
                        | Intent.FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS);
                intent.putExtra(Intent.EXTRA_PACKAGE_NAME, s.packageName);
                intent.putExtra(Intent.EXTRA_REMOTE_CALLBACK, callback);

                if (DEBUG_PERMISSIONS_REVIEW) {
                    Slog.i(TAG, "u" + s.userId + " Launching permission review for package "
                            + s.packageName);
                }

                mAm.mHandler.post(new Runnable() {
                    @Override
                    public void run() {
                    
                        // 启动权限校验界面！
                        mAm.mContext.startActivityAsUser(intent, new UserHandle(userId));
                    }
                });
            }
        }

        final long origId = Binder.clearCallingIdentity();

        try {
            
            //【2】取消服务的重启任务！
            if (unscheduleServiceRestartLocked(s, callerApp.info.uid, false)) {
                if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "BIND SERVICE WHILE RESTART PENDING: " + s);
            }
            
            // 如果启动 flags 设置了 Context.BIND_AUTO_CREATE 标签！
            if ((flags&Context.BIND_AUTO_CREATE) != 0) {
            
                // 更新服务的活跃时间！
                s.lastActivity = SystemClock.uptimeMillis();
                if (!s.hasAutoCreateConnections()) {

                    // 这是第一次 bind，通知服务状态监控器！
                    ServiceState stracker = s.getTracker();
                    if (stracker != null) {
                        stracker.setBound(true, mAm.mProcessStats.getMemFactorLocked(),
                                s.lastActivity);
                    }
                }
            }

            mAm.startAssociationLocked(callerApp.uid, callerApp.processName, callerApp.curProcState,
                    s.appInfo.uid, s.name, s.processName);
            
            //【3】创建 AppBindRecord 对象，用来记录绑定到这个服务的应用的信息！
            // s 是要绑定的 ServiceRecord!
            AppBindRecord b = s.retrieveAppBindingLocked(service, callerApp);
            
            // 创建 ConnectionRecord 对象，用来记本次的绑定连接！
            ConnectionRecord c = new ConnectionRecord(b, activity,
                    connection, flags, clientLabel, clientIntent);
            
            // connection 是一个从应用进程传过来的 InnerConnection 对象！
            IBinder binder = connection.asBinder();

            // s.connections 记录了这个服务的 IServiceConnection 对象和 ConnectionRecord 对象的对应关系！
            ArrayList<ConnectionRecord> clist = s.connections.get(binder);
            if (clist == null) {
                clist = new ArrayList<ConnectionRecord>();
                s.connections.put(binder, clist);
            }
            
            // 将 ConnectionRecord 保存到 s.connections 中！
            clist.add(c);
            
            // 将 ConnectionRecord 对象添加到对应的 AppBindRecord 的 connections 中！
            b.connections.add(c);
            
            // 如果 activity 不为 null，说明是 Activity 执行的 bind，所以要把 ConnectionRecord 添加到其 connections 中;
            if (activity != null) {
                if (activity.connections == null) {
                    activity.connections = new HashSet<ConnectionRecord>();
                }
                activity.connections.add(c);
            }
            
            // 将 ConnectionRecord 保存到调用者进程的 connections 列表中！
            b.client.connections.add(c);

            if ((c.flags&Context.BIND_ABOVE_CLIENT) != 0) {
                b.client.hasAboveClient = true;
            }
            if ((c.flags&Context.BIND_ALLOW_WHITELIST_MANAGEMENT) != 0) {
                s.whitelistManager = true;
            }
            if (s.app != null) {
                updateServiceClientActivitiesLocked(s.app, c, true);
            }

            // mServiceConnections 是 AS 的成员变量，记录着 IServiceConnection 和其 ConnectionRecord 的对应关系！
            clist = mServiceConnections.get(binder);
            if (clist == null) {
                clist = new ArrayList<ConnectionRecord>();
                mServiceConnections.put(binder, clist);
            }
            clist.add(c);
            
            // 如果启动的 flags 设置了 Context.BIND_AUTO_CREATE 标志位，那么就进入这个分支，
            // 如果服务没有被创建，就会自动创建服务！
            if ((flags&Context.BIND_AUTO_CREATE) != 0) {
                
                // 再次更新服务的活跃时间！
                s.lastActivity = SystemClock.uptimeMillis();
                
                //【4】启动该服务，如果服务所在进程未启动，还要启动相应进程，其返回值为 null，说明启动成功！
                if (bringUpServiceLocked(s, service.getFlags(), callerFg, false,
                        permissionsReviewRequired) != null) {

                    return 0;
                }
            }

            // 如果服务所在进程的 ProcessRecord 不为 null，就要更新服务的优先级以及 oomAdj 的值！
            if (s.app != null) {
                if ((flags&Context.BIND_TREAT_LIKE_ACTIVITY) != 0) {
                    s.app.treatLikeActivity = true;
                }
                if (s.whitelistManager) {
                    s.app.whitelistManager = true;
                }

                mAm.updateLruProcessLocked(s.app, s.app.hasClientActivities
                        || s.app.treatLikeActivity, b.client);
                mAm.updateOomAdjLocked(s.app);
            }

            // 【*】接着进入这个分支，这里我们在下面分析：
            if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "Bind " + s + " with " + b
                    + ": received=" + b.intent.received
                    + " apps=" + b.intent.apps.size()
                    + " doRebind=" + b.intent.doRebind);
            
            // 第一次自动创建方式的绑定，两个分支都不进入，多次自动创建方式的绑定，只会进入第二个分支！
            if (s.app != null && b.intent.received) {

                try {

                    //【5】服务已经在运行中了，那就尝试完成连接，会判断是否已经已有连接
                    // 如果有，就不会创建连接！
                    c.conn.connected(s.name, b.intent.binder);

                } catch (Exception e) {
                    Slog.w(TAG, "Failure sending service " + s.shortName
                            + " to connection " + c.conn.asBinder()
                            + " (in " + c.binding.client.processName + ")", e);
                }

                // If this is the first app connected back to this binding,
                // and the service had previously asked to be told when
                // rebound, then do so.
                // 【6】尝试拉起服务的 onRebind 方法！
                if (b.intent.apps.size() == 1 && b.intent.doRebind) {
                    requestServiceBindingLocked(s, b.intent, callerFg, true);
                }

            } else if (!b.intent.requested) {

                // 【7】尝试拉起服务的 onBind 方法！
                requestServiceBindingLocked(s, b.intent, callerFg, false);
            }

            getServiceMap(s.userId).ensureNotStartingBackground(s);

        } finally {
            Binder.restoreCallingIdentity(origId);
        }

        return 1;
    }

```
这个方法的逻辑很长，我们来简单总结一下重点！！

**注意**：

对于 Context.BIND_AUTO_CREATE 的方式，bringUpServiceLocked 方法中拉起 onCreate 和 onBind 方法的 Binder 通信是 IBinder.FLAGS_ONEWAY 方式，所以 Binder 服务端系统进程在触发方法后，无需等待，会立即向下执行！

**这里来重点说下最后的分支部分**：

这部分内容会涉及到下面的章节！！

- 对于 BIND_AUTO_CREATE 的这种 bindService 方式：
   - 第一次绑定时，进入该分支前， received 为 false 而 requested 为 true（requested 是在 bringUpServiceLocked -> requestServiceBindingLocked 被置为 true 的），所以下面 `IF` 和 `ELSE` 分支都不会进入！
   - 多次绑定时，因为第一次绑定连接已经生成，所以 received 和 requested 都会为 true，这里会进入第一个 `IF` 分支，尝试执行 ServiceConnection.onServiceConnected 方法，因为之前绑定已经存在，就不会执行！！！

- 对于非 BIND_AUTO_CREATE 的这种 bindService 方式：
   - 如果服务之前没有启动，那么不管绑定几次，都只是将绑定所数据绑定到相应的集合中，但 received 和 requested 都为 false！，所以会进入第二个 `ELSE` 分支，尝试拉起 onbind 方法，但是由于服务没有被创建和运行，r.app 和 r.app.thread 都为 null，所以不会继续执行！
   - 如果服务之前已经被启动了，我们知道对于这种情况，如果之前进行了 bindService 的操作， 因为系统已经保存了绑定信息，所以会立刻拉起服务的 onbind 方法，并执行 ServiceConnection.onServiceConnected 方法！
   - 如果服务之前已经被启动了，这是我们再执行 bindService 操作：
       - 第一次绑定，进入该分支前， received 和 requested 均为 false（这个第一种情况不同），这样会进入第二个 `ELSE` 分支，拉起服务的 onBind 方法，并执行 ServiceConnection.onServiceConnected 方法！
       - 多次绑定，进入该分支前， received 和 requested 均为 true，所以会进入第一个 `IF` 分支，尝试执行 ServiceConnection.onServiceConnected 方法，因为之前绑定已经存在，就不会执行！！！

整个流程看起来复杂，但是体现在生命周期上很简单，这里就不说了！！

然后，我们来看下这个方法中的重点！！

### 2.2.1 ActiveServices.retrieveServiceLocked

根据 intent 来查询被启动服务的 ServiceRecord 对象，参数传递：

- boolean createIfNeeded：传入的是 true；
- boolean callingFromFg：表示是从前台调用，还是后台调用；
- boolean isBindExternal：表示 bind 的是否是 isolated, external 类型的服务，我们这里默认为 false；

```java
    private ServiceLookupResult retrieveServiceLocked(Intent service,
            String resolvedType, String callingPackage, int callingPid, int callingUid, int userId,
            boolean createIfNeeded, boolean callingFromFg, boolean isBindExternal) {
        
        // 被绑定的服务！
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
                        
                        // 设置 intent 的 ComponentName 对象！
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

### 2.2.2 ActiveServices.unscheduleServiceRestartLocked

如果服务此时已经在重启列表中了，就要取消这个任务！

参数传入：

- boolean force：传入 false
```java
    private final boolean unscheduleServiceRestartLocked(ServiceRecord r, int callingUid,
            boolean force) {

        if (!force && r.restartDelay == 0) {
            return false;
        }

        // Remove from the restarting list; if the service is currently on the
        // restarting list, or the call is coming from another app, then this
        // service has become of much more interest so we reset the restart interval.
        boolean removed = mRestartingServices.remove(r);
        if (removed || callingUid != r.appInfo.uid) {
            
            // 重置重启计数！
            r.resetRestartCounter();
        }

        if (removed) {
            clearRestartingIfNeededLocked(r);
        }
        
        // 从 mAm.mHandler 中移除对一个的 ServiceStarter 对象！
        c.removeCallbacks(r.restarter);
        return true;
    }
```
因为当前服务要重新启动，所以之前的重启就要重置！

### 2.2.3 ServiceRecord.retrieveAppBindingLocked

这里是很关键的一段逻辑，创建了很多的对象，并相互引用：
```java
            // 3、创建 AppBindRecord 对象，用来记录绑定到这个服务的应用的信息！
            AppBindRecord b = s.retrieveAppBindingLocked(service, callerApp);
            
            // 创建 ConnectionRecord 对象，用来记录一个连接！
            ConnectionRecord c = new ConnectionRecord(b, activity,
                    connection, flags, clientLabel, clientIntent);
            
            // connection 是一个绑定者进程的 IServiceConnection.proxy 对象，和 InnerConnection 对应！！
            IBinder binder = connection.asBinder();

            // s.connections 记录了这个服务的 IServiceConnection 和 ConnectionRecord 的对应关系！
            ArrayList<ConnectionRecord> clist = s.connections.get(binder);
            if (clist == null) {
                clist = new ArrayList<ConnectionRecord>();
                s.connections.put(binder, clist);
            }
            
            // 将 ConnectionRecord 保存到 s.connections 中！
            clist.add(c);
            
            // 将 ConnectionRecord 对象添加到对应的 AppBindRecord 的 connections 中！
            b.connections.add(c);
            
            // 如果 activity 不为 null，说明是 Activity 执行的 bind，所以要把
            // 添加到其 connections 中;
            if (activity != null) {
                if (activity.connections == null) {
                    activity.connections = new HashSet<ConnectionRecord>();
                }
                activity.connections.add(c);
            }
            
            // 将 ConnectionRecord 保存到调用者进程的 connections 列表中！
            b.client.connections.add(c);

            if ((c.flags&Context.BIND_ABOVE_CLIENT) != 0) {
                b.client.hasAboveClient = true;
            }
            if ((c.flags&Context.BIND_ALLOW_WHITELIST_MANAGEMENT) != 0) {
                s.whitelistManager = true;
            }
            if (s.app != null) {
                updateServiceClientActivitiesLocked(s.app, c, true);
            }

            // mServiceConnections 是 AS 的成员变量，记录着 IServiceConnection 和其 ConnectionRecord 的对应关系！
            clist = mServiceConnections.get(binder);
            if (clist == null) {
                clist = new ArrayList<ConnectionRecord>();
                mServiceConnections.put(binder, clist);
            }

            clist.add(c);
```


这里调用了 ServiceRecord 的 retrieveAppBindingLocked 方法，参数 ProcessRecord app 表示绑定者所在的进程！
```java
    // in ServiceRecord
    public AppBindRecord retrieveAppBindingLocked(Intent intent,
            ProcessRecord app) {

        // 首先，创建 FilterComparison 来封装启动 Intent！
        // 并获得其对应的 IntentBindRecord 对象，这里可以看到，多次 bindService，只会保存第一次的信息！
        Intent.FilterComparison filter = new Intent.FilterComparison(intent);
        IntentBindRecord i = bindings.get(filter);
        
        // 创建对应的 IntentBindRecord 对象，将 FilterComparison 和 IntentBindRecord 的映射保存到 SR.bindings 中！
        if (i == null) {
            i = new IntentBindRecord(this, filter);
            bindings.put(filter, i);
        }
        
        // 创建 AppBindRecord 对象！
        AppBindRecord a = i.apps.get(app);
        if (a != null) {
            return a;
        }

        a = new AppBindRecord(this, i, app);
        i.apps.put(app, a);
        return a;
    }
```

这里会创建了一下如下对象 AppBindRecord，**用来表示被绑定的 Service 和 bind 它的应用进程 ProcessRecord  的关系**！
```java
final class AppBindRecord {
    final ServiceRecord service;  // 被 bind 的服务！
    final IntentBindRecord intent;    // IntentBindRecord 对象
    final ProcessRecord client;  // bind 发起端的进程！
    final ArraySet<ConnectionRecord> connections = new ArraySet<>(); // 连接到这个服务的所有 ConnectionRecord！
}
```

以及 IntentBindRecord 对象，**用来表示绑定 Service 的 intent 的信息**！
```java
final class IntentBindRecord {
    final ServiceRecord service;  // 被 bind 的服务！
    final Intent.FilterComparison intent; // intent 封装类，等价于 intent！
    
    // 用来托管所有使用这个 intent 来绑定服务的进程 ProcessRecord 和其 AppBindRecord 的映射关系！
    final ArrayMap<ProcessRecord, AppBindRecord> apps = new ArrayMap<ProcessRecord, AppBindRecord>(); 
    IBinder binder; // 当绑定 Service 成功后，onBind 会返回 Service 的“桩”，而这个 binder 是“桩”的客户端代理！
    boolean requested;
    boolean received;
    boolean hasBound;
    boolean doRebind;
}
```
以及 ConnectionRecord 对象，**用来表示绑定 Service 的 intent 的信息**！！
```java
final class ConnectionRecord {
    final AppBindRecord binding;    // AppBindRecord 对象！
    final ActivityRecord activity;  // bind 服务的 activity
    final IServiceConnection conn;  // 来自 bind 端的 IServiceConnection.proxy 对象！
    final int flags;                // bindService 的第三个参数！
    final PendingIntent clientIntent; // 发送的 intent！
    boolean serviceDead;            // 表示服务是否死亡！
}
```

除此之外，在 ServiceRecord 中:
```java
    // 所有活跃的 intent 和其对应的 IntentBindRecord 对象！
    final ArrayMap<Intent.FilterComparison, IntentBindRecord> bindings
            = new ArrayMap<Intent.FilterComparison, IntentBindRecord>();
     
    // 所有 IServiceConnection 对象和其对应的 ArrayList<ConnectionRecord>> 列表！                   
    final ArrayMap<IBinder, ArrayList<ConnectionRecord>> connections
            = new ArrayMap<IBinder, ArrayList<ConnectionRecord>>();
                            // IBinder -> ConnectionRecord of all bound clients
```

以及在 ActiveServices 中:
```java
    final ArrayMap<IBinder, ArrayList<ConnectionRecord>> mServiceConnections = new ArrayMap<>();
```

这里我们用一张图来看下这几个数据结构间的关系；

![系统进程中的数据结构关系.png-467.2kB][2]

可以看到，引用关系很复杂，这也是方便 AMS 在调度时，更快的查询到数据！！


## 2.3 AServices.bringUpServiceLocked

上面的代码中， **如果 flags 有设置 hasAutoCreateConnections 的话**，会进入 bringUpServiceLocked 这个方法，这里和前面的 startService 很类似了，启动服务！

参数分析：

- **ServiceRecord r**：服务的 ServiceRecord 对象；
- **int intentFlags**：启动服务的 intent 的 flag，通过 getFlags 获得；
- **boolean execInFg**：传入 callFg，表示本次启动是前台调用还是后台调用；
- **boolean whileRestarting**：表示是否是正在重启，传入 false；
- **boolean permissionsReviewRequired**：是否需要校验权限；

我们这里**假设 Service 所在的进程没有启动**！

bringUpServiceLocked 的方法

```java
    private String bringUpServiceLocked(ServiceRecord r, int intentFlags, boolean execInFg,
            boolean whileRestarting, boolean permissionsReviewRequired)
            throws TransactionTooLargeException {

        // 如果服务所在的进程已经被启动，那就进入 sendServiceArgsLocked 方法，
        // 但对于 bind，这个方法是不会拉起 onStartCommand 方法的，原因下面说！
        // 对于 bindService，如果服务所在进程已经被拉起，条件为 true，这里不会继续往下执行！
        if (r.app != null && r.app.thread != null) {
            sendServiceArgsLocked(r, execInFg, false);
            return null;
        }

        if (!whileRestarting && r.restartDelay > 0) { // 如果正在等待重启，就退出！
            return null;
        }

        if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "Bringing up " + r + " " + r.intent);

        // 服务要被启动了，所以要从重启 mRestartingServices 列表中移除它，并清除内部的启动计数！
        if (mRestartingServices.remove(r)) {

            r.resetRestartCounter();
            clearRestartingIfNeededLocked(r);
        }

        // 如果这个服务是被延迟启动，即 delayed 为 true，那就立刻启动它，从 mDelayedStartList 中移除它，并置 delayed 为 false；
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
            
            // 执行 bringDown 这个服务！
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

        // 获得要启动的服务的所在进程名；
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
                    
                    //【1】启动指定服务！
                    realStartServiceLocked(r, app, execInFg);

                    return null;
                } catch (TransactionTooLargeException e) {
                    throw e;

                } catch (RemoteException e) {
                    Slog.w(TAG, "Exception when starting service " + r.shortName, e);

                }

            }

        } else {
            app = r.isolatedProc;
        }

        // 如果服务所在的进程没有启动！
        if (app == null && !permissionsReviewRequired) {
        
            //【2】首先要启动服务所在的进程，这里请去看进程启动和创建的博文！
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
        
        // 将这个服务加入到 mPendingServices 中，表示该服务正在其所在的进程启动；
        if (!mPendingServices.contains(r)) {
            mPendingServices.add(r);
        }

        if (r.delayedStop) { // 如果服务需要被延迟停止，

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
**注意**

bindService 时候，如果设置了 Context.BIND_AUTO_CREATE 标志，就会调用 bringUpServiceLocked 方法！

然而对应 bind 来说，是不会执行 sendServiceArgsLocked 方法中的逻辑，不会拉起服务的 onStartCommand 方法的！

因为只有 start 的方式，才会将启动项目 StartItem 添加到  ServiceRecord 的 pendingStarts 集合中，而对于 bind，pendingStarts 集合的大小始终为 0，所以不会继续执行的！！

同时，对于 bindService，如果服务所在的进程已经被拉起，那么 bringUpServiceLocked 方法会在 sendServiceArgsLocked 只会返回，不会执行下面的逻辑，而是返回 bindServiceLocked ，继续往下执行！！

**方法流程总结**

1、如果已经启动，就会进入 realStartServiceLocked 方法，启动服务！
2、如果没有启动，就会 startProcessLocked 先启动服务所在进程，，最后进入 attachApplicationLocked 方法。然后还是会进入 realStartServiceLocked ！

所以最终调用的是 realStartServiceLocked 方法，它会拉起 Service 的 onCreate、onBind 方法！

这里我们先假设，服务所在的进程，还没有被启动！！！！

## 2.4 ActivityManagerS.attachApplicationLocked

当应用进程启动后，会通过 Binder 通信，调用 AMS.attachApplicationLocked，我们去看看：
```java
    private final boolean attachApplicationLocked(IApplicationThread thread,
            int pid) {

            ... ... ... ...
            
            // 这里会回到应用进程中，创建 Applicaiton 对象，并调用其 onCreate 方法，这里我们不看！
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

            // 每当 bindApplication 操作失败，则重新启动进程, 此处有可能会导致进程无限重启。
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

                 // 检查是否有 service 组件要在这个新进程中运行！
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
bindApplication 方法会回到应用进程中，创建 Applicaiton 对象，并调用其 onCreate 方法，这里我们不关注！！

这里会调用 ActiveServices 的 attachApplicationLocked 方法！


## 2.5 ActiveServices.attachApplicationLocked

当应用进程启动成功，并且 Application 对象已经创建，其 onCreate 方法已经调用后，就要启动指定的服务了！
```java
    boolean attachApplicationLocked(ProcessRecord proc, String processName)
            throws RemoteException {

        boolean didSomething = false;

        // 还记得 mPendingServices 吗？这里面保存着所有正在等待其所在的进程启动的服务！
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

                    if (!isServiceNeeded(sr, false, false)) {

                        bringDownServiceLocked(sr);
                    }
                }
            } catch (RemoteException e) {
                Slog.w(TAG, "Exception in new application when starting service "
                        + sr.shortName, e);
                throw e;
            }
        }

        // 接着启动那些需要重启的服务，重启的服务都会保存到 mRestartingServices 服务中！
        // 因为重启的时间可能还没有到，所以这里并不是立刻启动它们，而是启动了一个任务，交给了 AMS 的 MainHandler 去做！
        if (mRestartingServices.size() > 0) {
            ServiceRecord sr;
            for (int i=0; i<mRestartingServices.size(); i++) {
                sr = mRestartingServices.get(i);
                if (proc != sr.isolatedProc && (proc.uid != sr.appInfo.uid
                        || !processName.equals(sr.processName))) {
                    continue;
                }
                mAm.mHandler.removeCallbacks(sr.restarter);
                mAm.mHandler.post(sr.restarter);
            }
        }
        return didSomething;
    }
```

这里最终又回到了 realStartServiceLocked 方法！


## 2.6 AS.realStartServiceLocked

对于非隔离的进程，最后都会通过调用 realStartServiceLocked 方法来启动进程中的指定服务：
```java
    private final void realStartServiceLocked(ServiceRecord r,
            ProcessRecord app, boolean execInFg) throws RemoteException {

        if (app.thread == null) {
            throw new RemoteException();
        }

        if (DEBUG_MU)
            Slog.v(TAG_MU, "realStartServiceLocked, ServiceRecord.uid = " + r.appInfo.uid
                    + ", ProcessRecord.uid = " + app.uid);
                    
        // 跟新活跃时间！
        r.app = app;
        r.restartTime = r.lastActivity = SystemClock.uptimeMillis();
        
        // 将被启动服务的 ServiceRecord 对象添加到所属进程的 app.services 中！
        // 如果本次启动的是一个新的服务，newService 值为 true！
        final boolean newService = app.services.add(r);
        
        // 针对 onCreate 方法，设置超时处理！
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

            app.forceProcessStateUpTo(ActivityManager.PROCESS_STATE_SERVICE);
            
            //【1】通过 binder 通信，拉起服务的 onCreate 方法！
            app.thread.scheduleCreateService(r, r.serviceInfo,
                    mAm.compatibilityInfoForPackageLocked(r.serviceInfo.applicationInfo),
                    app.repProcState);

            r.postNotification();
            
            // 启动成功的话，置 created 为 true 方法；
            created = true;

        } catch (DeadObjectException e) {
            Slog.w(TAG, "Application dead when creating service " + r);
            mAm.appDiedLocked(app); // 如果在服务启动时，应用进程死了，调用 appDiedLocked 通知死亡仆告对象！
            throw e;

        } finally {
            if (!created) { // 如果启动失败，并且不是一个被销毁的服务，那就尝试重启他！

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

        //【2】执行 bind 操作，这里会拉起服务的 onBind 方法！！
        requestServiceBindingsLocked(r, execInFg);

        updateServiceClientActivitiesLocked(app, null, true);

        // 注意 startRequested 只有在 startService 的情况下被置为 true，所以这里是不会添加启动项的！
        // 即：r.pendingStarts 的大小为 0！！
        if (r.startRequested && r.callStart && r.pendingStarts.size() == 0) {
            r.pendingStarts.add(new ServiceRecord.StartItem(r, false, r.makeNextStartId(),
                    null, null));
        }
        // 这里也不会拉起 Service 的 onSrartCommand 方法，原因是 ：r.pendingStarts 的大小为 0，找不到启动项！！
        sendServiceArgsLocked(r, execInFg, true);

        if (r.delayed) {
            if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE, "REM FR DELAY LIST (new proc): " + r);
            getServiceMap(r.userId).mDelayedStartList.remove(r);
            r.delayed = false;
        }

        if (r.delayedStop) {
            r.delayedStop = false;
            if (r.startRequested) {
                if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE,
                        "Applying delayed stop (from start): " + r);
                stopServiceLocked(r);

            }
        }
    }

```

**注意**：
startRequested 只有在 startService 的情况下被置为 true，所以这里是不会添加启动项的！所以 sendServiceArgsLocked 也不会拉起  Service 的 onSrartCommand 方法，原因是 ：r.pendingStarts 的大小为 0，找不到启动项！！

这就要进入 requestServiceBindingsLocked 方法了！！！

### 2.6.1 ApplicationThreadP.scheduleCreateService

这里首先会拉起 Serivce 的 onCreate 方法！
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
            
            // 通过 Binder 通信！
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

进入应用进程，下面再看！！

### 2.6.2 ActiveServices.requestServiceBindingsLocked
```java
    private final void requestServiceBindingsLocked(ServiceRecord r, boolean execInFg)
            throws TransactionTooLargeException {
            
        // 遍历 r.bindings！
        for (int i=r.bindings.size()-1; i>=0; i--) {
            IntentBindRecord ibr = r.bindings.valueAt(i);
            
            // 【1】请求 bind 操作，默认 rebind 为 false！
            if (!requestServiceBindingLocked(r, ibr, execInFg, false)) {
                break;
            }
        }
    }
```
这里会遍历 r.bindings 集合，还记得这个集合吗？这个集合保存这个 bind 的 intent 和 IntentBindRecord 之间的映射关系，只有 bindService 的情况下，该集合才会有元素，startService 的情况下，该集合的 size 为 0，这就解释了 startService 为什么不会执行该方法了！！

继续来看！！

#### 2.6.2.1 ActiveServices.requestServiceBindingLocked

参数传入：

- boolean rebind：表示是否重新 bind，这里传入的是 false！

```java
    private final boolean requestServiceBindingLocked(ServiceRecord r, IntentBindRecord i,
            boolean execInFg, boolean rebind) throws TransactionTooLargeException {

        if (r.app == null || r.app.thread == null) {
            // 如果服务没有运行，那就会退出！
            return false;
        }
        
        // 这里 i.requested 和 rebind 均为 false， i.apps.size() 也是大于 0 的！
        if ((!i.requested || rebind) && i.apps.size() > 0) {
            try {
                
                // 为 bind 操作设置超时消息处理！！
                bumpServiceExecutingLocked(r, execInFg, "bind");
                
                // 将服务所在的进程状态更新为 ActivityManager.PROCESS_STATE_SERVICE！
                r.app.forceProcessStateUpTo(ActivityManager.PROCESS_STATE_SERVICE);
                
                // 【1】执行 bind 操作，跨进程拉起应用的 onBind 方法！
                r.app.thread.scheduleBindService(r, i.intent.getIntent(), rebind,
                        r.app.repProcState);
                
                // 对于 bindService，因为 rebind 为 false，所以这里会把 requested 置为 true！
                if (!rebind) {
                    i.requested = true;
                }
                
                // 将 i.hasBound 置为 true，表示已经调用了
                i.hasBound = true;
                i.doRebind = false;

            } catch (TransactionTooLargeException e) {

                // Keep the executeNesting count accurate.
                if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "Crashed while binding " + r, e);
                final boolean inDestroying = mDestroyingServices.contains(r);
                serviceDoneExecutingLocked(r, inDestroying, inDestroying);
                throw e;
            } catch (RemoteException e) {

                if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "Crashed while binding " + r);
                // Keep the executeNesting count accurate.
                final boolean inDestroying = mDestroyingServices.contains(r);
                serviceDoneExecutingLocked(r, inDestroying, inDestroying);
                return false;
            }
        }
        return true;
    }
```

thread 是 ApplicationThreadProxy 代理对象！！


#### 2.6.2.2 ApplicationThreadP.scheduleBindService

这里的 IBinder token 是 ServiceRecord 对象!!
```java
    public final void scheduleBindService(IBinder token, Intent intent, boolean rebind,
            int processState) throws RemoteException {
        Parcel data = Parcel.obtain();
        data.writeInterfaceToken(IApplicationThread.descriptor);
        data.writeStrongBinder(token);
        intent.writeToParcel(data, 0);
        data.writeInt(rebind ? 1 : 0);
        data.writeInt(processState);
        
        // binder 通信！
        mRemote.transact(SCHEDULE_BIND_SERVICE_TRANSACTION, data, null,
                IBinder.FLAG_ONEWAY);
        data.recycle();
    }
```
可以看到，拉起服务的 onCreate 方法和 onBind 方法的 binder 通信都是 IBinder.FLAG_ONEWAY 的！也就是说 Binder 服务端无需等待 BInder 客户端的操作返回结果，只要 Binder 客户端一收到 Binder服务端传送的命令和数据后，Binder 服务端会立即向下执行！

下面，进入被绑定者进程！

# 3 被绑定者进程

## 3.1 ApplicationThread.scheduleCreateService
```java
        public final void scheduleCreateService(IBinder token,
                ServiceInfo info, CompatibilityInfo compatInfo, int processState) {

            updateProcessState(processState, false);
            
            // 创建了一个 CreateServiceData 对象！
            CreateServiceData s = new CreateServiceData();

            s.token = token;
            s.info = info;
            s.compatInfo = compatInfo;

            sendMessage(H.CREATE_SERVICE, s);
        }
```
这里和 startService 一样，不多说了！

## 3.2 ApplicationThread.scheduleBindService
```java
        public final void scheduleBindService(IBinder token, Intent intent,
                boolean rebind, int processState) {
            updateProcessState(processState, false);
            
            // 创建了一个 BindServiceData 对象！
            BindServiceData s = new BindServiceData();
            s.token = token;
            s.intent = intent;
            s.rebind = rebind;

            if (DEBUG_SERVICE)
                Slog.v(TAG, "scheduleBindService token=" + token + " intent=" + intent + " uid="
                        + Binder.getCallingUid() + " pid=" + Binder.getCallingPid());

            sendMessage(H.BIND_SERVICE, s);
        }
```
这里创建了一个 BindServiceData 对象！
```java
    static final class BindServiceData {
        IBinder token;
        Intent intent;
        boolean rebind;

        public String toString() {
            return "BindServiceData{token=" + token + " intent=" + intent + "}";
        }
    }
```
很简单，不多说了！

接着，发送 H.BIND_SERVICE 方法，进入主线程的 Handler！

## 3.3 ActivityThread.H
```java
    private class H extends Handler {
        public void handleMessage(Message msg) {
            if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
            switch (msg.what) {
                
                // 这里是处理 create service 消息；
                case CREATE_SERVICE:
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, ("serviceCreate: " + String.valueOf(msg.obj)));
                    
                    handleCreateService((CreateServiceData)msg.obj);
                    
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                    break;
                    
                // 这里是处理 bind service 消息；
                case BIND_SERVICE:
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "serviceBind");
                    
                    handleBindService((BindServiceData)msg.obj);
                    
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                    break;
  
            }

            Object obj = msg.obj;
            if (obj instanceof SomeArgs) {
                ((SomeArgs) obj).recycle();
            }

            if (DEBUG_MESSAGES) Slog.v(TAG, "<<< done: " + codeToString(msg.what));
        }
    }

```
不多说了！！

### 3.3.1 ActivityThread.handleCreateService
```java
    private void handleCreateService(CreateServiceData data) {
        
        // 如果此时准备要 GC，那就跳过本次 GC！
        unscheduleGcIdler();
        
        // 获得应用程序的加载信息，先会从缓存中获取，找不到才创建；
        LoadedApk packageInfo = getPackageInfoNoCheck(
                data.info.applicationInfo, data.compatInfo);

        Service service = null;
        try {
        
            // 通过反射创建要启动的服务的实例；
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
            
            // 创建 Context 对象；
            ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
            context.setOuterContext(service);
            
            // 返回该进程的 Application 对象！
            Application app = packageInfo.makeApplication(false, mInstrumentation);

            // 将服务实例和 context，Application 进行绑定；
            service.attach(context, this, data.info.name, data.token, app,
                    ActivityManagerNative.getDefault());

            // 调用了服务的 onCreate 方法；
            service.onCreate();
            
            // 将创建的服务实例保存到 ActivityThread 的托管集合中！
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
成功地拉起了 Service 的 onCreate 方法！最后有调用了 ActivityManagerNative.getDefault().serviceDoneExecuting，这个和 startService 的流程一样，这里就不多数了！

### 3.3.2 ActivityThread.handleBindService
```java
    private void handleBindService(BindServiceData data) {
        Service s = mServices.get(data.token);

        if (DEBUG_SERVICE)
            Slog.v(TAG, "handleBindService s=" + s + " rebind=" + data.rebind);
        if (s != null) {
            try {
                data.intent.setExtrasClassLoader(s.getClassLoader());
                data.intent.prepareToEnterProcess();
                try {
                    if (!data.rebind) {  // 如果 rebind 为 false！！
                    
                        // 调用服务的 onBind 方法，返回值是 Service 的中定义的 “桩”！
                        IBinder binder = s.onBind(data.intent);
                        
                        // 这里回到了系统进程！
                        ActivityManagerNative.getDefault().publishService(
                                data.token, data.intent, binder);

                    } else { // 如果 rebind 为 true！！
                        
                        // 拉起服务的 onRebind 方法！
                        s.onRebind(data.intent);
                        
                        // 这里回到了系统进程！
                        ActivityManagerNative.getDefault().serviceDoneExecuting(
                                data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
                    }
                    ensureJitEnabled();

                } catch (RemoteException ex) {
                    throw ex.rethrowFromSystemServer();
                }
            } catch (Exception e) {
                if (!mInstrumentation.onException(s, e)) {
                    throw new RuntimeException(
                            "Unable to bind to service " + s
                            + " with " + data.intent + ": " + e.toString(), e);
                }
            }
        }
    }
```

这里，我们先默认是第一次绑定，rebind 为 false，我们在重写 Service 的 onBind 方法后，会返回一个 Stub 类型的对象，这个是 Service 这边定义的 "桩"，我们继续看！！！！

## 3.4 ActivityManagerProxy

这里不多说了！

```java
    public void serviceDoneExecuting(IBinder token, int type, int startId,
            int res) throws RemoteException {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IActivityManager.descriptor);
        data.writeStrongBinder(token);
        
        // 这里的 type 是 SERVICE_DONE_EXECUTING_ANON 
        data.writeInt(type);
        data.writeInt(startId);
        data.writeInt(res);

        mRemote.transact(SERVICE_DONE_EXECUTING_TRANSACTION, data, reply, IBinder.FLAG_ONEWAY);

        reply.readException();
        data.recycle();
        reply.recycle();
    }

    public void publishService(IBinder token,
            Intent intent, IBinder service) throws RemoteException {

        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IActivityManager.descriptor);
        data.writeStrongBinder(token);
        intent.writeToParcel(data, 0);
        data.writeStrongBinder(service);

        mRemote.transact(PUBLISH_SERVICE_TRANSACTION, data, reply, 0);

        reply.readException();
        data.recycle();
        reply.recycle();
    }

```

# 4 系统进程

进入 AMS，首先会进入 ActivityManagerN 的 onTransact 方法!

## 4.1 ActivityManagerN.onTransact
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

        case PUBLISH_SERVICE_TRANSACTION: {
            data.enforceInterface(IActivityManager.descriptor);
            IBinder token = data.readStrongBinder();
            Intent intent = Intent.CREATOR.createFromParcel(data);
            IBinder service = data.readStrongBinder();
            
            // 进入 AMS !
            publishService(token, intent, service);
            reply.writeNoException();
            return true;
        }
```

### 4.1.1 ActivityManagerS.serviceDoneExecuting
```java
    public void serviceDoneExecuting(IBinder token, int type, int startId, int res) {
        synchronized(this) {

            if (!(token instanceof ServiceRecord)) {
                Slog.e(TAG, "serviceDoneExecuting: Invalid service token=" + token);
                throw new IllegalArgumentException("Invalid service token");
            }
            mServices.serviceDoneExecutingLocked((ServiceRecord)token, type, startId, res);
        }
    }
```
最后进入 AS.serviceDoneExecutingLocked 方法！

#### 4.1.1.1 ActiveServices.serviceDoneExecutingLocked

参数传递：

- int type：这里的 type 是 `SERVICE_DONE_EXECUTING_ANON`  
```java
    void serviceDoneExecutingLocked(ServiceRecord r, int type, int startId, int res) {
        boolean inDestroying = mDestroyingServices.contains(r);
        if (r != null) {
            if (type == ActivityThread.SERVICE_DONE_EXECUTING_START) {
            
                ... ... ... // 不进入，不看
                
            } else if (type == ActivityThread.SERVICE_DONE_EXECUTING_STOP) {
            
                ... ... ... // 不进入，不看
                
            }
            final long origId = Binder.clearCallingIdentity();
            
            // 进入 serviceDoneExecutingLocked 方法，看
            serviceDoneExecutingLocked(r, inDestroying, inDestroying);

            Binder.restoreCallingIdentity(origId);
        } else {
            Slog.w(TAG, "Done executing unknown service from pid "
                    + Binder.getCallingPid());
        }
    }
```

### 4.1.2 ActivityManagerS.publishService
```JAVA
    public void publishService(IBinder token, Intent intent, IBinder service) {
        // Refuse possible leaked file descriptors
        if (intent != null && intent.hasFileDescriptors() == true) {
            throw new IllegalArgumentException("File descriptors passed in Intent");
        }

        synchronized(this) {
            // 这里不多说了！！
            if (!(token instanceof ServiceRecord)) {
                throw new IllegalArgumentException("Invalid service token");
            }
            
            // 调用 ActiveServices 的 publishServiceLocked 方法！
            mServices.publishServiceLocked((ServiceRecord)token, intent, service);
        }
    }
```

继续看：
#### 4.1.2.1 ActiveServices.publishService
```java
    void publishServiceLocked(ServiceRecord r, Intent intent, IBinder service) {
        final long origId = Binder.clearCallingIdentity();
        try {
    
            if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "PUBLISHING " + r
                    + " " + intent + ": " + service);
                    
            if (r != null) {

                // 尝试找到 intent 对应的 IntentBindRecord 对象!
                Intent.FilterComparison filter
                        = new Intent.FilterComparison(intent);
                IntentBindRecord b = r.bindings.get(filter);
                
                if (b != null && !b.received) {  // 这里的 received 还是 false!
                    
                    // 将服务的 “桩” 保存到！
                    b.binder = service;

                    // 将 IntentBindRecord.requested 设为 true！
                    b.requested = true;
                    
                    // 将 IntentBindRecord.received 设为 true！
                    b.received = true;
    
                    for (int conni=r.connections.size()-1; conni>=0; conni--) {
                        ArrayList<ConnectionRecord> clist = r.connections.valueAt(conni);
                        for (int i=0; i<clist.size(); i++) {
                            ConnectionRecord c = clist.get(i);

                            if (!filter.equals(c.binding.intent.intent)) {
                                if (DEBUG_SERVICE) Slog.v(
                                        TAG_SERVICE, "Not publishing to: " + c);
                                if (DEBUG_SERVICE) Slog.v(
                                        TAG_SERVICE, "Bound intent: " + c.binding.intent.intent);
                                if (DEBUG_SERVICE) Slog.v(
                                        TAG_SERVICE, "Published intent: " + intent);
                                continue;
                            }

                            if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "Publishing to: " + c);
                            try {
                            
                                // 通过 Bind 回调绑定者进程的 ServcieConnection 的 onServiceDisconnected 方法！
                                // 注意这个方法是非阻塞的，是异步的，因为有 oneway 的标志！！
                                c.conn.connected(r.name, service);

                            } catch (Exception e) {
                                Slog.w(TAG, "Failure sending service " + r.name +
                                      " to connection " + c.conn.asBinder() +
                                      " (in " + c.binding.client.processName + ")", e);
                            }

                        }
                    }
                }
                
                // 接着，调用 serviceDoneExecutingLocked 方法，表示执行完成！
                serviceDoneExecutingLocked(r, mDestroyingServices.contains(r), false);
            }
        } finally {
            Binder.restoreCallingIdentity(origId);
        }
    }
```
这里会继续通过 Bind 回调绑定者进程的 ServcieConnection 的 onServiceDisconnected 方法！这个方法是非阻塞的，是异步的，因为有 oneway 的标志！！

c.conn 是 IServiceConnection.proxy ，即服务连接代理对象！！！这里，就要回到绑定者进程，我们后面再看！！


### 4.1.3 ActivityManagerS.serviceDoneExecutingLocked
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
                r.app.executingServices.remove(r);

                if (r.app.executingServices.size() == 0) {
                    if (DEBUG_SERVICE || DEBUG_SERVICE_EXECUTING) Slog.v(TAG_SERVICE_EXECUTING,
                            "No more executingServices of " + r.shortName);
                            
                    // 移除 onCreate / onBind 方法的超时处理任务！
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

            if (finishing) {
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
这里很简单了，就不详细说了！！

# 5 绑定者进程

下面来看看绑定者进程的 connected 方法的回调问题！！

IServiceConnection.aild 文件中只定义了一个接口：

```java
import android.content.ComponentName;

/** @hide */
oneway interface IServiceConnection {
    void connected(in ComponentName name, IBinder service);
}
```
可以看到，这个方法是 oneway 类型的，就是说，binder 通信是非阻塞的！

## 5.1 InnerConnection.connected

进入 LoadedApk 的 InnerConnection 类！
```java
        private static class InnerConnection extends IServiceConnection.Stub {
            final WeakReference<LoadedApk.ServiceDispatcher> mDispatcher;

            InnerConnection(LoadedApk.ServiceDispatcher sd) {
                mDispatcher = new WeakReference<LoadedApk.ServiceDispatcher>(sd);
            }
            
            // 接着，调用 ServiceDispatcher 的 connected 方法！
            public void connected(ComponentName name, IBinder service) throws RemoteException {
                LoadedApk.ServiceDispatcher sd = mDispatcher.get();
                if (sd != null) {
                    sd.connected(name, service);
                }
            }
        }
```

## 5.2 ServiceDispatcher.connected
```java
        public void connected(ComponentName name, IBinder service) {
            
            // 执行一个 RunConnection 的 Runnable！
            if (mActivityThread != null) {
                
                mActivityThread.post(new RunConnection(name, service, 0));
            } else {
                doConnected(name, service);
            }
        }
```

## 5.3 ServiceDispatcher.RunConnection

参数传递：

- int command：0；
```java
        private final class RunConnection implements Runnable {
            RunConnection(ComponentName name, IBinder service, int command) {
                mName = name;
                mService = service;
                mCommand = command;
            }

            public void run() {
                if (mCommand == 0) {
                    
                    // 执行连接！
                    doConnected(mName, mService);

                } else if (mCommand == 1) {
                    doDeath(mName, mService);
                }
            }

            final ComponentName mName;
            final IBinder mService;
            final int mCommand;
        }
```
继续看：

## 5.4 ServiceDispatcher.doConnected
```java
        public void doConnected(ComponentName name, IBinder service) {
            ServiceDispatcher.ConnectionInfo old;
            ServiceDispatcher.ConnectionInfo info;

            synchronized (this) {
                // 当 mForgotten 为 true，表示之前执行了 unBindService 操作！
                if (mForgotten) {
                    // We unbound before receiving the connection; ignore
                    // any connection received.
                    return;
                }
                
                // 先尝试获得旧的可用连接，如果已有，就退出！
                // 所以说，对于多次 bindService，onServiceConnected 只会执行一次！
                old = mActiveConnections.get(name);
                if (old != null && old.binder == service) {
                    // Huh, already have this one.  Oh well!
                    return;
                }

                // service 是被绑定的 Service 的桩对应的代理！
                if (service != null) {

                    // 创建一个连接信息对象！
                    info = new ConnectionInfo();

                    //将远程 Service 代理保存到 binder 类中！
                    info.binder = service;
                    
                    // 设置死亡监听器
                    info.deathMonitor = new DeathMonitor(name, service);

                    try {
                      
                        service.linkToDeath(info.deathMonitor, 0);
                        
                        // 将本次的连接信息保存到 mActiveConnections 中！
                        mActiveConnections.put(name, info);

                    } catch (RemoteException e) {

                        // This service was dead before we got it...  just
                        // don't do anything with it.
                        mActiveConnections.remove(name);
                        return;
                    }

                } else {
                    // The named service is being disconnected... clean up.
                    mActiveConnections.remove(name);
                }

                if (old != null) {
                    old.binder.unlinkToDeath(old.deathMonitor, 0);
                }
            }

            // If there was an old service, it is now disconnected.
            if (old != null) {
                mConnection.onServiceDisconnected(name);
            }
            // If there is a new service, it is now connected.
            if (service != null) {
            
                // 调用了 Connection 的 onServiceConnected 方法！
                mConnection.onServiceConnected(name, service);
            }
        }
```
最后，我们看到，调用了 mConnection.onServiceConnected，这个方法大家很熟悉了，不多说了！


# 6 总结

## 6.1 数据结构总结

> 绑定者进程的数据结构关系图

<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/23/绑定者进程中的数据结构关系.png" alt="绑定者进程中的数据结构关系.png-164.8kB" style="zoom: 33%;" />

一个应用进程，只会有一个 InnerConnection 对象！

> 系统进程的数据结构关系图

<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/23/系统进程中的数据结构关系.png" alt="系统进程中的数据结构关系.png-467.2kB" style="zoom: 33%;" /> 

## 6.2 binder 通信总结

## 6.3 生命周期总结

- 对于**只设置了 BIND_AUTO_CREATE 标志位**的情况，只调用 bindService，Service 的周期如下：

  - 服务没有被创建：
    ```java
    Service.onCreate() -> Service.onBind() -> ServiceConnection.onServiceConnected()
    ```
  - 服务已经被创建：
    ```java
    Service.onBind() -> ServiceConnection.onServiceConnected()
    ```

- 对于**没有设置 BIND_AUTO_CREATE 标志，但设置了其他标志的情况**，只调用 bindService，Service 的周期如下：

  - 服务没有被创建：
    ```java
    // 无法实现 bind 操作, 但是会将本次 bind 操作的信息保存到系统进程中！
    ```
  - 服务之前已经被创建（之前通过 startService 启动服务，或者其他组件通过 BIND_AUTO_CREATE 方式绑定拉起）：
    ```java
    Service.onBind() -> ServiceConnection.onServiceConnected()
    ```
  - 服务之后才被创建（之后通过 startService 启动服务，或者其他组件通过 BIND_AUTO_CREATE 方式绑定拉起）：
    ```java
    Service.onCreate() -> Service.onBind() (-> Service.onStartCommand) -> ServiceConnection.onServiceConnected()
    ```

对于第三中情况，是不是很奇怪？原因很简单，如果服务之前没有被创建，调用 bindService（只设置了其他标志），是不会拉起 Service.onBind 方法的，但是会把本次 bind 的信息保存相应的集合中，当服务被 startService 或者通过 BIND_AUTO_CREATE 方式绑定拉起后，会对之前的 bind 信息处理，返回之前 bind 对应的代理对象！！详情请参考 startService 篇！

- 对于多次调用 bindService

  - 设置了 BIND_AUTO_CREATE 标志！
    ```java
    Service.onCreate()(once) -> Service.onBind()(once) -> ServiceConnection.onServiceConnected()（once）
    ```
    如果服务已经被创建的话，那 onCreate 方法是不会调用的，直接调用 onBind 方法！
    
  - 未设置 BIND_AUTO_CREATE 标志！ 
     - 服务没有被启动！
    ```java
    // 无法实现 bind 操作, 但是会将本次 bind 操作的信息保存到系统进程中，等到服务被 startService 或者其他自动创建类型的 bindService 拉起后才会执行如下周期：
    Service.onBind(once) -> ServiceConnection.onServiceConnected()(once)
    ```
     - 服务已经被启动！
    ```java
     Service.onBind()(once) -> ServiceConnection.onServiceConnected()(once)
    ```
    

以上就是 bindService 相关的生命周期，有不对的地方，欢迎指正，谢谢！


## 6.4 相关 Log 参考
```shell
07-04 17:24:21.523  1761  4194 V ActiveServices: bindService: Intent { cmp=com.cooqi.servicedemo/.service.AService } type=null conn=android.os.BinderProxy@abcfcca flags=0x1
07-04 17:24:21.524  1761  4194 V ActiveServices: retrieveServiceLocked: Intent { cmp=com.cooqi.servicedemo/.service.AService } type=null callingUid=10106
07-04 17:24:21.525  1761  4194 V ActivityManagerService: processName: com.cooqi.servicedemo uid 10106 keepIfLarge false
07-04 17:24:21.526  1761  4194 V ActiveServices: Bringing up ServiceRecord{7454958 u0 com.cooqi.servicedemo/.service.AService} android.content.Intent$FilterComparison@7df9497b
07-04 17:24:21.526  1761  4194 V ActivityManagerService: processName: com.cooqi.servicedemo uid 10106 keepIfLarge false
07-04 17:24:21.527  1761  4194 V ActiveServices_MU: bringUpServiceLocked: appInfo.uid=10106 app=ProcessRecord{39802a4 7994:com.cooqi.servicedemo/u0a106}
07-04 17:24:21.527  1761  4194 V ActiveServices_MU: realStartServiceLocked, ServiceRecord.uid = 10106, ProcessRecord.uid = 10106
07-04 17:24:21.527  1761  4194 V ActiveServices: >>> EXECUTING create of ServiceRecord{7454958 u0 com.cooqi.servicedemo/.service.AService} in app ProcessRecord{39802a4 7994:com.cooqi.servicedemo/u0a106}
07-04 17:24:21.527  1761  4194 V ActiveServices: bumpServiceExecutingLocked r.executeNesting 0
07-04 17:24:21.527  1761  4194 V ActiveServices: bumpServiceExecutingLocked r.app.executingServices.size() 1
07-04 17:24:21.544  1761  4194 D ActivityManagerService: oom: memFactor=0 last=0 allowLow=false numProcs=66 last=66
07-04 17:24:21.545  1761  4194 D ActivityManagerService: Did OOM ADJ in 18ms
07-04 17:24:21.546  1761  4194 V ActiveServices: >>> EXECUTING bind of ServiceRecord{7454958 u0 com.cooqi.servicedemo/.service.AService} in app ProcessRecord{39802a4 7994:com.cooqi.servicedemo/u0a106}
07-04 17:24:21.547  1761  4194 V ActiveServices: bumpServiceExecutingLocked r.executeNesting 1
07-04 17:24:21.547  7994  8006 V ActivityThread: SCHEDULE 114 CREATE_SERVICE: 0 / CreateServiceData{token=android.os.BinderProxy@23ffb1d className=com.cooqi.servicedemo.service.AService packageName=com.cooqi.servicedemo intent=null}
07-04 17:24:21.547  1761  4194 V ActiveServices: Bind ServiceRecord{7454958 u0 com.cooqi.servicedemo/.service.AService} with AppBindRecord{6e93ab1 com.cooqi.servicedemo/.service.AService:com.cooqi.servicedemo}: received=false apps=1 doRebind=false
07-04 17:24:21.548  7994  8006 V ActivityThread: scheduleBindService token=android.os.BinderProxy@23ffb1d intent=Intent { cmp=com.cooqi.servicedemo/.service.AService } uid=1000 pid=0
07-04 17:24:21.548  7994  8006 V ActivityThread: SCHEDULE 121 BIND_SERVICE: 0 / BindServiceData{token=android.os.BinderProxy@23ffb1d intent=Intent { cmp=com.cooqi.servicedemo/.service.AService }}
07-04 17:24:21.552  7994  7994 V ActivityThread: >>> handling: CREATE_SERVICE
07-04 17:24:21.552  7994  7994 V ActivityThread: Creating service com.cooqi.servicedemo.service.AService
07-04 17:24:21.553  7994  7994 D AService: onCreate
07-04 17:24:21.553  7994  7994 V ActivityThread: <<< done: CREATE_SERVICE
07-04 17:24:21.553  7994  7994 V ActivityThread: >>> handling: BIND_SERVICE
07-04 17:24:21.554  7994  7994 V ActivityThread: handleBindService s=com.cooqi.servicedemo.service.AService@92ef192 rebind=false
07-04 17:24:21.554  7994  7994 D AService: onBind
07-04 17:24:21.554  1761  4106 V ActiveServices: serviceDoneExecutingLocked ServiceRecord= ServiceRecord{7454958 u0 com.cooqi.servicedemo/.service.AService} type= 0 startId= 0 res= 0
07-04 17:24:21.554  1761  4106 V ActiveServices: <<< DONE EXECUTING ServiceRecord{7454958 u0 com.cooqi.servicedemo/.service.AService}: nesting=2, inDestroying=false, app=ProcessRecord{39802a4 7994:com.cooqi.servicedemo/u0a106}
07-04 17:24:21.555  1761  3764 V ActiveServices: PUBLISHING ServiceRecord{7454958 u0 com.cooqi.servicedemo/.service.AService} Intent { cmp=com.cooqi.servicedemo/.service.AService }: android.os.BinderProxy@7161b96
07-04 17:24:21.556  1761  3764 V ActiveServices: Publishing to: ConnectionRecord{5b95e3b u0 CR com.cooqi.servicedemo/.service.AService:@abcfcca}
07-04 17:24:21.556  1761  3764 V ActiveServices: <<< DONE EXECUTING ServiceRecord{7454958 u0 com.cooqi.servicedemo/.service.AService}: nesting=1, inDestroying=false, app=ProcessRecord{39802a4 7994:com.cooqi.servicedemo/u0a106}
07-04 17:24:21.556  1761  3764 V ActiveServices: Nesting at 0 of com.cooqi.servicedemo/.service.AService
07-04 17:24:21.556  1761  3764 V ActiveServices: r.app.executingServices.size(): 0
07-04 17:24:21.556  1761  3764 V ActiveServices: No more executingServices of com.cooqi.servicedemo/.service.AService
07-04 17:24:21.557  7994  7994 V ActivityThread: <<< done: BIND_SERVICE
07-04 17:24:21.561  7994  7994 D MainActivity: onServiceConnected
```




http://www.tuicool.com/articles/67zyea
