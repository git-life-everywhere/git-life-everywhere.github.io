# Handler篇 2 - Handler 源码分析
title: Handler篇 2 - Handler 源码分析
date: 2017/01/19  20:46:25
catalog: true
categories:
- AndroidFramework源码分析
- Handler线程消息机制
tags: Handler线程消息机制
copyright: true
---

基于 Android 7.1.1 源码，分析 handler 的架构和原理。


# 0 前言

本片博客通过以下几个方面总结 Handler 源码的实现架构！


# 1 create Handler

Handler 提供了如下的构造器方法：

```java
    //【方法1】默认使用当前的线程的 Looper 对象创建 Handler，如果当前线程的没有 Looper，那就会抛出异常！
    public Handler() {...}

    //【方法2】默认使用当前的线程的 Looper 对象创建 Handler，如果当前线程的没有 Looper，那就会抛出异常！
    // 同时传入一个 Callback 接口，用于处理消息回调！
    public Handler(Callback callback) {...}

    //【方法3】显示指定一个 Looper 对象，不能为 null
    public Handler(Looper looper) {...}

    //【方法4】可以看成方法 2 和方法 3 的合并，其中 callback 可以为 null！
    public Handler(Looper looper, Callback callback) {...}

    //【方法5】默认使用当前的线程的 Looper 对象创建 Handler，并且显式指定 handler 是否是异步的！
    // handler 默认是同步的，除非显示的指定 async 为 true！
    // @hide
    public Handler(boolean async) {...}

    //【方法6】可以看成方法 2 和方法 5 的合并!!
    // @hide
    public Handler(Callback callback, boolean async) {...}

    //【方法7】 可以看成方法 2，方法 3 和方法 5 的合并!!
    // @hide
    public Handler(Looper looper, Callback callback, boolean async) {...}
```
Handler 默认是同步的，如果我们显式的设置 async 为 true，那么其该 handler 会以异步的方式处理 message 和 runnable！

关于异步的实现，我们后续会分析！

其中，最核心的构造方法是方法 6 和方法 7，其他方法都是调用它们完成初始化的：

## 1.1 Handler(Callback, boolean)

默认使用当前线程的 Looper 对象：

```java
    public Handler(Callback callback, boolean async) {
        if (FIND_POTENTIAL_LEAKS) {
            final Class<? extends Handler> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }

        mLooper = Looper.myLooper(); // 使用当前线程的 Looper 对象！
        if (mLooper == null) { // 如果当前线程没有 Looper 
            throw new RuntimeException(
                "Can't create handler inside thread that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
```

## 1.2 Handler(Looper，Callback, boolean)

显示指定了 Looper 对象！

```java
    public Handler(Looper looper, Callback callback, boolean async) {
        mLooper = looper; 
        mQueue = looper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
```

## 1.2 成员变量

构造器中出现了 Handler 的主要成员变量，下面解释一下，也便于后续分析：

```java
    final Looper mLooper;
```
Looper 对象;

```java
    final MessageQueue mQueue; 
```
消息队列，来自 Looper 对象;

```java
    final Callback mCallback;
```
Callback 回调，该回调会被封装成 message；

```java
    final boolean mAsynchronous; 
```
该 handler 是否是异步的，默认为 false；



# 2 Handler.obtainMessage

Handler 提供了多个 obtainMessage 方法，来复用一个 Message 对象！

```java
    public final Message obtainMessage() {...}

    public final Message obtainMessage(int what) {...}
    
    public final Message obtainMessage(int what, Object obj) {...}

    public final Message obtainMessage(int what, int arg1, int arg2) {...}
    
    public final Message obtainMessage(int what, int arg1, int arg2, Object obj) {...}
```
Handler 允许我们复用 Message 对象，这样可以避免过多的创建 Message 对象！

Message 有一个静态变量 Message sPool 他是一个消息池链表的头元素，obtainMessage 会从该消息池中复用 Message，并用参数初始化复用的 Message！

obtainMessage 方法最终会调用 Message.obtain 方法：

```java
    public final Message obtainMessage(int what, int arg1, int arg2) {
        return Message.obtain(this, what, arg1, arg2);
    }
```
关于 Message.obtain，我们后面会分析，这里我们只需要知道，复用的时候，会从消息池链表的头元素开始，复用消息！


# 3 Handler.sendMessage

Handler 提供了多个 sendMessage 方法，来发送一个 Message 对象！

```java
    public final boolean sendMessage(Message msg) {...}

    public final boolean sendEmptyMessage(int what) {...}

    public final boolean sendEmptyMessageDelayed(int what, long delayMillis) {...}

    public final boolean sendEmptyMessageAtTime(int what, long uptimeMillis) {...}

    public final boolean sendMessageDelayed(Message msg, long delayMillis) {...}

    public boolean sendMessageAtTime(Message msg, long uptimeMillis) {...}

    public final boolean sendMessageAtFrontOfQueue(c msg) {...} // 发生消息到消息队列的头部
```
通过这些方法我们可以实现立即或者延迟发送消息！

如果指定的是延迟时间，那么会通过 SystemClock.uptimeMillis() + delayMillis 方式计算为绝对时间！

sendMessage 最后调用的是 Handler.enqueueMessage，就是将 Message 插入到消息队列中！！

## 3.1 Handler.enqueueMessage

我们来看看 enqueueMessage 方法：
```java
    private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        if (mAsynchronous) { // 如果 Handler 是异步的，那么 Message 也是异步的！
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }
```
通过 MessageQueue.enqueueMessage 方法，将消息压入到消息队列中！

如果 uptimeMillis 为 0 ，会将消息插入到消息队列的头部；如果 uptimeMillis 不为 0 ，会按照 uptimeMillis 将消息插入到消息队列的指定位置；

# 4 Handler.post

Handler 除了可以发送消息，进行线程间通信，还可以执行指定的任务 Runnable，Handler 提供了多个 post 方法供我们选择：

```java
    public final boolean post(Runnable r) {...}
    
    public final boolean postAtTime(Runnable r, long uptimeMillis) {...}
    
    public final boolean postAtTime(Runnable r, Object token, long uptimeMillis) {...}
    
    public final boolean postDelayed(Runnable r, long delayMillis) {...}
    
    public final boolean postAtFrontOfQueue(Runnable r) {...}
```
通过这些方法我们可以实现立即或者延迟执行任务！

Runnable 的执行很有意思：

```java
    public final boolean post(Runnable r){
       return sendMessageDelayed(getPostMessage(r), 0); // 将 Runnable 封装为 Message！！
    }
```
我们去看看 getPostMessage() 方法：

## 4.1 Handler.getPostMessage

getPostMessage 用于将 Runnable 封装为 Message！

```java
    private static Message getPostMessage(Runnable r) {
        Message m = Message.obtain();
        m.callback = r;
        return m;
    }

    private static Message getPostMessage(Runnable r, Object token) {
        Message m = Message.obtain();
        m.obj = token;
        m.callback = r; // 将 Runnable 保存到 Message 的属性中！
        return m;
    }
```
优先复用消息池中的消息！


# 5 Handler.dispatchMessage

回顾 Loop.loop() 方法，当 queue.next() 能够返回一个可以分发的 Message 后，会调用下面的逻辑，处理消息！

```java
    try {
        msg.target.dispatchMessage(msg);
    } finally {
        if (traceTag != 0) {
            Trace.traceEnd(traceTag);
        }
    }
```
这里的 msg.target 就是目标 Handler：

```java
    public void dispatchMessage(Message msg) {
        if (msg.callback != null) { // 处理 Runnbale
            handleCallback(msg);
        } else { // 处理 Message
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) { // 如果设置了 Callback，就通过 Callback 处理消息！
                    return;
                }
            }
            handleMessage(msg); // 默认使用 handler 的方法处理消息！
        }
    }
```
对于 Runable 来说，虽然其被封装为了 Message，但是由于其 what 的特殊性，无法按照一般的 Message 的去处理，

## 5.1 Handler.handleCallback

对于 Runable，通过 handleCallback 方法处理，该方法会调用 Runnable.run() 方法来执行任务！

```java
    private static void handleCallback(Message message) {
        message.callback.run();
    }
```

不多说了！

## 5.2 Callback.handleMessage

对于 Message，如果我们指定了回调接口 Callback，那就通过 Callback 处理 Message：
```java
    public interface Callback {
        public boolean handleMessage(Message msg);
    }
```
mCallback 为 Handler 的成员变量，需要显式指定！

Callback 是 Handler 的内部接口！


## 5.3 Handler.handleMessage

对于 Message，默认使用 Handler 的 handleMessage 来处理：

```java
    public void handleMessage(Message msg) {
    }
```
我们需要复写该方法，实现自己的逻辑！！


# 6 handler.removeMessages

同时 Handler 也提供了移除消息的操作；
```java
    public final void removeMessages(int what) {...}

    public final void removeMessages(int what, Object object) {...}

    public final void removeCallbacksAndMessages(Object token) {...}
```
最终调用的是 MessageQueue 的 remove 方法;

```java
mQueue.removeMessages(this, what, object);
mQueue.removeCallbacksAndMessages(this, token);
```

MessageQueue 的 remove 操作的原理很简单，根据输入的参数，进行匹配即可！！

# 7 跨进程通信

Handler 也可以用于跨进程通信：Messenger

Handler 有一个变量 IMessenger mMessenger 用于保存通过 Messenger 跨进程通信是服务端的桩对象！

```java
    final IMessenger getIMessenger() {
        synchronized (mQueue) {
            if (mMessenger != null) {
                return mMessenger;
            }
            mMessenger = new MessengerImpl(); // 创建桩对象
            return mMessenger;
        }
    }
```

Messenger 信使本质上实现了 Aidl 模板，下面是服务端桩的实现！

```java
    private final class MessengerImpl extends IMessenger.Stub {
        public void send(Message msg) {
            msg.sendingUid = Binder.getCallingUid(); // 因为会跨进程通信，所以会设置 msg.sendingUid 为当前进程的 uid
            Handler.this.sendMessage(msg);
        }
    }
```
跨进程的分析会在单独的文章中，这里不再过多的介绍！！

# 8 其他方法

- 判断消息队列中是否有指定的消息！

```java
    public final boolean hasMessages(int what, Object object) {
        return mQueue.hasMessages(this, what, object);
    }

    public final boolean hasCallbacks(Runnable r) {
        return mQueue.hasMessages(this, r, null);
    }
```