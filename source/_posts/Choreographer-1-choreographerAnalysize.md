#  Choreographer 第一篇 Choreographer 原理分析

title: Choreographer 第一篇 Choreographer  原理分析
date: 2018/07/15 20:46:25 
catalog: true
categories: 

- View 视图
- Choreographer 编舞者

tags: Choreographer
copyright: true

------

本篇文章基于 Android N（7.1.1）主要分析下 Choreographer 的原理，以对 Android 系统有更好的理解。

# 1 回顾

## 1.1 scheduleTraversals - 核心

触发视图遍历：

```java
    void scheduleTraversals() {
        if (!mTraversalScheduled) {
            //【1】表示是否已经发起重绘，这是要设置为 true；
            mTraversalScheduled = true;
            //【1】在主线程的消息队列中放一个障栅；
            mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
            //【-->10.5】设置一个回调到编舞者 Choreographer 中，在下一次的绘制触发时，执行 mTraversalRunnable
            // mTraversalRunnable 是一个 runnbale 实例；
            mChoreographer.postCallback(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
            if (!mUnbufferedInputDispatch) {
                scheduleConsumeBatchedInput();
            }
            notifyRendererOfFramePending();
            pokeDrawLockIfNeeded();
        }
    }
```

前面分析 View 的绘制的时候，有讲到，Choreographer 会请求 Vsync 信号，然后在 Vsync 信号触发后，执行布局和绘制任务！

这一篇，我们来分析下 Choreographer！



# 2 什么是 Choreographer

（这一部分来自 **SDK** 的翻译，其实 SDK 的注释已经将 Choreographer 的作用讲的很清楚了）

编舞者，作用是协调动画，输入和绘图的时间；



编舞者会从显示子系统接收定时脉冲（也就是垂直同步 Vsync 信号），然后触发指定的工作，结果会作为渲染下一个显示帧的一部分；

应用一般使用动画框架或视图层次结构中的更高级别的抽象间接地和编舞者交互。



下面是一些可以使用的高级 API，用于和 Choreographer  间接通信：

- **ValueAnimator.start**：用于使用与显示框架渲染同步的要在常规时间进行处理的动画

- **View.postOnAnimation**：传入一个 Runnable，在下一个显示帧的开头被调用一次；
- **View.postOnAnimationDelayed**：传入一个 Runnable，在下一个显示帧的开头延迟指定时间被调用一次；
- **View.postInvalidateOnAnimation**：在下一个显示帧开始时触发 **View.invalidate** 一次；

 

为确保 **View** 的内容平滑滚动并与显示框架渲染同步绘制，请不要执行任何操作。系统会自动处理，**View.onDraw** 将在适当的时候被调用。

但是，在某些情况下，您可能希望直接在应用程序中使用编舞者的功能。譬如说：

- **Choreographer.postFrameCallback**：如果应用使用 GL，或完全不使用动画框架或视图层次结构在其他线程中进行渲染，并且你想确保它与显示适当同步。



每个  **Looper 线程** 都有自己的编舞者。其他线程可以发布回调以在编舞者上运行，但是它们将持有编舞者所属的 **Looper**。



# 3 Choreographer

我们来开始分析 Choreographer 的代码：

## 3.1 getInstance()

编舞者是线程单例模式，每一个线程都会有一个：**ThreadLocal**

```java
public final class Choreographer {
   //【1】通过 ThreadLocal 实现线程单例模式；
   private static final ThreadLocal<Choreographer> sThreadInstance =
            new ThreadLocal<Choreographer>() {
        @Override
        protected Choreographer initialValue() {
            //【2】获取当前线程的 looper 对象；
            Looper looper = Looper.myLooper();
            if (looper == null) {
                throw new IllegalStateException("The current thread must have a looper!");
            }
            //【-->3.2】创建 Choreographer 对象；
            Choreographer choreographer = new Choreographer(looper, VSYNC_SOURCE_APP);
            if (looper == Looper.getMainLooper()) {
                //【3】如果 looper 是 ui thread 的，会保存到内部的 mMainInstance；
                mMainInstance = choreographer;
            }
            return choreographer;
        }
    };
  
    public static Choreographer getInstance() {
        return sThreadInstance.get();
    }
}
```



## 3.2 new Choreographer

创建 Choreographer:

```java
    private Choreographer(Looper looper, int vsyncSource) {
        //【1】线程的 looper 实例；
        mLooper = looper;
        //【-->6.1】创建 FrameHandler 对象，用于处理消息；
        mHandler = new FrameHandler(looper);
        //【2】创建 VSYNC 的信号接受对象；
        mDisplayEventReceiver = USE_VSYNC
                ? new FrameDisplayEventReceiver(looper, vsyncSource)
                : null;
        //【3】初始化上一个 frame 渲染的时间点
        mLastFrameTimeNanos = Long.MIN_VALUE;
        //【4】计算帧率，也就是一帧所需的渲染时间，getRefreshRate 是刷新率，一般是 60；
        mFrameIntervalNanos = (long)(1000000000 / getRefreshRate());
        //【5】创建消息处理队列
        mCallbackQueues = new CallbackQueue[CALLBACK_LAST + 1];
        for (int i = 0; i <= CALLBACK_LAST; i++) {
            mCallbackQueues[i] = new CallbackQueue();
        }
        // b/68769804: For low FPS experiments.
        setFPSDivisor(SystemProperties.getInt(ThreadedRenderer.DEBUG_FPS_DIVISOR, 1));
    }
```

USE_VSYNC 的值来自下面：

```java
//【1】判断系统是否打开了 vsync，读取 "debug.choreographer.vsync" 属性；
private static final boolean USE_VSYNC = SystemProperties.getBoolean(
                   "debug.choreographer.vsync", true);
```

如果开启了 Vsync，就会创建一个 FrameDisplayEventReceiver 实例，用于请求并接收 Vsync 事件：

Choreographer 创建了一个大小为 3 的 CallbackQueue 数组，用于保存不同类型的 Callback；

```java
public static final int CALLBACK_COMMIT = 3;
private static final int CALLBACK_LAST = CALLBACK_COMMIT;
```



## 3.3 postCallback\[Delayed][DelayedInternal]

callbackType 表示回调的类型！

```java
public void postCallback(int callbackType, Runnable action, Object token) {
    //【1】继续调用；
    postCallbackDelayed(callbackType, action, token, 0);
}
```

进入 postCallbackDelayed 方法：

```java
public void postCallbackDelayed(int callbackType,
                                Runnable action, Object token, long delayMillis) {
    ... ... ...// 省略对 action 和 callbackType 的判断；

    postCallbackDelayedInternal(callbackType, action, token, delayMillis);
}
```

进入 postCallbackDelayedInternal 方法：

```java
private void postCallbackDelayedInternal(int callbackType,
                                         Object action, Object token, long delayMillis) {
    if (DEBUG_FRAMES) {
        Log.d(TAG, "PostCallback: type=" + callbackType
              + ", action=" + action + ", token=" + token
              + ", delayMillis=" + delayMillis);
    }

    synchronized (mLock) {
        final long now = SystemClock.uptimeMillis();
        final long dueTime = now + delayMillis; // delayMillis 是我们设置的延迟，这里为 0；
        //【1】可以看到其将 action 根据 callbackType 放入了 mCallbackQueues 数组中；
        mCallbackQueues[callbackType].addCallbackLocked(dueTime, action, token);

        if (dueTime <= now) {
            //【-->3.4】延迟时间到了，请求一个 Vsync 信号；
            scheduleFrameLocked(now);
        } else {
            //【-->6.X】时间没到，那就创建一个 MSG_DO_SCHEDULE_CALLBACK 的异步消息，
            // 用于异步执行 action 任务，延迟 dueTime 发送给 FrameHandler 处理；
            Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_CALLBACK, action);
            msg.arg1 = callbackType;
            msg.setAsynchronous(true); // 注意：这是异步的！
            mHandler.sendMessageAtTime(msg, dueTime);
        }
    }
}
```

### 3.3.1 CallbackQueues

CallbackQueues 是一个用于链表实现的队列，用于保存每种类型的回调链表：

```java
   private final class CallbackQueue {
        //【-->3.3.2】回调链表的 head；
        private CallbackRecord mHead;
   		... ... ...// 省略掉操作链表的方法；
   }
```

CallbackQueue 数组由三种类型不同 CallbackQueue，每个都有一个 CallbackRecord 链表，链表按照任务触发时间由小到大排列。

```java
public static final int CALLBACK_INPUT = 0;
public static final int CALLBACK_ANIMATION = 1;
public static final int CALLBACK_TRAVERSAL = 2;
```

内部的 CallbackRecord 以链表组织，以执行时间

#### 3.3.1.1 extractDueCallbacksLocked

返回要执行的 CallbackRecord 子链表：

```java
public CallbackRecord extractDueCallbacksLocked(long now) {
    CallbackRecord callbacks = mHead;
    if (callbacks == null || callbacks.dueTime > now) {
        return null;
    }
    //【1】last/next 指针；用于断开链表；
    CallbackRecord last = callbacks;
    CallbackRecord next = last.next;
    while (next != null) {
        //【2】找到执行时间晚于 now 的了，断开；
        if (next.dueTime > now) {
            last.next = null;
            break;
        }
        last = next;
        next = next.next;
    }
    mHead = next;
    //【3】返回执行时间早于 now 的子链表；
    return callbacks;
}
```

#### 3.3.1.2 addCallbackLocked

添加一个 CallbackRecord 到链表：

```java
        public void addCallbackLocked(long dueTime, Object action, Object token) {
            //【1】返回一个可复用的 CallbackRecord 实例；
            // obtainCallbackLocked 方法会返回这个复用的实例，代码简单就不多数了；
            CallbackRecord callback = obtainCallbackLocked(dueTime, action, token);
            CallbackRecord entry = mHead;
            if (entry == null) {
                mHead = callback;
                return;
            }
            //【2】判断下 head；
            if (dueTime < entry.dueTime) {
                callback.next = entry;
                mHead = callback;
                return;
            }
            //【3】遍历链表，找到合适的地方插入；
            while (entry.next != null) {
                if (dueTime < entry.next.dueTime) {
                    callback.next = entry.next;
                    break;
                }
                entry = entry.next;
            }
            entry.next = callback;
        }
```

编舞者内部有一个 mCallbackPool 实例，表示一个可复用的 CallbackRecord 对象；

```java
    private CallbackRecord mCallbackPool;
```

**obtainCallbackLocked** 方法会返回这个复用的实例：

- 如果 mCallbackPool 不为 null，就设置值，返回；
- 如果 mCallbackPool 为 null，初始化新的，再设置值，返回；



#### 3.3.2.3 hasDueCallbacksLocked

判断是否有已经触发的回调：

```java
        public boolean hasDueCallbacksLocked(long now) {
            //【1】就是比较下时间；
            return mHead != null && mHead.dueTime <= now;
        }
```



### 3.3.2 CallbackRecord

每一个回调：

```java
    private static final class CallbackRecord {
        public CallbackRecord next; // 指向下一个对象；
        public long dueTime;
        public Object action; // Runnable or FrameCallback
        public Object token;

        public void run(long frameTimeNanos) {
            if (token == FRAME_CALLBACK_TOKEN) {
                ((FrameCallback)action).doFrame(frameTimeNanos);
            } else {
                ((Runnable)action).run();
            }
        }
    }
```

根据 token 不同，执行不同的处理：

- **FrameCallback** 的 token 是 FRAME_CALLBACK_TOKEN
- **Runnable** 的 token 为 null；



## 3.4 scheduleFrameLocked

请求一个 Vsync 信号：

```java
    private void scheduleFrameLocked(long now) {
        if (!mFrameScheduled) {
            //【1】设置 mFrameScheduled 为 true；
            mFrameScheduled = true;
            if (USE_VSYNC) { 
                //【1】如果开启了 Vsync（默认开启）
                if (DEBUG_FRAMES) {
                    Log.d(TAG, "Scheduling next frame on vsync.");
                }

                // If running on the Looper thread, then schedule the vsync immediately,
                // otherwise post a message to schedule the vsync from the UI thread
                // as soon as possible.
                // 这里会判断下 ？Looper.myLooper() == mLooper，如果一样;
                if (isRunningOnLooperThreadLocked()) {
                    //【-->3.4.1】立刻请求 Vsync 信号；
                    scheduleVsyncLocked();
                } else {
                    //【-->6.1】发送一个 异步 msg[MSG_DO_SCHEDULE_VSYNC] 给 Framehandler;
					// 这个异步消息会在 message queue 的队头；
                    Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_VSYNC);
                    msg.setAsynchronous(true);
                    mHandler.sendMessageAtFrontOfQueue(msg);
                }
            } else {
                //【2】没有开启了 Vsync，那么这里会手动计算一个 delay 时间；
                final long nextFrameTime = Math.max(
                        mLastFrameTimeNanos / TimeUtils.NANOS_PER_MS + sFrameDelay, now);
                if (DEBUG_FRAMES) {
                    Log.d(TAG, "Scheduling next frame in " + (nextFrameTime - now) + " ms.");
                }
                //【-->6.X】发送一个 异步 msg[MSG_DO_FRAME] 给 Framehander；
                Message msg = mHandler.obtainMessage(MSG_DO_FRAME);
                msg.setAsynchronous(true);
                mHandler.sendMessageAtTime(msg, nextFrameTime);
            }
        }
    }
```

### 3.4.1 scheduleVsyncLocked

```java
    private void scheduleVsyncLocked() {
        //【-->4.2】通过 FrameDisplayEventReceiver 请求 vSync 信号；
        mDisplayEventReceiver.scheduleVsync();
    }
```

# 4  FrameDisplayEventReceiver

FrameDisplayEventReceiver 用于请求和接受 vsync 信号：

## 4.1 new FrameDisplayEventReceiver

FrameDisplayEventReceiver 继承了 **DisplayEventReceiver**，同时注意，也是**实现了 Runnable**：

```java
private final class FrameDisplayEventReceiver extends DisplayEventReceiver
            implements Runnable {
   private boolean mHavePendingVsync;
   private long mTimestampNanos;
   private int mFrame;

   public FrameDisplayEventReceiver(Looper looper, int vsyncSource) {
        super(looper, vsyncSource);
   }
   ... ... ...
}
```

我们来看看其父类 **DisplayEventReceiver**：

```java
public DisplayEventReceiver(Looper looper, int vsyncSource) {
    if (looper == null) {
        throw new IllegalArgumentException("looper must not be null");
    }
    //【1】获取到当前 looper 的 MessageQueue；
    mMessageQueue = looper.getQueue();
    //【-->4.1.1】创建 NativeDisplayEventReceiver；
    mReceiverPtr = nativeInit(new WeakReference<DisplayEventReceiver>(this), mMessageQueue,
                              vsyncSource);

    mCloseGuard.open("dispose");
}
```

**NativeDisplayEventReceiver** 中会获取 Java MessageQueue 对应的 NativeMessageQueue，这个和前面的 input 很类似了；

这里的 nativeInit 方法位于

```java
/frameworks/base/core/jni/android_view_DisplayEventReceiver.cpp
```



### 4.1.1 nativeInit

用于初始化 NativeDisplayEventReceiver


```java
static jlong nativeInit(JNIEnv* env, jclass clazz, jobject receiverObj,
        jobject messageQueueObj) {
    //【1】获取 NativeMessageQueue 实例；
    sp<MessageQueue> messageQueue = android_os_MessageQueue_getMessageQueue(env, messageQueueObj);
    if (messageQueue == NULL) {
        jniThrowRuntimeException(env, "MessageQueue is not initialized.");
        return 0;
    }

    //【-->5.1】创建 NativeDisplayEventReceiver 实例；
    sp<NativeDisplayEventReceiver> receiver = new NativeDisplayEventReceiver(env,
            receiverObj, messageQueue);
    //【-->5.2】初始化 receiver；
    status_t status = receiver->initialize();
    if (status) {
        String8 message;
        message.appendFormat("Failed to initialize display event receiver.  status=%d", status);
        jniThrowRuntimeException(env, message.string());
        return 0;
    }

    //【2】增加强引用计数；
    receiver->incStrong(gDisplayEventReceiverClassInfo.clazz); // retain a reference for the object
    return reinterpret_cast<jlong>(receiver.get());
}
```







## 4.2 scheduleVsync

请求一个 Vsync 信号，父类方法：

```java
    public void scheduleVsync() {
        if (mReceiverPtr == 0) {
            Log.w(TAG, "Attempted to schedule a vertical sync pulse but the display event "
                    + "receiver has already been disposed.");
        } else {
            //【-->4.2.1】进入 native 层；
            nativeScheduleVsync(mReceiverPtr);
        }
    }
```



这里的 nativeScheduleVsync 方法位于

```java
/frameworks/base/core/jni/android_view_DisplayEventReceiver.cpp
```



### 4.2.1 nativeScheduleVsync

```c++
static void nativeScheduleVsync(JNIEnv* env, jclass clazz, jlong receiverPtr) {
    sp<NativeDisplayEventReceiver> receiver =
            reinterpret_cast<NativeDisplayEventReceiver*>(receiverPtr);
    //【-->5.3】进入 native 的 DisplayEventReceiver 请求 Vsync 信号；
    status_t status = receiver->scheduleVsync();
    if (status) {
        String8 message;
        message.appendFormat("Failed to schedule next vertical sync pulse.  status=%d", status);
        jniThrowRuntimeException(env, message.string());
    }
}
```





## 4.3 dispatchVsync

这个方法是在父类里面：

```c++
// Called from native code.
@SuppressWarnings("unused")
private void dispatchVsync(long timestampNanos, int builtInDisplayId, int frame) {
    //【-->4.4】处理 Vsync 信号
	onVsync(timestampNanos, builtInDisplayId, frame);
}
```

## 4.4 onVsync

处理 Vsync 信号；

```java
@Override
public void onVsync(long timestampNanos, int builtInDisplayId, int frame) {
    //【1】忽略来自第二显示屏的 Vsync 信号；
    if (builtInDisplayId != SurfaceControl.BUILT_IN_DISPLAY_ID_MAIN) {
        Log.d(TAG, "Received vsync from secondary display, but we don't support "
              + "this case yet.  Choreographer needs a way to explicitly request "
              + "vsync for a specific display to ensure it doesn't lose track "
              + "of its scheduled vsync.");
        //【-->5.3】请求下一个 Vsync；
        scheduleVsync();
        return;
    }

    // Post the vsync event to the Handler.
    // The idea is to prevent incoming vsync events from completely starving
    // the message queue.  If there are no messages in the queue with timestamps
    // earlier than the frame time, then the vsync event will be processed immediately.
    // Otherwise, messages that predate the vsync event will be handled first.
    //【1】调整 Vsync 时间；
    long now = System.nanoTime();
    if (timestampNanos > now) {
        Log.w(TAG, "Frame time is " + ((timestampNanos - now) * 0.000001f)
              + " ms in the future!  Check that graphics HAL is generating vsync "
              + "timestamps using the correct timebase.");
        timestampNanos = now;
    }

    //【2】设置 mHavePendingVsync 为 ture，表示正在有一个处理中的 Vsync 信号；
    if (mHavePendingVsync) {
        Log.w(TAG, "Already have a pending vsync event.  There should only be "
              + "one at a time.");
    } else {
        mHavePendingVsync = true;
    }
    //【-->6.X】发送一个消息给 FrameHandler，该消息的 callback 为当前对象 FrameDisplayEventReceiver；
    mTimestampNanos = timestampNanos;
    mFrame = frame;
    Message msg = Message.obtain(mHandler, this);
    msg.setAsynchronous(true); // 注意：这是异步消息；
    mHandler.sendMessageAtTime(msg, timestampNanos / TimeUtils.NANOS_PER_MS);
}
```

还记得为什么是异步消息吗？

这是因为，Vsync 信号是优先级很高的信号，所以，要优先处理他，这里将其设置成异步的。

好处是可以通过设置障栅，阻塞同步，优先处理异步消息，这个在 view draw 中有使用到；



**重点：**

- 当 Vsync 请求到后，这里会把 **FrameDisplayEventReceiver** 最为 callback，通过 msg 发送给 Handler；



## 4.5 run

执行任务：

```java
@Override
public void run() {
	mHavePendingVsync = false;
    //【-->7.2】处理这一帧！
	doFrame(mTimestampNanos, mFrame);
}
```



# 5 NativeDisplayEventReceiver - 核心

我们去看看 native 的逻辑：



## 5.1 create NativeDisplayEventReceiver

NativeDisplayEventReceiver 继承 DisplayEventDispatcher：

```C++
#include <android_runtime/AndroidRuntime.h>
#include <androidfw/DisplayEventDispatcher.h>
#include <utils/Log.h>
#include <utils/Looper.h>
#include <utils/threads.h>
#include <gui/DisplayEventReceiver.h>
#include "android_os_MessageQueue.h"
    
class NativeDisplayEventReceiver : public DisplayEventDispatcher {
public:
    NativeDisplayEventReceiver(JNIEnv* env,
            jobject receiverWeak, const sp<MessageQueue>& messageQueue);

private:
    jobject mReceiverWeakGlobal; // 全局引用，指向 java 层的 FrameDisplayEventReceiver 实例；
    sp<MessageQueue> mMessageQueue; // NativeMessageQueue 实例；
    DisplayEventReceiver mReceiver; // DisplayEventReceiver 实例 frameworks/nivate/libs/gui/DisplayEventReceiver.cpp
    bool mWaitingForVsync; // 表示是否正在等待 Vsync，初始化为 false；
    
NativeDisplayEventReceiver::NativeDisplayEventReceiver(JNIEnv* env,
        jobject receiverWeak, const sp<MessageQueue>& messageQueue) :
        DisplayEventDispatcher(messageQueue->getLooper()), // 调用父类的构造器，传入 nativeMessageQueue 的 looper 对象；
        mReceiverWeakGlobal(env->NewGlobalRef(receiverWeak)),
        mMessageQueue(messageQueue), mWaitingForVsync(false) {
    ALOGV("receiver %p ~ Initializing display event receiver.", this);
}
```

去看看父类的构造器：

### 5.1.1 create DisplayEventDispatcher - 父类

参数 const sp<Looper>& looper 是 NativeMessageQueue 对应的 Looper；

```C++
class DisplayEventDispatcher : public LooperCallback {
public:
    DisplayEventDispatcher(const sp<Looper>& looper);

... ... ...

private:
    sp<Looper> mLooper; // NativeMessageQueue 对应的 Looper 实例；
    DisplayEventReceiver mReceiver;
    bool mWaitingForVsync;
    
DisplayEventDispatcher::DisplayEventDispatcher(const sp<Looper>& looper) :
        mLooper(looper), mWaitingForVsync(false) {
    ALOGV("dispatcher %p ~ Initializing display event dispatcher.", this);
}
```



## 5.2 initialize

初始化：

```c++
status_t DisplayEventDispatcher::initialize() {
    //【-->5..6.1】检查这里的 DisplayEventReceiver 是否被创建
    status_t result = mReceiver.initCheck();
    if (result) {
        ALOGW("Failed to initialize display event receiver, status=%d", result);
        return result;
    }
    //【2】使用 NativeMessageQueue 的 Looper 对象监听 
    //【-->5..6.3】mReceiver.getFd 返回的文件句柄，这是 fd 是 bittube 的文件句柄。
    // 同时将自己作为回调！
    int rc = mLooper->addFd(mReceiver.getFd(), 0, Looper::EVENT_INPUT,
            this, NULL);
    if (rc < 0) {
        return UNKNOWN_ERROR;
    }
    return OK;
}
```

## 5.3 scheduleVsync

请求 Vsync 信号：

```C++
status_t DisplayEventDispatcher::scheduleVsync() {
    if (!mWaitingForVsync) {
        ALOGV("dispatcher %p ~ Scheduling vsync.", this);

        // Drain all pending events.
        nsecs_t vsyncTimestamp;
        int32_t vsyncDisplayId;
        uint32_t vsyncCount;
        if (processPendingEvents(&vsyncTimestamp, &vsyncDisplayId, &vsyncCount)) {
            ALOGE("dispatcher %p ~ last event processed while scheduling was for %" PRId64 "",
                    this, ns2ms(static_cast<nsecs_t>(vsyncTimestamp)));
        }
        //【-->5..6.4】请求下一次 Vsync 信息!
        status_t status = mReceiver.requestNextVsync();
        if (status) {
            ALOGW("Failed to request next vsync, status=%d", status);
            return status;
        }

        mWaitingForVsync = true;
    }
    return OK;
}
```



## 5.4 handleEvent

前面我们看到，native Looper 监听指定的 fd，当 fd 有事件写入后，handleEvent 就会触发：

```c++
int DisplayEventDispatcher::handleEvent(int, int events, void*) {
    if (events & (Looper::EVENT_ERROR | Looper::EVENT_HANGUP)) {
        ALOGE("Display event receiver pipe was closed or an error occurred.  "
                "events=0x%x", events);
        return 0; // remove the callback
    }

    if (!(events & Looper::EVENT_INPUT)) {
        ALOGW("Received spurious callback for unhandled poll event.  "
                "events=0x%x", events);
        return 1; // keep the callback
    }

    nsecs_t vsyncTimestamp;
    int32_t vsyncDisplayId;
    uint32_t vsyncCount;
    //【-->5.4.1】获取最后一次的 Vsync。
    if (processPendingEvents(&vsyncTimestamp, &vsyncDisplayId, &vsyncCount)) {
        ALOGV("dispatcher %p ~ Vsync pulse: timestamp=%" PRId64 ", id=%d, count=%d",
                this, ns2ms(vsyncTimestamp), vsyncDisplayId, vsyncCount);
        mWaitingForVsync = false;
        //【-->5.5】分发最后一次的 Vsync。
        dispatchVsync(vsyncTimestamp, vsyncDisplayId, vsyncCount);
    }

    return 1; // keep the callback
}
```

### 5.4.1 processPendingEvents

前面我们看到，Native Looper 监听指定的 fd，当 fd 由事件写入后，handleEvent 就会触发：

```java
bool DisplayEventDispatcher::processPendingEvents(
        nsecs_t* outTimestamp, int32_t* outId, uint32_t* outCount) {
    bool gotVsync = false;
    DisplayEventReceiver::Event buf[EVENT_BUFFER_SIZE];
    ssize_t n;
    //【-->[5..6.5]】通过 mReceiver.getEvents 获取 event，保存到 buf 中；
    // mReceiver 就是 DisplayEventReceiver 对象；
    while ((n = mReceiver.getEvents(buf, EVENT_BUFFER_SIZE)) > 0) {
        ALOGV("dispatcher %p ~ Read %d events.", this, int(n));
        for (ssize_t i = 0; i < n; i++) {
            const DisplayEventReceiver::Event& ev = buf[i];
            //【2】判断 event 的 type；
            switch (ev.header.type) {
            case DisplayEventReceiver::DISPLAY_EVENT_VSYNC:
                //【3】最新的 Vsync 将会覆盖之前的消息，也就是说我们获取的是最近的那个；
                // 并设置 gotVsync 为 true！
                gotVsync = true;
                *outTimestamp = ev.header.timestamp;
                *outId = ev.header.id;
                *outCount = ev.vsync.count;
                break;
            case DisplayEventReceiver::DISPLAY_EVENT_HOTPLUG:
                dispatchHotplug(ev.header.timestamp, ev.header.id, ev.hotplug.connected);
                break;
            default:
                ALOGW("dispatcher %p ~ ignoring unknown event type %#x", this, ev.header.type);
                break;
            }
        }
    }
    if (n < 0) {
        ALOGW("Failed to get events from display event dispatcher, status=%d", status_t(n));
    }
    return gotVsync; // 返回 gotVsync；
}
```

## 5.5 dispatchVsync

分发 Vsync 信号：

```c++
void NativeDisplayEventReceiver::dispatchVsync(nsecs_t timestamp, int32_t id, uint32_t count) {
    JNIEnv* env = AndroidRuntime::getJNIEnv();
    //【1】这个就是 java 层的 FrameDisplayEventReceiver 实例；
    ScopedLocalRef<jobject> receiverObj(env, jniGetReferent(env, mReceiverWeakGlobal));
    if (receiverObj.get()) {
        ALOGV("receiver %p ~ Invoking vsync handler.", this);
        //【-->4.3】调用 FrameDisplayEventReceiver 的 dispatchVsync 方法；
        env->CallVoidMethod(receiverObj.get(),
                gDisplayEventReceiverClassInfo.dispatchVsync, timestamp, id, count);
        ALOGV("receiver %p ~ Returned from vsync handler.", this);
    }

    mMessageQueue->raiseAndClearException(env, "dispatchVsync");
}
```



# [5..6] DisplayEventReceiver - 核心

在 NativeDisplayEventReceiver 中会创建 DisplayEventReceiver 对象；

但是很奇怪，我在 N 的代码里面没有找到 create DisplayEventReceiver 的地方，倒是 O 的代码里面有：

但是这里依然要看下创建的过程；

## [5..6].1 create DisplayEventReceiver

这里创建了 DisplayEventReceiver 实例：

```c++
DisplayEventReceiver::DisplayEventReceiver() {
    //【1】获取 SurfaceFlinger 的代理对象 BpSurfaceComposer
    sp<ISurfaceComposer> sf(ComposerService::getComposerService());
    if (sf != NULL) {
        //【2】创建了 EventConnection，同时初始化 DataChannel 实例，他是 BitTube 类型的，支持跨进程；
        // 这里会通过 IPC 进入 SF；
        mEventConnection = sf->createDisplayEventConnection();
        if (mEventConnection != NULL) {
            mDataChannel = mEventConnection->getDataChannel();
        }
    }
}
```

我们看下，这两个变量的类型：

```c++
// /frameworks/native/include/gui/DisplayEventReceiver.h
private:
    sp<IDisplayEventConnection> mEventConnection;
    sp<BitTube> mDataChannel; // 很重要，Vsync 信号最后就是通过 BitTube 传递上来的；
};
```



这里进行了 IPC 通信：/[frameworks](http://androidxref.com/7.1.1_r6/xref/frameworks/)/[native](http://androidxref.com/7.1.1_r6/xref/frameworks/native/)/[libs](http://androidxref.com/7.1.1_r6/xref/frameworks/native/libs/)/[gui](http://androidxref.com/7.1.1_r6/xref/frameworks/native/libs/gui/)/[ISurfaceComposer.cpp](http://androidxref.com/7.1.1_r6/xref/frameworks/native/libs/gui/ISurfaceComposer.cpp)，进入 SurfaceFlinger 进程：

### extend - createDisplayEventConnection

跨进程 binder 调用：

```c++
    virtual sp<IDisplayEventConnection> createDisplayEventConnection(VsyncSource vsyncSource)
    {
        Parcel data, reply;
        sp<IDisplayEventConnection> result;
        int err = data.writeInterfaceToken(
                ISurfaceComposer::getInterfaceDescriptor());
        if (err != NO_ERROR) {
            return result;
        }
        data.writeInt32(static_cast<int32_t>(vsyncSource));
        err = remote()->transact(
                BnSurfaceComposer::CREATE_DISPLAY_EVENT_CONNECTION,
                data, &reply);
        if (err != NO_ERROR) {
            ALOGE("ISurfaceComposer::createDisplayEventConnection: error performing "
                    "transaction: %s (%d)", strerror(-err), -err);
            return result;
        }
        result = interface_cast<IDisplayEventConnection>(reply.readStrongBinder());
        return result;
    }
```

而在  SF 中是通过 EventThread 线程创建的：

```c++
sp<IDisplayEventConnection> SurfaceFlinger::createDisplayEventConnection() {
   // 进入 EventThread 方法：
   return mEventThread->createEventConnection();
}
```

EventThread 是一个 native 层的线程：/[frameworks](http://androidxref.com/7.1.1_r6/xref/frameworks/)/[native](http://androidxref.com/7.1.1_r6/xref/frameworks/native/)/[services](http://androidxref.com/7.1.1_r6/xref/frameworks/native/services/)/[surfaceflinger](http://androidxref.com/7.1.1_r6/xref/frameworks/native/services/surfaceflinger/)/[EventThread.cpp](http://androidxref.com/7.1.1_r6/xref/frameworks/native/services/surfaceflinger/EventThread.cpp)

他的作用是接收 VSync 信号，并分发 VSync 信号给 Connection 对应的监听者；

VSync 类似于心跳，不断地会通知到 Surfaceflinger 中的 EventThread 线程，由后者再分发给感兴趣的注册者。

```c++
sp<EventThread::Connection> EventThread::createEventConnection() const {
    return new Connection(const_cast<EventThread*>(this));
}
```

创建 Connection 的方法如下：

```c++
// 创建了一个 Connection 对象；
EventThread::Connection::Connection(
        const sp<EventThread>& eventThread)
    : count(-1), mEventThread(eventThread), mChannel(new BitTube()) // 这里创建了一个字节管道
{
}
```

我们再去看看  Connection 类：

```c++
class EventThread : public Thread, private VSyncSource::Callback {
    //【1】Connection 是内部类，实现了 BnDisplayEventConnection，具有跨进程能力；
    class Connection : public BnDisplayEventConnection {
    public:
        Connection(const sp<EventThread>& eventThread);
        status_t postEvent(const DisplayEventReceiver::Event& event);

        // count >= 1 : continuous event. count is the vsync rate
        // count == 0 : one-shot event that has not fired
        // count ==-1 : one-shot event that fired this round / disabled
        int32_t count;

    private:
        virtual ~Connection();
        virtual void onFirstRef();
        virtual sp<BitTube> getDataChannel() const;
        virtual void setVsyncRate(uint32_t count);
        virtual void requestNextVsync();    // asynchronous
        sp<EventThread> const mEventThread; // mEventThread 实例的引用；
        sp<BitTube> const mChannel; // BitTube 实例的引用；
    };
```

创建完成 Connection 后，紧接着就会触发 onFirstRef 方法：

```c++
void EventThread::Connection::onFirstRef() {
    // NOTE: mEventThread doesn't hold a strong reference on us
    // 注册 connection 到 EventThread 中：
    mEventThread->registerDisplayEventConnection(this);
}
```

onFirstRef 属于其父类 RefBase，该函数在强引用 sp 新增引用计数时调用

```c++
status_t EventThread::registerDisplayEventConnection(
        const sp<EventThread::Connection>& connection) {
    Mutex::Autolock _l(mLock);
    mDisplayEventConnections.add(connection); // 添加到 mDisplayEventConnections 中去；
    mCondition.broadcast();
    return NO_ERROR;
}
```

这里我们先看这么多，后续再分析；

而对于 `sp<BitTube> mDataChannel`

```c++
sp<BitTube> EventThread::Connection::getDataChannel() const {
    // 返回 BitTube 给了 app 进程；
    return mChannel;
}
```



## [5..6].2 initCheck

初始化检查，判断 mDataChannel 是否创建；

```c++
status_t DisplayEventReceiver::initCheck() const {
    if (mDataChannel != NULL)
        return NO_ERROR;
    return NO_INIT;
}
```



## [5..6].3 getFd

获取 DataChannel 对应的文件描述符 fd 

```c++
int DisplayEventReceiver::getFd() const {
    if (mDataChannel == NULL)
        return NO_INIT;

    return mDataChannel->getFd();
}
```



## [5..6].4 requestNextVsync

请求下一个 Vsync 信号；

```c++
status_t DisplayEventReceiver::requestNextVsync() {
    if (mEventConnection != NULL) {
        // 这里是通过 EventConnection 请求下一个 Vsync 信号；
        mEventConnection->requestNextVsync();
        return NO_ERROR;
    }
    return NO_INIT;
}
```





## [5..6].5 getEvents

我们来看下 getEvents 方法：

```c++
ssize_t DisplayEventReceiver::getEvents(DisplayEventReceiver::Event* events,
        size_t count) {
    //【1】从 BitTube 中读取 Vsync 事件；
    // 这里的 mDataChannel 就是上面创建的；
    return DisplayEventReceiver::getEvents(mDataChannel, events, count);
}

ssize_t DisplayEventReceiver::getEvents(const sp<BitTube>& dataChannel,
        Event* events, size_t count)
{
    return BitTube::recvObjects(dataChannel, events, count);
}
```

看起来一切都串起来了呀～～



# 6 FrameHandler

FrameHandler 主要用于处理内部消息，触发响应机制：

## 6.1 new FrameHandler

```java
    private final class FrameHandler extends Handler {
        public FrameHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case MSG_DO_FRAME: 
                    //【1】这个消息是没有开启 VSYNC 的时候，java 层通过延迟模拟 Vsync 信号；
                    // 延迟发送 doFrame 消息；
                    //【-->7.2】处理这一帧；
                    doFrame(System.nanoTime(), 0);
                    break;
                case MSG_DO_SCHEDULE_VSYNC:
                    //【2】这个消息用于请求 Vsync 信号；
                    //【-->7.1】请求 Vsync 信号；
                    doScheduleVsync();
                    break;
                case MSG_DO_SCHEDULE_CALLBACK:
                    doScheduleCallback(msg.arg1);
                    break;
            }
        }
    }
```



# 7 Choreographer - back

这里我们再次回到了 Choreographer：

## 7.1 doScheduleVsync

doScheduleVsync 方法只是再次调用 scheduleVsyncLocked 方法；

```java
    void doScheduleVsync() {
        synchronized (mLock) {
            if (mFrameScheduled) {
                //【-->3.4.1】涛声依旧~~
                scheduleVsyncLocked();
            }
        }
    }
```

不多说了！



## 7.2 doFrame

处理当前帧：

```java
    void doFrame(long frameTimeNanos, int frame) {
        final long startNanos;
        synchronized (mLock) {
            if (!mFrameScheduled) {
                return; //【1】如果 mFrameScheduled 为 false，则不需要处理当前帧；
            }

            if (DEBUG_JANK && mDebugPrintNextFrameTimeDelta) {
                mDebugPrintNextFrameTimeDelta = false;
                Log.d(TAG, "Frame time delta: "
                        + ((frameTimeNanos - mLastFrameTimeNanos) * 0.000001f) + " ms");
            }

            long intendedFrameTimeNanos = frameTimeNanos; // 预期帧时间
            startNanos = System.nanoTime();
            //【2】计算时间差值：当前时间 - 帧触发的时间；
            final long jitterNanos = startNanos - frameTimeNanos;
            //【3】判断时间差值是否超过 mFrameIntervalNanos，说明此时已经不满足 16ms 一帧了；
            if (jitterNanos >= mFrameIntervalNanos) {
                //【3.1】计算下跳过了多少帧。
                final long skippedFrames = jitterNanos / mFrameIntervalNanos;
                if (skippedFrames >= SKIPPED_FRAME_WARNING_LIMIT) {
                    Log.i(TAG, "Skipped " + skippedFrames + " frames!  "
                            + "The application may be doing too much work on its main thread.");
                }
                //【3.2】余下的偏移量；
                final long lastFrameOffset = jitterNanos % mFrameIntervalNanos;
                if (DEBUG_JANK) {
                    Log.d(TAG, "Missed vsync by " + (jitterNanos * 0.000001f) + " ms "
                            + "which is more than the frame interval of "
                            + (mFrameIntervalNanos * 0.000001f) + " ms!  "
                            + "Skipping " + skippedFrames + " frames and setting frame "
                            + "time to " + (lastFrameOffset * 0.000001f) + " ms in the past.");
                }
                //【3.3】当前时间 - 余下的偏移量，作为本帧的实际时间；
                frameTimeNanos = startNanos - lastFrameOffset;
            }
            //【4】如果帧触发时间比上一帧的时间早，那就要重新请求 Vsync 信号；
            if (frameTimeNanos < mLastFrameTimeNanos) {
                if (DEBUG_JANK) {
                    Log.d(TAG, "Frame time appears to be going backwards.  May be due to a "
                            + "previously skipped frame.  Waiting for next vsync.");
                }
                //【-->3.4.1】重新请求 Vsync 信号；
                scheduleVsyncLocked();
                return;
            }

            if (mFPSDivisor > 1) { // 针对于低 FPs 的情况，这里没看懂；
                long timeSinceVsync = frameTimeNanos - mLastFrameTimeNanos;
                if (timeSinceVsync < (mFrameIntervalNanos * mFPSDivisor) && timeSinceVsync > 0) {
                    scheduleVsyncLocked();
                    return;
                }
            }
            //【-->8.2】保存帧信息（预期帧时间, 实际帧时间）
            mFrameInfo.setVsync(intendedFrameTimeNanos, frameTimeNanos);
            //【4】设置 mFrameScheduled 为 false；
            mFrameScheduled = false;
            //【5】保存当前帧时间到 mLastFrameTimeNanos；
            mLastFrameTimeNanos = frameTimeNanos;
        }
        //【6】核心来了，这里会执行 CallbackQueue 中的 CallbackRecord！
        try {
            Trace.traceBegin(Trace.TRACE_TAG_VIEW, "Choreographer#doFrame");
            AnimationUtils.lockAnimationClock(frameTimeNanos / TimeUtils.NANOS_PER_MS);
           
            //【-->8.3】标记 input 处理，动画开始，PerformTraversals 开始的时间
            mFrameInfo.markInputHandlingStart();
            //【-->7.3】执行 CALLBACK_INPUT 类型的 CallbackQueue 中的 CallbackRecord！
            doCallbacks(Choreographer.CALLBACK_INPUT, frameTimeNanos);

            mFrameInfo.markAnimationsStart(); // same;
            //【-->7.3】执行 CALLBACK_ANIMATION 类型的 CallbackQueue 中的 CallbackRecord！
            doCallbacks(Choreographer.CALLBACK_ANIMATION, frameTimeNanos);

            mFrameInfo.markPerformTraversalsStart(); // same;
            //【-->7.3】执行 CALLBACK_TRAVERSAL 类型的 CallbackQueue 中的 CallbackRecord！
            doCallbacks(Choreographer.CALLBACK_TRAVERSAL, frameTimeNanos);

            doCallbacks(Choreographer.CALLBACK_COMMIT, frameTimeNanos);
        } finally {
            AnimationUtils.unlockAnimationClock();
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }

        if (DEBUG_FRAMES) {
            final long endNanos = System.nanoTime();
            Log.d(TAG, "Frame " + frame + ": Finished, took "
                    + (endNanos - startNanos) * 0.000001f + " ms, latency "
                    + (startNanos - frameTimeNanos) * 0.000001f + " ms.");
        }
    }
```

FrameInfo 表示帧信息，是编舞者的内部成员：

```java
  FrameInfo mFrameInfo = new FrameInfo();
```

这里看到有一个对帧时间调整的操作：



- 预期帧时间，就是 Vsync 触发的时间，但是对于实际帧时间，可能因为做一些耗时操作，导致延后，错过多个帧时间周期；
- jitterNanos = startNanos - frameTimeNanos，就是计算错过的总时间；
- skippedFrames = jitterNanos / mFrameIntervalNanos，计算出了实际错过了帧数；
- lastFrameOffset = jitterNanos % mFrameIntervalNanos，计算出余下的不满一帧的时间，然后要做调整；
- frameTimeNanos = startNanos - lastFrameOffset，当前时间减去不满一帧的时间，保证相同的时间周期；



接下来就是核心了，和前面看的一样， 执行 CallbackQueue 中的 CallbackRecord，次序：

- Choreographer.CALLBACK_INPUT；
- Choreographer.CALLBACK_ANIMATION；
- Choreographer.CALLBACK_TRAVERSAL；



## 7.3 doCallbacks

执行 CallBack：

```java
    void doCallbacks(int callbackType, long frameTimeNanos) {
        CallbackRecord callbacks;
        synchronized (mLock) {
            // We use "now" to determine when callbacks become due because it's possible
            // for earlier processing phases in a frame to post callbacks that should run
            // in a following phase, such as an input event that causes an animation to start.
            final long now = System.nanoTime();
            //【-->3.3.1】返回指定类型 callbackType 对应的 CallbackRecord，
            // 通过 last/next 两个指针，找到 CallbackRecord 所有执行时间早于 now 的（也就是已经到执行时间的）
            // CallbackRecord, 以链表形式返回（头结点，断开链表）
            callbacks = mCallbackQueues[callbackType].extractDueCallbacksLocked(
                    now / TimeUtils.NANOS_PER_MS);
            if (callbacks == null) {
                return;
            }
            //【1】讲 mCallbacksRunning 设置为 true；
            mCallbacksRunning = true;

            // Update the frame time if necessary when committing the frame.
            // We only update the frame time if we are more than 2 frames late reaching
            // the commit phase.  This ensures that the frame time which is observed by the
            // callbacks will always increase from one frame to the next and never repeat.
            // We never want the next frame's starting frame time to end up being less than
            // or equal to the previous frame's commit frame time.  Keep in mind that the
            // next frame has most likely already been scheduled by now so we play it
            // safe by ensuring the commit time is always at least one frame behind.
            //【2】针对于最后一种类型 CALLBACK_COMMIT；会重新调整一次 mLastFrameTimeNanos。
            // CALLBACK_COMMIT 没有对应的 CallbackQueues；
            if (callbackType == Choreographer.CALLBACK_COMMIT) {
                final long jitterNanos = now - frameTimeNanos;
                Trace.traceCounter(Trace.TRACE_TAG_VIEW, "jitterNanos", (int) jitterNanos);
                if (jitterNanos >= 2 * mFrameIntervalNanos) {
                    final long lastFrameOffset = jitterNanos % mFrameIntervalNanos
                            + mFrameIntervalNanos;
                    if (DEBUG_JANK) {
                        Log.d(TAG, "Commit callback delayed by " + (jitterNanos * 0.000001f)
                                + " ms which is more than twice the frame interval of "
                                + (mFrameIntervalNanos * 0.000001f) + " ms!  "
                                + "Setting frame time to " + (lastFrameOffset * 0.000001f)
                                + " ms in the past.");
                        mDebugPrintNextFrameTimeDelta = true;
                    }
                    frameTimeNanos = now - lastFrameOffset;
                    //【2.1】更新 mLastFrameTimeNanos；
                    mLastFrameTimeNanos； = frameTimeNanos;
                }
            }
        }
        try {
            Trace.traceBegin(Trace.TRACE_TAG_VIEW, CALLBACK_TRACE_TITLES[callbackType]);
            //【-->3.3.2】遍历要执行的 CallbackRecord 链表，执行 run 方法；
            for (CallbackRecord c = callbacks; c != null; c = c.next) {
                if (DEBUG_FRAMES) {
                    Log.d(TAG, "RunCallback: type=" + callbackType
                            + ", action=" + c.action + ", token=" + c.token
                            + ", latencyMillis=" + (SystemClock.uptimeMillis() - c.dueTime));
                }
                c.run(frameTimeNanos); // run
            }
        } finally {
            synchronized (mLock) {
                mCallbacksRunning = false;
                //【3】回收子链表中的节点，其实就是逐个断开链接，置空属性，在 recycleCallbackLocked 方法中；
                do {
                    final CallbackRecord next = callbacks.next;
                    recycleCallbackLocked(callbacks);
                    callbacks = next;
                } while (callbacks != null);
            }
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
    }
```

不多数了。

现在我们回过头看回顾哪里，显然：

```java
    final class TraversalRunnable implements Runnable {
        @Override
        public void run() {
            //【-->5.7】开始遍历；
            doTraversal();
        }
    }

    final TraversalRunnable mTraversalRunnable = new TraversalRunnable();
```

开始遍历和绘制的操作这里就开始了；



## 7.4 doScheduleCallback

显然，这里依然从 CallbackQueues 判断 callbackType 是否有回调时间到了；

```java
    void doScheduleCallback(int callbackType) {
        synchronized (mLock) {
            if (!mFrameScheduled) {
                final long now = SystemClock.uptimeMillis();
                //【-->3.3.2.3】判断时间是否到了；
                if (mCallbackQueues[callbackType].hasDueCallbacksLocked(now)) {
                    //【-->3.4】 涛声依旧了~
                    scheduleFrameLocked(now);
                }
            }
        }
    }
```

不多说了；

# 8 FrameInfo

用于表示帧的信息：

## 8.1 new FrameInfo

```java
final class FrameInfo {
    long[] mFrameInfo = new long[9];
}
```

内部有一个数组：**mFrameInfo**，每一个元素的下标和存储的值得含义如下：

```java
private static final int INTENDED_VSYNC = 1; // 预期帧时间, 和 Vsync 相关，不受抖动调整；
private static final int VSYNC = 2; // 实际帧时间，会被抖动调整，是动画和绘图系统的时间输入
private static final int OLDEST_INPUT_EVENT = 3; // 最旧的输入事件的时间；
private static final int NEWEST_INPUT_EVENT = 4; // 最新的输入事件的时间；
private static final int HANDLE_INPUT_START = 5; // 输入事件开始处理时的时间；

private static final int ANIMATION_START = 6; // 动画评估开始的时间；
private static final int PERFORM_TRAVERSALS_START = 7; // ViewRootImpl#performTraversals() 开始的时间； 
private static final int DRAW_START = 8; // Draw 方法开始的时间；
```





## 8.2 setVsync

设置同步信息：

```java
    public void setVsync(long intendedVsync, long usedVsync) {
        mFrameInfo[INTENDED_VSYNC] = intendedVsync;
        mFrameInfo[VSYNC] = usedVsync;
        mFrameInfo[OLDEST_INPUT_EVENT] = Long.MAX_VALUE;
        mFrameInfo[NEWEST_INPUT_EVENT] = 0;
        mFrameInfo[FLAGS] = 0;
    }
```



## 8.3 markInputHandlingStart (Animations/PerformTraversals)

标记 input 处理，动画开始，PerformTraversals 开始的时间：**System.nanoTime**

```java
    public void markInputHandlingStart() {
        mFrameInfo[HANDLE_INPUT_START] = System.nanoTime();
    }

    public void markAnimationsStart() {
        mFrameInfo[ANIMATION_START] = System.nanoTime();
    }

    public void markPerformTraversalsStart() {
        mFrameInfo[PERFORM_TRAVERSALS_START] = System.nanoTime();
    }
```





# 9 总结

我们来看下整个流程：

## 9.1 图示

- **创建编舞者**

流程图如下：

```sequence
ViewRootImpl -> Choreographer: getInstance (线程单例)

Choreographer -> Choreographer: 2.new Choreographer
Choreographer -> FrameHandler: 3.new FrameHandler（ui 线程 Handler）
Choreographer -> FrameDisplayEventReceiver: 4.new FrameDisplayEventReceiver（用于请求 Vsync）

Note over FrameDisplayEventReceiver,NativeDisplayEventReceiver: Java 这之间隔着一层 jni 调用层(...) Native

FrameDisplayEventReceiver -> FrameDisplayEventReceiver: 5.nativeInit

Note right of NativeDisplayEventReceiver: 创建时，会获取 java MQ 对应的 \n native MQ，以及 native Looper
FrameDisplayEventReceiver -> NativeDisplayEventReceiver: 6.new NativeDisplayEventReceiver
NativeDisplayEventReceiver --> FrameDisplayEventReceiver: 6.return NativeDisplayEventReceiver
FrameDisplayEventReceiver -> NativeDisplayEventReceiver: 7.initialize （监听 mReceiver.getFd 返回的文件句柄）

Choreographer --> ViewRootImpl: return Choreographer
```





- **请求 Vsync 信号**

流程图如下：

```sequence
ViewRootImpl -> Choreographer: 5.postCallback（请求 Vsync）
Choreographer -> Choreographer: 6.postCallbackDelayed
Choreographer -> Choreographer: 7.postCallbackDelayedInternal

Choreographer -> CallbackQueue: 8.addCallbackLocked (将回调根据 type 加入到不同 CallbackQueue 中)

CallbackQueue -> CallbackQueue: 9.obtainCallbackLocked (创建新的/使用缓存)
CallbackQueue -> CallbackRecord: 10.new CallbackRecord
CallbackRecord --> CallbackQueue: 10.return CallbackRecord/temp one

Choreographer -> Choreographer : 11.scheduleFrameLocked (调度帧操作)
Choreographer -> Choreographer : 12.scheduleVsyncLocked
Choreographer -> FrameDisplayEventReceiver: 13.scheduleVsync
FrameDisplayEventReceiver -> FrameDisplayEventReceiver: 14.nativeScheduleVsync
FrameDisplayEventReceiver -> [extends DisplayEventDispatcher] \n NativeDisplayEventReceiver : 15.scheduleVsync (最后进入 native 层请求 Vsync)
```



- **触发 Callback**

流程图如下：



```sequence
NativeDisplayEventReceiver -> NativeDisplayEventReceiver : 1.handleEvent（Looper 监听到 fd 变化回调） 
NativeDisplayEventReceiver -> NativeDisplayEventReceiver : 2.processPendingEvents 
NativeDisplayEventReceiver -> NativeDisplayEventReceiver : 3.dispatchVsync 

Note over FrameDisplayEventReceiver, NativeDisplayEventReceiver: Native 这之间隔着一层 jni 调用层(...) Java
NativeDisplayEventReceiver -> FrameDisplayEventReceiver : 4.dispatchVsync 
FrameDisplayEventReceiver -> FrameHandler : 5.sendMessageAtTime（将自己作为 runnable 传过去） 
FrameHandler --> FrameDisplayEventReceiver : 6.run

FrameDisplayEventReceiver -> Choreographer : 7.doFrame

Choreographer -> FrameInfo : 8.setVsync (设置帧信息)
Choreographer -> Choreographer : 9.doCallbacks
```



## 9.2 EventThread 线程

这里简单的总结下 EventThread 线程：

### 9.2.1  threadLoop

EventThread 启动后会进入 loop 循环，等待 Vsync 事件：

```c++
bool EventThread::threadLoop() {
    DisplayEventReceiver::Event event;
    Vector< sp<EventThread::Connection> > signalConnections;
    //【1】等待 Vsync 事件，waitForEvent 内部是一个 do while 循环，但是，并不是无限循环下去；
    signalConnections = waitForEvent(&event);

    //【2】分发 Vsync 事件给监听器：
    const size_t count = signalConnections.size();
    for (size_t i=0 ; i<count ; i++) {
        const sp<Connection>& conn(signalConnections[i]);
        //【1】分发 Vsync 事件，这里
        status_t err = conn->postEvent(event);
        if (err == -EAGAIN || err == -EWOULDBLOCK) {
            // The destination doesn't accept events anymore, it's probably
            // full. For now, we just drop the events on the floor.
            // FIXME: Note that some events cannot be dropped and would have
            // to be re-sent later.
            // Right-now we don't have the ability to do this.
            ALOGW("EventThread: dropping event (%08x) for connection %p",
                    event.header.type, conn.get());
        } else if (err < 0) {
            // handle any other error on the pipe as fatal. the only
            // reasonable thing to do is to clean-up this connection.
            // The most common error we'll get here is -EPIPE.
            removeDisplayEventConnection(signalConnections[i]);
        }
    }
    return true;
}

```

我们看到当监听到 Vsync 信号后，会调用 Connection::postEvent 方法将 event 分发出去！

#### 9.2.1.1  Connection::postEvent

```c++
status_t EventThread::Connection::postEvent(
        const DisplayEventReceiver::Event& event) {
    //【1】这里是通过 DisplayEventReceiver 的方法，发送 Vsync 事件；
    ssize_t size = DisplayEventReceiver::sendEvents(mChannel, &event, 1);
    return size < 0 ? status_t(size) : status_t(NO_ERROR);
}
```

#### 9.2.1.1  DisplayEventReceiver::sendEvents

```c++
ssize_t DisplayEventReceiver::sendEvents(const sp<BitTube>& dataChannel,
        Event const* events, size_t count)
{
    //【1】这里我们看到他是通过 BitTube 字节管道发送 Vsync 事件，给 app 进程；
    // 参数 dataChannel 从上面看，就是创建 connection 对象的时候，创建的 BitTube 实例；
    return BitTube::sendObjects(dataChannel, events, count);
}
```

也就是说，这里用到了另外一个进程间通讯的机制：BitTube，这里我们不探讨

### 9.2.2 waitForEvent

等待 event 事件，这里我们先关注下核心的代码（一些注释我们先放着）

```c++
// This will return when (1) a vsync event has been received, and (2) there was
// at least one connection interested in receiving it when we started waiting.
Vector< sp<EventThread::Connection> > EventThread::waitForEvent(
        DisplayEventReceiver::Event* event)
{
    Mutex::Autolock _l(mLock);
    Vector< sp<EventThread::Connection> > signalConnxections;
    //【1】这里是一个 do whidle 循环，条件是 signalConnections 为空；
    do {
        bool eventPending = false;
        bool waitForVSync = false;

        size_t vsyncCount = 0;
        nsecs_t timestamp = 0;
        for (int32_t i=0 ; i<DisplayDevice::NUM_BUILTIN_DISPLAY_TYPES ; i++) {
            timestamp = mVSyncEvent[i].header.timestamp;
            if (timestamp) {
                // we have a vsync event to dispatch
                *event = mVSyncEvent[i];
                mVSyncEvent[i].header.timestamp = 0;
                vsyncCount = mVSyncEvent[i].vsync.count;
                break;
            }
        }

        if (!timestamp) {
            // no vsync event, see if there are some other event
            eventPending = !mPendingEvents.isEmpty();
            if (eventPending) {
                // we have some other event to dispatch
                *event = mPendingEvents[0];
                mPendingEvents.removeAt(0);
            }
        }

        //【1】找到所有的监听 Vsync 事件的 connections 对象，实际上这里就是我们前面 DisplayEventReceiver 创建的链接对象；
        size_t count = mDisplayEventConnections.size();
        for (size_t i=0 ; i<count ; i++) {
            sp<Connection> connection(mDisplayEventConnections[i].promote());
            if (connection != NULL) {
                bool added = false;
                if (connection->count >= 0) {
                    //【1.1】将其设置为 true，表示有 connection 在等待 Vsync 信号；
                    waitForVSync = true;
                    if (timestamp) {
                        // we consume the event only if it's time
                        // (ie: we received a vsync event)
                        if (connection->count == 0) {
                            // fired this time around
                            connection->count = -1;
                            signalConnections.add(connection);
                            added = true;
                        } else if (connection->count == 1 ||
                                (vsyncCount % connection->count) == 0) {
                            // continuous event, and time to report it
                            signalConnections.add(connection);
                            added = true;
                        }
                    }
                }

                if (eventPending && !timestamp && !added) {
                    // we don't have a vsync event to process
                    // (timestamp==0), but we have some pending
                    // messages.
                    signalConnections.add(connection);
                }
            } else {
                // we couldn't promote this reference, the connection has
                // died, so clean-up!
                mDisplayEventConnections.removeAt(i);
                --i; --count;
            }
        }

        // Here we figure out if we need to enable or disable vsyncs
        if (timestamp && !waitForVSync) {
            // we received a VSYNC but we have no clients
            // don't report it, and disable VSYNC events
            disableVSyncLocked();
        } else if (!timestamp && waitForVSync) {
            // we have at least one client, so we want vsync enabled
            // (TODO: this function is called right after we finish
            // notifying clients of a vsync, so this call will be made
            // at the vsync rate, e.g. 60fps.  If we can accurately
            // track the current state we could avoid making this call
            // so often.)
            enableVSyncLocked();
        }

        // note: !timestamp implies signalConnections.isEmpty(), because we
        // don't populate signalConnections if there's no vsync pending
        if (!timestamp && !eventPending) {
            // wait for something to happen
            if (waitForVSync) {
                // This is where we spend most of our time, waiting
                // for vsync events and new client registrations.
                //
                // If the screen is off, we can't use h/w vsync, so we
                // use a 16ms timeout instead.  It doesn't need to be
                // precise, we just need to keep feeding our clients.
                //
                // We don't want to stall if there's a driver bug, so we
                // use a (long) timeout when waiting for h/w vsync, and
                // generate fake events when necessary.
                bool softwareSync = mUseSoftwareVSync;
                nsecs_t timeout = softwareSync ? ms2ns(16) : ms2ns(1000);
                if (mCondition.waitRelative(mLock, timeout) == TIMED_OUT) {
                    if (!softwareSync) {
                        ALOGW("Timed out waiting for hw vsync; faking it");
                    }
                    // FIXME: how do we decide which display id the fake
                    // vsync came from ?
                    mVSyncEvent[0].header.type = DisplayEventReceiver::DISPLAY_EVENT_VSYNC;
                    mVSyncEvent[0].header.id = DisplayDevice::DISPLAY_PRIMARY;
                    mVSyncEvent[0].header.timestamp = systemTime(SYSTEM_TIME_MONOTONIC);
                    mVSyncEvent[0].vsync.count++;
                }
            } else {
                // 没有人对 vsync 感兴趣，所以我们只想睡觉。h/w vsync 会被禁用，线程会进入等待状态，直到我们
                // 获取新连接，或现有连接变为有兴趣再次接收 vsync。
                mCondition.wait(mLock);
            }
        }
    } while (signalConnections.isEmpty());

    // here we're guaranteed to have a timestamp and some connections to signal
    // (The connections might have dropped out of mDisplayEventConnections
    // while we were asleep, but we'll still have strong references to them.)
    return signalConnections;
}
```

可以看到，当没有 vsync 信号的时候，EventThread 是进入等待队列的，而不是无限的 while 循环下去；

```c++
mCondition.wait(mLock);
```



## 9.3 BitTube

这里我们看下 BitTube 的发送和接收方法：

```c++
// 通过 tube->write
ssize_t BitTube::sendObjects(const sp<BitTube>& tube,
        void const* events, size_t count, size_t objSize)
{
    const char* vaddr = reinterpret_cast<const char*>(events);
    ssize_t size = tube->write(vaddr, count*objSize);

    // should never happen because of SOCK_SEQPACKET
    LOG_ALWAYS_FATAL_IF((size >= 0) && (size % static_cast<ssize_t>(objSize)),
            "BitTube::sendObjects(count=%zu, size=%zu), res=%zd (partial events were sent!)",
            count, objSize, size);

    //ALOGE_IF(size<0, "error %d sending %d events", size, count);
    return size < 0 ? size : size / static_cast<ssize_t>(objSize);
}

// 通过 tube->read
ssize_t BitTube::recvObjects(const sp<BitTube>& tube,
        void* events, size_t count, size_t objSize)
{
    char* vaddr = reinterpret_cast<char*>(events);
    ssize_t size = tube->read(vaddr, count*objSize);

    // should never happen because of SOCK_SEQPACKET
    LOG_ALWAYS_FATAL_IF((size >= 0) && (size % static_cast<ssize_t>(objSize)),
            "BitTube::recvObjects(count=%zu, size=%zu), res=%zd (partial events were received!)",
            count, objSize, size);

    //ALOGE_IF(size<0, "error %d receiving %d events", size, count);
    return size < 0 ? size : size / static_cast<ssize_t>(objSize);
}

// ----------------------------------------------------------------------------
}; // namespace android
```

