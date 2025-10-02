# AlarmManager第 3 篇 - trigger Alarm 流程分析
title: AlarmManager第 3 篇 - trigger Alarm 流程分析
date: 2017/08/15 20:46:25 
categories: 
- AndroidFramework源码分析
- AlarmManager闹钟管理
tags: AlarmManager闹钟管理
---

[toc]

基于 Android7.1.1 源码，分析 AlarmManagerService 的架构和逻辑，本篇文章来分析下 trigger Alarm 的流程！

# 0 回顾

在上一篇 set Alarm 文章中，我们知道, set 方法会调用 AlarmMS.rescheduleKernelAlarmsLocked 方法来设置下一个 alarm！

该方法用于设置下一个 Alarm！

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
下一个要执行的 wake up alarm 和 no wake up alarm 最终会通过 setLocked 方法进行设置！

```java
    private void setLocked(int type, long when) {
        if (mNativeData != 0) {
            // The kernel never triggers alarms with negative wakeup times
            // so we ensure they are positive.
            long alarmSeconds, alarmNanoseconds;
            if (when < 0) {
                alarmSeconds = 0;
                alarmNanoseconds = 0;
            } else {
                alarmSeconds = when / 1000;
                alarmNanoseconds = (when % 1000) * 1000 * 1000;
            }
            
            set(mNativeData, type, alarmSeconds, alarmNanoseconds);
        } else {
            Message msg = Message.obtain();
            msg.what = ALARM_EVENT;
            
            mHandler.removeMessages(ALARM_EVENT);
            mHandler.sendMessageAtTime(msg, when);
        }
    }
```
setLocked 中提供了 2 种方法来设置 Alarm：

 - 一种是通过 Alarm 驱动来设置 Alarm，同时启动一个 AlarmThread 来监听 Alarm 驱动的消息，处理触发的 Alarm！
 - 一种是通过自身的消息循环来设置和触发 Alarm；

正常情况下，Alarm 驱动是存在的，那么，我们先来看看正常情况！

# 1 AlarmThread.run

AlarmThread 内部有一个 while 循环，条件为 true，通过不断循环，来处理来自 Kernel 的消息，下面我们来看卡 AlarmThread 的 run 方法！

```java
        public void run() {
            ArrayList<Alarm> triggerList = new ArrayList<Alarm>();
            while (true) {   
                //【*1.1】waitForAlarm 最终会调用 native 层的方法，是同步阻塞的，如果该方法没有返回值
                // 那 AlarmThread 将一直阻塞在这里，当其返回，说明有 alarm 触发了！
                int result = waitForAlarm(mNativeData);
                mLastWakeup = SystemClock.elapsedRealtime(); // 更新上一次唤醒的时间
                
                //【1】清空 triggerList，triggerList 用于保存已经触发的 Alarm！
                triggerList.clear();

                final long nowRTC = System.currentTimeMillis();
                final long nowELAPSED = SystemClock.elapsedRealtime();

                //【2】如果只是时间发生改变，而不是闹钟触发，进入这里！
                if ((result & TIME_CHANGED_MASK) != 0) {
                    //【2.1】由于内核内部的一些微小调整，其会返回给系统一些实际上并不是闹钟出发引起的时间变化通知
                    // 这里会过滤掉这些时间变化！
                    final long lastTimeChangeClockTime;
                    final long expectedClockTime;
                    synchronized (mLock) {
                        lastTimeChangeClockTime = mLastTimeChangeClockTime;
                        expectedClockTime = lastTimeChangeClockTime
                                + (nowELAPSED - mLastTimeChangeRealtime);
                    }
                    if (lastTimeChangeClockTime == 0 || nowRTC < (expectedClockTime-500)
                            || nowRTC > (expectedClockTime+500)) {
                        // The change is by at least +/- 500 ms (or this is the first change),
                        // let's do it!
                        // 
                        if (DEBUG_BATCH) {
                            Slog.v(TAG, "Time changed notification from kernel; rebatching");
                        }
                        
                        // 从管理列表 mAlarmBatches 移除时间改变和日期改变的 Alarm 对象！
                        // 该方法也会触发 rebatch 过程！
                        removeImpl(mTimeTickSender);
                        removeImpl(mDateChangeSender);
                        
                        // 重新对系统中所有的 Alarm 进行排序和批处理！
                        rebatchAllAlarms();
                        
                        // 设置下一个发送 Intent.ACTION_TIME_TICK 和 Intent.ACTION_DATE_CHANGED 的 Alarm！
                        // 该方法会调用 setImpl 方法！
                        mClockReceiver.scheduleTimeTickEvent();
                        mClockReceiver.scheduleDateChangedEvent();

                        synchronized (mLock) {
                            // 记录时间改变的次数，同时记录上一次发生时间改变的时间点！
                            mNumTimeChanged++;
                            mLastTimeChangeClockTime = nowRTC;
                            mLastTimeChangeRealtime = nowELAPSED;
                        }
                        
                        // 发送时间改变的广播给所有的 userId!
                        Intent intent = new Intent(Intent.ACTION_TIME_CHANGED);
                        intent.addFlags(Intent.FLAG_RECEIVER_REPLACE_PENDING
                                | Intent.FLAG_RECEIVER_REGISTERED_ONLY_BEFORE_BOOT);
                        getContext().sendBroadcastAsUser(intent, UserHandle.ALL);

                        // The world has changed on us, so we need to re-evaluate alarms
                        // regardless of whether the kernel has told us one went off.
                        result |= IS_WAKEUP_MASK;
                    }
                }

                //【4】闹钟触发，会进入这个流程！
                if (result != TIME_CHANGED_MASK) {
                    // If this was anything besides just a time change, then figure what if
                    // anything to do about alarms.
                    synchronized (mLock) {
                        if (localLOGV) Slog.v(
                            TAG, "Checking for alarms... rtc=" + nowRTC
                            + ", elapsed=" + nowELAPSED);

                        if (WAKEUP_STATS) {
                            if ((result & IS_WAKEUP_MASK) != 0) {
                                long newEarliest = nowRTC - RECENT_WAKEUP_PERIOD;
                                int n = 0;
                                for (WakeupEvent event : mRecentWakeups) {
                                    if (event.when > newEarliest) break;
                                    n++; // number of now-stale entries at the list head
                                }
                                for (int i = 0; i < n; i++) {
                                    mRecentWakeups.remove();
                                }

                                recordWakeupAlarms(mAlarmBatches, nowELAPSED, nowRTC);
                            }
                        }
                        //【4.1】收集那些触发时间已经到了的 Alarm，该函数返回值表示是否有 wake up 类型的 alarm！！
                        // triggerAlarmsLocked 方法会将触发列表中的 alarm 排序！
                        boolean hasWakeup = triggerAlarmsLocked(triggerList, nowELAPSED, nowRTC);

                        //【4.2】如果 hasWakeup 为 false，说明无 wake up 类型的 alarm ，
                        // 如果 checkAllowNonWakeupDelayLocked 返回 true，说明该列表中的 alarm 要延迟发送！
                        if (!hasWakeup && checkAllowNonWakeupDelayLocked(nowELAPSED)) {

                            // 如果没有 no wake up alarms 要分发，如果此时处于 screen off 或者其他一些特殊的情况！
                            // 我们会将 triggerList 添加到延迟列表 mPendingNonWakeupAlarms 中延迟分发！
                            if (mPendingNonWakeupAlarms.size() == 0) {

                                // 如果 mPendingNonWakeupAlarms 为 empty，初始化延迟分发时间！
                                mStartCurrentDelayTime = nowELAPSED; // 设置 delay 的时间点！
                                mNextNonWakeupDeliveryTime = nowELAPSED
                                        + ((currentNonWakeupFuzzLocked(nowELAPSED)*3)/2);
                            }

                            mPendingNonWakeupAlarms.addAll(triggerList); // 添加到延迟集合中
                            mNumDelayedAlarms += triggerList.size(); // 统计延迟分发的 Alarm 个数
                            
                            //【4.2.1】rebatch 所有的 Alarm，更新下一个 AlarmClock！
                            rescheduleKernelAlarmsLocked();
                            updateNextAlarmClockLocked();

                        } else {
                            //【4.3】进入这里的，就开始要分发 Alarm 了！
                            // now deliver the alarm intents; if there are pending non-wakeup
                            // alarms, we need to merge them in to the list.  note we don't
                            // just deliver them first because we generally want non-wakeup
                            // alarms delivered after wakeup alarms.
                            //【4.3.1】rebatch 所有的 Alarm，更新下一个 AlarmClock！
                            rescheduleKernelAlarmsLocked();
                            updateNextAlarmClockLocked();
                            
                            // 如果 mPendingNonWakeupAlarms 不为 empty，那么，我们也要分发这些 Alarm！
                            if (mPendingNonWakeupAlarms.size() > 0) {

                                //【1.2.1】这里对 mPendingNonWakeupAlarms 列表中的 Alarm 计算优先级！
                                // 然后将 alarms 添加到 triggerList 中，然后在进行排序！
                                calculateDeliveryPriorities(mPendingNonWakeupAlarms);
                                triggerList.addAll(mPendingNonWakeupAlarms);
                                Collections.sort(triggerList, mAlarmDispatchComparator);

                                final long thisDelayTime = nowELAPSED - mStartCurrentDelayTime;
                                mTotalDelayTime += thisDelayTime; // 累计延迟的时间段；
                                if (mMaxDelayTime < thisDelayTime) {
                                    mMaxDelayTime = thisDelayTime; // 最大延迟的时间段；
                                }
                                mPendingNonWakeupAlarms.clear(); // 清空 mPendingNonWakeupAlarms！
                            }
                            
                            // 分发本次要触发的 Alarm！
                            deliverAlarmsLocked(triggerList, nowELAPSED);
                        }
                    }

                } else {
                    // Just in case -- even though no wakeup flag was set, make sure
                    // we have updated the kernel to the next alarm time.
                    synchronized (mLock) {
                        rescheduleKernelAlarmsLocked();
                    }
                }
            }
        }
```
waitForAlarm 会调用 static jint android_server_AlarmManagerService_waitForAlarm 这个 native 方法，对 /dev/alarm 驱动文件进行操作！

## 1.1 AlarmMS.waitForAlarm

waitForAlarm 方法用来等待 Alarm 驱动的返回，其实阻塞性的！

```java
    private native int waitForAlarm(long nativeData);
```

该方法最后会调用 navtive 的方法，位于 frameworks/base/services/core/jni/com_android_server_AlarmManagerService.cpp 文件中：

```java

```


## 1.2 AlarmMS.triggerAlarmsLocked

triggerAlarmsLocked 方法用于分发那些触发时间已到的 Alarm！

```java
    boolean triggerAlarmsLocked(ArrayList<Alarm> triggerList, final long nowELAPSED,
            final long nowRTC) {
        boolean hasWakeup = false;

        //【1】mAlarmBatches 中的 Batch 都是按照触发时间点升序排列，
        // 接下来会遍历 mAlarmBatches，分发那些触发时间已到的 Alarm！
        while (mAlarmBatches.size() > 0) {
            // 获得触发事件最早的 Batch！
            Batch batch = mAlarmBatches.get(0);
            // 如果时间最早的 Batch 的触发时间还没到，那就不作任何操作，退出循环！
            if (batch.start > nowELAPSED) {
                // Everything else is scheduled for the future
                break;
            }

            //【1.1】能够走到这里，说明 batch 的触发时间已经到了，那么我们要移除这个 Batch，防止重复分发！
            mAlarmBatches.remove(0);

            //【1.2】分发这个 Batch 中的所有的 Alarm！
            final int N = batch.size();
            for (int i = 0; i < N; i++) {
                Alarm alarm = batch.get(i);
                
                //【1.2.1】如果 Alarm 设置了 AlarmManager.FLAG_ALLOW_WHILE_IDLE 标志位，
                // 说明其在系统处于 idle 状态也可以触发！
                if ((alarm.flags&AlarmManager.FLAG_ALLOW_WHILE_IDLE) != 0) {

                    // 对于 allow while idle 类型的 Alarm，我们要检验其触发时间，看其距离上次触发事件是否
                    // 不小于规定的最小时间间隔：mAllowWhileIdleMinTime
                    long lastTime = mLastAllowWhileIdleDispatch.get(alarm.uid, 0);
                    long minTime = lastTime + mAllowWhileIdleMinTime;
                    
                    // 如果 nowELAPSED < minTime，说明虽然该 alarm 的触发时间到了，但是时间小于规定的最小时间
                    // 那么，按照最小时间重新设置 alarm！
                    if (nowELAPSED < minTime) {

                        // 设置 alarm 的触发时间为 minTime，最大触发时间至少为 minTime！
                        alarm.whenElapsed = minTime;
                        if (alarm.maxWhenElapsed < minTime) {
                            alarm.maxWhenElapsed = minTime;
                        }

                        if (RECORD_DEVICE_IDLE_ALARMS) {
                            IdleDispatchEntry ent = new IdleDispatchEntry();
                            ent.uid = alarm.uid;
                            ent.pkg = alarm.operation.getCreatorPackage();
                            ent.tag = alarm.operation.getTag("");
                            ent.op = "RESCHEDULE";
                            ent.elapsedRealtime = nowELAPSED;
                            ent.argRealtime = lastTime;
                            mAllowWhileIdleDispatches.add(ent);
                        }

                        // 按照最小时间，重新设置 alarm，然后继续处理下一个 alarm！！
                        setImplLocked(alarm, true, false);
                        continue;
                    }
                }
                
                //【1.2.2】进入这里，说明 Alarm 此时需要被分发！
                alarm.count = 1; // Alarm 闹钟的触发次数初始化为 1；
                
                triggerList.add(alarm); // 将这个 alarm 添加到 triggerList 中去！
                
                // 如果 alarm 是 wake from idle 类型的，在 event log 文件中打印出来，方便调试！
                if ((alarm.flags&AlarmManager.FLAG_WAKE_FROM_IDLE) != 0) {
                    EventLogTags.writeDeviceIdleWakeFromIdle(mPendingIdleUntil != null ? 1 : 0,
                            alarm.statsTag);
                }

                //【1.2.3】如果此时要触发的 Alarm 是 mPendingIdleUntil，那么说明系统要从 idle 状态恢复了，那么
                // 设置 mPendingIdleUntil 为 null，同时 rebatch 其他的 alarm，恢复因为 idle 状态而处于等待状态的 alarm
                if (mPendingIdleUntil == alarm) {
                    mPendingIdleUntil = null;
                    rebatchAllAlarmsLocked(false);
                    restorePendingWhileIdleAlarmsLocked();
                }

                //【1.2.3】如果此时要触发的 Alarm 是 mNextWakeFromIdle，说明该 alarm 会将系统从 idle 状态唤醒！
                // 那么设置 mNextWakeFromIdle 为 null，同时 rebatch 其他的 alarm！
                if (mNextWakeFromIdle == alarm) {
                    mNextWakeFromIdle = null;
                    rebatchAllAlarmsLocked(false);
                }

                //【1.2.3】如果 alarm.repeatInterval 大于 0，说明该 alarm 是一个 repeating alarm！
                // 我们计算下一次触发的时间，再次设置 alarm！
                // 这里要注意一点，因为系统可能进入休眠或者关机，alarm 可能错过了多个周期时间的触发！
                if (alarm.repeatInterval > 0) {

                    // 这里是计算，我们可能因为系统休眠或者关机，错过了几个周期，当然该结果是可以为 0 的；
                    // 我们需要把错过的触发次数也统计进去！！
                    alarm.count += (nowELAPSED - alarm.whenElapsed) / alarm.repeatInterval;

                    // 计算下次的触发时间！
                    final long delta = alarm.count * alarm.repeatInterval;
                    final long nextElapsed = alarm.whenElapsed + delta;

                    // 设置 alarm！
                    setImplLocked(alarm.type, alarm.when + delta, nextElapsed, alarm.windowLength,
                            maxTriggerTime(nowELAPSED, nextElapsed, alarm.repeatInterval),
                            alarm.repeatInterval, alarm.operation, null, null, alarm.flags, true,
                            alarm.workSource, alarm.alarmClock, alarm.uid, alarm.packageName);
                }

                // 如果该 alarm 是 wake up 类型的，hasWakeup 为 true！
                if (alarm.wakeup) {
                    hasWakeup = true;
                }

                // 该 alarm 是通过 setAlarmClock 方法设置的，当该 alarm 触发后，我们会计算下一个 AlarmClock
                if (alarm.alarmClock != null) {
                    mNextAlarmClockMayChange = true;
                }
            }
        }

        // 用于记录分发次数。每分发一个新的 alarm 集，该值会加 1；
        mCurrentSeq++;
        //【2】计算分发优先级！
        calculateDeliveryPriorities(triggerList);
        //【3】对要分发的 alarm 进行排序，先依据优先级，优先级相同，按照时间升序排序！
        Collections.sort(triggerList, mAlarmDispatchComparator);

        if (localLOGV) {
            for (int i=0; i<triggerList.size(); i++) {
                Slog.v(TAG, "Triggering alarm #" + i + ": " + triggerList.get(i));
            }
        }

        return hasWakeup;
    }

```

### 1.2.1 AlarmMS.calculateDeliveryPriorities

calculateDeliveryPriorities 方法用于计算分发的优先级！参数 alarms 表示所有需要触发的 alarm

```java
    void calculateDeliveryPriorities(ArrayList<Alarm> alarms) {
        final int N = alarms.size();
        for (int i = 0; i < N; i++) {
            Alarm a = alarms.get(i);
            
            //【1】计算 alarm 优先级！
            final int alarmPrio;
            if (a.operation != null
                    && Intent.ACTION_TIME_TICK.equals(a.operation.getIntent().getAction())) {
                //【1.1】如果该 alarm 是 Intent.ACTION_TIME_TICK，那么优先级为 PRIO_TICK
                alarmPrio = PRIO_TICK;
            } else if (a.wakeup) {
                //【1.2】如果该 alarm 是 wake up 类型的，那么优先级为 PRIO_WAKEUP
                alarmPrio = PRIO_WAKEUP;
            } else {
                //【1.3】其他类型的 alarm 优先级为 PRIO_NORMAL
                alarmPrio = PRIO_NORMAL;
            }

            //【2】获得 alarm 的优先级对象，和其发送者 packageName
            PriorityClass packagePrio = a.priorityClass;
            String alarmPackage = (a.operation != null)
                    ? a.operation.getCreatorPackage()
                    : a.packageName;

            // 如果 packagePrio 为 null，那就优先在缓存 mPriorities 中找，找不到；创建新的，并加入到缓存中！
            if (packagePrio == null) packagePrio = mPriorities.get(alarmPackage);
            if (packagePrio == null) {
                packagePrio = a.priorityClass = new PriorityClass(); // lowest prio & stale sequence
                mPriorities.put(alarmPackage, packagePrio);
            }

            // 更新 alarm 的 priorityClass 优先级对象!
            a.priorityClass = packagePrio;

            //【3】更新 alarm 优先级！
            if (packagePrio.seq != mCurrentSeq) {
                //【3.1】如果 packagePrio.seq 不等于 mCurrentSeq，说明在本次的发送列表中，这是来自这个包的第一个 Alarm
                // 那么执行初始化操作！
                packagePrio.priority = alarmPrio;
                packagePrio.seq = mCurrentSeq;

            } else {
                //【3.2】如果 packagePrio.seq == mCurrentSeq，说明在发送列表中，有来自同一个 package 的多个 Alarm。
                // 那就调整优先级，规则： TICK < WAKEUP < NORMAL
                if (alarmPrio < packagePrio.priority) {
                    packagePrio.priority = alarmPrio;
                }
            }
        }
    }
```
这里设置到了如下的几种 alarm 优先级，值越小，优先级越高！

```java
    static final int PRIO_TICK = 0; // 时间改变!
    static final int PRIO_WAKEUP = 1; // 唤醒!
    static final int PRIO_NORMAL = 2; // 正常!
```
同时有一个集合 mPriorities，用于保存发送者的 package 和其对应 Alarm 优先级：

```java
    final class PriorityClass {
        int seq;
        int priority;

        PriorityClass() {
            seq = mCurrentSeq - 1;
            priority = PRIO_NORMAL;
        }
    }

    final HashMap<String, PriorityClass> mPriorities = new HashMap<>();
    int mCurrentSeq = 0;
```

PriorityClass 用于表示优先级别！

### 1.2.2 Collections.sort(triggerList, mAlarmDispatchComparator)

接下来，对 triggerList 进行排序，这里用到了一个比较器 mAlarmDispatchComparator！

```java
    final Comparator<Alarm> mAlarmDispatchComparator = new Comparator<Alarm>() {
        @Override
        public int compare(Alarm lhs, Alarm rhs) {
            // priority class trumps everything.  TICK < WAKEUP < NORMAL
            if (lhs.priorityClass.priority < rhs.priorityClass.priority) {
                return -1;
            } else if (lhs.priorityClass.priority > rhs.priorityClass.priority) {
                return 1;
            }

            // within each class, sort by nominal delivery time
            if (lhs.whenElapsed < rhs.whenElapsed) {
                return -1;
            } else if (lhs.whenElapsed > rhs.whenElapsed) {
                return 1;
            }

            // same priority class + same target delivery time
            return 0;
        }
    };
```

首先按照优先级从高到低进行排序，然后按照触发时间从小到大进行排序，如果优先级和时间相同，保持先后顺序不变！


## 1.3 AlarmMS.checkAllowNonWakeupDelayLocked

该方法用于检查是否需要延迟 no wake up 类型的 Alarm 的分发！

```java
    boolean checkAllowNonWakeupDelayLocked(long nowELAPSED) {
        //【1】如果此时处于亮屏状态，无需延迟！
        if (mInteractive) {
            return false;
        }
        //【2】如果此时处于熄屏状态，但是我们还没有分发过任何 Alarm，无需延迟！
        if (mLastAlarmDeliveryTime <= 0) {
            return false;
        }
        //【3】如果此时系统中有等待中的 no wake up 类型的 alarm，但是其分发时间 mNextNonWakeupDeliveryTime 已经过去了
        // 那就无需延迟（看注释是为了解决一个在 Looper 中卡住的 bugs）！
        if (mPendingNonWakeupAlarms.size() > 0 && mNextNonWakeupDeliveryTime < nowELAPSED) {
            return false;
        }
        // 计算当前时间距离上次分发的时间间隔，比较其和熄屏时间间隔的大小！！
        long timeSinceLast = nowELAPSED - mLastAlarmDeliveryTime;
        return timeSinceLast <= currentNonWakeupFuzzLocked(nowELAPSED);
    }
```
规则如下：

- 如果此时处于亮屏状态，无需延迟！
- 如果此时处于熄屏状态，但是我们还没有分发过任何 Alarm，无需延迟！
- 如果此时处于熄屏状态，我们之前也分发过 Alarm，那么如果 此时系统中有等待中的 no wake up 类型的 alarm，其分发时间已经过去了，无需延迟；
- 如果上面三个条件都不满足，那么如果距离上次分发 Alarm 的时间超过了 no wake up 类型 alarm 的最大延迟分发时间，那就无需延迟！


currentNonWakeupFuzzLocked 用来计算了 no wake up 类型 alarm 的最大延迟分发时间

### 1.3.1 AlarmMS.currentNonWakeupFuzzLocked

该方法是用来计算熄屏幕的时长，然后根据时常，来确定 no wake up 类型的 alarm 的延迟触发的最长时间！！

```java
    long currentNonWakeupFuzzLocked(long nowELAPSED) {
        long timeSinceOn = nowELAPSED - mNonInteractiveStartTime;
        //【1】如果熄屏时间小于 5 mins，那么延迟最多 2 mins
        if (timeSinceOn < 5*60*1000) {
            return 2*60*1000;
            
        //【2】如果熄屏时间小于 30 mins，那么延迟最多 15 mins
        } else if (timeSinceOn < 30*60*1000) {
            return 15*60*1000;
        } else {
        //【3】其他情况，那么延迟最多 60 mins
            return 60*60*1000;
        }
    }
```
不多说了！


## 1.4 AlarmMS.deliverAlarmsLocked

deliverAlarmsLocked 方法用于分发已经出发的 Alarm！

```java
    void deliverAlarmsLocked(ArrayList<Alarm> triggerList, long nowELAPSED) {
        //【1】设置上一次的发送时间点！
        mLastAlarmDeliveryTime = nowELAPSED;
        //【2】遍历 triggerList，分发 Alarm！
        for (int i=0; i<triggerList.size(); i++) {
            Alarm alarm = triggerList.get(i);
            //【2.1】判断该 Alarm 是否是 allow while idle 的！
            final boolean allowWhileIdle = (alarm.flags&AlarmManager.FLAG_ALLOW_WHILE_IDLE) != 0;
            try {
                if (localLOGV) {
                    Slog.v(TAG, "sending alarm " + alarm);
                }
                if (RECORD_ALARMS_IN_HISTORY) { // 用于系统记录信息！
                    if (alarm.workSource != null && alarm.workSource.size() > 0) {
                        for (int wi=0; wi<alarm.workSource.size(); wi++) {
                            ActivityManagerNative.noteAlarmStart(
                                    alarm.operation, alarm.workSource.get(wi), alarm.statsTag);
                        }
                    } else {
                        ActivityManagerNative.noteAlarmStart(
                                alarm.operation, alarm.uid, alarm.statsTag);
                    }
                }
                //【1.5】调用 DeliveryTracker 的 deliverLocked 方法，分发 Alarm！
                mDeliveryTracker.deliverLocked(alarm, nowELAPSED, allowWhileIdle);
            } catch (RuntimeException e) {
                Slog.w(TAG, "Failure sending alarm.", e);
            }
        }
    }
```
接下来，我们去看看 DeliveryTracker：

## 1.5 DeliveryTracker.deliverLocked

DeliveryTracker 用以监控和跟踪 Alarm 的分发！

```java
    class DeliveryTracker extends IAlarmCompleteListener.Stub implements PendingIntent.OnFinished {
        ... ... ... ...

        /**
         * Deliver an alarm and set up the post-delivery handling appropriately
         */
        public void deliverLocked(Alarm alarm, long nowELAPSED, boolean allowWhileIdle) {
            //【1】针对于 Alarm 的触发方式，做不同的处理！
            if (alarm.operation != null) {
                //【1.1】如果是通过 PendingIntent 触发，进入这里，这里会调用 PendingIntent 的 send 方法，将 Intent
                // 发送给接收方！
                try {
                    alarm.operation.send(getContext(), 0,
                            mBackgroundIntent.putExtra(
                                    Intent.EXTRA_ALARM_COUNT, alarm.count),
                                    mDeliveryTracker, mHandler, null,
                                    allowWhileIdle ? mIdleOptions : null);

                } catch (PendingIntent.CanceledException e) {
                    // 如果 send 出现异常，那么如果是重复闹钟，那就将这个 Alarm 移除；否则，直接 return 掉！
                    if (alarm.repeatInterval > 0) {
                        removeImpl(alarm.operation); //  该方法会 rebatch 其他的 A
                    }
                    return;
                }

            } else {
                //【1.2】如果是通过 onAlarmListener 接收，进入这里，调用其 doAlarm 方法！
                try {
                    if (DEBUG_LISTENER_CALLBACK) {
                        Slog.v(TAG, "Alarm to uid=" + alarm.uid
                                + " listener=" + alarm.listener.asBinder());
                    }
                    // 跨进程调用应用进程中的 onAlarmListener 的 doAlarm 方法！
                    alarm.listener.doAlarm(this);

                    // 设置监听超时处理！
                    mHandler.sendMessageDelayed(
                            mHandler.obtainMessage(AlarmHandler.LISTENER_TIMEOUT,
                                    alarm.listener.asBinder()),
                            mConstants.LISTENER_TIMEOUT);
                } catch (Exception e) {
                    // 如果分发出现异常，那就 return，不做处理！！
                    if (DEBUG_LISTENER_CALLBACK) {
                        Slog.i(TAG, "Alarm undeliverable to listener "
                                + alarm.listener.asBinder(), e);
                    }
                    return;
                }
            }

            //【2】alarm 要开始准备分发了，开始统计跟踪 wakelock 和 status！
            if (mBroadcastRefCount == 0) {
                setWakelockWorkSource(alarm.operation, alarm.workSource,
                        alarm.type, alarm.statsTag, (alarm.operation == null) ? alarm.uid : -1,
                        true);
                mWakeLock.acquire(); // 申请 wakeLock 锁，防止分发过程睡下去！
                // 发送 REPORT_ALARMS_ACTIVE 给 AlarmHandler，AlarmHandler 会通知 DeviceIdleController
                //，有 alarm 处于 active 状态！
                mHandler.obtainMessage(AlarmHandler.REPORT_ALARMS_ACTIVE, 1).sendToTarget();
            }

            //【3】创建 InFlight 对象，表示该 Alarm 正在分发过程中！
            final InFlight inflight = new InFlight(AlarmManagerService.this,
                    alarm.operation, alarm.listener, alarm.workSource, alarm.uid,
                    alarm.packageName, alarm.type, alarm.statsTag, nowELAPSED);
            mInFlight.add(inflight);

            mBroadcastRefCount++; // 引用计数加 1；

            //【4】如果这个 alarm 是 allow while idle 的，那就要记录下该 alarm 的上一次分发时间
            // 保存到 mLastAllowWhileIdleDispatch 中！
            if (allowWhileIdle) {
                mLastAllowWhileIdleDispatch.put(alarm.uid, nowELAPSED);
                
                if (RECORD_DEVICE_IDLE_ALARMS) { // 用于 dump 操作，不关注！
                    IdleDispatchEntry ent = new IdleDispatchEntry();
                    ent.uid = alarm.uid;
                    ent.pkg = alarm.packageName;
                    ent.tag = alarm.statsTag;
                    ent.op = "DELIVER";
                    ent.elapsedRealtime = nowELAPSED;
                    mAllowWhileIdleDispatches.add(ent);
                }
            }

            //【5】设置 alarm 状态！
            final BroadcastStats bs = inflight.mBroadcastStats;
            bs.count++;
            if (bs.nesting == 0) {
                bs.nesting = 1;
                bs.startTime = nowELAPSED;
            } else {
                bs.nesting++;
            }
            final FilterStats fs = inflight.mFilterStats;
            fs.count++;
            if (fs.nesting == 0) {
                fs.nesting = 1;
                fs.startTime = nowELAPSED;
            } else {
                fs.nesting++;
            }
            
            //【6】如果 alarm 是 wake up 类型的！
            if (alarm.type == ELAPSED_REALTIME_WAKEUP
                    || alarm.type == RTC_WAKEUP) {
                bs.numWakeup++; // 更新 wake up 的次数！
                fs.numWakeup++;
                if (alarm.workSource != null && alarm.workSource.size() > 0) {
                    for (int wi=0; wi<alarm.workSource.size(); wi++) {
                        final String wsName = alarm.workSource.getName(wi);
                        ActivityManagerNative.noteWakeupAlarm(
                                alarm.operation, alarm.workSource.get(wi),
                                (wsName != null) ? wsName : alarm.packageName,
                                alarm.statsTag);
                    }
                } else {
                    ActivityManagerNative.noteWakeupAlarm(
                            alarm.operation, alarm.uid, alarm.packageName, alarm.statsTag);
                }
            }
        }
    }    
```
mDeliveryTracker 是 AlarmManagerService 的内部变量，用于监控分发的过程！

这里我们看到，DeliveryTracker 继承了 IAlarmCompleteListener.Stub，这个是由 IAlarmCompleteListener.aidl 文件生成的一个服务端“桩”对象，作用很明显了，用于跨进程通信，即：系统进程和应用进程！

原因很简单，DeliveryTracker 用于监控 Alarm 的分发，所以系统需要知道客户端对 Alarm 的分发和处理结果，所以应用进程那边一定会有 DeliveryTracker 的代理对象的！


同时 DeliveryTracker 还实现了 PendingIntent.OnFinished 接口，用于另外一种分发方式！

我们继续看！


### 1.5.1 new InFlight

系统会对每一个 Alarm 都创建其 InFlight 对象，并保存到 mInFlight！

```java
    static final class InFlight {
        final PendingIntent mPendingIntent;
        final IBinder mListener;
        final WorkSource mWorkSource;
        final int mUid;
        final String mTag;
        final BroadcastStats mBroadcastStats;
        final FilterStats mFilterStats;
        final int mAlarmType;

        InFlight(AlarmManagerService service, PendingIntent pendingIntent, IAlarmListener listener,
                WorkSource workSource, int uid, String alarmPkg, int alarmType, String tag,
                long nowELAPSED) {
            mPendingIntent = pendingIntent; // PendingIntent 对象！
            mListener = listener != null ? listener.asBinder() : null; // AlarmListener 对象！
            mWorkSource = workSource;
            mUid = uid;
            mTag = tag;
            mBroadcastStats = (pendingIntent != null)
                    ? service.getStatsLocked(pendingIntent)
                    : service.getStatsLocked(uid, alarmPkg);
            FilterStats fs = mBroadcastStats.filterStats.get(mTag);
            if (fs == null) {
                fs = new FilterStats(mBroadcastStats, mTag);
                mBroadcastStats.filterStats.put(mTag, fs);
            }
            fs.lastTime = nowELAPSED;
            mFilterStats = fs;
            mAlarmType = alarmType; // Alarm 的类型！
        }
    }

```

#### 1.5.1.1 AlarmMS.getStatsLocked

getStatsLocked 用于获得 Alarm 对应的 BroadcastStats 对象！

```java
    private final BroadcastStats getStatsLocked(PendingIntent pi) {
        String pkg = pi.getCreatorPackage();
        int uid = pi.getCreatorUid();
        return getStatsLocked(uid, pkg);
    }

    private final BroadcastStats getStatsLocked(int uid, String pkgName) {
        ArrayMap<String, BroadcastStats> uidStats = mBroadcastStats.get(uid);
        if (uidStats == null) {
            uidStats = new ArrayMap<String, BroadcastStats>();
            mBroadcastStats.put(uid, uidStats);
        }
        BroadcastStats bs = uidStats.get(pkgName);
        if (bs == null) {
            bs = new BroadcastStats(uid, pkgName);
            uidStats.put(pkgName, bs);
        }
        return bs;
    }
```
每一个 Alarm 都会有一个 BroadcastStats，用于保存该 Alarm 的分发状态！

同时，AlarmManagerService 也有一个集合：mBroadcastStats，来管理每个 uid 对应的广播分发状态！

```java
    final SparseArray<ArrayMap<String, BroadcastStats>> mBroadcastStats
            = new SparseArray<ArrayMap<String, BroadcastStats>>();
```

映射关系：**create uid** -> ArrayMap：**create packageName** -> **BroadcastStats**！


# 2 分发的流程

前面我们看到，如果 setAlarm 是通过 PendingIntent 的方式来设置的，系统会通过如下方式分发 Alarm！

```java
                    alarm.operation.send(getContext(), 0,
                            mBackgroundIntent.putExtra(
                                    Intent.EXTRA_ALARM_COUNT, alarm.count),
                                    mDeliveryTracker, mHandler, null,
                                    allowWhileIdle ? mIdleOptions : null);
```
如果 setAlarm 是通过 AlarmListener 的方式来设置的，系统会通过如下方式分发 Alarm！

```java
                    // 跨进程调用应用进程中的 onAlarmListener 的 doAlarm 方法！
                    alarm.listener.doAlarm(this);
                    // 设置监听超时处理！
                    mHandler.sendMessageDelayed(
                            mHandler.obtainMessage(AlarmHandler.LISTENER_TIMEOUT,
                                    alarm.listener.asBinder()),
                            mConstants.LISTENER_TIMEOUT);
```
下面我们来看看二者的区别！

## 2.1 Alarm.PendingIntent.send

我们可以看到，对于 PendingIntent，我们会将 Alarm 的重复分发次数通过 Intent.EXTRA_ALARM_COUNT 发送给处理方！

```java
    public void send(Context context, int code, @Nullable Intent intent,
            @Nullable OnFinished onFinished, @Nullable Handler handler,
            @Nullable String requiredPermission, @Nullable Bundle options)
            throws CanceledException {
        try {
            String resolvedType = intent != null ?
                    intent.resolveTypeIfNeeded(context.getContentResolver())
                    : null;

            //【1】调用 AMS 的 sendIntentSender 方法发送这个 Alarm！！
            int res = ActivityManagerNative.getDefault().sendIntentSender(
                    mTarget, code, intent, resolvedType,
                    onFinished != null
                            ? new FinishedDispatcher(this, onFinished, handler) // 创建了 FinishedDispatcher 对象！
                            : null,
                    requiredPermission, options);

            if (res < 0) {
                throw new CanceledException();
            }
        } catch (RemoteException e) {
            throw new CanceledException(e);
        }
    }
```
这里要注意，我们传入的参数 onFinished 实现对象是 mDeliveryTracker！

这里我们要简单提一下 mTarget，他是 PendingIntent 的内部成员变量，是一个 IIntentSender Binder 对象！我们在调用 PendingIntent.getService 等方法的时候，会有如下的逻辑：


### 2.1.1 new FinishedDispatcher

```java
    private static class FinishedDispatcher extends IIntentReceiver.Stub
            implements Runnable {
        private final PendingIntent mPendingIntent;
        private final OnFinished mWho; // mDeliveryTracker 对象！
        private final Handler mHandler;
        private Intent mIntent;
        private int mResultCode;
        private String mResultData;
        private Bundle mResultExtras;
        private static Handler sDefaultSystemHandler;

        FinishedDispatcher(PendingIntent pi, OnFinished who, Handler handler) {
            mPendingIntent = pi;
            mWho = who;
            if (handler == null && ActivityThread.isSystem()) {
                if (sDefaultSystemHandler == null) {
                    sDefaultSystemHandler = new Handler(Looper.getMainLooper());
                }
                mHandler = sDefaultSystemHandler;
            } else {
                mHandler = handler;
            }
        }
    }
```
如果没有传入指定的 Handler，

### 2.1.2 ActivityMS.sendIntentSender

PendingIntent.send 方法最终会调用 AMS.sendIntentSender 方法，当然这里涉及到了 PendingIntent 架构和逻辑，这里不做过多的讨论！

```java
    @Override
    public int sendIntentSender(IIntentSender target, int code, Intent intent, String resolvedType,
            IIntentReceiver finishedReceiver, String requiredPermission, Bundle options) {
        //【1】一般情况，每一个 PendingIntent 都会有一个 PendingIntentRecord 在系统中与之对应！
        // 默认情况下会调用 PendingIntentRecord 的 sendWithResult 方法！
        if (target instanceof PendingIntentRecord) {
            return ((PendingIntentRecord)target).sendWithResult(code, intent, resolvedType,
                    finishedReceiver, requiredPermission, options);
        } else {
            // 下面是特殊情况，我们不关注！
            if (intent == null) {
                Slog.wtf(TAG, "Can't use null intent with direct IIntentSender call");
                intent = new Intent(Intent.ACTION_MAIN);
            }
            try {
                target.send(code, intent, resolvedType, null, requiredPermission, options);
            } catch (RemoteException e) {
            }
            if (finishedReceiver != null) {
                try {
                    // 处理分发结果！
                    finishedReceiver.performReceive(intent, 0,
                            null, null, false, false, UserHandle.getCallingUserId());
                } catch (RemoteException e) {
                }
            }
            return 0;
        }
    }
```
正常情况下，会调用 PendingIntentRecord.sendWithResult 分发 alarm！

### 2.1.3 PendingIntentRecord.sendWithResult

```java
    public int sendWithResult(int code, Intent intent, String resolvedType,
            IIntentReceiver finishedReceiver, String requiredPermission, Bundle options) {
        // 继续调用 sendInner 方法！
        return sendInner(code, intent, resolvedType, finishedReceiver,
                requiredPermission, null, null, 0, 0, 0, options, null);
    }
```


### 2.1.4 PendingIntentRecord.sendInner


```java
    int sendInner(int code, Intent intent, String resolvedType, IIntentReceiver finishedReceiver,
            String requiredPermission, IBinder resultTo, String resultWho, int requestCode,
            int flagsMask, int flagsValues, Bundle options, IActivityContainer container) {
        if (intent != null) intent.setDefusable(true);
        if (options != null) options.setDefusable(true);

        if (whitelistDuration > 0 && !canceled) {
            // Must call before acquiring the lock. It's possible the method return before sending
            // the intent due to some validations inside the lock, in which case the UID shouldn't
            // be whitelisted, but since the whitelist is temporary, that would be ok.
            owner.tempWhitelistAppForPowerSave(Binder.getCallingPid(), Binder.getCallingUid(), uid,
                    whitelistDuration);
        }

        synchronized (owner) {
            final ActivityContainer activityContainer = (ActivityContainer)container;
            if (activityContainer != null && activityContainer.mParentActivity != null &&
                    activityContainer.mParentActivity.state
                            != ActivityStack.ActivityState.RESUMED) {
                // Cannot start a child activity if the parent is not resumed.
                return ActivityManager.START_CANCELED;
            }
            if (!canceled) {
                sent = true;
                if ((key.flags&PendingIntent.FLAG_ONE_SHOT) != 0) {
                    owner.cancelIntentSenderLocked(this, true);
                    canceled = true;
                }

                Intent finalIntent = key.requestIntent != null
                        ? new Intent(key.requestIntent) : new Intent();

                final boolean immutable = (key.flags & PendingIntent.FLAG_IMMUTABLE) != 0;
                if (!immutable) {
                    if (intent != null) {
                        int changes = finalIntent.fillIn(intent, key.flags);
                        if ((changes & Intent.FILL_IN_DATA) == 0) {
                            resolvedType = key.requestResolvedType;
                        }
                    } else {
                        resolvedType = key.requestResolvedType;
                    }
                    flagsMask &= ~Intent.IMMUTABLE_FLAGS;
                    flagsValues &= flagsMask;
                    finalIntent.setFlags((finalIntent.getFlags() & ~flagsMask) | flagsValues);
                } else {
                    resolvedType = key.requestResolvedType;
                }

                final long origId = Binder.clearCallingIdentity();

                boolean sendFinish = finishedReceiver != null;
                int userId = key.userId;
                if (userId == UserHandle.USER_CURRENT) {
                    userId = owner.mUserController.getCurrentOrTargetUserIdLocked();
                }
                int res = 0;
                // 根据 alarm 的类型，做不同的处理！
                switch (key.type) {
                    case ActivityManager.INTENT_SENDER_ACTIVITY:
                        if (options == null) {
                            options = key.options;
                        } else if (key.options != null) {
                            Bundle opts = new Bundle(key.options);
                            opts.putAll(options);
                            options = opts;
                        }
                        try {
                            if (key.allIntents != null && key.allIntents.length > 1) {
                                Intent[] allIntents = new Intent[key.allIntents.length];
                                String[] allResolvedTypes = new String[key.allIntents.length];
                                System.arraycopy(key.allIntents, 0, allIntents, 0,
                                        key.allIntents.length);
                                if (key.allResolvedTypes != null) {
                                    System.arraycopy(key.allResolvedTypes, 0, allResolvedTypes, 0,
                                            key.allResolvedTypes.length);
                                }
                                allIntents[allIntents.length-1] = finalIntent;
                                allResolvedTypes[allResolvedTypes.length-1] = resolvedType;
                                owner.startActivitiesInPackage(uid, key.packageName, allIntents,
                                        allResolvedTypes, resultTo, options, userId);
                            } else {
                                owner.startActivityInPackage(uid, key.packageName, finalIntent,
                                        resolvedType, resultTo, resultWho, requestCode, 0,
                                        options, userId, container, null);
                            }
                        } catch (RuntimeException e) {
                            Slog.w(TAG, "Unable to send startActivity intent", e);
                        }
                        break;
                    case ActivityManager.INTENT_SENDER_ACTIVITY_RESULT:
                        if (key.activity.task.stack != null) {
                            key.activity.task.stack.sendActivityResultLocked(-1, key.activity,
                                    key.who, key.requestCode, code, finalIntent);
                        }
                        break;
                    case ActivityManager.INTENT_SENDER_BROADCAST:
                        try {
                            // If a completion callback has been requested, require
                            // that the broadcast be delivered synchronously
                            int sent = owner.broadcastIntentInPackage(key.packageName, uid,
                                    finalIntent, resolvedType, finishedReceiver, code, null, null,
                                    requiredPermission, options, (finishedReceiver != null),
                                    false, userId);
                            if (sent == ActivityManager.BROADCAST_SUCCESS) {
                                sendFinish = false;
                            }
                        } catch (RuntimeException e) {
                            Slog.w(TAG, "Unable to send startActivity intent", e);
                        }
                        break;
                    case ActivityManager.INTENT_SENDER_SERVICE:
                        try {
                            owner.startServiceInPackage(uid, finalIntent,
                                    resolvedType, key.packageName, userId);
                        } catch (RuntimeException e) {
                            Slog.w(TAG, "Unable to send startService intent", e);
                        } catch (TransactionTooLargeException e) {
                            res = ActivityManager.START_CANCELED;
                        }
                        break;
                }

                // 处理分发结果！
                if (sendFinish && res != ActivityManager.START_CANCELED) {
                    try {
                        finishedReceiver.performReceive(new Intent(finalIntent), 0,
                                null, null, false, false, key.userId);
                    } catch (RemoteException e) {
                    }
                }

                Binder.restoreCallingIdentity(origId);

                return res;
            }
        }
        return ActivityManager.START_CANCELED;
    }
```


## 2.2 Alarm.Listener.doAlarm


Alarm.Listener 是一个 Binder 对象，其服务端是应用进程的 ListenerWrapper 对象，之前我们知道 ListenerWrapper 实现了 IAlarmListener.Stub：

```java

```



# 3 特殊的触发路径

如果没有 Alarm 驱动，那么我们知道 Alarm 的触发是通过内部的一个 AlarmHandler 实现的：

```java
    private void setLocked(int type, long when) {
        if (mNativeData != 0) {
            ... ... ...
        } else {
            Message msg = Message.obtain();
            msg.what = ALARM_EVENT;
            mHandler.removeMessages(ALARM_EVENT);
            mHandler.sendMessageAtTime(msg, when);
        }
    }
```
可以看到，最后发送的是 ALARM_EVENT 给 AlarmHandler：

## 3.1 AlarmHandler.handleMessage[ALARM_EVENT]

```java
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case ALARM_EVENT: {
                    // 同样的 triggerList！
                    ArrayList<Alarm> triggerList = new ArrayList<Alarm>();
                    synchronized (mLock) {
                        final long nowRTC = System.currentTimeMillis();
                        final long nowELAPSED = SystemClock.elapsedRealtime();
                        // 调用 triggerAlarmsLocked 收集触发的 alarm！
                        triggerAlarmsLocked(triggerList, nowELAPSED, nowRTC);
                        updateNextAlarmClockLocked(); // 更新下一个 AlarmClock
                    }

                    // 分发 Alarm！
                    for (int i=0; i<triggerList.size(); i++) {
                        Alarm alarm = triggerList.get(i);
                        try {
                            alarm.operation.send(); // 这里再次调用了 PendingIntent 的 send 方法！
                        } catch (PendingIntent.CanceledException e) {
                            if (alarm.repeatInterval > 0) { // 发送出问题，如果是重复性 Alarm，那就移除该 Alarm！
                                removeImpl(alarm.operation);
                            }
                        }
                    }
                    break;
                }
            }
        }
```
这个方法逻辑很简单，不多说了！