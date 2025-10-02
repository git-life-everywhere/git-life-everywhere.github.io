# EventBus 第三篇 - 初始化、注册和取消注册
title: EventBus 第三篇 - 初始化、注册和取消注册
date: 2019/09/10 20:46:25 
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

本篇文章是 EventBus 的第三篇，主要分析 初始化，注册和取消注册；

Eventbus 翻译过来就是事件总线，用于简化组件和组件，线程和线程之间的消息通信，可以看成是 Handler + Thread 的替代品。

# 1 回顾

我们在使用的过程中，需要先进行注册：

```java
EventBus.getDefault().register(this);
```

当我们的组件在销毁以后，就要执行取消注册：

```java
EventBus.getDefault().unregister(this);
```

本篇文章，主要分析 register 和 unregister 的流程！

# 2 EventBus

EventBus 这个类是总的入口，我们可以通过 getDefault 返回默认配置的 EventBus，也可以通过 EventBusBuilder 去自定义配置：

EventBus 使用了单例模式！

## 2.1 成员属性

我们先去看看 EventBus 的成员属性，当然我们后面也会详细分析：



- **核心的变量：**

```java
// 保存了
private static final Map<Class<?>, List<Class<?>>> eventTypesCache = new HashMap<>();
// 保存了 [订阅事件 class 实例 --> 该事件的订阅关系的 list] 的映射关系；
private final Map<Class<?>, CopyOnWriteArrayList<Subscription>> subscriptionsByEventType;
// 保存了 [订阅者实例 --> 订阅事件 class 的 list] 的映射关系；
private final Map<Object, List<Class<?>>> typesBySubscriber;
// 保存了 [粘性订阅事件 class 实例 --> 粘性订阅事件实例] 的映射关系，只要是已经发送过的 sticky event 都会被加入这里；
private final Map<Class<?>, Object> stickyEvents;

// 用于保存每个 post 线程的状态；
private final ThreadLocal<PostingThreadState> currentPostingThreadState = new ThreadLocal<PostingThreadState>() {
  @Override
  protected PostingThreadState initialValue() {
    return new PostingThreadState();
  }
};

//【-->4】保存主线程的支持类，对 Looper 的封装；@Nullable
private final MainThreadSupport mainThreadSupport; 

//【-->4】用于分发消息；@Nullable
private final Poster mainThreadPoster;  // 用于主线程的消息分发处理；
private final BackgroundPoster backgroundPoster; // 用于后台线程的消息分发处理；
private final AsyncPoster asyncPoster; // 用于异步线程的消息分发处理；

//【-->5】用于查找订阅方法；
private final SubscriberMethodFinder subscriberMethodFinder;
```

- **其他的变量：**

```java
static volatile EventBus defaultInstance; //【1】EventBus 的单例对象；
private static final EventBusBuilder DEFAULT_BUILDER = new EventBusBuilder(); //【-->3.1】默认配置的 EventBusBuilder

private final ExecutorService executorService; // 线程池，用于分发异步和后台的消息

private final boolean throwSubscriberException; // 这些变量请参考 EventBusBuidler，这里不再多说！
private final boolean logSubscriberExceptions;
private final boolean logNoSubscriberMessages;
private final boolean sendSubscriberExceptionEvent;
private final boolean sendNoSubscriberEvent;
private final boolean eventInheritance;

private final int indexCount; // subscriberInfoIndex 实例的个数；
```

这里我简单的解释下：

- mainThreadPoster，backgroundPoster，asyncPoster 用于不同类型消息的分发，它们都是实现了 Post 接口，后面我们分析的时候再看！
- currentPostingThreadState 是一个 ThreadLocal 变量，每个 post 线程都会有一个 PostingThreadState 属性，表示 post 的状态！
- SubscriberMethodFinder 用于查找订阅方法；    



## 2.2 创建 EventBus 实例

EventBus 提供了多种创建方式，既可以通过单例模式创建一个统一的 EventBus 对象，也可以创建多个 EventBus 实例，每个实例都是一个单独的作用域！

### 2.2.1 getDefault

单例模式方法，创建默认的 EventBus：

```java
    public static EventBus getDefault() {
        if (defaultInstance == null) {
            synchronized (EventBus.class) {
                if (defaultInstance == null) {
                    //【-->2.2.2】构造器；
                    defaultInstance = new EventBus();
                }
            }
        }
        return defaultInstance;
    }
```

### 2.2.2 new EventBus

非单例模式方法，两个构造器，默认构造器会传入一个默认的 `DEFAULT_BUILDER`，另一个需要传入指定的 EventBusBuilder 实例；

但是，我们只能通过无参数的构造器创建非单例的  EventBus 实例！

```java
public class EventBus {
    ... ... ...
      
		public EventBus() {
        //【1】默认的构造器是通过默认的 builder 对象处理的；【-->2.1】默认的 buidler 实例；
        this(DEFAULT_BUILDER); 
    }

    //【2】通过建造者模式来初始化，注意这个构造器是 protected 的，我们无法访问！
    EventBus(EventBusBuilder builder) { 
        //【-->3.2.3】通过 EventBusBuilder 初始化 log 系统；
        logger = builder.getLogger();
        subscriptionsByEventType = new HashMap<>();
        typesBySubscriber = new HashMap<>();
        stickyEvents = new ConcurrentHashMap<>();
      
        //【-->3.2.1】通过 EventBusBuilder 初始化 mainThread 相关变量；
        mainThreadSupport = builder.getMainThreadSupport();
      
        mainThreadPoster = mainThreadSupport != null ? mainThreadSupport.createPoster(this) : null;
        backgroundPoster = new BackgroundPoster(this);
        asyncPoster = new AsyncPoster(this);
        
        //【-->3.1】通过 EventBusBuilder 初始化 EventBus 内部的变量：
        indexCount = builder.subscriberInfoIndexes != null ? builder.subscriberInfoIndexes.size() : 0;
        subscriberMethodFinder = new SubscriberMethodFinder(builder.subscriberInfoIndexes,
                builder.strictMethodVerification, builder.ignoreGeneratedIndex); //【-->5.2】创建 SubscriberMethodFinder 对象；
        logSubscriberExceptions = builder.logSubscriberExceptions;
        logNoSubscriberMessages = builder.logNoSubscriberMessages;
        sendSubscriberExceptionEvent = builder.sendSubscriberExceptionEvent;
        sendNoSubscriberEvent = builder.sendNoSubscriberEvent;
        throwSubscriberException = builder.throwSubscriberException;
        eventInheritance = builder.eventInheritance;
        executorService = builder.executorService;
    }
    ... ... ... 
}
```

通过建造者模式来初始化部分变量，同时也会对其他变量做默认的初始化；

### 2.2.3 通过 EventBusBuilder 创建

EventBusBuilder 也提供了两个方法，通过 build 模式创建 EventBus 实例！

#### 2.2.3.1 installDefaultEventBus

单例模式方法，创建默认的 EventBus，但是如果已经创建了 EventBus 的单例，那就不能调用这个方法：

```java
    public EventBus installDefaultEventBus() {
        synchronized (EventBus.class) {
            if (EventBus.defaultInstance != null) {
                throw new EventBusException("Default instance already exists." +
                        " It may be only set once before it's used the first time to ensure consistent behavior.");
            }
            //【-->2.2.3.2】通过 build 方法创建单例！
            EventBus.defaultInstance = build();
            return EventBus.defaultInstance;
        }
    }
```

#### 2.2.3.2 build

这个方法可以用与创建非单例的 EventBus 实例：

```java
    public EventBus build() {
        //【-->2.2.2】调用一参数构造器；
        return new EventBus(this);
    }
```



## 2.3 register - 注册

我们来看 register  的过程：

```java
    public void register(Object subscriber) {
        //【1】获取订阅者的 class 对象；
        Class<?> subscriberClass = subscriber.getClass();
        //【-->5.3】查询订阅者的方法；
        List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
        synchronized (this) {
            for (SubscriberMethod subscriberMethod : subscriberMethods) {
                //【-->2.3.1】建立订阅关系；
                subscribe(subscriber, subscriberMethod);
            }
        }
    }
```

整体流程简单，不多说了；

注意：这里的 subscriberClass 是调用 register 方法所在的类；

### 2.3.1 subscribe

创建订阅关系：

```java
    private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
        //【1】获取事件类型 eventType
        Class<?> eventType = subscriberMethod.eventType;
        //【-->7.1】创建订阅关系；
        Subscription newSubscription = new Subscription(subscriber, subscriberMethod);
        //【2】获取该 eventType 对应的订阅关系的 list；
        CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
        if (subscriptions == null) {
            subscriptions = new CopyOnWriteArrayList<>();
            subscriptionsByEventType.put(eventType, subscriptions); // 初始化操作；
        } else {
            if (subscriptions.contains(newSubscription)) {
                throw new EventBusException("Subscriber " + subscriber.getClass() + " already registered to event "
                        + eventType);
            }
        }
        //【3】调整优先级顺序，订阅关系的 list 是以 priority 从小到大排序的；
        // 将订阅关系加入到 subscriptionsByEventType 中；
        int size = subscriptions.size();
        for (int i = 0; i <= size; i++) {
            if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
                subscriptions.add(i, newSubscription);
                break;
            }
        }
        
        //【4】将订阅者实例和要订阅的 eventType 的 class 实例保存到 subscribedEvents 中；
        List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
        if (subscribedEvents == null) {
            subscribedEvents = new ArrayList<>();
            typesBySubscriber.put(subscriber, subscribedEvents); // 初始化；
        }
        subscribedEvents.add(eventType);

        //【5】方法订阅的事件是 sticky 的，特殊处理；；
        if (subscriberMethod.sticky) {
            //【5.1】如果允许事件继承，默认是为 true 的，可以去看 EventBusBuilder；
            if (eventInheritance) {
                //【5.2】获取已经存在的 sticky 事件；
                Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
                for (Map.Entry<Class<?>, Object> entry : entries) {
                    Class<?> candidateEventType = entry.getKey();
                    //【5.3】因为可能一个父类有多个子类，所以这里要处理所有的 sticky event。
                    if (eventType.isAssignableFrom(candidateEventType)) {
                        Object stickyEvent = entry.getValue();
                        //【-->2.3.2】分发 sticky Event
                        checkPostStickyEventToSubscription(newSubscription, stickyEvent);
                    }
                }
            } else {
                //【5.4】如果不允许事件继承，那就只能找对应类型的 sticky event。
                Object stickyEvent = stickyEvents.get(eventType);
                //【-->2.3.2】分发 sticky Event
                checkPostStickyEventToSubscription(newSubscription, stickyEvent);
            }
        }
    }
```

流程：

- 将订阅关系保存到对应的缓存中；
- 处理 sticky 事件的分发；

这里的 eventInheritance 是啥意思呢，其实就是事件继承关系：

比如 MessageEvent2 继承了 MessageEvent，那么如果订阅方法的参数是 MessageEvent，而粘性事件是 MessageEvent2，那么我们依然可以分发该消息；

### 2.3.2 checkPostStickyEventToSubscription

```java
    private void checkPostStickyEventToSubscription(Subscription newSubscription, Object stickyEvent) {
        if (stickyEvent != null) {
            //【-->2.3.3】分发 event；
            postToSubscription(newSubscription, stickyEvent, isMainThread());
        }
    }
```

// If the subscriber is trying to abort the event, it will fail (event is not tracked in posting state)
--> Strange corner case, which we don't take care of here.

### 2.3.3 postToSubscription

对于 sticky event 这个和粘性广播的道理是一样，如果它之前就已经分发过，那么他会被存储在系统里，下一个订阅者一旦注册，那就能够收到：

```java
    private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
        switch (subscription.subscriberMethod.threadMode) {
            case POSTING:
                invokeSubscriber(subscription, event);
                break;
            case MAIN:
                if (isMainThread) {
                    invokeSubscriber(subscription, event);
                } else {
                    mainThreadPoster.enqueue(subscription, event);
                }
                break;
            case MAIN_ORDERED:
                if (mainThreadPoster != null) {
                    mainThreadPoster.enqueue(subscription, event);
                } else {
                    // temporary: technically not correct as poster not decoupled from subscriber
                    invokeSubscriber(subscription, event);
                }
                break;
            case BACKGROUND:
                if (isMainThread) {
                    backgroundPoster.enqueue(subscription, event);
                } else {
                    invokeSubscriber(subscription, event);
                }
                break;
            case ASYNC:
                asyncPoster.enqueue(subscription, event);
                break;
            default:
                throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
        }
    }
```

可以看到，这里开始事件的 post 了！

event post 这里我们不关注，后面会分析；

## 2.4 unregister - 反注册

我们来看看反注册的过程：

```java
    public synchronized void unregister(Object subscriber) {
        List<Class<?>> subscribedTypes = typesBySubscriber.get(subscriber);
        if (subscribedTypes != null) {
            for (Class<?> eventType : subscribedTypes) {
                //【-->2.4.1】解除订阅；
                unsubscribeByEventType(subscriber, eventType);
            }
            typesBySubscriber.remove(subscriber);
        } else {
            logger.log(Level.WARNING, "Subscriber to unregister was not registered before: " + subscriber.getClass());
        }
    }
```

### 2.4.1 unsubscribeByEventType

这个方法很简单，就是将订阅关系移除掉；

```java
    private void unsubscribeByEventType(Object subscriber, Class<?> eventType) {
        //【1】从 subscriptionsByEventType 返回 eventType 的所有订阅关系；
        List<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
        if (subscriptions != null) {
            int size = subscriptions.size();
            for (int i = 0; i < size; i++) {
                //【2】处理当前订阅者的订阅关系，设置为 no active，同时从集合中移除；
                Subscription subscription = subscriptions.get(i);
                if (subscription.subscriber == subscriber) {
                    subscription.active = false;
                    subscriptions.remove(i);
                    i--;
                    size--;
                }
            }
        }
    }
```

不多说了。

# 3 EventBusBuilder

建造者模式，用于自定义 EventBus 的配置，并创建 EventBus 实例：

## 3.1 成员变量

我们来看下成员变量：

```java
public class EventBusBuilder {
    // 内置的默认线程池
    private final static ExecutorService DEFAULT_EXECUTOR_SERVICE = Executors.newCachedThreadPool();

    boolean logSubscriberExceptions = true; // 是否记录订阅者的异常信息；默认为 true；
    boolean logNoSubscriberMessages = true; // 
    boolean sendSubscriberExceptionEvent = true; // 是否发送订阅异常的事件；默认为 true；
    boolean sendNoSubscriberEvent = true; // 是否发送没有订阅者的消息；默认为 true；
    boolean throwSubscriberException; // 是否抛出订阅异常，用于 debug；默认为 false；
    boolean eventInheritance = true; // 是否开启事件继承机制，默认为 true；
    boolean ignoreGeneratedIndex; // 是否强制使用反射，即使开启了 Apt 特性；默认为 false；
    boolean strictMethodVerification; // 是否强制方法校验；默认为 false；

    ExecutorService executorService = DEFAULT_EXECUTOR_SERVICE; // 线程池，用于分发异步和后台的消息；默认为 DEFAULT_EXECUTOR_SERVICE
    List<Class<?>> skipMethodVerificationForClasses; // 用于保存哪些跳过方法校验的 class
    List<SubscriberInfoIndex> subscriberInfoIndexes; // 用于保存 SubscriberInfoIndex 实例，也就是 APT 技术生成的动态 java 类；
    
    Logger logger; // log 系统；
    MainThreadSupport mainThreadSupport; // 用于提供对主线程的支持，指向 AndroidHandlerMainThreadSupport 实例；

    EventBusBuilder() {
    }
```

如果我们使用默认的 EventBusBuilder 来初始化 EventBus 的话，那么 EventBusBuilder 的方法会返回属性的默认值：

这里我简单的说下：

- 阿道夫
- 安抚
- 党法



## 3.2 方法

前面我们顺便看了下通过 EventBusBuilder  创建 EventBus 的相关方法，这里就不再看了，我们来看下 EventBusBuilder 其中的部分方法：

### 3.2.1 getMainThreadSupport

获取主线程的支持对象！

```java
    MainThreadSupport getMainThreadSupport() {
        if (mainThreadSupport != null) {
            return mainThreadSupport;
        } else if (AndroidLogger.isAndroidLogAvailable()) {
            //【-->3.2.1.1】获取主线程的 looper 对象；
            Object looperOrNull = getAndroidMainLooperOrNull();
            return looperOrNull == null ? null :
              			//【-->4】创建 AndroidHandlerMainThreadSupport 实例对象；
                    new MainThreadSupport.AndroidHandlerMainThreadSupport((Looper) looperOrNull);
        } else {
            return null;
        }
    }
```

#### 3.2.1.1 getAndroidMainLooperOrNull

获取主线程的 looper 对象；

```java
    Object getAndroidMainLooperOrNull() {
        try {
            return Looper.getMainLooper();
        } catch (RuntimeException e) {
            return null;
        }
    }
```

不多说了。

### 3.2.2 addIndex

改方法用于将通过 EventBus 的 annotation preprocessor 生成的 SubscriberInfoIndex 子类实例，加入到 EventBusBuilder.subscriberInfoIndexes 中！

```java
    public EventBusBuilder addIndex(SubscriberInfoIndex index) {
        if (subscriberInfoIndexes == null) {
            subscriberInfoIndexes = new ArrayList<>();
        }
        subscriberInfoIndexes.add(index);
        return this;
    }
```

不多说了。

### 3.2.3 getLogger

获取 log 系统：

```java
Logger getLogger() {
    if (logger != null) {
        return logger;
    } else {
        // also check main looper to see if we have "good" Android classes (not Stubs etc.)
        return AndroidLogger.isAndroidLogAvailable() && getAndroidMainLooperOrNull() != null
                ? new AndroidLogger("EventBus") :
                new Logger.SystemOutLogger();
    }
}
```

# 4 MainThreadSupport

是一个接口，AndroidHandlerMainThreadSupport 内部类实现了该接口，作为主线程的支持类，是对 looper 对象的封装；

```java
public interface MainThreadSupport {

    boolean isMainThread();

    Poster createPoster(EventBus eventBus);

    class AndroidHandlerMainThreadSupport implements MainThreadSupport {
        //【1】主线程的 looper 对象；
        private final Looper looper;

        public AndroidHandlerMainThreadSupport(Looper looper) {
            this.looper = looper;
        }
        ... ... ...
    }
}
```

这里我们先不看 AndroidHandlerMainThreadSupport 的其他方法，后面会分析。

暂时只需要知道，AndroidHandlerMainThreadSupport 的 createPoster 方法会创建一个 **HandlerPoster** 实例，他是 Handler 的子类，同时实现了 Poster 接口！

看到这里，其实能猜到，HandlerPoster 会持有主线程的 Looper 对象，像主线程发送消息！！



# 5 SubscriberMethodFinder

该类用于查找订阅者的方法：

## 5.1 成员变量

我们来看一些核心的成员变量：

```java
private static final int BRIDGE = 0x40;
private static final int SYNTHETIC = 0x1000;
// 方法校验位。
private static final int MODIFIERS_IGNORE = Modifier.ABSTRACT | Modifier.STATIC | BRIDGE | SYNTHETIC;
// 方法缓存，key 为订阅者的 class 实例，value 为订阅方法的 list；
private static final Map<Class<?>, List<SubscriberMethod>> METHOD_CACHE = new ConcurrentHashMap<>(); 

// 以下三个变量的值来自 EventBusBuilder，意思我已经解释过了；
private List<SubscriberInfoIndex> subscriberInfoIndexes;
private final boolean strictMethodVerification;
private final boolean ignoreGeneratedIndex;

// 用于保存 FindState 对象，每一个 FindState 用于记录查询订阅方法的结果和状态；
private static final int POOL_SIZE = 4;
private static final FindState[] FIND_STATE_POOL = new FindState[POOL_SIZE];
```

这里可以看到，subscriberInfoIndexes 是要手动设置到 EventBusBuilder 中；

## 5.2 new SubscriberMethodFinder

- 参数 **List<SubscriberInfoIndex> subscriberInfoIndexes**：表示 SubscriberInfoIndex 集合，就是通过 APT 技术生成的，存储了订阅方法的对象；
- 参数 **boolean strictMethodVerification**：是否强制方法校验，默认为 false；
- 参数 **boolean ignoreGeneratedIndex**：是否强制使用反射，即使开启了 APT 特性；默认为 false；

以上参数值均来自于 EventBusBuilder 中的默认值/自定义值；

```java
SubscriberMethodFinder(List<SubscriberInfoIndex> subscriberInfoIndexes, boolean strictMethodVerification,
                       boolean ignoreGeneratedIndex) {
    this.subscriberInfoIndexes = subscriberInfoIndexes;
    this.strictMethodVerification = strictMethodVerification;
    this.ignoreGeneratedIndex = ignoreGeneratedIndex;
}
```



## 5.3 findSubscriberMethods

查询订阅者拥有的订阅方法：

```java
    List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
        //【1】默认从方法 cache 中查询；
        List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass);
        if (subscriberMethods != null) {
            return subscriberMethods;
        }
        //【2】如果从方法 cache中查询不到，那就判断是否不使用 GeneratedIndex；
        if (ignoreGeneratedIndex) {
            //【-->5.3.1】不使用的话，就通过反射的方式访问方法；
            subscriberMethods = findUsingReflection(subscriberClass);
        } else {
            //【-->5.3.2】使用的话，就通过 GeneratedIndex 获取方法；
            subscriberMethods = findUsingInfo(subscriberClass);
        }
        if (subscriberMethods.isEmpty()) {
            throw new EventBusException("Subscriber " + subscriberClass
                    + " and its super classes have no public methods with the @Subscribe annotation");
        } else {
            //【3】加入到方法 cache 中去；
            METHOD_CACHE.put(subscriberClass, subscriberMethods);
            return subscriberMethods;
        }
    }
```

ignoreGeneratedIndex 默认是 false 的；

核心代码在 **findUsingReflection** 和 **findUsingInfo** 中！



### 5.3.1 findUsingReflection - 反射获取

我们来看下通过反射是如何获取的：

```java
    private List<SubscriberMethod> findUsingReflection(Class<?> subscriberClass) {
        //【-->5.4】返回一个 FindState 对象，用于记录查询的结果和状态；
        FindState findState = prepareFindState();
        //【-->6.2】初始化 FindState；
        findState.initForSubscriber(subscriberClass);
        while (findState.clazz != null) {
            //【-->5.5】通过发射的方式收集注解方法；
            findUsingReflectionInSingleClass(findState);
            //【-->6.3】处理其 superClass；
            findState.moveToSuperclass();
        }
        //【-->5.6】返回所有的订阅方法；
        return getMethodsAndRelease(findState);
    }
```

这里的 subscriberClass 是调用 register 方法所在的类，所以找父类肯定是向上搜索；

对于每一个调用了 register 的 class，都会创建一个 FindState 对象，保存相关信息；

### 5.3.2 findUsingInfo - APT 获取

开启了 APT 预处理技术的话，那就通过动态生成的类获取，这个过程比反射更快；

```java
    private List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {
        //【-->5.4】返回一个 FindState 对象，用于记录查询的结果和状态；
        FindState findState = prepareFindState();
        //【-->6.2】初始化 FindState；
        findState.initForSubscriber(subscriberClass);
        while (findState.clazz != null) {
            //【-->5.3.2.1】返回订阅者对应的 SubscriberInfo 实例，保存到 findState.subscriberInfo 中；
            findState.subscriberInfo = getSubscriberInfo(findState);
            if (findState.subscriberInfo != null) {
                //【1】返回 SubscriberInfo 的所有订阅方法 SubscriberMethod[]；
                SubscriberMethod[] array = findState.subscriberInfo.getSubscriberMethods();
                for (SubscriberMethod subscriberMethod : array) {
                    //【-->6.4】对方法做检查，和反射调用一个吊样；
                    if (findState.checkAdd(subscriberMethod.method, subscriberMethod.eventType)) {
                        //【2】检查没啥问题，就加入到 findState.subscriberMethods 中；
                        findState.subscriberMethods.add(subscriberMethod);
                    }
                }
            } else {
                //【-->5.5】如果订阅者没有对应的 SubscriberInfo 实例，通过发射的方式收集注解方法；
                findUsingReflectionInSingleClass(findState);
            }
            //【-->6.3】处理其 superClass；
            findState.moveToSuperclass();
        }
        //【-->5.6】返回所有的订阅方法；
        return getMethodsAndRelease(findState);
    }
```

可以看到，这里会优先获取通过 APT 技术生成的类，如果没有对应的 SubscriberInfo，那就仍然通过反射来获取方法 Method；

这里的 subscriberClass 是调用 register 方法所在的类，所以找父类肯定是向上搜索；

#### 5.3.2.1 getSubscriberInfo

返回订阅者对应的 SubscriberInfo 实例，前面我们知道 findState.subscriberInfo  在初始化的时候是 null 的：

```java
    private SubscriberInfo getSubscriberInfo(FindState findState) {
        //【1】这里是针对于继承关系的，因为是先处理子类，再处理父类，所以先处理的子类的话，findState.subscriberInfo 肯定不是 null
        // 那么就要通过 subscriberInfo.getSuperSubscriberInfo() 获取父类的 SubscriberInfo。
        if (findState.subscriberInfo != null && findState.subscriberInfo.getSuperSubscriberInfo() != null) {
            SubscriberInfo superclassInfo = findState.subscriberInfo.getSuperSubscriberInfo();
            //【2】额外还要做一次 class 判断，因为 while 循环会调整 clazz；
            if (findState.clazz == superclassInfo.getSubscriberClass()) {
                return superclassInfo;
            }
        }
        //【2】通过 subscriberInfoIndexes 来查找，getSubscriberInfo 方法是动态类的内置方法，通过 class 实例获取 SubscriberInfo；
        if (subscriberInfoIndexes != null) {
            for (SubscriberInfoIndex index : subscriberInfoIndexes) {
                SubscriberInfo info = index.getSubscriberInfo(findState.clazz);
                if (info != null) {
                    return info;
                }
            }
        }
        return null;
    }
```

这里我们知道，通过前面的分析，每一个订阅者都是一个 SubscriberInfo 实例！

## 5.4 prepareFindState

主要是为每个 find 操作，创建一个 FindState 对象；

```java
    private FindState prepareFindState() {
        synchronized (FIND_STATE_POOL) {
            for (int i = 0; i < POOL_SIZE; i++) {
                FindState state = FIND_STATE_POOL[i];
                if (state != null) {
                    FIND_STATE_POOL[i] = null;
                    return state;
                }
            }
        }
        //【-->6】返回了一个 FindState 实例；
        return new FindState();
    }
```

默认情况下，**FIND_STATE_POOL** 是空的，所以会创建一个新的 **FindState** 实例；



## 5.5 findUsingReflectionInSingleClass

通过反射的方式获取注册方法，通过 SubscriberMethod 实例封装，保存到 findState.subscriberMethods 中：

参数为 FindState 实例，这里获取方法的是通过 findState.clazz，这个要注意；

```java
    private void findUsingReflectionInSingleClass(FindState findState) {
        Method[] methods;
        try {
            //【0】这个方法比 getMethods 快，特别是当订阅者是像活动这样的方法很多的类的时候；
            methods = findState.clazz.getDeclaredMethods();
        } catch (Throwable th) {
            // 可能会抛出 java.lang.NoClassDefFoundError, see [https://github.com/greenrobot/EventBus/issues/149]
            // 这里会使用 getMethods() 方法获取，注意，会跳过父类；
            methods = findState.clazz.getMethods();
            findState.skipSuperClasses = true;
        }
        for (Method method : methods) {
            int modifiers = method.getModifiers();
            //【1】这里是对 method 方法访问域做校验，必须是 public，不能是 abstract 和 static 的；
            if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & MODIFIERS_IGNORE) == 0) {
                Class<?>[] parameterTypes = method.getParameterTypes();
                if (parameterTypes.length == 1) {
                    //【1.1】获取方法对应的注解 Subscribe，只处理被该注解修饰的方法;
                    Subscribe subscribeAnnotation = method.getAnnotation(Subscribe.class);
                    if (subscribeAnnotation != null) {
                        //【1.2】获取方法对应的参数；
                        Class<?> eventType = parameterTypes[0];
                        //【-->6.4】检查要添加方法的信息，没问题的话，就创建方法对应的 SubscriberMethod，添加到 findState.subscriberMethods；
                        if (findState.checkAdd(method, eventType)) {
                            //【1.3】获取订阅的分发线程信息；
                            ThreadMode threadMode = subscribeAnnotation.threadMode();
                            //【1.4】创建方法对应的 SubscriberMethod，添加到 findState.subscriberMethods；
                            // SubscriberMethod 参数：订阅方法 method，订阅事件的 class 实例 eventType，线程模式 threadMode，优先级，粘性；
                            findState.subscriberMethods.add(new SubscriberMethod(method, eventType, threadMode,
                                    subscribeAnnotation.priority(), subscribeAnnotation.sticky()));
                        }
                    }
                } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                    //【2】如果开启方法校验，那么被 @Subscribe 修饰的方法只能一个参数；
                    String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                    throw new EventBusException("@Subscribe method " + methodName +
                            "must have exactly 1 parameter but has " + parameterTypes.length);
                }
            } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                throw new EventBusException(methodName +
                        " is a illegal @Subscribe method: must be public, non-static, and non-abstract");
            }
        }
    }
```

**strictMethodVerification** 表示是否强制校验方法的访问修饰符；

## 5.6 getMethodsAndRelease

返回收集到的方法：

```java
    private List<SubscriberMethod> getMethodsAndRelease(FindState findState) {
        //【1】通过 FindState 获取查询到的方法；
        List<SubscriberMethod> subscriberMethods = new ArrayList<>(findState.subscriberMethods);
        //【-->6.5】clear 掉缓存；
        findState.recycle();
        synchronized (FIND_STATE_POOL) {
            for (int i = 0; i < POOL_SIZE; i++) {
                if (FIND_STATE_POOL[i] == null) {
                    FIND_STATE_POOL[i] = findState; // 将这个对象缓存下来，防止频繁的创建 FindState 对象；
                    break;
                }
            }
        }
        return subscriberMethods;
    }
```



# 6 FindState

FindState 是 SubscriberMethodFinder 的内部类，用于保存查询的结果和状态信息（包括子类和父类）：

## 6.1 成员变量

```java
    static class FindState {
        // 保存订阅者的方法 SubscriberMethod；
        final List<SubscriberMethod> subscriberMethods = new ArrayList<>();
        // 保存订阅事件 class 实例 和 [订阅方法 Method/所属 FindState] 的映射关系；
        final Map<Class, Object> anyMethodByEventType = new HashMap<>();
        // 保存了 methodKey 和 method 所属的订阅类的 class 实例；
        final Map<String, Class> subscriberClassByMethodKey = new HashMap<>();
      
        // 用于生成 subscriberClassByMethodKey 中的 methodKey
        final StringBuilder methodKeyBuilder = new StringBuilder(128);

        Class<?> subscriberClass; // 订阅者 class 实例，也就是调用 register 方法的类；
        Class<?> clazz; // 初始化时，取值和 subscriberClass 一样，但是在处理继承关系时，会转为 superClass
        boolean skipSuperClasses; // 是否 skip 父类，初始化为 false；
        SubscriberInfo subscriberInfo; // 订阅者信息，开启了 APT 才有，否则为 null；
```

- subscriberClassByMethodKey：如果存在继承关系，同时方法有覆盖，那么以子类为准；

## 6.2 initForSubscriber

初始化操作；

```java
        void initForSubscriber(Class<?> subscriberClass) {
            //【1】初始化：subscriberClass == clazz
            this.subscriberClass = clazz = subscriberClass;
            skipSuperClasses = false;
            subscriberInfo = null;
        }
```

## 6.3 moveToSuperclass

跳转到 superClass，处理父类：

```java
        void moveToSuperclass() {
            //【1】如果要跳过 superClass，那么 clazz 为 null；
            if (skipSuperClasses) {
                clazz = null;
            } else {
                //【2】获取其 superClass；
                clazz = clazz.getSuperclass();
                String clazzName = clazz.getName();
                //【3】跳过系统的类；
                if (clazzName.startsWith("java.") || clazzName.startsWith("javax.") || clazzName.startsWith("android.")) {
                    clazz = null;
                }
            }
        }
```

不多说了；

##  6.4 checkAdd

检查要添加方法的信息：

```java
        boolean checkAdd(Method method, Class<?> eventType) {
            //【1】这里会将订阅事件 class 实例 --> 订阅方法 Method 放入 anyMethodByEventType 中；
            // 同时会返回以之前已经存在的 value；
            Object existing = anyMethodByEventType.put(eventType, method);
            if (existing == null) {
                //【2】如果是第一次添加，那么 check 成功，直接返回 true；
                return true;
            } else {
                //【3】如果之前添加过 eventType，说明可能一个类有多个处理该 eventType 的函数；
                // 也有可能是有继承关系，此时处理的是父类的方法，子类覆盖了父类的同名方法；
                if (existing instanceof Method) {
                    //【-->6.4.1】检查已经添加的方法 existing 的方法签名，只有第二次处理同一个 eventType 才会进入这里
                    // 此时该方法是会返回 true 的，因为第一次添加的 method 还没有做签名校验；
                    if (!checkAddWithMethodSignature((Method) existing, eventType)) {
                        // Paranoia check
                        throw new IllegalStateException();
                    }
                    //【4】这里很奇怪，直接将之前的 Method 替换成了 FindState 实例；
                    // 如果有多个方法都处理同一个 eventType 的话，显然 value 就不是 Method 的实例了；
                    anyMethodByEventType.put(eventType, this);
                }
                //【-->6.4.1】检查新添加的方法 method 的方法签名；
                return checkAddWithMethodSignature(method, eventType);
            }
        }
```

可以看到，作者其实在注释里面也有说明：有两级的检查：

- 第一级：检查时间类型；
- 第二级：检查方法签名；

可能有多个处理该 eventType 的函数：

- 只有第一次添加会进入 `if (existing instanceof Method) {` 分支；
- 第二次就会将 anyMethodByEventType 中的 value 从 Method 变为 FindState，那么就不会进入 `if (existing instanceof Method) {` 分支了；
- 无论一类多方法，还是继承一方法，都是上面的流程；

（但是看作者的注释：貌似没有考虑一个订阅者有多个监听相同事件类型的方法。）

如果该 checkAdd 方法返回的是 false，那么 @Subscribe 修饰的方法就不会被收集！！！

### 6.4.1 checkAddWithMethodSignature

检查方法签名：

```java
        private boolean checkAddWithMethodSignature(Method method, Class<?> eventType) {
            methodKeyBuilder.setLength(0);
            methodKeyBuilder.append(method.getName());
            methodKeyBuilder.append('>').append(eventType.getName());
            //【1】生成 methodKey：methodName>eventTypeName
            String methodKey = methodKeyBuilder.toString();
            //【2】获取方法所在的类 class 实例；
            Class<?> methodClass = method.getDeclaringClass();
            //【3】将 methodKey 和 methodClass 的映射关系放入 subscriberClassByMethodKey 中，同时返回旧的 value；
            Class<?> methodClassOld = subscriberClassByMethodKey.put(methodKey, methodClass);
            //【4】这里针对 old class 实例和 new class 做了比较；
            // 如果是第一次 add ，或者有 old class 实例，同时 old class 是 new class 的 super class
            // 那么就用 new class 替换旧的值；
            if (methodClassOld == null || methodClassOld.isAssignableFrom(methodClass)) {
                // Only add if not already found in a sub class
                return true;
            } else {
                // Revert the put, old class is further down the class hierarchy
                subscriberClassByMethodKey.put(methodKey, methodClassOld); // 恢复旧的值；
                return false;
            }
        }
```

这里实际上是方法的签名：方法名>参数

对于同一个类，如果有多个函数处理同一个 eventType，显然方法签名是不一样的，那么这个 checkAddWithMethodSignature 返回的是 true；

对于继承关系，对于子类和父类有相同的方法签名的情况，以子类为准，也就是说父类的同名同参方法是不会被收集的 checkAddWithMethodSignature 返回的是 false；；



## 6.5 recycle

回收内部的变量，就是 clear 操作：

```java
        void recycle() {
            subscriberMethods.clear();
            anyMethodByEventType.clear();
            subscriberClassByMethodKey.clear();
            methodKeyBuilder.setLength(0);
            subscriberClass = null;
            clazz = null;
            skipSuperClasses = false;
            subscriberInfo = null;
        }
```

就不多说了，啊哈哈哈哈哈～～

# 7 Subscription

表示一种订阅关系；



## 7.1 成员变量

我们来看下成员属性：

```java
final class Subscription {
    final Object subscriber; // 订阅者；
    final SubscriberMethod subscriberMethod; // 订阅方法；
```



## 7.2 new Subscription

创建订阅关系：

```java
    Subscription(Object subscriber, SubscriberMethod subscriberMethod) {
        this.subscriber = subscriber; // 订阅者；
        this.subscriberMethod = subscriberMethod; // 订阅方法；
        active = true;
    }
```


# 8 总结

我们分析了 EventBus 的创建，注册和反注册，整个初始化和注册的过程主要分为下面的基本：

- 通过 EventBusBuilder 创建 EventBus；
- EventBus 收集当前类以及其父类所有的订阅方法；
- 根据事件类型和每一个订阅方法，创建订阅关系；

遗留了如下的几个问题：

- post 操作的执行流程；
- 不同线程模式的消息是如何分发和处理的；