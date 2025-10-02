# EventBus 第四篇 - 发送消息
title: EventBus 第四篇 - 发送消息
date: 2019/09/13 20:46:25 
catalog: true
categories: 

- 开源库源码分析 
- EventBus
tags: EventBus
copyright: true
---


本系列文章主要分析 EventBus 框架的架构和原理，，基于最新的 **3.1.0** 版本。

> 这是 EventBus 开源库的地址，大家可以直接访问
> https://github.com/greenrobot/EventBus

本篇文章是 EventBus 的第四篇，主要分析发送消息的流程；

# 1 回顾

我们回顾下 eventbus 的使用：

- 发送普通的消息

```java
EventBus.getDefault().post(messageEvent);
```

- 发送 sticky 消息

```java
EventBus.getDefault().postSticky(messageEvent)
```

这里我们来分析下 **post** 的流程，也是最后一篇了；

# 2 EventBus

## 2.1 post

发送普通消息：

```java
    public void post(Object event) {
        //【-->3.1】获取当前线程的 PostingThreadState 实例；
        PostingThreadState postingState = currentPostingThreadState.get();
        //【2】获取每个线程的事件队列 queue；
        List<Object> eventQueue = postingState.eventQueue;
        eventQueue.add(event);
        //【3】如果当前的状态不是正在 posting；
        if (!postingState.isPosting) {
            //【4】判断当前是否是主线程；
            postingState.isMainThread = isMainThread();
            //【5】将 posting 状态设置为 true；
            postingState.isPosting = true;
            if (postingState.canceled) {
                throw new EventBusException("Internal error. Abort state was not reset");
            }
            try {
                //【4】事件队列不为 Empty，所以要分发事件；
                while (!eventQueue.isEmpty()) {
                    //【-->2.1.1】分发单个消息；
                    postSingleEvent(eventQueue.remove(0), postingState);
                }
            } finally {
                postingState.isPosting = false;
                postingState.isMainThread = false;
            }
        }
    }
```

这段逻辑不是很复杂！！

isMainThread 方法很简单，就不多说了。。。

### 2.1.1 postSingleEvent

发送单个事件：

```java
    private void postSingleEvent(Object event, PostingThreadState postingState) throws Error {
        //【1】获取事件的 class 实例；
        Class<?> eventClass = event.getClass();
        boolean subscriptionFound = false;
        //【2】如果允许继承的话，那就要针对事件类型做处理，因为可能有继承的关系；
        if (eventInheritance) {
            //【-->2.1.1.1】查询所有的事件类型；
            List<Class<?>> eventTypes = lookupAllEventTypes(eventClass);
            int countTypes = eventTypes.size();
            for (int h = 0; h < countTypes; h++) {
                Class<?> clazz = eventTypes.get(h);
                //【-->2.1.2】开始根据每一种事件类型去分发事件（多态）；
                subscriptionFound |= postSingleEventForEventType(event, postingState, clazz);
            }
        } else {
            //【-->2.1.2】开始根据传入的事件类型去分发事件（无需继承）；
            subscriptionFound = postSingleEventForEventType(event, postingState, eventClass);
        }
        //【3】处理没有订阅者的情况；
        if (!subscriptionFound) {
            if (logNoSubscriberMessages) {
                logger.log(Level.FINE, "No subscribers registered for event " + eventClass);
            }
            if (sendNoSubscriberEvent && eventClass != NoSubscriberEvent.class &&
                    eventClass != SubscriberExceptionEvent.class) {
                post(new NoSubscriberEvent(this, event));
            }
        }
    }
```

EventBus  中有一个 eventTypesCache 的 hash：

```java
    private static final Map<Class<?>, List<Class<?>>> eventTypesCache = new HashMap<>();
```

key 是事件的 class，而 value 是一个 list，用于保存 class 和其 superClass，以及其他的所有接口；

因为如果允许事件继承的话，那么根据多态的概念，必须要收集所有的父类和接口；



#### 2.1.1.1  lookupAllEventTypes

查询所有的事件类型：

```java
    private static List<Class<?>> lookupAllEventTypes(Class<?> eventClass) {
        synchronized (eventTypesCache) {
            //【1】从 eventTypesCache 中获取事件 class 对应的事件类型列表；
            List<Class<?>> eventTypes = eventTypesCache.get(eventClass);
            if (eventTypes == null) {
                eventTypes = new ArrayList<>();
                Class<?> clazz = eventClass;
                while (clazz != null) {
                    eventTypes.add(clazz);
                    //【-->2.1.1.2】添加接口，也就是获取 class 的所有接口；
                    addInterfaces(eventTypes, clazz.getInterfaces());、
                    //【2】获取其父类，继续遍历；
                    clazz = clazz.getSuperclass();
                }
                //【3】最后放到 cache 目录中；
                eventTypesCache.put(eventClass, eventTypes);
            }
            return eventTypes;
        }
    }
```

这**部分的代码**，**主要逻辑如下**：

- 将 **eventClass** 加入到 **eventTypesCache** 的 **eventTypes list** 中；
- **向上遍历**，对于每一个 **super class**，都会将其加入到  **eventTypesCache** 的 **eventTypes list** 中；
- 对于**每个 class**，将其直接实现和间接实现的所有接口，都添加到 **eventTypesCache** 的 **eventTypes list** 中；



#### 2.1.1.2 addInterfaces

添加接口集合，就是事件类实现的所有接口：

```java
    static void addInterfaces(List<Class<?>> eventTypes, Class<?>[] interfaces) {
        //【1】遍历所有的接口，将其收集到 eventTypes 中；
        for (Class<?> interfaceClass : interfaces) {
            if (!eventTypes.contains(interfaceClass)) {
                eventTypes.add(interfaceClass);
                //【-->】处理的接口的接口；
                addInterfaces(eventTypes, interfaceClass.getInterfaces());
            }
        }
    }
```

逻辑很简单！

### 2.1.2  postSingleEventForEventType

```java
    private boolean postSingleEventForEventType(Object event, PostingThreadState postingState, Class<?> eventClass) {
        CopyOnWriteArrayList<Subscription> subscriptions;
        synchronized (this) {
            //【1】首先去查询该事件是否已经有订阅关系了，这个关系在 register 的时候会确定；
            subscriptions = subscriptionsByEventType.get(eventClass);
        }
        //【2】存在订阅关系的话，那就 post 消息；
        if (subscriptions != null && !subscriptions.isEmpty()) {
            //【2.1】处理每一个订阅关系；
            for (Subscription subscription : subscriptions) {
                postingState.event = event;
                postingState.subscription = subscription;
                boolean aborted = false;
                try {
                    //【-->2.2.1】分发事件；
                    postToSubscription(subscription, event, postingState.isMainThread);
                    //【2.2】判断是否取消事件分发；
                    aborted = postingState.canceled;
                } finally {
                    postingState.event = null;
                    postingState.subscription = null;
                    postingState.canceled = false;
                }
                if (aborted) { // 如果要取消分发，那么会跳出循环；
                    break;
                }
            }
            return true;
        }
        return false;
    }
```

方法很简单；

#### 2.2.1 postToSubscription - 线程模式处理

分发事件，根据订阅方法的线程模式启动不同的 poster；

```java
    private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
        switch (subscription.subscriberMethod.threadMode) {
            case POSTING:
                //【1】POSTING，就在事件分发的线程分发订阅；
                //【-->2.2.2】分发订阅；
                invokeSubscriber(subscription, event);
                break;
            case MAIN:
                //【2】MAIN，在主线程分发订阅，这里会判断是否已经在 main 线程，
                // 如果是的话，那就直接分发订阅，否则就通过 mainThreadPoster 分发；
                if (isMainThread) {
                    invokeSubscriber(subscription, event); //【-->2.2.2】分发订阅；
                } else {
                    mainThreadPoster.enqueue(subscription, event); //【-->4.1.2】加入队列；
                }
                break;
            case BACKGROUND:
                //【3】BACKGROUND，通过子线程分发订阅，如果当前是在 main 线程，
                // 那就通过 backgroundPoster 新起一个线程分发，如果当前是在自线程，那就当前线程分发；
                if (isMainThread) {
                    backgroundPoster.enqueue(subscription, event); //【-->4.2.2】加入队列；
                } else {
                    invokeSubscriber(subscription, event); //【-->2.2.2】分发订阅；
                }
                break;
            case ASYNC:
                //【4】ASYNC，异步分发订阅，通过 asyncPoster 每次都新起一个线程分发；
                asyncPoster.enqueue(subscription, event); //【-->4.3.2】加入队列；
                break;
            default:
                throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
        }
    }
```

这里我们看到了四种不同的线程模式，每种模式有着不同的处理！

同时也看到了一个重要的数据结构：Poster

```java
    private final Poster mainThreadPoster; // 主线程 poster，指向一个 HandlerPoster 实例；
    private final BackgroundPoster backgroundPoster; // 后台线程 poster
    private final AsyncPoster asyncPoster; // 异步 poster
```

对于 mainThreadPoster，他是在 AndroidHandlerMainThreadSupport 中创建的：

```java
        @Override
        public Poster createPoster(EventBus eventBus) {
            return new HandlerPoster(eventBus, looper, 10);
        }
```

不多说了！



### 2.2.2 invokeSubscriber 

分发订阅，也就是调用订阅者的方法处理订阅事件：

```java
    void invokeSubscriber(Subscription subscription, Object event) {
        try {
            //【1】method.invoke 反射调用；
            subscription.subscriberMethod.method.invoke(subscription.subscriber, event);
        } catch (InvocationTargetException e) {
            handleSubscriberException(subscription, event, e.getCause());
        } catch (IllegalAccessException e) {
            throw new IllegalStateException("Unexpected exception", e);
        }
    }
```

方法很简单，不多说了；

## 2.2 postSticky

发送粘性消息，这可以看到，该方法会将 **event** 保存到 **stickyEvents** 表中：

```java
    public void postSticky(Object event) {
        synchronized (stickyEvents) {
            //【1】保存到 stickyEvents 中；
            stickyEvents.put(event.getClass(), event);
        }
        //【-->2.1】发送该消息；
        post(event);
    }
```

在前面 **register** 的时候，我们有分析过在 **register** 时会立刻处理 **Sticky** 事件的分发；

这里就不再多说了；

# 3 PostingThreadState

这个类用于保存 **thread post** 的状态，在 **EventBus** 中有个 **ThreadLocal** 成员变量：

```java
    private final ThreadLocal<PostingThreadState> currentPostingThreadState = new ThreadLocal<PostingThreadState>() {
        @Override
        protected PostingThreadState initialValue() {
            //【-->3.1】创建 PostingThreadState 对象；
            return new PostingThreadState();
        }
    };
```

用于保存每一个线程的 **post** 状态！！



## 3.1 成员变量

```java
    final static class PostingThreadState {
        final List<Object> eventQueue = new ArrayList<>(); // 事件队列；
        boolean isPosting; // 线程是否正在 post 消息；
        boolean isMainThread; // post 的线程是否是主线程；
        Subscription subscription; // 订阅关系；
        Object event; // 正在 post 的事件，会从 eventQueue 按照顺序来分发；
        boolean canceled; // 是否被取消了；
    }
```

# 4 Poster

poster 用于订阅事件的最终分发，所有的 Poster 都实现了下面的接口：

```java
/**
 * Posts events.
 *
 * @author William Ferguson
 */
interface Poster {

    /**
     * Enqueue an event to be posted for a particular subscription.
     *
     * @param subscription Subscription which will receive the event.
     * @param event        Event that will be posted to subscribers.
     */
    void enqueue(Subscription subscription, Object event);
}
```

我们接着分析：

## 4.1 HandlerPoster

处理 main thread 的事件分发：

###  4.1.1 成员变量

```java
public class HandlerPoster extends Handler implements Poster {
    private final PendingPostQueue queue; // 正在分发的 post 队列；
    private final int maxMillisInsideHandleMessage;
    private final EventBus eventBus;
    private boolean handlerActive; // 是否处于激活状态；
```

参数 maxMillisInsideHandleMessage 表示处理消息的函数的执行事件，单位是毫秒，传入的是 10；

```java
---> [AndroidHandlerMainThreadSupport.java]

@Override
public Poster createPoster(EventBus eventBus) {
    return new HandlerPoster(eventBus, looper, 10);
}
```

在 AndroidHandlerMainThreadSupport 创建了一个 HandlerPoster，他会作为 EventBus 单例的成员变量；



### 4.1.2 enqueue

添加 post 到队列 中：

```java
    public void enqueue(Subscription subscription, Object event) {
        //【-->6.2】创建一个 PendingPost；
        PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);
        synchronized (this) {
            //【-->7.2】入队列：
            queue.enqueue(pendingPost);
            if (!handlerActive) {
                handlerActive = true;
                //【3】发送消息；
                if (!sendMessage(obtainMessage())) {
                    throw new EventBusException("Could not send handler message");
                }
            }
        }
    }
```

HandlerPoster 本质上是一个 handler！

### 4.1.3 run

```java
    @Override
    public void handleMessage(Message msg) {
        boolean rescheduled = false;
        try {
            long started = SystemClock.uptimeMillis();
            //【1】一个 while 循环，处理 PendingPostQueue 中所有的 post 操作；
            while (true) {
                //【-->7.3】post 出队列；
                PendingPost pendingPost = queue.poll();
                if (pendingPost == null) {
                    synchronized (this) {
                        //【-->7.3】第一次为 null，post 出队列；
                        pendingPost = queue.poll();
                        if (pendingPost == null) {
                            handlerActive = false;
                            return;
                        }
                    }
                }
                //【-->2.2.2】执行方法；
                eventBus.invokeSubscriber(pendingPost);
                long timeInMethod = SystemClock.uptimeMillis() - started;
                //【2】判断了函数的执行时间，如果大于 10 毫秒，那么说明主线程比较卡顿，
                // 这里会再次发送消息，然后立刻退出循环，这是防止 while 循环堵塞主线程；
                if (timeInMethod >= maxMillisInsideHandleMessage) {
                    if (!sendMessage(obtainMessage())) {
                        throw new EventBusException("Could not send handler message");
                    }
                    //【3】设置为 true，handlerActive 会被设置为 rescheduled
                    // 因为上面已经再次发送了消息。
                    rescheduled = true;
                    return;
                }
            }
        } finally {
            handlerActive = rescheduled;
        }
    }
```

可以看到，主线程的分发策略是：

- 尽可能一次性处理完成 **PendingPostQueue** 中的所有消息；
- 如果某个消息的处理时间超过 **10** 毫秒，说明主线程很卡，那么就会退出 **while** 循环；

## 4.2 BackgroundPoster

处理 background thread 的事件分发：

###  4.2.1 成员变量


```java
final class BackgroundPoster implements Runnable, Poster {
    private final PendingPostQueue queue; // 消息队列；
    private final EventBus eventBus;
    private volatile boolean executorRunning; // 线程池是否在运行；
```

可以看到 BackgroundPoster 是一个 Runnable；

### 4.2.2 enqueue

添加消息到 poster 中：

```java
    public void enqueue(Subscription subscription, Object event) {
        //【-->6.2】创建一个 PendingPost；
        PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);
        synchronized (this) {
            //【-->7.2】入队列：
            queue.enqueue(pendingPost);
            if (!executorRunning) { // executorRunning 设置为 true；
                executorRunning = true;
                //【-->4.2.3】执行 BackgroundPoster;
                eventBus.getExecutorService().execute(this);
            }
        }
    }
```

这个地方加了锁，这是因为 **post** 方法可以在多线程调用；





### 4.2.3 run

```java
    @Override
    public void run() {
        try {
            try {
                while (true) {
                    //【1】post 出队列，这里有个超时处理：
                    PendingPost pendingPost = queue.poll(1000);
                    if (pendingPost == null) {
                        synchronized (this) { // 这里加锁了；
                            //【2】如果为 null，那就再次检查，如果依然为 null
                            // 那就退出 run 执行，executorRunning 设置为 false；
                            pendingPost = queue.poll();
                            if (pendingPost == null) {
                                executorRunning = false;
                                return;
                            }
                        }
                    }
                    //【-->2.2.2】执行方法；
                    eventBus.invokeSubscriber(pendingPost);
                }
            } catch (InterruptedException e) {
                eventBus.getLogger().log(Level.WARNING, Thread.currentThread().getName() + " was interruppted", e);
            }
        } finally {
            executorRunning = false;
        }
    }
```

可以看到，这个线程因为 while (true) 一直处于 runnable/running 的状态；

## 4.3 AsyncPoster

处理 async thread 的事件分发：

###  4.3.1 成员变量

```java
class AsyncPoster implements Runnable, Poster {
    private final PendingPostQueue queue; // 队列；
    private final EventBus eventBus;
```

### 4.3.2 enqueue

添加消息到 poster 中：

```java
    public void enqueue(Subscription subscription, Object event) {
        //【-->6.2】创建一个 PendingPost；
        PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);
        //【-->7.2】入队列：
        queue.enqueue(pendingPost);
        eventBus.getExecutorService().execute(this);
    }
```

这个人地方竟然没有加锁，奇怪啊～

### 4.3.3 run

```java
    @Override
    public void run() {
        //【-->7.3】post 出队列；
        PendingPost pendingPost = queue.poll();
        if(pendingPost == null) {
            throw new IllegalStateException("No pending post available");
        }
        //【-->2.2.2】执行方法；
        eventBus.invokeSubscriber(pendingPost);
    }
```

# 6 PendingPost

表示一个正在分发的 post。



## 6.1 成员变量

```java
final class PendingPost {
    // 缓存 post，用于复用；
    private final static List<PendingPost> pendingPostPool = new ArrayList<PendingPost>(); 
    Object event; // 要分发的事件；
    Subscription subscription; // 订阅关系；
    PendingPost next; // 下一个要分发的 post，用户构成链表结构；！
```



## 6.2 obtainPendingPost

获取一个 **PendingPost**：

```java
    static PendingPost obtainPendingPost(Subscription subscription, Object event) {
        synchronized (pendingPostPool) {
            int size = pendingPostPool.size();
            //【1】优先从 pendingPostPool 中获取；
            if (size > 0) {
                PendingPost pendingPost = pendingPostPool.remove(size - 1);
                pendingPost.event = event;
                pendingPost.subscription = subscription;
                pendingPost.next = null;
                return pendingPost;
            }
        }
        //【2】没有的话，再创建新的；
        return new PendingPost(event, subscription);
    }
```

这里有加锁的！



## 6.3 releasePendingPost

消息 **post** 完成后，会缓存 **post**：

```java
    static void releasePendingPost(PendingPost pendingPost) {
        pendingPost.event = null;
        pendingPost.subscription = null;
        pendingPost.next = null;
        synchronized (pendingPostPool) {
            //【1】缓存已经 post 的消息的 PendingPost！！！
            if (pendingPostPool.size() < 10000) {
                pendingPostPool.add(pendingPost);
            }
        }
    }
```

可以看到：pendingPostPool 不会超过 10000 个；

# 7 PendingPostQueue

这是一个由链表构成的 正在分发的 post 的队列！

## 7.1 成员变量

```java
final class PendingPostQueue {
    private PendingPost head; // 队列头；
    private PendingPost tail; // 队列尾；
```

内部有队列头和队列尾两个属性；

这个方法的 **enqueue** 和 **poll** 是加锁的～

## 7.2 enqueue

将 PendingPost 放入到队列中，默认是加入到队尾，该方法是加锁了：

```java
    synchronized void enqueue(PendingPost pendingPost) {
        if (pendingPost == null) {
            throw new NullPointerException("null cannot be enqueued");
        }
        //【1】入队列；
        if (tail != null) {
            tail.next = pendingPost;
            tail = pendingPost;
        } else if (head == null) {
            head = tail = pendingPost;
        } else {
            throw new IllegalStateException("Head present, but no tail");
        }
        //【2】提醒其他阻塞的线程；
        notifyAll();
    }
```



## 7.3 poll

PendingPost 出队列：

```java
    synchronized PendingPost poll() {
        //【1】从 head 出队列，更改指针；
        PendingPost pendingPost = head;
        if (head != null) {
            head = head.next;
            if (head == null) {
                tail = null;
            }
        }
        return pendingPost;
    }

    synchronized PendingPost poll(int maxMillisToWait) throws InterruptedException {
        if (head == null) {
            //【2】这个方法会在队列为 null 的时候，等待一会儿；
            wait(maxMillisToWait);
        }
        return poll();
    }
```

# 8 总结

到这里，EventBus 就整完了，驾鹤西去呦～～