# Handler篇 1 - Handler 初识
title: Handler篇 1 - Handler 初识
date: 2017/01/13 20:46:25
catalog: true
categories:
- AndroidFramework源码分析
- Handler线程消息机制
tags: Handler线程消息机制
copyright: true
---

基于 Android 7.1.1 源码，分析 handler 的架构和原理。


# 0 前言


Android 中有 2 中常见的通信方式，进程间的通信使用 Binder，而线程间的通信则使用 Handler，该系列文章就来总结下和 Handler 相关的知识点！


# 1 创建 Handler 

我们知道，每一个 Handler 都要和一个 Thread 的 Looper 对象相关联，一个线程可以有多个 Handler，下面来看看  Handler 的创建！

## 1.1 主线程的 Handler 


通常，我们可以这样创建一个 Handler：

```java
Handler handler = new Handler();
```
这种创建方式，默认会将 Handler 和当前线程的 Looper 对象相关联，这个我们后续分析！

当然也可以显式的传入 Looper 对象！

```java
Handler handler = new Handler(Looper.myLooper());
Handler handler = new Handler(Looper.getMainLooper());
```

这里涉及到几个 Looper 的几个方法：

```java
Looper.myLooper() // 当前线程的 Looper 对象！
Looper.getMainLooper() // 主线程的 Looper 对象！
```
对于 Looper 我们后续分析；

## 1.2 子线程的 Handler 

在 Ui 线程中直接创建 Handler 是没有问题的，因为 Ui 线程默认会创建 Looper 对象，对于子线程，默认是不会有 Looper 对象，直接创建是会报错的：

```java
E AndroidRuntime: FATAL EXCEPTION: AsyncTask #1
E AndroidRuntime: Process: com.coolqi.papapa:ui, PID: 27199
E AndroidRuntime: java.lang.RuntimeException: An error occurred while executing doInBackground()
E AndroidRuntime:        at android.os.AsyncTask$3.done(AsyncTask.java:353)
E AndroidRuntime:        at java.util.concurrent.FutureTask.finishCompletion(FutureTask.java:383)
E AndroidRuntime:        at java.util.concurrent.FutureTask.setException(FutureTask.java:252)
E AndroidRuntime:        at java.util.concurrent.FutureTask.run(FutureTask.java:271)
E AndroidRuntime:        at android.os.AsyncTask$SerialExecutor$1.run(AsyncTask.java:245)
E AndroidRuntime:        at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1162)
E AndroidRuntime:        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:636)
E AndroidRuntime:        at java.lang.Thread.run(Thread.java:764)
E AndroidRuntime: Caused by: java.lang.RuntimeException: Can't create handler inside thread that has not called Looper.prepare()
E AndroidRuntime:        at android.os.Handler.<init>(Handler.java:204)
E AndroidRuntime:        at android.os.Handler.<init>(Handler.java:118)
E AndroidRuntime:        at com.coolqi.papapa.f.doInBackground(EntryActivity.java:2368)
E AndroidRuntime:        at android.os.AsyncTask$2.call(AsyncTask.java:333)
E AndroidRuntime:        at java.util.concurrent.FutureTask.run(FutureTask.java:266)
E AndroidRuntime:        ... 4 more
```

我们必须在创建之前，显式的调用 Looper.prepare() 方法，为了能使 Handler 正常工作，创建完 Handler 后要 Looper.loop() 启动消息循环！

```java
        new Thread(new Runnable() {
            @Override
            public void run() {
            Looper.prepare();
            new Handler(Looper.myLooper());
            Looper.loop();
            }
        }).start();
```

那么，我们可以不用调用 Looper.prepare() 吗？

每次都调用，很麻烦，实际上是可以的，系统已经给我们内置了一个 HandlerThread，HandlerThread 和 Ui 线程一样，也会默认创建一个 Looper 对象！

```java
HandlerThread mWorkThread =
            new HandlerThread("handler", android.os.Process.THREAD_PRIORITY_BACKGROUND);
mWorkThread.start();
Handler mHandler = new Handler(mWorkThread.getLooper());
```

到这里，关于 Handler 的创建，就讲这么多！

# 2 使用 Handler 

Handler 的主要用途有 2 个：

- 进行线程间通信，能在其他线程中执行指定操作，比如一些异步，或者耗时的操作！
- 能够指定在某个时间点执行一些任务！

使用的方式很简单：

- send message
- post runnable

Handler 本质上是作为线程的消息队列的管理者，不管是发送消息，还是执行任务，都其实是将其添加到队列中依次处理！

## 2.1 send messsage

Handler 和消息相关的方法有很多，这我们只看一些常见的：

- **发送消息**：

```java
        int MSG_FIRST = 1001;
        handler.sendEmptyMessage(MSG_FIRST);
```
这是发送一个空消息！

- **延迟发送消息**：

我们也可以延迟发送消息：
```java
        int MSG_FIRST = 1001;
        handler.sendMessageDelayed(MSG_FIRST, 1000);
```

- **指定某个时刻发送消息**：
```java
        int MSG_FIRST = 1001;
        handler.sendEmptyMessageAtTime(MSG_FIRST， System.currentTimeMillis() + 60 * 60 * 1000);
```

- **发送非空消息**：
```java
        int MSG_FIRST = 1001;
        message.what = MSG_FIRST;
        Message message = new Message();
        handler.sendMessage(message)
        // handler.sendMessageDelayed(message, 2000)
```
用法和发送空消息很类似，可以立刻发送，也可以延迟发送！

对于非空的消息 Message，我们可以携带一些数据：

```java
        bundle.putFloat("cat", 0.11f);
        message.setData(bundle);
```
对于 Bundle，这里就不多说了，以上是基本的用法！

## 2.2 post runnable

post 操作其实本质上是将 Runnable 转化为了 message 和 callback，并不会指定消息的 what 属性，因为他的处理方式不同！

- **执行任务**：

```java
        handler.post(new Runnable() {
            @Override
            public void run() {
                // do something！
            }
        });
```

- **延迟执行任务**：
```java
        handler.postDelayed(...);
```

- **延迟执行任务**：
```java
        handler.postDelayed(Runnable r, long delayMillis)
```


## 2.3 handle message/runnable

接下来，看看如何处理 Message 和 runnable：

处理 message，需要我们实现 handleMessage，在内部来根据 msg.what 来分别对 msg 进行处理！

```java
    public void handleMessage(Message msg) {
    }
```

处理 runnable 则不同，runnable 不需要我们像 message 那样处理，系统会自动调用下面的方法来帮我们执行 runnable：

```java
    private static void handleCallback(Message message) {
        message.callback.run();
    }
```

# 3 线程间通信的实现

实现线程间通信的方法很简单，当前线程持有其他线程的 Handler，然后向其发送 message 即可！


关于 Handler 的简单用法，到这里就分析结束了，后面会继续分析其源码的实现！