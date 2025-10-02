# Serivce 篇 6 - startForeground 和 stopForeground 分析
title: Serivce 篇 6 - startForeground 和 stopForeground 分析
date: 2016/08/11 20:46:25
catalog: true
categories:
- AndroidFramework源码分析
- Service服务
tags: Service服务
---

本文基于 Android 7.1.1 源码分析，转载请说明出处！

# 0 综述

我们可以通过 startForeground 方法来将一个服务设置成前台服务，具体的使用如下：
```java
    private void initNotification(Context context) {

        Notification.Builder builder = new Notification.Builder(context);
        builder.setOngoing(true).setSmallIcon(R.drawable.ic_launcher)
                .setContentTitle(context.getResources().getText(R.string.ticker_text))
                .setContentText(context.getResources().getText(R.string.ticker_text));
        builder.setPriority(Notification.PRIORITY_HIGH);
        mNotification = builder.build();
        mNotification.flags = mNotification.flags | Notification.FLAG_AUTO_CANCEL;
    }
    
    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        OppoLog.d(TAG, "onStartCommand");
        initNotification(this);
        
        startForeground(ID, mNotification);
        return super.onStartCommand(intent, flags, startId);
    }

    @Override
    public void onDestroy() {
        super.onDestroy();
        OppoLog.d(TAG, "onDestroy");
        
        stopForeground(true);
    }
```
其中，涉及到的方法有如下几个：
```java
// 将服务设置到前台，如果 id 为 0，那就取消设置前台服务；
startForeground(int id, Notification notification) 

// 取消前台设置，removeNotification 表示是否移除通知；
stopForeground(boolean removeNotification)
// 取消前台设置，flags 可选 STOP_FOREGROUND_REMOVE 或者 STOP_FOREGROUND_DETACH
stopForeground(int flags)
```
我们来具体分析下，startForeground 方法的处理流程！

# 1 服务所在进程
## 1.1 Service.startForeground

我们先来看看 startForeground 方法！
```java
    public final void startForeground(int id, Notification notification) {
        try {
            
            // 调用 AMS 的 setServiceForeground 方法！
            mActivityManager.setServiceForeground(
                    new ComponentName(this, mClassName), mToken, id,
                    notification, 0);

        } catch (RemoteException ex) {
        }
    }
```
这里的 mToken，在之前 Service 的创建时，系统进程赋给 Service 的，是一个 IBinder 对象，是 ServiceRecord 对象的引用！

这里的 mActivityManager 也是 Service 在创建后，内部保存的 AMS 的代理对象！

接着进入 ActivityManagerProxy 代理对象中：
```java
    public void setServiceForeground(ComponentName className, IBinder token,
            int id, Notification notification, int flags) throws RemoteException {

        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IActivityManager.descriptor);
        ComponentName.writeToParcel(className, data);
        data.writeStrongBinder(token);
        data.writeInt(id);
        if (notification != null) {
            data.writeInt(1);
            notification.writeToParcel(data, 0);
        } else {
            data.writeInt(0);
        }
        data.writeInt(flags);

        mRemote.transact(SET_SERVICE_FOREGROUND_TRANSACTION, data, reply, 0);

        reply.readException();
        data.recycle();
        reply.recycle();
    }
```
接着进入系统进程！

# 2 系统进程

## 2.1 ActivityManagerN.onTransact
```java
        case SET_SERVICE_FOREGROUND_TRANSACTION: {
            data.enforceInterface(IActivityManager.descriptor);
            ComponentName className = ComponentName.readFromParcel(data);
            IBinder token = data.readStrongBinder();
            int id = data.readInt();
            Notification notification = null;
            if (data.readInt() != 0) {
                notification = Notification.CREATOR.createFromParcel(data);
            }
            
            // 传入 0；
            int sflags = data.readInt();
            
            // 进入 AMS！
            setServiceForeground(className, token, id, notification, sflags);
            reply.writeNoException();
            return true;
        }
```
接着，进入 AMS:

## 2.2 ActivityManagerS.setServiceForeground
```java
    @Override
    public void setServiceForeground(ComponentName className, IBinder token,
            int id, Notification notification, int flags) {
        synchronized(this) {
        
            // 进入 AS!
            mServices.setServiceForegroundLocked(className, token, id, notification, flags);
        }
    }
```
接着，进入 ActiveServices:

## 2.3 ActiveServices.setServiceForegroundLocked

```java
    public void setServiceForegroundLocked(ComponentName className, IBinder token,
            int id, Notification notification, int flags) {
        final int userId = UserHandle.getCallingUserId();
        final long origId = Binder.clearCallingIdentity();
        try {
            
            //【1】根据传入的 className，token 和 userId，找到对应的 ServiceRecord 对象！
            ServiceRecord r = findServiceLocked(className, token, userId);

            if (r != null) {
                if (id != 0) {
                    if (notification == null) {

                        // 如果 notification 为 null，抛出异常！
                        throw new IllegalArgumentException("null notification");
                    }
                    
                    //【2】如果服务的 foregroundId 和传入的 id 不一样，就取消 foregroundId 对应的旧的 notification
                    // 将 foregroundId 设置为本次传入的新的 id
                    if (r.foregroundId != id) {
                        cancelForegroudNotificationLocked(r);
                        r.foregroundId = id;
                    }

                    notification.flags |= Notification.FLAG_FOREGROUND_SERVICE;
                    
                    // 设置服务 r.foregroundNoti 为创建的 notification；
                    // 设置服务 r.isForeground 为 true，表示该服务为前台服务；
                    r.foregroundNoti = notification;
                    r.isForeground = true;
                    
                    //【3】显示 notification！
                    r.postNotification();

                    //【4】这里是重点，更新服务的优先级和 oomAdj，设置服务为前台服务！！
                    if (r.app != null) {
                        updateServiceForegroundLocked(r.app, true);
                    }
                    
                    //【5】让服务不再后台执行，同时也不让服务延迟启动！
                    getServiceMap(r.userId).ensureNotStartingBackground(r);
                    mAm.notifyPackageUse(r.serviceInfo.packageName,
                                         PackageManager.NOTIFY_PACKAGE_USE_FOREGROUND_SERVICE);

                } else {
                    if (r.isForeground) {
                        r.isForeground = false;
                        if (r.app != null) {
                            mAm.updateLruProcessLocked(r.app, false, null);
                            updateServiceForegroundLocked(r.app, true);
                        }
                    }

                    if ((flags & Service.STOP_FOREGROUND_REMOVE) != 0) {
                        cancelForegroudNotificationLocked(r);
                        r.foregroundId = 0;
                        r.foregroundNoti = null;

                    } else if (r.appInfo.targetSdkVersion >= Build.VERSION_CODES.LOLLIPOP) {
                        r.stripForegroundServiceFlagFromNotification();
                        if ((flags & Service.STOP_FOREGROUND_DETACH) != 0) {
                            r.foregroundId = 0;
                            r.foregroundNoti = null;
                        }
                    }
                }
            }
        } finally {
            Binder.restoreCallingIdentity(origId);
        }
    }
```

### 2.3.1 ActiveServices.findServiceLocked

找到对应的 ServiceRecord 对象！
```java
    private final ServiceRecord findServiceLocked(ComponentName name,
            IBinder token, int userId) {
        ServiceRecord r = getServiceByName(name, userId);
        return r == token ? r : null;
    }
```
根据组件名和设别用户id，在 ActiveServices 的 mServiceMap 稀疏数组中，找到对应的 ServiceRecord 对象，然后和 Service 进程传递过来的 token，二者必须相等才行！

这里就不多说了！

### 2.3.2 ActiveServices.cancelForegroudNotificationLocked
```java
    private void cancelForegroudNotificationLocked(ServiceRecord r) {
        if (r.foregroundId != 0) {

            ServiceMap sm = getServiceMap(r.userId);

            if (sm != null) {
                for (int i = sm.mServicesByName.size()-1; i >= 0; i--) {
                    ServiceRecord other = sm.mServicesByName.valueAt(i);
                    if (other != r && other.foregroundId == r.foregroundId
                            && other.packageName.equals(r.packageName)) {

                        // Found one!  Abort the cancel.
                        return;
                    }
                }
            }
            r.cancelNotification();
        }
    } 
```
ServiceRecord r 是本次需要设置为前台的服务，r.foregroundId 为前台 notification 的 id，这个函数的主要作用是，判断本次需要设置为前台的服务所属的应用是否有其他服务的 foregroundId 和 r.foregroundId 相同，如果相同，那就不能取消这个通知！

如果只有当前服务在使用这个通知的 id，那就取消这个旧的通知：

#### 2.3.2.1 ServiceRecord.cancelNotification
```java
    public void cancelNotification() {

        // Do asynchronous communication with notification manager to
        // avoid deadlocks.
        final String localPackageName = packageName;
        final int localForegroundId = foregroundId;
        ams.mHandler.post(new Runnable() {
            public void run() {
                INotificationManager inm = NotificationManager.getService();
                if (inm == null) {
                    return;
                }
                try {
                    // 调用 NotificationManager 取消这个通知！
                    inm.cancelNotificationWithTag(localPackageName, null,
                            localForegroundId, userId);
                } catch (RuntimeException e) {
                    Slog.w(TAG, "Error canceling notification for service", e);
                } catch (RemoteException e) {
                }
            }
        });
    }
```
调用 NotificationManager 取消这个通知！

### 2.3.3 ServiceRecord.postNotification
```java
    public void postNotification() {
        final int appUid = appInfo.uid;
        final int appPid = app.pid;

        if (foregroundId != 0 && foregroundNoti != null) {

            // Do asynchronous communication with notification manager to
            // avoid deadlocks.
            final String localPackageName = packageName;
            final int localForegroundId = foregroundId;
            final Notification _foregroundNoti = foregroundNoti;
            
            // 通过 AMS.MainHandler，在系统进程主线程执行任务！
            ams.mHandler.post(new Runnable() {
                public void run() {
                    NotificationManagerInternal nm = LocalServices.getService(
                            NotificationManagerInternal.class);
                    if (nm == null) {
                        return;
                    }

                    Notification localForegroundNoti = _foregroundNoti;

                    try {
                        if (localForegroundNoti.getSmallIcon() == null) {
                            Slog.v(TAG, "Attempted to start a foreground service ("
                                    + name
                                    + ") with a broken notification (no icon: "
                                    + localForegroundNoti
                                    + ")");

                            CharSequence appName = appInfo.loadLabel(
                                    ams.mContext.getPackageManager());
                            if (appName == null) {
                                appName = appInfo.packageName;
                            }
                            Context ctx = null;
                            try {
                                //【1】创建一个上下文运行环境 Context，用于创建通知！
                                ctx = ams.mContext.createPackageContextAsUser(
                                        appInfo.packageName, 0, new UserHandle(userId));
                                
                                //【2】创建通知建造者，并设置通知相关属性！
                                Notification.Builder notiBuilder = new Notification.Builder(ctx);

                                // it's ugly, but it clearly identifies the app
                                notiBuilder.setSmallIcon(appInfo.icon);

                                // 设置 notification 的标志位 Notification.FLAG_FOREGROUND_SERVICE！
                                // 表示该通知代表一个在前台运行的服务！
                                notiBuilder.setFlag(Notification.FLAG_FOREGROUND_SERVICE, true);

                                // 设置通知的优先级为 Notification.PRIORITY_MIN！
                                notiBuilder.setPriority(Notification.PRIORITY_MIN);

                                Intent runningIntent = new Intent(
                                        Settings.ACTION_APPLICATION_DETAILS_SETTINGS);
                                runningIntent.setData(Uri.fromParts("package",
                                        appInfo.packageName, null));
                                PendingIntent pi = PendingIntent.getActivity(ams.mContext, 0,
                                        runningIntent, PendingIntent.FLAG_UPDATE_CURRENT);
                                notiBuilder.setColor(ams.mContext.getColor(
                                        com.android.internal
                                                .R.color.system_notification_accent_color));
                                notiBuilder.setContentTitle(
                                        ams.mContext.getString(
                                                com.android.internal.R.string
                                                        .app_running_notification_title,
                                                appName));
                                notiBuilder.setContentText(
                                        ams.mContext.getString(
                                                com.android.internal.R.string
                                                        .app_running_notification_text,
                                                appName));
                                notiBuilder.setContentIntent(pi);

                                localForegroundNoti = notiBuilder.build();
                            } catch (PackageManager.NameNotFoundException e) {
                            }
                        }
                        
                        //【3】校验 notification 的 SmallIcon 是否仍然为 null，如果仍然为 null
                        // 抛出运行时异常！
                        if (localForegroundNoti.getSmallIcon() == null) {
                            throw new RuntimeException("invalid service notification: "
                                    + foregroundNoti);
                        }
                        int[] outId = new int[1];
                        
                        //【4】显示 notification！
                        nm.enqueueNotification(localPackageName, localPackageName,
                                appUid, appPid, null, localForegroundId, localForegroundNoti,
                                outId, userId);

                        foregroundNoti = localForegroundNoti; // save it for amending next time

                    } catch (RuntimeException e) {
                        Slog.w(TAG, "Error showing notification for service", e);
                        
                        // 如果出现异常，就不会设置服务为前台服务，同时抛出 Crash 异常！
                        ams.setServiceForeground(name, ServiceRecord.this,
                                0, null, 0);
                        ams.crashApplication(appUid, appPid, localPackageName,
                                "Bad notification for startForeground: " + e);
                    }
                }
            });
        }
    }

```
这个方法的主要作用是显示 notification！


### 2.3.4 ActiveServices.updateServiceForegroundLocked

这里是重点，更新服务的优先级和 oomAdj，设置服务为前台服务，参数分析：

- ProcessRecord proc：服务所在的进程！
- boolean oomAdj：传入 true，表示需要更新 oomAdj 值！

```java
    private void updateServiceForegroundLocked(ProcessRecord proc, boolean oomAdj) {
        boolean anyForeground = false;

        // 遍历进程，如果该进程中有服务 sr.isForeground 为 true 了，
        // 表示该进程存在前台服务，所以 anyForeground 为 true，跳出循环！
        for (int i=proc.services.size()-1; i>=0; i--) {
            ServiceRecord sr = proc.services.valueAt(i);
            if (sr.isForeground) {
                anyForeground = true;
                break;
            }
        }
        
        // 进入这里进行进程的优先级和 oomAdj 的设置！
        mAm.updateProcessForegroundLocked(proc, anyForeground, oomAdj);
    }
```
显然，根据前面的属性设置 anyForeground 为 true ！

下面就是更新服务所在进程的优先级和 oomAdj 的值了！


### 2.3.5 ServiceMap.ensureNotStartingBackground

```java
        void ensureNotStartingBackground(ServiceRecord r) {

            //【1】从 AS.mStartingBackground 中尝试删除该服务，不让服务在后台运行！
            if (mStartingBackground.remove(r)) {
                if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE,
                        "No longer background starting: " + r);
                
                rescheduleDelayedStarts();
            }
            
            //【2】从 AS.mDelayedStartList 中尝试删除该服务，不让服务延迟启动！ 
            if (mDelayedStartList.remove(r)) {
                if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE, "No longer delaying start: " + r);
            }
        }
```
这里和 startService 中的一样的，目的是将当前服务从 mStartingBackground 和 mDelayedStartList 中删除，因为服务被设置成了前台服务，如果当前服务从 mStartingBackground 中删除成功了，就要调用 rescheduleDelayedStarts 方法，继续发送 MSG_BG_START_TIMEOUT 消息！

这里具体的分析，**请去看 startService 博文的第 2.3 节**！

通过上面的分析，关键的地方是这里：
```java
    private void updateServiceForegroundLocked(ProcessRecord proc, boolean oomAdj) {
        
        ... ... ... ...
        
        // 进入这里进行进程的优先级和 oomAdj 的设置！
        mAm.updateProcessForegroundLocked(proc, anyForeground, oomAdj);
    }
```
这里调用了 AMS 的 updateProcessForegroundLocked 方法来更新 Service 的进程优先级和 oomAdj 的值。我们继续来看！

## 2.4 ActivityManagerS.updateProcessForegroundLocked
参数传递：

- ProcessRecord proc：服务所在的进程；
- boolean isForeground：传入 true；
- boolean oomAdj：传入 true；

这里我们假设之前服务所在进程没有运行任何的前台服务！！
```java
    final void updateProcessForegroundLocked(ProcessRecord proc, boolean isForeground,
            boolean oomAdj) {
            
        // proc.foregroundServices 表示进程是否运行前台服务！
        // 如果 isForeground 不等于 proc.foregroundServices，说明进程的
        if (isForeground != proc.foregroundServices) {
            
            // 将 proc.foregroundServices 设置为 isForeground 的值！
            proc.foregroundServices = isForeground;
            
            // mForegroundPackages 集合用来保存所有当前正在运行着前台服务的应用程序包信息
            // curProcs 则是运行着前台服务的进程对象 ProcessRecord!
            ArrayList<ProcessRecord> curProcs = mForegroundPackages.get(proc.info.packageName,
                    proc.info.uid);
            
            // 因为我们现在是设置服务为前台服务，所以 isForeground 是 true！
            if (isForeground) {
                
                // 将进程所属的应用程序包，添加到 mForegroundPackages 集合中；
                // 并将服务所在的进程也添加到 mForegroundPackages 中；
                if (curProcs == null) {
                    curProcs = new ArrayList<ProcessRecord>();
                    mForegroundPackages.put(proc.info.packageName, proc.info.uid, curProcs);
                }
                if (!curProcs.contains(proc)) {
                    curProcs.add(proc);
                    
                    // 通知 BatteryStatsService 服务，有应用在前台执行服务！
                    mBatteryStatsService.noteEvent(BatteryStats.HistoryItem.EVENT_FOREGROUND_START,
                            proc.info.packageName, proc.info.uid);
                }

            } else {
                if (curProcs != null) {
                    if (curProcs.remove(proc)) {
                        mBatteryStatsService.noteEvent(
                                BatteryStats.HistoryItem.EVENT_FOREGROUND_FINISH,
                                proc.info.packageName, proc.info.uid);
                        if (curProcs.size() <= 0) {
                            mForegroundPackages.remove(proc.info.packageName, proc.info.uid);
                        }
                    }
                }

            }
            
            // 这里为 true，更新进程的 oomAdj 的值！
            if (oomAdj) {
                updateOomAdjLocked();
            }
        }
    }
```
这里有一些数据结构，我们来简单是说一下：

`ProcessRecord.foregroundServices` 表示该进程中是否在运行前台服务；

初次之外，还有一个 mForegroundPackages，用来保存运行着前台服务的应用程序包信息：
```java
    final ProcessMap<ArrayList<ProcessRecord>> mForegroundPackages
            = new ProcessMap<ArrayList<ProcessRecord>>();
```
他是 ProcessMap 类的对象，ProcessMap 是一个模板类：
```java
public class ProcessMap<E> {
    final ArrayMap<String, SparseArray<E>> mMap
            = new ArrayMap<String, SparseArray<E>>();
    
    public E get(String name, int uid) {
        SparseArray<E> uids = mMap.get(name);
        if (uids == null) return null;
        return uids.get(uid);
    }
    
    public E put(String name, int uid, E value) {
        SparseArray<E> uids = mMap.get(name);
        if (uids == null) {
            uids = new SparseArray<E>(2);
            mMap.put(name, uids);
        }
        uids.put(uid, value);
        return value;
    }
    ... ... ...
}
```
ProcessMap 的内部变量 mMap 是一个 ArrayMap 类型的集合，其中 key 是应用程序的报名，而 value 是 SparseArray<ProcessRecord> 类型的稀疏数组，其中数组下标是应用在指定设备用户下的 uid，而下标对应的值是 uid 对应的运行着前台服务的进程对象 ProcessRecord，这里就不多说了！！

这里我们就先看到这里，关于 updateOomAdjLocked 的逻辑处理，请去看。。。。

# 3 stopForeground 分析

上面分析了一些 startForeground 方法的主要流程，下面分析下 stopForeground 方法，stopForeground 方法一共有两个：
```java
    public final void stopForeground(boolean removeNotification) {
        stopForeground(removeNotification ? STOP_FOREGROUND_REMOVE : 0);
    }

    public final void stopForeground(int flags) {
        try {
            mActivityManager.setServiceForeground(
                    new ComponentName(this, mClassName), mToken, 0, null, flags);
        } catch (RemoteException ex) {
        }
    }
```
第一个方法最后还是会调用第二个方法！
这里我们先来解释两个参数变量：
```java
    public static final int STOP_FOREGROUND_REMOVE = 1<<0;
```
如果设置了这个标志位，取消前台服务的同时，还会移除对应的通知，但是如果同一个应用中有其他前台服务关联着相同的通知，就不会移除该通知；

如果不设置这个标志位，通知只能通过 startForeground(int, Notification) 或者 stopForeground(int)，或者服务被销毁的方式移除！

```java
   public static final int STOP_FOREGROUND_DETACH = 1<<1;
```
如果设置了这个标志位，取消前台服务的同时，不会移除对应的通知，但是会将通知和服务完全解除绑定，这时通知只有通过 NotificationManager 才能被取消！该标志位不能和 STOP_FOREGROUND_REMOVE 混合使用哦！

方法调用和 startForeground 方法很类似，最后会进入 ActiveSerivces 中去：

## 3.1 ActiveSerivces.setServiceForegroundLocked

参数传递：

- ComponentName className：服务的类名；
- IBinder token；服务的 ServiceRecord 对象；
- int id：传入 0；
- Notification notification：传入 null；
- int flags：传入具体的 flags；

```java
    public void setServiceForegroundLocked(ComponentName className, IBinder token,
            int id, Notification notification, int flags) {
        final int userId = UserHandle.getCallingUserId();
        final long origId = Binder.clearCallingIdentity();
        try {
            
            //【1】根据传入的 className，token 和 userId，找到对应的 ServiceRecord 对象！
            ServiceRecord r = findServiceLocked(className, token, userId);

            if (r != null) {
                if (id != 0) {
                   
                   ... ... ... ...// 这里不进入该分支；

                } else {

                    // 如果服务 r.isForeground 为 true，就设置 r.isForeground 为 false；
                    // 同时调用 AMS.updateLruProcessLocked 和 updateServiceForegroundLocked 
                    // 更新进程状态！
                    if (r.isForeground) {
                        r.isForeground = false;
                        if (r.app != null) {
                            mAm.updateLruProcessLocked(r.app, false, null);
                            updateServiceForegroundLocked(r.app, true);
                        }
                    }

                    // 如果 flags 设置了 STOP_FOREGROUND_REMOVE 标志位，就移除通知！！
                    if ((flags & Service.STOP_FOREGROUND_REMOVE) != 0) {

                        // 尝试移除该通知，如果当前服务所属的应用程序有其他前台服务
                        // 也关联着同一个通知，那就不移除通知！
                        cancelForegroudNotificationLocked(r);

                        r.foregroundId = 0;
                        r.foregroundNoti = null;

                    } else if (r.appInfo.targetSdkVersion >= Build.VERSION_CODES.LOLLIPOP) {

                        // 将服务和通知解除绑定！！
                        r.stripForegroundServiceFlagFromNotification();
                        
                        // 如果 flags 设置了 STOP_FOREGROUND_DETACH 标志位，
                        // 就清除服务的 r.foregroundId 和 r.foregroundNoti；
                        if ((flags & Service.STOP_FOREGROUND_DETACH) != 0) {
                            r.foregroundId = 0;
                            r.foregroundNoti = null;
                        }
                    }
                }
            }
        } finally {
            Binder.restoreCallingIdentity(origId);
        }
    }
```
这里就先分析这么多，其他内容后续在补充！