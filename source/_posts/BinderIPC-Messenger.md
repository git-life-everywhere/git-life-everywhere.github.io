# Binder跨进程通信 - 信使 Messenger
title: Binder跨进程通信 - 信使 Messenger
date: 2017/03/06 20:46:25
categories:
- AndroidFramework源码分析
- Binder跨进程通信
tags: Binder跨进程通信
---


Android 已经内置了一些模板和工具类来帮助我们更好的实现跨进程通信，除了 aidl 模板，还有一个就是信使 Messenger，下面我们来分析下信使的原理！

# 1 Messenger 源码分析

Messenger 本质上是实现了 aidl 模板，aidl 文件名为 IMessenger.aidl，位于 android/frameworks/base/core/java/android/os/IMessenger.aidl 目录下：

```java
package android.os;

import android.os.Message;

/** @hide */
oneway interface IMessenger {
    void send(in Message msg);
}
```
同样的，该 aidl 会生成一个用于跨进程通信的接口！！

```java
package android.os;

public interface IMessenger extends IInterface {
    void send(Message var1) throws RemoteException;

    // 用于创建服务端桩对象！
    public abstract static class Stub extends Binder implements IMessenger {
        public Stub() {
            throw new RuntimeException("stub");
        }

        public static IMessenger asInterface(IBinder var0) {
            throw new RuntimeException("stub");
        }

        public IBinder asBinder() {
            throw new RuntimeException("stub");
        }

        public boolean onTransact(int var1, Parcel var2, Parcel var3, int var4) throws RemoteException {
            throw new RuntimeException("stub");
        }
    }
}
```
一切又是那么熟悉，下面我们重点分析 Messenger 是如何封装的！

## 1.1 Messenger create

创建 Messenger 的方法有如下 2 种：

```java
public Messenger(IBinder target) {...}
public Messenger(Handler target) {...}
```
参数不同，作用和目的也不同！


### 1.1.1 create with IBinder


下面我们来看第一个构造器：

```java
    public Messenger(IBinder target) {
        mTarget = IMessenger.Stub.asInterface(target);
    }
```

第一个构造器很简单，和使用 aidl 模板很相似：
```java
    mTarget = IMessenger.Stub.asInterface(target);
```
其实就是通过 asInterface 将服务端转为一个代理对象，保存到内部的成员变量 mTarget 中！

其实我们类比 aidl 模板，要实现跨进程通信，服务端需要有一个桩对象，客户端需要有其对应的代理对象！

所以，**以 IBinder target 为参数的构造器用于跨进程通信的客户端进程，用以生成客户端代理对象**！

### 1.1.2 create with Handler

下面我们来看看第二个构造器：

```java
    public Messenger(Handler target) {
        mTarget = target.getIMessenger();
    }
```
这里调用了 Handler.getIMessenger() 方法：

```java
    final IMessenger getIMessenger() {
        synchronized (mQueue) {
            if (mMessenger != null) {
                return mMessenger;
            }

            //【1】这里会创建一个 MessengerImpl 实例！
            mMessenger = new MessengerImpl();
            return mMessenger;
        }
    }
```

IMessenger mMessenger 是 Handler 内部的一个变量，getIMessenger() 方法会创建 MessengerImpl 实例！

```java
    private final class MessengerImpl extends IMessenger.Stub {
        public void send(Message msg) {
            msg.sendingUid = Binder.getCallingUid();
            
            //【1】MessengerImpl 为 Handler 的内部类，持有外部类的引用！
            // 调用 Handler.sendMessage 方法！
            Handler.this.sendMessage(msg);
        }
    }
```

其实，可以看到 MessengerImpl 继承了 IMessenger.Stub 抽象类，并实现了其 send 方法，就是调用 Handler 的 sendMessage 方法！

类比 aidl 模板，**以 Handler target 为参数的构造器用于跨进程通信的服务端进程，用以生成服务端 “桩” 对象**！

```java
    public IBinder getBinder() {
        return mTarget.asBinder();
    }
```

同时，Messenger 也会提供一个 getBinder()，用于获得用于通信的 IBinder 实例！

## 1.2 Messenger send

Messenger 是通过 Message 来封装通信数据的，因为 Message 实现了 Parcelable 接口，可以序列化，跨进程传输！

```java
    public void send(Message message) throws RemoteException {
        mTarget.send(message);
    }
```

send 方法很简单，只需要调用内部的 IMessenger mTarget 的 send 方法，那么服务端的 MessengerImpl.send 方法会调用，最终触发


# 2 Messenger 序列化处理

Messenger 实现了 Parcelable 接口，因此可以序列化，而**使用 Messenger 进行双向通信，正式依赖于序列化的特性**，下面我们来看下：


Messenger 内部定义了 2 个方法，来实现序列化！

## 2.1 writeMessengerOrNullToParcel

```java
    public void writeToParcel(Parcel out, int flags) {
        out.writeStrongBinder(mTarget.asBinder());
    }

    public static void writeMessengerOrNullToParcel(Messenger messenger,
            Parcel out) {
        out.writeStrongBinder(messenger != null ? messenger.mTarget.asBinder()
                : null);
    }
```
我们可以看到，我们写入到 Parcel out 中的并不是我们的 Messenger 对象，而是 messenger.mTarget.asBinder()！！

## 2.1 readMessengerOrNullFromParcel

```java
    public static Messenger readMessengerOrNullFromParcel(Parcel in) {
        IBinder b = in.readStrongBinder();
        return b != null ? new Messenger(b) : null;
    }
```

同样的，当我们从 Parcel out 中读取时，Messenger 又帮我们做了一些的处理，调用了构造器：Messenger(IBinder target)，通过传递过来的 IBinder 对象，创建客户端 Messenger 对象！

可以看到，Messenger 序列化传递过程中，利用了序列化的特性，自动帮我们是实现了代理和装对象的转换！

# 3 Messenger 的使用

下面列出 Messenger 使用的核心代码，这里我会使用一个

## 3.1 服务端实现

```java
public class MServerService extends Service {
    private static final String TAG = "MServerService";
    private Messenger mSMessenger = null;
    private HandlerThread mThread = null;
    private MyHandler mHandler = null;

    @Override
    public void onCreate() {
        super.onCreate();
        mThread = new HandlerThread("Messenger", Thread.MIN_PRIORITY);
        mThread.start();
        mHandler = new MyHandler(mThread.getLooper());
        //【关键代码】创建 Messenger 传入指定的 Handler！
        mSMessenger = new Messenger(mHandler);

    }

    @Override
    public IBinder onBind(Intent intent) {
        if (mSMessenger == null) {
            mSMessenger = new Messenger(mHandler);
        }
        //【关键代码】调用 Messenger.getBinder() 方法！
        return mSMessenger.getBinder();
    }
    
    class MyHandler extends Handler {
        public MyHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
            switch (msg.what) {
                case 1:
                    Log.d(TAG, "BIND SUCCESSFUL");
                    //【关键代码】客户端注册，然后回调
                    Message msgR = new Message();
                    msgR.what = 2;
                    try {
                        mtoCMessenger = msg.replyTo;
                        mtoCMessenger.send(msgR);
                    } catch (RemoteException e) {
                        e.printStackTrace();
                    }
                    break;
                default:
                     break;
            }
        }
    }
}
```

## 3.2 客户端实现

```java
public class MClientActivity extends Activity {
    private static final String TAG = "MClientActivity";
    private Messenger mtoSProxy;
    private Messenger mCMessenger;
    private CHandler mHandler;
    private ServiceConnection mConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            //【关键代码】bind 成功，同时客户端注册！        
            mtoSProxy = new Messenger(service);
            sendMessage(mtoSProxy);
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            mtoSProxy = null;
        }
    };

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_aidl);
        //【关键代码】创建服务端的 Messenger！
        mHandler = new CHandler(getMainLooper());
        mCMessenger = new Messenger(mHandler);
        bindMServerService();
    }

    private void bindMServerService() {
        Intent intent = new Intent();
        intent.setClass(this, MServerService.class);
        bindService(intent, mConnection, BIND_AUTO_CREATE);
    }

    private void sendMessage(Messenger proxy) {
        try {
            //【关键代码】bind 成功，注册客户端的 Messenger
            Message msg = new Message();
            msg.what = 1;
            msg.replyTo = mCMessenger;
            proxy.send(msg);
        } catch (RemoteException e) {
            e.printStackTrace();
        }
    }

    static class CHandler extends Handler {
        public CHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
            switch (msg.what) {
                case 2:
                    Log.d(TAG, "REGISTER SUCCESSFUL");
                    break;
                default:
                    break;
            }
        }
    }
```
代码很简单，就不多说了！

# 4 总结

Messenger 本质上是对 AIDL 模板的封装，通过 Messenger 我们可以实现基于消息的跨进程通信！

同样的，由于 Messenger 是基于消息的跨进程通信，通过 Handler 实现，所以无法实现并发的通信操作！！

