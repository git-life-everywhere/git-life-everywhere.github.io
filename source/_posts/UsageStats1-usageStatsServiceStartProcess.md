# UsageStats 第 1 篇 - UsageStatsService 的启动
title: UsageStats 第 1 篇 - UsageStatsService 的启动
date: 2017/01/03 20:46:25
categories:
- AndroidFramework源码分析
- UsageStats使用状态管理
tags: UsageStats使用状态管理
---

[toc]

基于 Android7.1.1 源码分析 UsageStatsService 的架构和原理！

# 0 综述

启动 UsageStatsService 服务，是从 SystemServer.startCoreServices 开始！
```java
    private void startCoreServices() {
        mSystemServiceManager.startService(BatteryService.class);

        mSystemServiceManager.startService(UsageStatsService.class);
        mActivityManagerService.setUsageStatsManager(
                LocalServices.getService(UsageStatsManagerInternal.class));

        mWebViewUpdateService = mSystemServiceManager.startService(WebViewUpdateService.class);
    }
```

# 1 new UsageStatsService

```java
    public UsageStatsService(Context context) {
        super(context);
    }
```

UsageStatsService 的构造器很简单，没有过多的数据！

# 2 UsageStatsS.onStart
```java
    public void onStart() {
        //【1】获得一些重要的服务管理对象！
        mAppOps = (AppOpsManager) getContext().getSystemService(Context.APP_OPS_SERVICE);
        mUserManager = (UserManager) getContext().getSystemService(Context.USER_SERVICE);
        mPackageManager = getContext().getPackageManager();
        
        //【*2.1】创建 H 消息处理 Handler！
        mHandler = new H(BackgroundThread.get().getLooper());

        //【2】创建数据目录
        File systemDataDir = new File(Environment.getDataDirectory(), "system");
        mUsageStatsDir = new File(systemDataDir, "usagestats");
        mUsageStatsDir.mkdirs();
        if (!mUsageStatsDir.exists()) {
            throw new IllegalStateException("Usage stats directory does not exist: "
                    + mUsageStatsDir.getAbsolutePath());
        }
        //【*2.2】动态注册一个广播接收者：UserActionsReceiver
        // 监听 Intent.ACTION_USER_STARTED 和 Intent.ACTION_USER_REMOVED 广播；
        IntentFilter filter = new IntentFilter(Intent.ACTION_USER_REMOVED);
        filter.addAction(Intent.ACTION_USER_STARTED);
        getContext().registerReceiverAsUser(new UserActionsReceiver(), UserHandle.ALL, filter,
                null, mHandler);

        //【2.3】动态注册一个广播接收者：PackageReceiver
        // 监听 Intent.ACTION_PACKAGE_ADDED，Intent.ACTION_PACKAGE_REMOVED 和 Intent.ACTION_PACKAGE_CHANGED 广播；
        IntentFilter packageFilter = new IntentFilter();
        packageFilter.addAction(Intent.ACTION_PACKAGE_ADDED);
        packageFilter.addAction(Intent.ACTION_PACKAGE_CHANGED);
        packageFilter.addAction(Intent.ACTION_PACKAGE_REMOVED);
        packageFilter.addDataScheme("package");

        getContext().registerReceiverAsUser(new PackageReceiver(), UserHandle.ALL, packageFilter,
                null, mHandler);

        //【*2.4】通过系统属性 config_enableAutoPowerModes 值来判断是否支持 app idle，注意 doze 模式也是通过这个值判断的
        // 如果支持，创建一个动态注册的接收者：DeviceStateReceiver，用于接收 ACTION_BATTERY_CHANGED
        // ACTION_DISCHARGING，ACTION_DEVICE_IDLE_MODE_CHANGED（doze 模式的广播），监听设备状态！
        mAppIdleEnabled = getContext().getResources().getBoolean(
                com.android.internal.R.bool.config_enableAutoPowerModes);
        if (mAppIdleEnabled) {
            IntentFilter deviceStates = new IntentFilter(Intent.ACTION_BATTERY_CHANGED);
            deviceStates.addAction(BatteryManager.ACTION_DISCHARGING);
            deviceStates.addAction(PowerManager.ACTION_DEVICE_IDLE_MODE_CHANGED);
            getContext().registerReceiver(new DeviceStateReceiver(), deviceStates);
        }

        synchronized (mLock) {
            //【*2.5】清楚被移除的用户信息！
            cleanUpRemovedUsersLocked();
            //【*2.6】创建一个 AppIdleHistory 对象，用于加载和保存 app idle 的历史信息！
            mAppIdleHistory = new AppIdleHistory(SystemClock.elapsedRealtime());
        }

        //【3】记录此时的系统时间到 mSystemTimeSnapshot，距离开始的时间间隔到 mRealTimeSnapshot
        // 后续时间检查时会用到！
        mRealTimeSnapshot = SystemClock.elapsedRealtime();
        mSystemTimeSnapshot = System.currentTimeMillis();

        //【*2.7】注册 LocalService 服务对象，方便进程内部通信
        // 同时将自身注册到 ServiceManager 中，用于跨进程通信！
        publishLocalService(UsageStatsManagerInternal.class, new LocalService());
        publishBinderService(Context.USAGE_STATS_SERVICE, new BinderService());
    }
```
继续分析;


## 2.1 new H

创建了一个 Handler 对，处理 UsageStatsService 中的一些重要的消息，下面我们先开看看有哪些消息！
```java
    class H extends Handler {
        public H(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case MSG_REPORT_EVENT: //【1】处理其他进程传递的 UsageEvents；
                    break;
                case MSG_FLUSH_TO_DISK: //【2】处理其他进程传递的 UsageEvents 时间；
                    break;
                case MSG_REMOVE_USER: //【3】移除某个 user；
                    break;
                case MSG_INFORM_LISTENERS: //【4】
                    break;
                case MSG_FORCE_IDLE_STATE: //【5】设置应用进入 idle 状态；
                    break;
                case MSG_CHECK_IDLE_STATES:  //【6】每隔一段时间检查 idle 状态；
                    break;
                case MSG_ONE_TIME_CHECK_IDLE_STATES: //【7】只检查一次 idle 状态；
                    break;
                case MSG_CHECK_PAROLE_TIMEOUT: //【8】
                    break;
                case MSG_PAROLE_END_TIMEOUT: //【9】
                    break;
                case MSG_REPORT_CONTENT_PROVIDER_USAGE: //【10】记录 content provider 的使用；
                    break;
                case MSG_PAROLE_STATE_CHANGED: //【11】充电状态变化；
                    break;
                default:
                    super.handleMessage(msg);
                    break;
            }
        }
    }
```

在启动的过程中也会发送一些 MSG 给 H 进行处理，对于消息的处理，我们放在第四节分析！

## 2.2 new UserActionsReceiver - 监听用户状态

UserActionsReceiver 接收者用于监听 User 相关的广播！
```java
    private class UserActionsReceiver extends BroadcastReceiver {
        @Override
        public void onReceive(Context context, Intent intent) {
            final int userId = intent.getIntExtra(Intent.EXTRA_USER_HANDLE, -1);
            final String action = intent.getAction();
            if (Intent.ACTION_USER_REMOVED.equals(action)) {
                if (userId >= 0) {
                    //【1】如果是用户被移除的广播，发送 MSG_REMOVE_USER 消息给 H！
                    mHandler.obtainMessage(MSG_REMOVE_USER, userId, 0).sendToTarget();
                }

            } else if (Intent.ACTION_USER_STARTED.equals(action)) {
                if (userId >=0) {
                    //【*2.2.1】如果是用户被启动的广播，调用 postCheckIdleStates 方法，检查 idle 状态信息！
                    postCheckIdleStates(userId);
                }

            }
        }
    }
```
### 2.2.1 UsageStatsService.postCheckIdleStates
```java
    void postCheckIdleStates(int userId) {
        mHandler.sendMessage(mHandler.obtainMessage(MSG_CHECK_IDLE_STATES, userId, 0));
    }
```

如果是用户被启动的广播，调用 postCheckIdleStates 方法，发送 MSG_CHECK_IDLE_STATES 消息给 H，检查 idle 状态信息！

## 2.3 new PackageReceiver - 监听包状态

PackageReceiver 接收者用于监听 package 相关的广播！
```java
    private class PackageReceiver extends BroadcastReceiver {
        @Override
        public void onReceive(Context context, Intent intent) {
            final String action = intent.getAction();
            if (Intent.ACTION_PACKAGE_ADDED.equals(action)
                    || Intent.ACTION_PACKAGE_CHANGED.equals(action)) {
                //【*2.3.1】清除运营商特权应用程序列表！
                clearCarrierPrivilegedApps();
            }

            if ((Intent.ACTION_PACKAGE_REMOVED.equals(action) ||
                    Intent.ACTION_PACKAGE_ADDED.equals(action))
                    && !intent.getBooleanExtra(Intent.EXTRA_REPLACING, false)) {

                //【*2.3.2】清楚 package 的 idle 状态
                clearAppIdleForPackage(intent.getData().getSchemeSpecificPart(),
                        getSendingUserId());
            }
        }
    }
```
如果接收到的广播是：Intent.ACTION_PACKAGE_ADDED 或者 Intent.ACTION_PACKAGE_CHANGED，那就调用 clearCarrierPrivilegedApps 方法，清除运营商特权应用程序列表了；

如果接收到的广播是：Intent.ACTION_PACKAGE_REMOVED(移除应用) 或者 Intent.ACTION_PACKAGE_ADDED，且 Intent.EXTRA_REPLACING 为 false  (新安装的应用)，那就会清楚 package 的 idle 状态！

### 2.3.1 UsageStatsS.clearCarrierPrivilegedApps
清除运营商特权应用程序列表!
```java
    void clearCarrierPrivilegedApps() {
        if (DEBUG) {
            Slog.i(TAG, "Clearing carrier privileged apps list");
        }
        synchronized (mLock) {
            mHaveCarrierPrivilegedApps = false;
            mCarrierPrivilegedApps = null; // Need to be refetched.
        }
    }
```
这里涉及到 2 个变量：

- mHaveCarrierPrivilegedApps 表示是否持有运营商特权应用程序；
- mCarrierPrivilegedApps 是一个 list，保存了运营商特权应用程序；

当然了，有删除也就有添加的方法 fetchCarrierPrivilegedAppsLocked：

```java
    private void fetchCarrierPrivilegedAppsLocked() {
        // 调用了 TelephonyManager 的方法获得运营商特权应用程序列表！
        // 同时设置 mHaveCarrierPrivilegedApps 为 true！
        TelephonyManager telephonyManager =
                getContext().getSystemService(TelephonyManager.class);
        mCarrierPrivilegedApps = telephonyManager.getPackagesWithCarrierPrivileges();
        mHaveCarrierPrivilegedApps = true;
        if (DEBUG) {
            Slog.d(TAG, "apps with carrier privilege " + mCarrierPrivilegedApps);
        }
    }
```
至于运营商特权应用程序列表相关内容，我们后续在看！

### 2.3.2 UsageStatsS.clearAppIdleForPackage

清楚 package 的 idle 状态：
```java
    void clearAppIdleForPackage(String packageName, int userId) {
        synchronized (mLock) {
            //【2.3.2.1】调用 AppIdleHistory.clearUsageLocked 方法！
            mAppIdleHistory.clearUsageLocked(packageName, userId);
        }
    }
```
可以看到，调用的是 AppIdleHistory.clearUsageLocked 方法！

#### 2.3.2.1 AppIdleHistory.clearUsageLocked
```java
    public void clearUsageLocked(String packageName, int userId) {
        //【2.3.2.2】获得指定 userId 下的所有应用的历史信息，然后从中移除 packageName 的信息！
        ArrayMap<String, PackageHistory> userHistory = getUserHistoryLocked(userId);
        userHistory.remove(packageName);
    }
```

#### 2.3.2.2 AppIdleHistory.getUserHistoryLocked
```java
    private ArrayMap<String, PackageHistory> getUserHistoryLocked(int userId) {
        //【1】从 mIdleHistory 中查找 userId 的数据，如果 userHistory 为 null。就从本地数据恢复！
        ArrayMap<String, PackageHistory> userHistory = mIdleHistory.get(userId);
        if (userHistory == null) {
            userHistory = new ArrayMap<>();
            mIdleHistory.put(userId, userHistory);
            //【2.3.2.3】尝试从本地持久化文件中读取数据；
            readAppIdleTimesLocked(userId, userHistory);
        }
        //【2】返回！
        return userHistory;
    }
```
AppIdleHistory 内部有一个 mIdleHistory 集合，用于保存每个 userId 下的所有 package 的空闲历史信息！

#### 2.3.2.3 AppIdleHistory.readAppIdleTimesLocked
```java
    private void readAppIdleTimesLocked(int userId, ArrayMap<String, PackageHistory> userHistory) {
        FileInputStream fis = null;
        try {
            //【1】准备读取 /data/system/users/<userId>/app_idle_stats.xml 文件
            AtomicFile appIdleFile = new AtomicFile(getUserFile(userId));
            fis = appIdleFile.openRead();
            XmlPullParser parser = Xml.newPullParser();
            parser.setInput(fis, StandardCharsets.UTF_8.name());

            int type;
            while ((type = parser.next()) != XmlPullParser.START_TAG
                    && type != XmlPullParser.END_DOCUMENT) {
            }

            if (type != XmlPullParser.START_TAG) {
                Slog.e(TAG, "Unable to read app idle file for user " + userId);
                return;
            }
            if (!parser.getName().equals(TAG_PACKAGES)) { // packages 标签
                return;
            }
            while ((type = parser.next()) != XmlPullParser.END_DOCUMENT) {
                if (type == XmlPullParser.START_TAG) {
                    final String name = parser.getName();
                    //【2】解析 package 标签和其属性！
                    if (name.equals(TAG_PACKAGE)) { // 解析 “package” 标签
                        final String packageName = parser.getAttributeValue(null, ATTR_NAME);  // 解析 “name” 属性
                        
                        //【3】创建 PackageHistory 对象，封装解析到的信息；
                        PackageHistory packageHistory = new PackageHistory();
                        
                        packageHistory.lastUsedElapsedTime =   // 解析 “screenIdleTime” 属性
                                Long.parseLong(parser.getAttributeValue(null, ATTR_ELAPSED_IDLE));
                       
                        packageHistory.lastUsedScreenTime =    // 解析 “elapsedIdleTime” 属性
                                Long.parseLong(parser.getAttributeValue(null, ATTR_SCREEN_IDLE));
                        //【4】添加到 userHistory 中，最后返回！
                        userHistory.put(packageName, packageHistory);
                    }
                }
            }
        } catch (IOException | XmlPullParserException e) {
            Slog.e(TAG, "Unable to read app idle file for user " + userId);
        } finally {
            IoUtils.closeQuietly(fis);
        }
    }
```
我们来看看 /data/system/users/0/app_idle_stats.xml 文件中的主要内容：
```java
<?xml version='1.0' encoding='utf-8' standalone='yes' ?>
<packages>
    <package name="com.github.shadowsocks" elapsedIdleTime="6836535861" screenIdleTime="2173999944" />
</packages>
```


getUserFile 方法返回的是：`/data/system/users/<userId>/app_idle_stats.xml` 文件对象：
```java
    static final String APP_IDLE_FILENAME = "app_idle_stats.xml";

    private File getUserFile(int userId) {
        return new File(new File(new File(mStorageDir, "users"),
                Integer.toString(userId)), APP_IDLE_FILENAME);
    }
```

#####2.3.2.3.1 new PackageHistory
创建 PackageHistory 对象！
```java
    private static class PackageHistory {
        final byte[] recent = new byte[HISTORY_SIZE];
        long lastUsedElapsedTime;
        long lastUsedScreenTime;
    }

```

## 2.4 new DeviceStateReceiver - 监听设备状态

DeviceStateReceiver 监听设备状态的变化！
```java

    private class DeviceStateReceiver extends BroadcastReceiver {
        @Override
        public void onReceive(Context context, Intent intent) {
            final String action = intent.getAction();
            if (Intent.ACTION_BATTERY_CHANGED.equals(action)) {
                //【2.4.1】设置充电状态！
                setChargingState(intent.getIntExtra("plugged", 0) != 0);
                
            } else if (PowerManager.ACTION_DEVICE_IDLE_MODE_CHANGED.equals(action)) {
                //【2.4.2】处理 device idle (doze)模式变化！
                onDeviceIdleModeChanged();

            }
        }
    }
```

- 如果广播是 Intent.ACTION_BATTERY_CHANGED，说明此时正在充电，那么会调用 setChargingState 设置充电状态；
- 如果广播是 PowerManager.ACTION_DEVICE_IDLE_MODE_CHANGED，说明此时 device idle 模式的状态发生了变化；

### 2.4.1 UsageStatsS.setChargingState - 充电状态变化

```java
    void setChargingState(boolean charging) {
        synchronized (mLock) {
            if (mCharging != charging) {
                // 更新 mCharging！
                mCharging = charging;
                //【2.4.1.1】调用 postParoleStateChanged 方法！
                postParoleStateChanged();
            }
        }
    }
```
mCharging 表示当前设备是否正在充电，可以看到，只有当设备在未充电和充电状态之间变化！

#### 2.4.1.1 UsageStatsS.postParoleStateChanged
```java
    private void postParoleStateChanged() {
        if (DEBUG) Slog.d(TAG, "Posting MSG_PAROLE_STATE_CHANGED");
        //【4.11】发送 MSG_PAROLE_STATE_CHANGED 消息给 H!
        mHandler.removeMessages(MSG_PAROLE_STATE_CHANGED);
        mHandler.sendEmptyMessage(MSG_PAROLE_STATE_CHANGED);
    }
```
发送 MSG_PAROLE_STATE_CHANGED 消息给 H！

### 2.4.2 UsageStatsS.onDeviceIdleModeChanged - doze 状态变化

当 device idle (doze)模式发生了变化后，onDeviceIdleModeChanged 方法会被触发：
```java
    void onDeviceIdleModeChanged() {
        //【1】调用 PowerManager.isDeviceIdleMode 方法，判断是否进入了 doze 模式！
        final boolean deviceIdle = mPowerManager.isDeviceIdleMode();
        
        if (DEBUG) Slog.i(TAG, "DeviceIdleMode changed to " + deviceIdle);
        
        synchronized (mLock) {
            //【2】计算距离里上一次的应用假释时间，已经过去的时间！
            final long timeSinceLastParole = checkAndGetTimeLocked() - mLastAppIdleParoledTime;
            if (!deviceIdle
                    && timeSinceLastParole >= mAppIdleParoleIntervalMillis) {
                if (DEBUG) Slog.i(TAG, "Bringing idle apps out of inactive state due to deviceIdleMode=false");
                //【2.1】如果已经退出了 device idle 模式，并且距离上一次的应用假释时间已经超过了 
                // mAppIdleParoleIntervalMillis，那么我们就进入假释状态！
                setAppIdleParoled(true);

            } else if (deviceIdle) {
                if (DEBUG) Slog.i(TAG, "Device idle, back to prison");
                //【2.2】如果当前处于 device idle 状态，那么不允许应用假释；
                setAppIdleParoled(false);

            }
        }
    }
```

#### 2.4.2.1 UsageStatsS.checkAndGetTimeLocked

checkAndGetTimeLocked 方法用于计算当前的时间！

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
    
## 2.5 UsageStatsS.cleanUpRemovedUsersLocked - 删除被移除的 User 使用数据

删除已经被移除的 User 的使用数据！
```java
    private void cleanUpRemovedUsersLocked() {
        //【1】获得所有的 user 信息！
        final List<UserInfo> users = mUserManager.getUsers(true);
        if (users == null || users.size() == 0) {
            throw new IllegalStateException("There can't be no users");
        }
        //【2】如果 /data/system/usagestats 目录下没有任何文件，不处理！
        ArraySet<String> toDelete = new ArraySet<>();
        String[] fileNames = mUsageStatsDir.list();
        if (fileNames == null) {
            // No users to delete.
            return;
        }
        //【3】去除那些存在的 user 的使用信息！
        toDelete.addAll(Arrays.asList(fileNames));

        final int userCount = users.size();
        for (int i = 0; i < userCount; i++) {
            final UserInfo userInfo = users.get(i);
            toDelete.remove(Integer.toString(userInfo.id));
        }
        //【4】移除剩下的没用的 user 信息；
        final int deleteCount = toDelete.size();
        for (int i = 0; i < deleteCount; i++) {
            // 递归删除！
            deleteRecursively(new File(mUsageStatsDir, toDelete.valueAt(i)));
        }
    }

```

## 2.6 new AppIdleHistory - 管理 App Idle 信息

创建一个 AppIdleHistory 对象，保存 app idle 相关的状态和信息！

```java
    AppIdleHistory(long elapsedRealtime) {
        this(Environment.getDataSystemDirectory(), elapsedRealtime);
    }

    @VisibleForTesting
    AppIdleHistory(File storageDir, long elapsedRealtime) {
        //【1】可以看到，初始化时候 mElapsedSnapshot 等于 mScreenOnSnapshot
        mElapsedSnapshot = elapsedRealtime;
        mScreenOnSnapshot = elapsedRealtime;
        mStorageDir = storageDir;
        //【×2.6.1】读取亮屏时间信息！
        readScreenOnTimeLocked();
    }
```
mStorageDir 指向 /data/system 目录！

### 2.6.1 AppIdleHistory.readScreenOnTimeLocked
```java
    private void readScreenOnTimeLocked() {
        //【1】获得 /data/system/screen_on_time 文件
        File screenOnTimeFile = getScreenOnTimeFile();
        if (screenOnTimeFile.exists()) {
            try {
                BufferedReader reader = new BufferedReader(new FileReader(screenOnTimeFile));
                //【2】读取 mScreenOnDuration 和 mElapsedDuration 时间值！
                mScreenOnDuration  = Long.parseLong(reader.readLine());
                mElapsedDuration = Long.parseLong(reader.readLine());
                reader.close();
            } catch (IOException | NumberFormatException e) {
            }
        } else {
            writeScreenOnTimeLocked();
        }
    }
```
整个流程很简单，不多说了！

## 2.7 publish LocalService

LocalService 用于系统进程中服务间的相互通信，本地服务实现主要由ActivityManagerService 使用。ActivityManagerService 调用这些方法的时会持有自身的锁，不应该在这些方法中执行任何 IO 工作或其他长时间运行的任务。
```java
    private final class LocalService extends UsageStatsManagerInternal {
        //【1】向 UsageStatsService 发送应用的使用信息 UsageEvents；
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
            //【1.1】发送 MSG_REPORT_EVENT 给 H；
            mHandler.obtainMessage(MSG_REPORT_EVENT, userId, 0, event).sendToTarget();
        }

        @Override
        public void reportEvent(String packageName, int userId, int eventType) {
            if (packageName == null) {
                Slog.w(TAG, "Event reported without a package name");
                return;
            }
            UsageEvents.Event event = new UsageEvents.Event();
            event.mPackage = packageName;
            event.mTimeStamp = SystemClock.elapsedRealtime();
            event.mEventType = eventType;
            mHandler.obtainMessage(MSG_REPORT_EVENT, userId, 0, event).sendToTarget();
        }

        //【2】向 UsageStatsService 发送配置的使用信息 UsageEvents；
        @Override
        public void reportConfigurationChange(Configuration config, int userId) {
            if (config == null) {
                Slog.w(TAG, "Configuration event reported with a null config");
                return;
            }
            UsageEvents.Event event = new UsageEvents.Event();
            event.mPackage = "android";
            event.mTimeStamp = SystemClock.elapsedRealtime();
            event.mEventType = UsageEvents.Event.CONFIGURATION_CHANGE;
            event.mConfiguration = new Configuration(config);
            //【2.1】发送 MSG_REPORT_EVENT 给 H；
            mHandler.obtainMessage(MSG_REPORT_EVENT, userId, 0, event).sendToTarget();
        }

        //【3】向 UsageStatsService 发送 Shortcut 的使用信息 UsageEvents；
        @Override
        public void reportShortcutUsage(String packageName, String shortcutId, int userId) {
            if (packageName == null || shortcutId == null) {
                Slog.w(TAG, "Event reported without a package name or a shortcut ID");
                return;
            }
            UsageEvents.Event event = new UsageEvents.Event();
            event.mPackage = packageName.intern();
            event.mShortcutId = shortcutId.intern();
            event.mTimeStamp = SystemClock.elapsedRealtime();
            event.mEventType = Event.SHORTCUT_INVOCATION;
            //【3.1】发送 MSG_REPORT_EVENT 给 H；
            mHandler.obtainMessage(MSG_REPORT_EVENT, userId, 0, event).sendToTarget();
        }
        
        //【4】向 UsageStatsService 发送 ContentProvider 的使用信息；
        @Override
        public void reportContentProviderUsage(String name, String packageName, int userId) {
            SomeArgs args = SomeArgs.obtain();
            args.arg1 = name;
            args.arg2 = packageName;
            args.arg3 = userId;
            //【4.1】发送 MSG_REPORT_CONTENT_PROVIDER_USAGE 给 H；
            mHandler.obtainMessage(MSG_REPORT_CONTENT_PROVIDER_USAGE, args)
                    .sendToTarget();
        }

        //【5】判断应用是否处于 idle 状态；
        @Override
        public boolean isAppIdle(String packageName, int uidForAppId, int userId) {
            return UsageStatsService.this.isAppIdleFiltered(packageName, uidForAppId, userId,
                    SystemClock.elapsedRealtime());
        }

        //【6】获得指定 userId 下的处于 idle 状态的 uid；
        @Override
        public int[] getIdleUidsForUser(int userId) {
            return UsageStatsService.this.getIdleUidsForUser(userId);
        }

        //【7】获得指定 userId 下的处于 idle 状态的 uid；
        @Override
        public boolean isAppIdleParoleOn() {
            return isParoledOrCharging();
        }

        //【8】关机时调用！
        @Override
        public void prepareShutdown() {
            shutdown();
        }
        //【9】添加和移除 app idle 状态改变监听器
        @Override
        public void addAppIdleStateChangeListener(AppIdleStateChangeListener listener) {
            UsageStatsService.this.addListener(listener);
            listener.onParoleStateChanged(isAppIdleParoleOn());
        }

        @Override
        public void removeAppIdleStateChangeListener(
                AppIdleStateChangeListener listener) {
            UsageStatsService.this.removeListener(listener);
        }

        @Override
        public byte[] getBackupPayload(int user, String key) {
            // Check to ensure that only user 0's data is b/r for now
            if (user == UserHandle.USER_SYSTEM) {
                final UserUsageStatsService userStats =
                        getUserDataAndInitializeIfNeededLocked(user, checkAndGetTimeLocked());
                return userStats.getBackupPayload(key);
            } else {
                return null;
            }
        }

        @Override
        public void applyRestoredPayload(int user, String key, byte[] payload) {
            if (user == UserHandle.USER_SYSTEM) {
                final UserUsageStatsService userStats =
                        getUserDataAndInitializeIfNeededLocked(user, checkAndGetTimeLocked());
                userStats.applyRestoredPayload(key, payload);
            }
        }
    }
```
关于 LocalService 中的方法的触发，我们后面再看！


## 2.8 publish BinderService

BinderService 的所用是跨进程通信！

```java
    private final class BinderService extends IUsageStatsManager.Stub {
        //【1】判断调用者是否有 PACKAGE_USAGE_STATS 相关的权限！
        private boolean hasPermission(String callingPackage) {
            final int callingUid = Binder.getCallingUid();
            if (callingUid == Process.SYSTEM_UID) {
                return true;
            }
            final int mode = mAppOps.checkOp(AppOpsManager.OP_GET_USAGE_STATS,
                    callingUid, callingPackage);
            if (mode == AppOpsManager.MODE_DEFAULT) {
                // The default behavior here is to check if PackageManager has given the app
                // permission.
                return getContext().checkCallingPermission(Manifest.permission.PACKAGE_USAGE_STATS)
                        == PackageManager.PERMISSION_GRANTED;
            }
            return mode == AppOpsManager.MODE_ALLOWED;
        }
 
        //【2】查询应用的使用状态！
        @Override
        public ParceledListSlice<UsageStats> queryUsageStats(int bucketType, long beginTime,
                long endTime, String callingPackage) {
            if (!hasPermission(callingPackage)) {
                return null;
            }

            final int userId = UserHandle.getCallingUserId();
            final long token = Binder.clearCallingIdentity();
            try {
                final List<UsageStats> results = UsageStatsService.this.queryUsageStats(
                        userId, bucketType, beginTime, endTime);
                if (results != null) {
                    return new ParceledListSlice<>(results);
                }
            } finally {
                Binder.restoreCallingIdentity(token);
            }
            return null;
        }

        //【3】查询配置的使用状态！
        @Override
        public ParceledListSlice<ConfigurationStats> queryConfigurationStats(int bucketType,
                long beginTime, long endTime, String callingPackage) throws RemoteException {
            if (!hasPermission(callingPackage)) {
                return null;
            }

            final int userId = UserHandle.getCallingUserId();
            final long token = Binder.clearCallingIdentity();
            try {
                final List<ConfigurationStats> results =
                        UsageStatsService.this.queryConfigurationStats(userId, bucketType,
                                beginTime, endTime);
                if (results != null) {
                    return new ParceledListSlice<>(results);
                }
            } finally {
                Binder.restoreCallingIdentity(token);
            }
            return null;
        }

        //【4】查询应用的使用事件信息！
        @Override
        public UsageEvents queryEvents(long beginTime, long endTime, String callingPackage) {
            if (!hasPermission(callingPackage)) {
                return null;
            }

            final int userId = UserHandle.getCallingUserId();
            final long token = Binder.clearCallingIdentity();
            try {
                return UsageStatsService.this.queryEvents(userId, beginTime, endTime);
            } finally {
                Binder.restoreCallingIdentity(token);
            }
        }

        //【5】判断应用是否处于 inactive 状态！
        @Override
        public boolean isAppInactive(String packageName, int userId) {
            try {
                userId = ActivityManagerNative.getDefault().handleIncomingUser(Binder.getCallingPid(),
                        Binder.getCallingUid(), userId, false, true, "isAppInactive", null);
            } catch (RemoteException re) {
                throw re.rethrowFromSystemServer();
            }
            final long token = Binder.clearCallingIdentity();
            try {
                return UsageStatsService.this.isAppIdleFilteredOrParoled(packageName, userId,
                        SystemClock.elapsedRealtime());
            } finally {
                Binder.restoreCallingIdentity(token);
            }
        }
        
        //【5】判断应用是否处于 inactive 状态！
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
                UsageStatsService.this.setAppIdle(packageName, idle, userId);
            } finally {
                Binder.restoreCallingIdentity(token);
            }
        }
        
        //【6】将应用添加到 doze 模式临时白名单中！
        @Override
        public void whitelistAppTemporarily(String packageName, long duration, int userId)
                throws RemoteException {
            StringBuilder reason = new StringBuilder(32);
            reason.append("from:");
            UserHandle.formatUid(reason, Binder.getCallingUid());
            mDeviceIdleController.addPowerSaveTempWhitelistApp(packageName, duration, userId,
                    reason.toString());
        }

        //【7】运营商特权应用变化！
        @Override
        public void onCarrierPrivilegedAppsChanged() {
            if (DEBUG) {
                Slog.i(TAG, "Carrier privileged apps changed");
            }
            getContext().enforceCallingOrSelfPermission(
                    android.Manifest.permission.BIND_CARRIER_SERVICES,
                    "onCarrierPrivilegedAppsChanged can only be called by privileged apps.");
            UsageStatsService.this.clearCarrierPrivilegedApps();
        }

        //【8】用于 dumpsys！
        @Override
        protected void dump(FileDescriptor fd, PrintWriter pw, String[] args) {
            if (getContext().checkCallingOrSelfPermission(android.Manifest.permission.DUMP)
                    != PackageManager.PERMISSION_GRANTED) {
                pw.println("Permission Denial: can't dump UsageStats from pid="
                        + Binder.getCallingPid() + ", uid=" + Binder.getCallingUid()
                        + " without permission " + android.Manifest.permission.DUMP);
                return;
            }
            UsageStatsService.this.dump(args, pw);
        }
    }

```
关于 BinderService 中的方法的触发，我们后面再看！

# 3 UsageStatsS.onBootPhase
```java
    @Override
    public void onBootPhase(int phase) {
        if (phase == PHASE_SYSTEM_SERVICES_READY) {
            //【*3.1】创建一个 SettingsObserver， 监听数据库变化！
            SettingsObserver settingsObserver = new SettingsObserver(mHandler);
            settingsObserver.registerObserver();
            settingsObserver.updateSettings();

            //【2】获得系统中的一些其他重要服务管理对象：DeviceIdleController，
            // BatteryStats，DisplayManager 和 PowerManager！
            mAppWidgetManager = getContext().getSystemService(AppWidgetManager.class);
            mDeviceIdleController = IDeviceIdleController.Stub.asInterface(
                    ServiceManager.getService(Context.DEVICE_IDLE_CONTROLLER));
            mBatteryStats = IBatteryStats.Stub.asInterface(
                    ServiceManager.getService(BatteryStats.SERVICE_NAME));
            mDisplayManager = (DisplayManager) getContext().getSystemService(
                    Context.DISPLAY_SERVICE);
            mPowerManager = getContext().getSystemService(PowerManager.class);

            //【*3.2】注册 DisplayListener
            mDisplayManager.registerDisplayListener(mDisplayListener, mHandler);
            synchronized (mLock) {
                //【*3.2.1】初始化 AppIdleHistory 的亮屏信息！
                mAppIdleHistory.updateDisplayLocked(isDisplayOn(), SystemClock.elapsedRealtime());
            }

            //【*3.4】mPendingOneTimeCheckIdleStates 为 true 表示正在等待查询 idle 状态！
            // 那么就会再次调用 postOneTimeCheckIdleStates 方法！
            if (mPendingOneTimeCheckIdleStates) {
                postOneTimeCheckIdleStates();
            }

            mSystemServicesReady = true;
        } else if (phase == PHASE_BOOT_COMPLETED) {
            //【*2.4.1】初始化充电状态信息；
            setChargingState(getContext().getSystemService(BatteryManager.class).isCharging());
        }
    }
```
onBootPhase 方法会在两个阶段下调用：

- **PHASE_SYSTEM_SERVICES_READY**：此时系统服务已经都启动了；
- **PHASE_BOOT_COMPLETED**：此时设备重启完成了；

## 3.1 new SettingsObserver - 监听数据库变化

这里会创建一个 SettingsObserver 观察者，监听数据表变化！
```java
    private class SettingsObserver extends ContentObserver {

        private final KeyValueListParser mParser = new KeyValueListParser(',');

        SettingsObserver(Handler handler) {
            super(handler);
        }
        ... ... ...
}
```
### 3.1.1 SettingsObserver.registerObserver

监听的 Settings 表单是 app_idle_constants！
```java
        void registerObserver() {
            getContext().getContentResolver().registerContentObserver(Settings.Global.getUriFor(
                    Settings.Global.APP_IDLE_CONSTANTS), false, this);
        }
```
### 3.1.2 SettingsObserver.onChange

当数据库有变化后，会触发 onChange 方法：
```java
        @Override
        public void onChange(boolean selfChange) {
            //【3.1.3】初始化数据！
            updateSettings();
            //【3.4】调用 postOneTimeCheckIdleStates 方法进行一次 idle 检查！
            postOneTimeCheckIdleStates();
        }
```
### 3.1.3 SettingsObserver.updateSettings

```java
        void updateSettings() {
            synchronized (mLock) {
                try {
                    mParser.setString(Settings.Global.getString(getContext().getContentResolver(),
                            Settings.Global.APP_IDLE_CONSTANTS));
                } catch (IllegalArgumentException e) {
                    Slog.e(TAG, "Bad value for app idle settings: " + e.getMessage());
                }

                // 值为 12 hours；
                mAppIdleScreenThresholdMillis = mParser.getLong(KEY_IDLE_DURATION,
                       COMPRESS_TIME ? ONE_MINUTE * 4 : 12 * 60 * ONE_MINUTE);
                // 值为 48 hours；
                mAppIdleWallclockThresholdMillis = mParser.getLong(KEY_WALLCLOCK_THRESHOLD,
                        COMPRESS_TIME ? ONE_MINUTE * 8 : 2L * 24 * 60 * ONE_MINUTE);
                // 值为 8 hours；
                mCheckIdleIntervalMillis = Math.min(mAppIdleScreenThresholdMillis / 4,
                        COMPRESS_TIME ? ONE_MINUTE : 8 * 60 * ONE_MINUTE);
                // 置为 24 hours；
                mAppIdleParoleIntervalMillis = mParser.getLong(KEY_PAROLE_INTERVAL,
                        COMPRESS_TIME ? ONE_MINUTE * 10 : 24 * 60 * ONE_MINUTE);
                // 置为 10 mins；
                mAppIdleParoleDurationMillis = mParser.getLong(KEY_PAROLE_DURATION,
                        COMPRESS_TIME ? ONE_MINUTE : 10 * ONE_MINUTE);

                //【3.1.3.1】设置阈值到 AppIdleHistory 中！
                mAppIdleHistory.setThresholds(mAppIdleWallclockThresholdMillis,
                        mAppIdleScreenThresholdMillis);
            }
        }
    }
```
这里涉及到几个重要的时间变量：

- **mAppIdleScreenThresholdMillis**：12 hours，是应用是否进入 idle 状态的临界值！

</br>

- **mAppIdleWallclockThresholdMillis**：48 hours，是应用是否进入 idle 状态的临界值！

</br>

- **mCheckIdleIntervalMillis**：表示执行 check idle 操作的时间间隔，8 hours！

</br>

- **mAppIdleParoleIntervalMillis**：相邻 2 次进入假释状态的时间间隔，24 hours！

</br>

- **mAppIdleParoleDurationMillis**：假释状态的持续时间，10mins！


这里的 COMPRESS_TIME 的值恒定为 false！

```java
    static final boolean COMPRESS_TIME = false;
```
#### 3.1.3.1 AppIdleHistory.setThresholds

设置阈值信息，该阈值会用于判断应用是否处于 idle 状态！
```java
    public void setThresholds(long elapsedTimeThreshold, long screenOnTimeThreshold) {
        mElapsedTimeThreshold = elapsedTimeThreshold;
        mScreenOnTimeThreshold = screenOnTimeThreshold;
    }
```

我们来解释下这两个阈值的作用：


## 3.2 register DisplayListener - 监听屏幕状态

UsageStatsService 内部有一个 DisplayListener 实例：mDisplayListener，专门用于监听屏幕的状态，然后触发 AppIdleHistory 的更新！

```java
    private final DisplayManager.DisplayListener mDisplayListener
            = new DisplayManager.DisplayListener() {
        ... ... ...
        @Override public void onDisplayChanged(int displayId) {
            if (displayId == Display.DEFAULT_DISPLAY) {
                final boolean displayOn = isDisplayOn();
                synchronized (UsageStatsService.this.mLock) {
                    //【3.3】更新 AppIdleHistory 中的屏幕信息！
                    mAppIdleHistory.updateDisplayLocked(displayOn, SystemClock.elapsedRealtime());
                }
            }
        }
    };

```
当屏幕的状态发生变化后，会触发 DisplayListener.onDisplayChanged 方法，紧接着触发 AppIdleHistory.updateDisplayLocked 方法！

### 3.2.1 AppIdleHistory.updateDisplayLocked

updateDisplayLocked 方法用于更新屏幕状态信息！

```java
    public void updateDisplayLocked(boolean screenOn, long elapsedRealtime) {
        //【1】如果相等，无需更新！
        if (screenOn == mScreenOn) return;
        //【2】更新 mScreenOn 的值；
        mScreenOn = screenOn;
        //【3】根据当前屏幕的状态，更新不同的值；
        if (mScreenOn) {
            //【3.1】如果此时亮屏，记录亮屏时间点！
            mScreenOnSnapshot = elapsedRealtime;

        } else {
            //【3.2】如果此时灭屏，记录本次亮屏时长，累加到 mScreenOnDuration；
            mScreenOnDuration += elapsedRealtime - mScreenOnSnapshot; 
            // 计算距离上次灭屏时间点的时间间隔，累加到 mElapsedDuration；
            mElapsedDuration += elapsedRealtime - mElapsedSnapshot;
            // 记录最新灭屏时间点；
            mElapsedSnapshot = elapsedRealtime;
        }
    }
```
通过 isDisplayOn 方法，来判断当前是亮屏还是熄屏：
```java
    private boolean isDisplayOn() {
        return mDisplayManager
                .getDisplay(Display.DEFAULT_DISPLAY).getState() == Display.STATE_ON;
    }
```
AppIdleHistory 内部是有一个变量 mScreenOn，保存屏幕的状态信息，默认为 false：

mScreenOnSnapshot 用于每次亮屏的时间点，mElapsedSnapshot 用于记录每次灭屏的时间点；

mScreenOnDuration 用于累计所有亮屏的总时长，mElapsedDuration 用于了累计所有相邻两次灭屏的总时长！

如果是此时是亮屏的话，会将时间记录到 mScreenOnSnapshot 中！

## 3.4 UsageStatsS.postOneTimeCheckIdleStates

```java
    void postOneTimeCheckIdleStates() {
        if (mDeviceIdleController == null) {
            mPendingOneTimeCheckIdleStates = true;

        } else {
            //【4.7】发送 MSG_ONE_TIME_CHECK_IDLE_STATES 消息给 H，触发 check idle 操作！
            mHandler.sendEmptyMessage(MSG_ONE_TIME_CHECK_IDLE_STATES);
            mPendingOneTimeCheckIdleStates = false;
        }
    }
```
- 如果 mDeviceIdleController 为 null，说明 UsageStatsService 还没有启动完成，这里会先将 mPendingOneTimeCheckIdleStates 置为 true！等到启动完成后会判断该变量的值，如果为 true，那就会继续调用 postOneTimeCheckIdleStates 方法！

- 如果 mDeviceIdleController 不为 null，发送 MSG_ONE_TIME_CHECK_IDLE_STATES 消息给 H，触发 check idle 操作！同时设置 mPendingOneTimeCheckIdleStates 为 false；


