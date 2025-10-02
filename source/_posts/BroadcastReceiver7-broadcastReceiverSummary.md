# BroadcastReceiver篇 7 - BroadcastReceiver 广播机制总结
title: BroadcastReceiver篇 7 - BroadcastReceiver 广播机制总结
date: 2016/08/13 20:46:25
catalog: true
categories:
- AndroidFramework源码分析
- BroadcastReceiver广播接收者
tags: BroadcastReceiver广播接收者
---

基于 `Android 7.1.1` 分析 `BroadcastReceiver` 组件的机制，本文为作者原创，转载请说明出处！

# 0 综述

本篇文章总结一下广播和广播接收者相关的知识点，广播是 Android 组件间的通信方式，本质上是 Intent 意图，可用于以下场景：

- 同一应用内部的同一进程间；
- 同一应用内部的不同进程间的不同组件的通信；


# 1 广播的使用

`Android` 系统的广播，本质上就是 `Intent`，对其熟悉的朋友都知道，`Intent` 可以携带一些重要的数据，下面，我列举些和广播相关的参数，对于 `Intent` 的具体分析，请看其他的博文！

## 1.1 广播的类型

按照前台和后台来区分，会有前台广播和后台广播，`AMS` 内部有 `2` 个队列来管理前台和后台广播：
```java
    BroadcastQueue mFgBroadcastQueue;
    BroadcastQueue mBgBroadcastQueue;
```

按照有序和无序来区分，分为无序发送广播和有序发送广播，前台和后台队列中都有如下两个列表：
```
    final ArrayList<BroadcastRecord> mParallelBroadcasts = new ArrayList<>();
    final ArrayList<BroadcastRecord> mOrderedBroadcasts = new ArrayList<>();
```
`mParallelBroadcasts` 是并发发送列表，`mOrderedBroadcasts` 是有序发送列表；

按照发送的方式来分，又可以分为以下的几种：

- **普通广播**

发送的接口为：`sendBroadcast(intent)`，对于普通类型的广播，如果他的目标接收者是动态接收者，那就会添加到 `mParallelBroadcasts`列表中并发发送，如果他的目标接收者是静态接收者，那就会添加到 `mOrderedBroadcasts`中有序发送；

- **有序广播**

发送的接口为：`sendOrderedBroadcast(intent)`，对于有序类型的广播，不管是静态接收者，还是动态接收者，都会被添加到 `mOrderedBroadcasts`中有序发送；

- **粘性广播**
- **粘性有序广播**

粘性广播很特殊，系统会将其保存到一个列表 `mStickyBroadcasts`中，当还有新的接收者注册后，系统会将其发送给接收者！！但是，在 `Android5.0/API level 21` 开始粘性广播和粘性有序广播都不再建议使用了，主要还是安全性问题；

除此之外，还有以下类型的广播：

- **系统广播**：安卓系统内置了很多的广播，用于满足系统功能的基本需求，比如：熄屏亮屏广播，开关机广播的等等，都是由系统发送的；
- **本地广播**：本地广播也叫做应用内部广播，为什么会有这种广播呢，主要也是为了安全，防止其他应用通过一些途径平凡的拉起接收者的方法，或者伪装成接收者接收者指定的广播，为了解决这种问题，可以有如下几种方法：

    - 将接收者的 `android：exported` 属性改为 `false`，但是这样就只能接受来自应用内部的广播了；
    - 发送方和接收者都指定权限信息，权限不匹配，就不能发送和接受；
    - 对于广播可以显示指定包名和组件名；

最后一种方式，就是使用本地广播，有一个专门的实现类 `LocalBroadcastManager`，这个我后面会单独分析！


## 1.2 Flags 标志位

`Android 7.1.1` 一共提供了以下几种和 `Receiver` 相关的 `flags`，我们来看一下：

```java
    public static final int FLAG_RECEIVER_REGISTERED_ONLY = 0x40000000;
    public static final int FLAG_RECEIVER_REPLACE_PENDING = 0x20000000;
    public static final int FLAG_RECEIVER_FOREGROUND = 0x10000000;
    public static final int FLAG_RECEIVER_NO_ABORT = 0x08000000;
    public static final int FLAG_RECEIVER_REGISTERED_ONLY_BEFORE_BOOT = 0x04000000;
    public static final int FLAG_RECEIVER_BOOT_UPGRADE = 0x02000000;
    public static final int FLAG_RECEIVER_INCLUDE_BACKGROUND = 0x01000000;
    public static final int FLAG_RECEIVER_EXCLUDE_BACKGROUND = 0x00800000;

```

下面我逐个解释一下每个标签的作用，和他们在系统中是如何处理的：

- **FLAG_RECEIVER_REGISTERED_ONLY**：只有动态注册的接收者才能接受该广播，静态接收者无法接受；

```java
        // Figure out who all will receive this broadcast.
        List receivers = null;
        List<BroadcastFilter> registeredReceivers = null;
        // Need to resolve the intent to interested receivers...
        if ((intent.getFlags()&Intent.FLAG_RECEIVER_REGISTERED_ONLY)
                 == 0) {
            receivers = collectReceiverComponents(intent, resolvedType, callingUid, users);
        }
```

`AMS` 内部会做处理，如果广播设置了 `FLAG_RECEIVER_REGISTERED_ONLY` 标签，那他就不会收集静态注册的接收者；

</br>

- **FLAG_RECEIVER_REPLACE_PENDING**：新发送的广播会取代之前的已发送但未处理的相同广播，广播是否相同，是通过 `Intent.filterEquals` 方法进行匹配的；如果匹配成功，新广播会取代旧广播，但在等待列表中的位置不变，该 `flags` 常常被粘性广播使用，只保证将最新的广播发送给对应的接收者；

```java
        final boolean replacePending =
                (intent.getFlags()&Intent.FLAG_RECEIVER_REPLACE_PENDING) != 0;
        ... ... ...        
                
        if ((receivers != null && receivers.size() > 0)
                || resultTo != null) {
            ... ... ...

            boolean replaced = replacePending && queue.replaceOrderedBroadcastLocked(r);
            if (!replaced) {
                queue.enqueueOrderedBroadcastLocked(r);
                queue.scheduleBroadcastsLocked();
            }
        } else {
        }

```

这里，省略了一下重要的代码，可以看到，如果广播设置了 `FLAG_RECEIVER_REPLACE_PENDING`，他会取代之前的旧的未被处理的相同广播，这里就不多说了！！

</br>

- **FLAG_RECEIVER_FOREGROUND**：设置该标志位后，广播的接收者会以前台的优先级运行，超时时间会变短，正常的接收者是后台优先级的，是不会被自动提升的！
```java
    BroadcastQueue broadcastQueueForIntent(Intent intent) {
        final boolean isFg = (intent.getFlags() & Intent.FLAG_RECEIVER_FOREGROUND) != 0;
        if (DEBUG_BROADCAST_BACKGROUND) Slog.i(TAG_BROADCAST,
                "Broadcast intent " + intent + " on "
                + (isFg ? "foreground" : "background") + " queue");
        return (isFg) ? mFgBroadcastQueue : mBgBroadcastQueue;
    }
```
在 AMS 中，有两个广播队列：前台队列和后台队列，可以看到，默认情况下，不设置该标志位，广播都是被添加到后台队列中的！！

</br>

- **FLAG_RECEIVER_NO_ABORT**：设置该标志位后，如果该广播是一个有序发送的广播，不允许接收者过滤不处理该广播；

```java
        if (resultAbort && (r.intent.getFlags()&Intent.FLAG_RECEIVER_NO_ABORT) == 0) {
            r.resultAbort = resultAbort;

        } else {
            r.resultAbort = false;

        }
```
可以看到这里，会对该 `flags` 做一个判断，看是否过滤不处理该广播；

</br>

- **FLAG_RECEIVER_REGISTERED_ONLY_BEFORE_BOOT**：设置该标志位后，当在开机完成之前发送该广播，只有已经被注册的接收者（静态）会被调用；如果是粘性广播，仍然会被系统保留下来，即使没有接收者被调用；如果设置了 `FLAG_RECEIVER_REGISTERED_ONLY` 标志位，该标志位无效！

```java
    final Intent verifyBroadcastLocked(Intent intent) {
        ... ... ... ...

        //【1】mProcessesReady 表示系统进程是否准备好！
        if (!mProcessesReady) {

            if ((flags&Intent.FLAG_RECEIVER_REGISTERED_ONLY_BEFORE_BOOT) != 0) {
                // This will be turned into a FLAG_RECEIVER_REGISTERED_ONLY later on if needed.
                
            } else if ((flags&Intent.FLAG_RECEIVER_REGISTERED_ONLY) == 0) {
                Slog.e(TAG, "Attempt to launch receivers of broadcast intent " + intent
                        + " before boot completion");
                throw new IllegalStateException("Cannot broadcast before boot completed");
            }
        }

        return intent;
    }

```

可以看到，如果在系统进程没有准备好时，如果广播设置了 `FLAG_RECEIVER_REGISTERED_ONLY_BEFORE_BOOT`，但如果此时没有设置 `FLAG_RECEIVER_REGISTERED_ONLY` 标签，那会报出异常。

</br>

- **FLAG_RECEIVER_BOOT_UPGRADE**：

```java

```

</br>

- **FLAG_RECEIVER_INCLUDE_BACKGROUND**：如果设置了该标签，该广播将**始终发送**给后台（缓存或不运行）应用程序的静态接收者；

</br>

- **FLAG_RECEIVER_EXCLUDE_BACKGROUND**：如果设置了该标签，该广播将**始终不会发送**给后台（缓存或不运行）应用程序的静态接收者，但是如果发送者显示指定了接收者的组件名或者包名，那么后台接收者仍然可以接收到该广播！！

这两个标签的处理如下，这里的 `info` 是静态接收者的数据对象！
```java
            if (!skip) {

                //【1】判断是否允许后台启动！
                final int allowed = mService.checkAllowBackgroundLocked(
                        info.activityInfo.applicationInfo.uid, info.activityInfo.packageName, -1,
                        false);
                
                if (allowed != ActivityManager.APP_START_MODE_NORMAL) {
                    //【2】如果是禁止启动，就跳过该应用程序，但 checkAllowBackgroundLocked 不会返回该值；
                    if (allowed == ActivityManager.APP_START_MODE_DISABLED) {
                        Slog.w(TAG, "Background execution disabled: receiving "
                                + r.intent + " to "
                                + component.flattenToShortString());

                        skip = true;
                    
                    //【3】如果是延迟启动，那其属于一个后台的接收者，那就要判断下标志位；
                    } else if (((r.intent.getFlags()&Intent.FLAG_RECEIVER_EXCLUDE_BACKGROUND) != 0)
                            || (r.intent.getComponent() == null
                                && r.intent.getPackage() == null
                                && ((r.intent.getFlags()
                                        & Intent.FLAG_RECEIVER_INCLUDE_BACKGROUND) == 0))) {

                        Slog.w(TAG, "Background execution not allowed: receiving "
                                + r.intent + " to "
                                + component.flattenToShortString());

                        skip = true;
                    }
                }
            }
```
Android 系统在发送广播时会判断是否会跳过一些接收者！如果 `checkAllowBackgroundLocked` 返回值，返回值为 `ActivityManager.APP_START_MODE_DELAYED`，表示接收者是需要后台启动的！

这种情况下：

- 如果设置了 `FLAG_RECEIVER_EXCLUDE_BACKGROUND` 标签，那就会逃过该后台的静态接收者；
- 如果没设置 `FLAG_RECEIVER_INCLUDE_BACKGROUND` 标签，且没有设置 `Component` 或者 `Package`，那就会跳过该后台的静态接收者；



</br>

对于标志位的分析，我们就简单的提一下，详细的分析，大家可以去看 `sendBroadcast` 过程！！


## 1.3 广播的发送方法

对于广播的发送，很简单，有很多的方法，来发送不同类型的广播：

- 普通广播：
```

```

- 有序广播：
```java

```

- 粘性广播：
```java

```

- 粘性有序广播：
```java

```

# 2 接收者的使用

广播接收者分为 `2` 种，静态接收者，动态接收者：

- 动态注册的接收者不是常驻型，也就是说广播跟随 `Activity` 的生命周期。注意在 `Activity` 结束前，移除广播接收器。
- 静态注册的接收者不是常驻型，也就是说当应用程序关闭后，如果有信息广播来，程序也会被系统调用自动运行。

下面我们来看看他们的使用方法：

## 2.1 静态注册的接收者

静态接收者需要单独存在于一个 `.java` 文件：
```java
public class InstallResultReceiver extends BroadcastReceiver {

    @Override
    public void onReceive(Context context, Intent intent) {
        if (intent == null) {
            OppoLog.i (TAG, "ota intent is null!");
            return;
        }
    }
}
```

并且在 `AndroidManifest.xml` 需要说明该组件的存在：

```java
        <receiver
            android:name=".install.InstallResultReceiver"
            android:exported="true">
            <intent-filter>
                <action android:name="coolqi.intent.action.package_install_success" />
            </intent-filter>
        </receiver>
```
`AndroidManifest.xml` 可以对接收者配置很多的属性，这里列举几个比较重要的：

```java
    android:name=".install.InstallResultReceiver"
    android:singleUser="true"
    android:process="com.demo.coolqi"
    android:enabled="true"
    android:permission="coolqi.send.package_install_success"
    android:exported="true">
```
解释说明下：

- **name**：接收者的全名；
- **singleUser**：该接收者是否在所有；
- **process**：该接收者所在的进程名；
- **enabled**：该接收者是否为单用户模式，如果为 `true`，所有用户使用的 `BroadcastReceiver` 是同一个；
- **permission**：该接收者指定发送者应具有的权限；
- **exported**：该接收者是否对其他应用暴露；

初次之外：`intent-filter` 也可以设置一些属性，这里列举一些重要的属性：
```java
android:priority=""
```
解释说明下：

- **priority**：表示接收者的优先级，对于有序发送的广播，接收者优先级越高，越先接受到广播；

关于优先级对于广播接受的影响，我们后面会谈到；

## 2.2 动态注册的接收者

动态接收者不能单独存在于一个独立的 `.java` 文件中，他需要最为其他类的内部类来定义，它依附于其他的类：

```
public class InstallResultReceiver extends AppCompatActivity {

    private InstallResultReceiver resultReceiver;
    
    class InstallResultReceiver extends BroadcastReceiver{
  
        @Override
        public void onReceive(Context context, Intent intent) {
     
        }
  }
}
```

需要在代码中实时注册：

```
IntentFilter filter = new IntentFilter();
filter.addAction("coolqi.intent.action.package_install_success");
reReceiver = new InstallResultReceiver();

registerReceiver(reReceiver, filter);
```

同时在不需要该接收者时，需要动态取消注册：

```
unregisterReceiver(reReceiver);
```

对于动态注册的接收者，也可以设置他的优先级：

```java
filter.setPriority(100);
```

# 3 广播的发送和处理

接下来，总结一下，广播的发送和处理流程！


## 3.1 接收者的收集流程

我们先通过一张图来看看广播接收者的收集流程：

<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/23/广播的发送流程.png" alt="广播的发送流程.png-84kB" style="zoom:50%;" />


我们可以看到，发送过程中，首先会发送目标为动态注册的接收者的普通广播，发送方式是并发！

接下来会收集静态接收者和动态接收者，注意如果是普通广播，这里只收集静态注册的接收者，然后根据优先级对接收者进行排序，排序的算法如下：

```java
            while (it < NT && ir < NR) {
                if (curt == null) {
                    curt = (ResolveInfo)receivers.get(it); // 静态
                }
                if (curr == null) {
                    curr = registeredReceivers.get(ir); // 动态
                }
                if (curr.getPriority() >= curt.priority) {
                    receivers.add(it, curr);
                    ir++;
                    curr = null;
                    it++;
                    NT++;
                } else {
                    it++;
                    curt = null;
                }
            }
```
其中：`NT` 是静态接收者的数量， `NR` 是动态注册的接收者的数量!

对于普通广播，由于 `NR` 是 0，目标接收者只有静态接收者，所以这里是不会进行优先级排序的；

对于有序广播，由于其既存在动态接收者，又存在静态接收者，所以这会进行优先级排序，排序的方式如下：

- 如果动态接收者的优先级 `Priority` 大于等于静态接收者的优先级，动态接收者排在前面；
- 否则，静态接收者在前！


通过上面，我们可以看出：

- 对于普通广播，动态接收者是要比静态接收者先接受到广播的，无视优先级！
- 对于有序广播，则是按照优先级来处理！

## 3.2 广播发送流程

接下来，我们来分析下广播的发送过程！

在发送的过程中，我们需要注意一些细节问题：

- 发送的广播 `Intent` 会被强制添加 `Intent.FLAG_EXCLUDE_STOPPED_PACKAGES`，禁止广播发送给被强制停止的接收者！该标志位是从 `Android 3.1` 开始新增了：

```java
    public static final int FLAG_EXCLUDE_STOPPED_PACKAGES = 0x00000010;
    public static final int FLAG_INCLUDE_STOPPED_PACKAGES = 0x00000020;
```

下面我来解释一下：

- `FLAG_EXCLUDE_STOPPED_PACKAGES`：如果设置了该标志位，该 `Intent` 将不会匹配那些被强制停止的应用中的组件，如果不设置该标志位，默认是会匹配被强制停止的应用的组件的！
- `FLAG_INCLUDE_STOPPED_PACKAGES`：：如果设置了该标志位，该 `Intent` 将会匹配那些被强制停止的应用中的组件，如果不设置 `FLAG_EXCLUDE_STOPPED_PACKAGES` 标志位，将会默认设置该标志位，如果两个标志位都被设置了，该标志位才生效！

对于系统广播，一般是无法更改标志位的，当然，系统开发者除外；对于应用的自定义的广播，可以设置 `FLAG_INCLUDE_STOPPED_PACKAGES`，使其能够发送给被停止的接收者！

这里简单提一下，有 `2` 种情况，应用会处于停止状态：

- 应用第一次安装并且没有被启动过；
- 用户在应用管理中强制停止了该应用；


在收集完接收者后，就会创建对应的广播，封装接收者列表，然后，将广播添加到指定队列的列表中，触发发送，我们来简单的回顾下；

<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/23/收集接收者图示.png" alt="收集接收者图示.png-71.7kB" style="zoom:50%;" />


广播分发的调用链如下:
```java
queue.scheduleBroadcastsLocked(); -> send BROADCAST_INTENT_MSG -> processNextBroadcast();
```

我们用一张图来直观的看一下:

<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/23/广播的发送流程2.png" alt="广播的发送流程2.png-65.3kB" style="zoom:50%;" />


可以看到，每次分发，都会率先的处理 `BroadcastQueue` 队列中的并发集合 `mParallelBroadcasts`  中的所有广播：目标为动态接收者的普通广播，直到 `mParallelBroadcasts` 为空！

然后，接着是，处理那些正在等待目标进程启动的广播 `mPendingBroadcast`，如果目标进程没有启动完成或者没有死亡，那就不能继续发送广播，因为需要的等待进程启动后处理该广播！

接着是，处理有序列表 `mOrderedBroadcasts ` 中的广播，首先会遍历 `mOrderedBroadcasts`，移除那些没有接收者 / 接收者都已经接收完成 / 被终止 / 超时的广播，定位到下一次需要发送的广播，进行分发；

> 这里要重点说一下：mOrderedBroadcasts 中的广播有 2 中：一种是目标是静态接收者的普通广播；另一种是有序广播，这二种广播都采用的是有序的发送方式；



# 4 总结


