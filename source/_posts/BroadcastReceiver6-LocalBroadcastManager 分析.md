# BroadcastReceiver篇 6 - LocalBroadcastManager 分析
title: BroadcastReceiver篇 6 - LocalBroadcastManager 分析
date: 2016/07/13 20:46:25
catalog: true
categories:
- AndroidFramework源码分析
- BroadcastReceiver广播接收者
tags: BroadcastReceiver广播接收者
---

[toc]


本文基于 Android 7.1.1 源码，分析 LocalBroadcastManager 机制！


# 0 前言

BroadcastReceiver 是基于 Binder 通信的，其可以用于跨进程的通信，而 LocalBroadcastManager 是基于 Handler 的，其适用于进程内的通信，在进程内进行局部广播发送与注册。

相比 BroadcastReceiver 的广播，LocalBroadcastManager 有以下几点优点。

- 广播数据只在本应用内传播，不用担心数据泄露；
- 广播数据不用担心别的应用伪造广播，更加安全；
- 因为只在应用内广播，所以更加的高效；


LocalBroadcastManager 位于 android.support.v4.content 包中，适用于动态注册的广播：


# 1 基本使用


注册接收者到 LocalBroadcastManager：

```java
        filter = new IntentFilter();
        filter.addAction(action);
        mLocalBroadcastManager = LocalBroadcastManager.getInstance(this); // 获得实例；
        mLocalBroadcastManager.registerReceiver(receiver, filter); // 注册监听；
```
解除注册：

```java
mLocalBroadcastManager.unregisterReceiver(receiver); // 取消监听；
```

然后就是发送广播：

```java
mLocalBroadcastManager.sendBroadcast(intent); // 发送广播；
```



# 2 源码分析


## 2.1 LocalBroadcastManager.getInstance - 创建单例

LocalBroadcastManager 采用的是单例模式：

```java
    public static LocalBroadcastManager getInstance(Context context) {
        synchronized (mLock) {
            if (mInstance == null) {
                //【2.1.1】创建单例！
                mInstance = new LocalBroadcastManager(context.getApplicationContext());
            }
            return mInstance;
        }
    }
```
getInstance 方法中会调用 LocalBroadcastManager 构造器：
```java
    private LocalBroadcastManager(Context context) {
        mAppContext = context;
        //【1】创建主线程对应的 Handler！
        mHandler = new Handler(context.getMainLooper()) {
            @Override
            public void handleMessage(Message msg) {
                switch (msg.what) {
                    //【1.2】接收到  MSG_EXEC_PENDING_BROADCASTS 消息！
                    case MSG_EXEC_PENDING_BROADCASTS:
                        //【2.6】触发 executePendingBroadcasts 方法！
                        executePendingBroadcasts();
                        break;
                    default:
                        super.handleMessage(msg);
                }
            }
        };
    }
```
context.getMainLooper() 会返回当前进程主线程的 Looper 对象！

继续分析！

## 2.2 LocalBroadcastManager.registerReceiver - 注册

注册接收者：

```java
    public void registerReceiver(BroadcastReceiver receiver, IntentFilter filter) {
        synchronized (mReceivers) {
            //【2.2.1】创建了一个 ReceiverRecord 实例，封装注册的接收者的信息！
            ReceiverRecord entry = new ReceiverRecord(filter, receiver);
            //【1】添加 receiver 和 intentfilter 的映射关系！
            ArrayList<IntentFilter> filters = mReceivers.get(receiver);
            if (filters == null) {
                filters = new ArrayList<IntentFilter>(1);
                mReceivers.put(receiver, filters);
            }
            filters.add(filter);
            //【2】解析该 filter 能过滤处理的 action，并添加 action 和 receiver 的映射关系！
            for (int i=0; i<filter.countActions(); i++) {
                String action = filter.getAction(i);
                ArrayList<ReceiverRecord> entries = mActions.get(action);
                if (entries == null) {
                    entries = new ArrayList<ReceiverRecord>(1);
                    mActions.put(action, entries);
                }
                entries.add(entry);
            }
        }
    }
```

在 LocalBroadcastManager 有如下两个 hash 表：

```java
    // 存储 BroadcastReceiver 和其 IntentFilter 的映射关系！
    private final HashMap<BroadcastReceiver, ArrayList<IntentFilter>> mReceivers
            = new HashMap<BroadcastReceiver, ArrayList<IntentFilter>>();
    // 存储 filter action 和  BroadcastReceiver 的映射关系！
    private final HashMap<String, ArrayList<ReceiverRecord>> mActions
            = new HashMap<String, ArrayList<ReceiverRecord>>();
```
不多说了！

### 2.2.1 new ReceiverRecord

ReceiverRecord 的代码很简单，不多说了！

```java
    private static class ReceiverRecord {
        final IntentFilter filter; // intent filter
        final BroadcastReceiver receiver;  // 接收者
        boolean broadcasting;  // 表示 receiver 是否被收集到目标列表中！

        ReceiverRecord(IntentFilter _filter, BroadcastReceiver _receiver) {
            filter = _filter;
            receiver = _receiver;
        }
        ... ... ...
    }
```

## 2.3 LocalBroadcastManager.unregisterReceiver - 取消注册


取消注册：

```java
    public void unregisterReceiver(BroadcastReceiver receiver) {
        synchronized (mReceivers) {
            //【1】移除 receiver 以及和 intentfilter 的映射关系！
            ArrayList<IntentFilter> filters = mReceivers.remove(receiver);
            if (filters == null) {
                return;
            }
            //【2】处理 filter 能够过滤的所有 action！
            for (int i=0; i<filters.size(); i++) {
                IntentFilter filter = filters.get(i);
                for (int j=0; j<filter.countActions(); j++) {
                    String action = filter.getAction(j);
                    //【2.1】移除 action 和该 BroadcastReceiver 的映射关系！
                    ArrayList<ReceiverRecord> receivers = mActions.get(action);
                    if (receivers != null) {
                        for (int k=0; k<receivers.size(); k++) {
                            if (receivers.get(k).receiver == receiver) {
                                receivers.remove(k);
                                k--;
                            }
                        }
                        //【2.2】如果没有 receiver 接收该 action，那就移除映射集合！
                        if (receivers.size() <= 0) {
                            mActions.remove(action);
                        }
                    }
                }
            }
        }
    }
```

不多说了！


## 2.4 LocalBroadcastManager.sendBroadcast - 异步广播


sendBroadcast 用于发送异步的广播，该方法在调用会后立刻返回！！

```java
    public boolean sendBroadcast(Intent intent) {
        synchronized (mReceivers) {
            //【1】解析 intent 的相关属性！
            final String action = intent.getAction();
            final String type = intent.resolveTypeIfNeeded(
                    mAppContext.getContentResolver());
            final Uri data = intent.getData();
            final String scheme = intent.getScheme();
            final Set<String> categories = intent.getCategories();

            final boolean debug = DEBUG ||
                    ((intent.getFlags() & Intent.FLAG_DEBUG_LOG_RESOLUTION) != 0);
            if (debug) Log.v(
                    TAG, "Resolving type " + type + " scheme " + scheme
                    + " of intent " + intent);
                    
            //【2】从 mActions 映射表中找到能够处理该 intent 的接收者列表！
            ArrayList<ReceiverRecord> entries = mActions.get(intent.getAction());
            if (entries != null) {
                if (debug) Log.v(TAG, "Action list: " + entries);

                ArrayList<ReceiverRecord> receivers = null; // 保存目标 receiver 实例；
                for (int i=0; i<entries.size(); i++) {
                    //【2.1】处理每一个 ReceiverRecord 实例；
                    ReceiverRecord receiver = entries.get(i);
                    if (debug) Log.v(TAG, "Matching against filter " + receiver.filter);
                    //【2.2】如果 receiver.broadcasting 为 true，说明目标 receiver 已经添加到目标集合中了；
                    if (receiver.broadcasting) {
                        if (debug) {
                            Log.v(TAG, "  Filter's target already added");
                        }
                        continue;
                    }
                    
                    //【2.3】匹配 receiver！
                    int match = receiver.filter.match(action, type, scheme, data,
                            categories, "LocalBroadcastManager");
                            
                    //【2.4】匹配成功，将目标 receiver 加入到目标列表中！
                    if (match >= 0) {
                        if (debug) Log.v(TAG, "  Filter matched!  match=0x" +
                                Integer.toHexString(match));
                        if (receivers == null) {
                            receivers = new ArrayList<ReceiverRecord>();
                        }
                        receivers.add(receiver);
                        receiver.broadcasting = true; // 设置 broadcasting 为 true！
                    } else {
                        if (debug) {
                            String reason;
                            switch (match) {
                                case IntentFilter.NO_MATCH_ACTION: reason = "action"; break;
                                case IntentFilter.NO_MATCH_CATEGORY: reason = "category"; break;
                                case IntentFilter.NO_MATCH_DATA: reason = "data"; break;
                                case IntentFilter.NO_MATCH_TYPE: reason = "type"; break;
                                default: reason = "unknown reason"; break;
                            }
                            Log.v(TAG, "  Filter did not match: " + reason);
                        }
                    }
                }
                
                //【3】收集完所有匹配的 receiver，开始发送广播！
                if (receivers != null) {
                    //【3.1】将所有 receiver 的 broadcasting 置为 false；
                    for (int i=0; i<receivers.size(); i++) {
                        receivers.get(i).broadcasting = false;
                    }
                    //【2.4.1】创建 BroadcastRecord 实例，将其添加到 mPendingBroadcasts 列表中！
                    mPendingBroadcasts.add(new BroadcastRecord(intent, receivers));
                    if (!mHandler.hasMessages(MSG_EXEC_PENDING_BROADCASTS)) {
                        //【3.1】发送 MSG_EXEC_PENDING_BROADCASTS，处理广播！
                        mHandler.sendEmptyMessage(MSG_EXEC_PENDING_BROADCASTS);
                    }
                    return true;
                }
            }
        }
        return false;
    }
```

在 LocalBroadcastManager 还有如下的一个 list 表：
```java
    // 保存所有等待发送的 broadcast！
    private final ArrayList<BroadcastRecord> mPendingBroadcasts
            = new ArrayList<BroadcastRecord>();
```

其实，sendBroadcast 方法，本质上是将 BroadcastRecord 添加到内部的 mPendingBroadcasts 集合中，然后发送 MSG_EXEC_PENDING_BROADCASTS 消息，处理广播！

之所以是异步的原因是，sendBroadcast 方法只是将广播加入到了 mPendingBroadcasts 中后，会立刻返回；

而广播的分发是在主线程，如果我们在子线程中 sendBroadcast 的话，整个过程显然是异步的！

继续分析：

### 2.4.1 new BroadcastRecord

用于表示一个广播实例：

```java
    private static class BroadcastRecord {
        final Intent intent; // 意图 
        final ArrayList<ReceiverRecord> receivers; // 目标 receiver 列表；

        BroadcastRecord(Intent _intent, ArrayList<ReceiverRecord> _receivers) {
            intent = _intent;
            receivers = _receivers;
        }
    }
```

不用多说了！



## 2.5 LocalBroadcastManager.sendBroadcastSync - 同步广播

sendBroadcastSync 用于同步发送广播：

```java
    public void sendBroadcastSync(Intent intent) {
        //【2.4】将广播添加到 mPendingBroadcasts 列表中！
        if (sendBroadcast(intent)) {
            //【2.6】立刻触发广播的接收！
            executePendingBroadcasts();
        }
    }
```
sendBroadcastSync 方法之所以是同步的是因为，当我们 sendBroadcast 将广播加入到 mPendingBroadcasts 列表中后，会立刻调用 executePendingBroadcasts 分发广播，整个流程都是在同一个线程中执行的！

**注意**：

> **按照以往认知**：BroadcastReceiver 的 onReceive 应该是在主线程中调用的，但是在这里，BroadcastReceiver 的 onReceive 方法却可以在非主线程中调用！

> **主要差别是**：两种情况下的实现方式不同，前者是跨进程通信，所以 onReceive 始终是在主线程中拉起的；而后者由于实质上只是一个回调，所以可以在任何线程中拉起！


## 2.6 LocalBroadcastManager.executePendingBroadcasts - 接收广播


executePendingBroadcasts 用于分发和接收广播！

```java
    private void executePendingBroadcasts() {
        while (true) {
            //【1】将 mPendingBroadcasts 中待处理的广播拷贝到临时分发数组 brs 中
            // 并清空 mPendingBroadcasts；
            BroadcastRecord[] brs = null;
            synchronized (mReceivers) {
                final int N = mPendingBroadcasts.size();
                if (N <= 0) {
                    return; // 如果 mPendingBroadcasts 为 null，那就结束发送；
                }
                brs = new BroadcastRecord[N];
                mPendingBroadcasts.toArray(brs);
                mPendingBroadcasts.clear();
            }
            //【2】拉起所有 receiver 的 onReceive 方法，处理广播；
            for (int i=0; i<brs.length; i++) {
                BroadcastRecord br = brs[i];
                for (int j=0; j<br.receivers.size(); j++) {
                    br.receivers.get(j).receiver.onReceive(mAppContext, br.intent);
                }
            }
        }
    }
```
逻辑也很清楚！
