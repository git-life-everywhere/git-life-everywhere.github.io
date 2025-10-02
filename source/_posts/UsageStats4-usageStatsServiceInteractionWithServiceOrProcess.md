# UsageStats 第 4 篇 - UsageStatsService 和其他服务和进程的交互
title: UsageStats 第 4 篇 - UsageStatsService 和其他服务和进程的交互
date: 2017/08/03 20:46:25
categories:
- AndroidFramework源码分析
- UsageStats使用状态管理
tags: UsageStats使用状态管理
---

[toc]

基于 Android7.1.1 源码分析 UsageStatsService 的架构和原理！

# 0 综述

本文主要分析 UsageStatsService 和其他服务 / 进程交互！

我们知道 UsageStatsService 内部有两个类 LocalService 和 BinderService，他们是 UsageStatsService 提供給其他服务或进程的通信接口类！

- **BinderService**

BinderService 主要用于跨进程调用：

```java
private final class BinderService extends IUsageStatsManager.Stub {...}
```

BinderService 继承了 IUsageStatsManager.Stub，作为服务端桩对象，注册进 ServiceManager！

我们在其他进程中可以通过 UsageStatsManager 来获得客户端代理对象！


- **LocalService**

LocalService 用于系统进程中的其他服务调用，比如 ActivityManagerService，NotificationManagerService 等服务，下面是一些主要接口的调用逻辑：

```java
LocalService.reportEvent                 -> UsageStatsService.H.MSG_REPORT_EVENT
LocalService.reportConfigurationChange   -> UsageStatsService.H.MSG_REPORT_EVENT
LocalService.reportShortcutUsage         -> UsageStatsService.H.MSG_REPORT_EVENT
LocalService.reportContentProviderUsage  -> UsageStatsService.H.MSG_REPORT_CONTENT_PROVIDER_USAGE

LocalService.isAppIdle                   -> UsageStatsService.isAppIdleFiltered
LocalService.getIdleUidsForUser          -> UsageStatsService.getIdleUidsForUse
LocalService.isAppIdleParoleOn           -> UsageStatsService.isParoledOrCharging


LocalService.addAppIdleStateChangeListener  -> UsageStatsService.addListener
```

我们重点关注 LocalService 的逻辑调用！


# 1 和 UsageStatsService 通信的服务

- **ActivityManagerService**

```java
        mActivityManagerService.setUsageStatsManager(
                LocalServices.getService(UsageStatsManagerInternal.class));
```

- **NotificationManagerService**

```java
    @Override
    public void onStart() {
        ... ... ...
        mAppUsageStats = LocalServices.getService(UsageStatsManagerInternal.class);
        ... ... ..
    }
```

- **NetworkPolicyManagerService**

```java
    public void systemReady() {
        Trace.traceBegin(Trace.TRACE_TAG_NETWORK, "systemReady");
        try {
            if (!isBandwidthControlEnabled()) {
                Slog.w(TAG, "bandwidth controls disabled, unable to enforce policy");
                return;
            }

            mUsageStats = LocalServices.getService(UsageStatsManagerInternal.class);
        ... ... ...
    }
```

- **AppIdleController**

```java
    private AppIdleController(JobSchedulerService service, Context context, Object lock) {
        super(service, context, lock);
        mJobSchedulerService = service;
        mUsageStatsInternal = LocalServices.getService(UsageStatsManagerInternal.class);
        mAppIdleParoleOn = true;
        mUsageStatsInternal.addAppIdleStateChangeListener(new AppIdleStateChangeListener());
    }
```

主要有以上的服务！


# 2 ActivityManagerService

ActivityManagerService 和 UsageStatsService 交互可算是重头戏！


## 2.1 updateUsageStats

updateUsageStats 方法用于更新 Activity 的使用信息！


### 2.1.1 调用时机

我们来看看调用时机：

- **ActivityStack.startPausingLocked**

当 activty 进入 paused 状态的时候，会触发 startPausingLocked 方法！

```java
    final boolean startPausingLocked(boolean userLeaving, boolean uiSleeping,
            ActivityRecord resuming, boolean dontWait) {
        ... ... ...

        if (prev.app != null && prev.app.thread != null) {
            if (DEBUG_PAUSE) Slog.v(TAG_PAUSE, "Enqueueing pending pause: " + prev);
            try {
                EventLog.writeEvent(EventLogTags.AM_PAUSE_ACTIVITY,
                        prev.userId, System.identityHashCode(prev),
                        prev.shortComponentName);
                //【2.1.2】记录 activity 组件的 UsageEvents.Event.MOVE_TO_BACKGROUND 事件！
                mService.updateUsageStats(prev, false);
                //【1】拉起 activity 的 onPause 方法！
                prev.app.thread.schedulePauseActivity(prev.appToken, prev.finishing,
                        userLeaving, prev.configChangeFlags, dontWait);
            } catch (Exception e) {
                // Ignore exception, if process died other code will cleanup.
                Slog.w(TAG, "Exception thrown during pause", e);
                mPausingActivity = null;
                mLastPausedActivity = null;
                mLastNoHistoryActivity = null;
            }
        } else {
            mPausingActivity = null;
            mLastPausedActivity = null;
            mLastNoHistoryActivity = null;
        }
        ... ... ...
    }
```

- **ActivityStackSupervisor.reportResumedActivityLocked**

当 activty 进入 resume 状态的时候，会触发 reportResumedActivityLocked 方法！
```java
    boolean reportResumedActivityLocked(ActivityRecord r) {
        final ActivityStack stack = r.task.stack;
        if (isFocusedStack(stack)) {
            //【2.1.2】记录 activity 组件的 UsageEvents.Event.MOVE_TO_FOREGROUND 事件！
            mService.updateUsageStats(r, true);
        }
        if (allResumedActivitiesComplete()) {
            ensureActivitiesVisibleLocked(null, 0, !PRESERVE_WINDOWS);
            mWindowManager.executeAppTransition();
            return true;
        }
        return false;
    }
```

- **ActivityStack.handleAppDiedLocked**

当 app process 死亡后，会触发 ActivityStack.handleAppDiedLocked 方法！

```java
    boolean handleAppDiedLocked(ProcessRecord app) {
        if (mPausingActivity != null && mPausingActivity.app == app) {
            if (DEBUG_PAUSE || DEBUG_CLEANUP) Slog.v(TAG_PAUSE,
                    "App died while pausing: " + mPausingActivity);
            mPausingActivity = null;
        }
        if (mLastPausedActivity != null && mLastPausedActivity.app == app) {
            mLastPausedActivity = null;
            mLastNoHistoryActivity = null;
        }
        //【1】调用自身的 removeHistoryRecordsForAppLocked 方法！
        return removeHistoryRecordsForAppLocked(app);
    }
```
进入 removeHistoryRecordsForAppLocked 方法！

```java
   boolean removeHistoryRecordsForAppLocked(ProcessRecord app) {
        ... ... ...
        for (int taskNdx = mTaskHistory.size() - 1; taskNdx >= 0; --taskNdx) {
            final ArrayList<ActivityRecord> activities = mTaskHistory.get(taskNdx).mActivities;
            for (int activityNdx = activities.size() - 1; activityNdx >= 0; --activityNdx) {
                final ActivityRecord r = activities.get(activityNdx);
                --i;
                if (DEBUG_CLEANUP) Slog.v(TAG_CLEANUP,
                        "Record #" + i + " " + r + ": app=" + r.app);
                if (r.app == app) {
                    if (r.visible) {
                        hasVisibleActivities = true;
                    }
                    final boolean remove;
                    ... ... ...
                    if (remove) {
                        if (DEBUG_ADD_REMOVE || DEBUG_CLEANUP) Slog.i(TAG_ADD_REMOVE,
                                "Removing activity " + r + " from stack at " + i
                                + ": haveState=" + r.haveState
                                + " stateNotNeeded=" + r.stateNotNeeded
                                + " finishing=" + r.finishing
                                + " state=" + r.state + " callers=" + Debug.getCallers(5));
                        if (!r.finishing) {
                            Slog.w(TAG, "Force removing " + r + ": app died, no saved state");
                            EventLog.writeEvent(EventLogTags.AM_FINISH_ACTIVITY,
                                    r.userId, System.identityHashCode(r),
                                    r.task.taskId, r.shortComponentName,
                                    "proc died without state saved");
                            if (r.state == ActivityState.RESUMED) {
                                //【2.2.2】如果 activity 处于 resume 状态，记录状态！
                                mService.updateUsageStats(r, false);
                            }
                        }
                    } else {
                        ... ... ..
                    }
                    ... ... ...
                }
            }
        }

        return hasVisibleActivities;
    }
```

逻辑很简单，不多说！

### 2.1.2 方法流程分析

参数 boolean resumed 表示是否处于 resume 状态！

```java
    void updateUsageStats(ActivityRecord component, boolean resumed) {
        if (DEBUG_SWITCH) Slog.d(TAG_SWITCH,
                "updateUsageStats: comp=" + component + "res=" + resumed);
        final BatteryStatsImpl stats = mBatteryStatsService.getActiveStatistics();
        if (resumed) {
            if (mUsageStatsService != null) {
                //【2.1.3】如果是 resume 状态，记录 UsageEvents.Event.MOVE_TO_FOREGROUND 事件！
                mUsageStatsService.reportEvent(component.realActivity, component.userId,
                        UsageEvents.Event.MOVE_TO_FOREGROUND);
            }
            synchronized (stats) {
                stats.noteActivityResumedLocked(component.app.uid);
            }
        } else {
            if (mUsageStatsService != null) {
                //【2.1.3】如果不是 resume 状态，记录 UsageEvents.Event.MOVE_TO_BACKGROUND 事件！
                mUsageStatsService.reportEvent(component.realActivity, component.userId,
                        UsageEvents.Event.MOVE_TO_BACKGROUND);
            }
            synchronized (stats) {
                stats.noteActivityPausedLocked(component.app.uid);
            }
        }
    }
```
不多说了！

### 2.1.3 LocalService.reportEvent


这里最终调用了 reportEvent 方法，传入的 ComponentName component 为 Activity！
```java
        @Override
        public void reportEvent(ComponentName component, int userId, int eventType) {
            if (component == null) {
                Slog.w(TAG, "Event reported without a component name");
                return;
            }

            UsageEvents.Event event = new UsageEvents.Event();
            event.mPackage = component.getPackageName();
            event.mClass = component.getClassName();

            event.mTimeStamp = SystemClock.elapsedRealtime();

            event.mEventType = eventType;
            //【1】发送了 MSG_REPORT_EVENT 消息！
            mHandler.obtainMessage(MSG_REPORT_EVENT, userId, 0, event).sendToTarget();
        }
```
不多说了！

## 2.2 updateConfigurationLocked

### 2.2.1 调用时机

主要是和 Configuration 配置更新相关的内容，调用地方过多，这里先不列出！

### 2.2.2 方法流程分析

和 updateConfigurationLocked 相关的有多个重载函数：

```java
    public void updateConfiguration(Configuration values) {
        enforceCallingPermission(android.Manifest.permission.CHANGE_CONFIGURATION,
                "updateConfiguration()");

        synchronized(this) {
            if (values == null && mWindowManager != null) {
                // sentinel: fetch the current configuration from the window manager
                values = mWindowManager.computeNewConfiguration();
            }

            if (mWindowManager != null) {
                mProcessList.applyDisplaySize(mWindowManager);
            }

            final long origId = Binder.clearCallingIdentity();
            if (values != null) {
                Settings.System.clearConfiguration(values);
            }
            updateConfigurationLocked(values, null, false);
            Binder.restoreCallingIdentity(origId);
        }
    }

    void updateUserConfigurationLocked() {
        Configuration configuration = new Configuration(mConfiguration);
        Settings.System.adjustConfigurationForUser(mContext.getContentResolver(), configuration,
                mUserController.getCurrentUserIdLocked(), Settings.System.canWrite(mContext));
        updateConfigurationLocked(configuration, null, false);
    }

    boolean updateConfigurationLocked(Configuration values, ActivityRecord starting,
            boolean initLocale) {
        return updateConfigurationLocked(values, starting, initLocale, false /* deferResume */);
    }

    boolean updateConfigurationLocked(Configuration values, ActivityRecord starting,
            boolean initLocale, boolean deferResume) {
        // pass UserHandle.USER_NULL as userId because we don't persist configuration for any user
        return updateConfigurationLocked(values, starting, initLocale, false /* persistent */,
                UserHandle.USER_NULL, deferResume);
    }
```
updateConfigurationLocked 方法用于更新指定 activity 的 Configuration 属性，主要有两个作用：

- 改变当前的配置；
- 确保指定的 activity 使用的是当前最新的 activity；

```java
    private boolean updateConfigurationLocked(Configuration values, ActivityRecord starting,
            boolean initLocale, boolean persistent, int userId, boolean deferResume) {
        int changes = 0;
        //【1】停止布局！
        if (mWindowManager != null) {
            mWindowManager.deferSurfaceLayout();
        }
        //【2】处理配置变化！
        if (values != null) {
            //【2.1】获得旧配置的拷贝；
            Configuration newConfig = new Configuration(mConfiguration);
            //【2.2】获得配置的更新信息！
            changes = newConfig.updateFrom(values);
            if (changes != 0) {
                if (DEBUG_SWITCH || DEBUG_CONFIGURATION) Slog.i(TAG_CONFIGURATION,
                        "Updating configuration to: " + values);

                EventLog.writeEvent(EventLogTags.CONFIGURATION_CHANGED, changes);
                
                //【2.3】处理多语言的更新！
                if (!initLocale && !values.getLocales().isEmpty() && values.userSetLocale) {
                    final LocaleList locales = values.getLocales();
                    int bestLocaleIndex = 0;
                    if (locales.size() > 1) {
                        if (mSupportedSystemLocales == null) {
                            mSupportedSystemLocales =
                                    Resources.getSystem().getAssets().getLocales();
                        }
                        bestLocaleIndex = Math.max(0,
                                locales.getFirstMatchIndex(mSupportedSystemLocales));
                    }
                    SystemProperties.set("persist.sys.locale",
                            locales.get(bestLocaleIndex).toLanguageTag());
                    LocaleList.setDefault(locales, bestLocaleIndex);
                    mHandler.sendMessage(mHandler.obtainMessage(SEND_LOCALE_TO_MOUNT_DAEMON_MSG,
                            locales.get(bestLocaleIndex)));
                }
                //【2.4】计算本次更新的序列号；
                mConfigurationSeq++;
                if (mConfigurationSeq <= 0) {
                    mConfigurationSeq = 1;
                }
                //【2.5】更新配置引用对象！
                newConfig.seq = mConfigurationSeq;
                mConfiguration = newConfig;
                Slog.i(TAG, "Config changes=" + Integer.toHexString(changes) + " " + newConfig);
                
                //【×2.2.3】调用 reportConfigurationChange 方法！
                mUsageStatsService.reportConfigurationChange(newConfig,
                        mUserController.getCurrentUserIdLocked());

                final Configuration configCopy = new Configuration(mConfiguration);

                // TODO: If our config changes, should we auto dismiss any currently
                // showing dialogs?
                mShowDialogs = shouldShowDialogs(newConfig, mInVrMode);

                AttributeCache ac = AttributeCache.instance();
                if (ac != null) {
                    ac.updateConfiguration(configCopy);
                }

                //【2.6】更新资源配置！
                mSystemThread.applyConfigurationToResources(configCopy);
                
                if (persistent && Settings.System.hasInterestingConfigurationChanges(changes)) {
                    Message msg = mHandler.obtainMessage(UPDATE_CONFIGURATION_MSG);
                    msg.obj = new Configuration(configCopy);
                    msg.arg1 = userId;
                    mHandler.sendMessage(msg);
                }
                //【2.7】调整屏幕密度改变
                final boolean isDensityChange = (changes & ActivityInfo.CONFIG_DENSITY) != 0;
                if (isDensityChange) {
                    // Reset the unsupported display size dialog.
                    mUiHandler.sendEmptyMessage(SHOW_UNSUPPORTED_DISPLAY_SIZE_DIALOG_MSG);

                    killAllBackgroundProcessesExcept(Build.VERSION_CODES.N,
                            ActivityManager.PROCESS_STATE_FOREGROUND_SERVICE);
                }
                //【2.8】修改进程的配置信息！
                for (int i=mLruProcesses.size()-1; i>=0; i--) {
                    ProcessRecord app = mLruProcesses.get(i);
                    try {
                        if (app.thread != null) {
                            if (DEBUG_CONFIGURATION) Slog.v(TAG_CONFIGURATION, "Sending to proc "
                                    + app.processName + " new config " + mConfiguration);
                            app.thread.scheduleConfigurationChanged(configCopy);
                        }
                    } catch (Exception e) {
                    }
                }
                //【2.9】发送配置改变的广播！
                Intent intent = new Intent(Intent.ACTION_CONFIGURATION_CHANGED);
                intent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY
                        | Intent.FLAG_RECEIVER_REPLACE_PENDING
                        | Intent.FLAG_RECEIVER_FOREGROUND);
                broadcastIntentLocked(null, null, intent, null, null, 0, null, null,
                        null, AppOpsManager.OP_NONE, null, false, false,
                        MY_PID, Process.SYSTEM_UID, UserHandle.USER_ALL);
                if ((changes&ActivityInfo.CONFIG_LOCALE) != 0) {
                    intent = new Intent(Intent.ACTION_LOCALE_CHANGED);
                    intent.addFlags(Intent.FLAG_RECEIVER_FOREGROUND);
	            if (initLocale || !mProcessesReady) {
                        intent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY);
                    }
                    broadcastIntentLocked(null, null, intent,
                            null, null, 0, null, null, null, AppOpsManager.OP_NONE,
                            null, false, false, MY_PID, Process.SYSTEM_UID, UserHandle.USER_ALL);
                }
            }

            //【2.10】更新 wms 中的配置信息，并且检查在更新了配置后，是否有 stack 需要被 resize；
            // 如果有的话，那就进行 resize 和 relaunch！
            if (mWindowManager != null) {
                final int[] resizedStacks = mWindowManager.setNewConfiguration(mConfiguration);
                if (resizedStacks != null) {
                    for (int stackId : resizedStacks) {
                        final Rect newBounds = mWindowManager.getBoundsForNewConfiguration(stackId);
                        mStackSupervisor.resizeStackLocked(
                                stackId, newBounds, null, null, false, false, deferResume);
                    }
                }
            }
        }

        boolean kept = true;
        final ActivityStack mainStack = mStackSupervisor.getFocusedStack();
        //【4】获得焦点栈，调整 top activity 的配置信息！
        // mainStack is null during startup.
        if (mainStack != null) {
            if (changes != 0 && starting == null) {
                starting = mainStack.topRunningActivityLocked();
            }

            if (starting != null) {
                kept = mainStack.ensureActivityConfigurationLocked(starting, changes, false);
                mStackSupervisor.ensureActivitiesVisibleLocked(starting, changes,
                        !PRESERVE_WINDOWS);
            }
        }
        //【5】继续布局！
        if (mWindowManager != null) {
            mWindowManager.continueSurfaceLayout();
        }
        return kept;
    }
```

这里调用了 UsageStatsService.reportConfigurationChange 方法，更新配置信息！

### 2.2.3 LocalService.updateConfigurationLocked

```java
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
            mHandler.obtainMessage(MSG_REPORT_EVENT, userId, 0, event).sendToTarget();
        }
```
对于 config 相关的 event 来说，目标包名是 android！


## 2.3 maybeUpdateProviderUsageStatsLocked

### 2.3.1 调用时机

- **ActivityManagerService.publishContentProviders**

当注册 ContentProvider 的时候，会触发 publishContentProviders 方法！

```java
    public final void publishContentProviders(IApplicationThread caller,
            List<ContentProviderHolder> providers) {
        if (providers == null) {
            return;
        }

        enforceNotIsolatedCaller("publishContentProviders");
        synchronized (this) {
            final ProcessRecord r = getRecordForAppLocked(caller);
            if (DEBUG_MU) Slog.v(TAG_MU, "ProcessRecord uid = " + r.uid);
            if (r == null) {
                throw new SecurityException(
                        "Unable to find app for caller " + caller
                      + " (pid=" + Binder.getCallingPid()
                      + ") when publishing content providers");
            }

            final long origId = Binder.clearCallingIdentity();

            final int N = providers.size();
            for (int i = 0; i < N; i++) {
                ContentProviderHolder src = providers.get(i);
                    ... ... ...
                    updateOomAdjLocked(r);
                    //【2.3.2】调用 maybeUpdateProviderUsageStatsLocked 方法，记录 ContentProvider 使用信息！
                    maybeUpdateProviderUsageStatsLocked(r, src.info.packageName,
                            src.info.authority);
                }
            }

            Binder.restoreCallingIdentity(origId);
        }
    }
```

- **ActivityManagerService.getContentProviderImpl**

当获得 ContentProvider 的时候，会触发 getContentProviderImpl 方法！
```java
    private ContentProviderHolder getContentProviderImpl(IApplicationThread caller,
            String name, IBinder token, boolean stable, int userId) {
        ContentProviderRecord cpr;
        ContentProviderConnection conn = null;
        ProviderInfo cpi = null;

        synchronized(this) {
            long startTime = SystemClock.uptimeMillis();
            
            ... ... ...
            
            boolean providerRunning = cpr != null && cpr.proc != null && !cpr.proc.killed;
            if (providerRunning) {
                cpi = cpr.info;
                String msg;
                checkTime(startTime, "getContentProviderImpl: before checkContentProviderPermission");
                if ((msg = checkContentProviderPermissionLocked(cpi, r, userId, checkCrossUser))
                        != null) {
                    throw new SecurityException(msg);
                }
                checkTime(startTime, "getContentProviderImpl: after checkContentProviderPermission");

                if (r != null && cpr.canRunHere(r)) {
                    ContentProviderHolder holder = cpr.newHolder(null);
                    holder.provider = null;
                    return holder;
                }

                final long origId = Binder.clearCallingIdentity();

                checkTime(startTime, "getContentProviderImpl: incProviderCountLocked");

                conn = incProviderCountLocked(r, cpr, token, stable);
                if (conn != null && (conn.stableCount+conn.unstableCount) == 1) {
                    if (cpr.proc != null && r.setAdj <= ProcessList.PERCEPTIBLE_APP_ADJ) {
                        checkTime(startTime, "getContentProviderImpl: before updateLruProcess");
                        updateLruProcessLocked(cpr.proc, false, null);
                        checkTime(startTime, "getContentProviderImpl: after updateLruProcess");
                    }
                }

                checkTime(startTime, "getContentProviderImpl: before updateOomAdj");
                final int verifiedAdj = cpr.proc.verifiedAdj;
                boolean success = updateOomAdjLocked(cpr.proc);
                if (success && verifiedAdj != cpr.proc.setAdj && !isProcessAliveLocked(cpr.proc)) {
                    success = false;
                }

                //【2.2.2】调用 maybeUpdateProviderUsageStatsLocked 方法再次记录！
                maybeUpdateProviderUsageStatsLocked(r, cpr.info.packageName, name);
                checkTime(startTime, "getContentProviderImpl: after updateOomAdj");
                if (DEBUG_PROVIDER) Slog.i(TAG_PROVIDER, "Adjust success: " + success);

                if (!success) {
                    Slog.i(TAG, "Existing provider " + cpr.name.flattenToShortString()
                            + " is crashing; detaching " + r);
                    boolean lastRef = decProviderCountLocked(conn, cpr, token, stable);
                    checkTime(startTime, "getContentProviderImpl: before appDied");
                    appDiedLocked(cpr.proc);
                    checkTime(startTime, "getContentProviderImpl: after appDied");
                    if (!lastRef) {
                        return null;
                    }
                    providerRunning = false;
                    conn = null;
                } else {
                    cpr.proc.verifiedAdj = cpr.proc.setAdj;
                }

                Binder.restoreCallingIdentity(origId);
            }
            ... ... ...
        }
        
        return cpr != null ? cpr.newHolder(conn) : null;
    }
```
对于 getContentProviderImpl 这里我们不再过多分析！

### 2.3.2 方法流程分析

maybeUpdateProviderUsageStatsLocked 方法中会进行事件上报！
```java
    private void maybeUpdateProviderUsageStatsLocked(ProcessRecord app, String providerPkgName,
            String authority) {
        if (app == null) return;
        if (app.curProcState <= ActivityManager.PROCESS_STATE_IMPORTANT_FOREGROUND) {
            UserState userState = mUserController.getStartedUserStateLocked(app.userId);
            if (userState == null) return;
            final long now = SystemClock.elapsedRealtime();
            Long lastReported = userState.mProviderLastReportedFg.get(authority);
            if (lastReported == null || lastReported < now - 60 * 1000L) {
                if (mSystemReady) {
                    //【2.3.3】调用了 reportContentProviderUsage 方法！
                    mUsageStatsService.reportContentProviderUsage(
                            authority, providerPkgName, app.userId);
                }
                userState.mProviderLastReportedFg.put(authority, now);
            }
        }
    }
```

### 2.3.3 LocalService.reportContentProviderUsage

```java
        @Override
        public void reportContentProviderUsage(String name, String packageName, int userId) {
            SomeArgs args = SomeArgs.obtain();
            args.arg1 = name;
            args.arg2 = packageName;
            args.arg3 = userId;
            //【1】发送了 MSG_REPORT_CONTENT_PROVIDER_USAGE 消息！
            mHandler.obtainMessage(MSG_REPORT_CONTENT_PROVIDER_USAGE, args)
                    .sendToTarget();
        }
```
不多说了！


## 2.4 maybeUpdateUsageStatsLocked

### 2.4.1 调用时机

- **ActivityManagerService.applyOomAdjLocked**

在调整进程的 oomAdj 的时候，会触发 maybeUpdateUsageStatsLocked！

```java
    private final boolean applyOomAdjLocked(ProcessRecord app, boolean doingAll, long now,
            long nowElapsed) {
        boolean success = true;

        if (app.curRawAdj != app.setRawAdj) {
            app.setRawAdj = app.curRawAdj;
        }

        int changes = 0;

        if (app.curAdj != app.setAdj) {
            ProcessList.setOomAdj(app.pid, app.info.uid, app.curAdj);
            if (DEBUG_SWITCH || DEBUG_OOM_ADJ) Slog.v(TAG_OOM_ADJ,
                    "Set " + app.pid + " " + app.processName + " adj " + app.curAdj + ": "
                    + app.adjType);
            app.setAdj = app.curAdj;
            app.verifiedAdj = ProcessList.INVALID_ADJ;
        }

        ... ... ...

        //【1】当进程的 ProcState 发生了变化是，进入 IF 分支！
        if (app.setProcState != app.curProcState) {
            if (DEBUG_SWITCH || DEBUG_OOM_ADJ) Slog.v(TAG_OOM_ADJ,
                    "Proc state change of " + app.processName
                            + " to " + app.curProcState);
            boolean setImportant = app.setProcState < ActivityManager.PROCESS_STATE_SERVICE;
            boolean curImportant = app.curProcState < ActivityManager.PROCESS_STATE_SERVICE;
            if (setImportant && !curImportant) {
                BatteryStatsImpl stats = mBatteryStatsService.getActiveStatistics();
                synchronized (stats) {
                    app.lastWakeTime = stats.getProcessWakeTime(app.info.uid,
                            app.pid, nowElapsed);
                }
                app.lastCpuTime = app.curCpuTime;

            }

            //【2.4.1】在更新进程的状态之前，调用 maybeUpdateUsageStatsLocked 方法
            // 记录进程的信息！
            maybeUpdateUsageStatsLocked(app, nowElapsed);

            app.setProcState = app.curProcState;
            if (app.setProcState >= ActivityManager.PROCESS_STATE_HOME) {
                app.notCachedSinceIdle = false;
            }
            if (!doingAll) {
                setProcessTrackerStateLocked(app, mProcessStats.getMemFactorLocked(), now);
            } else {
                app.procStateChanged = true;
            }
        } else if (app.reportedInteraction && (nowElapsed-app.interactionEventTime)
                > USAGE_STATS_INTERACTION_INTERVAL) {
            //【2.4.1】对于长期处于互动状态的应用，我们需要每天至少报告一次，以免它们进入 idle 状态。
            maybeUpdateUsageStatsLocked(app, nowElapsed);
        }

        ... ... ...

        return success;
    }
```
这里调用了 maybeUpdateUsageStatsLocked 方法！

### 2.4.2 方法流程分析

下面我们来看看 maybeUpdateUsageStatsLocked 方法流程：

```java
    private void maybeUpdateUsageStatsLocked(ProcessRecord app, long nowElapsed) {
        if (DEBUG_USAGE_STATS) {
            Slog.d(TAG, "Checking proc [" + Arrays.toString(app.getPackageList())
                    + "] state changes: old = " + app.setProcState + ", new = "
                    + app.curProcState);
        }
        if (mUsageStatsService == null) {
            return;
        }
        //【1】判断此时进程是否处于交互中！        
        boolean isInteraction;
        
        if (app.curProcState <= ActivityManager.PROCESS_STATE_BOUND_FOREGROUND_SERVICE) {
            //【2】如果进程的 curProcState 小于等于 PROCESS_STATE_BOUND_FOREGROUND_SERVICE: 3
            // 此时是处于交互状态的！
            isInteraction = true;
            app.fgInteractionTime = 0;
            
        } else if (app.curProcState <= ActivityManager.PROCESS_STATE_TOP_SLEEPING) {
            //【3】如果进程的 curProcState 小于等于 PROCESS_STATE_TOP_SLEEPING: 5
            // 进入这里！！
            if (app.fgInteractionTime == 0) {
                app.fgInteractionTime = nowElapsed;
                isInteraction = false;
            } else {
                isInteraction = nowElapsed > app.fgInteractionTime + SERVICE_USAGE_INTERACTION_TIME;
            }
            
        } else {
            //【4】如果 app.forcingToForeground 为 null，并且进程的 curProcState 
            // 小于等于 PROCESS_STATE_IMPORTANT_FOREGROUND:6 说明该进程是可以被感知的重要进程！
            // isInteraction 为 true！
            isInteraction = app.forcingToForeground == null
                    && app.curProcState <= ActivityManager.PROCESS_STATE_IMPORTANT_FOREGROUND;
            app.fgInteractionTime = 0;
        }

        //【5】如果本次进程是处于交互状态，并且上次是处于非交互状态（或者此时距离上次交互时间点超过了交互间隔）
        // 那么我们会上报一次记录！
        if (isInteraction && (!app.reportedInteraction
                || (nowElapsed-app.interactionEventTime) > USAGE_STATS_INTERACTION_INTERVAL)) {
            app.interactionEventTime = nowElapsed;
            String[] packages = app.getPackageList();
            if (packages != null) {
                for (int i = 0; i < packages.length; i++) {
                    //【5.1】调用了 reportEvent 方法，更新进程中所有 pacakge 的使用信息
                    // event 类型为 SYSTEM_INTERACTION！
                    mUsageStatsService.reportEvent(packages[i], app.userId,
                            UsageEvents.Event.SYSTEM_INTERACTION);
                }
            }
        }
        //【6】更新 app.reportedInteraction 和 app.interactionEventTime！
        app.reportedInteraction = isInteraction;
        if (!isInteraction) {
            app.interactionEventTime = 0;
        }
    }
```

最终调用了 UsageStatsService.reportEvent 方法！

### 2.4.3 LocalService.reportEvent

```java
        @Override
        public void reportEvent(String packageName, int userId, int eventType) {
            if (packageName == null) {
                Slog.w(TAG, "Event reported without a package name");
                return;
            }

            UsageEvents.Event event = new UsageEvents.Event();
            event.mPackage = packageName;

            // This will later be converted to system time.
            event.mTimeStamp = SystemClock.elapsedRealtime();

            event.mEventType = eventType;
            mHandler.obtainMessage(MSG_REPORT_EVENT, userId, 0, event).sendToTarget();
        }
```

## 2.5 shutdown

shutdown 主要用关机时候，保存重要的信息！
```java
    @Override
    public boolean shutdown(int timeout) {
        if (checkCallingPermission(android.Manifest.permission.SHUTDOWN)
                != PackageManager.PERMISSION_GRANTED) {
            throw new SecurityException("Requires permission "
                    + android.Manifest.permission.SHUTDOWN);
        }

        boolean timedout = false;

        synchronized(this) {
            mShuttingDown = true;
            updateEventDispatchingLocked();
            timedout = mStackSupervisor.shutdownLocked(timeout);
        }

        mAppOpsService.shutdown();
        if (mUsageStatsService != null) {
            //【1】调用 UsageStatsService 的 prepareShutdown 方法！
            mUsageStatsService.prepareShutdown();
        }
        mBatteryStatsService.shutdown();
        synchronized (this) {
            mProcessStats.shutdownLocked();
            notifyTaskPersisterLocked(null, true);
        }

        return timedout;
    }
```
这个逻辑不多说了！


# 3 NotificationManagerService

我们来看看 NotificationManagerService 和 UsageStatsService 的通信：

## 3.1 onStart

```java
   @Override
    public void onStart() {
        Resources resources = getContext().getResources();

        mMaxPackageEnqueueRate = Settings.Global.getFloat(getContext().getContentResolver(),
                Settings.Global.MAX_NOTIFICATION_ENQUEUE_RATE,
                DEFAULT_MAX_NOTIFICATION_ENQUEUE_RATE);
        ... ... ...

        mAppUsageStats = LocalServices.getService(UsageStatsManagerInternal.class);
```
## 3.2 setNotificationsShownFromListener

当通知可见后 setNotificationsShownFromListener 会被触发！

```java
        @Override
        public void setNotificationsShownFromListener(INotificationListener token, String[] keys) {
            long identity = Binder.clearCallingIdentity();
            try {
                synchronized (mNotificationList) {
                    final ManagedServiceInfo info = mListeners.checkServiceTokenLocked(token);
                    if (keys != null) {
                        final int N = keys.length;
                        for (int i = 0; i < N; i++) {
                            NotificationRecord r = mNotificationsByKey.get(keys[i]);
                            if (r == null) continue;
                            final int userId = r.sbn.getUserId();
                            if (userId != info.userid && userId != UserHandle.USER_ALL &&
                                    !mUserProfiles.isCurrentProfile(userId)) {
                                throw new SecurityException("Disallowed call from listener: "
                                        + info.service);
                            }
                            if (!r.isSeen()) {
                                if (DBG) Slog.d(TAG, "Marking notification as seen " + keys[i]);
                                //【1】调用了 UsageStatsService 的 reportEvent 方法！
                                mAppUsageStats.reportEvent(r.sbn.getPackageName(),
                                        userId == UserHandle.USER_ALL ? UserHandle.USER_SYSTEM
                                                : userId,
                                        UsageEvents.Event.USER_INTERACTION);
                                r.setSeen();
                            }
                        }
                    }
                }
            } finally {
                Binder.restoreCallingIdentity(identity);
            }
        }
```
这里对传入的 UserId 做了一个判断，如果为 UserHandle.USER_ALL，那么就设置为 UserHandle.USER_SYSTEM！

最终，调用了 UsageStatsService.reportEvent 上报事件：
```java
        mAppUsageStats.reportEvent(r.sbn.getPackageName(),
                userId == UserHandle.USER_ALL ? UserHandle.USER_SYSTEM
                        : userId,
                UsageEvents.Event.USER_INTERACTION);
```

# 4 NetworkPolicyManagerService

我们来看看 NetworkPolicyManagerService 和 UsageStatsService 的通信方式！

## 4.1 systemReady

先来看看 NetworkPolicyManagerService 的 systemReady 方法！

```java
    public void systemReady() {
        Trace.traceBegin(Trace.TRACE_TAG_NETWORK, "systemReady");
        try {
            if (!isBandwidthControlEnabled()) {
                Slog.w(TAG, "bandwidth controls disabled, unable to enforce policy");
                return;
            }
            //【1】获得了 UsageStats 对象！
            mUsageStats = LocalServices.getService(UsageStatsManagerInternal.class);
            ... ... ... ...
            //【2】注册 AppIdleStateChangeListener 监听器！
            mUsageStats.addAppIdleStateChangeListener(new AppIdleStateChangeListener());
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_NETWORK);
        }
    }
```
可以看到，最关键的一个地方，就是监听器的注册！

## 4.2 new AppIdleStateChangeListener

AppIdleStateChangeListener 继承了 UsageStatsManagerInternal.AppIdleStateChangeListener，覆写了两个父类方法：

- **onAppIdleStateChanged**：用于监听 app idle 状态变化！


- **onParoleStateChanged**：用于监听 app 假释状态的变化！

```java
    private class AppIdleStateChangeListener
            extends UsageStatsManagerInternal.AppIdleStateChangeListener {

        @Override
        public void onAppIdleStateChanged(String packageName, int userId, boolean idle) {
            try {
                final int uid = mContext.getPackageManager().getPackageUidAsUser(packageName,
                        PackageManager.MATCH_UNINSTALLED_PACKAGES, userId);
                if (LOGV) Log.v(TAG, "onAppIdleStateChanged(): uid=" + uid + ", idle=" + idle);
                synchronized (mUidRulesFirstLock) {
                    //【4.3.1】更新 app idle 规则；
                    updateRuleForAppIdleUL(uid);
                    //【4.3.2】更新省电模式的规则；
                    updateRulesForPowerRestrictionsUL(uid);
                }
            } catch (NameNotFoundException nnfe) {
            }
        }

        @Override
        public void onParoleStateChanged(boolean isParoleOn) {
            synchronized (mUidRulesFirstLock) {
                //【4.4.1】更新 app idle parole 规则；
                updateRulesForAppIdleParoleUL();
            }
        }
    }
```

## 4.3 App Idle State Changed

处理 app idle 状态的改变！

当 AppIdleStateChangeListener.onAppIdleStateChanged 触发！

### 4.3.1 updateRuleForAppIdleUL
```java
    void updateRuleForAppIdleUL(int uid) {
        //【1】判断 uid 是否在名单中，在名单中的应用不受机制影响！！
        if (!isUidValidForBlacklistRules(uid)) return;

        int appId = UserHandle.getAppId(uid);
        if (!mPowerSaveTempWhitelistAppIds.get(appId) && isUidIdle(uid)
                && !isUidForegroundOnRestrictPowerUL(uid)) {
            //【4.3.1.1】修改 FIREWALL_CHAIN_STANDBY 的规则为 FIREWALL_RULE_DENY
            setUidFirewallRule(FIREWALL_CHAIN_STANDBY, uid, FIREWALL_RULE_DENY);

        } else {
            //【4.3.1.2】修改 FIREWALL_CHAIN_STANDBY 的规则为 FIREWALL_RULE_DEFAULT
            setUidFirewallRule(FIREWALL_CHAIN_STANDBY, uid, FIREWALL_RULE_DEFAULT);

        }
    }
```

#### 4.3.1.1 setUidFirewallRule
```java
    private void setUidFirewallRule(int chain, int uid, int rule) {
        if (chain == FIREWALL_CHAIN_DOZABLE) {
            mUidFirewallDozableRules.put(uid, rule);
        } else if (chain == FIREWALL_CHAIN_STANDBY) {
            //【1】将 FIREWALL_CHAIN_STANDBY 相关的规则加入 mUidFirewallStandbyRules！
            mUidFirewallStandbyRules.put(uid, rule);
        } else if (chain == FIREWALL_CHAIN_POWERSAVE) {
            mUidFirewallPowerSaveRules.put(uid, rule);
        }

        try {
            mNetworkManager.setFirewallUidRule(chain, uid, rule);
        } catch (IllegalStateException e) {
            Log.wtf(TAG, "problem setting firewall uid rules", e);
        } catch (RemoteException e) {
            // ignored; service lives in system_server
        }
    }
```


### 4.3.2 updateRulesForPowerRestrictionsUL[1]

```java
    private void updateRulesForPowerRestrictionsUL(int uid) {
        final int oldUidRules = mUidRules.get(uid, RULE_NONE);

        final int newUidRules = updateRulesForPowerRestrictionsUL(uid, oldUidRules, false);

        if (newUidRules == RULE_NONE) {
            mUidRules.delete(uid);
        } else {
            mUidRules.put(uid, newUidRules);
        }
    }
```


## 4.4 Parole State Changed

处理 app 假释状态的改变！


### 4.4.1 updateRulesForAppIdleParoleUL

```java
    void updateRulesForAppIdleParoleUL() {
        //【1】判断此时是否处于假释状态！
        boolean paroled = mUsageStats.isAppIdleParoleOn();

        //【2】判断是否可以访问网网络！
        boolean enableChain = !paroled;
        enableFirewallChainUL(FIREWALL_CHAIN_STANDBY, enableChain);

        //【3】遍历 mUidFirewallStandbyRules 中的所有 rule！
        int ruleCount = mUidFirewallStandbyRules.size();
        for (int i = 0; i < ruleCount; i++) {
            int uid = mUidFirewallStandbyRules.keyAt(i);
            int oldRules = mUidRules.get(uid);
            if (enableChain) {
                //【3.1】如果可以访问网络，取消掉 RULE_ALLOW_ALL/RULE_REJECT_ALL 位！
                oldRules &= MASK_METERED_NETWORKS;

            } else {
                //【3.2】如果不可以访问网络，并且 oldRules 没有设置 RULE_ALLOW_ALL/RULE_REJECT_ALL 位！
                // 那就跳过，否则进入下面的环节！
                if ((oldRules & MASK_ALL_NETWORKS) == 0) continue;

            }
            //【4.4.2】进入 updateRulesForPowerRestrictionsUL 方法！
            updateRulesForPowerRestrictionsUL(uid, oldRules, paroled);
        }
    }
``` 
### 4.4.2 updateRulesForPowerRestrictionsUL[3]

```java
    private int updateRulesForPowerRestrictionsUL(int uid, int oldUidRules, boolean paroled) {
        if (!isUidValidForBlacklistRules(uid)) {
            if (LOGD) Slog.d(TAG, "no need to update restrict power rules for uid " + uid);
            return RULE_NONE;
        }
        //【1】判断是否是 idle 状态！
        final boolean isIdle = !paroled && isUidIdle(uid);
        //【2】判断是否是强制模式！
        final boolean restrictMode = isIdle || mRestrictPower || mDeviceIdleMode;
        //【3】判断 uid 下的进程是否是前台进程！
        final boolean isForeground = isUidForegroundOnRestrictPowerUL(uid);
        //【4】判断 uid 是否在省电白名单中！
        final boolean isWhitelisted = isWhitelistedBatterySaverUL(uid);
        final int oldRule = oldUidRules & MASK_ALL_NETWORKS;
        int newRule = RULE_NONE;

        //【5】
        if (isForeground) {
            if (restrictMode) {
                newRule = RULE_ALLOW_ALL;
            }
        } else if (restrictMode) {
            newRule = isWhitelisted ? RULE_ALLOW_ALL : RULE_REJECT_ALL;
        }

        //【6】计算新的 rules,oldUidRules & MASK_METERED_NETWORKS 取消旧的高四位！
        final int newUidRules = (oldUidRules & MASK_METERED_NETWORKS) | newRule;

        if (LOGV) {
            Log.v(TAG, "updateRulesForPowerRestrictionsUL(" + uid + ")"
                    + ", isIdle: " + isIdle
                    + ", mRestrictPower: " + mRestrictPower
                    + ", mDeviceIdleMode: " + mDeviceIdleMode
                    + ", isForeground=" + isForeground
                    + ", isWhitelisted=" + isWhitelisted
                    + ", oldRule=" + uidRulesToString(oldRule)
                    + ", newRule=" + uidRulesToString(newRule)
                    + ", newUidRules=" + uidRulesToString(newUidRules)
                    + ", oldUidRules=" + uidRulesToString(oldUidRules));
        }

        //【7】如果规则发生了变化，那就发送 MSG_RULES_CHANGED 消息！
        if (newRule != oldRule) {
            if (newRule == RULE_NONE || (newRule & RULE_ALLOW_ALL) != 0) {
                if (LOGV) Log.v(TAG, "Allowing non-metered access for UID " + uid);
            } else if ((newRule & RULE_REJECT_ALL) != 0) {
                if (LOGV) Log.v(TAG, "Rejecting non-metered access for UID " + uid);
            } else {
                Log.wtf(TAG, "Unexpected change of non-metered UID state for " + uid
                        + ": foreground=" + isForeground
                        + ", whitelisted=" + isWhitelisted
                        + ", newRule=" + uidRulesToString(newUidRules)
                        + ", oldRule=" + uidRulesToString(oldUidRules));
            }
            mHandler.obtainMessage(MSG_RULES_CHANGED, uid, newUidRules).sendToTarget();
        }

        return newUidRules;
    }

```


# 5 AppIdleController

学习过 JobSchedulerSerivce，我们知道 AppIdleController 会判断 app 是否进入 idle 状态，这样动态控制 app 对应的 job！


## 5.1 new AppIdleController
```java
    private AppIdleController(JobSchedulerService service, Context context, Object lock) {
        super(service, context, lock);
        mJobSchedulerService = service;
        //【1】获得 UsageStatsService 的 LocalService 对象！
        mUsageStatsInternal = LocalServices.getService(UsageStatsManagerInternal.class);
        //【2】初始化 mAppIdleParoleOn 为 true！
        mAppIdleParoleOn = true;
        //【3】注册 AppIdleStateChangeListener 监听器！
        mUsageStatsInternal.addAppIdleStateChangeListener(new AppIdleStateChangeListener());
    }
```

## 5.2 new AppIdleStateChangeListener

AppIdleStateChangeListener 用于监听 app idle 状态的变化！
```java
    private class AppIdleStateChangeListener
            extends UsageStatsManagerInternal.AppIdleStateChangeListener {
            
        @Override
        public void onAppIdleStateChanged(String packageName, int userId, boolean idle) {
            boolean changed = false;
            synchronized (mLock) {
                //【1】如果此时应用进入了假释状态，不处理！
                if (mAppIdleParoleOn) {
                    return;
                }
                //【2】为系统中的所有 Job 执行 PackageUpdateFunc 操作！
                // idle 表示是否处于 idle 状态！
                PackageUpdateFunc update = new PackageUpdateFunc(userId, packageName, idle);
                mJobSchedulerService.getJobStore().forEachJob(update);
                if (update.mChanged) {
                    changed = true;
                }
            }
            //【3】通知 JobSchedulerService，AppIdleController 状态发生了变化！
            // 需要执行/停止 Job！
            if (changed) {
                mStateChangedListener.onControllerStateChanged();
            }
        }

        @Override
        public void onParoleStateChanged(boolean isParoleOn) {
            if (DEBUG) {
                Slog.d(LOG_TAG, "Parole on: " + isParoleOn);
            }
            //【4】当应用的假释模式发生变化后，调用 setAppIdleParoleOn 方法！
            setAppIdleParoleOn(isParoleOn);
        }
    }
```
- 当应用的 app idle 状态发生了变化，会触发 onAppIdleStateChanged 方法！
- 当应用的 app 假释状态发生了变化，会触发 onParoleStateChanged 方法！

### 5.2.1 PackageUpdateFunc.process

当 app idle 状态变化后，会触发 AppIdleController.AppIdleStateChangeListener.onAppIdleStateChanged 方法！

```java
    final static class PackageUpdateFunc implements JobStore.JobStatusFunctor {
        final int mUserId;
        final String mPackage;
        final boolean mIdle;
        boolean mChanged; // 是否有 job 的条件发生变化！

        PackageUpdateFunc(int userId, String pkg, boolean idle) {
            mUserId = userId;
            mPackage = pkg;
            mIdle = idle;
        }

        @Override public void process(JobStatus jobStatus) {
            //【1】匹配到对应的 JobStatus！
            if (jobStatus.getSourcePackageName().equals(mPackage)
                    && jobStatus.getSourceUserId() == mUserId) {

                //【1.1】设置 Job 的 AppNotIdle 条件，如果 Job 的条件发生变化
                // mChanged 为 true！
                if (jobStatus.setAppNotIdleConstraintSatisfied(!mIdle)) {
                    if (DEBUG) {
                        Slog.d(LOG_TAG, "App Idle state changed, setting idle state of "
                                + mPackage + " to " + mIdle);
                    }
                    mChanged = true;
                }
            }
        }
    };
```
jobStatus.setAppNotIdleConstraintSatisfied 方法用于设置 app 是否满足不处于 idle 状态下的条件！

### 5.2.2 setAppIdleParoleOn

当应用的 app 假释状态发生了变化，会触发 setAppIdleParoleOn 方法！

```java
    void setAppIdleParoleOn(boolean isAppIdleParoleOn) {
        boolean changed = false;
        synchronized (mLock) {
            //【1】如果假释状态没有变化，不处理!
            if (mAppIdleParoleOn == isAppIdleParoleOn) {
                return;
            }
            //【2】保存最新的假释状态！
            mAppIdleParoleOn = isAppIdleParoleOn;
            
            //【3】对所有的 Job 执行 GlobalUpdateFunc！
            GlobalUpdateFunc update = new GlobalUpdateFunc();
            mJobSchedulerService.getJobStore().forEachJob(update);
            
            //【4】如果 Job 触发条件变化，changed 为 true！
            if (update.mChanged) {
                changed = true;
            }
        }
        //【4】通知 JobSchedulerService，AppIdleController 状态发生了变化！
        // 需要执行/停止 Job！
        if (changed) {
            mStateChangedListener.onControllerStateChanged();
        }
    }
```
方法流程很简单！

#### 5.2.2.1 GlobalUpdateFunc.process

我们来看看 GlobalUpdateFunc 的逻辑！
```java
    final class GlobalUpdateFunc implements JobStore.JobStatusFunctor {
        boolean mChanged;

        @Override public void process(JobStatus jobStatus) {
            String packageName = jobStatus.getSourcePackageName();
            //【1】判断 app 是否处于 idle 状态，满足条件是进入了 idle 状态，但是不能是假释状态！
            final boolean appIdle = !mAppIdleParoleOn && mUsageStatsInternal.isAppIdle(packageName,
                    jobStatus.getSourceUid(), jobStatus.getSourceUserId());
            if (DEBUG) {
                Slog.d(LOG_TAG, "Setting idle state of " + packageName + " to " + appIdle);
            }
            //【2】修改 Job 的 appNotIdle 执行条件，如果条件发生了变化，mChanged 为 true！
            if (jobStatus.setAppNotIdleConstraintSatisfied(!appIdle)) {
                mChanged = true;
            }
        }
    };
```
方法流程很简单！

## 5.3 maybeStartTrackingJobLocked

AppIdleController 开始监控指定的 Job！

```java
    @Override
    public void maybeStartTrackingJobLocked(JobStatus jobStatus, JobStatus lastJob) {
        //【1】如果还没有初始化过假释状态，进行初始化！
        if (!mInitializedParoleOn) {
            mInitializedParoleOn = true;
            //【1.1】调用 UsageStatsService.isAppIdleParoleOn 方法判断此时是否是应用假释模式！
            mAppIdleParoleOn = mUsageStatsInternal.isAppIdleParoleOn();
        }

        String packageName = jobStatus.getSourcePackageName();
        //【2】判断 app 是否处于 app idle ，满足 2 个条件，应用处于 idle 状态，但是不能处于假释模式！
        final boolean appIdle = !mAppIdleParoleOn && mUsageStatsInternal.isAppIdle(packageName,
                jobStatus.getSourceUid(), jobStatus.getSourceUserId());
    
        if (DEBUG) {
            Slog.d(LOG_TAG, "Start tracking, setting idle state of "
                    + packageName + " to " + appIdle);
        }
        //【3】设置 Job 的条件属性！
        jobStatus.setAppNotIdleConstraintSatisfied(!appIdle);
    }
```
方法流程很简单！

