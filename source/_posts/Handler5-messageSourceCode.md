# Handler篇 5 - Message 源码分析
title: Handler篇 5 - Message 源码分析
date: 2017/03/13 20:46:25
catalog: true
categories:
- AndroidFramework源码分析
- Handler线程消息机制
tags: Handler线程消息机制
copyright: true
---

基于 Android 7.1.1 源码，分析 Message 的架构和原理。

# 1 成员变量

```java
    public int what;
```
用于标识 Message！

```java
    public int arg1;
    public int arg2;
```
用于传递简单的整型数据，如果想传递复杂的数据使用 Bundle

```java
    public Object obj;
```
用于发送任意对象，如果要用 Message 进行跨进程通信，obj 必须实现序列化接口！

```java
    public Messenger replyTo;
```
用于跨进程双向通信，接受到该 message 的进程，可以通过 Message.replyTo 向发送方进程发送 message，从而实现双向通信！

这个在分析 Messenger 的时候会看到！

```java
    public int sendingUid = -1;
```
发送方的 uid，只有在通过 Messenger 跨进程通信的时候才有效，否则为 -1；

```java
    int flags;
```
Message 的标志位，可以取如下的值；

- **static final int FLAG_IN_USE = 1 << 0**：用于标识当前的 Message 处于使用状态，当 Message 处于消息队列中、处于消息池中或者 Handler 正在处理该 Message 的时候，它就处于使用状态；

- **static final int FLAG_ASYNCHRONOUS = 1 << 1**：用于标识当前的 Message 是异步的；

- **static final int FLAGS_TO_CLEAR_ON_COPY_FROM = FLAG_IN_USE**：当调用 copyFrom 方法的时候，会去掉 FLAG_IN_USE 标志位！

```java
    long when;
```
Message 的分发时间；

```java
    Bundle data;
```
用于传输比较复杂的数据；

```java
    Handler target;
```
表示消息的目标处理 Handler！

```java
    Runnable callback;
```
表示要处理的任务 Runnable，通过 handler post Runnable 的时候，Runnable 会被封装成一个 Message，如果要用 Message 进行跨进程通信，callback 必须为 null，因为其不能序列化！！

```java
    // sometimes we store linked lists of these things
    Message next;
```

```java
    private static final Object sPoolSync = new Object();
```
用于同步的锁对象！


```java
    private static Message sPool;
```
静态变量，我们知道 Handler 发送过的 message 会被缓存到消息池中，方便复用，其实消息池也是一个列表，sPool 是这个链表的头元素！！


```java
    private static int sPoolSize = 0;
```
静态变量，用于记录消息池中 Message 的数量，也就是链表的长度，其最大不能超过 MAX_POOL_SIZE 规定的大小！


```java
    private static final int MAX_POOL_SIZE = 50;
```
消息池中 Message 的最大数量！


```java
    private static boolean gCheckRecycle = true;
```

# 2 create Message

想要发送 Message，首先要创建 Message，创建分为 2 种：

- 直接创建；
- 复用已有消息；

下面我们来分别看看；

## 2.1 new Message

直接创建，能够保证我们每次创建的都是一个新的消息对象：

```java
    public Message() {
    }
```
构造器很简单，没有多么复杂，我们可以创建一个 Message 对象，然后设置它的属性：
```java
        Message msg = new Message();
        Bundle b = new Bundle();
        b.putString("name", "coolqi");
        b.putString("face", "cool");
        msg.setData(b);
        mHandler.sendMessage(msg);
```
这里就不多说了！

## 2.2 Message.obtain

复用已有消息，我们知道 handler 会将发送过的消息，保存到一个消息池中，便于复用！

Message 提供了多个 obtain 用于复用一个 Message！

```java
    public static Message obtain() {...}

    public static Message obtain(Message orig) {...}

    public static Message obtain(Handler h) {...}

    public static Message obtain(Handler h, Runnable callback) {...}

    public static Message obtain(Handler h, int what) {...}

    public static Message obtain(Handler h, int what, Object obj) {...}

    public static Message obtain(Handler h, int what, int arg1, int arg2) {...}

    public static Message obtain(Handler h, int what, int arg1, int arg2, Object obj) {...}
```
上面的多个 obtain 都会先尝试从消息池中获取一个缓存的消息，如果找不到，再创建一个新的消息，然后**用传入的参数更新这个消息对应的属性值**，并返回！


```java
    public static Message obtain() {
        synchronized (sPoolSync) {
            if (sPool != null) { // 每次都复用消息池的头消息！
                Message m = sPool;
                sPool = m.next;
                m.next = null;
                m.flags = 0; // 清空 flag，不再为使用状态
                sPoolSize--;
                return m;
            }
        }
        return new Message(); // 如果没有就创建一个新的消息！
    }
```
代码很简单，不多了！


# 3 recycle Message

回收一个 Message 分为 2 种情况，一种是安全的回收，一种是不安全的回收！

## 3.1 safe recycle

安全的回收：

```java
    public void recycle() {
        if (isInUse()) { // 判断是否在使用中!
            if (gCheckRecycle) {
                throw new IllegalStateException("This message cannot be recycled because it "
                        + "is still in use.");
            }
            return;
        }
        recycleUnchecked(); // 不再使用了，回收！
    }
```

安全回收在回收 Message 之前会先检查 Message 是否处于使用状态，处于使用状态下是不能回收的！！

recycle() 在判断安全后，会调用 recycleUnchecked() 进行回收！

```java
    /*package*/ boolean isInUse() {
        return ((flags & FLAG_IN_USE) == FLAG_IN_USE);
    }
```
上面的方法用于判断 Message 是否在使用中！

gCheckRecycle 属性在 SDK 21 以及以后默认为 true，他用来决定在回收时如果正在使用，是否抛出异常！

```java
    /** @hide */
    public static void updateCheckRecycle(int targetSdkVersion) {
        if (targetSdkVersion < Build.VERSION_CODES.LOLLIPOP) {
            gCheckRecycle = false;
        }
    }
```

上面是具体的判断方法！

## 3.2 unsafe recycle

不安全的回收：

```java
    void recycleUnchecked() {
        flags = FLAG_IN_USE;
        what = 0;
        arg1 = 0;
        arg2 = 0;
        obj = null;
        replyTo = null;
        sendingUid = -1;
        when = 0;
        target = null;
        callback = null;
        data = null;

        synchronized (sPoolSync) {
            if (sPoolSize < MAX_POOL_SIZE) { // 如果消息池未满，将回收后的消息加入到消息池中！
                next = sPool;
                sPool = this;
                sPoolSize++;
            }
        }
    }
```
可以看到 recycleUnchecked() 会重置掉 Message 的属性！


# 4 跨进程传输

由于 Message 实现了 Parcelable 接口，所以可以序列化，跨进程传输数据，我们知道信使 Messenger 就是通过 Message 来实现跨进程通信的！

## 4.1 writeToParcel

序列化方法：

```java
    public void writeToParcel(Parcel dest, int flags) {
        if (callback != null) { // 如果消息的 callback 不为，不能序列化！
            throw new RuntimeException(
                "Can't marshal callbacks across processes.");
        }
        dest.writeInt(what);
        dest.writeInt(arg1);
        dest.writeInt(arg2);
        if (obj != null) { // 如果要传递任意对象，必须实现 Parcelable 接口！
            try {
                Parcelable p = (Parcelable)obj;
                dest.writeInt(1);
                dest.writeParcelable(p, flags);
            } catch (ClassCastException e) {
                throw new RuntimeException(
                    "Can't marshal non-Parcelable objects across processes.");
            }
        } else {
            dest.writeInt(0);
        }
        dest.writeLong(when);
        dest.writeBundle(data);
        Messenger.writeMessengerOrNullToParcel(replyTo, dest); // 针对于 Messenger 的特殊处理！
        dest.writeInt(sendingUid);
    }
```
如果 Message 设置了 Runnable callback，那么不能序列化！

## 4.2 readFromParcel

反序列化方法：

```java
    private void readFromParcel(Parcel source) {
        what = source.readInt();
        arg1 = source.readInt();
        arg2 = source.readInt();
        if (source.readInt() != 0) {
            obj = source.readParcelable(getClass().getClassLoader());
        }
        when = source.readLong();
        data = source.readBundle();
        replyTo = Messenger.readMessengerOrNullFromParcel(source); // 针对于 Messenger 的特殊处理！
        sendingUid = source.readInt();
    }
```

对于 Messenger 的内容，我们会单独开一篇文章来分析！


# 5 其他方法

Message 提供了一些其他的方法：

- copyFrom 用于将一个 Message 的属性 copy 到另一个 Message 中：

```java
    public void copyFrom(Message o) {
        this.flags = o.flags & ~FLAGS_TO_CLEAR_ON_COPY_FROM; // 去掉 FLAG_IN_USE 标志位！
        this.what = o.what;
        this.arg1 = o.arg1;
        this.arg2 = o.arg2;
        this.obj = o.obj;
        this.replyTo = o.replyTo;
        this.sendingUid = o.sendingUid;

        if (o.data != null) {
            this.data = (Bundle) o.data.clone();
        } else {
            this.data = null;
        }
    }
```

- 异步相关的方法：

```java
    public boolean isAsynchronous() {
        return (flags & FLAG_ASYNCHRONOUS) != 0;
    }

    public void setAsynchronous(boolean async) {
        if (async) {
            flags |= FLAG_ASYNCHRONOUS;
        } else {
            flags &= ~FLAG_ASYNCHRONOUS;
        }
    }
```

以及一些其他的用于 get 属性，打印属性的方法，这里就不多说了！！

