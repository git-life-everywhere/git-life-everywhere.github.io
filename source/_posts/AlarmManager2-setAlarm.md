# AlarmManager第 2 篇 - set Alarm 流程分析
title: AlarmManager第 2 篇 - set Alarm 流程分析
date: 2017/07/25 20:46:25 
categories: 
- AndroidFramework源码分析
- AlarmManager闹钟管理
tags: AlarmManager闹钟管理
copyright: true
---


[toc]

基于 Android 7.1.1 源码，分析 AlarmManagerService 的机制

# 0 综述


下面是设置精确 alarm 的方法：

```java
    public static void setGlobalNoticeDialogForceShowAlarm(Context context) {
        Intent intent = new Intent("com.coolqi.alarm_start");
        PendingIntent pi = PendingIntent.getBroadcast(context, 0, intent, 0);
        AlarmManager alarmManager = (AlarmManager) context.getSystemService(Context.ALARM_SERVICE);
        alarmManager.cancel(pi);

        int timeHour = 2*24*60*60;// two days

        OppoLog.d(TAG, "show global time: " + timeHour);
        long triggerTime = CommonUtil.getTriggerTime(System.currentTimeMillis(), timeHour);
        alarmManager.setExact(AlarmManager.RTC_WAKEUP, triggerTime, pi);
    }
```

# 1 AlarmManager.setAlarm

AlarmManagerSerivce 提供了很丰富的接口来设置不同类型的 alarm，可以通过 AlarmManager.java 来看到所有的接口：

## 1.1 AlarmManager.set

set 接口用于设置一个一次性的闹钟，该闹钟是非精确的，我们需要传入一个 triggerAtMillis 毫秒值被表示闹钟触发的时间点！

```java
    public void set(@AlarmType int type, long triggerAtMillis, PendingIntent operation) {
        setImpl(type, triggerAtMillis, legacyExactLength(), 0, 0, operation, null, null,
                null, null, null);
    }
```
这个 set 方法最常用：第一个参数是 alarm 的类型，第二个是触发时间，第三个是 PendingIntent，用于启动 Service，activity，或者是 broadcastReceiver！

可以实现跨进程，即：设置该 alarm 的进程和处理 alarm 触发的进程可以不是同一个！

**参数传递**：

- **int type**：闹钟类型 type；
- **long triggerAtMillis**：触发时间，单位毫秒；
- **long windowMillis**：legacyExactLength()，API 19 以后返回值为 WINDOW_HEURISTIC，即 -1；
- **long intervalMillis**：0；
- **int flags**：0；
- **PendingIntent operation**：operation；
- **final OnAlarmListener listener**：null；
- **String listenerTag**：null；
- **Handler targetHandler**：null；
- **WorkSource workSource**：null；
- **AlarmClockInfo alarmCloc**：null；

</br>

```java
    public void set(@AlarmType int type, long triggerAtMillis, String tag, OnAlarmListener listener,
            Handler targetHandler) {
        setImpl(type, triggerAtMillis, legacyExactLength(), 0, 0, null, listener, tag,
                targetHandler, null, null);
    }
```

这个 set 方法并不适用于跨进程通信，其需要传入一个实现了 OnAlarmListener 接口的对象用于监听 alarm 的触发，当 alarm 触发后，OnAlarmListener 的会被 onAlarm() 执行！

targetHandler 表示的是 OnAlarmListener.onAlarm 执行时，目标线程的 handler 对象！

```java
    /** @hide */
    @SystemApi
    @RequiresPermission(android.Manifest.permission.UPDATE_DEVICE_STATS)
    public void set(@AlarmType int type, long triggerAtMillis, long windowMillis,
            long intervalMillis, PendingIntent operation, WorkSource workSource) {
        setImpl(type, triggerAtMillis, windowMillis, intervalMillis, 0, operation, null, null,
                null, workSource, null);
    }
```

```java
    public void set(@AlarmType int type, long triggerAtMillis, long windowMillis,
            long intervalMillis, String tag, OnAlarmListener listener, Handler targetHandler,
            WorkSource workSource) {
        setImpl(type, triggerAtMillis, windowMillis, intervalMillis, 0, null, listener, tag,
                targetHandler, workSource, null);
    }
```

```java
    @SystemApi
    @RequiresPermission(android.Manifest.permission.UPDATE_DEVICE_STATS)
    public void set(@AlarmType int type, long triggerAtMillis, long windowMillis,
            long intervalMillis, OnAlarmListener listener, Handler targetHandler,
            WorkSource workSource) {
        setImpl(type, triggerAtMillis, windowMillis, intervalMillis, 0, null, listener, null,
                targetHandler, workSource, null);
    }
```
set 方法设置的 alarm **是非精确的**！

## 1.2 AlarmManager.setExact

用于设置一个精确的闹钟，该方法是相对于 set 方法的，参数和 set 方法一样，不多说，也有 2 个方法！

```java
    public void setExact(@AlarmType int type, long triggerAtMillis, PendingIntent operation) {
        setImpl(type, triggerAtMillis, WINDOW_EXACT, 0, 0, operation, null, null, null,
                null, null);
    }

    public void setExact(@AlarmType int type, long triggerAtMillis, String tag,
            OnAlarmListener listener, Handler targetHandler) {
        setImpl(type, triggerAtMillis, WINDOW_EXACT, 0, 0, null, listener, tag,
                targetHandler, null, null);
    }
```
不多说了！！

## 1.3 AlarmManager.setWindow

用于设置一个在给定的时间窗触发的闹钟。该方法允许应用程序精确地控制操作系统调整闹钟触发时间的程度。

其中，windowStartMillis 表示时间窗的起始时间！windowStartMillis 表示时间窗的长度，其他参数和 set 方法一样！

```java
    public void setWindow(@AlarmType int type, long windowStartMillis, long windowLengthMillis,
            PendingIntent operation) {
        setImpl(type, windowStartMillis, windowLengthMillis, 0, 0, operation,
                null, null, null, null, null);
    }

    public void setWindow(@AlarmType int type, long windowStartMillis, long windowLengthMillis,
            String tag, OnAlarmListener listener, Handler targetHandler) {
        setImpl(type, windowStartMillis, windowLengthMillis, 0, 0, null, listener, tag,
                targetHandler, null, null);
    }
```
setWindow 设置的是**非精确**的闹钟！

## 1.4 AlarmManager.setXXXRepeating

setRepeating 和 setInexactRepeating 用于设置一个可重复触发的闹钟，但是二者却有着不同：

```java
    public void setRepeating(@AlarmType int type, long triggerAtMillis,
            long intervalMillis, PendingIntent operation) {
        setImpl(type, triggerAtMillis, legacyExactLength(), intervalMillis, 0, operation,
                null, null, null, null, null);
    }
```
setRepeating 方法在 API19 以前是精确的，其时间间隔是固定的，但是在 API19 以后则是**非精确闹钟**，其等价于 setInexactRepeating 方法！

```java
    public void setInexactRepeating(@AlarmType int type, long triggerAtMillis,
            long intervalMillis, PendingIntent operation) {
        setImpl(type, triggerAtMillis, WINDOW_HEURISTIC, intervalMillis, 0, operation, null,
                null, null, null, null);
    }
```
setInexactRepeating 则是**非精确闹钟**，间隔时间不固定！

## 1.5 AlarmManager.setAlarmClock

setAlarmClock 方法用于通过 AlarmClockInfo 来设置一个**精确闹钟**！

```java
    public void setAlarmClock(AlarmClockInfo info, PendingIntent operation) {
        setImpl(RTC_WAKEUP, info.getTriggerTime(), WINDOW_EXACT, 0, 0, operation,
                null, null, null, null, info);
    }

```

## 1.6 AlarmManager.setIdleUntil

setIdleUntil 方法会将 alarm manager service 置为 idle 状态，并设置一个精确闹钟，当该闹钟触发后 alarm manager service 才会退出 idle 状态！

```java
    public void setIdleUntil(@AlarmType int type, long triggerAtMillis, String tag,
            OnAlarmListener listener, Handler targetHandler) {
        setImpl(type, triggerAtMillis, WINDOW_EXACT, 0, FLAG_IDLE_UNTIL, null,
                listener, tag, targetHandler, null, null);
    }
```
可以看到，其调用 setImpl 方法的时候，传入了一个 flag，表示要将 AlarmManagerService 置为 idle 状态！

```java
    public static final int FLAG_IDLE_UNTIL = 1<<4;
```
该方法只能是系统调用！

## 1.7 AlarmManager.setXXXAndAllowWhileIdle

setAndAllowWhileIdle 和 setExactAndAllowWhileIdle 方法用于设置在设备处于 idle 状态下，仍然能够触发的 alarm，但是二者有不同之处：

```java
    public void setAndAllowWhileIdle(@AlarmType int type, long triggerAtMillis,
            PendingIntent operation) {
        setImpl(type, triggerAtMillis, WINDOW_HEURISTIC, 0, FLAG_ALLOW_WHILE_IDLE,
                operation, null, null, null, null, null);
    }
```
setAndAllowWhileIdle 方法设置的是**非精确闹钟**！
```java
    public void setExactAndAllowWhileIdle(@AlarmType int type, long triggerAtMillis,
            PendingIntent operation) {
        setImpl(type, triggerAtMillis, WINDOW_EXACT, 0, FLAG_ALLOW_WHILE_IDLE, operation,
                null, null, null, null, null);
    }
```
setExactAndAllowWhileIdle 方法设置的是**精确闹钟**！

当 setXXXAndAllowWhileIdle 设置闹钟时候，会传入一个 flags，表示该闹钟在设别处于 idle 状态时依然可以触发！
```java
public static final int FLAG_ALLOW_WHILE_IDLE = 1<<2;
```

# 2 AlarmManager.setImpl

可以看到，最后都会调用 setImpl 接口：

```java
    private void setImpl(@AlarmType int type, long triggerAtMillis, long windowMillis,
            long intervalMillis, int flags, PendingIntent operation, final OnAlarmListener listener,
            String listenerTag, Handler targetHandler, WorkSource workSource,
            AlarmClockInfo alarmClock) {
        // 触发时间最小为 0；
        if (triggerAtMillis < 0) {
            triggerAtMillis = 0;
        }

        ListenerWrapper recipientWrapper = null;
        // 当传入的 listener 不为 null 时候，才会进入下面的逻辑！
        if (listener != null) {
            synchronized (AlarmManager.class) {
                // sWrappers 是一个 ArrayMap 集合，用于保存 OnAlarmListener 和 ListenerWrapper 的映射！
                if (sWrappers == null) {
                    sWrappers = new ArrayMap<OnAlarmListener, ListenerWrapper>();
                }
                
                // 如果该 OnAlarmListener 已经有了 recipientWrapper 对象，那就复用该实例；
                // 如果没有那就创建一个新的 recipientWrapper 实例，并将应映射关系保存进 sWrappers！
                recipientWrapper = sWrappers.get(listener);
                // no existing wrapper => build a new one
                if (recipientWrapper == null) {
                    recipientWrapper = new ListenerWrapper(listener);
                    sWrappers.put(listener, recipientWrapper);
                }
            }
            
            // 初始化 targetHandler，如果没有设置 targetHandler 那就为主线程的 Handler！
            final Handler handler = (targetHandler != null) ? targetHandler : mMainThreadHandler;
            // 设置 recipientWrapper 的 Handler 变量！
            recipientWrapper.setHandler(handler);
        }

        try {
            //【3】调用 AlarmManagerService 的 set 方法，设置闹钟！
            mService.set(mPackageName, type, triggerAtMillis, windowMillis, intervalMillis, flags,
                    operation, recipientWrapper, listenerTag, workSource, alarmClock);
        } catch (RemoteException ex) {
            throw ex.rethrowFromSystemServer();
        }
    }
```

## 2.1 new ListenerWrapper 

如果我们指定了 OnAlarmListener，那么就会创建对应的 ListenerWrapper 实例：

```java
    final class ListenerWrapper extends IAlarmListener.Stub implements Runnable {
        final OnAlarmListener mListener; 
        Handler mHandler;
        IAlarmCompleteListener mCompletion;
        
        public ListenerWrapper(OnAlarmListener listener) {
            mListener = listener;
        }

        public void setHandler(Handler h) {
           mHandler = h;
        }
        
        ... ... ...
    }
```
其实 ListenerWrapper 是一个 Runnable 对象，其构造函数和 setHandler 都很简单，这里就不说了！

### 2.1.1 ListenerWrapper.doAlarm

这里先简单说一下，当闹钟触发后，会回调其 doAlarm 方法：
```java
        @Override
        public void doAlarm(IAlarmCompleteListener alarmManager) {
            mCompletion = alarmManager;

            // 从本地的 sWrappers 中移除映射关系！
            synchronized (AlarmManager.class) {
                if (sWrappers != null) {
                    sWrappers.remove(mListener);
                }
            }
            // 调用 set alarm 是传入的 mHandler，执行任务!
            mHandler.post(this);
        }
```
这里的 IAlarmCompleteListener 是一个接口，支持 Binder 通信，当闹钟触发后，AlarmManagerService 会传递一个实现了 IAlarmCompleteListener 接口的对象给当前进程

然后会调用自身的 run 方法，
```java
        @Override
        public void run() {
            try {
                // 执行 onAlarm 回调！
                mListener.onAlarm();
            } finally {
                // No catch -- make sure to report completion to the system process,
                // but continue to allow the exception to crash the app.

                try {
                    mCompletion.alarmComplete(this);
                } catch (Exception e) {
                    Log.e(TAG, "Unable to report completion to Alarm Manager!", e);
                }
            }
        }
```
逻辑很简单，不多说了！

# 3 AlarmManagerService

我们知道 set 方法最后调用了：
```java
 mService.set(mPackageName, type, triggerAtMillis, windowMillis, intervalMillis, flags,
                    operation, recipientWrapper, listenerTag, workSource, alarmClock);
```
这个 mService 是服务端的 proxy 对象！AlarmManager 框架实现了 Aidl 模板，实现跨进程通讯：

```java
interface IAlarmManager {
	/** windowLength == 0 means exact; windowLength < 0 means the let the OS decide */
    void set(String callingPackage, int type, long triggerAtTime, long windowLength,
            long interval, int flags, in PendingIntent operation, in IAlarmListener listener,
            String listenerTag, in WorkSource workSource, in AlarmManager.AlarmClockInfo alarmClock);
    boolean setTime(long millis);
    void setTimeZone(String zone);
    void remove(in PendingIntent operation, in IAlarmListener listener);
    long getNextWakeFromIdleTime();
    AlarmManager.AlarmClockInfo getNextAlarmClock(int userId);
}

```

AlarmManagerService 内部有一个 IBinder 对象，是 IAlarmManager.Stub 的实现对象，作为服务端的 “桩”：

## 3.1 AlarmMS.mService.set

```java
    private final IBinder mService = new IAlarmManager.Stub() {
        @Override
        public void set(String callingPackage,
                int type, long triggerAtTime, long windowLength, long interval, int flags,
                PendingIntent operation, IAlarmListener directReceiver, String listenerTag,
                WorkSource workSource, AlarmManager.AlarmClockInfo alarmClock) {
            final int callingUid = Binder.getCallingUid();

            //【1】通过 appOps 校验 uid 和包名是匹配的！
            mAppOps.checkPackage(callingUid, callingPackage);

            //【2】重复 alarm 必须要使用 PendingIntent，不能使用 directReceiver，那就是异常！
            if (interval != 0) {
                if (directReceiver != null) {
                    throw new IllegalArgumentException("Repeating alarms cannot use AlarmReceivers");
                }
            }
            //【3】如果 workSource 不为 null，
            // 那就检查调用者是否具有 android.Manifest.permission.UPDATE_DEVICE_STATS 权限！
            if (workSource != null) {
                getContext().enforcePermission(
                        android.Manifest.permission.UPDATE_DEVICE_STATS,
                        Binder.getCallingPid(), callingUid, "AlarmManager.set");
            }

            //【4】清除 flags 中的 FLAG_WAKE_FROM_IDLE 和 FLAG_ALLOW_WHILE_IDLE_UNRESTRICTED 标志位
            // 后续会添加！
            flags &= ~(AlarmManager.FLAG_WAKE_FROM_IDLE
                    | AlarmManager.FLAG_ALLOW_WHILE_IDLE_UNRESTRICTED);

            //【5】如果调用者不是 system 进程，那么其不能调用 setIdleUntil 方法设置闹钟，
            // 所以 flags 需要去掉 FLAG_IDLE_UNTIL 位！这个标志位是告诉 AlarmManagerService 什么时候退出 idle 状态！
            // 被 DeviceIdleController 调用！
            if (callingUid != Process.SYSTEM_UID) {
                flags &= ~AlarmManager.FLAG_IDLE_UNTIL;
            }

            //【6】如果闹钟的精确闹钟，那就设置 FLAG_STANDALONE 标志位，精确闹钟不会和其他闹钟进行批处理！
            // Flags 设置 FLAG_STANDALONE 标志位！
            if (windowLength == AlarmManager.WINDOW_EXACT) {
                flags |= AlarmManager.FLAG_STANDALONE;
            }

            //【7】如果是通过 setAlarmClock 方法设置的闹钟，那么其是精确的并且是 idle 状态依然生效的
            // flags 设置 FLAG_WAKE_FROM_IDLE 和 FLAG_STANDALONE！
            if (alarmClock != null) {
                flags |= AlarmManager.FLAG_WAKE_FROM_IDLE | AlarmManager.FLAG_STANDALONE;

            //【7】如果不是调用 setAlarmClock 方法设置 alarm，进入下面的分支！
            } else if (workSource == null && (callingUid < Process.FIRST_APPLICATION_UID
                    || Arrays.binarySearch(mDeviceIdleUserWhitelist,
                            UserHandle.getAppId(callingUid)) >= 0)) {
                            
                // 如果 workSource 为 null，且调用者是系统应用，或者调用者在 doze 模式白名单中
                // 那么该 alarm 在 idle 状态下，不受限制，flags 设置 FLAG_ALLOW_WHILE_IDLE_UNRESTRICTED！
                // 取消 FLAG_ALLOW_WHILE_IDLE 标志位！
                flags |= AlarmManager.FLAG_ALLOW_WHILE_IDLE_UNRESTRICTED;
                flags &= ~AlarmManager.FLAG_ALLOW_WHILE_IDLE;
            }

            //【3.2】调用 AlarmManagerService 的 setImpl 方法继续设置闹钟！
            setImpl(type, triggerAtTime, windowLength, interval, operation, directReceiver,
                    listenerTag, flags, workSource, alarmClock, callingUid, callingPackage);
        }
        ... ... ... ...
    }
```
**标志位**：

- AlarmManager.FLAG_WAKE_FROM_IDLE: 
    - 如果设备处于 idle 状态，该类型的 alarm 会将设别唤醒！
    - AlarmClock 默认就是 FLAG_WAKE_FROM_IDLE 类型的！

</br>

- AlarmManager.FLAG_IDLE_UNTIL: 
    - doze 模式的闹钟，只能由系统通过 setIdleUtil 来设置，这个方法会使得系统进入 idle 状态，直到这个 alarm 触发；如果系统中已经有 FLAG_WAKE_FROM_IDLE 类型的 alarm，那么 FLAG_IDLE_UNTIL 类型的 alarm 的触发事件会提前！

</br>

- AlarmManager.FLAG_STANDALONE: 
    - 精确闹钟，如果设置 alarm 时，指定了闹钟为精确闹钟：WINDOW_EXACT，那么该 flags 会被设置 FLAG_STANDALONE 标志位

</br>

- AlarmManager.FLAG_ALLOW_WHILE_IDLE：
    - 即使设备处于 idle 状态，该 alarm 也能触发，通过 setAndAllowWhileIdle 和 setExactAndAllowWhileIdle 设置！

</br>

- AlarmManager.FLAG_ALLOW_WHILE_IDLE_UNRESTRICTED
    - 如果是系统应用， 或者调用者在 doze 模式的白名单中，那么会设置 FLAG_ALLOW_WHILE_IDLE_UNRESTRICTED 标志位，而不是 FLAG_ALLOW_WHILE_IDLE 标志位！

**方法流程总结**：

- 通过 appOps 校验，uid 和包名是否一致；
- 重复触发的 alarm 必须要使用 PendingIntent，不能使用 directReceiver！
- 如果 workSource 不为 null，调用者必须有 android.Manifest.permission.UPDATE_DEVICE_STATS 的权限！
- 非 system uid 的调用者，其不能调用 setIdleUntil 方法设置 FLAG_IDLE_UNTIL！
- 如果是精确闹钟，那就设置 AlarmManager.FLAG_STANDALONE 标志位！
- 如果是 alarmClock，那就设置 AlarmManager.FLAG_WAKE_FROM_IDLE 和 AlarmManager.FLAG_STANDALONE！
- 如果是系统应用， 或者调用者在 doze 模式的白名单中，那么会设置 FLAG_ALLOW_WHILE_IDLE_UNRESTRICTED 标志位，而不是 FLAG_ALLOW_WHILE_IDLE 标志位！
- 最后，调用 setImpl 方法继续设置！

可以看到，我们在 AlarmManager 中调用的 set 接口，会调用该“桩”对象的 set 方法，桩对象的 set 最后会调用 AlarmManagerService 的 setImpl 方法！

## 3.2 AlarmMS.setImpl

setImpl 方法中，首先会做一些参数校验！

```java
    void setImpl(int type, long triggerAtTime, long windowLength, long interval,
            PendingIntent operation, IAlarmListener directReceiver, String listenerTag,
            int flags, WorkSource workSource, AlarmManager.AlarmClockInfo alarmClock,
            int callingUid, String callingPackage) {

        //【1】校验 PendingIntent 和 directReceiver 是否设置正确！
        if ((operation == null && directReceiver == null)
                || (operation != null && directReceiver != null)) {
            Slog.w(TAG, "Alarms must either supply a PendingIntent or an AlarmReceiver");
            return;
        }

        //【2】校验时间窗的长度
        // 如果时间窗长度超过 12h，那就设置为 1h
        if (windowLength > AlarmManager.INTERVAL_HALF_DAY) {
            Slog.w(TAG, "Window length " + windowLength
                    + "ms suspiciously long; limiting to 1 hour");
            windowLength = AlarmManager.INTERVAL_HOUR;
        }

        //【3】校验重复触发的时间间隔，最小的重复触发的时间间隔为 1min!
        final long minInterval = mConstants.MIN_INTERVAL;
        if (interval > 0 && interval < minInterval) {
            Slog.w(TAG, "Suspiciously short interval " + interval
                    + " millis; expanding to " + (minInterval/1000)
                    + " seconds");
            interval = minInterval;
        }

        //【4】校验闹钟的类型 type 是否设置的正确！
        if (type < RTC_WAKEUP || type > ELAPSED_REALTIME) {
            throw new IllegalArgumentException("Invalid alarm type " + type);
        }

        //【5】校验触发时间是否正确！
        if (triggerAtTime < 0) {
            final long what = Binder.getCallingPid();
            Slog.w(TAG, "Invalid alarm trigger time! " + triggerAtTime + " from uid=" + callingUid
                    + " pid=" + what);
            triggerAtTime = 0;
        }
        
        //【3.2.1】计算触发时间，根据闹钟是否是 RTC 类型的，进行调整！
        // 如果是 RTC 格式，就将其转换成 elapsedRealtime 格式的触发时间！
        final long nowElapsed = SystemClock.elapsedRealtime();
        final long nominalTrigger = convertToElapsed(triggerAtTime, type);

        //【7】校验触发时间是否大于最低触发时间，默认是 5s，防止 alarm 频繁触发！
        final long minTrigger = nowElapsed + mConstants.MIN_FUTURITY;
        final long triggerElapsed = (nominalTrigger > minTrigger) ? nominalTrigger : minTrigger;

        //【8】根据时间窗取值，设置最晚触发时间！
        final long maxElapsed;
        if (windowLength == AlarmManager.WINDOW_EXACT) {
            //【8.1】如果取值为 WINDOW_EXACT，那就为精确闹钟，最晚触发时间和设置的时间一样！
            maxElapsed = triggerElapsed;

        } else if (windowLength < 0) {
            //【3.2.2】如果取值小于 0，其为非精确闹钟，那么这里会计算最晚触发时间！
            // 并根据最晚触发时间重新计算时间窗！
            maxElapsed = maxTriggerTime(nowElapsed, triggerElapsed, interval);
            windowLength = maxElapsed - triggerElapsed;

        } else {
            //【8.3】其他情况，也是非精确闹钟，说明用户显示指定了时间窗，那就依此计算最晚触发时间！
            maxElapsed = triggerElapsed + windowLength;
        }

        synchronized (mLock) {
            if (DEBUG_BATCH) {
                Slog.v(TAG, "set(" + operation + ") : type=" + type
                        + " triggerAtTime=" + triggerAtTime + " win=" + windowLength
                        + " tElapsed=" + triggerElapsed + " maxElapsed=" + maxElapsed
                        + " interval=" + interval + " flags=0x" + Integer.toHexString(flags));
            }
            //【3.1】调用 setImplLocked 继续设置 alarm！
            setImplLocked(type, triggerAtTime, triggerElapsed, windowLength, maxElapsed,
                    interval, operation, directReceiver, listenerTag, flags, true, workSource,
                    alarmClock, callingUid, callingPackage);
        }
    }
```
我们来看看这个过程：

- 校验 PendingIntent 和 directReceiver 是否设置正确，二者只能设置一个！
- 检验时间窗，如果 windowLength 超过 12h，那就设置为 1h！
- 校验重复触发的时间间隔，最小的重复触发的时间间隔为 1min!
- 调整触发时间
    - 如果是 RTC 类型的闹钟，将触发事件转为相对于开机的时间！
    - 触发时间最短是 5s

- 调整最晚触发时间和时间窗
    - 精确闹钟，最晚触发时间和设置的时间一样，无时间窗！
    - 非精确闹钟，如果没有显式设置时间窗，那么就根据 triggerElapsed，nowElapsed 和 interval 计算合适的最晚触发时间和时间窗！
    - 非精确闹钟，如果显式设置了时间窗。那么最晚触发时间为：triggerElapsed + windowLength！

- 最后，调用 setImplLocked 进一步设置 alarm！
    
### 3.2.1 AlarmMS.convertToElapsed

对于触发事件，要根据闹钟的类型，来修正：
```java
    static long convertToElapsed(long when, int type) {
        //【1】判断是否是 RTC 闹钟类型！
        final boolean isRtc = (type == RTC || type == RTC_WAKEUP);
        //【2】如果是 RTC 类型的，就将其转为相对于开机的时间点！
        if (isRtc) {
            when -= System.currentTimeMillis() - SystemClock.elapsedRealtime();
        }
        return when;
    }
```
计算结果：
when = when - System.currentTimeMillis() + SystemClock.elapsedRealtime();

如果是 rtc 类型，那就将其转为了相对于开机的时间！

### 3.2.2 AlarmMS.maxTriggerTime

接着是计算最大的触发时间，这里的 MIN_FUZZABLE_INTERVAL 表示的是最小的时间窗间隔！

```java
    // minimum recurrence period or alarm futurity for us to be able to fuzz it
    static final long MIN_FUZZABLE_INTERVAL = 10000;
```
继续来看：
```java
    static long maxTriggerTime(long now, long triggerAtTime, long interval) {
        //【1】如果没有设置重复的时间间隔 interval，那么时间间隔为触发时间 triggerAtTime - 当前时间 now！
        // 否则，时间间隔为 interval！
        long futurity = (interval == 0)
                ? (triggerAtTime - now)
                : interval;
        //【2】如果计算出的时间间隔 futurity 小于 10s，那么设置其为 0；
        if (futurity < MIN_FUZZABLE_INTERVAL) {
            futurity = 0;
        }
        //【3】最后，最大的触发时间点为：开始触发时间点 + 0.75 倍的时间间隔 futurity！
        return triggerAtTime + (long)(.75 * futurity);
    }
```
对于非精确的 alarm，这里通过 maxTriggerTime 方法计算其批处理的时间窗为：
```java
0.75 * (interval or triggerAtTime - now)
```

如果计算出的时间窗小于 10 s，那么就不设置最晚执行时间！

## 3.3 AlarmMS.setImplLocked[15]

**参数传递**：

- **int type**：闹钟的类型
- **long when**: 触发时间点；
- **long whenElapsed**：触发时间点，相对于开机时间；
- **long windowLength**：时间窗；
- **long maxWhen**：最晚触发的时间点，触发时间点，相对于开机时间！
- **long interval**：重复触发的时间间隔；
- **PendingIntent operation**：这个很简单，不多说！
- **IAlarmListener directReceiver**：这个很简单，也不多说！
- **String listenerTag**：AlarmListener 的字符串描述信息！
- **int flags**：alarm 属性标志位！
- **boolean doValidate**：传入 true！
- **WorkSource workSource**：工作源对象！
- **AlarmManager.AlarmClockInfo alarmClock**：通过 setAlarmClock 方法设置才不为 null；
- **int callingUid**：调用者的 uid；
- **String callingPackage**：调用者的 package name；

对于参数，就简单的介绍到这里！

```java
    private void setImplLocked(int type, long when, long whenElapsed, long windowLength,
            long maxWhen, long interval, PendingIntent operation, IAlarmListener directReceiver,
            String listenerTag, int flags, boolean doValidate, WorkSource workSource,
            AlarmManager.AlarmClockInfo alarmClock, int callingUid, String callingPackage) {
        //【3.3.1】创建一个 Alarm 对象！
        Alarm a = new Alarm(type, when, whenElapsed, windowLength, maxWhen, interval,
                operation, directReceiver, listenerTag, workSource, flags, alarmClock,
                callingUid, callingPackage);
        try {
            // 如果不允许设置 alarm，就退出！
            if (ActivityManagerNative.getDefault().getAppStartMode(callingUid, callingPackage)
                    == ActivityManager.APP_START_MODE_DISABLED) {
                Slog.w(TAG, "Not setting alarm from " + callingUid + ":" + a
                        + " -- package not allowed to start");
                return;
            }
        } catch (RemoteException e) {
        }

        //【3.3.2】移除已经设置的相同的 alarm！
        removeLocked(operation, directReceiver);

        //【3.3.3】进一步设置 alarm！
        setImplLocked(a, false, doValidate);
    }
```

### 3.3.1 AlarmMS.Alarm

下面是会创建一个 Alarm 对象，保存本次 set 的 alarm 的信息：

```java
    private static class Alarm {
        public final int type;
        public final long origWhen;
        public final boolean wakeup;
        public final PendingIntent operation;
        public final IAlarmListener listener;
        public final String listenerTag;
        public final String statsTag;
        public final WorkSource workSource;
        public final int flags;
        public final AlarmManager.AlarmClockInfo alarmClock;
        public final int uid;
        public final int creatorUid;
        public final String packageName;
        public int count;
        public long when;
        public long windowLength;
        public long whenElapsed;    // 'when' in the elapsed time base
        public long maxWhenElapsed; // also in the elapsed time base
        public long repeatInterval;
        public PriorityClass priorityClass;

        public Alarm(int _type, long _when, long _whenElapsed, long _windowLength, long _maxWhen,
                long _interval, PendingIntent _op, IAlarmListener _rec, String _listenerTag,
                WorkSource _ws, int _flags, AlarmManager.AlarmClockInfo _info,
                int _uid, String _pkgName) {

            type = _type; // 闹钟类型，对应的是 AlarmManager 中的四种类型！
            origWhen = _when; // 触发时间点！
            wakeup = _type == AlarmManager.ELAPSED_REALTIME_WAKEUP // 是否是 wake up 类型的！
                    || _type == AlarmManager.RTC_WAKEUP;
            when = _when; // 触发事件点！
            whenElapsed = _whenElapsed; // 触发时间点，相对于开机时间！
            windowLength = _windowLength; // 时间窗；
            maxWhenElapsed = _maxWhen; // 最晚触发的时间点，触发时间点，相对于开机时间！
            repeatInterval = _interval; // 重复触发的时间间隔！
            operation = _op;
            listener = _rec;
            listenerTag = _listenerTag;
            statsTag = makeTag(_op, _listenerTag, _type);
            workSource = _ws;
            flags = _flags; // 闹钟属性的标志位！
            alarmClock = _info;
            uid = _uid;
            packageName = _pkgName;
            creatorUid = (operation != null) ? operation.getCreatorUid() : uid; // 创建 intent 的 uid
        }
        
        public static String makeTag(PendingIntent pi, String tag, int type) {
            final String alarmString = type == ELAPSED_REALTIME_WAKEUP || type == RTC_WAKEUP
                    ? "*walarm*:" : "*alarm*:";
            return (pi != null) ? pi.getTag(alarmString) : (alarmString + tag);
        }
        ... ... ...
    }

```
Alarm 有几个方法，我们来看下：

#### 3.3.1.1 Alarm.make
```java
        public WakeupEvent makeWakeupEvent(long nowRTC) {
            return new WakeupEvent(nowRTC, creatorUid,
                    (operation != null)
                        ? operation.getIntent().getAction()
                        : ("<listener>:" + listenerTag));
        }
```
这个方法是用来创建一个 wake up event！

#### 3.3.1.2 Alarm.matches
```java
        // Returns true if either matches
        public boolean matches(PendingIntent pi, IAlarmListener rec) {
            return (operation != null)
                    ? operation.equals(pi)
                    : rec != null && listener.asBinder().equals(rec.asBinder());
        }

        public boolean matches(String packageName) {
            return (operation != null)
                    ? packageName.equals(operation.getTargetPackage())
                    : packageName.equals(this.packageName);
        }
```
这个方法是用来匹配 Alarm 的，代码逻辑很简单，使用 packageName 或者 PendingIntent，AlarmListener 进行匹配！

我们继续来看：

### 3.3.2 AlarmMS.removeLocked

removeLocked 方法用于移除一个已经存在的 alarm！

```java
    private void removeLocked(PendingIntent operation, IAlarmListener directReceiver) {
        boolean didRemove = false;
        //【1】遍历 mAlarmBatches 中所有的 Batch 批处理对象，从 Batch 中移除该 alarm！
        for (int i = mAlarmBatches.size() - 1; i >= 0; i--) {
            Batch b = mAlarmBatches.get(i);
            //【3.3.2.1】匹配并移除成功会返回 true！
            didRemove |= b.remove(operation, directReceiver);
            if (b.size() == 0) { // 如果该 Batch 中没有了其他的 alarm，就移除这个 Batch！
                mAlarmBatches.remove(i);
            }
        }

        //【2】遍历 mPendingWhileIdleAlarms 中所有因为进入 idle 状态而等待执行的 Alarm 对象！
        // 移除和本次 alarm 匹配的 alarm！
        for (int i = mPendingWhileIdleAlarms.size() - 1; i >= 0; i--) {
            if (mPendingWhileIdleAlarms.get(i).matches(operation, directReceiver)) {
                mPendingWhileIdleAlarms.remove(i);
            }
        }

        //【3】如果 didRemove 为 true，表示确实是移除了一个相同的 alarm！
        if (didRemove) {
            if (DEBUG_BATCH) {
                Slog.v(TAG, "remove(operation) changed bounds; rebatching");
            }
            boolean restorePending = false; // 表示是否恢复正在等待中的 alarm

            // 如果 mPendingIdleUntil 不为 null，且移除的 alarm 和 mPendingIdleUntil 匹配，那就说明
            // 要退出 idle 状态！
            if (mPendingIdleUntil != null && mPendingIdleUntil.matches(operation, directReceiver)) {
                // 那就设置 mPendingIdleUntil 为 null，同时设置 restorePending 为 true！
                mPendingIdleUntil = null;
                restorePending = true;
            }
            
            // 如果 mNextWakeFromIdle 不为 null，且移除的 alarm 和 mPendingIdleUntil 匹配，
            // 那就设置 mNextWakeFromIdle 为 null！！
            if (mNextWakeFromIdle != null && mNextWakeFromIdle.matches(operation, directReceiver)) {
                mNextWakeFromIdle = null;
            }
            
            //【3.3.2.2】对其他的 alarm 重新进行批处理分配！
            rebatchAllAlarmsLocked(true);
            
            //【3.3.2.3】如果 restorePending 为 true，说明从 idle 状态恢复了，那就要恢复 mPendingWhileIdleAlarms
            // 中所有 alarm！
            if (restorePending) {
                restorePendingWhileIdleAlarmsLocked();
            }
            
            //【3.4.5】更新下一个 AlarmClock 的时间！
            updateNextAlarmClockLocked();
        }
    }
```
这边是匹配到和本次设置的 alarm 相同的 alarm，然后做移除操作，然后恢复一些需要执行的 alarm！

#### 3.3.2.1 AlarmMS.Batch.remove

从一个 Batch 中移除一个 alarm，Batch 中的非精确 alarm 是按照触发事件排序的，同时 Batch 也有一个 start 变量，表示批处理内部所有 alarm 的时间点！

```java
        boolean remove(final PendingIntent operation, final IAlarmListener listener) {
            if (operation == null && listener == null) {
                if (localLOGV) {
                    Slog.w(TAG, "requested remove() of null operation",
                            new RuntimeException("here"));
                }
                return false;
            }
            boolean didRemove = false;
            long newStart = 0;
            long newEnd = Long.MAX_VALUE;
            int newFlags = 0;
            for (int i = 0; i < alarms.size(); ) {
                Alarm alarm = alarms.get(i);
                //【3.3.1.2】这里用到了 alarm 的 matches 匹配方法，前面有说过！
                if (alarm.matches(operation, listener)) {
                    alarms.remove(i);
                    //【1】设置 didRemove 为 true；
                    didRemove = true;

                    //【2】如果匹配的 alarm 是通过 setAlarmClock 设置的，就设置 mNextAlarmClockMayChange 为 true！
                    if (alarm.alarmClock != null) {
                        mNextAlarmClockMayChange = true;
                    }
                } else {
                    if (alarm.whenElapsed > newStart) { // 更新 newStart 时间点！
                        newStart = alarm.whenElapsed;
                    }
                    if (alarm.maxWhenElapsed < newEnd) { // 更新 newStart 时间点！
                        newEnd = alarm.maxWhenElapsed;
                    }
                    newFlags |= alarm.flags; // 获得该 Batch 中所有 Alarm 的标志位！
                    i++;
                }
            }
            //【3】如果移除了，那就更新 Batch 的 start 和 end 时间点！
            if (didRemove) {
                start = newStart;
                end = newEnd;
                flags = newFlags;
            }
            return didRemove;
        }
```
我们可以看到：

- 从这个 batch 移除 operation 或者 listener 匹配的 alarm，并更新 batch 的 start， end 和 flags 对象！

- 如果被移除的 alarm 是通过 setAlarmClock 设置的，那么 mNextAlarmClockMayChange 为 true，后面系统会调用指定方法来更新下一个 AlarmClock！

我们可以看到，当我们从 Batch 中移除一个 Alarm 后，Batch 的 start 和 end 时间点发生了变化，这样会导致 Batch 在 mAlarmBatches 中的顺序发生变化！

#### 3.3.2.2 AlarmMS.rebatchAllAlarmsLocked

该方法用于重新对所有的 alarm 进行 batch 批处理！

```java
    void rebatchAllAlarmsLocked(boolean doValidate) {
        //【1】将 mAlarmBatches 拷贝一份到 oldSet 中，并清空 mAlarmBatches！
        ArrayList<Batch> oldSet = (ArrayList<Batch>) mAlarmBatches.clone();
        mAlarmBatches.clear();

        Alarm oldPendingIdleUntil = mPendingIdleUntil;
        final long nowElapsed = SystemClock.elapsedRealtime();
        final int oldBatches = oldSet.size();

        //【2】遍历每一个 Batch 的中 Alarm，重新进行批处理分配！
        for (int batchNum = 0; batchNum < oldBatches; batchNum++) {
            Batch batch = oldSet.get(batchNum);
            final int N = batch.size();
            for (int i = 0; i < N; i++) {
                //【3.3.2.1.1】重新添加 Alarm！
                reAddAlarmLocked(batch.get(i), nowElapsed, doValidate);
            }
        }
        //【3】如果 oldPendingIdleUntil 不为 null，而 mPendingIdleUntil 为 null
        // 说明，此时推出了 idle 状态，那就恢复那些因为 idle 状态而等待的 alarm！
        // × 这里感觉是为了解决 bug，因为看逻辑，不会进入这里
        if (oldPendingIdleUntil != null && oldPendingIdleUntil != mPendingIdleUntil) {
            Slog.wtf(TAG, "Rebatching: idle until changed from " + oldPendingIdleUntil
                    + " to " + mPendingIdleUntil);
            if (mPendingIdleUntil == null) {
                //【3.3.2.3】恢复等待中的 alarm！
                restorePendingWhileIdleAlarmsLocked();
            }
        }
        
        //【3.4.4】重新调度并设置下一个 alarm！
        rescheduleKernelAlarmsLocked();

        //【3.4.5】更新下一个 AlarmClock 闹钟！
        updateNextAlarmClockLocked();
    }
```


##### 3.3.2.2.1 AlarmManagerService.reAddAlarmLocked

该方法用于重新添加 alarm！

```java
    void reAddAlarmLocked(Alarm a, long nowElapsed, boolean doValidate) {
        a.when = a.origWhen;
        long whenElapsed = convertToElapsed(a.when, a.type);
        final long maxElapsed;
        if (a.windowLength == AlarmManager.WINDOW_EXACT) { // 根据类型的不同，计算最大触发时间！
            maxElapsed = whenElapsed; // 精确闹钟！
        } else {
            //【3.2.2】非精确闹钟！
            maxElapsed = (a.windowLength > 0)
                    ? (whenElapsed + a.windowLength)
                    : maxTriggerTime(nowElapsed, whenElapsed, a.repeatInterval);
        }
        a.whenElapsed = whenElapsed;
        a.maxWhenElapsed = maxElapsed;

        //【3.4】调用 setImplLocked 方法重新设置 alarm，注意这里的 rebatching 值为 true！
        setImplLocked(a, true, doValidate);
    }
```

可以看到这里调用了 setImplLocked 三参数方法，重新设置该 alarm！

在 setImplLocked 方法中，会重新为 Alarm 分配 Batch，并对 mAlarmBatches 中的所有 Batch 重新排序，后面我们能够看到对该方法的分析，在第 3.4 节！

#### 3.3.2.3 AlarmMS.restorePendingWhileIdleAlarmsLocked

接着，当系统退出 idle 状态后，要恢复那些因为 idle 状态而等待触发的 alarm！

```java
    void restorePendingWhileIdleAlarmsLocked() {
        if (RECORD_DEVICE_IDLE_ALARMS) { // RECORD_DEVICE_IDLE_ALARMS 默认为 false，用于 dump 不处理；
            IdleDispatchEntry ent = new IdleDispatchEntry();
            ent.uid = 0;
            ent.pkg = "FINISH IDLE";
            ent.elapsedRealtime = SystemClock.elapsedRealtime();
            mAllowWhileIdleDispatches.add(ent);
        }

        // 如果此时 mPendingWhileIdleAlarms 不为 empty，那就说明有等待触发的 alarm！
        // 将其重新进行批处理分配！
        if (mPendingWhileIdleAlarms.size() > 0) {
            ArrayList<Alarm> alarms = mPendingWhileIdleAlarms;
            mPendingWhileIdleAlarms = new ArrayList<>();
            final long nowElapsed = SystemClock.elapsedRealtime();
            for (int i=alarms.size() - 1; i >= 0; i--) {
                Alarm a = alarms.get(i);
                //【3.3.2.1.1】这里调用了 reAddAlarmLocked 方法，重新批处理 alarm，该方法上面有说过！
                reAddAlarmLocked(a, nowElapsed, false);
            }
        }

        // 更新 ALLOW_WHILE_IDLE 最小时间！
        mConstants.updateAllowWhileIdleMinTimeLocked();

        //【3.3.2.4】重新调度，设置 alarm！
        rescheduleKernelAlarmsLocked();
        //【3.3.2.5】更新下一个 Alarm 的触发时间！
        updateNextAlarmClockLocked();

        // 发送一个时间改变的广播，用于更新 ui！
        try {
            mTimeTickSender.send();
        } catch (PendingIntent.CanceledException e) {
        }
    }

```
继续看！

## 3.4 AlarmMS.setImplLocked[3]

接下来，我们来看一个非常重要的方法 setImplLocked！，第二个参数 rebatching 表示是否是 rebatch，正常流程下，rebatching 是为 false；

但是，当我们 rebatch 或者 restore 其他 alarm 时，rebatching 传入的是 true！

```java
    private void setImplLocked(Alarm a, boolean rebatching, boolean doValidate) {
        //【1】如果 alarm 设置了 AlarmManager.FLAG_IDLE_UNTIL 标志位，说明是 setIdleUtil 方法设置的！
        // 这是一种很特殊的闹钟，用于将系统设置为 idle 状态，但其触发后退出 idle 状态！
        // 如果有其他 alarm 会将系统从 idle 状态唤醒的话，并且其触发事件比 setIdleUtil 更早的的话，我们需要将
        // setIdleUtil 闹钟的时间提前！
        if ((a.flags&AlarmManager.FLAG_IDLE_UNTIL) != 0) {
        
            // 如果 mNextWakeFromIdle 不为 null，说明有 alarm 会在设备处于 idle 的状态下唤醒设备
            // 如果 mNextWakeFromIdle 的触发时间更早，那么需要调整 setIdleUtil 的时间为 mNextWakeFromIdle 的触发时间！
            if (mNextWakeFromIdle != null && a.whenElapsed > mNextWakeFromIdle.whenElapsed) {
                // 因为是精确闹钟，所以所有时间相同！
                a.when = a.whenElapsed = a.maxWhenElapsed = mNextWakeFromIdle.whenElapsed;
            }

            //【3.4.1】接着对 setIdleUtil 闹钟的触发时间做一个细微调整，将前面设置的触发时间提前一个时间段！
            final long nowElapsed = SystemClock.elapsedRealtime();
            final int fuzz = fuzzForDuration(a.whenElapsed-nowElapsed);
            if (fuzz > 0) {
                if (mRandom == null) {
                    mRandom = new Random();
                }
                // 可以看到，最后进一步调整的提前时间间隔为 0 到 fuzz 的一个随机整数时间，单位是分钟！
                final int delta = mRandom.nextInt(fuzz);
                a.whenElapsed -= delta;
                if (false) {
                    Slog.d(TAG, "Alarm when: " + a.whenElapsed);
                    Slog.d(TAG, "Delta until alarm: " + (a.whenElapsed-nowElapsed));
                    Slog.d(TAG, "Applied fuzz: " + fuzz);
                    Slog.d(TAG, "Final delta: " + delta);
                    Slog.d(TAG, "Final when: " + a.whenElapsed);
                }
                a.when = a.maxWhenElapsed = a.whenElapsed;
            }

        } else if (mPendingIdleUntil != null) {
            // 如果没有设置 FLAG_IDLE_UNTIL 标志位，那么就要判断下 mPendingIdleUntil 是否为 null；
            // 其不为 null，说明系统现在处于 idle 状态，如果 alarm 没有设置一下的几种标志位的话，
            // 那就将其加入到 mPendingWhileIdleAlarms 列表中，等到系统退出 idle 状态后，再设置这些闹钟！
            if ((a.flags&(AlarmManager.FLAG_ALLOW_WHILE_IDLE
                    | AlarmManager.FLAG_ALLOW_WHILE_IDLE_UNRESTRICTED
                    | AlarmManager.FLAG_WAKE_FROM_IDLE))
                    == 0) {
                mPendingWhileIdleAlarms.add(a);
                return;
            }
        }

        // RECORD_DEVICE_IDLE_ALARMS 变量默认为 false，如果为 true，那么系统会记录那些在 idle 状态下能够触发的 alarm！
        // 保存到 mAllowWhileIdleDispatches 中，我们在 dumpsys alarm 的时候能看到！
        if (RECORD_DEVICE_IDLE_ALARMS) {
            if ((a.flags & AlarmManager.FLAG_ALLOW_WHILE_IDLE) != 0) {
                IdleDispatchEntry ent = new IdleDispatchEntry();
                ent.uid = a.uid;
                ent.pkg = a.operation.getCreatorPackage();
                ent.tag = a.operation.getTag("");
                ent.op = "SET";
                ent.elapsedRealtime = SystemClock.elapsedRealtime();
                ent.argRealtime = a.whenElapsed;
                mAllowWhileIdleDispatches.add(ent);
            }
        }

        //【3.4.2】为新的 alarm 选择一个合适的 Batch，
        // 如果闹钟的 alarm 的 flags 设置了 AlarmManager.FLAG_STANDALONE 标志
        // 那么其为精确闹钟，那么必须单独在一个 Batch 中！
        int whichBatch = ((a.flags&AlarmManager.FLAG_STANDALONE) != 0)
                ? -1 : attemptCoalesceLocked(a.whenElapsed, a.maxWhenElapsed);
                
        if (whichBatch < 0) {
            //【3.4.3】创建一个新的 Batch，并添加到 mAlarmBatch，并对 mAlarmBatches 按开始时间升序排列！
            Batch batch = new Batch(a);
            addBatchLocked(mAlarmBatches, batch);

        } else {
            //【3.4.3】如果 whichBatch >= 0，说明已经找到合适的 Batch 了，那我们就将其添加进去
            Batch batch = mAlarmBatches.get(whichBatch);

            // 如果当前 alarm 的加入使得了 batch 开始时间和结束时间的改变，则 add 返回 true
            // 那么此时，我们需要对 mAlarmBatches 重新排序！
            if (batch.add(a)) {

                // 排序方法很简单，先移除这个 Batch，再重新添加！
                mAlarmBatches.remove(whichBatch);
                addBatchLocked(mAlarmBatches, batch);
            }
        }

        // 如果 a.alarmClock 不为 null，则是通过 setAlarmClock 设置的，那就设置 mNextAlarmClockMayChange 为 true！
        // 下面会根据这个变量更新 AlarmClock！
        if (a.alarmClock != null) {
            mNextAlarmClockMayChange = true;
        }

        boolean needRebatch = false;

        // 如果是通过 setIdleUtil 设置的闹钟，系统此时将进入 idle 状态！
        if ((a.flags&AlarmManager.FLAG_IDLE_UNTIL) != 0) {
            if (RECORD_DEVICE_IDLE_ALARMS) {
                if (mPendingIdleUntil == null) {
                    IdleDispatchEntry ent = new IdleDispatchEntry();
                    ent.uid = 0;
                    ent.pkg = "START IDLE";
                    ent.elapsedRealtime = SystemClock.elapsedRealtime();
                    mAllowWhileIdleDispatches.add(ent);
                }
            }
            // 设置 mPendingIdleUntil，更新 allow while idle 最小时间！
            // 因为此时系统进入了 idle 状态，需要 rebatch 其他的闹钟，needRebatch 置为 true！
            mPendingIdleUntil = a;
            mConstants.updateAllowWhileIdleMinTimeLocked();
            needRebatch = true;


        } else if ((a.flags&AlarmManager.FLAG_WAKE_FROM_IDLE) != 0) {
            // 如果该 alarm 是会在 idle 状态下唤醒，如果此时 mNextWakeFromIdle 为 null，或者
            // mNextWakeFromIdle 的触发时间晚，那就更新 mNextWakeFromIdle 为本次设置的新的 alarm！
            if (mNextWakeFromIdle == null || mNextWakeFromIdle.whenElapsed > a.whenElapsed) {
                mNextWakeFromIdle = a;

                // If this wake from idle is earlier than whatever was previously scheduled,
                // and we are currently idling, then we need to rebatch alarms in case the idle
                // until time needs to be updated.
                // 如果此时
                if (mPendingIdleUntil != null) {
                    needRebatch = true;
                }
            }
        }
        
        // 如果 rebatching 为 true，表示本次是 rebatch 恢复操作，那么就不会进入下面的分支，原因很简单
        // 因为 rebatch 在前面就已经执行的相应的操作了！
        if (!rebatching) {
            if (DEBUG_VALIDATE) { // 用于 debug。
                if (doValidate && !validateConsistencyLocked()) {
                    Slog.v(TAG, "Tipping-point operation: type=" + a.type + " when=" + a.when
                            + " when(hex)=" + Long.toHexString(a.when)
                            + " whenElapsed=" + a.whenElapsed
                            + " maxWhenElapsed=" + a.maxWhenElapsed
                            + " interval=" + a.repeatInterval + " op=" + a.operation
                            + " flags=0x" + Integer.toHexString(a.flags));
                    rebatchAllAlarmsLocked(false);
                    needRebatch = false;
                }
            }

            if (needRebatch) { // 如果 needRebatch 为 true，那么我们要重新批处理所有的 alarm！
                rebatchAllAlarmsLocked(false);
            }
            // 调度 kernel 设置 alarm！
            rescheduleKernelAlarmsLocked();
            // 更新下一个 AlarmClock！
            updateNextAlarmClockLocked();
        }
    }

```
该方法的主要逻辑如下：



### 3.4.1 AlarmMS.fuzzForDuration
我们来看看 fuzzForDuration 方法：

```java
    static int fuzzForDuration(long duration) {
        if (duration < 15*60*1000) {
            //【1】如果 duration 小于 15 分钟，那么我们要调整的时间间隔为 duration
            return (int)duration;
        } else if (duration < 90*60*1000) {
            //【2】如果 duration 大于等于 15 分钟，小于 1 小时 30 分钟，那么我们要调整的时间间隔为 15 分钟！
            return 15*60*1000;
        } else {
            //【2】如果 duration 大于等于 1 小时 30 分钟，那调整的时间间隔为 30 分钟！
            return 30*60*1000;
        }
    }
```

### 3.4.2 AlarmMS.attemptCoalesceLocked

通过非精确闹钟的触发时间，找到一个合适的 Batch ！

```java
    int attemptCoalesceLocked(long whenElapsed, long maxWhen) {
        final int N = mAlarmBatches.size();
        for (int i = 0; i < N; i++) {
            Batch b = mAlarmBatches.get(i);
            // 匹配到合适的 Batch，那就返回该 Batch 下标！
            if ((b.flags&AlarmManager.FLAG_STANDALONE) == 0 && b.canHold(whenElapsed, maxWhen)) {
                return i;
            }
        }
        // 无法找到一个合适的，就返回 -1，创建一个新的！
        return -1;
    }
```
可以看到，一个合适的 Batch 满足的条件如下：

- Batch 的 flags 没有 AlarmManager.FLAG_STANDALONE 标志位，即该 Batch 是只能用于保存非精确闹钟！
- canHold 方法返回 true，即：这个 Batch 能够容纳这个 Alarm！

下面，我们来看看 canHold 方法的逻辑：

#### 3.4.2.1 AlarmMS.Batch.canHold

canHold 方法用来判断，该 alarm 是否可以加入到这个 Batch 中：
```java
        boolean canHold(long whenElapsed, long maxWhen) {
            return (end >= whenElapsed) && (start <= maxWhen);
        }
```
从方法中可以看出，可以容纳的依据是：

- batch.end >= alarm.whenElapsed
- batch.start <= alarm.maxWhen

即：batch 的时间间隔和 alarm 的触发时间间隔必须有交集！！


### 3.4.3 AlarmMS.addBatchLocked

我们来看看 addBatchLocked 方法的逻辑：

```java
    static boolean addBatchLocked(ArrayList<Batch> list, Batch newBatch) {
        //【1】其实就是根据 Batch 的 start 时间，查找到一个更合适的 index！
        int index = Collections.binarySearch(list, newBatch, sBatchOrder);
        if (index < 0) {
            index = 0 - index - 1;
        }
        //【2】插入 index 位置处！
        list.add(index, newBatch);
        //【3】如果返回 true，表示 Batch 处于 list 的开头位置！
        return (index == 0);
    }
```

这里的 sBatchOrder 是一个 BatchTimeOrder 实例，实现了 Comparator 接口，用来比较两个 Batch 的 start 时间的大小！

```java
static final BatchTimeOrder sBatchOrder = new BatchTimeOrder();

    static class BatchTimeOrder implements Comparator<Batch> {
        public int compare(Batch b1, Batch b2) {
            long when1 = b1.start;
            long when2 = b2.start;
            if (when1 > when2) {
                return 1;
            }
            if (when1 < when2) {
                return -1;
            }
            return 0;
        }
    }
```
可以看到，mAlarmBatches 中的 Batch 是按照开始时间从小到大排序的！！

下面我们来看看 Batch 的相关方法：

#### 3.4.3.1 AlarmMS.Batch.Batch

当我们要将一个 Alarm 添加到新创建的 Batch 中的时候，会对 Batch 进行初始化：

```java
        Batch() {
            start = 0;
            end = Long.MAX_VALUE;
            flags = 0;
        }

        Batch(Alarm seed) {
            // 初始化 start 为 alarm 的开始触发时间，end 为 alarm 的最大开始触发时间
            start = seed.whenElapsed;
            end = seed.maxWhenElapsed;
            flags = seed.flags;
            alarms.add(seed);
        }
```
不多说了，继续看！

#### 3.4.3.2 AlarmMS.Batch.add

将一个 Alarm 添加到已存在的一个 Batch，通过 add 方法：
```java
        boolean add(Alarm alarm) {
            boolean newStart = false;
            //【1】按照开始触发时间递增的顺序，给这个 alarm 找到合适的位置！
            int index = Collections.binarySearch(alarms, alarm, sIncreasingTimeOrder);
            if (index < 0) {
                index = 0 - index - 1;
            }
            //【2】添加到指定的位置！
            alarms.add(index, alarm);
            if (DEBUG_BATCH) {
                Slog.v(TAG, "Adding " + alarm + " to " + this);
            }
            //【3】更改 Batch 的 start 和 end 时间！
            if (alarm.whenElapsed > start) {
                start = alarm.whenElapsed;
                newStart = true;
            }
            if (alarm.maxWhenElapsed < end) {
                end = alarm.maxWhenElapsed;
            }
            //【4】更新 Batch 的 flags！
            flags |= alarm.flags;

            if (DEBUG_BATCH) {
                Slog.v(TAG, "    => now " + this);
            }
            return newStart;
        }
```
可以看到，如果 Batch 的 start 时间发生变化，那么 add 会返回 true！

这里用到了一个比较器对象：
```java
static final IncreasingTimeOrder sIncreasingTimeOrder = new IncreasingTimeOrder();

public static class IncreasingTimeOrder implements Comparator<Alarm> {
        public int compare(Alarm a1, Alarm a2) {
            long when1 = a1.whenElapsed;
            long when2 = a2.whenElapsed;
            if (when1 > when2) {
                return 1;
            }
            if (when1 < when2) {
                return -1;
            }
            return 0;
        }
}
```
sIncreasingTimeOrder 是用来对 Batch 中的 alarm 进行排序的，规则是按照开始时间递增的顺序！

### 3.4.4 AlarmMS.rescheduleKernelAlarmsLocked

该方法用于设置下一个 alarm！

```java
    void rescheduleKernelAlarmsLocked() {
        long nextNonWakeup = 0;
        // 设置最早的 ELAPSED_REALTIME_WAKEUP 类型的 alarm！
        if (mAlarmBatches.size() > 0) {

            //【3.4.4.1】从 mAlarmBatches 中找到第一个包含 wake up 类型 alarm 的 Batch！
            final Batch firstWakeup = findFirstWakeupBatchLocked();
            
            //【2】从 mAlarmBatches 中找到第一个的 Batch！
            final Batch firstBatch = mAlarmBatches.get(0);

            //【3】如果 firstWakeup 不为 null，且 firstWakeup 中的开始触发时间 start，不等于 mNextWakeup！
            // 那就要更新 mNextWakeup，调整下一个 wake up alarm 的触发时间！
            if (firstWakeup != null && mNextWakeup != firstWakeup.start) {
                // 更新 mNextWakeup 和 mLastWakeupSet！
                mNextWakeup = firstWakeup.start;
                mLastWakeupSet = SystemClock.elapsedRealtime();
                
                //【3.4.4.2】设置下一个要触发的 wake up 类型的 alarm！
                setLocked(ELAPSED_REALTIME_WAKEUP, firstWakeup.start);
            }

            //【4】如果 firstBatch 不等于 firstWakeup，说明 firstBatch 中不包含 wake up 类型的 alarm！
            // 那就设置下一个非 wake up alarm 的触发时间！
            if (firstBatch != firstWakeup) {
                nextNonWakeup = firstBatch.start;
            }
        }
        
        // mPendingNonWakeupAlarms 不为 empty，说明系统中存在处于等待状态的 no wakeup 类型的 alarm！
        // 那么如果 nextNonWakeup 为 0，或者 mNextNonWakeupDeliveryTime 小于 nextNonWakeup！
        // 那么更新 nextNonWakeup 为 mNextNonWakeupDeliveryTime 的值！
        if (mPendingNonWakeupAlarms.size() > 0) {
            if (nextNonWakeup == 0 || mNextNonWakeupDeliveryTime < nextNonWakeup) {
                nextNonWakeup = mNextNonWakeupDeliveryTime;
            }
        }
        
        // 最后如果，本次计算的下一个要触发的 no wake up alarm 的时间和之前的不同，更新 mNextNonWakeup 的值
        // 并设置下一个 no wake up 的 alarm！
        if (nextNonWakeup != 0 && mNextNonWakeup != nextNonWakeup) {
            mNextNonWakeup = nextNonWakeup;
            //【3.4.4.2】设置下一个要触发的 no wake up 类型的 alarm！
            setLocked(ELAPSED_REALTIME, nextNonWakeup);
        }
    }
```
**变量解释：**

- mNextWakeup 表示下一个最早的 wake up 类型 alarm 的触发时间；
- mLastWakeupSet 表示上一次设置 wake up 类型 alarm 的时间，取值为 SystemClock.elapsedRealtime()；
- mNextNonWakeup 表示的是下一个最早的 no wake up 类型 alarm 的触发时间；

通常，二者是一起设置的！

**逻辑梳理**：

- 该方法首先是确定下一个要触发的 wake up 和非 wake up 类型的 alarm 的触发时间：mNextWakeup 和 mNextNonWakeup！
   - 对于 wake up 类型，那就在所有的 Batch 中找到第一个持有 wake up 类型 alarm 的 Batch，其 Batch.start 就是 mNextWakeup！
   - 对于非 wake up 类型，确定 mNextNonWakeup 要分为以下几步：
        - 如果第一个持有 wake up 类型 alarm 的 Batch 不是 first batch，那么 firstBatch.Start 为可选值，保存到 nextNonWakeup 中！
        - 如果此时 mPendingNonWakeupAlarms 不为 empty，说明系统中有正在等待的 no wake up 类型的闹钟，如果此时 nextNonWakeup 为 0，或者 mNextNonWakeupDeliveryTime 小于 nextNonWakeup，那么 mNextNonWakeupDeliveryTime 就是一个更优的选择，保存到 nextNonWakeup 中；
        - 最后，如果 nextNonWakeup 和 mNextNonWakeup 不相等，那就使用 nextNonWakeup 更新 mNextNonWakeup！

</br>

- 然后，分别设置两种闹钟！

#### 3.4.4.1 AlarmMS.findFirstWakeupBatchLocked

findFirstWakeupBatchLocked 方法在所有的 Batch 中找到第一个包含 wake up 类型 alarm 的 Batch，然后返回！

```java
    private Batch findFirstWakeupBatchLocked() {
        final int N = mAlarmBatches.size();
        for (int i = 0; i < N; i++) {
            Batch b = mAlarmBatches.get(i);
            if (b.hasWakeups()) { // 判断该 Batch 是否包含 wake up 类型的 alarm！
                return b;
            }
        }
        return null;
    }
```
这里调用了 Batch 的 hasWakeups 接口：

##### 3.4.4.1.1 AlarmMS.Batch.hasWakeups

```java
        boolean hasWakeups() {
            final int N = alarms.size();
            // 遍历其内部的所有的 Alarm！
            for (int i = 0; i < N; i++) {
                Alarm a = alarms.get(i);
                // 这里的 type 取值为 AlarmManager 中的四种
                if ((a.type & TYPE_NONWAKEUP_MASK) == 0) {
                    return true;
                }
            }
            return false;
        }
```

#### 3.4.4.2 AlarmMS.setLocked

用于设置一个 alarm！
```java
    private void setLocked(int type, long when) {
        if (mNativeData != 0) {
            // 记录秒和纳秒！
            long alarmSeconds, alarmNanoseconds;
            if (when < 0) {
                alarmSeconds = 0;
                alarmNanoseconds = 0;
            } else {
                alarmSeconds = when / 1000;
                alarmNanoseconds = (when % 1000) * 1000 * 1000;
            }
            // 设置 alarm！
            set(mNativeData, type, alarmSeconds, alarmNanoseconds);

        } else {
            Message msg = Message.obtain();
            msg.what = ALARM_EVENT;
            
            mHandler.removeMessages(ALARM_EVENT);
            mHandler.sendMessageAtTime(msg, when);
        }
    }
```
我们可以看到，这里使用了 2 中方式来设置闹钟！

**第一种**：通过 Alarm 驱动来设置

mNativeData 不为 0，表示 Alarm 驱动存在，就直接调用 set 方法，通过 Alarm 驱动设置这个 alarm！

```java
    private native void set(long nativeData, int type, long seconds, long nanoseconds);
```

当然，正常情况，是通过 Alarm 驱动来设置的，因为这样能够实现休眠唤醒！

除非找不到 Alarm 驱动

mNativeData 为 0，说明 Alarm 驱动不存在，那就通过 Timer 定时器设置，这里通过发送一个 ALARM_EVENT 给 AlarmHandler，然后处理消息，设置 alarm！



#### 3.4.4.3 AlarmMS.updateNextAlarmInfoForUserLocked

```java
    private void updateNextAlarmInfoForUserLocked(int userId,
            AlarmManager.AlarmClockInfo alarmClock) {
        if (alarmClock != null) {
            if (DEBUG_ALARM_CLOCK) {
                Log.v(TAG, "Next AlarmClockInfoForUser(" + userId + "): " +
                        formatNextAlarm(getContext(), alarmClock, userId));
            }
            //【1】更新 alarmClock！
            mNextAlarmClockForUser.put(userId, alarmClock);
        } else {
            if (DEBUG_ALARM_CLOCK) {
                Log.v(TAG, "Next AlarmClockInfoForUser(" + userId + "): None");
            }
            //【1】移除指定的 userId 信息！
            mNextAlarmClockForUser.remove(userId);
        }
        
        // 将 mPendingSendNextAlarmClockChangedForUser 中 userId 对应的置为 true！
        mPendingSendNextAlarmClockChangedForUser.put(userId, true);
        
        mHandler.removeMessages(AlarmHandler.SEND_NEXT_ALARM_CLOCK_CHANGED);
        // 发送 SEND_NEXT_ALARM_CLOCK_CHANGED 给 AlarmHandler！
        mHandler.sendEmptyMessage(AlarmHandler.SEND_NEXT_ALARM_CLOCK_CHANGED);
    }
```
关于 AlarmHandler 我们后面会提到！

### 3.4.5 AlarmMS.updateNextAlarmClockLocked

该方法用于更新下一个 AlarmClock 的时间，当我们触发或者移除了一个 setAlarmClock 设置的 alarm 后，需要更新下一个 AlarmClock 的触发！

```java
    private void updateNextAlarmClockLocked() {
        //【1】如果 mNextAlarmClockMayChange 为 false，表示并没有移除或者触发一个 AlarmClock 闹钟，那就返回！
        if (!mNextAlarmClockMayChange) {
            return;
        }
        //【2】设置 mNextAlarmClockMayChange 为 false；
        mNextAlarmClockMayChange = false;

        //【3】用于缓存更新后的所有的 AlarmClock，用于后续更新 mNextAlarmClockForUser！
        SparseArray<AlarmManager.AlarmClockInfo> nextForUser = mTmpSparseAlarmClockArray;
        nextForUser.clear();

        //【4】遍历所有的 Alarm Batch，找到所有的 AlarmClock 闹钟！
        final int N = mAlarmBatches.size();
        for (int i = 0; i < N; i++) {
            ArrayList<Alarm> alarms = mAlarmBatches.get(i).alarms;
            final int M = alarms.size();
            for (int j = 0; j < M; j++) {
                Alarm a = alarms.get(j);

                //【4.1】a.alarmClock 不为 null，说明这是一个 AlarmClock 类型的闹钟！ 
                if (a.alarmClock != null) {
                
                    // 获得 AlarmClock 所属的 userId！
                    final int userId = UserHandle.getUserId(a.uid);
                    // 获得该 userId 下的当前 AlarmClock！
                    AlarmManager.AlarmClockInfo current = mNextAlarmClockForUser.get(userId);

                    if (DEBUG_ALARM_CLOCK) {
                        Log.v(TAG, "Found AlarmClockInfo " + a.alarmClock + " at " +
                                formatNextAlarm(getContext(), a.alarmClock, userId) +
                                " for user " + userId);
                    }

                    // Alarms 和 Batches 是通过时间排序的，所以这需要比较时间！
                    if (nextForUser.get(userId) == null) {
                        // 如果缓存中的 userId 是第一次添加，那将 a.alarmClock 添加到缓存中，不考虑 current！
                        nextForUser.put(userId, a.alarmClock); 

                    } else if (a.alarmClock.equals(current)
                            && current.getTriggerTime() <= nextForUser.get(userId).getTriggerTime()) {
                         // nextForUser.get(userId) 不等于 null，说明该 userId 在缓存中已经有 AlarmClock 了！
                        // 相同 userId，alarmClock 相同，但是 current 触发时间比缓存中的更早，那就用 current 替换缓存中！
                        nextForUser.put(userId, current);
                    }
                }
            }
        }

        //【5】通过上面的预处理，缓存 nextForUser 中存储的是每个 userId 下最早触发的 alarmClock！
        // 接着是用缓存 nextForUser 更新 mNextAlarmClockForUser！
        final int NN = nextForUser.size();
        for (int i = 0; i < NN; i++) {
            AlarmManager.AlarmClockInfo newAlarm = nextForUser.valueAt(i);
            int userId = nextForUser.keyAt(i);
            AlarmManager.AlarmClockInfo currentAlarm = mNextAlarmClockForUser.get(userId);

            //【4.1】相同 userId，但是 AlarmClockInfo 不相等，那就更新 mNextAlarmClockForUser！
            if (!newAlarm.equals(currentAlarm)) {
                updateNextAlarmInfoForUserLocked(userId, newAlarm);
            }
        }

        //【6】移除 mNextAlarmClockForUser 已经触发的 AlarmClock！
        final int NNN = mNextAlarmClockForUser.size();
        for (int i = NNN - 1; i >= 0; i--) {
            int userId = mNextAlarmClockForUser.keyAt(i);
            if (nextForUser.get(userId) == null) {
            
                //【3.4.5.1】调用 updateNextAlarmInfoForUserLocked
                updateNextAlarmInfoForUserLocked(userId, null);
            }
        }
    }

```
**变量说明**：

mNextAlarmClockForUser 用于保存每个 userId 下即将出发的下一个 AlarmClock！

**方法的流程总结**：

- 获得当前系统中最新的 AlarmClock 信息，保存到缓存 nextForUser 中；
    - 如果缓存 nextForUser 中 userId 下还没有 AlarmClock，那就将该 AlarmClock 添加到缓存中！
    - 如果缓存 nextForUser 中 userId 下已经有某个 AlarmClock，那就进一步比较：
        - 只有当该 userId 下的缓存 AlarmClock 和 mNextAlarmClockForUser 中对应的 AlarmClock 相等，且 mNextAlarmClockForUser 中对应的 AlarmClock 触发时间更早，才会替换缓存；否则，保留缓存！

</br>

- 用缓存 nextForUser 来更新 mNextAlarmClockForUser 列表；
   - 再次比较缓存 nextForUser 和 mNextAlarmClockForUser 中 userId 相同的 AlarmClock，如果二者不相同，用缓存更新 mNextAlarmClockForUser！

</br>

- 删除 mNextAlarmClockForUser 中某些 userId 下已经无效的 Alarm；


#### 3.4.5.1 AlarmMS.updateNextAlarmInfoForUserLocked

为指定 userId 更新 AlarmClock

```java
    private void updateNextAlarmInfoForUserLocked(int userId,
            AlarmManager.AlarmClockInfo alarmClock) {
        //【1】如果传入的 alarmClock 不为 null，那就替换 userId 下已有的 alarmClock！
        // 如果传入的 alarmClock 为 null，那就移除 userId 和对应的 alarmClock！
        if (alarmClock != null) {
            if (DEBUG_ALARM_CLOCK) {
                Log.v(TAG, "Next AlarmClockInfoForUser(" + userId + "): " +
                        formatNextAlarm(getContext(), alarmClock, userId));
            }
            mNextAlarmClockForUser.put(userId, alarmClock);
        } else {
            if (DEBUG_ALARM_CLOCK) {
                Log.v(TAG, "Next AlarmClockInfoForUser(" + userId + "): None");
            }
            mNextAlarmClockForUser.remove(userId);
        }
        
        //【2】当某个 userId 下的 AlarmClock 发生了变化，我们会将其记录到 mPendingSendNextAlarmClockChangedForUser 列表中！
        mPendingSendNextAlarmClockChangedForUser.put(userId, true);

        //【3】发送 SEND_NEXT_ALARM_CLOCK_CHANGED 给 AlarmHandler，AlarmHandler 会处理该消息！
        mHandler.removeMessages(AlarmHandler.SEND_NEXT_ALARM_CLOCK_CHANGED);
        mHandler.sendEmptyMessage(AlarmHandler.SEND_NEXT_ALARM_CLOCK_CHANGED);
    }
```

mPendingSendNextAlarmClockChangedForUser 列表用来记录某个 userId 下的 AlarmClock 发生了变化，如果发生了变化，他会用 userId -> true 的映射关系保存！

最后会发送 AlarmHandler.SEND_NEXT_ALARM_CLOCK_CHANGED 给 AlarmHandler 来处理，我们去看看：


##### 3.4.5.1.1 AlarmMS.AlarmHandler[SEND_NEXT_ALARM_CLOCK_CHANGED]

```java
                case SEND_NEXT_ALARM_CLOCK_CHANGED:
                    sendNextAlarmClockChanged();
                    break;
```
AlarmHandler 会调用 sendNextAlarmClockChanged 来处理消息：

##### 3.4.5.1.2 AlarmMS.sendNextAlarmClockChanged
```java
    private void sendNextAlarmClockChanged() {
        //【1】创建了一个 SparseArray 数组，用于保存要处理的 userId 和其 AlarmClock！
        SparseArray<AlarmManager.AlarmClockInfo> pendingUsers = mHandlerSparseAlarmClockArray;
        pendingUsers.clear();

        //【2】将 mPendingSendNextAlarmClockChangedForUser 的 user Id 和对应的 AlarmClock 保存到 pendingUsers 中！
        synchronized (mLock) {
            final int N  = mPendingSendNextAlarmClockChangedForUser.size();
            for (int i = 0; i < N; i++) {
                int userId = mPendingSendNextAlarmClockChangedForUser.keyAt(i);
                pendingUsers.append(userId, mNextAlarmClockForUser.get(userId));
            }
            // 清空 mPendingSendNextAlarmClockChangedForUser！
            mPendingSendNextAlarmClockChangedForUser.clear();
        }

        //【3】保存 AlarmClock 信息，并发送广播！
        final int N = pendingUsers.size();
        for (int i = 0; i < N; i++) {
            int userId = pendingUsers.keyAt(i);
            AlarmManager.AlarmClockInfo alarmClock = pendingUsers.valueAt(i);

            // 将这个 AlarmClock 的信息保存到 Settings 数据库中！
            Settings.System.putStringForUser(getContext().getContentResolver(),
                    Settings.System.NEXT_ALARM_FORMATTED,
                    formatNextAlarm(getContext(), alarmClock, userId),
                    userId);
            
            // 发送 NEXT_ALARM_CLOCK_CHANGED_INTENT 给指定的 userId！
            getContext().sendBroadcastAsUser(NEXT_ALARM_CLOCK_CHANGED_INTENT,
                    new UserHandle(userId));
        }
    }
```
mHandlerSparseAlarmClockArray 是一个预先创建的 SparseArray，没有很特殊的用途！

在 sendNextAlarmClockChanged 方法中，我们会处理前面的 mPendingSendNextAlarmClockChangedForUser！

## 3.5 阶段总结

到这里，我们 setAlarm 的整个流程就分析完成了，下面，我们来总结一下整个过程！

... ... ...