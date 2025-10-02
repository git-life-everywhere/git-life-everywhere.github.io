# BroadcastReceiver篇 2 - registerReceiver 动态注册流程分析
title: BroadcastReceiver篇 2 - registerReceiver 动态注册流程分析
date: 2016/05/01 20:46:25
catalog: true
categories:
- AndroidFramework源码分析
- BroadcastReceiver广播接收者
tags: BroadcastReceiver广播接收者
---

[toc]


基于 Android 7.1.1 源码，分析 BroadcastReceiver **动态注册** 的过程，转载请说明出处！

# 0 综述

BroadcastReceiver 动态注册，就是应用程序在运行过程中，调用 registerReceiver 方法注册：
```java
    @Override
    public Intent registerReceiver(
        BroadcastReceiver receiver, IntentFilter filter)；
    
    @Override
    public Intent registerReceiver(
        BroadcastReceiver receiver, IntentFilter filter,
        String broadcastPermission, Handler scheduler)；

    @Override
    public Intent registerReceiverAsUser(
        BroadcastReceiver receiver, UserHandle user, IntentFilter filter,
        String broadcastPermission, Handler scheduler)；
```
可以看到，动态注册广播接收者者时，我们需要创建一个 IntentFilter 过滤器对象，我们还可以指定广播发送者的权限，除此之外，我们还可以通过传入 Handler 来指定 onReceive 方法执行的线程（默认是主线程）。

当应用不再需要这个广播接收者后，需要动态取消注册：

```java
   public void unregisterReceiver(BroadcastReceiver receiver);
```

下面，我们来进入源码，分析动态注册的流程！

# 1 应用进程

## 1.1 ContextWapper.registerReceiver
```java
    @Override
    public Intent registerReceiver(
        BroadcastReceiver receiver, IntentFilter filter) {

        return mBase.registerReceiver(receiver, filter);
    }
    
    // broadcastPermission 用来指定发送者需要的权限！
    // scheduler 用来指定 onReceive 方法执行的线程！
    @Override
    public Intent registerReceiver(
        BroadcastReceiver receiver, IntentFilter filter,
        String broadcastPermission, Handler scheduler) {

        return mBase.registerReceiver(receiver, filter, broadcastPermission,
                scheduler);
    }

    /** @hide */
    @Override
    public Intent registerReceiverAsUser(
        BroadcastReceiver receiver, UserHandle user, IntentFilter filter,
        String broadcastPermission, Handler scheduler) {

        return mBase.registerReceiverAsUser(receiver, user, filter, broadcastPermission,
                scheduler);
    }
```
这里的 mBase 是 ContextImpl 对象！

## 1.2 ContextImpl.registerReceiver

进入 ContextImpl 继续看：
```java
    @Override
    public Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter) {
        return registerReceiver(receiver, filter, null, null);
    }

    @Override
    public Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter,
            String broadcastPermission, Handler scheduler) {

        return registerReceiverInternal(receiver, getUserId(),
                filter, broadcastPermission, scheduler, getOuterContext());
    }

    @Override
    public Intent registerReceiverAsUser(BroadcastReceiver receiver, UserHandle user,
            IntentFilter filter, String broadcastPermission, Handler scheduler) {
            
        return registerReceiverInternal(receiver, user.getIdentifier(),
                filter, broadcastPermission, scheduler, getOuterContext());
    }
```
我们可以看到，虽然是不同的注册方法，最后都调用了 registerReceiverInternal 方法！getOuterContext() 方法用于获得注册时的上下文运行环境！也就是代表注册时所属的组件！！

## 1.3 ContextImpl.registerReceiverInternal - 核心入口
```java
    private Intent registerReceiverInternal(BroadcastReceiver receiver, int userId,
            IntentFilter filter, String broadcastPermission,
            Handler scheduler, Context context) {

        IIntentReceiver rd = null;
        if (receiver != null) {
            //【1】mPackageInfo 是 LoadedApk 对象，每个进程都有一个！
            if (mPackageInfo != null && context != null) {
                if (scheduler == null) {
                    // 如果传入的 Handler 为 null，就设置 scheduler 为所在进程的主线程 Handler!
                    scheduler = mMainThread.getHandler();
                }
                
                //【*1.3.1】获得 BroadcastReceiver 对应的 IIntentReceiver 对象！
                rd = mPackageInfo.getReceiverDispatcher(
                    receiver, context, scheduler,
                    mMainThread.getInstrumentation(), true);

            } else {
                if (scheduler == null) {
                    scheduler = mMainThread.getHandler();
                }
                //【*1.3.2】创建一个 ReceiverDispatcher 对象！
                rd = new LoadedApk.ReceiverDispatcher(
                        receiver, context, scheduler, null, true).getIIntentReceiver();
            }
        }
        try {
            
            //【*1.4】进入系统进程，继续注册！
            final Intent intent = ActivityManagerNative.getDefault().registerReceiver(
                    mMainThread.getApplicationThread(), mBasePackageName,
                    rd, filter, broadcastPermission, userId);

            if (intent != null) {
                intent.setExtrasClassLoader(getClassLoader());
                intent.prepareToEnterProcess();
            }
            return intent;
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
```
IIntentReceiver 类实例 rd 用来在应用进程和系统进程中进行 Binder 通信！

Handler 类对象 scheduler 用来设置接受到广播后 onReceive 执行的线程，如果传入 null，默认就在主线程！

这里涉及到一个 mPackageInfo，他是 LoadedApk 类的实例。每一个进程在创建时，都会有一个 LoadedApk 对象。用来分装一个应用程序的信息！

我们去 LoadedApk 目录去看看：

### 1.3.1 LoadedApk.getReceiverDispatcher

这里的 boolean registered 为 true：
```java
    public getReceiverDispatcher(BroadcastReceiver r,
            Context context, Handler handler,
            Instrumentation instrumentation, boolean registered) {

        synchronized (mReceivers) {
            //【1】尝试获得对应的 ReceiverDispatcher，如果没有就重新创建！
            LoadedApk.ReceiverDispatcher rd = null;

            ArrayMap<BroadcastReceiver, LoadedApk.ReceiverDispatcher> map = null;
            if (registered) {
                map = mReceivers.get(context);
                if (map != null) {
                    rd = map.get(r);
                }
            }
            
            //【*1.3.2】第一次注册，创建全新的 ReceiverDispatcher 对象！
            if (rd == null) {
                rd = new ReceiverDispatcher(r, context, handler,
                        instrumentation, registered);

                if (registered) {
                    if (map == null) {
                        map = new ArrayMap<BroadcastReceiver, LoadedApk.ReceiverDispatcher>();
                        mReceivers.put(context, map);
                    }
                    map.put(r, rd);
                }
            } else {
                //【2】校验一下 context 和 handler 是否和之前注册的一样，不一样会抛出异常！
                rd.validate(context, handler);
            }

            rd.mForgotten = false;
            
            //【3】返回内部的 InnerReceiver 对象！
            return rd.getIIntentReceiver();
        }
    }
```

LoadedApk 有一个 ArrayMap 集合 mReceivers：
```java
    private final ArrayMap<Context, ArrayMap<BroadcastReceiver, ReceiverDispatcher>> mReceivers
        = new ArrayMap<Context, ArrayMap<BroadcastReceiver, LoadedApk.ReceiverDispatcher>>();
```
用来保存当前进程下，每个上下文运行环境 Context 中，对应的 BroadcastReceiver 和 ReceiverDispatcher 的映射关系！

### 1.3.2 new LoadedApk.ReceiverDispatcher

我们再去看看 ReceiverDispatcher 对象：

```java
        ReceiverDispatcher(BroadcastReceiver receiver, Context context,
                Handler activityThread, Instrumentation instrumentation,
                boolean registered) {

            if (activityThread == null) {
                throw new NullPointerException("Handler must not be null");
            }

            //【*1.3.2.1】创建了一个 InnerReceiver 对象，用于 Binder 通信！
            mIIntentReceiver = new InnerReceiver(this, !registered);
            mReceiver = receiver;
            mContext = context;

            mActivityThread = activityThread;
            mInstrumentation = instrumentation;
            mRegistered = registered;
            mLocation = new IntentReceiverLeaked(null);
            mLocation.fillInStackTrace();
        }
```
这里的 IntentReceiverLeaked，我们不过多分析，很简单！
```java
final class IntentReceiverLeaked extends AndroidRuntimeException {
    public IntentReceiverLeaked(String msg) {
        super(msg);
    }
}
```

ReceiverDispatcher 用于保存 BroadcastReceiver 和 InnerReceiver 映射关系，我们继续看：

#### 1.3.2.1 new ReceiverDispatcher.InnerReceiver
```java
        final static class InnerReceiver extends IIntentReceiver.Stub {
            final WeakReference<LoadedApk.ReceiverDispatcher> mDispatcher;
            final LoadedApk.ReceiverDispatcher mStrongRef;

            InnerReceiver(LoadedApk.ReceiverDispatcher rd, boolean strong) {
                //【1】创建了一个 ReceiverDispatcher 的弱引用！
                mDispatcher = new WeakReference<LoadedApk.ReceiverDispatcher>(rd);
                //【2】这里 strong 为 false，所以 mStrongRef 为 null；
                mStrongRef = strong ? rd : null;
            }

            @Override
            public void performReceive(Intent intent, int resultCode, String data,
                    Bundle extras, boolean ordered, boolean sticky, int sendingUser) {
                ... ... ... ... // 用于处理广播的接收！
            }
        }
```
我们可以看到 InnerReceiver 继承了 IIntentReceiver.Stub，他是作为 Binder 通信的服务端，其实，这种架构，我们在 Service 的创建和启动中，也同样看到过！！ 

## 1.4 ActivityManagerP.registerReceiver

接着调用 AMP 代理对象的 registerReceiver 方法，进入系统进程，注册 BroadcastReceiver！
```java
    public Intent registerReceiver(IApplicationThread caller, String packageName,
            IIntentReceiver receiver,
            IntentFilter filter, String perm, int userId) throws RemoteException {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IActivityManager.descriptor);

        //【1】处理 ApplicationThread 对象！
        data.writeStrongBinder(caller != null ? caller.asBinder() : null);
        data.writeString(packageName);

        //【2】处理 IIntentReceiver 对象！
        data.writeStrongBinder(receiver != null ? receiver.asBinder() : null);
        filter.writeToParcel(data, 0);
        data.writeString(perm);
        data.writeInt(userId);

        mRemote.transact(REGISTER_RECEIVER_TRANSACTION, data, reply, 0);

        reply.readException();
        Intent intent = null;
        int haveIntent = reply.readInt();
        if (haveIntent != 0) {
            intent = Intent.CREATOR.createFromParcel(reply);
        }
        reply.recycle();
        data.recycle();
        return intent;
    }
```

下面，进入系统进程！

# 2 系统进程

首先，进入 ActivityManagerN.onTransact 方法：
```java
    @Override
    public boolean onTransact(int code, Parcel data, Parcel reply, int flags)
            throws RemoteException {
        ... ... ...
        case REGISTER_RECEIVER_TRANSACTION:
        {
            data.enforceInterface(IActivityManager.descriptor);
            
            //【1】获得 ApplicationThreadProxy 对象！
            IBinder b = data.readStrongBinder();
            IApplicationThread app =
                b != null ? ApplicationThreadNative.asInterface(b) : null;
                
            String packageName = data.readString();
            b = data.readStrongBinder();
            
            //【2】获得 IIntentReceiver.proxy 对象！
            IIntentReceiver rec
                = b != null ? IIntentReceiver.Stub.asInterface(b) : null;
                
            //【3】获得 IntentFilter 对象！
            IntentFilter filter = IntentFilter.CREATOR.createFromParcel(data);
            String perm = data.readString();
            int userId = data.readInt();
            
            //【*2.1】继续调用 AMS.registerReceiver 方法，注册广播接收者！
            Intent intent = registerReceiver(app, packageName, rec, filter, perm, userId);

            reply.writeNoException();
            if (intent != null) {
                reply.writeInt(1);
                intent.writeToParcel(reply, 0);
            } else {
                reply.writeInt(0);
            }
            return true;
        }    
        ... ... ...    
    }
```

## 2.1 ActivityManagerS.registerReceiver

参数分析：

- **IApplicationThread caller**：注册操作所属的应用进程的 ApplicationThreadProxy 对象
- **String callerPackage**：注册者所属的应用程序包名；
- **IIntentReceiver receiver**：IIntentReceiver.Proxy 对象，映射注册者进程中的 InnerReceiver 对象；
- **IntentFilter filter**：Intent 过滤器；
- **String permission**：需要的权限！
- **int userId**：当前设备用户 id

```java
    public Intent registerReceiver(IApplicationThread caller, String callerPackage,
            IIntentReceiver receiver, IntentFilter filter, String permission, int userId) {
        enforceNotIsolatedCaller("registerReceiver");
        //【1】用来保存能够处理的粘性广播列表！
        ArrayList<Intent> stickyIntents = null;
        ProcessRecord callerApp = null;
        int callingUid;
        int callingPid;
        synchronized(this) {

            if (caller != null) {
                //【*2.1.1】获得注册者进程对应的 ProcessRecord 对象！
                // 如果注册者进程为 null，抛出异常！
                callerApp = getRecordForAppLocked(caller);
                if (callerApp == null) {
                    throw new SecurityException(
                            "Unable to find app for caller " + caller
                            + " (pid=" + Binder.getCallingPid()
                            + ") when registering receiver " + receiver);
                }
                
                //【2】如果注册者进程的 uid 不是系统 uid，并且注册者包名不在注册者进程的列表中
                // 且注册者包名不是 “android” 抛出异常！
                if (callerApp.info.uid != Process.SYSTEM_UID &&
                        !callerApp.pkgList.containsKey(callerPackage) &&
                        !"android".equals(callerPackage)) {
                    throw new SecurityException("Given caller package " + callerPackage
                            + " is not running in process " + callerApp);
                }
                callingUid = callerApp.info.uid;
                callingPid = callerApp.pid;

            } else {
                callerPackage = null;
                callingUid = Binder.getCallingUid();
                callingPid = Binder.getCallingPid();
            }

            userId = mUserController.handleIncomingUser(callingPid, callingUid, userId, true,
                    ALLOW_FULL_ONLY, "registerReceiver", callerPackage);
            
            //【2】通过解析 IntentFilter 获得 BroadcastReceiver 设置的 action 列表！
            Iterator<String> actions = filter.actionsIterator();
            if (actions == null) {
                ArrayList<String> noAction = new ArrayList<String>(1);
                noAction.add(null);
                actions = noAction.iterator();
            }

            // 收集所有设备用户；
            int[] userIds = { UserHandle.USER_ALL, UserHandle.getUserId(callingUid) };
            
            //【3】遍历广播接收者的 actions 列表，找到能够和 action 匹配的粘性广播，添加到 stickyIntents 集合中！
            while (actions.hasNext()) {
                String action = actions.next();
                for (int id : userIds) {
                    //【3.1】获得每个设备用户下的“粘性”广播！
                    ArrayMap<String, ArrayList<Intent>> stickies = mStickyBroadcasts.get(id);
                    if (stickies != null) {
                        ArrayList<Intent> intents = stickies.get(action);
                        if (intents != null) {
                            if (stickyIntents == null) {
                                stickyIntents = new ArrayList<Intent>();
                            }
                            stickyIntents.addAll(intents);
                        }
                    }
                }
            }
        }
        
        //【4】因为 Intent 出了设置 action 这个属性之外，还会设置其他的一些属性！
        // 这里是查找完全匹配的“粘性”广播，添加到 allSticky 集合中！
        ArrayList<Intent> allSticky = null;
        if (stickyIntents != null) {
            final ContentResolver resolver = mContext.getContentResolver();
            for (int i = 0, N = stickyIntents.size(); i < N; i++) {
                Intent intent = stickyIntents.get(i);
                if (filter.match(resolver, intent, true, TAG) >= 0) {
                    if (allSticky == null) {
                        allSticky = new ArrayList<Intent>();
                    }
                    allSticky.add(intent);
                }
            }
        }

        //【5】我们会将第一个完全匹配的粘性广播发给接收者，如果动态注册的接收者为 null 的话，立刻返回！
        Intent sticky = allSticky != null ? allSticky.get(0) : null;
        if (DEBUG_BROADCAST) Slog.v(TAG_BROADCAST, "Register receiver " + filter + ": " + sticky);
        if (receiver == null) {
            return sticky;
        }

        synchronized (this) {
            if (callerApp != null && (callerApp.thread == null
                    || callerApp.thread.asBinder() != caller.asBinder())) {
                //【6】如果注册者进程死亡了，就退出！
                return null;
            }
            
            //【7】尝试获得广播接收者在系统进程中的 ReceiverList 对象，如果为 null，说明没有注册！ 
            ReceiverList rl = mRegisteredReceivers.get(receiver.asBinder());
            if (rl == null) {

                //【*3.1.4】创建一个全新的 ReceiverList 对象，封装该动态注册的广播接收者！
                rl = new ReceiverList(this, callerApp, callingPid, callingUid,
                        userId, receiver);
                
                //【*5.2】如果 rl.app 不为 null，说明注册者进程已经启动，就将  ReceiverList 对象添加
                // 到注册者的进程的 receivers 集合中，对于动态注册，显然进入这个分支！
                if (rl.app != null) {
                    rl.app.receivers.add(rl);

                } else {
                    try {
                        // 如果 rl.app 为 bull，这可能是一种异常状态！！直接 linkToDeath 进行 Binder 监听！
                        receiver.asBinder().linkToDeath(rl, 0);
                    } catch (RemoteException e) {
                        return sticky;
                    }
                    rl.linkedToDeath = true;
                }
                
                //【6】将 InnerReceiver.proxy 和 ReceiverList 的映射关系保存到 mRegisteredReceivers 中！
                // 表示该广播接收者已经被注册！
                mRegisteredReceivers.put(receiver.asBinder(), rl);

            } else if (rl.uid != callingUid) {
                throw new IllegalArgumentException(
                        "Receiver requested to register for uid " + callingUid
                        + " was previously registered for uid " + rl.uid);
            } else if (rl.pid != callingPid) {
                throw new IllegalArgumentException(
                        "Receiver requested to register for pid " + callingPid
                        + " was previously registered for pid " + rl.pid);
            } else if (rl.userId != userId) {
                throw new IllegalArgumentException(
                        "Receiver requested to register for user " + userId
                        + " was previously registered for user " + rl.userId);
            }
            
            //【7】创建 BroadcastFilter 对象 bf，用来封装广播接收者的 IntentFilter 对象！
            BroadcastFilter bf = new BroadcastFilter(filter, rl, callerPackage,
                    permission, callingUid, userId);

            // 将 BroadcastFilter 对象添加到 ReceiverList 列表中！
            rl.add(bf);

            if (!bf.debugCheck()) {
                Slog.w(TAG, "==> For Dynamic broadcast");
            }
            
            //【8】将创建的广播过滤器 BroadcastFilter 保存到 mReceiverResolver 中，用于查询使用！
            mReceiverResolver.addFilter(bf);

            //【9】如果 allSticky 不为 null，说明有能匹配该广播过滤器的 “粘性广播”！
            // 接着就是要处理粘性广播的分发了！
            if (allSticky != null) {
                //【9.1】将该动态注册的广播接收者添加到 ArrayList 中！
                // 准备将该粘性广播发送给接收者！
                ArrayList receivers = new ArrayList();
                receivers.add(bf);

                final int stickyCount = allSticky.size();
                for (int i = 0; i < stickyCount; i++) {
                    Intent intent = allSticky.get(i);
                    //【9.2】根据 Intent 是否设置 Intent.FLAG_RECEIVER_FOREGROUND 标志位，
                    // 选择前台或者后台广播队列！
                    BroadcastQueue queue = broadcastQueueForIntent(intent);
                    
                    //【9.3】创建 BroadcastRecord 对象！！
                    BroadcastRecord r = new BroadcastRecord(queue, intent, null,
                            null, -1, -1, null, null, AppOpsManager.OP_NONE, null, receivers,
                            null, 0, null, null, false, true, true, -1);

                    //【9.4】将粘性广播加入到广播队列的并发广播集合中！！
                    queue.enqueueParallelBroadcastLocked(r);
                    
                    //【2.3】处理广播队列中的已有的广播！！
                    queue.scheduleBroadcastsLocked();
                }
            }

            return sticky;
        }
    }
```

下面我们来分析下这个方法中的一些重要步骤：

### 2.1.1 ActivityManagerS.getRecordForAppLocked

获得注册者进程的 ProcessRecord 对象，传入参数：

- **IApplicationThread thread**：注册者进程在系统进程中的 ApplicationThreadProxy 对象！

```java
    final ProcessRecord getRecordForAppLocked(
            IApplicationThread thread) {
        if (thread == null) {
            return null;
        }

        // 获得进程 ProcessRecord 在 mLruProcesses 中的下标！
        int appIndex = getLRURecordIndexForAppLocked(thread);
        return appIndex >= 0 ? mLruProcesses.get(appIndex) : null;
    }
```
这里调用了 getLRURecordIndexForAppLocked 方法：
```java
    private final int getLRURecordIndexForAppLocked(IApplicationThread thread) {
        IBinder threadBinder = thread.asBinder();

        // Find the application record.
        for (int i=mLruProcesses.size()-1; i>=0; i--) {
            ProcessRecord rec = mLruProcesses.get(i);

            // 找到正在运行，并且 IApplicationThread 对象可以匹配的进程！
            if (rec.thread != null && rec.thread.asBinder() == threadBinder) {
                return i;
            }
        }
        return -1;
    }
```
mLruProcesses 是一个 ArrayList 集合，用来保存正在运行中的应用程序进程，这里我们先不看！

### 2.1.2 IntentFilter.match

主要代码段：
```java
    if (filter.match(resolver, intent, true, TAG) >= 0) {
        if (allSticky == null) {
            allSticky = new ArrayList<Intent>();
        }
        allSticky.add(intent);
    }
```
当前要注册的广播接收者只能接收到能够和 IntentFilter 匹配的广播，所以对于“粘性广播”，我们还需要再进行一次匹配操作，找到能够匹配 IntentFilter 的粘性广播，保存到 allSticky 列表中，我们来看看匹配过程：

```java
    public final int match(ContentResolver resolver, Intent intent,
            boolean resolve, String logTag) {

        String type = resolve ? intent.resolveType(resolver) : intent.getType();

        return match(intent.getAction(), type, intent.getScheme(),
                     intent.getData(), intent.getCategories(), logTag);
    }
```


### 2.1.3 ActivityManagerS.broadcastQueueForIntent

该方法的作用是为广播选择合适的队列：
```java
    BroadcastQueue broadcastQueueForIntent(Intent intent) {
        final boolean isFg = (intent.getFlags() & Intent.FLAG_RECEIVER_FOREGROUND) != 0;

        if (DEBUG_BROADCAST_BACKGROUND) Slog.i(TAG_BROADCAST,
                "Broadcast intent " + intent + " on "
                + (isFg ? "foreground" : "background") + " queue");

        return (isFg) ? mFgBroadcastQueue : mBgBroadcastQueue;
    }
```
如果 Intent 设置了 Intent.FLAG_RECEIVER_FOREGROUND 标志位，那该广播就是一个前台广播，应该被加入 mFgBroadcastQueue 队列；否则，这是一个后台广播，就应该被添加到 mBgBroadcastQueue 队列中；

### 2.1.4 new BroadcastRecord

创建广播管理对象 BroadcastRecord，参数传递：

- **BroadcastQueue _queue**：表示广播所在的队列，传入 queue
- **Intent _intent**：传入 intent
- **ProcessRecord _callerApp**：传入 null
- **String _callerPackage**：传入 null
- **int _callingPid**：传入 -1
- **int _callingUid**：传入 -1
- **String _resolvedType**：传入 null
- **String[] _requiredPermissions**：传入 null
- **int _appOp**：传入 AppOpsManager.OP_NONE
- **BroadcastOptions _options**：传入 null
- **List _receivers**：传入 receivers，这里是
- **IIntentReceiver _resultTo**：传入 null
- **int _resultCode**：传入 0
- **String _resultData**：传入 null
- **Bundle _resultExtras**：传入 null
- **boolean _serialized**：传入 false
- **boolean _sticky**：传入 true
- **boolean _initialSticky**：传入 true
- **int _userId**：传入 -1

```java
    BroadcastRecord(BroadcastQueue _queue,
            Intent _intent, ProcessRecord _callerApp, String _callerPackage,
            int _callingPid, int _callingUid, String _resolvedType, String[] _requiredPermissions,
            int _appOp, BroadcastOptions _options, List _receivers, IIntentReceiver _resultTo,
            int _resultCode, String _resultData, Bundle _resultExtras, boolean _serialized,
            boolean _sticky, boolean _initialSticky,
            int _userId) {

        // 如果广播对应的 Intent 为 null，抛出异常
        if (_intent == null) {
            throw new NullPointerException("Can't construct with a null intent");
        }
        
        queue = _queue;  // 广播所属的广播队列，有前台和后台之分；
        intent = _intent; // 广播 Intent
        targetComp = _intent.getComponent(); // 广播的目标组件，也就是目标 BroadcastReceiver

        callerApp = _callerApp; // 发送者所在的进程
        callerPackage = _callerPackage; // 发送者所属的包名
        callingPid = _callingPid; // 发送者的进程 pid
        callingUid = _callingUid; // 发送者进程的 uid

        resolvedType = _resolvedType;
        requiredPermissions = _requiredPermissions; // 发送该广播需要的权限
        appOp = _appOp;
        options = _options;
    
        receivers = _receivers; //
        delivery = new int[_receivers != null ? _receivers.size() : 0]; // 该广播分发给每个接收者的次数

        resultTo = _resultTo;
        resultCode = _resultCode;
        resultData = _resultData;
        resultExtras = _resultExtras;
        ordered = _serialized;
        sticky = _sticky;
        initialSticky = _initialSticky;
        userId = _userId;
        nextReceiver = 0;
        state = IDLE;
    }
```

这里是对“粘性”广播创建对应的 BroadcastRecord 管理对象！

### 2.1.5 BroadcastQueue.enqueueParallelBroadcastLocked

将“粘性”广播 BroadcastRecord 添加到 BroadcastQueue 对象内部的 mParallelBroadcasts 列表中，并初始化 r.enqueueClockTime 为当前系统时间，表示广播进入队列的时间；
```java
    public void enqueueParallelBroadcastLocked(BroadcastRecord r) {
        mParallelBroadcasts.add(r);
        r.enqueueClockTime = System.currentTimeMillis();
    }
```
可以看出，粘性广播属于并发的广播！

### 2.1.6 BroadcastQueue.scheduleBroadcastsLocked

开始处理广播，mBroadcastsScheduled 表示是否开始广播处理任务！
```java
    public void scheduleBroadcastsLocked() {
        if (DEBUG_BROADCAST) Slog.v(TAG_BROADCAST, "Schedule broadcasts ["
                + mQueueName + "]: current="
                + mBroadcastsScheduled);

        if (mBroadcastsScheduled) {
            return;
        }
        
        //【1】发送 BROADCAST_INTENT_MSG 消息！
        mHandler.sendMessage(mHandler.obtainMessage(BROADCAST_INTENT_MSG, this));
        mBroadcastsScheduled = true;
    }
```
收集了粘性广播，把粘性广播添加到了指定的广播队列中，然后就是要处理广播了！mHandler 是 BroadcastHandler 对象，这里是向 BroadcastHandler 发送 BROADCAST_INTENT_MSG 消息，开始分发广播！！

## 2.2 BroadcastQueue.scheduleBroadcastsLocked

```java
    public void scheduleBroadcastsLocked() {
        if (DEBUG_BROADCAST) Slog.v(TAG_BROADCAST, "Schedule broadcasts ["
                + mQueueName + "]: current="
                + mBroadcastsScheduled);

        if (mBroadcastsScheduled) {
            return;
        }
        //【1】触发广播的分发！
        mHandler.sendMessage(mHandler.obtainMessage(BROADCAST_INTENT_MSG, this));
        mBroadcastsScheduled = true;
    }
```

## 2.3 BroadcastQueue.BroadcastHandler
```java
    private final class BroadcastHandler extends Handler {
        public BroadcastHandler(Looper looper) {
            super(looper, null, true);
        }

        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                
                case BROADCAST_INTENT_MSG: {
                    if (DEBUG_BROADCAST) Slog.v(
                            TAG_BROADCAST, "Received BROADCAST_INTENT_MSG");
                    //【1】接收到 BROADCAST_INTENT_MSG 消息，开始分发广播！
                    processNextBroadcast(true);

                } break;
                
                case BROADCAST_TIMEOUT_MSG: {
                    //【2】广播处理超时
                    synchronized (mService) {
                        broadcastTimeoutLocked(true);
                    }
                } break;
                
                case SCHEDULE_TEMP_WHITELIST_MSG: {
                    //【3】添加 doze 模式白名单，这个在 doze 模式篇中会讲解！！
                    DeviceIdleController.LocalService dic = mService.mLocalDeviceIdleController;
                    if (dic != null) {
                        dic.addPowerSaveTempWhitelistAppDirect(UserHandle.getAppId(msg.arg1),
                                msg.arg2, true, (String)msg.obj);
                    }
                } break;
            }
        }
    }
```
接着，调用 processNextBroadcast 方法，分发队列中的广播，分发的具体流程，我们会在 sendBroadcast 流程分析那篇博文中详细解析，这里先不看！


# 3 总结

## 3.1 数据结构分析

以上过程涉及到了一些重要的数据结构！

### 3.1.1 **BroadcastRecevier** 
```java
public abstract class BroadcastReceiver {
    private PendingResult mPendingResult;
    
    public abstract void onReceive(Context context, Intent intent);
}
```
BroadcastReceiver 是一个抽象类，我们在开发时，需要继承 BroadcastReceiver 来实现自己的广播接收者！

### 3.1.2 **mStickyBroadcasts**

```java
    final SparseArray<ArrayMap<String, ArrayList<Intent>>> mStickyBroadcasts =
            new SparseArray<ArrayMap<String, ArrayList<Intent>>>();
```
mStickyBroadcasts 用来保存每个设备用户下活跃的粘性广播！

这里来解释下粘性广播，粘性广播是可以发送给以后注册的接受者的广播，意思是**系统会将前面的粘性广播保存在 AMS 中，一旦注册了能够接受对应粘性广播的 BroadcastReceiver，在注册结束后，BroadcastReceiver 会立即收到粘性广播**！

mStickyBroadcasts 是一个稀疏数组，数组下标为设备用户ID：userId，对应的数组元素是设备用户 ID 下所有活跃的粘性广播：`ArrayMap<String, ArrayList<Intent>>`；


`ArrayMap<String, ArrayList<Intent>>`  的 key 是粘性广播 Intent 的 action，value 是一个 ArrayList：是所有设置了同一个 action 的粘性广播 Intent；

### 3.1.3 **mRegisteredReceivers**

```java
    final HashMap<IBinder, ReceiverList> mRegisteredReceivers = new HashMap<>();
```
用来保存动态注册的广播接收者，是一个 HashMap，key 是广播接收者的 IInnerReceiver 对象，value 是 ReceiverList 对象，是一个 ArrayList 集合，用来封装该广播接收者的信息！

### 3.1.4 **ReceiverList**

ReceiverList 用于封装每一个动态创建

```java
final class ReceiverList extends ArrayList<BroadcastFilter>
        implements IBinder.DeathRecipient {

    final ActivityManagerService owner; // AMS 对象
    public final IIntentReceiver receiver; // 广播接收者的 InnerReceiver.Proxy 对象

    public final ProcessRecord app; // 注册者所在的进程
    public final int pid; // 注册者的进程 pid
    public final int uid; // 注册者的进程 uid
    public final int userId; // 注册者所在的设备用户 id

    BroadcastRecord curBroadcast = null; // 接收者当前正在处理的广播

    boolean linkedToDeath = false;

    String stringName;
    
    ReceiverList(ActivityManagerService _owner, ProcessRecord _app,
            int _pid, int _uid, int _userId, IIntentReceiver _receiver) {
        owner = _owner;
        receiver = _receiver;
        app = _app;
        pid = _pid;
        uid = _uid;
        userId = _userId;
    }
}
```
ReceiverList 继承了 ArrayList<BroadcastFilter>，所以它本质上就是一个 ArrayList 集合！

### 3.1.5 **BroadcastFilter**

BroadcastFilter 表示的动态设定的

```java
final class BroadcastFilter extends IntentFilter {
    // Back-pointer to the list this filter is in.
    final ReceiverList receiverList; // BroadcastFilter 所属的 receiverList 列表；
    final String packageName; // 注册者的包名；
    final String requiredPermission; // 广播需要的权限
    final int owningUid; // 注册者的 uid
    final int owningUserId; // 注册者所属的 UserId

    BroadcastFilter(IntentFilter _filter, ReceiverList _receiverList,
            String _packageName, String _requiredPermission, int _owningUid, int _userId) {
        super(_filter);
        receiverList = _receiverList;
        packageName = _packageName;
        requiredPermission = _requiredPermission;
        owningUid = _owningUid;
        owningUserId = _userId;
    }
    
}
```
BroadcastFilter 继承了 IntentFilter，其本身是一个 IntentFilter 扩展类，出了有 IntentFilter 的属性和方法之外，他还封装了一些其他的属性，我们可以把 BroadcastFilter 理解为注册广播是传入的 IntentFilter！

### 3.1.6 **mReceiverResolver**

mReceiverResolver 是模板类 IntentResolver 的子类，用于解析 Intent 对象！

```java
    final IntentResolver<BroadcastFilter, BroadcastFilter> mReceiverResolver
            = new IntentResolver<BroadcastFilter, BroadcastFilter>() {

        @Override
        protected boolean allowFilterResult(
                BroadcastFilter filter, List<BroadcastFilter> dest) {
            IBinder target = filter.receiverList.receiver.asBinder();
            for (int i = dest.size() - 1; i >= 0; i--) {
                if (dest.get(i).receiverList.receiver.asBinder() == target) {
                    return false;
                }
            }
            return true;
        }

        @Override
        protected BroadcastFilter newResult(BroadcastFilter filter, int match, int userId) {
            if (userId == UserHandle.USER_ALL || filter.owningUserId == UserHandle.USER_ALL
                    || userId == filter.owningUserId) {
                return super.newResult(filter, match, userId);
            }
            return null;
        }

        @Override
        protected BroadcastFilter[] newArray(int size) {
            return new BroadcastFilter[size];
        }

        @Override
        protected boolean isPackageForFilter(String packageName, BroadcastFilter filter) {
            return packageName.equals(filter.packageName);
        }
    };
```
mReceiverResolver 持有所有动态注册的广播接受者的 BroadcastFilter 对象，也就是 IntentFilter，当广播没有设置组件信息时，会通过 mReceiverResolver 查询接收者的信息！


### 3.1.7 **BroadcastQueue**

BroadcastQueue 表示的是广播队列，任何一个广播都需要被添加到这个队列才能被分发！Android 系统提供了两个队列：
```java
    BroadcastQueue mFgBroadcastQueue;
    BroadcastQueue mBgBroadcastQueue;
    
    final BroadcastQueue[] mBroadcastQueues = new BroadcastQueue[2];

    // 广播接收者运行超时时间
    static final int BROADCAST_FG_TIMEOUT = 10*1000;
    static final int BROADCAST_BG_TIMEOUT = 60*1000;
```
他们是在 AMS 的启动时候初始化的：
```java
        mFgBroadcastQueue = new BroadcastQueue(this, mHandler,
                "foreground", BROADCAST_FG_TIMEOUT, false);
        mBgBroadcastQueue = new BroadcastQueue(this, mHandler,
                "background", BROADCAST_BG_TIMEOUT, true);

        mBroadcastQueues[0] = mFgBroadcastQueue;
        mBroadcastQueues[1] = mBgBroadcastQueue;
```

一个是前台队列，一个是后台队列！一个广播会根据是否设置了 Intent.FLAG_RECEIVER_FOREGROUND 标志位，会被放到 mFgBroadcastQueue 或者 mBgBroadcastQueue 队列中进行分发，我们去看看 BroadcastQueue 类的一些重要的成员变量：

```java
public final class BroadcastQueue {
    static final int MAX_BROADCAST_HISTORY = ActivityManager.isLowRamDeviceStatic() ? 10 : 50;
    static final int MAX_BROADCAST_SUMMARY_HISTORY
            = ActivityManager.isLowRamDeviceStatic() ? 25 : 300;

    final ActivityManagerService mService;

    final String mQueueName; // 队列的名称
    final long mTimeoutPeriod;  // 广播接收者运行超时时间
    final boolean mDelayBehindServices; // 是否延迟发送广播，后台队列才会为 true！
    
    final ArrayList<BroadcastRecord> mParallelBroadcasts = new ArrayList<>(); // 并行广播列表
    final ArrayList<BroadcastRecord> mOrderedBroadcasts = new ArrayList<>(); // 有序广播列表

    // 队列中广播的历史记录，用于 debugging！
    final BroadcastRecord[] mBroadcastHistory = new BroadcastRecord[MAX_BROADCAST_HISTORY];
    int mHistoryNext = 0;

    final Intent[] mBroadcastSummaryHistory = new Intent[MAX_BROADCAST_SUMMARY_HISTORY];
    int mSummaryHistoryNext = 0;
    
    // 用于记录所有广播的操作时间！
    final long[] mSummaryHistoryEnqueueTime = new  long[MAX_BROADCAST_SUMMARY_HISTORY];
    final long[] mSummaryHistoryDispatchTime = new  long[MAX_BROADCAST_SUMMARY_HISTORY];
    final long[] mSummaryHistoryFinishTime = new  long[MAX_BROADCAST_SUMMARY_HISTORY];

    boolean mBroadcastsScheduled = false; // 分发广播时（BROADCAST_INTENT_MSG），该变量会被置为 true！
    boolean mPendingBroadcastTimeoutMessage; // 广播处理超时（BROADCAST_TIMEOUT_MSG），该变量会被置为 true！

    BroadcastRecord mPendingBroadcast = null; // 等待目标进程启动的挂起广播
    int mPendingBroadcastRecvIndex; // 等待目标进程启动的挂起广播的目标接收者的下标

    final BroadcastHandler mHandler; // 用于在系统进程中处理发送广播和广播超时的操作！

    BroadcastQueue(ActivityManagerService service, Handler handler,
            String name, long timeoutPeriod, boolean allowDelayBehindServices) {
        mService = service;
        mHandler = new BroadcastHandler(handler.getLooper());
        mQueueName = name;
        mTimeoutPeriod = timeoutPeriod;
        mDelayBehindServices = allowDelayBehindServices;
    }
    
    ... ... ... ...

}
```
BroadcastQueue 里面还有很多的方法，用于处理广播的分发，超时等等，这个我们后面再看！！

### 3.1.8 **BroadcastRecord**

在应用端，广播的表现形式是一个个 Intent 对象，但是在系统进程中，AMS 会将 Intent 封装成 BroadcastRecord 对象：

```java
final class BroadcastRecord extends Binder {

    final Intent intent;    // 广播对应的 Intent
    final ComponentName targetComp; // Intent 设置的组件
    
    final ProcessRecord callerApp; // 发送该广播的进程
    final String callerPackage; // 发送该广播的 package
    final int callingPid;   // 发送该广播的进程 pid
    final int callingUid;   // 发送该广播的进程 uid
    
    final boolean ordered;  // 该广播是否是有序广播
    final boolean sticky;   // 该广播是否是粘性广播
    final boolean initialSticky; // initial broadcast from register to sticky?
    
    final int userId;       // 广播所属的设备用户 id
    final String resolvedType; // the resolved data type
    final String[] requiredPermissions; // 发送改变广播需要的权限
    final int appOp;        // 和该广播相关联的应用操作
    final BroadcastOptions options; // 发送者提供广播参数
    
    final List receivers;   // 该广播的目标接收者列表，动态接收者为 BroadcastFilter，静态接收者为 ResolveInfo；
    
    final int[] delivery;   // 该广播对于每个广播接收者的分发状态
    IIntentReceiver resultTo; // 用来接收发送广播的结果

    long enqueueClockTime;  // 该广播加入队列的时间！
    long dispatchTime;      // 该广播开始分发的时间（uptimeMillis）
    long dispatchClockTime; // 该广播开始分发的时间（currentTimeMillis）
    long receiverTime;      // 接收者接收该广播的时间，用于计算超时
    long finishTime;        // 该广播完成分发的时间（uptimeMillis）

    int resultCode;         // current result code value.
    String resultData;      // current result data value.
    Bundle resultExtras;    // current result extra data values.
    boolean resultAbort;    // current result abortBroadcast value.
    
    int nextReceiver;       // 下一个广播的序号
    IBinder receiver;       // 当前接收者的 Binder 实体，用于跨进程通信
    int state;              // 广播的状态！

    int anrCount;           // ANR 的次数；
    int manifestCount;      // 目标静态接收者的个数；
    int manifestSkipCount;  // 跳过的静态接收者个数；
    BroadcastQueue queue;   // 该广播所属的队列；

    BroadcastFilter curFilter; // 目标动态接收者！

    ProcessRecord curApp;       // 接受该广播的广播接收者所在进程！
    ComponentName curComponent; // 接受该广播的广播接收者的组件名！
    ActivityInfo curReceiver;   // 接受该广播的静态广播接收者的信息对象！
    
    ... ... ... ...
       
}
```

## 3.2 结构关系图

结构关系图我们从 2 个方面来说明！

### 3.2.1 注册者进程关系图

处理广播的类是 BroadcastReceiver，InnerReceiver 用于实现注册者进程和系统进程的跨进程通信，ReceiverDispatcher 实现了二者的低耦合，用于管理二者的映射关系！

系统进程通过 InnerReceiver.Proxy 代理对象，通过 Binder 通信将广播传递给注册者进程的 BroadcastReceiver！performReceive 方法最终会创建一个 Args 对象，用于拉起 BroadcastReceiver 的 onReceive 方法！

### 3.2.2 系统进程关系图

![广播接收者动态注册系统进程关系图.png-318.2kB](https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/23/广播接收者动态注册系统进程关系图-20221024234842723.png)

