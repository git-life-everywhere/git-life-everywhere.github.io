# EventBus 第一篇 - 基本使用
title: EventBus 第一篇 - 基本使用
date: 2019/08/19 20:46:25 
catalog: true
categories: 

- 开源库源码分析 
- EventBus
tags: EventBus
copyright: true
---

本系列文章主要分析 EventBus 框架的架构和原理，基于最新的 **3.1.0** 版本。

> 这是 EventBus 开源库的地址，大家可以直接访问
> https://github.com/greenrobot/EventBus

本篇文章是 EventBus 的第一篇，主要总结下基本的使用；

Eventbus 翻译过来就是事件总线，用于简化组件和组件，线程和线程之间的消息通信，可以捆成是 Handler + Thread 的替代品。

# 1 引入

Eventbus 的引入没有 ARouter 那么复杂，他的核心 api 和 AnnotationProcessor 是在同一个 jar 中：

```java
compile 'org.greenrobot:eventbus:3.0.0'
```

以上就是引入的方式，很简单；

# 2 基本使用

Eventbus 的使用还是很简单的。

- 首先，**我们要在组件生命周期的开始 register、生命周期的结束 unregister**：

```java
EventBus.getDefault().register(this);

EventBus.getDefault().unregister(this);
```

我们之后将组件 register 到 EventBus 中，该组件才能监听到事件；

当然，当组件生命周期结束后，需要 unregister！

- 接着，**我们要定义接收 Event 的方法**；

```java
    @Subscribe
    public void onEventMainThread(MessageEvent event) {
    }
```

在 EventBus 中，处理 event 的方法需要被注解  @Subscribe 修饰，这是因为 EventBus 的机制，提供了一个 EventBusAnnotationProcessor，他负责自动处理   @Subscribe 修饰的方法，动态生成管理集合。

在事件分发的时候，会自动调用我们的方法；

对于注解 @Subscribe，我们可以设置其属性：

```java
 @Subscribe(threadMode = ThreadMode.MAIN, sticky = true, priority = 2)
```

1、threadMode 用于指定线程模型（默认为 POSTING ），EventBus 提供了四种线程模型，下面会简单介绍；

2、sticky 表示方法是否开启粘性事件；

3、priority 表示多个订阅者收到事件的优先级顺序；



- 最后，我们要**发送消息**

消息这里分为普通消息和粘性消息，和 broadcast 很类似哦：

```java
EventBus.getDefault().postSticky(..)

EventBus.getDefault().post(...)
```

对于普通消息和粘性消息的处理，后面再分析。

方法很简答，就不多说了～～

# 3 线程模型

EventBus 提供了四种线程模型，定义在 ThreadMode.java 中：

- **POSTING**

这是默认的线程模型，发布事件和接收事件在同一个线程进行，不要做耗时操作，因为可能是在 UI 线程，导致 ANR；

- **MAIN**

接收事件在 UI 线程中进行；不要做耗时操作，会导致 ANR；

- **BACKGROUND**

如果发送事件是在 UI 线程，那么接收事件会在一个新的子线程；

如果发送事件是在子线程，那么接收事件和发送事件会在同一个子线程；

不能处理 UI 相关操作！

- **ASYNC**

接收事件始终会在一个新的子线程中，不能处理 UI 相关操作！

> 这里简单分析了下线程模型，我们后面在分析源码的时候，再来分析每种线程模型的处理方式；

# 4 整体架构初识

可以看到这种订阅和接收的关系，很类似于 Rxjava 的模式，其实就是观察者模式，这里直接引用 EventBus 官方的一张图来说明下：

![](https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/23/1240.png)


# 5 总结

本篇文章就到这里了，下一篇会从 @Subscribe 注解的处理入手，看下 EventBus 是如何处理该注解的；







