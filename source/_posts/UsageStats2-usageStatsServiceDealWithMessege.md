# UsageStats 第 2 篇 - UsageStatsService 消息处理
title: UsageStats 第 2 篇 - UsageStatsService 消息处理
date: 2017/04/03 20:46:25
categories:
- AndroidFramework源码分析
- UsageStats使用状态管理
tags: UsageStats使用状态管理
---

[toc]

基于 Android7.1.1 源码分析 UsageStatsService 的架构和原理

UsageStatsService 定义的事件类型目前一共有 8 种类型，全部定义在 UsageEvents.java 中，如下：

```java
    /**
     * An event type denoting that a component moved to the foreground.
     */
    public static final int MOVE_TO_FOREGROUND = 1;

    /**
     * An event type denoting that a component moved to the background.
     */
    public static final int MOVE_TO_BACKGROUND = 2;

    /**
     * An event type denoting that a component was in the foreground when the stats
     * rolled-over. This is effectively treated as a {@link #MOVE_TO_BACKGROUND}.
     * {@hide}
     */
    public static final int END_OF_DAY = 3;

    /**
     * An event type denoting that a component was in the foreground the previous day.
     * This is effectively treated as a {@link #MOVE_TO_FOREGROUND}.
     * {@hide}
     */
    public static final int CONTINUE_PREVIOUS_DAY = 4;

    /**
     * An event type denoting that the device configuration has changed.
     */
    public static final int CONFIGURATION_CHANGE = 5;

    /**
     * An event type denoting that a package was interacted with in some way by the system.
     * @hide
     */
    public static final int SYSTEM_INTERACTION = 6;

    /**
     * An event type denoting that a package was interacted with in some way by the user.
     */
    public static final int USER_INTERACTION = 7;

    /**
     * An event type denoting that an action equivalent to a ShortcutInfo is taken by the user.
     *
     * @see android.content.pm.ShortcutManager#reportShortcutUsed(String)
     */
    public static final int SHORTCUT_INVOCATION = 8;
```

# 0 综述

UsageStatsService 的消息处理是在 H 中， 下面我们去看看具体的消息处理！

# 1 消息：MSG_REPORT_EVENT - OK

## 1.0 消息触发时机

当 AcitivtyManagerService 调用 LocalService 的以下方法的时候，会发送 MSG_REPORT_EVENT 消息！

```java
    private final class LocalService extends UsageStatsManagerInternal {

        @Override
        public void reportEvent(ComponentName component, int userId, int eventType) {
            if (component == null) {
                Slog.w(TAG, "Event reported without a component name");
                return;
            }

            UsageEvents.Event event = new UsageEvents.Event();
            event.mPackage = component.getPackageName();
            event.mClass = component.getClassName();

            // This will later be converted to system time.
            event.mTimeStamp = SystemClock.elapsedRealtime();

            event.mEventType = eventType;
            mHandler.obtainMessage(MSG_REPORT_EVENT, userId, 0, event).sendToTarget();
        }
```
还有其他的 report 方法：
```java
        @Override
        public void reportEvent(String packageName, int userId, int eventType) {...}

        @Override
        public void reportConfigurationChange(Configuration config, int userId) {...}

        @Override
        public void reportShortcutUsage(String packageName, String shortcutId, int userId) {...}
}
```
这些方法会接收到传入的 event，然后发送 MSG_REPORT_EVENT 消息！

```java
        case MSG_REPORT_EVENT:
            reportEvent((UsageEvents.Event) msg.obj, msg.arg1);
            break;
```
该消息会携带  2 个重要数据：UsageEvents 和 userId！


## 1.1 UsageStatsService.reportEvent

```java
    void reportEvent(UsageEvents.Event event, int userId) {
        synchronized (mLock) {
            //【*1.1.1】获得当前的实际时间点；
            final long timeNow = checkAndGetTimeLocked();
            final long elapsedRealtime = SystemClock.elapsedRealtime();
            //【*1.1.2】将 UsageEvents 的时间戳转为系统时间；
            convertToSystemTimeLocked(event);
            //【*1.1.3】获得 userId 的使用信息；
            final UserUsageStatsService service =
                    getUserDataAndInitializeIfNeededLocked(userId, timeNow);
    
            //【*1.1.4】判断 event 所属应用当前是否处于 idle 状态！
            final boolean previouslyIdle = mAppIdleHistory.isIdleLocked(
                    event.mPackage, userId, elapsedRealtime);
                    
            //【1】调用了 UserUsageStatsService 的 reportEvent 方法记录该 event！
            service.reportEvent(event);
            
            //【2】如果本次的 event type 是如下类型，进入下面的 IF 分支！
            if ((event.mEventType == Event.MOVE_TO_FOREGROUND
                    || event.mEventType == Event.MOVE_TO_BACKGROUND
                    || event.mEventType == Event.SYSTEM_INTERACTION
                    || event.mEventType == Event.USER_INTERACTION)) {
    
                //【×1.1.6】更新下 app idle 信息！
                mAppIdleHistory.reportUsageLocked(event.mPackage, userId, elapsedRealtime);

                //【2.1】如果之前是 idle 状态，此时退出 idle 状态！
                if (previouslyIdle) {
                    mHandler.sendMessage(mHandler.obtainMessage(MSG_INFORM_LISTENERS, userId,
                            /* idle = */ 0, event.mPackage));
                    notifyBatteryStats(event.mPackage, userId, false);
                }
            }
        }
    }
```

对于 UserUsageStatsService.reportEvent 方法，这里不再多说！

### 1.1.1 UsageStatsService.checkAndGetTimeLocked

checkAndGetTimeLocked 方法用于计算当前的实际时间！

```java
    private long checkAndGetTimeLocked() {
        //【1】获得当前的系统时间，可以被系统设置修改；
        final long actualSystemTime = System.currentTimeMillis();
        //【2】获得自开机后，经过的时间，包括深度睡眠的时间；
        final long actualRealtime = SystemClock.elapsedRealtime();

        final long expectedSystemTime = (actualRealtime - mRealTimeSnapshot) + mSystemTimeSnapshot;
        final long diffSystemTime = actualSystemTime - expectedSystemTime;
        //【3】判断时间是否有发生变化，
        if (Math.abs(diffSystemTime) > TIME_CHANGE_THRESHOLD_MILLIS) {
            // The time has changed.
            Slog.i(TAG, "Time changed in UsageStats by " + (diffSystemTime / 1000) + " seconds");
            final int userCount = mUserState.size();
            for (int i = 0; i < userCount; i++) {
                final UserUsageStatsService service = mUserState.valueAt(i);
                //【3.1】更新本地数据！
                service.onTimeChanged(expectedSystemTime, actualSystemTime);
            }
            
            //【3.2】记录本次的时间点到 mRealTimeSnapshot，mSystemTimeSnapshot！
            mRealTimeSnapshot = actualRealtime;
            mSystemTimeSnapshot = actualSystemTime;
        }
        //【4】返回当前的实际时间！
        return actualSystemTime;
    }
```
可以看到 checkAndGetTimeLocked 返回的时间是  actualSystemTime 的值，也就是 System.currentTimeMillis()，这个时间值可以被系统设置修改，然后值就会发生跳变，比如联网对时，手动调时！

而 actualRealtime 的值为 System.elapsedRealtime 自开机后，经过的时间，包括深度睡眠的时间，这部分时间值是不会被修改；

如果判断时间是否有发生调时，对时情况呢？

- mSystemTimeSnapshot 中保存的是上一次 check 时的系统时间；
- mRealTimeSnapshot 中保存的是上一次 check 时的自开机后，经过的时间；
- 先计算出期望的时间：
    - 本次距离开机的时间 actualRealtime - 上次距离开机的时间 mRealTimeSnapshot，这个时间差值是正常情况下的时间差值；
    - 然后再加上上一次 check 时的系统时间 mSystemTimeSnapshot，如果没有发生调时的话，这个应该是理想的时间点 expectedSystemTime；
    - 如果发生了调时，对时的情况，actualSystemTime 一定是会发生变化的！
    - 计算 actualSystemTime 和 expectedSystemTime 的差值，如果大于 TIME_CHANGE_THRESHOLD_MILLIS，说明铁定发生了调时，对时；

如果发生上述情况，那就调用 UserUsageStatsService.onTimeChanged 更新本地持久化文件的日期！

对于 UsageStatsService 是如何存储应用数据，如何更新本地持久化文件的，这里我先不关注，我们只需要知道，该方法返回的时间值是实际的时间（正常，手动调时，联网对时）

对于 service.onTimeChanged 方法，这里不再分析！

### 1.1.2 AppIdleHistory.convertToSystemTimeLocked

UsageEvents.Event 的 mTimeStamp 是在设置的时候，是通过 SystemClock.elapsedRealtime() 计算的；
```java
    private void convertToSystemTimeLocked(UsageEvents.Event event) {
        event.mTimeStamp = Math.max(0, event.mTimeStamp - mRealTimeSnapshot) + mSystemTimeSnapshot;
    }
```
convertToSystemTimeLocked 的作用是：

- 将相对于开机的时间转为系统时间：SystemClock.elapsedRealtime() -> SystemClock.currentTimeMillis()!


### 1.1.3 UsageStatsS.getUserDataAndInitializeIfNeededLocked


尝试获得指定 userId 对应的 UserUsageStatsService 对象，封装了该 userId 的使用情况！

```java
    private UserUsageStatsService getUserDataAndInitializeIfNeededLocked(int userId,
            long currentTimeMillis) {
        //【1】尝试获得 userId 对应的使用情况
        UserUsageStatsService service = mUserState.get(userId);
        //【1.1.3.1】如果为 null，那就默认会初始化一个！
        if (service == null) {
            service = new UserUsageStatsService(getContext(), userId,
                    new File(mUsageStatsDir, Integer.toString(userId)), this);
            service.init(currentTimeMillis);
            mUserState.put(userId, service);
        }
        //【3】返回对应的 UserUsageStatsService 对象
        return service;
    }
```

这里要涉及到另外一个数据结构 UserUsageStatsService，UserUsageStatsService 用于保存每个 userId 的使用情况！

UsageStatsService 内部会通过 mUserState 来记录 userId 的使用情况：
```java
    private final SparseArray<UserUsageStatsService> mUserState = new SparseArray<>();
```
#### 1.1.3.1 new UserUsageStatsService

创建一个 UserUsageStatsService 对象！
```java
    UserUsageStatsService(Context context, int userId, File usageStatsDir,
            StatsUpdatedListener listener) {
        mContext = context;
        mDailyExpiryDate = new UnixCalendar(0);
        mDatabase = new UsageStatsDatabase(usageStatsDir);
        mCurrentStats = new IntervalStats[UsageStatsManager.INTERVAL_COUNT];
        mListener = listener;
        mLogPrefix = "User[" + Integer.toString(userId) + "] ";
        mUserId = userId;
    }
```


### 1.1.4 AppIdleHistory.isIdleLocked

判断在 elapsedRealtime 指定的时间点下，该 package 是否是 idle 状态！

```java
    public boolean isIdleLocked(String packageName, int userId, long elapsedRealtime) {
        //【1.4.1】获得该 userId 下的历史信息！
        ArrayMap<String, PackageHistory> userHistory = getUserHistoryLocked(userId);
        //【1.4.2】获得该 pacakge 的历史信息 PackageHistory！
        PackageHistory packageHistory =
                getPackageHistoryLocked(userHistory, packageName, elapsedRealtime);
        //【3】如果 packageHistory，那就返回 false，默认为不处于 idle 状态！
        if (packageHistory == null) {
            return false;
        } else {
            //【1.1.4.1】判断该 package 是否属于 idle 状态！
            return hasPassedThresholdsLocked(packageHistory, elapsedRealtime);
        }
    }
```

#### 1.1.4.1 AppIdleHistory.hasPassedThresholdsLocked

接下来看看系统是如何判断一个应用是否进入 idle 状态！
```java
    private boolean hasPassedThresholdsLocked(PackageHistory packageHistory, long elapsedRealtime) {
        return (packageHistory.lastUsedScreenTime
                    <= getScreenOnTimeLocked(elapsedRealtime) - mScreenOnTimeThreshold)
                && (packageHistory.lastUsedElapsedTime
                        <= getElapsedTimeLocked(elapsedRealtime) - mElapsedTimeThreshold);
    }
```
需要满足 2 个条件：

- 自开机时算起，当前时间距离应用上次被使用的时间（自开机时算起），超过了 24 小时！
- 只算亮屏时间，当前时间距离应用上次被使用的时间（只算亮屏时间），超过了 12 小时！


### 1.1.6 AppIdleHistory.reportUsageLocked

根据 report event 的时间点，更新 app idle 相关的时间信息！

```java
    public void reportUsageLocked(String packageName, int userId, long elapsedRealtime) {
        ArrayMap<String, PackageHistory> userHistory = getUserHistoryLocked(userId);
        PackageHistory packageHistory = getPackageHistoryLocked(userHistory, packageName,
                elapsedRealtime);

        shiftHistoryToNow(userHistory, elapsedRealtime);
        //【1】自开机后，该应用最后一次使用的时间；     
        packageHistory.lastUsedElapsedTime = mElapsedDuration  + (elapsedRealtime - mElapsedSnapshot);
        //【2】在只计算亮屏时间的条件下，该应用最后一次使用的时间；
        packageHistory.lastUsedScreenTime = getScreenOnTimeLocked(elapsedRealtime);

        packageHistory.recent[HISTORY_SIZE - 1] = FLAG_LAST_STATE | FLAG_PARTIAL_ACTIVE;
    }
```
我们看到，当每次 report event 后，会更新 PackageHistory 的如下两个变量：

- **lastUsedElapsedTime**：自开机后，应用最后一次被使用的时间；
- **lastUsedScreenTime**：只计算亮屏时间的情况下，应用最后一次被使用的时间；


# 2 消息：MSG_FLUSH_TO_DISK - OK


## 2.0 消息触发时机

在如下的情况下会触发 UsageStatsService.onStatsUpdated ：

- **UserUsageStatsService.init** 方法在加载完数据后，会将 LastEvent 为 MOVE_TO_FOREGROUND 或者 CONTINUE_PREVIOUS_DAY 的 UsageStats 的  LastEvent 改为 END_OF_DAY，此时会触发 notifyStatsChanged 方法！

</br>

- **UserUsageStatsService.reportEvent** 方法处理完上报的事件后，会触发 notifyStatsChanged 方法！

</br>

- **UserUsageStatsService.rolloverStats** 方法一天时间过去会后，数据进行回滚后，会触发 notifyStatsChanged 方法！

</br>


当 UserUsageStatsService 的 notifyStatsChanged 触发后，会触发 UsageStatsService.onStatsUpdated 方法：
```java
    private void notifyStatsChanged() {
        if (!mStatsChanged) {
            mStatsChanged = true;
            mListener.onStatsUpdated();
        }
    }
```
在 UsageStatsService.onStatsUpdated 方法中会发送 MSG_FLUSH_TO_DISK 消息！
```
    @Override
    public void onStatsUpdated() {
        //【1】延迟 20mins 发送 MSG_FLUSH_TO_DISK 消息！
        mHandler.sendEmptyMessageDelayed(MSG_FLUSH_TO_DISK, FLUSH_INTERVAL);
    }
```
这里延迟了 FLUSH_INTERVAL 时间间隔发送 MSG_FLUSH_TO_DISK 消息！
```java
    static final boolean COMPRESS_TIME = false;

    private static final long TEN_SECONDS = 10 * 1000;
    private static final long TWENTY_MINUTES = 20 * 60 * 1000;
    private static final long FLUSH_INTERVAL = COMPRESS_TIME ? TEN_SECONDS : TWENTY_MINUTES;
```
由于 COMPRESS_TIME 为 false，所以是 20mins！

最后进入 H：
```java
        case MSG_FLUSH_TO_DISK:
            //【2.1】进入 flushToDisk 方法！
            flushToDisk();
            break;
```

MSG_FLUSH_TO_DISK 消息用于将 App idle 的数据写入本地持久化文件！

## 2.1 UsageStatsService.flushToDisk

调用了 flushToDisk 方法：
```java
    void flushToDisk() {
        synchronized (mLock) {
            //【2.2】调用了 flushToDiskLocked 方法！
            flushToDiskLocked ();
        }
    }
```
继续来看：

## 2.2 UsageStatsService.flushToDiskLocked
```java
    private void flushToDiskLocked() {
        final int userCount = mUserState.size();
        for (int i = 0; i < userCount; i++) {
            UserUsageStatsService service = mUserState.valueAt(i);
            //【1】这里会调用 UserUsageStatsService.persistActiveStats 方法，将该 userId 下的最新 UsageStats 信息
            // 持久化到本地文件！
            service.persistActiveStats();
            
            //【×2.3.1】将 app idle 的时间信息保存到本地文件！
            mAppIdleHistory.writeAppIdleTimesLocked(mUserState.keyAt(i));
        }

        // Persist elapsed and screen on time. If this fails for whatever reason, the apps will be
        // considered not-idle, which is the safest outcome in such an event.
        //【×2.3.2】将 app idle 的时间间隔保存到本地文件！
        mAppIdleHistory.writeAppIdleDurationsLocked();
        mHandler.removeMessages(MSG_FLUSH_TO_DISK);
    }
```
对于 UserUsageStatsService.persistActiveStats，这里我们不再分析，可以看 UserUsageStatsService 分析！

### 2.2.1 AppIdleHistory.writeAppIdleTimesLocked

将系统中的 app idle 信息保存到本地文件中！
```java
    public void writeAppIdleTimesLocked(int userId) {
        FileOutputStream fos = null;
        //【1】目标文件 /data/system/users/<userId>/app_idle_stats.xml 文件
        AtomicFile appIdleFile = new AtomicFile(getUserFile(userId));
        try {
            fos = appIdleFile.startWrite();
            final BufferedOutputStream bos = new BufferedOutputStream(fos);

            FastXmlSerializer xml = new FastXmlSerializer();
            xml.setOutput(bos, StandardCharsets.UTF_8.name());
            xml.startDocument(null, true);
            xml.setFeature("http://xmlpull.org/v1/doc/features.html#indent-output", true);

            xml.startTag(null, TAG_PACKAGES); //  packages 标签

            ArrayMap<String,PackageHistory> userHistory = getUserHistoryLocked(userId);
            final int N = userHistory.size();
            for (int i = 0; i < N; i++) {
                //【1】遍历处理每一个 PackageHistory 对象！
                String packageName = userHistory.keyAt(i);
                PackageHistory history = userHistory.valueAt(i);
                xml.startTag(null, TAG_PACKAGE); //  package 标签
                xml.attribute(null, ATTR_NAME, packageName); //  name 标签
                xml.attribute(null, ATTR_ELAPSED_IDLE,
                        Long.toString(history.lastUsedElapsedTime)); // elapsedIdleTime 标签
                xml.attribute(null, ATTR_SCREEN_IDLE,
                        Long.toString(history.lastUsedScreenTime)); // screenIdleTime 标签

                xml.endTag(null, TAG_PACKAGE);
            }

            xml.endTag(null, TAG_PACKAGES);
            xml.endDocument();
            appIdleFile.finishWrite(fos);
        } catch (Exception e) {
            appIdleFile.failWrite(fos);
            Slog.e(TAG, "Error writing app idle file for user " + userId);
        }
    }
```
我们来看看，保存后的文件内容：
```java
<packages>
    <package name="com.github.shadowsocks" elapsedIdleTime="6836535861" screenIdleTime="2173999944" />
</packages>
```

### 2.2.2 AppIdleHistory.writeAppIdleDurationsLocked

将系统中的屏幕亮灭时间信息保存到本地文件中！

```java
    public void writeAppIdleDurationsLocked() {
        final long elapsedRealtime = SystemClock.elapsedRealtime();
        // Only bump up and snapshot the elapsed time. Don't change screen on duration.
        mElapsedDuration += elapsedRealtime - mElapsedSnapshot;
        mElapsedSnapshot = elapsedRealtime;
        
        //【2.2.2.1】调用 writeScreenOnTimeLocked 方法！
        writeScreenOnTimeLocked();
    }
```

#### 2.2.2.1 AppIdleHistory.writeScreenOnTimeLocked
```java
    private void writeScreenOnTimeLocked() {
        //【1】目标文件： /data/system/screen_on_time
        AtomicFile screenOnTimeFile = new AtomicFile(getScreenOnTimeFile());
        FileOutputStream fos = null;
        try {
            fos = screenOnTimeFile.startWrite();
            //【2】将 mScreenOnDuration 和 mElapsedDuration 写入到 screen_on_time 文件中！
            fos.write((Long.toString(mScreenOnDuration) + "\n"
                    + Long.toString(mElapsedDuration) + "\n").getBytes());
            screenOnTimeFile.finishWrite(fos);
        } catch (IOException ioe) {
            screenOnTimeFile.failWrite(fos);
        }
    }
```
我们来看看 /data/system/screen_on_time 文件内容！
```java
2201351882
6898458370
```

# 3 消息：MSG_REMOVE_USER - OK

```java
        case MSG_REMOVE_USER:
            //【3.1】调用 onUserRemoved 方法！
            onUserRemoved(msg.arg1);
            break;
```

## 3.0 消息触发机制

UsageStatsService 内部有一个广播接收者 UserActionsReceiver，用于监听 User 相关的广播！

```java
    private class UserActionsReceiver extends BroadcastReceiver {
        @Override
        public void onReceive(Context context, Intent intent) {
            final int userId = intent.getIntExtra(Intent.EXTRA_USER_HANDLE, -1);
            final String action = intent.getAction();
            //【1】监听到 User 被移除的广播！
            if (Intent.ACTION_USER_REMOVED.equals(action)) {
                if (userId >= 0) {
                    mHandler.obtainMessage(MSG_REMOVE_USER, userId, 0).sendToTarget();
                }
            } else if (Intent.ACTION_USER_STARTED.equals(action)) {
                if (userId >=0) {
                    postCheckIdleStates(userId);
                }
            }
        }
    }
```
当其接收到用户被移除的广播，接受到被启动的 UserId，会发送 MSG_REMOVE_USER 消息!


## 3.1 UsageStatsService.onUserRemoved

这里调用了 onUserRemoved 方法，我们去看看该方法的逻辑：
```java
    void onUserRemoved(int userId) {
        synchronized (mLock) {
            Slog.i(TAG, "Removing user " + userId + " and all data.");
            //【1】从 mUserState 中移除该 userId 对应的 UserUsageStatsService 实例；
            mUserState.remove(userId);
            //【3.1.1】从 AppIdleHistory 中移除该 userId 下的 app idle 信息；
            mAppIdleHistory.onUserRemoved(userId);
            //【3】清楚被移除的 userId 对应的 UsageStats 信息！
            cleanUpRemovedUsersLocked();
        }
    }
```
### 3.1.1 AppIdleHistory.onUserRemoved
```java
    public void onUserRemoved(int userId) {
        mIdleHistory.remove(userId);
    }
```
流程很简单，不多说了！

# 4 消息：MSG_INFORM_LISTENERS - OK

## 4.0 消息触发时机

```java
        case MSG_INFORM_LISTENERS:
            informListeners((String) msg.obj, msg.arg1, msg.arg2 == 1);
            break;
```

## 4.1 UsageStatsService.informListeners

informListeners 用于通知那件监听 app idle 的应用，app idle 状态的变化！
```java
    void informListeners(String packageName, int userId, boolean isIdle) {
        for (AppIdleStateChangeListener listener : mPackageAccessListeners) {
            listener.onAppIdleStateChanged(packageName, userId, isIdle);
        }
    }
```
逻辑很见简单，不多说了！

# 5 消息：MSG_FORCE_IDLE_STATE

## 5.0 消息触发时机

当 BinderService 的 setAppInactive 方法触发后！

```java
private final class BinderService extends IUsageStatsManager.Stub {

        @Override
        public void setAppInactive(String packageName, boolean idle, int userId) {
            final int callingUid = Binder.getCallingUid();
            try {
                userId = ActivityManagerNative.getDefault().handleIncomingUser(
                        Binder.getCallingPid(), callingUid, userId, false, true,
                        "setAppIdle", null);
            } catch (RemoteException re) {
                throw re.rethrowFromSystemServer();
            }
            getContext().enforceCallingPermission(Manifest.permission.CHANGE_APP_IDLE_STATE,
                    "No permission to change app idle state");
            final long token = Binder.clearCallingIdentity();
            try {
                final int appId = getAppId(packageName);
                if (appId < 0) return;
                //【1】调用了 UsageStatsService.setAppIdle 方法！
                UsageStatsService.this.setAppIdle(packageName, idle, userId);
            } finally {
                Binder.restoreCallingIdentity(token);
            }
        }

}
```
UsageStatsService.setAppIdle 会发送 MSG_FORCE_IDLE_STATE 消息给 H:
```java
    void setAppIdle(String packageName, boolean idle, int userId) {
        if (packageName == null) return;

        mHandler.obtainMessage(MSG_FORCE_IDLE_STATE, userId, idle ? 1 : 0, packageName)
                .sendToTarget();
    }
```
我们来看看 H 对于消息 MSG_FORCE_IDLE_STATE 的处理：
```java
        case MSG_FORCE_IDLE_STATE:
            //【5.1】调用 forceIdleState 将 app 设置为 idle 状态！
            forceIdleState((String) msg.obj, msg.arg1, msg.arg2 == 1);
            break;
```

## 5.1 UsageStatsService.forceIdleState

设置 app 进入 idle 状态：

```java
    void forceIdleState(String packageName, int userId, boolean idle) {
        final int appId = getAppId(packageName);
        if (appId < 0) return;
        synchronized (mLock) {
            final long elapsedRealtime = SystemClock.elapsedRealtime();
            //【×6.1.1】判断 app 之前是否处于 idle 状态！
            final boolean previouslyIdle = isAppIdleFiltered(packageName, appId,
                    userId, elapsedRealtime);
                    
            //【×5.1.1】设置 app 的 idle 的状态！
            mAppIdleHistory.setIdleLocked(packageName, userId, idle, elapsedRealtime);
            
            //【×6.1.1】判断 app 之前现在是否处于 idle 状态！
            final boolean stillIdle = isAppIdleFiltered(packageName, appId,
                    userId, elapsedRealtime);
                    
            //【1】当前后 app idle 状态发生了变化，那就通知监听 app idle 的监听者！
            if (previouslyIdle != stillIdle) {
                mHandler.sendMessage(mHandler.obtainMessage(MSG_INFORM_LISTENERS, userId,
                        /* idle = */ stillIdle ? 1 : 0, packageName));
                        
                //【5.1.2】如果此时不处于 idle 状态，通知 PowerManagerService，此时退出了 idle 状态！
                if (!stillIdle) {
                    notifyBatteryStats(packageName, userId, idle);
                }
            }
        }
    }
```

### 5.1.1 AppIdleHistory.setIdleLocked

设置 package 的 idle 状态！
```java
    public void setIdleLocked(String packageName, int userId, boolean idle, long elapsedRealtime) {
        //【1】同样先获得该 userId 下该 packageName 的 PackageHistory 对象！
        ArrayMap<String, PackageHistory> userHistory = getUserHistoryLocked(userId);
        PackageHistory packageHistory = getPackageHistoryLocked(userHistory, packageName,
                elapsedRealtime);

        //【5.1.1.1】设置 package 上一次被使用的时间
        packageHistory.lastUsedElapsedTime = getElapsedTimeLocked(elapsedRealtime)
                - mElapsedTimeThreshold;
        packageHistory.lastUsedScreenTime = getScreenOnTimeLocked(elapsedRealtime)
                - (idle ? mScreenOnTimeThreshold : 0) - 1000 /* just a second more */;
    }
```
对于 getUserHistoryLocked 和 getPackageHistoryLocked 我们不再重新分析！

#### 5.1.1.1 AppIdleHistory.getElapsedTimeLocked
```java
    private long getElapsedTimeLocked(long elapsedRealtime) {
        return (elapsedRealtime - mElapsedSnapshot + mElapsedDuration);
    }
```

#### 5.1.1.2 AppIdleHistory.getScreenOnTimeLocked

获得亮屏的总时长！

```java
    public long getScreenOnTimeLocked(long elapsedRealtime) {
        long screenOnTime = mScreenOnDuration;
        if (mScreenOn) {
            screenOnTime += elapsedRealtime - mScreenOnSnapshot;
        }
        return screenOnTime;
    }
```
如果此时亮屏 mScreenOn 为 true，那么会在 mScreenOnDuration 基础上，再加上当前时间减去上次亮屏的时间点 mScreenOnSnapshot！

### 5.1.2 UsageStatsService.notifyBatteryState

通知 PowerManagerService，app 退出了 idle 状态！

````java
    private void notifyBatteryStats(String packageName, int userId, boolean idle) {
        try {
            final int uid = mPackageManager.getPackageUidAsUser(packageName,
                    PackageManager.MATCH_UNINSTALLED_PACKAGES, userId);
            if (idle) {
                mBatteryStats.noteEvent(BatteryStats.HistoryItem.EVENT_PACKAGE_INACTIVE,
                        packageName, uid);
            } else {
                mBatteryStats.noteEvent(BatteryStats.HistoryItem.EVENT_PACKAGE_ACTIVE,
                        packageName, uid);
            }
        } catch (NameNotFoundException | RemoteException e) {
        }
    }
```

# 6 消息：MSG_CHECK_IDLE_STATES

```java
        case MSG_CHECK_IDLE_STATES:
            if (checkIdleStates(msg.arg1)) {
                mHandler.sendMessageDelayed(mHandler.obtainMessage(
                        MSG_CHECK_IDLE_STATES, msg.arg1, 0),
                        mCheckIdleIntervalMillis);
            }
            break;
```

## 6.0 消息触发机制

UsageStatsService 内部有一个广播接收者 UserActionsReceiver，用于监听 User 相关的广播！

```java
    private class UserActionsReceiver extends BroadcastReceiver {
        @Override
        public void onReceive(Context context, Intent intent) {
            final int userId = intent.getIntExtra(Intent.EXTRA_USER_HANDLE, -1);
            final String action = intent.getAction();
            if (Intent.ACTION_USER_REMOVED.equals(action)) {
                if (userId >= 0) {
                    mHandler.obtainMessage(MSG_REMOVE_USER, userId, 0).sendToTarget();
                }
            } else if (Intent.ACTION_USER_STARTED.equals(action)) {
                //【1】监听到 User 被启动的广播！
                if (userId >=0) {
                    postCheckIdleStates(userId);
                }
            }
        }
    }
```
当其接收到用户被启动的广播，接受到被启动的 UserId：
```java
    void postCheckIdleStates(int userId) {
        mHandler.sendMessage(mHandler.obtainMessage(MSG_CHECK_IDLE_STATES, userId, 0));
    }
```

MSG_CHECK_IDLE_STATES 用于检查指定 userid 下的 app idle 状态：

```java
        case MSG_CHECK_IDLE_STATES:
            //【*6.1】调用 checkIdleStates 方法，检查 app idle 信息！
            if (checkIdleStates(msg.arg1)) {
                mHandler.sendMessageDelayed(mHandler.obtainMessage(
                        MSG_CHECK_IDLE_STATES, msg.arg1, 0),
                        mCheckIdleIntervalMillis);
            }
            break;
```
checkIdleStates 返回 true 表示检查操作完成，那么就会设置下一次的 check idle 的消息的发送时间，延迟  mCheckIdleIntervalMillis 8 hours 后再次发送 MSG_CHECK_IDLE_STATES 消息！

关于 mCheckIdleIntervalMillis 的取值，可以看下 SettingsObserver 相关代码！

## 6.1 UsageStatsService.checkIdleStates

checkIdleStates 方法用于检查当前时间系统中 app idle 的状态，同时更新 AppIdleHistory 中的数据！

```java
    boolean checkIdleStates(int checkUserId) {
        //【1】如果 mAppIdleEnabled 为 false，说明系统没有打开 device idle 状态！
        if (!mAppIdleEnabled) {
            return false;
        }
        
        //【2】获得系统中正在运行中的 UserId！
        // 如果要检查的 userId 不是 UserHandle.USER_ALL(所有用户)，并且也不是系统中正在运行的 userId
        // 那就返回 false；
        final int[] runningUserIds;
        try {
            runningUserIds = ActivityManagerNative.getDefault().getRunningUserIds();
            if (checkUserId != UserHandle.USER_ALL
                    && !ArrayUtils.contains(runningUserIds, checkUserId)) {
                return false;
            }
        } catch (RemoteException re) {
            throw re.rethrowFromSystemServer();
        }

        final long elapsedRealtime = SystemClock.elapsedRealtime();
        
        //【3】遍历收集到的所有正在运行中的 userId，如果指定的是所有用户 USER_ALL，那就检查所有的 userId
        // 否则，跳过那些不是 checkUserId 的 uid！
        for (int i = 0; i < runningUserIds.length; i++) {
            final int userId = runningUserIds[i];
            if (checkUserId != UserHandle.USER_ALL && checkUserId != userId) {
                continue;
            }
            if (DEBUG) {
                Slog.d(TAG, "Checking idle state for user " + userId);
            }

            //【3.1】获得 userId 下的所有的 package 信息！
            List<PackageInfo> packages = mPackageManager.getInstalledPackagesAsUser(
                    PackageManager.MATCH_DISABLED_COMPONENTS,
                    userId);
            final int packageCount = packages.size();

            //【3.2】遍历处理
            for (int p = 0; p < packageCount; p++) {
                final PackageInfo pi = packages.get(p);
                final String packageName = pi.packageName;
                
                //【*6.1.1】判断当前时间下，该 package 是否处于 idle 状态！
                final boolean isIdle = isAppIdleFiltered(packageName,
                        UserHandle.getAppId(pi.applicationInfo.uid),
                        userId, elapsedRealtime);
                        
                //【*4】发送 MSG_INFORM_LISTENERS 消息给，通知 app idle 监听者！
                mHandler.sendMessage(mHandler.obtainMessage(MSG_INFORM_LISTENERS,
                        userId, isIdle ? 1 : 0, packageName));
                        
                if (isIdle) {
                    synchronized (mLock) {
                        //【*6.1.2】如果 package 应该处于 idle 状态的话，那就通过 AppIdleHistory 记录下来！
                        mAppIdleHistory.setIdle(packageName, userId, elapsedRealtime);
                    }
                }
            }
        }
        if (DEBUG) {
            Slog.d(TAG, "checkIdleStates took "
                    + (SystemClock.elapsedRealtime() - elapsedRealtime));
        }
        return true;
    }
```

### 6.1.1 UsageStatsService.isAppIdleFiltered

isAppIdleFiltered 方法用于过滤一些特殊的条件！
```java
    private boolean isAppIdleFiltered(String packageName, int appId, int userId,
            long elapsedRealtime) {
        if (packageName == null) return false;
        //【1】如果不允许应用进入 idle 状态，显然，没有应用会处于 idle 状态！
        if (!mAppIdleEnabled) {
            return false;
        }
        //【2】如果 appId 小于 FIRST_APPLICATION_UID，说明是系统应用，系统应用不能进入 idle 状态！
        if (appId < Process.FIRST_APPLICATION_UID) {
            return false;
        }
        //【3】如果包名是 "android"，不能处于 idle 状态！
        if (packageName.equals("android")) {
            return false;
        }
        if (mSystemServicesReady) {
            try {
                //【4】如果 package 在 doze 模式的白名单中，那就不能进入 app idle (app standby) 状态
                if (mDeviceIdleController.isPowerSaveWhitelistExceptIdleApp(packageName)) {
                    return false;
                }
            } catch (RemoteException re) {
                throw re.rethrowFromSystemServer();
            }
            //【5】接着处理一些特殊情况，满足条件，也不能处于 idle 状态！
            if (isActiveDeviceAdmin(packageName, userId)) {
                return false;
            }

            if (isActiveNetworkScorer(packageName)) {
                return false;
            }

            if (mAppWidgetManager != null
                    && mAppWidgetManager.isBoundWidgetPackage(packageName, userId)) {
                return false;
            }

            if (isDeviceProvisioningPackage(packageName)) {
                return false;
            }
        }
        //【*6.1.1.1】进入 isAppIdleUnfiltered 方法！
        if (!isAppIdleUnfiltered(packageName, userId, elapsedRealtime)) {
            return false;
        }

        //【7】如果是运营商特权应用，不能进入 idle 状态！
        if (isCarrierApp(packageName)) {
            return false;
        }

        return true;
    }
```
接着来看 isAppIdleUnfiltered 方法！

#### 6.1.1.1 UsageStatsService.isAppIdleUnfiltered

isAppIdleUnfiltered 方法是不过滤条件，直接读取数据，判断 app 是否处于 idle 状态！一般是先调用 isAppIdleFiltered 过滤特殊情况，然后再调用该方法：
```java
    private boolean isAppIdleUnfiltered(String packageName, int userId, long elapsedRealtime) {
        synchronized (mLock) {
            //【×1.4】调用 isIdleLocked 判断是否是 idle 状态！
            return mAppIdleHistory.isIdleLocked(packageName, userId, elapsedRealtime);
        }
    }
```
我们看到，是直接调用 AppIdleHistory.isIdleLocked 方法读取数据！

对于 AppIdleHistory.isIdleLocked 方法，这里不再分析！


### 6.1.2 AppIdleHistory.setIdle

```java
    public void setIdle(String packageName, int userId, long elapsedRealtime) {
        //【1】从 mIdleHistory 读取该 userId 下的 app idle 数据，如果 userId 对应的数据为 null；
        // 会做一次初始化，从本地文件读取数据！
        ArrayMap<String, PackageHistory> userHistory = getUserHistoryLocked(userId);
        //【*6.1.2.1】获得 packageName 对应的 packageHistory！
        PackageHistory packageHistory = getPackageHistoryLocked(userHistory, packageName,
                elapsedRealtime);
                
        //【*6.1.2.2】根据周期调整数据！！
        shiftHistoryToNow(userHistory, elapsedRealtime);

        packageHistory.recent[HISTORY_SIZE - 1] &= ~FLAG_LAST_STATE;
    }
```

#### 6.1.2.1 AppIdleHistory.getPackageHistoryLocked

从指定的 userId 的 userHistory 获得该 package 的 PackageHistory 对象，如果没有就重新创建一个！

```java
    private PackageHistory getPackageHistoryLocked(ArrayMap<String, PackageHistory> userHistory,
            String packageName, long elapsedRealtime) {
        PackageHistory packageHistory = userHistory.get(packageName);
        //【1】如果没有该 package 的历史信息，那就初始化一个！
        if (packageHistory == null) {
            packageHistory = new PackageHistory();
            //【*6.1.2.1.1】初始化 lastUsedElapsedTime
            packageHistory.lastUsedElapsedTime = getElapsedTimeLocked(elapsedRealtime);
            //【*6.1.2.1.2】初始化 lastUsedScreenTime
            packageHistory.lastUsedScreenTime = getScreenOnTimeLocked(elapsedRealtime);
    
            userHistory.put(packageName, packageHistory);
        }
        return packageHistory;
    }
```
##### 6.1.2.1.1 AppIdleHistory.getElapsedTimeLocked
```java
    private long getElapsedTimeLocked(long elapsedRealtime) {
        return (elapsedRealtime - mElapsedSnapshot + mElapsedDuration);
    }
```

##### 6.1.2.1.2 AppIdleHistory.getScreenOnTimeLocked

获得亮屏的总时长！

```java
    public long getScreenOnTimeLocked(long elapsedRealtime) {
        long screenOnTime = mScreenOnDuration;
        if (mScreenOn) {
            screenOnTime += elapsedRealtime - mScreenOnSnapshot;
        }
        return screenOnTime;
    }
```
如果此时亮屏 mScreenOn 为 true，那么会在 mScreenOnDuration 基础上，再加上当前时间减去上次亮屏的时间点 mScreenOnSnapshot！

#### 6.1.2.2 AppIdleHistory.shiftHistoryToNow

```java
    private void shiftHistoryToNow(ArrayMap<String, PackageHistory> userHistory,
            long elapsedRealtime) {
        //【1】计算本次周期！
        long thisPeriod = elapsedRealtime / PERIOD_DURATION;

        // Has the period switched over? Slide all users' package histories
        if (mLastPeriod != 0 && mLastPeriod < thisPeriod
                && (thisPeriod - mLastPeriod) < HISTORY_SIZE - 1) {
            int diff = (int) (thisPeriod - mLastPeriod);
            final int NUSERS = mIdleHistory.size();
            for (int u = 0; u < NUSERS; u++) {
                userHistory = mIdleHistory.valueAt(u);
                for (PackageHistory idleState : userHistory.values()) {

                    // Shift left
                    System.arraycopy(idleState.recent, diff, idleState.recent, 0,
                            HISTORY_SIZE - diff);
                    // Replicate last state across the diff
                    for (int i = 0; i < diff; i++) {
                        idleState.recent[HISTORY_SIZE - i - 1] =
                            (byte) (idleState.recent[HISTORY_SIZE - diff - 1] & FLAG_LAST_STATE);
                    }
                }
            }
        }
        //【2】
        mLastPeriod = thisPeriod;
    }
```
mLastPeriod 用来记录能够更新的周期！

# 7 消息：MSG_ONE_TIME_CHECK_IDLE_STATES - OK

## 7.1 消息触发时机

- **UsageStatsService.onBootPhase**

```java
    @Override
    public void onBootPhase(int phase) {
        if (phase == PHASE_SYSTEM_SERVICES_READY) {
            ... ... ... ...

            if (mPendingOneTimeCheckIdleStates) {
                postOneTimeCheckIdleStates();
            }

            mSystemServicesReady = true;
        } else if (phase == PHASE_BOOT_COMPLETED) {
            setChargingState(getContext().getSystemService(BatteryManager.class).isCharging());
        }
    }
```
在 UsageStatsService 开机初始化的时候，会进行一次 app idle check！

- **SettingsObserver.onChange**

```java
    @Override
    public void onChange(boolean selfChange) {
        updateSettings();
        postOneTimeCheckIdleStates();
    }
```
当数据库时间属性变化后！

- **UsageStatsService.onStatsReloaded**

```java
    @Override
    public void onStatsReloaded() {
        postOneTimeCheckIdleStates();
    }
```

这几种情况下，都是开机初始化，或者数据库配置更新，使用状态重新加载的情况，这些情况会导致 app idle 发生变化，所以需要重新 check idle！


```java
        case MSG_ONE_TIME_CHECK_IDLE_STATES:
            mHandler.removeMessages(MSG_ONE_TIME_CHECK_IDLE_STATES);
            //【1】处理所有用户下的 app idle 检查操作！
            checkIdleStates(UserHandle.USER_ALL);
            break;
```

## 7.2 UsageStatsService.postOneTimeCheckIdleStates

```java
    void postOneTimeCheckIdleStates() {
        if (mDeviceIdleController == null) {
            // Not booted yet; wait for it!
            mPendingOneTimeCheckIdleStates = true;
        } else {
            mHandler.sendEmptyMessage(MSG_ONE_TIME_CHECK_IDLE_STATES);
            mPendingOneTimeCheckIdleStates = false;
        }
    }
```

MSG_ONE_TIME_CHECK_IDLE_STATES 也用于检查 device idle 状态！

```java
        case MSG_ONE_TIME_CHECK_IDLE_STATES:
            mHandler.removeMessages(MSG_ONE_TIME_CHECK_IDLE_STATES);
            checkIdleStates(UserHandle.USER_ALL);
            break;
```
和 MSG_CHECK_IDLE_STATES 的区别是：

- MSG_ONE_TIME_CHECK_IDLE_STATES 只会检测一次，而 MSG_CHECK_IDLE_STATES 会持续检查！
- MSG_ONE_TIME_CHECK_IDLE_STATES 会检查所有 user 下的 app idle 状态，而 MSG_CHECK_IDLE_STATES 是检查指定的 userId 下的 app idle 状态；

# 8 消息：MSG_CHECK_PAROLE_TIMEOUT

该消息用于进入下一次假释状态！

## 8.1 消息触发时机

我们在 DeviceStateReceiver 接收者中，当接收到 PowerManager.ACTION_DEVICE_IDLE_MODE_CHANGED 广播后，即 doze 模式发生了变化，我们会调用 onDeviceIdleModeChanged 方法！

```java
    void onDeviceIdleModeChanged() {
        final boolean deviceIdle = mPowerManager.isDeviceIdleMode();
        if (DEBUG) Slog.i(TAG, "DeviceIdleMode changed to " + deviceIdle);
        
        synchronized (mLock) {
            //【1】计算距离上次退出假释状态过去的时间！
            final long timeSinceLastParole = checkAndGetTimeLocked() - mLastAppIdleParoledTime;

            if (!deviceIdle 
                    && timeSinceLastParole >= mAppIdleParoleIntervalMillis) {
                //【1.1】如果当前不处于 device idle 状态，并且当前距离上次退出假释状态超过了 24 小时
                // 那么我们会进入假释状态！
                if (DEBUG) Slog.i(TAG, "Bringing idle apps out of inactive state due to deviceIdleMode=false");
                setAppIdleParoled(true);
                
            } else if (deviceIdle) {
                if (DEBUG) Slog.i(TAG, "Device idle, back to prison");
                //【1.2】如果当前处于 device idle 状态，那么无法进入假释状态！
                setAppIdleParoled(false);
                
            }
        }
    }
```

- **UsageStatsService.setAppIdleParoled**

setAppIdleParoled 方法用于将 app 在 idle 的状态下唤醒进入假释状态，参数 boolean paroled 表示当前是否进入假释状态！

mAppIdleTempParoled 用于保存假释状态，boolean paroled 表示是否进入假释状态！
```java
    void setAppIdleParoled(boolean paroled) {
        synchronized (mLock) {
            if (mAppIdleTempParoled != paroled) {
                //【1】缓存本次假释状态！
                mAppIdleTempParoled = paroled;
                if (DEBUG) Slog.d(TAG, "Changing paroled to " + mAppIdleTempParoled);
                if (paroled) {
                    //【1.1】如果本次是进入假释，设置退出假释的超时消息！
                    postParoleEndTimeout();

                } else {
                    //【1.2】如果本次是退出假释，设置下一次进入假释的消息，即 24Hours 后！
                    // 保存上次退出假释的时间！
                    mLastAppIdleParoledTime = checkAndGetTimeLocked();
                    
                    //【*8.1.1】设置进入下一次假释的消息！
                    postNextParoleTimeout();
                }
                
                //【common】处理假释状态的变化！
                postParoleStateChanged();
            }
        }
    }
```
最后，调用了 postNextParoleTimeout 方法！

### 8.1.1 UsageStatsService.postNextParoleTimeout

```java
    private void postNextParoleTimeout() {
        if (DEBUG) Slog.d(TAG, "Posting MSG_CHECK_PAROLE_TIMEOUT");
        mHandler.removeMessages(MSG_CHECK_PAROLE_TIMEOUT);

        //【1】在上次退出假释的时间基础上，加上 mAppIdleParoleIntervalMillis（24Hours）
        // 然后减去当前时间，得到延迟时间 timeLeft！
        long timeLeft = (mLastAppIdleParoledTime + mAppIdleParoleIntervalMillis)
                - checkAndGetTimeLocked();

        if (timeLeft < 0) {
            timeLeft = 0;
        }
        //【2】延迟 timeLeft 发送 MSG_CHECK_PAROLE_TIMEOUT 消息！
        mHandler.sendEmptyMessageDelayed(MSG_CHECK_PAROLE_TIMEOUT, timeLeft);
    }
```
我么你可以看到，延迟发送 MSG_CHECK_PAROLE_TIMEOUT 的时间是剩余的等待时间！！

我们来看看 H 对 MSG_CHECK_PAROLE_TIMEOUT 消息的处理：
```java
        case MSG_CHECK_PAROLE_TIMEOUT:
            //【8.2】调用了 checkParoleTimeout 方法！
            checkParoleTimeout();
            break;
```

## 8.2 UsageStatsService.checkParoleTimeout

当该消息触发后，进入 checkParoleTimeout 方法！

```java
    void checkParoleTimeout() {
        synchronized (mLock) {
            //【1】mAppIdleTempParoled 为 false，说明当前没有进入假释状态！
            if (!mAppIdleTempParoled) {
                //【1.1】计算距离上次退出假释模式，经过的时间！
                final long timeSinceLastParole = checkAndGetTimeLocked() - mLastAppIdleParoledTime;
                if (timeSinceLastParole > mAppIdleParoleIntervalMillis) {
                    if (DEBUG) Slog.d(TAG, "Crossed default parole interval");
                    //【1.1.1】距离上次退出假释模式经过的时间如果超过了 24 hours
                    // 进入假释模式！
                    setAppIdleParoled(true);

                } else {
                    if (DEBUG) Slog.d(TAG, "Not long enough to go to parole");
                    //【*8.1.1】当前不能进入假释模式，继续延迟！
                    postNextParoleTimeout();
                }
            }
        }
    }
```
当距离上次退出假释模式经过的时间如果超过了 24 hours 后，会调用 setAppIdleParoled 方法进入假释模式！

整个流程很简单，不多说了！

# 9 消息：MSG_PAROLE_END_TIMEOUT

该消息用于退出当前的假释状态！

## 9.1 消息触发时机

当 app 进入了假释状态时，会触发 setAppIdleParoled 方法，此时 boolean paroled 为 true！

```java
    void setAppIdleParoled(boolean paroled) {
        synchronized (mLock) {
            if (mAppIdleTempParoled != paroled) {
                //【1】更新 mAppIdleTempParoled
                mAppIdleTempParoled = paroled;
                if (DEBUG) Slog.d(TAG, "Changing paroled to " + mAppIdleTempParoled);
                if (paroled) {
                    //【1.1】调用 postParoleEndTimeout 设置退出假释状态的
                    postParoleEndTimeout();
                } else {
                    mLastAppIdleParoledTime = checkAndGetTimeLocked();
                    postNextParoleTimeout();
                }
                postParoleStateChanged();
            }
        }
    }
```
- **UsageStatsService.postParoleEndTimeout**
```java
    private void postParoleEndTimeout() {
        if (DEBUG) Slog.d(TAG, "Posting MSG_PAROLE_END_TIMEOUT");
        //【1】延迟 10mins，发送 MSG_PAROLE_END_TIMEOUT 消息，退出假释模式！
        mHandler.removeMessages(MSG_PAROLE_END_TIMEOUT);
        mHandler.sendEmptyMessageDelayed(MSG_PAROLE_END_TIMEOUT, mAppIdleParoleDurationMillis);
    }
```
mAppIdleParoleDurationMillis 值为 10mins！

我们来看看 H 是如处理该消息的：
```java
        case MSG_PAROLE_END_TIMEOUT:
            if (DEBUG) Slog.d(TAG, "Ending parole");
            //【1】调用了 setAppIdleParoled 退出假释模式，同时设置下一次进入假释的消息！
            setAppIdleParoled(false);
            break;
```

# 10 消息：MSG_REPORT_CONTENT_PROVIDER_USAGE - OK

## 10.0 消息触发时机

当 ActivityManagerService 触发了 LocalService 下面的方法时：

```java
    private final class LocalService extends UsageStatsManagerInternal {

        @Override
        public void reportContentProviderUsage(String name, String packageName, int userId) {
            SomeArgs args = SomeArgs.obtain();
            args.arg1 = name;
            args.arg2 = packageName;
            args.arg3 = userId;
            //【1】发送 MSG_REPORT_CONTENT_PROVIDER_USAGE 消息
            mHandler.obtainMessage(MSG_REPORT_CONTENT_PROVIDER_USAGE, args)
                    .sendToTarget();
        }    
    }
}
```
会发送 MSG_REPORT_CONTENT_PROVIDER_USAGE 消息！上报 ContentProvider 的使用情况！

```java
        case MSG_REPORT_CONTENT_PROVIDER_USAGE:
            SomeArgs args = (SomeArgs) msg.obj;
            //【10.1】调用了 reportContentProviderUsage 方法！
            reportContentProviderUsage((String) args.arg1, // authority name
                    (String) args.arg2, // package name
                    (int) args.arg3); // userId
            args.recycle();
            break;
```

## 10.1 UsageStatsService.reportContentProviderUsage
```java
    void reportContentProviderUsage(String authority, String providerPkgName, int userId) {
        //【1】获得同步适配器！
        String[] packages = ContentResolver.getSyncAdapterPackagesForAuthorityAsUser(
                authority, userId);
        for (String packageName: packages) {
            //【2】如果同步适配器是系统 package，同时 provider 并没有在相同的 package 中
            // 那么，我们需要强制将同步适配器所在的 package 设置为 active 状态！
            try {
                PackageInfo pi = mPackageManager.getPackageInfoAsUser(
                        packageName, PackageManager.MATCH_SYSTEM_ONLY, userId);
                if (pi == null || pi.applicationInfo == null) {
                    continue;
                }
                if (!packageName.equals(providerPkgName)) {
                    //【*5.1】调用了 forceIdleState 方法！
                    forceIdleState(packageName, userId, false);
                }
            } catch (NameNotFoundException e) {
                // Shouldn't happen
            }
        }
    }
```
对于 forceIdleState 方法，请看 5.1 节！

# 11 消息：MSG_PAROLE_STATE_CHANGED

当应用的假释状态发生变化后，会发送该消息！

## 11.1 消息触发时机

- **UsageStatsService.setChargingState**

当充电状态发生变化时，会触发 postParoleStateChanged 方法！
```java
    void setChargingState(boolean charging) {
        synchronized (mLock) {
            if (mCharging != charging) {
                mCharging = charging;
                //【*11.1.1-common】当充电状态发生变化后，处理假释状态的变化！
                postParoleStateChanged();
            }
        }
    }
```
我们在 UsageStatsService.onBootPhase 方法中会初始化下开机时的充电状态，这时候会调用 setChargingState 方法！

同时，我们在 DeviceStateReceiver 接收者中，当接收到 Intent.ACTION_BATTERY_CHANGED 广播后，我们会调用 setChargingState 方法！


- **UsageStatsService.setAppIdleParoled**

我们在 DeviceStateReceiver 接收者中，当接收到 PowerManager.ACTION_DEVICE_IDLE_MODE_CHANGED 广播后，即 doze 模式发生了变化，我们会调用 onDeviceIdleModeChanged 方法！

```java
    void onDeviceIdleModeChanged() {
        final boolean deviceIdle = mPowerManager.isDeviceIdleMode();
        if (DEBUG) Slog.i(TAG, "DeviceIdleMode changed to " + deviceIdle);
        synchronized (mLock) {
            final long timeSinceLastParole = checkAndGetTimeLocked() - mLastAppIdleParoledTime;
            if (!deviceIdle
                    && timeSinceLastParole >= mAppIdleParoleIntervalMillis) {
                //【1】如果当前不处于 device idle 状态，并且当前距离上次假释状态超过了 24 小时
                // 那么我们会进入假释状态！
                if (DEBUG) Slog.i(TAG, "Bringing idle apps out of inactive state due to deviceIdleMode=false");
                setAppIdleParoled(true);
                
            } else if (deviceIdle) {
                if (DEBUG) Slog.i(TAG, "Device idle, back to prison");
                //【2】如果当前处于 device idle 状态，那么无法进入假释状态！
                setAppIdleParoled(false);
                
            }
        }
    }
```
setAppIdleParoled 方法用于将 app 在 idle 的状态下唤醒进入假释状态，参数 boolean paroled 表示当前是否进入假释状态！

mAppIdleTempParoled 用于保存假释状态！
```java
    void setAppIdleParoled(boolean paroled) {
        synchronized (mLock) {
            if (mAppIdleTempParoled != paroled) {
                //【1】缓存本次假释状态！
                mAppIdleTempParoled = paroled;
                if (DEBUG) Slog.d(TAG, "Changing paroled to " + mAppIdleTempParoled);
                if (paroled) {
                    //【1.1】如果本次是进入假释，设置退出假释的超时消息！
                    postParoleEndTimeout();

                } else {
                    //【1.2】如果本次是退出假释，设置下一次进入假释的消息！
                    // 保存上次假释的最后时间！
                    mLastAppIdleParoledTime = checkAndGetTimeLocked();
                    postNextParoleTimeout();
                }
                
                //【*11.1.1-common】处理假释状态的变化！
                postParoleStateChanged();
            }
        }
    }
```
最后，调用了 postParoleStateChanged 方法！

### 11.1.1 UsageStatsService.postParoleStateChanged

postParoleStateChanged 会发送 MSG_PAROLE_STATE_CHANGED 消息给 H：
```java
    private void postParoleStateChanged() {
        if (DEBUG) Slog.d(TAG, "Posting MSG_PAROLE_STATE_CHANGED");
        mHandler.removeMessages(MSG_PAROLE_STATE_CHANGED);
        mHandler.sendEmptyMessage(MSG_PAROLE_STATE_CHANGED);
    }
```
我们去看看 H 是如何处理 MSG_PAROLE_STATE_CHANGED 消息的：
```java
        case MSG_PAROLE_STATE_CHANGED:
            if (DEBUG) Slog.d(TAG, "Parole state: " + mAppIdleTempParoled
                    + ", Charging state:" + mCharging);
            //【*11.2】处理 app 假释状态改变的消息！
            informParoleStateChanged();
            break;
```

## 11.2 UsageStatsService.informParoleStateChanged

当假释状态改变后，会通知所有的监听者！
```java
    void informParoleStateChanged() {
        final boolean paroled = isParoledOrCharging();
        for (AppIdleStateChangeListener listener : mPackageAccessListeners) {
            listener.onParoleStateChanged(paroled);
        }
    }
```