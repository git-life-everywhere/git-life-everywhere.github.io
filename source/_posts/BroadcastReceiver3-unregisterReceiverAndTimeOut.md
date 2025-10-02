# BroadcastReceiver篇 3 - unregisterReceiver 和 TimeOut 流程分析
title: BroadcastReceiver篇 3 - unregisterReceiver 和 TimeOut 流程分析
date: 2016/05/04 20:46:25
catalog: true
categories:
- AndroidFramework源码分析
- BroadcastReceiver广播接收者
tags: BroadcastReceiver广播接收者
---

[toc]

基于 Android 7.1.1 源码，分析 BroadcastReceiver **动态注册** 的过程，转载请说明出处！

# 0 综述

BroadcastReceiver 动态注册后，需要动态取消注册：

```java
   public void unregisterReceiver(BroadcastReceiver receiver);
```

下面，我们来进入源码，分析 `unregisterReceiver` 的流程！

# 1 应用进程

我们先去应用进程来看下：

## 1.1 ContextWrapper.unregisterReceiver
```java
    @Override
    public void unregisterReceiver(BroadcastReceiver receiver) {
        mBase.unregisterReceiver(receiver);
    }
```
这里的 `mBase` 为 `ContextImpl` 实例！

## 1.2 ContextImpl.unregisterReceiver

```java
    @Override
    public void unregisterReceiver(BroadcastReceiver receiver) {
        if (mPackageInfo != null) {
            //【*1.2.1】获得接收者对应的 InnerReceiver 对象！
            IIntentReceiver rd = mPackageInfo.forgetReceiverDispatcher(
                    getOuterContext(), receiver);
            try {

                //【*1.2.2】进入系统进程，取消动态接收者的注册！
                // 这是同步调用！
                ActivityManagerNative.getDefault().unregisterReceiver(rd);
            } catch (RemoteException e) {
                throw e.rethrowFromSystemServer();
            }
        } else {
            throw new RuntimeException("Not supported in system context");
        }
    }
```
mPackageInfo 是 LoadedApk 实例，用来保存应用程序的加载信息，每一个进程都会有一个！

这里我们来分析下：

### 1.2.1 LoadedApk.forgetReceiverDispatcher
```java
    public IIntentReceiver forgetReceiverDispatcher(Context context,
            BroadcastReceiver r) {
        synchronized (mReceivers) {
            //【1】首先，获得当前设备用户下，动态注册的所有接收者的 map 表！
            ArrayMap<BroadcastReceiver, LoadedApk.ReceiverDispatcher> map = mReceivers.get(context);
            LoadedApk.ReceiverDispatcher rd = null;
            if (map != null) {
                //【1.2】获得该接收者对应的 ReceiverDispatcher 对象！
                rd = map.get(r);
                if (rd != null) {

                    //【1.3】从 map 表中移除该 rd 对象！
                    map.remove(r);
                    if (map.size() == 0) {
                        mReceivers.remove(context);
                    }
                    
                    //【1.4】只有设置了 debug 模式，才会进入！
                    if (r.getDebugUnregister()) {
                        ArrayMap<BroadcastReceiver, LoadedApk.ReceiverDispatcher> holder
                                = mUnregisteredReceivers.get(context);
                        
                        //【1.5】会将该接触注册的接收者添加到 mUnregisteredReceivers 集合中！
                        if (holder == null) {
                            holder = new ArrayMap<BroadcastReceiver, LoadedApk.ReceiverDispatcher>();
                            mUnregisteredReceivers.put(context, holder);
                        }
                        RuntimeException ex = new IllegalArgumentException(
                                "Originally unregistered here:");
                        ex.fillInStackTrace();
                        rd.setUnregisterLocation(ex);
                        holder.put(r, rd);
                    }
                    
                    //【1.6】设置 ReceiverDispatcher 对象的 mForgotten 值为 true！
                    rd.mForgotten = true;
                    
                    //【1.7】返回内部的 InnerReceiver 对象！
                    return rd.getIIntentReceiver();
                }
            }
            
            //【2】每个动态注册的接收者都只能被 unregisteredReceivers 一次，不然会抛出异常！
            ArrayMap<BroadcastReceiver, LoadedApk.ReceiverDispatcher> holder
                    = mUnregisteredReceivers.get(context);
            if (holder != null) {
                rd = holder.get(r);
                if (rd != null) {
                    RuntimeException ex = rd.getUnregisterLocation();
                    throw new IllegalArgumentException(
                            "Unregistering Receiver " + r
                            + " that was already unregistered", ex);
                }
            }
            if (context == null) {
                throw new IllegalStateException("Unbinding Receiver " + r
                        + " from Context that is no longer in use: " + context);
            } else {
                throw new IllegalArgumentException("Receiver not registered: " + r);
            }

        }
    }
```
这个方法的主要作用是：

- 删除 ReceiverDispatcher 对象；
- 返回接收者对应的 InnerReceiver 对象；

继续来看：

### 1.2.2 ActivityManagerProxy.unregisterReceiver


```java
    public void unregisterReceiver(IIntentReceiver receiver) throws RemoteException
    {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IActivityManager.descriptor);
        data.writeStrongBinder(receiver.asBinder());
        //【1】Binder 通信：UNREGISTER_RECEIVER_TRANSACTION
        mRemote.transact(UNREGISTER_RECEIVER_TRANSACTION, data, reply, 0);
        reply.readException();
        data.recycle();
        reply.recycle();
    }
```
下面会进入系统进程，我们直接去看！


# 2 系统进程

首先会进入 ActivityManagerN 中去：
```java
        case UNREGISTER_RECEIVER_TRANSACTION:
        {
            data.enforceInterface(IActivityManager.descriptor);
            IBinder b = data.readStrongBinder();
            if (b == null) {
                return true;
            }
            //【1】这里将客户端的 InnerReceiver 转为了服务端的 IIntentReceiver.Proxy 对象！
            IIntentReceiver rec = IIntentReceiver.Stub.asInterface(b);
            //【2】继续调用 unregisterReceiver，解除注册！
            unregisterReceiver(rec);
            reply.writeNoException();
            return true;
        }
```

进入 ActivityManagerS 中去！

## 2.1 ActivityManagerS.unregisterReceiver

这里的 receiver 是接收者在系统进程中的 Binder 对象！

```java
    public void unregisterReceiver(IIntentReceiver receiver) {
        if (DEBUG_BROADCAST) Slog.v(TAG_BROADCAST, "Unregister receiver: " + receiver);

        final long origId = Binder.clearCallingIdentity();
        try {
            boolean doTrim = false;

            synchronized(this) {
                //【1】获得对应的 ReceiverList 过滤器对象！
                ReceiverList rl = mRegisteredReceivers.get(receiver.asBinder());
                if (rl != null) {
                    //【2.1.1】判断接收者是否在处理有序发送的广播，如果在处理广播，就要立刻结束处理！
                    final BroadcastRecord r = rl.curBroadcast;
                    if (r != null && r == r.queue.getMatchingOrderedReceiver(r)) {
                        //【×2.1.2】立刻结束广播处理！
                        final boolean doNext = r.queue.finishReceiverLocked(
                                r, r.resultCode, r.resultData, r.resultExtras,
                                r.resultAbort, false);
                        
                        //【2】继续分发广播！
                        if (doNext) {
                            doTrim = true;
                            r.queue.processNextBroadcast(false);
                        }
                    }
                    if (rl.app != null) {
                        rl.app.receivers.remove(rl);
                    }
                    //【×2.1.3】移除该广播接收者！
                    removeReceiverLocked(rl);
                    //【5】如果绑定了死亡监听，就取消绑定！
                    if (rl.linkedToDeath) {
                        rl.linkedToDeath = false;
                        rl.receiver.asBinder().unlinkToDeath(rl, 0);
                    }
                }
            }

            //【6】回收资源！
            if (doTrim) {
                trimApplications();
                return;
            }

        } finally {
            Binder.restoreCallingIdentity(origId);
        }
    }
```
这段代码主要作用：

- 如果当前接收者正在处理有序发送的广播，那就立刻结束处理；
- 对于有序发送的广播，继续分发广播；
- 移除当前接收者，同时解除绑定；

我们继续分析细节！

### 2.1.1 BroadcastQueue.getMatchingOrderedReceiver

该方法用来判断当前是否有有序分发的广播发送给当前的广播！
```java
    public BroadcastRecord getMatchingOrderedReceiver(IBinder receiver) {
        if (mOrderedBroadcasts.size() > 0) {
            final BroadcastRecord r = mOrderedBroadcasts.get(0);
            if (r != null && r.receiver == receiver) {
                return r;
            }
        }
        return null;
    }
```
判断依据很简单：当有序发送的广播指定了某个动态接收者时，其 `r.receiver` 会保存接收者对应的 `IIntentReceiver.Proxy` 对象！


### 2.1.2 BroadcastQueue.finishReceiverLocked

立刻结束广播的处理，注意：这里的 `waitForServices` 传入的是 `false`，结果表示是否 finish 完成！
```java
    public boolean finishReceiverLocked(BroadcastRecord r, int resultCode,
            String resultData, Bundle resultExtras, boolean resultAbort, boolean waitForServices) {
        final int state = r.state;
        final ActivityInfo receiver = r.curReceiver;
        //【1】初始化广播的状态为 BroadcastRecord.IDLE！
        r.state = BroadcastRecord.IDLE;
        if (state == BroadcastRecord.IDLE) {
            Slog.w(TAG, "finishReceiver [" + mQueueName + "] called but state is IDLE");
        }
        //【2】清空广播的当前目标接收者 Binder 实体！
        r.receiver = null;
        //【3】清空广播的 Intent 组件信息！
        r.intent.setComponent(null);
        
        //【4】清空广播目标进程的 curReceiver！
        if (r.curApp != null && r.curApp.curReceiver == r) {
            r.curApp.curReceiver = null;
        }
        //【5】清空广播目标 filter 所在的接收者处理的当前广播；
        if (r.curFilter != null) {
            r.curFilter.receiverList.curBroadcast = null;
        }
        r.curFilter = null;
        r.curReceiver = null;
        r.curApp = null;
        mPendingBroadcast = null;
        //【6】保存了和返回的结果码等信息；
        r.resultCode = resultCode;
        r.resultData = resultData;
        r.resultExtras = resultExtras;
        //【7】是否忽视结果；
        if (resultAbort && (r.intent.getFlags()&Intent.FLAG_RECEIVER_NO_ABORT) == 0) {
            r.resultAbort = resultAbort;
        } else {
            r.resultAbort = false;
        }
        
        //【8】因为 waitForServices 不进入该分支！
        if (waitForServices && r.curComponent != null && r.queue.mDelayBehindServices
                && r.queue.mOrderedBroadcasts.size() > 0
                && r.queue.mOrderedBroadcasts.get(0) == r) {
            ActivityInfo nextReceiver;
            if (r.nextReceiver < r.receivers.size()) {
                Object obj = r.receivers.get(r.nextReceiver);
                nextReceiver = (obj instanceof ActivityInfo) ? (ActivityInfo)obj : null;
            } else {
                nextReceiver = null;
            }

            if (receiver == null || nextReceiver == null
                    || receiver.applicationInfo.uid != nextReceiver.applicationInfo.uid
                    || !receiver.processName.equals(nextReceiver.processName)) {
                //【8.1】延迟结束该广播的本次分发！
                if (mService.mServices.hasBackgroundServices(r.userId)) {
                    Slog.i(TAG, "Delay finish: " + r.curComponent.flattenToShortString());
                    r.state = BroadcastRecord.WAITING_SERVICES;
                    return false;
                }
            }
        }

        r.curComponent = null;

        //【9】判断广播的状态！
        return state == BroadcastRecord.APP_RECEIVE
                || state == BroadcastRecord.CALL_DONE_RECEIVE;
    }

```
这个过程主要有以下的目的：

- 初始化广播的属性，为下次分发做准备；
- 然后根据广播的状态，判断是否需要继续分发；

对于有序发送广播，如果其目标接收者为动态注册，那么广播的状态为 `BroadcastRecord.CALL_DONE_RECEIVE`, 这里就会返回 true!

这里在 sendBroadcast 流程中有讲，这里就不再细说了！

### 2.1.3 ActivityManagerS.removeReceiverLocked

```java
    void removeReceiverLocked(ReceiverList rl) {
        //【1】从动态注册的接收者列表中移除该接收者！
        mRegisteredReceivers.remove(rl.receiver.asBinder());
        for (int i = rl.size() - 1; i >= 0; i--) {
            //【2】移除该接收者的 filter！
            mReceiverResolver.removeFilter(rl.get(i));
        }
    }
```
这个方法逻辑很简单；

- 从 **mRegisteredReceivers** 中删除该动态注册的接收者；
- 从 **mReceiverResolver** 中删除该动态注册的接收者对应的过滤器对象；

# 3 TimeOut 流程分析

下面我们来看看广播的超时处理！

## 3.1 设置超时

对于有序发送的广播（注意：不是有序广播），是会设置超时处理，防止一个接收者的处理超时，导致其他接收者无法接收到广播。设置超时的地方是在 `processNextBroadcast` 方法中：

### 3.1.1 BroadcastQueue.processNextBroadcast
```java
            r.receiverTime = SystemClock.uptimeMillis();
            if (recIdx == 0) {
                r.dispatchTime = r.receiverTime;
                r.dispatchClockTime = System.currentTimeMillis();
                if (DEBUG_BROADCAST_LIGHT) Slog.v(TAG_BROADCAST, "Processing ordered broadcast ["
                        + mQueueName + "] " + r);
            }
            if (!mPendingBroadcastTimeoutMessage) {
                //【1】计算超时时间点，时间为 r.receiverTime 加上 mTimeoutPeriod！
                long timeoutTime = r.receiverTime + mTimeoutPeriod;
                if (DEBUG_BROADCAST) Slog.v(TAG_BROADCAST,
                        "Submitting BROADCAST_TIMEOUT_MSG ["
                        + mQueueName + "] for " + r + " at " + timeoutTime);
                        
                //【×3.1.2】设置超时任务！
                setBroadcastTimeoutLocked(timeoutTime);
            }
```

每次在有序发送广播给接收者的时候，会设置 `r.receiverTime` 的值，超时是基于这个值来计算的！

`mPendingBroadcastTimeoutMessage` 表示当前是否有超时的消息被处理，默认是为 false！

`timeoutTime` 是超时任务的触发时间点，系统会该时间点发送 `BROADCAST_TIMEOUT_MSG`消息，系统进程主线程会处理该消息！

`mTimeoutPeriod` 表示超时周期，是一段时间间隔！前台队列和后台队列的 `mTimeoutPeriod` 是不一样的：

```java
        mFgBroadcastQueue = new BroadcastQueue(this, mHandler,
                "foreground", BROADCAST_FG_TIMEOUT, false);
        mBgBroadcastQueue = new BroadcastQueue(this, mHandler,
                "background", BROADCAST_BG_TIMEOUT, true);
```

对于**前台队列**，超时周期为 10 s：`static final int BROADCAST_FG_TIMEOUT = 10*1000;`

对于**后台队列**，超时周期为 60 s：`static final int BROADCAST_BG_TIMEOUT = 60*1000;`

### 3.1.2 BroadcastQueue.setBroadcastTimeoutLocked

设置超时消息！

```java
    final void setBroadcastTimeoutLocked(long timeoutTime) {
        if (!mPendingBroadcastTimeoutMessage) {
            Message msg = mHandler.obtainMessage(BROADCAST_TIMEOUT_MSG, this);
            //【1】发送消息！
            mHandler.sendMessageAtTime(msg, timeoutTime);
            
            //【2】将 mPendingBroadcastTimeoutMessage 置为 true！
            mPendingBroadcastTimeoutMessage = true;
        }
    }
```
超时时间点到了后，会发送 `BROADCAST_TIMEOUT_MSG` 消息给系统进程主线程的 Handler 处理！

### 3.1.3 BroadcastHandler.handleMessage[BROADCAST_TIMEOUT_MSG]

这里只列举重点的代码：
```java
           switch (msg.what) {
                case BROADCAST_TIMEOUT_MSG: {
                    synchronized (mService) {
                        //【×3.1.4】处理广播超时！
                        broadcastTimeoutLocked(true);
                    }
                } break;
            }
```

### 3.1.4 BroadcastQueue.broadcastTimeoutLocked

执行超时任务的处理，此时 fromMsg 为 true，因为是通过 BroadcastHandler 触发的超时处理！！
```java
    final void broadcastTimeoutLocked(boolean fromMsg) {
        if (fromMsg) {
            mPendingBroadcastTimeoutMessage = false;
        }
        
        //【1】有序发送的广播队列为空，不处理，返回！
        if (mOrderedBroadcasts.size() == 0) {
            return;
        }

        long now = SystemClock.uptimeMillis();

        //【2】获得超时的广播，因为每次发送的时候，都是从 mOrderedBroadcasts 的 0 index 处获取广播！
        BroadcastRecord r = mOrderedBroadcasts.get(0);

        if (fromMsg) {
            //【2.1】如果系统正在进行 odex 优化，就延迟超时任务的处理，并返回！
            if (mService.mDidDexOpt) {
                mService.mDidDexOpt = false;

                //【×3.1.2】这里延长了 time out 的时间！
                long timeoutTime = SystemClock.uptimeMillis() + mTimeoutPeriod;
                setBroadcastTimeoutLocked(timeoutTime);
                return;
            }
            
            //【2.2】系统进程没有准备好，返回！
            if (!mService.mProcessesReady) {
                return;
            }
            
            //【2.3】如果当前时间没到超时时间点，说明没有超时，再次重置任务，返回！
            long timeoutTime = r.receiverTime + mTimeoutPeriod;
            if (timeoutTime > now) {
                if (DEBUG_BROADCAST) Slog.v(TAG_BROADCAST,
                        "Premature timeout ["
                        + mQueueName + "] @ " + now + ": resetting BROADCAST_TIMEOUT_MSG for "
                        + timeoutTime);
                //【×3.1.2】重新设置超时时间点！
                setBroadcastTimeoutLocked(timeoutTime);
                return;
            }
        }

        BroadcastRecord br = mOrderedBroadcasts.get(0);

        //【3】判断广播的状态，如果此时是 WAITING_SERVICES，说明当前的接收者已经处理完毕，
        // 但是需要等待目标进程后台服务启动后，才能发送给下一个接收者！
        if (br.state == BroadcastRecord.WAITING_SERVICES) {

            Slog.i(TAG, "Waited long enough for: " + (br.curComponent != null
                    ? br.curComponent.flattenToShortString() : "(null)"));
            
            //【3.1】如果此时触发了该广播的超时任务，就直接初始化广播属性，发送给下一个接收者，然后返回！
            br.curComponent = null;
            br.state = BroadcastRecord.IDLE;
            //【3.2】继续分发！
            processNextBroadcast(false);
            return;
        }

        Slog.w(TAG, "Timeout of broadcast " + r + " - receiver=" + r. receiver
                + ", started " + (now - r.receiverTime) + "ms ago");
        
        //【4】广播的接收时间设置为当前！
        r.receiverTime = now;

        //【5】广播的 ANR 次数加 1；
        r.anrCount++;

        // Current receiver has passed its expiration date.
        if (r.nextReceiver <= 0) {
            Slog.w(TAG, "Timeout on receiver with nextReceiver <= 0");
            return;
        }

        ProcessRecord app = null;
        String anrMessage = null;
        
        //【7】设置广播的当前接收者的分发状态为 BroadcastRecord.DELIVERY_TIMEOUT，表示分发超时！
        Object curReceiver = r.receivers.get(r.nextReceiver-1);
        r.delivery[r.nextReceiver-1] = BroadcastRecord.DELIVERY_TIMEOUT;

        Slog.w(TAG, "Receiver during timeout: " + curReceiver);

        logBroadcastReceiverDiscardLocked(r);
        
        //【8】获得广播接收者所在的进程！
        if (curReceiver instanceof BroadcastFilter) {
            BroadcastFilter bf = (BroadcastFilter)curReceiver;
            if (bf.receiverList.pid != 0
                    && bf.receiverList.pid != ActivityManagerService.MY_PID) {
                synchronized (mService.mPidsSelfLocked) {
                    app = mService.mPidsSelfLocked.get(
                            bf.receiverList.pid);
                }
            }
        } else {
            app = r.curApp;
        }
        
        //【9】广播处理超时后，会导致 ANR 的问题，这里是设置 ANR 的信息！
        if (app != null) {
            anrMessage = "Broadcast of " + r.intent.toString();
        }
        
        //【10】如果正在进程启动的广播！
        if (mPendingBroadcast == r) {
            mPendingBroadcast = null;
        }

        //【11】结束当前的接收者；
        finishReceiverLocked(r, r.resultCode, r.resultData,
                r.resultExtras, r.resultAbort, false);
        //【12】将广播发送给下一个接收者；
        scheduleBroadcastsLocked();

        if (anrMessage != null) {
            //【×3.1.5】发送 ANR 消息！
            mHandler.post(new AppNotResponding(app, anrMessage));
        }
    }
```
这个函数的逻辑梳理如下：

- **1** 如果有序列表集合 `mOrderedBroadcasts` 为空，不做任何处理，直接返回；
- **2** 如果系统正在进行 `odex` 优化，就延迟超时任务的处理，并返回；
- **3** 系统进程没有准备好，不做任何处理，直接返回；
- **4** 如果超时处理提前触发了，就重置超时处理，并返回；
- **5** 如果当前广播的状态是 `BroadcastRecord.WAITING_SERVICES`，说明当前接受者已经处理完毕了，但是该广播在等待后台服务的启动，那就初始化广播属性，直接发送给下一个接受者，返回！

如果以上条件不满足，那么该广播处理超时会导致 `ANR` 的问题：

- 设置当前接收者的分发状态为：`BroadcastRecord.DELIVERY_TIMEOUT`
- 获得下一个广播接收者进程 `ProcessRecord `
- 结束当前的接收者 `finishReceiverLocked`；
- 将广播发送给下一个接收者 `scheduleBroadcastsLocked`;

最后，就是很关键的一步了：

- 创建一个 `Runable`，主线程会执行该任务，抛出 `ANR` 异常！


### 3.1.5 new AppNotResponding

`AppNotResponding` 继承了 `Runnable`，我们来看看它的 `run` 方法：
```java
    private final class AppNotResponding implements Runnable {
        private final ProcessRecord mApp;
        private final String mAnnotation;

        public AppNotResponding(ProcessRecord app, String annotation) {
            mApp = app;
            mAnnotation = annotation;
        }

        @Override
        public void run() {
            mService.mAppErrors.appNotResponding(mApp, null, null, false, mAnnotation);
        }
    }
```

系统进程的主线程会执行该 `Runable`，`AppNotResponding` 会调用 `mAppErrors.appNotResponding` 方法处理接收者进程的 `ANR`!

这里我简单的提一下，`AppErrors` 是 `ActivityManagerService` 专门用来处理系统和应用的 `ANR` 和 `Crash` 的类！


后面会写一篇文章来详细分析其处理的过程，这里就不先不细说了！！


## 3.2 取消超时

当有序发送的广播及时处理后，我们会取消超时消息，我们回到 processNextBroadcast 方法中：


```java
    final void processNextBroadcast(boolean fromMsg) {
        synchronized(mService) {
            ... ... ...
            
            do {
                if (mOrderedBroadcasts.size() == 0) {
                    mService.scheduleAppGcsLocked();
                    if (looped) {
                        mService.updateOomAdjLocked();
                    }
                    return;
                }

                r = mOrderedBroadcasts.get(0);
                boolean forceReceive = false;

                ... ... ... ...
                
                //【1】当有序发送的广播发送完成或者被终止的时候，会 remove 掉该超时消息！
                if (r.receivers == null || r.nextReceiver >= numReceivers
                        || r.resultAbort || forceReceive) {

                    if (r.resultTo != null) {
                        try {
                            if (DEBUG_BROADCAST) Slog.i(TAG_BROADCAST,
                                    "Finishing broadcast [" + mQueueName + "] "
                                    + r.intent.getAction() + " app=" + r.callerApp);
                            performReceiveLocked(r.callerApp, r.resultTo,
                                new Intent(r.intent), r.resultCode,
                                r.resultData, r.resultExtras, false, false, r.userId);
                            r.resultTo = null;
                        } catch (RemoteException e) {
                            r.resultTo = null;
                            Slog.w(TAG, "Failure ["
                                    + mQueueName + "] sending broadcast result of "
                                    + r.intent, e);

                        }
                    }

                    //【3.2.1】移除超时消息！
                    cancelBroadcastTimeoutLocked();

                    if (DEBUG_BROADCAST_LIGHT) Slog.v(TAG_BROADCAST,
                            "Finished with ordered broadcast " + r);

                    addBroadcastToHistoryLocked(r);
                    if (r.intent.getComponent() == null && r.intent.getPackage() == null
                            && (r.intent.getFlags()&Intent.FLAG_RECEIVER_REGISTERED_ONLY) == 0) {
                        // This was an implicit broadcast... let's record it for posterity.
                        mService.addBroadcastStatLocked(r.intent.getAction(), r.callerPackage,
                                r.manifestCount, r.manifestSkipCount, r.finishTime-r.dispatchTime);
                    }
                    mOrderedBroadcasts.remove(0);
                    r = null;
                    looped = true;
                    continue;
                }
            } while (r == null);
```
这里可以看到，只有在广播发送完成或者被终止的时候，才会移除消息！


因为每次有序发送广播的时候，都会重新设置超时消息，具体的逻辑见：processNextBroadcast 方法！



### 3.2.1 BroadcastQueue.cancelBroadcastTimeoutLocked

取消广播超时消息：

```java

    final void cancelBroadcastTimeoutLocked() {
        if (mPendingBroadcastTimeoutMessage) {
            mHandler.removeMessages(BROADCAST_TIMEOUT_MSG, this);
            mPendingBroadcastTimeoutMessage = false;
        }
    }
```

# 4 总结

关于 `unregisterReceiver` 和 `TimeOut` 超时处理的逻辑就分析到这里！