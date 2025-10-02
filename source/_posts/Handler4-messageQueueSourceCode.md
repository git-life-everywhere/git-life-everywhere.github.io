# Handler篇 4 - MessageQueue 源码分析
title: Handler篇 4 - MessageQueue 源码分析
date: 2017/02/27 20:46:25
catalog: true
categories:
- AndroidFramework源码分析
- Handler线程消息机制
tags: Handler线程消息机制
copyright: true
---

基于 Android 7.1.1 源码，分析 handler 的架构和原理。


# 1 成员变量

```java
    private final boolean mQuitAllowed;
```
该变量表示 MessageQueue 是否可以退出，主线程的消息队列不可退出。


```java
    @SuppressWarnings("unused")
    private long mPtr; // used by native code
```
java 层有一个 MessageQueue，同样的 native 层也有一个 MessageQueue，java 层 MessageQueue 在初始化是，也会初始化 native 层的 MessageQueue，这个变量用来保存 natvie 层的消息队列的句柄！


```java
    Message mMessages;
```
MessageQueue 消息队列中的所有消息是通过链表联系在一起的，mMessages 是这个链表的头元素！

```java
    private IdleHandler[] mPendingIdleHandlers;
```
用于保存将要被执行的 IdleHandler，当消息队列要执行 IdleHandler 的时候，他会将 mIdleHandlers 中的所有 IdleHandler 拷贝到 mPendingIdleHandlers！

```java
    private final ArrayList<IdleHandler> mIdleHandlers = new ArrayList<IdleHandler>();
```
用于保存该线程的所有 IdleHandler！


```java
    private SparseArray<FileDescriptorRecord> mFileDescriptorRecords;
```
用于记录文件描述符


```java
    private boolean mQuitting;
```
该变量表示 MessageQueue 是否正在关闭退出！

```java
    private boolean mBlocked;
```
该变量表示 MessageQueue 是否是阻塞的，当 MessageQueue 中没有任何消息的时候，消息队列会阻塞，等待消息的插入！


```java
    private int mNextBarrierToken;
```

这里要提及一个概念叫：障栅（Barrier）。

障栅是一个特殊的 Message，他的 target 为 null，并且其 Message.arg1 作为句柄，标识每一个独一无二的障栅！

障栅的作用很重要，它能够拦截同步 Message，阻止同步消息被执行，放行异步 Message！后面我们会看到！

mNextBarrierToken 的作用是计算下一个 Barrier 的 token！


# 2 create MessageQueue

我们来回顾下 Looper 的创建：
```java
    private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed); // 这里创建了一个消息队列！
        mThread = Thread.currentThread();
    }
```

## 2.1 new MessageQueue
这里会调用 MessageQueue 构造器，创建一个消息队列！

```java
    MessageQueue(boolean quitAllowed) {
        mQuitAllowed = quitAllowed;
        mPtr = nativeInit(); // 初始化 native 层的消息队列，并返回其指针！
    }
```

**mQuitAllowed 成员变量表示该消息队列是否可以关闭**，对于 ui 线程的消息队列，是不能退出的，我们可以回顾下，ui 线程的 Looper 创建：

```java
    public static void prepareMainLooper() {
        prepare(false); // prepare 方法的参数就是 quitAllowed，这里传入的是 false；
        synchronized (Looper.class) {
            if (sMainLooper != null) {
                throw new IllegalStateException("The main Looper has already been prepared.");
            }
            sMainLooper = myLooper();
        }
    }
```

nativeInit 方法是一个 native 方法，他会初始化处于 native 层的 MessageQueue，他和 java 层的 MessageQueue 一一对应！

在 Android 2.3 之前，只有 java 层才可以向 MessageQueue 中添加消息，在 Android 2.3 之后，MessageQueue 的核心部分移动到了 native 层。这样，java 层和 native 层都可以使用 MessageQueue！

也就是说， Java 层的 MessageQueue 处理 Java 层的消息，natvie 层的 MessageQueue 负责处理 native 层的消息！

# 3 enqueue Message

我们回到 Handler 中，当我们通过 handler 发送消息的时候，会调用 Handler.enqueueMessage 方法：

```java
    private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this; // 设置消息的目标
        if (mAsynchronous) { // 如果 Handler 是异步的，其内部消息都是异步的！
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis); // 将消息加入到队列中！
    }
```

## 3.1 MessageQueue.enqueueMessage

最终会调用 enqueueMessage 将消息插入到 MessageQueue 中，下面我们来看看 MessageQueue.enqueueMessage 方法：

```java
    boolean enqueueMessage(Message msg, long when) {
        //【1】校验下 Message 的有效性！
        if (msg.target == null) {
            throw new IllegalArgumentException("Message must have a target.");
        }
        if (msg.isInUse()) { //【2】如果 message 正在使用，抛出异常！
            throw new IllegalStateException(msg + " This message is already in use.");
        }

        synchronized (this) {
            if (mQuitting) { // 如果 MessageQueue 正在关闭，抛出异常，我们不能
                IllegalStateException e = new IllegalStateException(
                        msg.target + " sending message to a Handler on a dead thread");
                Log.w(TAG, e.getMessage(), e);
                msg.recycle();
                return false;
            }

            msg.markInUse(); // 设置 FLAG_IN_USE 标志位，表示正在使用中！
            msg.when = when;
            Message p = mMessages;
            boolean needWake;
            if (p == null || when == 0 || when < p.when) {
                // 新的 message 为消息队列的新头元素！
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked; // 此时是否唤醒消息队列，取决于是否阻塞！
            } else {
                // 如果不是新的头元素，那就将其添加到正确的位置！
                Message prev;
                for (;;) {
                    prev = p; // 确定 pre 和 next message！
                    p = p.next;
                    if (p == null || when < p.when) { // 按照分发时间从小到大排序！
                        break;
                    }
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                msg.next = p; // 找到合适的位置并插入！
                prev.next = msg;
            }

            if (needWake) {
                // 唤醒消息队列！
                nativeWake(mPtr);
            }
        }
        return true;
    }
```

这里来说下 MessageQueue 的成员变量 mMessages：在 MessageQueue 中，所有的 Message 是以链表的形式组织在一起的，mMessages 是链表的头元素！



# 4 dispatch Message

在前面 Looper 分析中，我们知道，Looper.loop() 方法会进入一个 for 死循环，不断的调用 MessageQueue.next 方法，返回下一个 Message。

## 4.1 MessageQueue.next
下面我们来看看 MessageQueue.next 方法：

```java
    Message next() {
        //【1】当 mPtr == 0 的时候，说明消息队列关闭了，那么我们会返回一个 null 的 Message
        // 这样消息循环就会关闭！
        final long ptr = mPtr;
        if (ptr == 0) {
            return null;
        }

        int pendingIdleHandlerCount = -1; // Looper 每次调用 MQ.next 方法，都会初始化为 -1；
        int nextPollTimeoutMillis = 0;
        for (;;) {
            if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
            }

            nativePollOnce(ptr, nextPollTimeoutMillis);

            synchronized (this) {
                // 从链表头开始，分发 Message！
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;
                if (msg != null && msg.target == null) {
                    // 如果 msg 不为 null，但是 msg.target，那说明这是障珊，根据障珊的特性，拦截同步，放行异步
                    // 那就顺序遍历，找到下一个异步 Message！
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous()); // 如果消息是同步的，继续查找！
                }
                
                // 如果 msg 不为 null，说明我们找到了要分发的消息（异步/同步）！
                if (msg != null) {
                    if (now < msg.when) {
                        // Next message is not ready.  Set a timeout to wake up when it is ready.
                        // 如果消息还没有到分发的时间，设置一个超时时间用于触发他！
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);

                    } else {
                        // 消息的分发时间已经到了，分发消息！
                        mBlocked = false;
                        if (prevMsg != null) {
                            // 如果 prevMsg 不为 null，说明我们分发的是异步消息，这里要重新设置链表元素关系！
                            prevMsg.next = msg.next;
                        } else {
                            // 如果 prevMsg 为 null，说明我们分发的是同步消息，修改链表头！
                            mMessages = msg.next;
                        }
                        
                        // 设置 msg.next 为 null，并设置 msg 为正在使用状态；
                        msg.next = null;
                        if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                        msg.markInUse();
                        return msg;
                    }
                } else {
                    nextPollTimeoutMillis = -1; // 没有要分发的 message！
                }

                // 如果 mQuitting 为 true，说明消息队列正在关闭，那就返回一个 null 的 Message，
                // 这样 Looper 就会结束消息循环！
                if (mQuitting) {
                    dispose();
                    return null;
                }

                // 能进入这里，说明前面我们没有找到分发的 message 或者消息触发事件未到！
                // 如果本次 next 第一次进入空闲状态，即没有消息去分发，那就执行 IdleHandler！
                // 执行的条件是：消息队列为空，或者消息队列的第一个消息还没有到执行时间！
                if (pendingIdleHandlerCount < 0
                        && (mMessages == null || now < mMessages.when)) {
                    pendingIdleHandlerCount = mIdleHandlers.size(); // 计算 IdleHandler 个数！
                }
                
                if (pendingIdleHandlerCount <= 0) {
                    // 如果连 IdleHandler 也没有，那么说明消息循环没有任何消息需要处理！
                    // 那就进入阻塞状态，设置 mBlocked 为 true！
                    mBlocked = true;
                    continue;
                }

                // 将 mIdleHandlers 中需要触发的 IdleHandler 拷贝到 mPendingIdleHandlers 中！
                if (mPendingIdleHandlers == null) {
                    mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
                }
                mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
            }

            // 运行收集到的 IdleHandler
            for (int i = 0; i < pendingIdleHandlerCount; i++) {
                final IdleHandler idler = mPendingIdleHandlers[i];
                mPendingIdleHandlers[i] = null; // 取消引用！

                boolean keep = false;
                try {
                    // queueIdle 返回值，表示是否保留该 IdleHandler！
                    keep = idler.queueIdle();
                } catch (Throwable t) {
                    Log.wtf(TAG, "IdleHandler threw exception", t);
                }

                if (!keep) {
                    synchronized (this) {
                        mIdleHandlers.remove(idler); // keep 为 false，不保留，移除！
                    }
                }
            }

            // 将 pendingIdleHandlerCount 重置为 0 ，本次 next 将不会再执行 IdleHandler！
            pendingIdleHandlerCount = 0; 

            // 我们在执行 IdleHandler 之后，会消耗一些时间，这时候消息队列里的可能有消息已经到达
            // 可执行时间，所以重置该变量回去重新检查消息队列。
            nextPollTimeoutMillis = 0;
        }
    }

```
方法逻辑：

- 查询下一个要触发的消息：next 方法会启动一个 for 循环，顺序遍历消息链表：
   - 如果头消息是障栅，那就顺序查找下一个异步消息！
   - 如果头消息不是障栅，那么那其就是同步消息，也是我们即将分发的消息！

</br>
   
- 如果消息链表中没有任何消息，或者消息的分发时间未到，那么消息循环会进入阻塞状态，其实就是不断的 for 循环；在进入阻塞状态之前，会查询是否有 idleHandler 触发，如果没有会立刻进入阻塞状态，否则，会触发 IdleHandler，然后在再进入阻塞状态！

    - 所谓的阻塞，实际上是不断地 for 循环，直到有消息被插入，idleHandler 触发只会在进入阻塞状态的第一次 for 循环执行！

</br>

- 当 MessageQueue 退出关闭的时候，mQuitting 会被置为 true，这样 MessageQueue.next 会返回一个 null 的 Message，回顾 Looper.loop，消息循环就可以退出了！

```java
    /*package*/ void markInUse() {
        flags |= FLAG_IN_USE;
    }
```
将一个 message 设置为正在使用的状态！

# 5 quit MessageQueue

回顾 Looper.quit 方法：
```java
    public void quit() {
        mQueue.quit(false);
    }
```
当我们不想使用消息队列了，我们可以关闭它，最终调用的 MessageQueue.quit 方法:

## 5.1 MessageQueue.quit 

```java
    void quit(boolean safe) {
        // 如果 mQuitAllowed 为 false，说明消息队列不可以关闭！
        if (!mQuitAllowed) {
            throw new IllegalStateException("Main thread not allowed to quit.");
        }

        synchronized (this) {
            if (mQuitting) { // 如果已经在退出了，不允许退出 2 次！
                return;
            }

            mQuitting = true; // 设置 mQuitting 为 true！

            // 根据是否安全推出做不同处理！
            if (safe) {
                removeAllFutureMessagesLocked();
            } else {
                removeAllMessagesLocked();
            }

            // We can assume mPtr != 0 because mQuitting was previously false.
            nativeWake(mPtr);
        }
    }
```
方法逻辑：

- 设置 mQuitting 为 true；
- 如果是安全关闭，调用 removeAllFutureMessagesLocked 移除当前时间点以后未分发的消息；
- 如果是非安全关闭，调用 removeAllMessagesLocked 移除所有的消息；


# 6 remove Message

回顾 Handler。Handler 提供了 remove 方法：

```java
    public final void removeMessages(int what) {
        mQueue.removeMessages(this, what, null);
    }
```
最终调用了 MessageQueue 的 removeMessages 方法！

MessageQueue 有多个 remove 方法，我们一个一个来看：

## 6.1 MessageQueue.removeMessages

- 移除 Messages：

```java
    void removeMessages(Handler h, int what, Object object) {
        if (h == null) {
            return;
        }

        synchronized (this) {
            Message p = mMessages;
            
            //【1】从头消息开始移除，如果头消息匹配到了。更新头消息指向，并移除 what 对应的消息！
            while (p != null && p.target == h && p.what == what
                   && (object == null || p.obj == object)) {
                Message n = p.next;
                mMessages = n;
                p.recycleUnchecked(); // 非安全回收！
                p = n;
            }

            //【2】进入这里说明头消息现在已经不用被移除了，然后在非头消息的剩余消息中移除能够匹配的消息！
            while (p != null) {
                Message n = p.next;
                if (n != null) {
                    if (n.target == h && n.what == what
                        && (object == null || n.obj == object)) {
                        Message nn = n.next;
                        n.recycleUnchecked(); // 非安全回收！
                        p.next = nn;
                        continue;
                    }
                }
                p = n;
            }
        }
    }
```

- 移除 Runnable：

```java
    void removeMessages(Handler h, Runnable r, Object object) {
        if (h == null || r == null) {
            return;
        }

        synchronized (this) {
            Message p = mMessages;

            //【1】从头消息开始移除，如果头消息匹配到了。更新头消息，并移除 Runnable 对应的消息！
            while (p != null && p.target == h && p.callback == r
                   && (object == null || p.obj == object)) {
                Message n = p.next;
                mMessages = n;
                p.recycleUnchecked(); // 非安全回收！
                p = n;
            }

            //【2】进入这里说明头消息现在已经不用被移除了，然后在非头消息的剩余消息中移除能够匹配的消息！
            while (p != null) {
                Message n = p.next;
                if (n != null) {
                    if (n.target == h && n.callback == r
                        && (object == null || n.obj == object)) {
                        Message nn = n.next;
                        n.recycleUnchecked(); // 非安全回收！
                        p.next = nn;
                        continue;
                    }
                }
                p = n;
            }
        }
    }
```
不管是移除 Messages 还是移除 Runnable，流程都是一样的：

- 先从头消息开始进行第一阶段的匹配，如果能够匹配，就移除头消息，并更新链表头！
- 当头消息无法匹配，那么我们就删除剩下的消息中能够匹配的消息！

## 6.2 MessageQueue.removeCallbacksAndMessages

```java
    void removeCallbacksAndMessages(Handler h, Object object) {
        if (h == null) {
            return;
        }

        synchronized (this) {
            Message p = mMessages;

            // Remove all messages at front.
            while (p != null && p.target == h
                    && (object == null || p.obj == object)) {
                Message n = p.next;
                mMessages = n;
                p.recycleUnchecked(); // 非安全回收！
                p = n;
            }

            // Remove all messages after front.
            while (p != null) {
                Message n = p.next;
                if (n != null) {
                    if (n.target == h && (object == null || n.obj == object)) {
                        Message nn = n.next;
                        n.recycleUnchecked(); // 非安全回收！
                        p.next = nn;
                        continue;
                    }
                }
                p = n;
            }
        }
    }
```

## 6.3 MessageQueue.removeAllMessagesLocked

移除消息队列中所有的 Message！

```java
    private void removeAllMessagesLocked() {
        Message p = mMessages;
        while (p != null) {
            Message n = p.next;
            p.recycleUnchecked(); // 回收消息！
            p = n;
        }
        mMessages = null; // 设置 mMessages 为 null；
    }
```
方法很简单，不多说！

## 6.4 MessageQueue.removeAllFutureMessagesLocked

移除消息队列中当前时间下所有未分发的 Message！

```java
    private void removeAllFutureMessagesLocked() {
        // 计算当前时间！
        final long now = SystemClock.uptimeMillis();
        Message p = mMessages;
        if (p != null) {
            if (p.when > now) {
                // 如果消息链表中所有消息都没有分发，那就移除所有消息！
                removeAllMessagesLocked();
            } else {
                Message n;
                // 找到第一个还没到分发时间的消息！
                for (;;) {
                    n = p.next;
                    if (n == null) {
                        return;
                    }
                    if (n.when > now) {
                        break;
                    }
                    p = n;
                }
                p.next = null;
                // 移除所有未分发的消息！
                do {
                    p = n;
                    n = p.next;
                    p.recycleUnchecked(); // 非安全回收！
                } while (n != null);
            }
        }
    }
```
方法很简单，不多说！！

# 7 障栅 Barrier

我们知道障栅本质上是一个特殊的 Message，其 target 为 null，他能够**拦截同步的消息，放行异步消息**！！

我们关心的是如何向消息队列中插入和移除障栅！！

## 7.1 MessageQueue.postSyncBarrier

MessageQueue 提供了 postSyncBarrier 来向消息队列中增加障栅 Barrier!

```java
    public int postSyncBarrier() {
        return postSyncBarrier(SystemClock.uptimeMillis());
    }

    private int postSyncBarrier(long when) {
        synchronized (this) {
            // 为障栅计算 token，mNextBarrierToken自增！
            final int token = mNextBarrierToken++;
            final Message msg = Message.obtain(); // 优先复用 message！
            msg.markInUse();
            msg.when = when;
            msg.arg1 = token; // 设置 msg.arg1 为 token！

            // 将障栅添加到合适的位置，可能是表头，也可能是中间某个节点！
            Message prev = null;
            Message p = mMessages;
            if (when != 0) {
                while (p != null && p.when <= when) {
                    prev = p;
                    p = p.next;
                }
            }
            if (prev != null) { // invariant: p == prev.next
                msg.next = p;
                prev.next = msg;
            } else {
                // 将障珊作为新的表头！
                msg.next = p;
                mMessages = msg;
            }
            return token;
        }
    }
```
参数 when 表示障栅的拦截时间点！

可以看到，当我们插入障栅后，其要么是位于消息队列的头，要么是根据拦截时间 when，将障栅插入到消息队列的合适位置！

这样，障栅就可以拦截其后的所有同步消息了！


## 7.2 MessageQueue.removeSyncBarrier

MessageQueue 提供了 removeSyncBarrier 来从消息队列中移除障栅 Barrier：

参数 token 用于识别指定的障珊！

```java
    public void removeSyncBarrier(int token) {
        synchronized (this) {
            Message prev = null;
            Message p = mMessages;
            // 顺序遍历链表，如果 p.target 不为 null，说明它不是障珊；
            // 如果 p.arg1 != token 说明他不是我们要删除的障珊；
            while (p != null && (p.target != null || p.arg1 != token)) {
                prev = p;
                p = p.next;
            }
            if (p == null) { // 如果队列中没有障珊，会抛出异常！
                throw new IllegalStateException("The specified message queue synchronization "
                        + " barrier token has not been posted or has already been removed.");
            }
            final boolean needWake;
            if (prev != null) {
                // pre 不为 null，说明障栅在消息队列的中间某个节点！
                prev.next = p.next;
                needWake = false; // 这种情况不需要唤醒！
            } else {
                // pre 为 null 说明障栅就是消息队列的头，那么设置新的头节点为下一个 message！
                mMessages = p.next;
                
                needWake = mMessages == null || mMessages.target != null;
            }
            p.recycleUnchecked(); // 回收障栅，加入消息池！

            // 如果需要唤醒 native 层队列，并且 java 层队列没有关闭，那就唤醒！
            if (needWake && !mQuitting) {
                nativeWake(mPtr);
            }
        }
    }
```
逻辑很简单，就是遍历消息队列，根据传入的 token，匹配合适的障栅！！

我们为什么要 remove 掉障栅？

道理很简单，**由于障栅是一种特殊的 message，其 target 为 null，所以其不能被分发，意味着如果障栅后面没有异步消息，那么整个队列就会一直阻塞下去！！**

# 8 Native 层分析

上面分析了 java 层的 MessageQueue 逻辑架构，但我们早已经知道 native 也有个 MessageQueue，java 层的消息队列可通过以下方法和 native 层的 MessageQueue 通信：
```java
    private native static long nativeInit();
    private native static void nativeDestroy(long ptr);
    private native void nativePollOnce(long ptr, int timeoutMillis); /*non-static for callbacks*/
    private native static void nativeWake(long ptr);
    private native static boolean nativeIsPolling(long ptr);
    private native static void nativeSetFileDescriptorEvents(long ptr, int fd, int events);
```

native 层的 MessageQueue 除了可以让 native 层实现消息通信机制，更重要的是，其能够保证 java 层的不会陷入死循环！！

对应的 native 层方法位于 android/frameworks/base/core/jni/android_os_MessageQueue.cpp 文件中！

```java
static const JNINativeMethod gMessageQueueMethods[] = {
    /* name, signature, funcPtr */
    { "nativeInit", "()J", (void*)android_os_MessageQueue_nativeInit },
    { "nativeDestroy", "(J)V", (void*)android_os_MessageQueue_nativeDestroy },
    { "nativePollOnce", "(JI)V", (void*)android_os_MessageQueue_nativePollOnce },
    { "nativeWake", "(J)V", (void*)android_os_MessageQueue_nativeWake },
    { "nativeIsPolling", "(J)Z", (void*)android_os_MessageQueue_nativeIsPolling },
    { "nativeSetFileDescriptorEvents", "(JII)V",
            (void*)android_os_MessageQueue_nativeSetFileDescriptorEvents },
};

int register_android_os_MessageQueue(JNIEnv* env) {
    int res = RegisterMethodsOrDie(env, "android/os/MessageQueue", gMessageQueueMethods,
                                   NELEM(gMessageQueueMethods));

    jclass clazz = FindClassOrDie(env, "android/os/MessageQueue");
    gMessageQueueClassInfo.mPtr = GetFieldIDOrDie(env, clazz, "mPtr", "J");
    gMessageQueueClassInfo.dispatchEvents = GetMethodIDOrDie(env, clazz,
            "dispatchEvents", "(II)I");

    return res;
}
```
register_android_os_MessageQueue 用来注册静态方法！

## 8.1 android_os_MessageQueue_nativeInit

我们来看一下这个 nativeInit 对应的 native 方法：

```java
static jlong android_os_MessageQueue_nativeInit(JNIEnv* env, jclass clazz) {
    //【1】创建 native 消息队列：NativeMessageQueue！
    NativeMessageQueue* nativeMessageQueue = new NativeMessageQueue();
    if (!nativeMessageQueue) {
        jniThrowRuntimeException(env, "Unable to allocate native queue");
        return 0;
    }

    nativeMessageQueue->incStrong(env);
    return reinterpret_cast<jlong>(nativeMessageQueue);
}
```
这里创建了 native 层的消息队列：NativeMessageQueue！

```Java
NativeMessageQueue::NativeMessageQueue() :
        mPollEnv(NULL), mPollObj(NULL), mExceptionObj(NULL) {
    mLooper = Looper::getForThread();
    if (mLooper == NULL) {
        mLooper = new Looper(false);
        Looper::setForThread(mLooper);
    }
}
```


## 8.1 android_os_MessageQueue_nativePollOnce


```java
static void android_os_MessageQueue_nativePollOnce(JNIEnv* env, jobject obj,
        jlong ptr, jint timeoutMillis) {
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    nativeMessageQueue->pollOnce(env, obj, timeoutMillis);
}
```


# 9 总结

下面我们来总结下 MessageQueue 的类图！