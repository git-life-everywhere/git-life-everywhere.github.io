# Serivce 篇 5 - unbindService 流程分析
title: Serivce 篇 5 - unbindService 流程分析
date: 2016/06/23 20:46:25
catalog: true
categories:
- AndroidFramework源码分析
- Service服务
tags: Service服务
---

本文基于 Android 7.1.1 源码分析，转载请说明出处！

# 0 综述

我们通过 bindService 绑定的服务，需要通过 unbindService 来解除绑定：

```java
context.unbindService(conn);
```

以前我们只是会调用，但是其底层的调用到底是什么样的呢？知其然知其所以然，今天我们就来学习下 unbindService 的过程！

# 1 发起端进程

## 1.1 ContextWrapper.unbindService


```java
    @Override
    public void unbindService(ServiceConnection conn) {

        mBase.unbindService(conn);
    }
```

`ContextWrapper` 提供了两个方法来启动 `Service`，其中一个是隐藏方法：`startServiceAsUser`！

`mBase` 是 `ContextImpl` 对象，继续看！

## 1.2 ContextImpl.unbindService
```java
class ContextImpl extends Context {

    @Override
    public void unbindService(ServiceConnection conn) {

        if (conn == null) {
            throw new IllegalArgumentException("connection is null");
        }

        if (mPackageInfo != null) {
            
            //【1】这里的 mPackageInfo 是 LoadedApk 类型的！
            // 获得 ServiceConnection 对应的 InnerConnection
            IServiceConnection sd = mPackageInfo.forgetServiceDispatcher(
                    getOuterContext(), conn);

            try {

                //【2】进入系统进程！
                ActivityManagerNative.getDefault().unbindService(sd);

            } catch (RemoteException e) {
                throw e.rethrowFromSystemServer();
            }

        } else {
            throw new RuntimeException("Not supported in system context");
        }
    }
}
```

`ContextImpl` 和 `ContextWrapper` 的具体关系，请来看另一博文：`Android` 系统的 `Context` 分析，这里我们不再详细说明！

`mUser`：表示的是当前的设备 `user`！

### 1.2.1 LoadedApk.forgetServiceDispatcher

参数传入：

 - `ServiceConnection c`：应用程序的链接对象！

```java
    public final IServiceConnection forgetServiceDispatcher(Context context,
            ServiceConnection c) {

        synchronized (mServices) {
            
            //【1】获得当前进程中 Context 运行环境对应的连接映射关系集合！
            ArrayMap<ServiceConnection, LoadedApk.ServiceDispatcher> map
                    = mServices.get(context);

            LoadedApk.ServiceDispatcher sd = null;

            if (map != null) {
            
                //【2】获得当前 ServiceConnection 对应的 ServiceDispatcher 服务分发对象！
                sd = map.get(c);

                if (sd != null) {
                    
                    // 移除对应 ServiceDispatcher 服务分发对象！
                    map.remove(c);
                    
                    //【2.1】设置 mForgotten 的值为 true！
                    sd.doForget();

                    if (map.size() == 0) {

                        // 当 Context 运行环境下已经没有连接映射，那就从 LoadkedApk.mServices 移除！
                        mServices.remove(context);
                    }
                    
                    // Context.BIND_DEBUG_UNBIND 标志是在 bindService 时设置的，用于 unbind 的 debug 调试！
                    // 这里我们不看！
                    if ((sd.getFlags()&Context.BIND_DEBUG_UNBIND) != 0) {

                        ArrayMap<ServiceConnection, LoadedApk.ServiceDispatcher> holder
                                = mUnboundServices.get(context);

                        if (holder == null) {
                            holder = new ArrayMap<ServiceConnection, LoadedApk.ServiceDispatcher>();
                            mUnboundServices.put(context, holder);
                        }

                        RuntimeException ex = new IllegalArgumentException(
                                "Originally unbound here:");

                        ex.fillInStackTrace();
                        sd.setUnbindLocation(ex);
                        holder.put(c, sd);
                    }
                    
                    // 返回 InnerConnection 对象！
                    return sd.getIServiceConnection();
                }
            }

            ArrayMap<ServiceConnection, LoadedApk.ServiceDispatcher> holder
                    = mUnboundServices.get(context);

            if (holder != null) {
                sd = holder.get(c);
                if (sd != null) {
                    RuntimeException ex = sd.getUnbindLocation();
                    throw new IllegalArgumentException(
                            "Unbinding Service " + c
                            + " that was already unbound", ex);
                }
            }
            if (context == null) {
                throw new IllegalStateException("Unbinding Service " + c
                        + " from Context that is no longer in use: " + context);
            } else {
                throw new IllegalArgumentException("Service not registered: " + c);
            }
        }
    }
```
这里逻辑很简单，其中，调用了 `ServiceDispatcher.doForget` 方法！

```java
        void doForget() {
            synchronized(this) {

                for (int i=0; i<mActiveConnections.size(); i++) {
                    ServiceDispatcher.ConnectionInfo ci = mActiveConnections.valueAt(i);
                    
                    //【1】取消死亡监控器
                    ci.binder.unlinkToDeath(ci.deathMonitor, 0);
                }
                
                // 清楚这个连接对应的 mActiveConnections 集合！
                mActiveConnections.clear();
                
                // 置 mForgotten 为 true！
                mForgotten = true;
            }
        }
```
我们继续看！！

接着，调用 `ActivityManagerNative.getDefault()` 方法，获得 `AMS` 的代理对象 `ActivityManagerProxy`！

`ActivityManagerProxy` 是 `ActivityManagerN` 的内部类，通过 `getDefault` 方法创建了对应的单例模式，保存在 `ActivityManagerNative` 的类变量 `getDefault` 中！

## 1.3 ActivityManagerP.unbindService

```java
    public boolean unbindService(IServiceConnection connection) throws RemoteException {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IActivityManager.descriptor);
        data.writeStrongBinder(connection.asBinder());

        //【1】传递 UNBIND_SERVICE_TRANSACTION，flags 为 0，是阻塞式通信！
        mRemote.transact(UNBIND_SERVICE_TRANSACTION, data, reply, 0);

        reply.readException();
        boolean res = reply.readInt() != 0;
        data.recycle();
        reply.recycle();
        return res;
    }
```
通过 `binder` 进程间通信，进入系统进程，参数分析：

- `IServiceConnection connection`：`InnerConnection` 对象！

# 2 系统进程

首先，进入 `ActivityManagerN` 中去看看：
```java
        case UNBIND_SERVICE_TRANSACTION: {
            data.enforceInterface(IActivityManager.descriptor);
            IBinder b = data.readStrongBinder();
            
            //【1】这里是转换为 IServiceConnection.proxy 对象！
            IServiceConnection conn = IServiceConnection.Stub.asInterface(b);

            boolean res = unbindService(conn);

            reply.writeNoException();
            reply.writeInt(res ? 1 : 0);

            return true;
        }
```

## 2.1 ActivityManagerP.unbindService

继续来看：
```java
    public boolean unbindService(IServiceConnection connection) {

        synchronized (this) {

            //【1】进入 ActiveServices 方法！
            return mServices.unbindServiceLocked(connection);
        }
    }
```

## 2.2 ActiveServices.unbindServiceLocked

解除服务 `bind`，参数传入如下：

- `IServiceConnection connection`：是 `IServiceConnection.Proxy` 类型的对象，是 `InnerConnection ` 在系统进程中的 `Binder`实体对象！
```java
    boolean unbindServiceLocked(IServiceConnection connection) {

        IBinder binder = connection.asBinder();

        if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "unbindService: conn=" + binder);

        //【1】从 mServiceConnections 中获得对应的 ConnectionRecord 集合！
        // 表示 IServiceConnection 对象对应的所有连接对象！
        ArrayList<ConnectionRecord> clist = mServiceConnections.get(binder);

        if (clist == null) {

            Slog.w(TAG, "Unbind failed: could not find connection for "
                  + connection.asBinder());

            return false;
        }

        final long origId = Binder.clearCallingIdentity();

        try {

            while (clist.size() > 0) {

                // 遍历 ConnectionRecord 列表！
                ConnectionRecord r = clist.get(0);
                
                //【2】移除连接，并拉起被绑定服务的 unbind 方法，这个我们后面看！
                removeConnectionLocked(r, null, null);

                if (clist.size() > 0 && clist.get(0) == r) {

                    // removeConnectionLocked 中会移除这个 ConnectionRecord 对象，这里是异常判断；
                    // 如果 removeConnectionLocked 移除失败，这里会再次移除！
                    Slog.wtf(TAG, "Connection " + r + " not removed for binder " + binder);
                    clist.remove(0);
                }
                
                // 如果绑定的服务的进程已经启动，进入该分支！
                if (r.binding.service.app != null) {
                    if (r.binding.service.app.whitelistManager) {
                        updateWhitelistManagerLocked(r.binding.service.app);
                    }

                    // This could have made the service less important.
                    if ((r.flags&Context.BIND_TREAT_LIKE_ACTIVITY) != 0) {

                        r.binding.service.app.treatLikeActivity = true;
                        mAm.updateLruProcessLocked(r.binding.service.app,
                                r.binding.service.app.hasClientActivities
                                || r.binding.service.app.treatLikeActivity, null);
                    }

                    // 更新服务的 adj 的值！
                    mAm.updateOomAdjLocked(r.binding.service.app);
                }
            }
        } finally {
            Binder.restoreCallingIdentity(origId);
        }

        return true;
    }
```
**`ActiveServices`** 的 **`mServiceConnections`** 用来封装所有服务的连接对象 `IServiceConnection.proxy` 和
`ConnectionRecord` 的映射关系，二者是一对多的关系！

继续来看！
## 2.3 ActiveServices.removeConnectionLocked

这个方法是 `unbindService` 整个流程中的关键！

- `ConnectionRecord c`：连接信息对象；
- `ProcessRecord skipApp`：传入为 `null`；
- `ActivityRecord skipAct`：传入为 `null`；

```java
    void removeConnectionLocked(ConnectionRecord c, ProcessRecord skipApp, ActivityRecord skipAct) {

        // 这里的 c.conn 是 IServiceConnection.Proxy 代理对象，映射着应用进程中的一个连接对象；
        IBinder binder = c.conn.asBinder();
        // 获得 ConnectionRecord 对象所属的进程绑定信息对象 AppBindRecord；
        AppBindRecord b = c.binding;
        // 获得所绑定的服务对象 ServiceRecord；
        ServiceRecord s = b.service;

        //【1】删除 bindService 时创建的数据结构和引用关系!
        // 从 ServiceRecord.connections 中获得 IServiceConnection.Proxy 对应的 ArrayList<ConnectionRecord> 集合！
        // 并中删除当前的 ConnectionRecord 对象！
        ArrayList<ConnectionRecord> clist = s.connections.get(binder);
        if (clist != null) {
            clist.remove(c);
            if (clist.size() == 0) {
                
                // 如果 ArrayList<ConnectionRecord> 长度为 0，就移除
                // IServiceConnection.Proxy 和 ArrayList<ConnectionRecord> 映射！
                s.connections.remove(binder);
            }
        }

        // 从 AppBindRecord.connections 中移除 ConnectionRecord！
        b.connections.remove(c);

        if (c.activity != null && c.activity != skipAct) {
            if (c.activity.connections != null) {
                c.activity.connections.remove(c);
            }
        }
 
        // AppBindRecord.client 是 ProcessRecord 对象，表示绑定者所在的进程，
        // 这里是从 ProcessRecord.connections 集合中删除当前的 ConnectionRecord！
        if (b.client != skipApp) {
            b.client.connections.remove(c);

            if ((c.flags&Context.BIND_ABOVE_CLIENT) != 0) {
                b.client.updateHasAboveClientLocked();
            }

            if ((c.flags&Context.BIND_ALLOW_WHITELIST_MANAGEMENT) != 0) {

                s.updateWhitelistManager();
                if (!s.whitelistManager && s.app != null) {
                    updateWhitelistManagerLocked(s.app);
                }
            }

            if (s.app != null) {
                updateServiceClientActivitiesLocked(s.app, c, true);
            }
        }

        // 从 ActiveServices.mServiceConnections 获得 IServiceConnection.Proxy 对应的 ArrayList<ConnectionRecord>！
        // 并从 ArrayList<ConnectionRecord> 中删除 ConnectionRecord！
        clist = mServiceConnections.get(binder);
        if (clist != null) {
            clist.remove(c);
            if (clist.size() == 0) {

                // 如果 ArrayList<ConnectionRecord> 长度为 0，就移除
                // IServiceConnection.Proxy 和 ArrayList<ConnectionRecord> 映射！
                mServiceConnections.remove(binder);
            }
        }

        mAm.stopAssociationLocked(b.client.uid, b.client.processName, s.appInfo.uid, s.name);

        // 如果 AppBindRecord.connections 的元素个数为 0，表示该进程中已经没有连接对象了！
        // 就删除 AppBindRecord.IntentBindRecord.apps 中调用者进程 ProcessRecord 和 AppBindRecord 的映射！
        if (b.connections.size() == 0) {
            b.intent.apps.remove(b.client);
        }

        // 此时，服务没有死亡，进入这个分支！！
        if (!c.serviceDead) {
            if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "Disconnecting binding " + b.intent
                    + ": shouldUnbind=" + b.intent.hasBound);
                    
            // 当服务所在进程没有被销毁，且已经没有任何进程通过该 intent 绑定该服务了，就进入这里！
            if (s.app != null && s.app.thread != null && b.intent.apps.size() == 0
                    && b.intent.hasBound) {

                try {
                
                    // 【2】设置拉起 onunbind 方法超时处理任务，这里不多说了，逻辑类似！
                    bumpServiceExecutingLocked(s, false, "unbind");

                    if (b.client != s.app && (c.flags&Context.BIND_WAIVE_PRIORITY) == 0
                            && s.app.setProcState <= ActivityManager.PROCESS_STATE_RECEIVER) {
                        mAm.updateLruProcessLocked(s.app, false, null);
                    }

                    mAm.updateOomAdjLocked(s.app);
                    
                    // 设置 IntentBindRecord.hasBound 为 false，表示绑定解除！
                    b.intent.hasBound = false;
                    
                    // 设置 IntentBindRecord.doRebind 为 false，后面会根据 onUnbind 的返回值来设置 doRebind！
                    b.intent.doRebind = false;
                    
                    // 【3】通过 binder 通信，进入被绑定服务所在的进程，拉起其 onUnbind 方法
                    // 采用的是 IBinder.FLAG_ONEWAY 的方式，非阻塞式！！
                    s.app.thread.scheduleUnbindService(s, b.intent.intent.getIntent());

                } catch (Exception e) {

                    Slog.w(TAG, "Exception when unbinding service " + s.shortName, e);
                    serviceProcessGoneLocked(s);
                }
            }

            // 如果接触绑定的时候，服务正在启动,那就将服务从 mPendingServices 中删除！
            mPendingServices.remove(s);
            
            //【4】如果服务之前 bind 时，flags 设置了 Context.BIND_AUTO_CREATE，就尝试停止服务！ 
            if ((c.flags&Context.BIND_AUTO_CREATE) != 0) {
                boolean hasAutoCreate = s.hasAutoCreateConnections();

                if (!hasAutoCreate) {
                    if (s.tracker != null) {
                        s.tracker.setBound(false, mAm.mProcessStats.getMemFactorLocked(),
                                SystemClock.uptimeMillis());
                    }
                }
                
                // 【4.1】尝试停止服务！
                bringDownServiceIfNeededLocked(s, true, hasAutoCreate);
            }
        }
    }

```
这段代码最开始是解除 `bindService` 时创建的数据结构和引用关系！然后拉起了被绑定服务的 `unbind` 方法，最后尝试停止服务！

注意：

如果之前对应的 `bindService` 使用了 `Context.BIND_AUTO_CREATE` 的 `flags`，这里就会尝试把服务停止，但是如果之前对应的 `bindService` 没有设置 `Context.BIND_AUTO_CREATE` 标志位，从前面的 `bindService` 流程中，可以看出，只会把之前 `bind` 时的绑定信息删除即可！


下面我们来一个一个分析！

### 2.3.1 ApplicationThreadP.scheduleUnbindService

首先，来看看拉起 onUnbind 方法的逻辑！
```java
    public final void scheduleUnbindService(IBinder token, Intent intent)
            throws RemoteException {
        Parcel data = Parcel.obtain();
        data.writeInterfaceToken(IApplicationThread.descriptor);

        data.writeStrongBinder(token);
        intent.writeToParcel(data, 0);

        //【1】binder 通信，采用的是 IBinder.FLAG_ONEWAY 的方式，非阻塞式！
        mRemote.transact(SCHEDULE_UNBIND_SERVICE_TRANSACTION, data, null,
                IBinder.FLAG_ONEWAY);

        data.recycle();
    }

```
这里通过 binder 通信，进入服务所在的应用进程中！我们在第三部分详细讨论！！

### 2.3.2 ActiveServices.bringDownServiceIfNeededLocked

拉起了服务的 `onUnbind` 方法后，对于 `bindService` 时，如果设置了 `Context.BIND_AUTO_CREATE` 的 `flags`，还要尝试停止服务！

我们来看一下具体的逻辑！

```java
    private final void bringDownServiceIfNeededLocked(ServiceRecord r, boolean knowConn,
            boolean hasConn) {
        
        //【1】如果这个服务还被需要，就不能停止！
        if (isServiceNeeded(r, knowConn, hasConn)) {
            return;
        }

        //【2】服务正在被启动，就不能停止！
        if (mPendingServices.contains(r)) {
            return;
        }
        
        //【3】继续停止服务！ 
        bringDownServiceLocked(r);
    }
```

如果这个服务仍然被需要，或者服务所在的进程正在启动，这两种情况下，不能停止服务！

#### 2.3.2.1 ActiveServices.isServiceNeeded
```java
    private final boolean isServiceNeeded(ServiceRecord r, boolean knowConn, boolean hasConn) {
        //【1】服务已经通过 startService 启动了，返回 true！
        if (r.startRequested) {
            return true;
        }

        //【2】仍然有应用通过 auto create 的方式绑定该服务，通过前面的分析，这里的自动创建的服务链接被移除了！！
        if (!knowConn) {
            hasConn = r.hasAutoCreateConnections();
        }

        if (hasConn) {
            return true;
        }

        return false;
    }
```
`hasAutoCreateConnections` 方法遍历 `ServiceRecord.connections` 集合：
```java
    // in ServiceRecord
    public boolean hasAutoCreateConnections() {
        for (int conni=connections.size()-1; conni>=0; conni--) {

            ArrayList<ConnectionRecord> cr = connections.valueAt(conni);
            for (int i=0; i<cr.size(); i++) {
                if ((cr.get(i).flags&Context.BIND_AUTO_CREATE) != 0) {
                    return true;
                }
            }
        }
        return false;
    }
```

判断一个服务是否仍被需要，有两种情况：

- 服务已经通过 `startService` 被请求启动了；
- 服务仍然被一些应用通过自动创建的方式绑定，即 `bindService` 时，`flags` 为 `Context.BIND_AUTO_CREATE`；

如果满足上面的任何一个条件，`isServiceNeeded` 返回的是 `true`，那么，就不会停止这个任务了！

通过我们上面的分析，当我们调用 `unbindService`后，会将之前 `bind` 时的创建的连接对象从集合中移除，所以`hasConn`为`false`，但是如果服务之前被`startService`启动了，并没有`stopService`，那么`unbindService`不会继续执行！


下面，我们继续来看：

#### 2.3.2.2 ActiveServices.bringDownServiceLocked

如果没有其他的应用通过 `BIND_AUTO_CREATE` 的方式来绑定这个服务，那么就要调用 `bringDownServiceLocked` 方法，来停止服务，这里我们假设没有其他的绑定信息！

```java
    private final void bringDownServiceLocked(ServiceRecord r) {

        // 首先，遍历 r.connections 集合，终止所有的 bind 连接！
        for (int conni=r.connections.size()-1; conni>=0; conni--) {

            ArrayList<ConnectionRecord> c = r.connections.valueAt(conni);
            for (int i=0; i<c.size(); i++) {
                ConnectionRecord cr = c.get(i);
                
                // 设置每一个 ConnectionRecord.serviceDead 为 true，表示服务要被 stop 掉，连接断开！
                // 因为这里设置了每一个 ConnectionRecord，所以前面只有第一次调用才会进入！
                cr.serviceDead = true;
                try {
                    
                    //【1】Binder 调用，进入应用进程触发 ServiceConnection 的 onServiceDisconnected，这是非阻塞的！
                    // 这里第二个参数，为 null，所以会触发 onServiceDisconnected！
                    cr.conn.connected(r.name, null);
                    
                } catch (Exception e) {
                    Slog.w(TAG, "Failure disconnecting service " + r.name +
                          " to connection " + c.get(i).conn.asBinder() +
                          " (in " + c.get(i).binding.client.processName + ")", e);
                }
            }
        }

        // 处理所有的 ServiceRecord.bindings 集合中的 IntentBindRecord 对象！
        // 因为绑定该服务的不止一个，现在要停止服务了，所以要解除所有的绑定！
        if (r.app != null && r.app.thread != null) {
            for (int i=r.bindings.size()-1; i>=0; i--) {

                IntentBindRecord ibr = r.bindings.valueAt(i);
                
                if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "Bringing down binding " + ibr
                        + ": hasBound=" + ibr.hasBound);
                
                // 显然是 hasBound 是 true 的！        
                if (ibr.hasBound) {
                    try {
                        
                        // 设置 unbind 超时处理
                        bumpServiceExecutingLocked(r, false, "bring down unbind");
                        mAm.updateOomAdjLocked(r.app);
                        
                        // 置 IntentBindRecord 对象的 hasBound 为 false，表示 bind 断开；
                        ibr.hasBound = false;
                        
                        //【2】通过 binder 通信，进入绑定了服务的应用进程，调用 AT.scheduleUnbindService 方法
                        // ，拉起服务的 onUnbind 方法！
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
            
            // 如果服务所在的进程没有被杀掉，进入这个分支！
            if (r.app.thread != null) {
                updateServiceForegroundLocked(r.app, false);
                try {
                     
                    // 设置 destroy 超时处理！                   
                    bumpServiceExecutingLocked(r, false, "destroy");
                    
                    // 将其添加到 AS 的 mDestroyingServices 机和中，表示服务正在销毁！
                    mDestroyingServices.add(r);
                    r.destroying = true;

                    mAm.updateOomAdjLocked(r.app);
                    
                    //【3】通过 binder 通信，回调 AT 的 scheduleStopService 方法！
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

- 首先进入绑定服务的应用进程，回调 `ServiceConnection` 的 `onServiceDisconnected` 方法，取消所有进程对该服务的绑定；
- 接着，进入服务所在的进程，拉起服务的 `onUnbind` 方法；
- 最后，拉起服务的 `onDestroy` 方法，销毁服务；

整个过程，还会清空和重置一些关键变量！！

下面我们重点分析一下上面的三个过程！


# 3 服务所在进程

首先要进入 ApplicationThreadN.onTransact 方法：
```java
        case SCHEDULE_UNBIND_SERVICE_TRANSACTION: {
            data.enforceInterface(IApplicationThread.descriptor);
            IBinder token = data.readStrongBinder();
            Intent intent = Intent.CREATOR.createFromParcel(data);
            
            //【1】执行 unbind 方法！
            scheduleUnbindService(token, intent);
            
            return true;
        }
```
继续来看：

进入 ApplicationThread 中！

## 3.1 ApplicationThread.scheduleUnbindService
```java
        public final void scheduleUnbindService(IBinder token, Intent intent) {
            BindServiceData s = new BindServiceData();
            s.token = token;
            s.intent = intent;
            
            //【1】发送 UNBIND_SERVICE 给主线程 Handler！
            sendMessage(H.UNBIND_SERVICE, s);
        }
```
进入 H！

### 3.1.1 ActivityThread.H
```java
    private class H extends Handler {
            public void handleMessage(Message msg) {
                case UNBIND_SERVICE:
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "serviceUnbind");
                    
                    //【2】继续调用 handleUnbindService！
                    handleUnbindService((BindServiceData)msg.obj);

                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                    break;
            
            }
    }
```
继续来看！！

### 3.1.2 ActivityThread.handleUnbindService

接着，这里是关键地方！：
```java
    private void handleUnbindService(BindServiceData data) {
    
        Service s = mServices.get(data.token);

        if (s != null) {
            try {
                data.intent.setExtrasClassLoader(s.getClassLoader());
                data.intent.prepareToEnterProcess();

                //【3】拉起服务的 onUnbind 方法，并获得返回值！
                boolean doRebind = s.onUnbind(data.intent);
                try {
                    
                    if (doRebind) {
                        // 如果 doRebind 值为 true，进入该分支！
                        ActivityManagerNative.getDefault().unbindFinished(
                                data.token, data.intent, doRebind);

                    } else {
                    
                        // 如果返回 false（默认），就会执行 serviceDoneExecuting 方法！
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
如果 `doRebind` 为 `true`，调用 `AMS` 的 `unbindFinished` 方法；
如果 `doRebind` 为 `false`，调用 `AMS` 的 `serviceDoneExecuting` 方法；

拉起了 `onUnbind` 方法后，需要根据返回值做相应的处理，接下来进入系统进程，具体逻辑见第四部分！！

## 3.2 InnerConnection.connected

系统进程会调用 `IServiceConnection.Proxy` 代理对象的 `connected` 方法，通过 `binder` 调用，进入绑定服务的应用进程，这个是异步调用：

```java
cr.conn.connected(r.name, null);
```
我们去看看！

```java
        private static class InnerConnection extends IServiceConnection.Stub {
            final WeakReference<LoadedApk.ServiceDispatcher> mDispatcher;

            InnerConnection(LoadedApk.ServiceDispatcher sd) {
                mDispatcher = new WeakReference<LoadedApk.ServiceDispatcher>(sd);
            }

            public void connected(ComponentName name, IBinder service) throws RemoteException {

                LoadedApk.ServiceDispatcher sd = mDispatcher.get();

                if (sd != null) {
                
                    //【1】调用 mDispatcher 对象的 connected，这里的 service 为 null；
                    sd.connected(name, service);
                }
            }
        }
```
继续来看：

### 3.2.1 ServiceDispatcher.connected
```java
        public void connected(ComponentName name, IBinder service) {
            if (mActivityThread != null) {
                mActivityThread.post(new RunConnection(name, service, 0));

            } else {
                doConnected(name, service);
            }
        }
```
最后都会调用 `doConnected` 方法：

### 3.2.2 ServiceDispatcher.doConnected

参数 `service` 这里为 `null`！
```java
        public void doConnected(ComponentName name, IBinder service) {
    
            ServiceDispatcher.ConnectionInfo old;
            ServiceDispatcher.ConnectionInfo info;

            synchronized (this) {
                
                // mForgotten 在最开始就被设置成了 true，所以这里就会直接返回的！
                //  ServiceConnection.onServiceDisconnected 方法不会被调用！
                if (mForgotten) {
                    // We unbound before receiving the connection; ignore
                    // any connection received.
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
这里，最后会进入应用的 `ServiceConnection` 对象！
```java
    private ServiceConnection mConnection = new ServiceConnection() {
        @Override
        public void onServiceDisconnected(ComponentName name) {
        
        }
    }
```
注意：`onServiceDisconnected` 方法是非阻塞的，即，系统进程不会等 `onServiceDisconnected` 执行完才继续执行！！

# 4 系统进程

首先会进入 `ActivityManagerNative` 的 `onTransact` 方法中：
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
接下来，进入 `AMS`！

## 4.1 ActivityManagerS.unbindFinished

如果 `onUnbind` 方法返回的是 `true`，进入该方法：
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
### 4.1.1 ActiveServices.unbindFinishedLocked

继续来看，`doRebind` 是服务的 `onUnbind` 方法的返回值！！
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

### 4.1.2 ActiveServices.serviceDoneExecutingLocked

参数传递：

- `boolean inDestroying`：传入 `false`；
- `boolean finishing`：传入 `false`；

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

## 4.2 ActivityManagerS.serviceDoneExecuting

对于拉起 `onDestroy` 方法，最后会调用：
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
接着，进入 `ActiveServices`！

### 4.2.1 ActiveServices.serviceDoneExecutingLocked

根据参数传递：

- **`int type`**：传入 `ActivityThread.SERVICE_DONE_EXECUTING_STOP`；
- **`int startId`**：传入 `0`；
- **`int res`**：传入` 0`；

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

### 4.2.2 ActiveServices.serviceDoneExecutingLocked

参数传递：

- `boolean inDestroying`：传入` true`；
- `boolean finishing`：传入 `true`；

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
                    
                    // 如果进程中没有在执行指定代码逻辑的服务了，就取消超时任务！
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
# 5 总结

这里我们来总结一下，接触绑定的流程！







