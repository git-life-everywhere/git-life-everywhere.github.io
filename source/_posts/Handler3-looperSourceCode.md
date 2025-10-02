# Handler篇 3 - Looper 源码分析
title: Handler篇 3 - Looper 源码分析
date: 2017/01/27 20:46:25
catalog: true
categories:
- AndroidFramework源码分析
- Handler线程消息机制
tags: Handler线程消息机制
copyright: true
---


基于 Android 7.1.1 源码，分析 Looper 的架构和原理。



# 1 成员变量

```java
    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
```
线程本地变量，用于保存每个线程创建出来的 Looper 对象！

```java
    private static Looper sMainLooper;
```
ui 线程的 Looper 对象！

```java
    final MessageQueue mQueue;
```
消息队列，每一个 Looper 有一个消息队列，Looper 会循环遍历该消息队列，分发消息！

```java
    final Thread mThread;
```
当前线程，也就是该 Looper 所属的线程！

```java
    private Printer mLogging;
    private long mTraceTag;
```
和调试监控相关的，我们不过多关注！



# 2 Looper.prepare

我们知道，如果要给一个指定的线程创建一个 handler，该线程必须要有一个 Looper，创建 looper 的方法就是 prepare：

```java
    public static void prepare() {
        prepare(true);
    }

    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) { // 每个线程只能有一个 Looper！
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed)); // 将创建的 Looper 加入到线程本地变量中！
    }
```
该方法会给当前的线程创建一个 Looper 对象！

注意第二个方法是**私有方法**！

## 2.1 new Looper

```java
    private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed); // 为该 looper 创建一个消息队列！
        mThread = Thread.currentThread();
    }
```
quitAllowed 表示该队列是否可以推出！

## 2.2 Looper.prepareMainLooper

对于 ui 主线程，Looper 有一个方法方法专门用于创建其对应的 Looper：
```java
    public static void prepareMainLooper() {
        prepare(false); // 调用 prepare 方法，创建 Looper
        synchronized (Looper.class) {
            if (sMainLooper != null) {
                throw new IllegalStateException("The main Looper has already been prepared.");
            }
            sMainLooper = myLooper(); // 将 ui 线程的 Looper 保存到 sMainLooper 中！
        }
    }
```
可以看到，ui 线程的 MessageQueue 是不能退出的！

### 2.2.1 Looper.myLooper

这里调用了 myLooper 方法，用于返回和当前线程相关联的 Looper 对象！

```java
    public static @Nullable Looper myLooper() {
        return sThreadLocal.get();
    }
```

方法很简单，不多说了！


# 3 Looper.loop

当 Looper 创建好后，需要调用 loop 方法，使其进入消息循环中：

```java
    public static void loop() {
        final Looper me = myLooper(); // 获得当前线程的 Looper 对象，并校验其有效性！
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue; // 获得对应的消息队列！

        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();

        for (;;) {
            //【1】通过 next 方法返回下一个要分发的 message，next 方法可能会阻塞！
            Message msg = queue.next();
            if (msg == null) {
                // 如果 msg 为 null，说明消息队列正在推出，那就 return，结束消息循环！
                return;
            }

            final Printer logging = me.mLogging;
            if (logging != null) {
                logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);
            }

            final long traceTag = me.mTraceTag;
            if (traceTag != 0 && Trace.isTagEnabled(traceTag)) {
                Trace.traceBegin(traceTag, msg.target.getTraceName(msg));
            }
            
            //【2】分发这个 message！
            try {
                msg.target.dispatchMessage(msg);
            } finally {
                if (traceTag != 0) {
                    Trace.traceEnd(traceTag);
                }
            }

            if (logging != null) {
                logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
            }

            final long newIdent = Binder.clearCallingIdentity();
            if (ident != newIdent) {
                Log.wtf(TAG, "Thread identity changed from 0x"
                        + Long.toHexString(ident) + " to 0x"
                        + Long.toHexString(newIdent) + " while dispatching to "
                        + msg.target.getClass().getName() + " "
                        + msg.callback + " what=" + msg.what);
            }

            msg.recycleUnchecked(); // 回收 Message。
        }
    }
```
其实方法流程很简单：

 - 获得当前线程的 Looper 对象，如果为 null，那就要抛出异常
 - 获得该 Looper 所有的消息队列
 - 通过 MessageQueue.next 方法，获得下一个要分发的 message！
 - 调用 msg.target.dispatchMessage 方法，分发消息！
 - 回收 Message，我们知道 MessageQueue 中的 Message 可以复用的！

注意：
-  queue.next() 方法会从消息队列中取出消息对象 Message，如果 MessageQueue 中没有任何 Message 的话，该方法将会阻塞等待新的消息。
-  当我们想要退出消息循环的时候，调用 quit 方法，那么 queue.next() 会返回一个 null 的 Message，这样就退出了！
 

# 4 Looper.quit

Looper 提供了 quit 方法，用于结束消息循环：

```java
    public void quit() {
        mQueue.quit(false);
    }

    public void quitSafely() {
        mQueue.quit(true);
    }
```
最终都调用了 MessageQueue 的 quit 方法，来结束消息循环！

quit 和 quitSafely 的区别是，quit 方法是不安全的，而 quitSafely 会在停止 Looper 的时候把当前时间点之后的已经达到处理时间点的消息处理完后才停止 Looper！



# 5 其他方法

Looper 提供了一些其他的方法：


- 获得主线程的 Looper 对象！

```java
    public static Looper getMainLooper() {
        synchronized (Looper.class) {
            return sMainLooper;
        }
    }
```

- 获得 Looper 对象所在的线程！

```java
    public @NonNull Thread getThread() {
        return mThread;
    }
```

- 获得 Looper 对象的消息队列！

```java
    public @NonNull MessageQueue getQueue() {
        return mQueue;
    }
```
- 判断 Looper 对象是否属于当前线程！
```java
    public boolean isCurrentThread() {
        return Thread.currentThread() == mThread;
    }
```

总体来看，Looper 的实现并不复杂，接下来，我们来看看 MessageQueue！
