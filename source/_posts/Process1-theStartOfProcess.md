# Process篇 1 - Android 进程的启动
title: Process篇 1 - Android 进程的启动
date: 2016/03/13 20:46:25
catalog: true
categories:
- AndroidFramework源码分析
- Process进程
tags: Process进程
---

[toc]

基于 Android 7.1.1 源码分析，进程

# 1 前言
我们知道，在 Android 系统中，每一个 app 都至少运行在一个进程中的，可以通过配置 Android:process 属性，来使 app 的某个组件运行在不同的进程中的，从而达到一个 app 在运行在多个进程中！

本文将总结和分析 Android 进程启动进程的主要流程，更深入地理解 Android 系统的架构！


# 2 启动进程的方法

AMS 中进程启动相关的方法有如下的 2 组：
```java
private final void startProcessLocked(ProcessRecord app,
                                      String hostingType, String hostingNameStr) {
        //【1】进一步调用                              
        startProcessLocked(app, hostingType, 
                           hostingNameStr, null /* abiOverride */,
                           null /* entryPoint */, null /* entryPointArgs */);
}

private final void startProcessLocked(ProcessRecord app, String hostingType,
                                      String hostingNameStr, String abiOverride,                                                  String entryPoint, String[] entryPointArgs) {
       ... ... ... ...
}
```
第一组，可以看出启动方式是通过进程的 ProcessRecord 对象来启动的！
参数传递：

- **ProcessRecord app**：进程对应的 ProcessRecord 对象！
- **String hostingType**：
- **String hostingNameStr**：
- **String abiOverride：null**
- **String entryPoint：null**
- **String[] entryPointArgs：null**

接着来看：
```java
    final ProcessRecord startProcessLocked(String processName, ApplicationInfo info，
                                           boolean knownToBeDead, int intentFlags,
                                           String hostingType, ComponentName hostingName, 
                                           boolean allowWhileBooting,
                                           boolean isolated, boolean keepIfLarge) {
        
        // 进一步调用
        return startProcessLocked(processName, info, 
                                  knownToBeDead, intentFlags, hostingType,
                                  hostingName, allowWhileBooting, isolated, 
                                  0 /* isolatedUid */, keepIfLarge,
                                  null /* ABI override */, null /* entryPoint */,
                                  null /* entryPointArgs */,null /* crashHandler */);
    }

    final ProcessRecord startProcessLocked(String processName, ApplicationInfo info,
                                           boolean knownToBeDead, int intentFlags, 
                                           String hostingType, ComponentName hostingName,
                                           boolean allowWhileBooting, boolean isolated, 
                                           int isolatedUid, boolean keepIfLarge,
                                           String abiOverride, String entryPoint, 
                                           String[] entryPointArgs, Runnable crashHandler) {
                                           
       ... ... ... ...

        // 最后又调用了第一组方法中的第二个方法！！
        startProcessLocked(app, hostingType, hostingNameStr, abiOverride,   
                                                  entryPoint, entryPointArgs);

        checkTime(startTime, "startProcess: done starting proc!");
        return (app.pid != 0) ? app : null;
    }
```
第二组，可以看出启动方式是通过应用的进程名和 ApplicationInfo来启动进程的！
参数传递：

- **String processName**：进程名字，默认为包名！
- **ApplicationInfo info**：进程所属应用的信息对象！
- **boolean knownToBeDead**：父进程是否需要在该进程死亡后接到通知！
- **int intentFlags**：intent 启动的 flags 参数！
- **String hostingType**：值为 “activity”，“service”，“broadcast” 或者 “content provider”；
- **String hostingNameStr**：数据类型为 ComponentName，代表的是具体相对应的组件名！
- **boolean allowWhileBooting**：这个在AMS的启动篇里有涉及过，表示开机时是否拉起该进程!
- **boolean isolated**：表示该进程是否是一个隔离进程！
- **int isolatedUid**：0 隔离进程的 uid！
- **boolean keepIfLarge**：
- **String abiOverride：null**
- **String entryPoint：null**
- **String[] entryPointArgs：null**
- **Runnable crashHandler：null**

最后，都调用了：
```java
    private final void startProcessLocked(ProcessRecord app, String hostingType,
                                      String hostingNameStr, String abiOverride,                                                  String entryPoint, String[] entryPointArgs) {
       ... ... ... ...
```
接着，我们进入今天的正文，进程的启动！！


# 3 startProcessLocked - 启动
接下来，我们来看看用进程名 precessName 和应用的 ApplicationInfo 对象启动进程的方法，这个方法略微有些长，我们来仔细的分析：

```java
    final ProcessRecord startProcessLocked(String processName, ApplicationInfo info,
            boolean knownToBeDead, int intentFlags, String hostingType, ComponentName hostingName,
            boolean allowWhileBooting, boolean isolated, int isolatedUid, boolean keepIfLarge,
            String abiOverride, String entryPoint, String[] entryPointArgs, Runnable crashHandler) {
            
        long startTime = SystemClock.elapsedRealtime();
        ProcessRecord app;
        //【1】如果该进程不是一个隔离进程！
        if (!isolated) { 
            //【1】尝试获得其对应的 ProcessRecord 对象！
            app = getProcessRecordLocked(processName, info.uid, keepIfLarge);
            checkTime(startTime, "startProcess: after getProcessRecord");
            
            if ((intentFlags & Intent.FLAG_FROM_BACKGROUND) != 0) {

                // 如果是后台操作启动该进程，需要判断该进程是否是一个 bad 进程，如果是，启动失败！
                if (mAppErrors.isBadProcessLocked(info)) {
                    if (DEBUG_PROCESSES) Slog.v(TAG, "Bad process: " + info.uid
                            + "/" + info.processName);
                    return null;
                }
            } else {

                if (DEBUG_PROCESSES) Slog.v(TAG, "Clearing bad process: " + info.uid
                        + "/" + info.processName);

                // 如果是用户交互，主动启动这个进程，那么，就要清空这个进程的 crash  次数
                // 并且将该进程从 bad 列表中移除，变为一个 good 进程！
                // 直到下一次再次 crash 才会变成一个 bad 进程！
                mAppErrors.resetProcessCrashTimeLocked(info);
                if (mAppErrors.isBadProcessLocked(info)) {
                    EventLog.writeEvent(EventLogTags.AM_PROC_GOOD,
                            UserHandle.getUserId(info.uid), info.uid,
                            info.processName);
    
                    mAppErrors.clearBadProcessLocked(info);
                    if (app != null) {
                        app.bad = false;
                    }
                }
            }
        } else {

            // 对于隔离进程，不能利用已存在的 ProcessRecord！
            app = null;
        }

        // app launch boost for big.little configurations
        // use cpusets to migrate freshly launched tasks to big cores
        nativeMigrateToBoost();
        mIsBoosted = true;
        mBoostStartTime = SystemClock.uptimeMillis();
        Message msg = mHandler.obtainMessage(APP_BOOST_DEACTIVATE_MSG);
        mHandler.sendMessageDelayed(msg, APP_BOOST_MESSAGE_DELAY);

        if (DEBUG_PROCESSES) Slog.v(TAG_PROCESSES, "startProcess: name=" + processName
                + " app=" + app + " knownToBeDead=" + knownToBeDead
                + " thread=" + (app != null ? app.thread : null)
                + " pid=" + (app != null ? app.pid : -1));

         // 如果 app 不为 null，且 pid 大于 0，说明这个进程已经或者正在启动！
        if (app != null && app.pid > 0) {
        
            // 如果调用者不知道该进程已死亡，并且进程并没有被 kill（正在运行）
            // 或是该 ProcessRecord 没有绑定应用程序进程的 ApplicationThread 对象！
            if ((!knownToBeDead && !app.killed) || app.thread == null) {

                if (DEBUG_PROCESSES) Slog.v(TAG_PROCESSES, "App already running: " + app);

                // 如果这是该进程中的一个新应用 package，将其加入到进程的列表中！
                app.addPackage(info.packageName, info.versionCode, mProcessStats);
                checkTime(startTime, "startProcess: done, added package to proc");
                
                // 直接返回这个已经存在的 ProcessRecord，退出！
                return app;
            }

            // An application record is attached to a previous process,
            // clean it up now.
            if (DEBUG_PROCESSES || DEBUG_CLEANUP) Slog.v(TAG_PROCESSES, "App died: " + app);
            checkTime(startTime, "startProcess: bad proc running, killing");

            // 该 ProcessRecord 已经绑定了上一个进程，杀死并清理该进程！
            killProcessGroup(app.uid, app.pid);
            handleAppDiedLocked(app, true, true);
            checkTime(startTime, "startProcess: done killing old proc");
        }

        String hostingNameStr = hostingName != null
                ? hostingName.flattenToShortString() : null;

        if (app == null) { // 如果 app 为 null，那说明这是一个新进程！
            checkTime(startTime, "startProcess: creating new process record");
            
            // 创建新进程对应的 ProcessRecord 对象！
            app = newProcessRecordLocked(info, processName, isolated, isolatedUid);
            if (app == null) {
                Slog.w(TAG, "Failed making new process record for "
                        + processName + "/" + info.uid + " isolated=" + isolated);
                return null;
            }
            
            // 设置进程的 crashHandler，默认为 null！
            app.crashHandler = crashHandler;
            checkTime(startTime, "startProcess: done creating new process record");
        } else {

            // 如果这是该进程中的一个新应用 package，将其加入到进程的列表中！
            app.addPackage(info.packageName, info.versionCode, mProcessStats);
            checkTime(startTime, "startProcess: added package to existing proc");
        }

        // 如果系统还没准备好，则将当前进程加入到 mPorcessOnHold 中！
        if (!mProcessesReady
                && !isAllowedWhileBooting(info)
                && !allowWhileBooting) {
    
            if (!mProcessesOnHold.contains(app)) {
                mProcessesOnHold.add(app);
            }
            if (DEBUG_PROCESSES) Slog.v(TAG_PROCESSES,
                    "System not ready, putting on hold: " + app);
            checkTime(startTime, "startProcess: returning with proc on hold");

            // 返回这个进程对应的 ProcessRecord 对象！
            return app;
        }

        checkTime(startTime, "startProcess: stepping in to startProcess");

        // 到这里，说明进程还没有创建，接着调用 startRrocessLocked 方法，进行进一步的启动！
        startProcessLocked(
                app, hostingType, hostingNameStr, abiOverride, entryPoint, entryPointArgs);
                

        checkTime(startTime, "startProcess: done starting proc!");
        return (app.pid != 0) ? app : null;
    }
```
继续分析！

## 3.1 AMS.newProcessRecordLocked

创建进程对应的 ProcessRecord 结构体，参数传递：

- ApplicationInfo info：应用程序的 info 对象！
- String customProcess：进程名
- boolean isolated：是否是隔离进程！
- int isolatedUid：隔离进程的 uid，默认传入 0！

```java
    final ProcessRecord newProcessRecordLocked(ApplicationInfo info, String customProcess,
            boolean isolated, int isolatedUid) {
        String proc = customProcess != null ? customProcess : info.processName;
        BatteryStatsImpl stats = mBatteryStatsService.getActiveStatistics();
        final int userId = UserHandle.getUserId(info.uid);
        int uid = info.uid;

        if (isolated) { // 隔离进程！

            if (isolatedUid == 0) { //如果传入的值的是默认值 0，系统会给隔离进程分配 uid！
                int stepsLeft = Process.LAST_ISOLATED_UID - Process.FIRST_ISOLATED_UID + 1;
                while (true) {
                    if (mNextIsolatedProcessUid < Process.FIRST_ISOLATED_UID
                            || mNextIsolatedProcessUid > Process.LAST_ISOLATED_UID) {
                        mNextIsolatedProcessUid = Process.FIRST_ISOLATED_UID;
                    }
                    uid = UserHandle.getUid(userId, mNextIsolatedProcessUid);
                    mNextIsolatedProcessUid++;
                    if (mIsolatedProcesses.indexOfKey(uid) < 0) {
                        // No process for this uid, use it.
                        break;
                    }
                    stepsLeft--;
                    if (stepsLeft <= 0) {
                        return null;
                    }
                }
            } else { // 如果传入的不是默认值 0，那隔离进程的 uid 就由传入参数指定！
                uid = isolatedUid;
            }

            // Register the isolated UID with this application so BatteryStats knows to
            // attribute resource usage to the application.
            //
            // NOTE: This is done here before addProcessNameLocked, which will tell BatteryStats
            // about the process state of the isolated UID *before* it is registered with the
            // owning application.
            mBatteryStatsService.addIsolatedUid(uid, info.uid);
        }
 
        // 创建 ProcessRecord 对象！
        final ProcessRecord r = new ProcessRecord(stats, info, proc, uid);
        if (!mBooted && !mBooting
                && userId == UserHandle.USER_SYSTEM
                && (info.flags & PERSISTENT_MASK) == PERSISTENT_MASK) {
            // 设置其 persistent 属性 为 true！
            r.persistent = true;
        }
        // 见 3.1.1
        addProcessNameLocked(r);
        return r;
    }
```
### 3.1.1 AMS.addProcessNameLocked
接着，调用该方法，将新创建的 ProcessRecord 加入到 AMS 对应的集合：
```java

    private final void addProcessNameLocked(ProcessRecord proc) {
        // We shouldn't already have a process under this name, but just in case we
        // need to clean up whatever may be there now.
        
        // 移除相同的之前被添加过的进程！
        ProcessRecord old = removeProcessNameLocked(proc.processName, proc.uid);
        if (old == proc && proc.persistent) {
            // We are re-adding a persistent process.  Whatevs!  Just leave it there.
            Slog.w(TAG, "Re-adding persistent process " + proc);
        } else if (old != null) {
            Slog.wtf(TAG, "Already have existing proc " + old + " when adding " + proc);
        }
        
        // 给这个进程的所属 uid 创建 UidRecord 对象，如果已创建过了，就不创建！！
        UidRecord uidRec = mActiveUids.get(proc.uid);
        if (uidRec == null) {
            uidRec = new UidRecord(proc.uid);
            // This is the first appearance of the uid, report it now!
            if (DEBUG_UID_OBSERVERS) Slog.i(TAG_UID_OBSERVERS,
                    "Creating new process uid: " + uidRec);
            // 第一次创建时，要加入到 mActiveUids 集合中去！        
            mActiveUids.put(proc.uid, uidRec);
            noteUidProcessState(uidRec.uid, uidRec.curProcState);
            enqueueUidChangeLocked(uidRec, -1, UidRecord.CHANGE_ACTIVE);
        }
        
        // 添加 proc 对 UidRecord 引用！
        proc.uidRecord = uidRec;

        // Reset render thread tid if it was already set, so new process can set it again.
        proc.renderThreadTid = 0;
        
        // UidRecord 的被引用计数加 1
        uidRec.numProcs++;
        
        // 将新创建的 ProcessRecord 对象加入到 ams 的 ProcessNames 中！
        mProcessNames.put(proc.processName, proc.uid, proc);
        if (proc.isolated) {
            // 如果该进程是隔离进程，还要添加到 mIsolatedProcesses 集合中去！！
            mIsolatedProcesses.put(proc.uid, proc);
        }
    }
```
这里有提到 AMS 的几个成员对象：

- mActiveUids：当前活跃的 uid 集合；
- mProcessNames：以进程的 processName 为 key，存储所有的 ProcessRecord 对象！
- mIsolatedProcesses：以进程的 pid 为 key，存储所有的 ProcessRecord 对象！


# 4 startProcessLocked - ProcessRecord
接着们就要进入最关键的一个方法哦，启动进程！
参数传递：

- ProcessRecord app, 
- String hostingType,
- String hostingNameStr：
- String abiOverride：
- String entryPoint：null
- String[] entryPointArg：null

```java
    private final void startProcessLocked(ProcessRecord app, String hostingType,
            String hostingNameStr, String abiOverride, String entryPoint, String[] entryPointArgs) {
        long startTime = SystemClock.elapsedRealtime();

        // 如果要启动的进程不是 systemServer 进程，也就是应用进程！
        if (app.pid > 0 && app.pid != MY_PID) {
            checkTime(startTime, "startProcess: removing from pids map");

            synchronized (mPidsSelfLocked) {
                // 将进程从 mPidsSelfLocked 中移除！
                mPidsSelfLocked.remove(app.pid);
                mHandler.removeMessages(PROC_START_TIMEOUT_MSG, app);
            }

            checkTime(startTime, "startProcess: done removing from pids map");

            // 先设置进程的 pid 为 0！
            app.setPid(0);
        }

        if (DEBUG_PROCESSES && mProcessesOnHold.contains(app)) Slog.v(TAG_PROCESSES,
                "startProcessLocked removing on hold: " + app);
                
        // 记得前面的 mProcessesOnHold 吗？也要把进程从这里 remove 掉！
        mProcessesOnHold.remove(app);

        checkTime(startTime, "startProcess: starting to update cpu stats");
        updateCpuStats();
        checkTime(startTime, "startProcess: done updating cpu stats");

        try {
            try {
                // 获得设备用户 id!
                final int userId = UserHandle.getUserId(app.uid);
                AppGlobals.getPackageManager().checkPackageStartable(app.info.packageName, userId);
            } catch (RemoteException e) {
                throw e.rethrowAsRuntimeException();
            }

            int uid = app.uid;
            int[] gids = null;
            int mountExternal = Zygote.MOUNT_EXTERNAL_NONE;
            
            if (!app.isolated) { // 对于非隔离的进程！

                int[] permGids = null;
                try {

                    checkTime(startTime, "startProcess: getting gids from package manager");
                    final IPackageManager pm = AppGlobals.getPackageManager();

                    permGids = pm.getPackageGids(app.info.packageName,
                            MATCH_DEBUG_TRIAGED_MISSING, app.userId);

                    MountServiceInternal mountServiceInternal = LocalServices.getService(
                            MountServiceInternal.class);

                    mountExternal = mountServiceInternal.getExternalStorageMountMode(uid,
                            app.info.packageName);

                } catch (RemoteException e) {
                    throw e.rethrowAsRuntimeException();
                }

                /*
                 * Add shared application and profile GIDs so applications can share some
                 * resources like shared libraries and access user-wide resources
                 */
                if (ArrayUtils.isEmpty(permGids)) {
                    gids = new int[2];
                } else {
                    gids = new int[permGids.length + 2];
                    System.arraycopy(permGids, 0, gids, 2, permGids.length);
                }

                gids[0] = UserHandle.getSharedAppGid(UserHandle.getAppId(uid));
                gids[1] = UserHandle.getUserGid(UserHandle.getUserId(uid));
            }
            checkTime(startTime, "startProcess: building args");

            if (mFactoryTest != FactoryTest.FACTORY_TEST_OFF) {
                if (mFactoryTest == FactoryTest.FACTORY_TEST_LOW_LEVEL
                        && mTopComponent != null
                        && app.processName.equals(mTopComponent.getPackageName())) {
                    uid = 0;
                }
                if (mFactoryTest == FactoryTest.FACTORY_TEST_HIGH_LEVEL
                        && (app.info.flags&ApplicationInfo.FLAG_FACTORY_TEST) != 0) {
                    uid = 0;
                }
            }
            
            // 处理 debugFlags 相关的取值！
            int debugFlags = 0;
           
            ... ... ... ...// 此处是设置调试相关的代码，暂时省略！ 
           
            // 设置 app 要运行的硬件环境！
            String requiredAbi = (abiOverride != null) ? abiOverride : app.info.primaryCpuAbi;
            if (requiredAbi == null) {
                requiredAbi = Build.SUPPORTED_ABIS[0];
            }

            String instructionSet = null;
            if (app.info.primaryCpuAbi != null) {
                instructionSet = VMRuntime.getInstructionSet(app.info.primaryCpuAbi);
            }

            // 设置进程的 gids，运行的环境等等！
            app.gids = gids;
            app.requiredAbi = requiredAbi;
            app.instructionSet = instructionSet;

            
            // 设置新进程创建后的入口："android.app.ActivityThread"
            boolean isActivityProcess = (entryPoint == null);
            if (entryPoint == null) entryPoint = "android.app.ActivityThread";
            Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "Start proc: " +
                    app.processName);

            checkTime(startTime, "startProcess: asking zygote to start proc");

            // 这里很关键！！调用了 Process.Start 方法，创建新进程，并返回创建结果！
            // 这类返回 startResult 包含子进程的 pid 等信息！
            Process.ProcessStartResult startResult = Process.start(entryPoint,
                    app.processName, uid, uid, gids, debugFlags, mountExternal,
                    app.info.targetSdkVersion, app.info.seinfo, requiredAbi, instructionSet,
                    app.info.dataDir, entryPointArgs);


            checkTime(startTime, "startProcess: returned from zygote!");
            Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);

            mBatteryStatsService.noteProcessStart(app.processName, app.info.uid);
            checkTime(startTime, "startProcess: done updating battery stats");

            EventLog.writeEvent(EventLogTags.AM_PROC_START,
                    UserHandle.getUserId(uid), startResult.pid, uid,
                    app.processName, hostingType,
                    hostingNameStr != null ? hostingNameStr : "");

            try {
                AppGlobals.getPackageManager().logAppProcessStartIfNeeded(app.processName, app.uid,
                        app.info.seinfo, app.info.sourceDir, startResult.pid);
            } catch (RemoteException ex) {
                // Ignore
            }

            if (app.persistent) { // 如果是 persistent 的进程，启动 WathDog 进行监控！
                Watchdog.getInstance().processStarted(app.processName, startResult.pid);
            }

            checkTime(startTime, "startProcess: building log message");
            StringBuilder buf = mStringBuilder;
            buf.setLength(0);
            buf.append("Start proc ");
            buf.append(startResult.pid);
            buf.append(':');
            buf.append(app.processName);
            buf.append('/');
            UserHandle.formatUid(buf, uid);
            if (!isActivityProcess) {
                buf.append(" [");
                buf.append(entryPoint);
                buf.append("]");
            }
            buf.append(" for ");
            buf.append(hostingType);
            if (hostingNameStr != null) {
                buf.append(" ");
                buf.append(hostingNameStr);
            }
            Slog.i(TAG, buf.toString());
            
            // 根据创建进程的返回结果设置对应的 ProcessRecord 的参数：pid 等等！
            app.setPid(startResult.pid);
            app.usingWrapper = startResult.usingWrapper;
            app.removed = false;
            app.killed = false;
            app.killedByAm = false;
            checkTime(startTime, "startProcess: starting to update pids map");

            ProcessRecord oldApp;
            synchronized (mPidsSelfLocked) {
                oldApp = mPidsSelfLocked.get(startResult.pid);
            }

            // 如果已经有一个应用进程占用了分配的 pid，那就对这个进程进行清理！
            if (oldApp != null && !app.isolated) {
                // Clean up anything relating to this pid first
                Slog.w(TAG, "Reusing pid " + startResult.pid
                        + " while app is still mapped to it");
                cleanUpApplicationRecordLocked(oldApp, false, false, -1,
                        true /*replacingPid*/);
            }

            synchronized (mPidsSelfLocked) {

                // 将新创建的进程加入到 mPidsSelfLocked，用于 AMS 进行管理！ 
                this.mPidsSelfLocked.put(startResult.pid, app);
                if (isActivityProcess) {
                    Message msg = mHandler.obtainMessage(PROC_START_TIMEOUT_MSG);
                    msg.obj = app;
                    mHandler.sendMessageDelayed(msg, startResult.usingWrapper
                            ? PROC_START_TIMEOUT_WITH_WRAPPER : PROC_START_TIMEOUT);
                }
            }
            checkTime(startTime, "startProcess: done updating pids map");

        } catch (RuntimeException e) {
            Slog.e(TAG, "Failure starting process " + app.processName, e);

            // 进程启动失败：比如，进程被冻结了等等，就调用  forceStopPackageLocked 清理进程！
            forceStopPackageLocked(app.info.packageName, UserHandle.getAppId(app.uid), false,
                    false, true, false, false, UserHandle.getUserId(app.userId), "start failure");
        }
    }

```
这个方法很长，我们回顾一下它的主要作用：

- 先将进程的 pid 设置为 0，
- 根据不同参数，设置相应的 debugFlags，比如在 AndroidManifest.xml 中设置 androidd:debuggable 为 true，代表 app 运行在 debug 模式，则增加 debugger 标识以及开启 JNI check 功能等等。
- 调用 **Process.start** 进入 Zygote，来创建新进程，并返回创建的结果 ProcessStartResult 。
- 根据 fork 的结果，再次设置 ProcessRecord 的成员变量， 包括 pid， killded，killedByAm 等等属性！

可以看出，这个方法的关键，就在于 Process.start 方法，是用来创建进程的，这个方法我们在第二篇：进程的创建一文中有讲，这里因为篇幅关系，就省略了！

接着就要进入这个新进程的 ActivityThread.main 方法了：


# 5 ActivityThread.main
这个方法是应用程序进程的入口：
```java
    public static void main(String[] args) {
        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ActivityThreadMain");
        SamplingProfilerIntegration.start();

        // CloseGuard defaults to true and can be quite spammy.  We
        // disable it here, but selectively enable it later (via
        // StrictMode) on debug builds, but using DropBox, not logs.
        CloseGuard.setEnabled(false);

        // 将当前进程所在 userId 赋值给 sCurrentUser
        Environment.initForCurrentUser();

        // Set the reporter for event logging in libcore
        EventLogger.setReporter(new EventLoggingReporter());

        // Make sure TrustedCertificateStore looks in the right place for CA certificates
        final File configDir = Environment.getUserConfigDirectory(UserHandle.myUserId());
        TrustedCertificateStore.setDefaultUserDirectory(configDir);

        Process.setArgV0("<pre-initialized>");
        // 创建主线程的 Looper 对象！
        Looper.prepareMainLooper();
        
        // 创建该进程的  ActivityThread 对象！
        ActivityThread thread = new ActivityThread();
        
        // attach 到 ActiviyManagerService ，保持进程和 SystemServer 的 binder 通信！
        // 这里我们下面重点讲！
        thread.attach(false);

        if (sMainThreadHandler == null) {
            // 获得主线程的 Handler 对象 mH；
            sMainThreadHandler = thread.getHandler();
        }

        if (false) { // 当设置为 true 时，可打开消息队列的 debug log 信息
            Looper.myLooper().setMessageLogging(new
                    LogPrinter(Log.DEBUG, "ActivityThread"));
        }

        // End of event ActivityThreadMain.
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
        
        // 进入消息循环，没消息进入休眠状态，有消息进入唤醒状态!
        Looper.loop();

        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
```
这个方法的作用有如下：

- 创建主线程的 Looper 对象，主线程的 Looper 对象是在进程创建后自动创建的！
- 创建该进程的 ActivityThread 对象，每一个进程都有一个 ActivityThread 对象！
  - mAppThread = new ApplicationThread()，创建 ApplicationThread 对象，他是应用进程和 AMS 通信的桥梁；
  - mLooper = Looper.myLooper()，获得主线程的 Looper 对象；
  - mH = new H()，H 继承于 Handler 对象，主线程的 Handler，用于处理组件的生命周期！
  - ... ... ...
- 获得主线程的 Handler 对象 mH：
- 进入消息循环：Looper.loop，没有消息进入休眠，有消息就被唤醒，处理消息！

> **注意**：这里要先说一下：ActivityThread 和 ApplicationThread 对象，二者都不是线程对象！一个应用程序进程对应一个 ActivityThread 实例，应用程序进程由 ActivityThread.main 打开消息循环；同时，一个应用程序进程也对应一个 ApplicationThread 对象，此对象是 ActivityThread 与 ActivityManagerService 连接的桥梁！



# 6 ActivityThread.attach
参数传递：

- system：表示是否是系统进程的 ActivityThread，调用传入 false。

```java
    private void attach(boolean system) {
        // 表示当前进程的 ActivityThread
        sCurrentActivityThread = this;
        mSystemThread = system;
        
        if (!system) { // 应用进程，进入这个分支。
            ViewRootImpl.addFirstDrawHandler(new Runnable() {
                @Override
                public void run() {

                    // 开启虚拟机的 jit 即时编译功能
                    ensureJitEnabled();
                }
            });
            android.ddm.DdmHandleAppName.setAppName("<pre-initialized>",
                                                    UserHandle.myUserId());
            RuntimeInit.setApplicationObject(mAppThread.asBinder());

            // 获得 AMS 
            final IActivityManager mgr = ActivityManagerNative.getDefault();
            try {
            
                // 调用 ams 的 attachApplication 方法
                mgr.attachApplication(mAppThread);
                
            } catch (RemoteException ex) {
                throw ex.rethrowFromSystemServer();
            }
            // Watch for getting close to heap limit.
            // 观察是否快接近 heap 的上限
            BinderInternal.addGcWatcher(new Runnable() {
                @Override public void run() {
                    if (!mSomeActivitiesChanged) {
                        return;
                    }
                    Runtime runtime = Runtime.getRuntime();
                    long dalvikMax = runtime.maxMemory();
                    long dalvikUsed = runtime.totalMemory() - runtime.freeMemory();
                    
                    // 当进程虚拟机的已用内存超过最大内存的 3/4
                    if (dalvikUsed > ((3*dalvikMax)/4)) {
                    
                        if (DEBUG_MEMORY_TRIM) Slog.d(TAG, "Dalvik max=" + (dalvikMax/1024)
                                + " total=" + (runtime.totalMemory()/1024)
                                + " used=" + (dalvikUsed/1024));
                                
                        mSomeActivitiesChanged = false;
                        try {
                            // 调用 AMS 的 releaseSomeActivities 方法释放进程的内存！
                            mgr.releaseSomeActivities(mAppThread);
                        } catch (RemoteException e) {
                            throw e.rethrowFromSystemServer();
                        }
                    }
                }
            });
        } else { // 这里是处理系统进程的：SystemServer，这里不看
            android.ddm.DdmHandleAppName.setAppName("system_process",
                    UserHandle.myUserId());
            try {
                mInstrumentation = new Instrumentation();
                ContextImpl context = ContextImpl.createAppContext(this, getSystemContext().mPackageInfo);

                // 创建系统进程的 Application 对象！        
                mInitialApplication = context.mPackageInfo.makeApplication(true, null);
                mInitialApplication.onCreate();
            } catch (Exception e) {
                throw new RuntimeException(
                        "Unable to instantiate Application():" + e.toString(), e);
            }
        }

        // add dropbox logging to libcore
        DropBox.setReporter(new DropBoxReporter());
        // 添加 Config 回调接口
        ViewRootImpl.addConfigCallback(new ComponentCallbacks2() {
            @Override
            public void onConfigurationChanged(Configuration newConfig) {
                synchronized (mResourcesManager) {
                    // We need to apply this change to the resources
                    // immediately, because upon returning the view
                    // hierarchy will be informed about it.
                    if (mResourcesManager.applyConfigurationToResourcesLocked(newConfig, null)) {
                        updateLocaleListFromAppContext(mInitialApplication.getApplicationContext(),
                                mResourcesManager.getConfiguration().getLocales());

                        // This actually changed the resources!  Tell
                        // everyone about it.
                        if (mPendingConfiguration == null ||
                                mPendingConfiguration.isOtherSeqNewer(newConfig)) {
                            mPendingConfiguration = newConfig;

                            sendMessage(H.CONFIGURATION_CHANGED, newConfig);
                        }
                    }
                }
            }
            @Override
            public void onLowMemory() {
            }
            @Override
            public void onTrimMemory(int level) {
            }
        });
    }
```
对于应用进程的 attach 主要流程：

- 创建线程来开启虚拟机的 JIT 即时编译。
- **通过 binder，调用到 AMS.attachApplication，其参数 mAppThread 的数据类型为 ApplicationThread**。
- 观察是否快接近 heap 的上限，当已用内存超过最大内存的 3 / 4，则请求释放内存空间。
- 添加 dropbox 日志到 libcore。
- 添加 Config 回调接口。

# 7 ActivityManagerProxy.attachApplication

- 参数传递：IApplicationThread app，这里传入的是 ApplicationThread 对象！

```
    public void attachApplication(IApplicationThread app) throws RemoteException
    {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IActivityManager.descriptor);
        data.writeStrongBinder(app.asBinder());
        // binder 通信！
        mRemote.transact(ATTACH_APPLICATION_TRANSACTION, data, reply, 0);
        reply.readException();
        data.recycle();
        reply.recycle();
    }
```
以上是在应用进程里面，下面就要通过 Binder 通信，进入系统进程中：
此处：descriptor = “android.app.IActivityManager”，用来校验使用的！

# 8 ActivityManagerNative.onTransact
```java

    @Override
    public boolean onTransact(int code, Parcel data, Parcel reply, int flags)
            throws RemoteException {
        switch (code) {

        ... ... ... ...

        case ATTACH_APPLICATION_TRANSACTION: {

            // 校验 binder 通信是否匹配!
            data.enforceInterface(IActivityManager.descriptor);
            
            // 通过 asInterface 接口，将 ApplicationThread 对象转为 ApplicationTreadProxy 对象！
            IApplicationThread app = ApplicationThreadNative.asInterface(
                    data.readStrongBinder());
                    
            if (app != null) {

                // 调用 IActivityManager 的方法，接下来，进入 AMS！
                attachApplication(app);
            }
            reply.writeNoException();
            return true;
        }
    
        ... ... ... ...
        
        }
    }
```

通过 Binder 通信，调用 AMS 的方法：！
# 9 ActivityManagerService.attachApplication
```java
    @Override
    public final void attachApplication(IApplicationThread thread) {
        synchronized (this) {

            // 获得调用方的 pid，这里是应用进程的 pid！
            int callingPid = Binder.getCallingPid();
            final long origId = Binder.clearCallingIdentity();
    
            // 调用 attachApplicationLocked 继续处理！
            attachApplicationLocked(thread, callingPid);
            Binder.restoreCallingIdentity(origId);
        }
    }
```
这里简单，再获得应用进程的 pid！

# 10 ActivityManagerService.attachApplicationLocked
传入参数：

- ApplicationThreadProxy 对象，应用程序进程的 pid

删掉了部分的注释！

```java
    private final boolean attachApplicationLocked(IApplicationThread thread,
            int pid) {

        ProcessRecord app;
        if (pid != MY_PID && pid >= 0) { // 显然，调用方进程不是 SystemServer 进程！
            synchronized (mPidsSelfLocked) {

                app = mPidsSelfLocked.get(pid); // 获得该进程的 ProcessRecord 对象！

            }
        } else {
            app = null;
        }

        if (app == null) {  // 如果获得的 ProcessRecord 为 null，这里是属于异常情况！
           
            Slog.w(TAG, "No pending application record for pid " + pid
                    + " (IApplicationThread " + thread + "); dropping process");
            EventLog.writeEvent(EventLogTags.AM_DROP_PROCESS, pid);

            if (pid > 0 && pid != MY_PID) { // 如果调用方不是 SystemServer 进程，杀掉进程组！

                Process.killProcessQuiet(pid);
                //TODO: killProcessGroup(app.info.uid, pid);
            } else {
                try {

                    thread.scheduleExit(); 
                    
                } catch (Exception e) {
                    // Ignore exceptions.
                }
            }
            return false;
        }

        // 如果 ProcessRecord 不为 null，且 ProcessRecord.thread 不为 null
        // 说明该 ProcessRecord 已经 attach 了另外一个进程，需要立即清空这个 ProcessRecord！
        if (app.thread != null) {
            handleAppDiedLocked(app, true, true);
        }

        // Tell the process all about itself.

        if (DEBUG_ALL) Slog.v(
                TAG, "Binding process pid " + pid + " to record " + app);

        // 获得进程的名称!
        final String processName = app.processName;
        try {

            // 创建进程死亡通知对象，并绑定死亡通知对象！
            AppDeathRecipient adr = new AppDeathRecipient(
                    app, pid, thread);
            thread.asBinder().linkToDeath(adr, 0);
            app.deathRecipient = adr;

        } catch (RemoteException e) {

            app.resetPackageList(mProcessStats);
            startProcessLocked(app, "link fail", processName);
            return false;
        }

        EventLog.writeEvent(EventLogTags.AM_PROC_BOUND, app.userId, app.pid, app.processName);

        // 重置进程的信息！ 
        // 这里将传入的 ApplicationThreadProxy 对象赋给 app.thread，完成 attach 操作！！
        // 这里进程开始处于 active 状态!
        app.makeActive(thread, mProcessStats);

        app.curAdj = app.setAdj = app.verifiedAdj = ProcessList.INVALID_ADJ;
        app.curSchedGroup = app.setSchedGroup = ProcessList.SCHED_GROUP_DEFAULT;
        app.forcingToForeground = null;
        updateProcessForegroundLocked(app, false, false);
        app.hasShownUi = false;
        app.debugging = false;
        app.cached = false;
        app.killedByAm = false;

        app.unlocked = StorageManager.isUserKeyUnlocked(app.userId);

        mHandler.removeMessages(PROC_START_TIMEOUT_MSG, app);
        
        // 进程处于 ready 状态或者该进程为 FLAG_PERSISTENT 进程，即常驻进程，则 normalMode 为 true！
        boolean normalMode = mProcessesReady || isAllowedWhileBooting(app.info);

        // 获得该进程中的 providers！
        List<ProviderInfo> providers = normalMode ? generateApplicationProvidersLocked(app) : null;
        
        // 检查新进程是否存在正在启动中的 provider，
        // 如果有，则超时 10s 后发送 CONTENT_PROVIDER_PUBLISH_TIMEOUT_MSG 消息！
        if (providers != null && checkAppInLaunchingProvidersLocked(app)) {
            Message msg = mHandler.obtainMessage(CONTENT_PROVIDER_PUBLISH_TIMEOUT_MSG);
            msg.obj = app;
            mHandler.sendMessageDelayed(msg, CONTENT_PROVIDER_PUBLISH_TIMEOUT);
        }

        if (!normalMode) {
            Slog.i(TAG, "Launching preboot mode app: " + app);
        }

        if (DEBUG_ALL) Slog.v(
            TAG, "New app record " + app
            + " thread=" + thread.asBinder() + " pid=" + pid);

        try {
            int testMode = IApplicationThread.DEBUG_OFF;
            if (mDebugApp != null && mDebugApp.equals(processName)) {
                testMode = mWaitForDebugger
                    ? IApplicationThread.DEBUG_WAIT
                    : IApplicationThread.DEBUG_ON;
                app.debugging = true;
                if (mDebugTransient) {
                    mDebugApp = mOrigDebugApp;
                    mWaitForDebugger = mOrigWaitForDebugger;
                }
            }

            String profileFile = app.instrumentationProfileFile;
            ParcelFileDescriptor profileFd = null;
            int samplingInterval = 0;
            boolean profileAutoStop = false;
            if (mProfileApp != null && mProfileApp.equals(processName)) {
                mProfileProc = app;
                profileFile = mProfileFile;
                profileFd = mProfileFd;
                samplingInterval = mSamplingInterval;
                profileAutoStop = mAutoStopProfiler;
            }

            boolean enableTrackAllocation = false;
            if (mTrackAllocationApp != null && mTrackAllocationApp.equals(processName)) {
                enableTrackAllocation = true;
                mTrackAllocationApp = null;
            }

            // If the app is being launched for restore or full backup, set it up specially
            boolean isRestrictedBackupMode = false;
            if (mBackupTarget != null && mBackupAppName.equals(processName)) {
                isRestrictedBackupMode = mBackupTarget.appInfo.uid >= Process.FIRST_APPLICATION_UID
                        && ((mBackupTarget.backupMode == BackupRecord.RESTORE)
                                || (mBackupTarget.backupMode == BackupRecord.RESTORE_FULL)
                                || (mBackupTarget.backupMode == BackupRecord.BACKUP_FULL));
            }

            if (app.instrumentationClass != null) {
                notifyPackageUse(app.instrumentationClass.getPackageName(),
                                 PackageManager.NOTIFY_PACKAGE_USE_INSTRUMENTATION);
            }
            if (DEBUG_CONFIGURATION) Slog.v(TAG_CONFIGURATION, "Binding proc "
                    + processName + " with config " + mConfiguration);

            // 获得进程的所有者应用程序的信息！        
            ApplicationInfo appInfo = app.instrumentationInfo != null
                    ? app.instrumentationInfo : app.info;
                    
            app.compat = compatibilityInfoForPackageLocked(appInfo);
            if (profileFd != null) {
                profileFd = profileFd.dup();
            }
            ProfilerInfo profilerInfo = profileFile == null ? null
                    : new ProfilerInfo(profileFile, profileFd, samplingInterval, profileAutoStop);
            
            // 这里很关键了，之前通过 attach 的方式，实现了从 SystemServer 进程到应用程序进程的单向 bind 
            // 通信，而这里的 bind 操作最终的结果，是实现了 SystemServer 进程和应用程序进程的双向 bind 通信！
            thread.bindApplication(processName, appInfo, providers, app.instrumentationClass,
                    profilerInfo, app.instrumentationArguments, app.instrumentationWatcher,
                    app.instrumentationUiAutomationConnection, testMode,
                    mBinderTransactionTrackingEnabled, enableTrackAllocation,
                    isRestrictedBackupMode || !normalMode, app.persistent,
                    new Configuration(mConfiguration), app.compat,
                    getCommonServicesLocked(app.isolated),
                    mCoreSettingsObserver.getCoreSettingsLocked());
            
            // 更新进程 LRU 队列！
            updateLruProcessLocked(app, false, null);

            app.lastRequestedGc = app.lastLowMemory = SystemClock.uptimeMillis();
        } catch (Exception e) {

            app.resetPackageList(mProcessStats);
            app.unlinkDeathRecipient();

            // 每当 bind 操作失败，则重新启动进程, 此处有可能会导致进程无限重启。
            startProcessLocked(app, "bind fail", processName);
            return false;
        }

        // 将当前已经被启动的新进程的 ProcessRecord 从正在启动的进程集合 mPersistentStartingProcesses！
        // 和等待启动的进程集合 mProcessesOnHold 中移除！
        mPersistentStartingProcesses.remove(app);
        mProcessesOnHold.remove(app);

        boolean badApp = false;
        boolean didSomething = false;

        if (normalMode) { // 如果是正常启动的情况下！
            try {

                // 检查是否有 activity 组件要在这个新进程中运行！
                if (mStackSupervisor.attachApplicationLocked(app)) {
                    didSomething = true;
                }
            } catch (Exception e) {
                Slog.wtf(TAG, "Exception thrown launching activities in " + app, e);
                
                // 检查异常，将 badApp 置为 true！
                badApp = true;
            }
        }

        if (!badApp) {
            try {

                 // 检查是否有 sevvce 组件要在这个新进程中运行！
                didSomething |= mServices.attachApplicationLocked(app, processName);
            } catch (Exception e) {
                Slog.wtf(TAG, "Exception thrown starting services in " + app, e);
                badApp = true;
            }
        }

        // 检查是否有 broadcast 组件要在这个新进程中运行！
        if (!badApp && isPendingBroadcastProcessLocked(pid)) {
            try {
                didSomething |= sendPendingBroadcastsLocked(app);
            } catch (Exception e) {
                // If the app died trying to launch the receiver we declare it 'bad'
                Slog.wtf(TAG, "Exception thrown dispatching broadcasts in " + app, e);
                badApp = true;
            }
        }

        // 检查是否在这个新进程中有下一个 backup 代理！
        if (!badApp && mBackupTarget != null && mBackupTarget.appInfo.uid == app.uid) {
            if (DEBUG_BACKUP) Slog.v(TAG_BACKUP,
                    "New app is backup target, launching agent for " + app);
            notifyPackageUse(mBackupTarget.appInfo.packageName,
                             PackageManager.NOTIFY_PACKAGE_USE_BACKUP);
            try {
                thread.scheduleCreateBackupAgent(mBackupTarget.appInfo,
                        compatibilityInfoForPackageLocked(mBackupTarget.appInfo),
                        mBackupTarget.backupMode);
            } catch (Exception e) {
                Slog.wtf(TAG, "Exception thrown creating backup agent in " + app, e);
                badApp = true;
            }
        }

        if (badApp) { // badApp 为 true，就杀掉这个进程
            app.kill("error during init", true);
            handleAppDiedLocked(app, false, true);
            
            return false;
        }

        if (!didSomething) { // 检查后发现，没有任何组件要运行在新进程，更新 OomAdj 的值！
            updateOomAdjLocked();
        }

        return true;
    }
```
我们来回顾一下这个方法：

- 根据 pid 从 mPidsSelfLocked 中查询到相应的 ProcessRecord 对象 app。
  - 当 app==null，意味着本次创建的进程不存在，则直接返回，这属于异常。
- 判断 app.thread 是否为 null，此时 thread 应该为 null，若不为 null 则表示该 app 附到上一个进程，则调用 handleAppDiedLocked 清理。
- 绑定死亡通知，当进程 pid 死亡时会通过 binder 死亡回调，来通知 system_server 进程死亡的消息。
- **重置 ProcessRecord 进程信息，并设置 app.thread 为新进程的 ApplicationThreadProxy ，attach 操作完成** 。
- app 进程存在正在启动中的 provider，则超时 10s 后发送 CONTENT_PROVIDER_PUBLISH_TIMEOUT_MSG 消息。
- 调用 thread.bindApplication 绑定应用进程，后面再进一步说明。
- 检查是否有 Provider，Activity，Service，Broadcast 等组件运行在该新进程。

## 10.1 new AppDeathRecipient

我们来看看死亡通知对象的创建！
```java
    private final class AppDeathRecipient implements IBinder.DeathRecipient {
        final ProcessRecord mApp; // 进程对应的 ProcessRecord 对象！
        final int mPid; // 进程的 pid
        final IApplicationThread mAppThread; // 进程的 ApplicationThreadProxy 对象！

        AppDeathRecipient(ProcessRecord app, int pid,
                IApplicationThread thread) {
            if (DEBUG_ALL) Slog.v(
                TAG, "New death recipient " + this
                + " for thread " + thread.asBinder());
            mApp = app;
            mPid = pid;
            mAppThread = thread;
        }

        @Override
        public void binderDied() {
            if (DEBUG_ALL) Slog.v(
                TAG, "Death received in " + this
                + " for thread " + mAppThread.asBinder());
            synchronized(ActivityManagerService.this) {
                appDiedLocked(mApp, mPid, mAppThread, true);
            }
        }
    }
```
当进程死亡后，其对应的死亡通知对象就会收到通知，其 binderDied 方法就会被调用，接着进入 appDiedLocked 方法：

## 10.1 AMS.appDiedLocked
参数 fromBinderDied 置为 true！
```java

    final void appDiedLocked(ProcessRecord app, int pid, IApplicationThread thread,
            boolean fromBinderDied) {
        // First check if this ProcessRecord is actually active for the pid.
        synchronized (mPidsSelfLocked) {
            ProcessRecord curProc = mPidsSelfLocked.get(pid);
            if (curProc != app) {
                Slog.w(TAG, "Spurious death for " + app + ", curProc for " + pid + ": " + curProc);
                return;
            }
        }

        BatteryStatsImpl stats = mBatteryStatsService.getActiveStatistics();
        synchronized (stats) {
            stats.noteProcessDiedLocked(app.info.uid, pid);
        }

        // 这里 killed 值为 false；
        if (!app.killed) {
            if (!fromBinderDied) {
                Process.killProcessQuiet(pid);
            }
            
            // 杀掉进程所在的进程组！
            killProcessGroup(app.uid, pid);
            app.killed = true;
        }

        // Clean up already done if the process has been re-started.
        if (app.pid == pid && app.thread != null &&
                app.thread.asBinder() == thread.asBinder()) {

            boolean doLowMem = app.instrumentationClass == null;
            boolean doOomAdj = doLowMem;
            if (!app.killedByAm) {
                Slog.i(TAG, "Process " + app.processName + " (pid " + pid
                        + ") has died");
                mAllowLowerMemLevel = true;
            } else {
                // Note that we always want to do oom adj to update our state with the
                // new number of procs.
                mAllowLowerMemLevel = false;
                doLowMem = false;
            }
            EventLog.writeEvent(EventLogTags.AM_PROC_DIED, app.userId, app.pid, app.processName);
            if (DEBUG_CLEANUP) Slog.v(TAG_CLEANUP,
                "Dying app: " + app + ", pid: " + pid + ", thread: " + thread.asBinder());
                
            // 杀掉进程后继续调用 handleAppDiedLocked 方法！
            handleAppDiedLocked(app, false, true);

            if (doOomAdj) {
                updateOomAdjLocked();
            }
            if (doLowMem) {
                doLowMemReportIfNeededLocked(app);
            }
        } else if (app.pid != pid) {
            // A new process has already been started.
            Slog.i(TAG, "Process " + app.processName + " (pid " + pid
                    + ") has died and restarted (pid " + app.pid + ").");
            EventLog.writeEvent(EventLogTags.AM_PROC_DIED, app.userId, app.pid, app.processName);
        } else if (DEBUG_PROCESSES) {
            Slog.d(TAG_PROCESSES, "Received spurious death notification for thread "
                    + thread.asBinder());
        }
    }
```

## 10.2 AMS.handleAppDiedLocked

- 参数 restarting 表示进程是否在重启，这里传入 false；
- 参数 allowRestart 表示是否允许进程重启，这里传入 true；
```java
    private final void handleAppDiedLocked(ProcessRecord app,
            boolean restarting, boolean allowRestart) {
        int pid = app.pid;

        boolean kept = cleanUpApplicationRecordLocked(app, restarting, allowRestart, -1,
                false /*replacingPid*/);

        if (!kept && !restarting) {
            removeLruProcessLocked(app);
            if (pid > 0) {
                ProcessList.remove(pid);
            }
        }

        if (mProfileProc == app) {
            clearProfilerLocked();
        }

        // Remove this application's activities from active lists.
        boolean hasVisibleActivities = mStackSupervisor.handleAppDiedLocked(app);

        app.activities.clear();

        if (app.instrumentationClass != null) {
            Slog.w(TAG, "Crash of app " + app.processName
                  + " running instrumentation " + app.instrumentationClass);

            Bundle info = new Bundle();
            info.putString("shortMsg", "Process crashed.");
            finishInstrumentationLocked(app, Activity.RESULT_CANCELED, info);
        }

        if (!restarting && hasVisibleActivities
                && !mStackSupervisor.resumeFocusedStackTopActivityLocked()) {

            // If there was nothing to resume, and we are not already restarting this process, but
            // there is a visible activity that is hosted by the process...  then make sure all
            // visible activities are running, taking care of restarting this process.
            mStackSupervisor.ensureActivitiesVisibleLocked(null, 0, !PRESERVE_WINDOWS);
        }
    }
```
### 10.2.1 AMS.cleanUpApplicationRecordLocked
```java
    private final boolean cleanUpApplicationRecordLocked(ProcessRecord app,
            boolean restarting, boolean allowRestart, int index, boolean replacingPid) {
        Slog.d(TAG, "cleanUpApplicationRecord -- " + app.pid);

        if (index >= 0) {
            removeLruProcessLocked(app);
            ProcessList.remove(app.pid);
        }

        mProcessesToGc.remove(app);
        mPendingPssProcesses.remove(app);

        // 取消掉这个进程所有已经可见的 Dialog！
        if (app.crashDialog != null && !app.forceCrashReport) {
            app.crashDialog.dismiss();
            app.crashDialog = null;
        }

        if (app.anrDialog != null) {
            app.anrDialog.dismiss();
            app.anrDialog = null;
        }

        if (app.waitDialog != null) {
            app.waitDialog.dismiss();
            app.waitDialog = null;
        }

        app.crashing = false;
        app.notResponding = false;

        app.resetPackageList(mProcessStats);
        app.unlinkDeathRecipient();
        app.makeInactive(mProcessStats);
        app.waitingToKill = null;
        app.forcingToForeground = null;
        updateProcessForegroundLocked(app, false, false);
        app.foregroundActivities = false;
        app.hasShownUi = false;
        app.treatLikeActivity = false;
        app.hasAboveClient = false;
        app.hasClientActivities = false;

        // 杀掉进程中所有的 Service！
        mServices.killServicesLocked(app, allowRestart);

        // restart 表示是否重启这个进程！
        boolean restart = false;

        // 移除这个进程的所有 publish 的 provider！
        for (int i = app.pubProviders.size() - 1; i >= 0; i--) {
            ContentProviderRecord cpr = app.pubProviders.valueAt(i);
            final boolean always = app.bad || !allowRestart;
            boolean inLaunching = removeDyingProviderLocked(app, cpr, always);
            if ((inLaunching || always) && cpr.hasConnectionOrHandle()) {

                // We left the provider in the launching list, need to
                // restart it.
                restart = true;
            }

            cpr.provider = null;
            cpr.proc = null;
        }
        app.pubProviders.clear();

        // 移除这个进程的所有正在启动的 provider！
        if (cleanupAppInLaunchingProvidersLocked(app, false)) {
            restart = true;
        }

        // 取消所有的 ContentProviderConnection 对象！
        if (!app.conProviders.isEmpty()) {
            for (int i = app.conProviders.size() - 1; i >= 0; i--) {

                ContentProviderConnection conn = app.conProviders.get(i);
                conn.provider.connections.remove(conn);
                stopAssociationLocked(app.uid, app.processName, conn.provider.uid,
                        conn.provider.name);
            }
            app.conProviders.clear();
        }

        // At this point there may be remaining entries in mLaunchingProviders
        // where we were the only one waiting, so they are no longer of use.
        // Look for these and clean up if found.
        // XXX Commented out for now.  Trying to figure out a way to reproduce
        // the actual situation to identify what is actually going on.
        if (false) {
            for (int i = mLaunchingProviders.size() - 1; i >= 0; i--) {
                ContentProviderRecord cpr = mLaunchingProviders.get(i);
                if (cpr.connections.size() <= 0 && !cpr.hasExternalProcessHandles()) {
                    synchronized (cpr) {
                        cpr.launchingApp = null;
                        cpr.notifyAll();
                    }
                }
            }
        }

        skipCurrentReceiverLocked(app);

        // Unregister any receivers.
        for (int i = app.receivers.size() - 1; i >= 0; i--) {
            removeReceiverLocked(app.receivers.valueAt(i));
        }
        app.receivers.clear();

        // If the app is undergoing backup, tell the backup manager about it
        if (mBackupTarget != null && app.pid == mBackupTarget.app.pid) {
            if (DEBUG_BACKUP || DEBUG_CLEANUP) Slog.d(TAG_CLEANUP, "App "
                    + mBackupTarget.appInfo + " died during backup");
            try {
                IBackupManager bm = IBackupManager.Stub.asInterface(
                        ServiceManager.getService(Context.BACKUP_SERVICE));
                bm.agentDisconnected(app.info.packageName);
            } catch (RemoteException e) {
                // can't happen; backup manager is local
            }
        }

        for (int i = mPendingProcessChanges.size() - 1; i >= 0; i--) {
            ProcessChangeItem item = mPendingProcessChanges.get(i);
            if (item.pid == app.pid) {
                mPendingProcessChanges.remove(i);
                mAvailProcessChanges.add(item);
            }
        }
        mUiHandler.obtainMessage(DISPATCH_PROCESS_DIED_UI_MSG, app.pid, app.info.uid,
                null).sendToTarget();

        // If the caller is restarting this app, then leave it in its
        // current lists and let the caller take care of it.
        if (restarting) {
            return false;
        }

        if (!app.persistent || app.isolated) {
            if (DEBUG_PROCESSES || DEBUG_CLEANUP) Slog.v(TAG_CLEANUP,
                    "Removing non-persistent process during cleanup: " + app);
            if (!replacingPid) {
                removeProcessNameLocked(app.processName, app.uid);
            }
            if (mHeavyWeightProcess == app) {
                mHandler.sendMessage(mHandler.obtainMessage(CANCEL_HEAVY_NOTIFICATION_MSG,
                        mHeavyWeightProcess.userId, 0));
                mHeavyWeightProcess = null;
            }
        } else if (!app.removed) {
            // This app is persistent, so we need to keep its record around.
            // If it is not already on the pending app list, add it there
            // and start a new process for it.
            if (mPersistentStartingProcesses.indexOf(app) < 0) {
                mPersistentStartingProcesses.add(app);
                restart = true;
            }
        }
        if ((DEBUG_PROCESSES || DEBUG_CLEANUP) && mProcessesOnHold.contains(app)) Slog.v(
                TAG_CLEANUP, "Clean-up removing on hold: " + app);
        mProcessesOnHold.remove(app);

        if (app == mHomeProcess) {
            mHomeProcess = null;
        }
        if (app == mPreviousProcess) {
            mPreviousProcess = null;
        }

        
        if (restart && !app.isolated) {
            // We have components that still need to be running in the
            // process, so re-launch it.
            if (index < 0) {
                ProcessList.remove(app.pid);
            }
            addProcessNameLocked(app);
            startProcessLocked(app, "restart", app.processName);
            return true;
        } else if (app.pid > 0 && app.pid != MY_PID) {
            // Goodbye!
            boolean removed;
            synchronized (mPidsSelfLocked) {
                mPidsSelfLocked.remove(app.pid);
                mHandler.removeMessages(PROC_START_TIMEOUT_MSG, app);
            }
            mBatteryStatsService.noteProcessFinish(app.processName, app.info.uid);
            if (app.isolated) {
                mBatteryStatsService.removeIsolatedUid(app.uid, app.info.uid);
            }
            app.setPid(0);
        }
        return false;
    }
```
从上面可以看到，如果已经进程是 persistent 进程，死亡通知对象接收到通知后会重启这个进程！

attach 结束了，接下来，是 SystemServer 进程 bind 应用程序进程！

# 11 ApplicationThreadProxy.bindApplication

这里是在 SystemServer 进程中：
```java
class ApplicationThreadProxy implements IApplicationThread {
    private final IBinder mRemote;

    public ApplicationThreadProxy(IBinder remote) {
        mRemote = remote;
    }

    public final IBinder asBinder() {
        return mRemote;
    }
    
    ... ... ...// 省略其他暂时不需要的方法！ 
    
    // 这里是 bind 操作！
    @Override
    public final void bindApplication(String packageName, ApplicationInfo info,
            List<ProviderInfo> providers, ComponentName testName, ProfilerInfo profilerInfo,
            Bundle testArgs, IInstrumentationWatcher testWatcher,
            IUiAutomationConnection uiAutomationConnection, int debugMode,
            boolean enableBinderTracking, boolean trackAllocation, boolean restrictedBackupMode,
            boolean persistent, Configuration config, CompatibilityInfo compatInfo,
            Map<String, IBinder> services, Bundle coreSettings) throws RemoteException {

        Parcel data = Parcel.obtain();
        data.writeInterfaceToken(IApplicationThread.descriptor);
        data.writeString(packageName);
        info.writeToParcel(data, 0);
        data.writeTypedList(providers);
        if (testName == null) {
            data.writeInt(0);
        } else {
            data.writeInt(1);
            testName.writeToParcel(data, 0);
        }
        if (profilerInfo != null) {
            data.writeInt(1);
            profilerInfo.writeToParcel(data, Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
        } else {
            data.writeInt(0);
        }
        data.writeBundle(testArgs);
        data.writeStrongInterface(testWatcher);
        data.writeStrongInterface(uiAutomationConnection);
        data.writeInt(debugMode);
        data.writeInt(enableBinderTracking ? 1 : 0);
        data.writeInt(trackAllocation ? 1 : 0);
        data.writeInt(restrictedBackupMode ? 1 : 0);
        data.writeInt(persistent ? 1 : 0);
        config.writeToParcel(data, 0);
        compatInfo.writeToParcel(data, 0);
        data.writeMap(services);
        data.writeBundle(coreSettings);

        // binder 通信！
        mRemote.transact(BIND_APPLICATION_TRANSACTION, data, null,
                IBinder.FLAG_ONEWAY);
        data.recycle();
    }
    
    ... ... ...
}
```
参数 Map<String, IBinder> services 是通过 AMS 的 getCommonServicesLocked 方法获得的：
```java
    private HashMap<String, IBinder> getCommonServicesLocked(boolean isolated) {
        if (isolated) {
            if (mIsolatedAppBindArgs == null) {
                mIsolatedAppBindArgs = new HashMap<>();
                mIsolatedAppBindArgs.put("package", ServiceManager.getService("package"));
            }
            return mIsolatedAppBindArgs;
        }

        if (mAppBindArgs == null) { // 对于非隔离进程，这里会获得一些常用服务的 binder 对象！！
            mAppBindArgs = new HashMap<>();

            // Setup the application init args
            mAppBindArgs.put("package", ServiceManager.getService("package"));
            mAppBindArgs.put("window", ServiceManager.getService("window"));
            mAppBindArgs.put(Context.ALARM_SERVICE,
                    ServiceManager.getService(Context.ALARM_SERVICE));
        }
        return mAppBindArgs;
    }
```


这里的 IApplicationThread.descriptor = "android.app.IApplicationThread"
ATP 经过 binder ipc 传递到 ATN 的 onTransact 方法。

# 12 ApplicationThreadNative.onTransact
进入到 ATN 的 onTransact 方法中，这里进入了应用程序的进程中了：
```java
public boolean onTransact(int code, Parcel data, Parcel reply, int flags)
        throws RemoteException {
    switch (code) {
    ...
    case BIND_APPLICATION_TRANSACTION:
    {
        data.enforceInterface(IApplicationThread.descriptor);
        // 获得应用程序的包名！
        String packageName = data.readString();
        ApplicationInfo info =
            ApplicationInfo.CREATOR.createFromParcel(data);
        List<ProviderInfo> providers =
            data.createTypedArrayList(ProviderInfo.CREATOR);
        ComponentName testName = (data.readInt() != 0)
            ? new ComponentName(data) : null;
        ProfilerInfo profilerInfo = data.readInt() != 0
                ? ProfilerInfo.CREATOR.createFromParcel(data) : null;
        Bundle testArgs = data.readBundle();

        IBinder binder = data.readStrongBinder();
        IInstrumentationWatcher testWatcher = IInstrumentationWatcher.Stub.asInterface(binder);

        binder = data.readStrongBinder();
        IUiAutomationConnection uiAutomationConnection =
                IUiAutomationConnection.Stub.asInterface(binder);

        int testMode = data.readInt();
        boolean openGlTrace = data.readInt() != 0;
        boolean restrictedBackupMode = (data.readInt() != 0);
        boolean persistent = (data.readInt() != 0);
        Configuration config = Configuration.CREATOR.createFromParcel(data);
        CompatibilityInfo compatInfo = CompatibilityInfo.CREATOR.createFromParcel(data);
        HashMap<String, IBinder> services = data.readHashMap(null);
        Bundle coreSettings = data.readBundle();

        // [见流程3.11]
        bindApplication(packageName, info, providers, testName, profilerInfo, testArgs,
                testWatcher, uiAutomationConnection, testMode, openGlTrace,
                restrictedBackupMode, persistent, config, compatInfo, services, coreSettings);
        return true;
    }
    ...
}
```

# 13 ApplicationThread.bindApplication
我们接着看：
```java
        public final void bindApplication(String processName, ApplicationInfo appInfo,
                List<ProviderInfo> providers, ComponentName instrumentationName,
                ProfilerInfo profilerInfo, Bundle instrumentationArgs,
                IInstrumentationWatcher instrumentationWatcher,
                IUiAutomationConnection instrumentationUiConnection, int debugMode,
                boolean enableBinderTracking, boolean trackAllocation,
                boolean isRestrictedBackupMode, boolean persistent, Configuration config,
                CompatibilityInfo compatInfo, Map<String, IBinder> services, Bundle coreSettings) {

            if (services != null) {

                // 这里将前面获得的系统公共 services 缓存到 ServiceManager 的 sCache 中, 
                // 减少 binder 检索服务的次数.
                ServiceManager.initServiceCache(services);
            }

            // 发送消息 H.SET_CORE_SETTINGS
            setCoreSettings(coreSettings);

            AppBindData data = new AppBindData(); // 初始化 AppBindData
            data.processName = processName;
            data.appInfo = appInfo;
            data.providers = providers;
            
            data.instrumentationName = instrumentationName;
            data.instrumentationArgs = instrumentationArgs;
            data.instrumentationWatcher = instrumentationWatcher;
            data.instrumentationUiAutomationConnection = instrumentationUiConnection;
            
            data.debugMode = debugMode;
            data.enableBinderTracking = enableBinderTracking;
            data.trackAllocation = trackAllocation;
            data.restrictedBackupMode = isRestrictedBackupMode;
            data.persistent = persistent;
            data.config = config;
            data.compatInfo = compatInfo;
            data.initProfilerInfo = profilerInfo;

            // 发送 H.BIND_APPLICATION 消息！
            sendMessage(H.BIND_APPLICATION, data);
        }
```
其中 setCoreSettings() 过程就是调用 sendMessage(H.SET_CORE_SETTINGS, coreSettings) 来向主线程发送 SET_CORE_SETTINGS 消息，如下：

```java
        public void setCoreSettings(Bundle coreSettings) {

            sendMessage(H.SET_CORE_SETTINGS, coreSettings);
        }
```

总结一下：bindApplication 方法的主要功能是依次向主线程发送消息 H.SET_CORE_SETTINGS 和 H.BIND_APPLICATION。

# 14 ActivityThread.H - 消息处理

进入到应用程序进程的主线程 Handler “H”：

```java
 private class H extends Handler {
 
        ... ... ...

        public void handleMessage(Message msg) {
        
                ... ... ...
                
                case SET_CORE_SETTINGS:
                
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "setCoreSettings");
                    
                    handleSetCoreSettings((Bundle) msg.obj);
                    
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                    break;
                    
                ... ... ... 
                
                case BIND_APPLICATION:

                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "bindApplication");
                    AppBindData data = (AppBindData)msg.obj;
                    
                    handleBindApplication(data);
                    
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                    break;
                    
                ... ... ...    
                
        } 
        
        ... ... ...
        
 }                   
```
我们继续来看吧：

## 14.1 消息 H.SET_CORE_SETTINGS
处理 H.SET_CORE_SETTINGS 消息，调用 handleSetCoreSettings 方法：
```java
    private void handleSetCoreSettings(Bundle coreSettings) {
        synchronized (mResourcesManager) {
            mCoreSettings = coreSettings;
        }

        onCoreSettingsChange();
    }

    private void onCoreSettingsChange() {
        boolean debugViewAttributes =
                mCoreSettings.getInt(Settings.Global.DEBUG_VIEW_ATTRIBUTES, 0) != 0;

        if (debugViewAttributes != View.mDebugViewAttributes) {
            View.mDebugViewAttributes = debugViewAttributes;

            // request all activities to relaunch for the changes to take place
            // 由于发生改变, 请求所有的 activities 重启启动！
            for (Map.Entry<IBinder, ActivityClientRecord> entry : mActivities.entrySet()) {
                requestRelaunchActivity(entry.getKey(), null, null, 0, false, null, null, false,
                        false /* preserveWindow */);
            }
        }
    }
```

## 14.2 消息 H.BIND_APPLICATION
处理 H.BIND_APPLICATION 消息，调用 handleBindApplication 方法：
```java
    private void handleBindApplication(AppBindData data) {
    
        // 将 UI 线程注册到 VMRuntime 中！
        VMRuntime.registerSensitiveThread();
        if (data.trackAllocation) {
            DdmVmInternal.enableRecentAllocations(true);
        }

        // 记录进程的开始时间！
        Process.setStartTimes(SystemClock.elapsedRealtime(), SystemClock.uptimeMillis());

        mBoundApplication = data;
        mConfiguration = new Configuration(data.config);
        mCompatConfiguration = new Configuration(data.config);

        mProfiler = new Profiler();
        if (data.initProfilerInfo != null) {
            mProfiler.profileFile = data.initProfilerInfo.profileFile;
            mProfiler.profileFd = data.initProfilerInfo.profileFd;
            mProfiler.samplingInterval = data.initProfilerInfo.samplingInterval;
            mProfiler.autoStopProfiler = data.initProfilerInfo.autoStopProfiler;
        }

        // 设置进程名, 也就是说进程名是在进程真正创建以后的 BIND_APPLICATION 过程中才取名!
        Process.setArgV0(data.processName);
        android.ddm.DdmHandleAppName.setAppName(data.processName,
                                                UserHandle.myUserId());

        if (data.persistent) { 
            // 如果常驻进程，在低内存设备, 不采用硬件加速绘制,以节省内存使用量！
            if (!ActivityManager.isHighEndGfx()) {
                ThreadedRenderer.disable(false);
            }
        }

        if (mProfiler.profileFd != null) {
            mProfiler.startProfiling();
        }

        // 如果应用程序的目标 sdk 小于等于 Honeycomb MR1
        // 设置 AsyncTask 默认的线程池为 THREAD_POOL_EXECUTOR！
        if (data.appInfo.targetSdkVersion <= android.os.Build.VERSION_CODES.HONEYCOMB_MR1) {
            AsyncTask.setDefaultExecutor(AsyncTask.THREAD_POOL_EXECUTOR);
        }

        Message.updateCheckRecycle(data.appInfo.targetSdkVersion);

        // 重置进程时区为系统时区！
        TimeZone.setDefault(null);
        // 设置语言环境列表！
        LocaleList.setDefault(data.config.getLocales());

        synchronized (mResourcesManager) {

            // 更新系统配置
            mResourcesManager.applyConfigurationToResourcesLocked(data.config, data.compatInfo);
            mCurDefaultDisplayDpi = data.config.densityDpi;

            // This calls mResourcesManager so keep it within the synchronized block.
            applyCompatConfiguration(mCurDefaultDisplayDpi);
        }
        
        // 创建应用程序的 LoadedApk 对象，每个进程都有一个。
        data.info = getPackageInfoNoCheck(data.appInfo, data.compatInfo);

        // 如果需要的话，设置该进程为像素
        if ((data.appInfo.flags&ApplicationInfo.FLAG_SUPPORTS_SCREEN_DENSITIES)
                == 0) {
            mDensityCompatMode = true;
            Bitmap.setDefaultDensity(DisplayMetrics.DENSITY_DEFAULT);
        }
        updateDefaultDensity();
        
        // 设置时间显示的格式！
        final boolean is24Hr = "24".equals(mCoreSettings.getString(Settings.System.TIME_12_24));
        DateFormat.set24HourTimePref(is24Hr);

        View.mDebugViewAttributes =
                mCoreSettings.getInt(Settings.Global.DEBUG_VIEW_ATTRIBUTES, 0) != 0;

        // 对于 userdebug/eng 编译方式的系统应用来说，使用 dropbox 工具来分析 log 堆栈！
        if ((data.appInfo.flags &
             (ApplicationInfo.FLAG_SYSTEM |
              ApplicationInfo.FLAG_UPDATED_SYSTEM_APP)) != 0) {
            StrictMode.conditionallyEnableDebugLogging();
        }

        // 如果应用的目标平台大于等于Honeycomb，就不允许在主线程使用网络相关功能
        // 不然会抛出 NetworkOnMainThreadException 异常！
        if (data.appInfo.targetSdkVersion >= Build.VERSION_CODES.HONEYCOMB) {
            StrictMode.enableDeathOnNetwork();
        }

        // 如果应用的目标平台不低于 Android N，那就不允许应用将 “file://” 这样的 Uri
        // 直接暴露出去，比如跨包传递等等，不然会抛出 FileUriExposedException
        if (data.appInfo.targetSdkVersion >= Build.VERSION_CODES.N) {
            StrictMode.enableDeathOnFileUriExposure();
        }

        NetworkSecurityPolicy.getInstance().setCleartextTrafficPermitted(
                (data.appInfo.flags & ApplicationInfo.FLAG_USES_CLEARTEXT_TRAFFIC) != 0);

        if (data.debugMode != IApplicationThread.DEBUG_OFF) {
            // XXX should have option to change the port.
            Debug.changeDebugPort(8100);
            if (data.debugMode == IApplicationThread.DEBUG_WAIT) {
                Slog.w(TAG, "Application " + data.info.getPackageName()
                      + " is waiting for the debugger on port 8100...");

                IActivityManager mgr = ActivityManagerNative.getDefault();
                try {
                    mgr.showWaitingForDebugger(mAppThread, true);
                } catch (RemoteException ex) {
                    throw ex.rethrowFromSystemServer();
                }

                Debug.waitForDebugger();

                try {
                    mgr.showWaitingForDebugger(mAppThread, false);
                } catch (RemoteException ex) {
                    throw ex.rethrowFromSystemServer();
                }

            } else {
                Slog.w(TAG, "Application " + data.info.getPackageName()
                      + " can be debugged on port 8100...");
            }
        }

        // 当处于调试模式,则运行应用生成 systrace 信息
        boolean isAppDebuggable = (data.appInfo.flags & ApplicationInfo.FLAG_DEBUGGABLE) != 0;
        Trace.setAppTracingAllowed(isAppDebuggable);
        if (isAppDebuggable && data.enableBinderTracking) {
            Binder.enableTracing();
        }

        /**
         * Initialize the default http proxy in this process for the reasons we set the time zone.
         */
        // 初始化默认的 http 代理！
        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "Setup proxies");
        final IBinder b = ServiceManager.getService(Context.CONNECTIVITY_SERVICE);
        if (b != null) {
            // In pre-boot mode (doing initial launch to collect password), not
            // all system is up.  This includes the connectivity service, so don't
            // crash if we can't get it.
            final IConnectivityManager service = IConnectivityManager.Stub.asInterface(b);
            try {
                final ProxyInfo proxyInfo = service.getProxyForNetwork(null);
                Proxy.setHttpProxySystemProperty(proxyInfo);
            } catch (RemoteException e) {
                Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                throw e.rethrowFromSystemServer();
            }
        }
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);

        // Instrumentation 是谷歌提供的一个基本测试类，具有监控交互的作用！
        // 每个进程都有一个 Instrumentation 对象，这里我们不看测试相关的！
        final InstrumentationInfo ii;
        if (data.instrumentationName != null) {
            ... ... ... ...
            
        } else {
            ii = null; // 进入此分支！
            
        }
        
        // 创建应用的上下文实例，更新本地语言平台列表！
        final ContextImpl appContext = ContextImpl.createAppContext(this, data.info);
        updateLocaleListFromAppContext(appContext,
                mResourcesManager.getConfiguration().getLocales());

        if (!Process.isIsolated() && !"android".equals(appContext.getPackageName())) {
    
            // 对于非隔离且所属包名不是 “android”（即系统进程）的进程
            // 创建缓存目录，来存放模板文集，和图形编译相关的代码！
            final File cacheDir = appContext.getCacheDir();
            if (cacheDir != null) {
                // Provide a usable directory for temporary files
                System.setProperty("java.io.tmpdir", cacheDir.getAbsolutePath());
            } else {
                Log.v(TAG, "Unable to initialize \"java.io.tmpdir\" property "
                        + "due to missing cache directory");
            }

            final Context deviceContext = appContext.createDeviceProtectedStorageContext();
            final File codeCacheDir = deviceContext.getCodeCacheDir();
            if (codeCacheDir != null) {
                setupGraphicsSupport(data.info, codeCacheDir);
            } else {
                Log.e(TAG, "Unable to setupGraphicsSupport due to missing code-cache directory");
            }
        }

        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "NetworkSecurityConfigProvider.install");
        NetworkSecurityConfigProvider.install(appContext);
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);

        // Continue loading instrumentation.
        if (ii != null) {
        
            ... ... ... ...
            
        } else {
        
            // 进入此分支，非测试情况，默认会创建一个 Instrumentation ！
            mInstrumentation = new Instrumentation(); 
        }

        if ((data.appInfo.flags&ApplicationInfo.FLAG_LARGE_HEAP) != 0) {
            dalvik.system.VMRuntime.getRuntime().clearGrowthLimit();
            
        } else {
            dalvik.system.VMRuntime.getRuntime().clampGrowthLimit();
            
        }

        // Allow disk access during application and provider setup. This could
        // block processing ordered broadcasts, but later processing would
        // probably end up doing the same disk access.
        final StrictMode.ThreadPolicy savedPolicy = StrictMode.allowThreadDiskWrites();
        try {
        
            // 通过反射，创建目标应用程序的 Application 对象！
            Application app = data.info.makeApplication(data.restrictedBackupMode, null);
    
            // 保存到 ActivityThread.mInitialApplication 中！
            mInitialApplication = app;

            // don't bring up providers in restricted mode; they may depend on the
            // app's custom Application class
            if (!data.restrictedBackupMode) {
                if (!ArrayUtils.isEmpty(data.providers)) {
                    installContentProviders(app, data.providers);
    
                    // For process that contains content providers, we want to
                    // ensure that the JIT is enabled "at some point".
                    mH.sendEmptyMessageDelayed(H.ENABLE_JIT, 10*1000);
                }
            }

            try {
                mInstrumentation.onCreate(data.instrumentationArgs);
                
            } catch (Exception e) {
                throw new RuntimeException(
                    "Exception thrown in onCreate() of "
                    + data.instrumentationName + ": " + e.toString(), e);
            }

            try {

                // 调用 Application.onCreate() 回调方法.
                mInstrumentation.callApplicationOnCreate(app);
            } catch (Exception e) {
                if (!mInstrumentation.onException(app, e)) {
                    throw new RuntimeException(
                        "Unable to create application " + app.getClass().getName()
                        + ": " + e.toString(), e);
                }
            }
        } finally {
            StrictMode.setThreadPolicy(savedPolicy);
        }
    }
```
总结一下，上面的方法的主要流程：

- 设置新进程的名字；
- 创建应用程序的 LoadedApk 对象；
- 创建应用的 ContextIpml 上下文实例；
- 创建应用程序的 Application 对象；
- 调用 Application.onCreate() 回调方法，初始化 app；

上面是这个方法的主要作用，下面我们一个一个看：

### 14.2.1 ContextImpl.createAppContext
创建应用程序的上下文实例，参数传递：

- ActivityThread mainThread：当前进程的 ActivityThread 对象！
- LoadedApk packageInfo：应用程序的信息对象;

```java
    static ContextImpl createAppContext(ActivityThread mainThread, LoadedApk packageInfo) {
        if (packageInfo == null) throw new IllegalArgumentException("packageInfo");

        return new ContextImpl(null, mainThread,
                packageInfo, null, null, false, null, null, Display.INVALID_DISPLAY);
    }
```
这里调用了 new ContextImpl，创建了 ContextImpl 对象：
```java
    private ContextImpl(ContextImpl container, ActivityThread mainThread,
            LoadedApk packageInfo, IBinder activityToken, UserHandle user, boolean restricted,
            Display display, Configuration overrideConfiguration, int createDisplayWithId) {
    
        mOuterContext = this;

        mMainThread = mainThread;
        mActivityToken = activityToken;
        mRestricted = restricted;

        if (user == null) {
            user = Process.myUserHandle();
        }
        mUser = user;

        mPackageInfo = packageInfo;
        mResourcesManager = ResourcesManager.getInstance();

        final int displayId = (createDisplayWithId != Display.INVALID_DISPLAY)
                ? createDisplayWithId
                : (display != null) ? display.getDisplayId() : Display.DEFAULT_DISPLAY;

        CompatibilityInfo compatInfo = null;
        if (container != null) {
            compatInfo = container.getDisplayAdjustments(displayId).getCompatibilityInfo();
        }
        if (compatInfo == null) {
            compatInfo = (displayId == Display.DEFAULT_DISPLAY)
                    ? packageInfo.getCompatibilityInfo()
                    : CompatibilityInfo.DEFAULT_COMPATIBILITY_INFO;
        }
        mDisplayAdjustments.setCompatibilityInfo(compatInfo);
        mDisplayAdjustments.setConfiguration(overrideConfiguration);

        mDisplay = (createDisplayWithId == Display.INVALID_DISPLAY) ? display
                : ResourcesManager.getInstance().getAdjustedDisplay(displayId, mDisplayAdjustments);

        Resources resources = packageInfo.getResources(mainThread);
        if (resources != null) {
            if (displayId != Display.DEFAULT_DISPLAY
                    || overrideConfiguration != null
                    || (compatInfo != null && compatInfo.applicationScale
                            != resources.getCompatibilityInfo().applicationScale)) {
                resources = mResourcesManager.getTopLevelResources(packageInfo.getResDir(),
                        packageInfo.getSplitResDirs(), packageInfo.getOverlayDirs(),
                        packageInfo.getApplicationInfo().sharedLibraryFiles, displayId,
                        overrideConfiguration, compatInfo);
            }
        }
        mResources = resources;

        if (container != null) {
            mBasePackageName = container.mBasePackageName;
            mOpPackageName = container.mOpPackageName;
        } else {
            mBasePackageName = packageInfo.mPackageName;
            ApplicationInfo ainfo = packageInfo.getApplicationInfo();
            if (ainfo.uid == Process.SYSTEM_UID && ainfo.uid != Process.myUid()) {

                // Special case: system components allow themselves to be loaded in to other
                // processes.  For purposes of app ops, we must then consider the context as
                // belonging to the package of this process, not the system itself, otherwise
                // the package+uid verifications in app ops will fail.
                mOpPackageName = ActivityThread.currentPackageName();
            } else {
                mOpPackageName = mBasePackageName;
            }
        }

        mContentResolver = new ApplicationContentResolver(this, mainThread, user);
    }

```

### 14.2.2 LoadedApk.makeApplication
调用了 LoadedApk.makeApplication 方法，利用反射创建应用的 Application 对象，参数传入：

- boolean forceDefaultAppClass：data.restrictedBackupMode
- Instrumentation instrumentation：null
```
    public Application makeApplication(boolean forceDefaultAppClass,
            Instrumentation instrumentation) {
        if (mApplication != null) {

            // 如果已经创建，就直接返回！
            return mApplication;
        }

        Application app = null;

        String appClass = mApplicationInfo.className;
        if (forceDefaultAppClass || (appClass == null)) {

            // 要反射的类！
            appClass = "android.app.Application";
        }

        try {
            java.lang.ClassLoader cl = getClassLoader();
            if (!mPackageName.equals("android")) { // 如果应用不是 framework-res.apk
                initializeJavaContextClassLoader();
            }
            
            // 创建应用的上下文实例！
            ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);

            // 利用上下文实例，创建应用的 Application 对象！
            app = mActivityThread.mInstrumentation.newApplication(
                    cl, appClass, appContext);
    
            // 这里是实现 Context 和 Application 的相互引用！
            appContext.setOuterContext(app);
            
        } catch (Exception e) {
            if (!mActivityThread.mInstrumentation.onException(app, e)) {
                throw new RuntimeException(
                    "Unable to instantiate application " + appClass
                    + ": " + e.toString(), e);
            }
        }
        mActivityThread.mAllApplications.add(app);
        mApplication = app;

        if (instrumentation != null) { //这里不看！
           ... ... ...
        }

        // Rewrite the R 'constants' for all library apks.
        SparseArray<String> packageIdentifiers = getAssets(mActivityThread)
                .getAssignedPackageIdentifiers();
                
        final int N = packageIdentifiers.size();
        for (int i = 0; i < N; i++) {
            final int id = packageIdentifiers.keyAt(i);
            if (id == 0x01 || id == 0x7f) {
                continue;
            }

            rewriteRValues(getClassLoader(), packageIdentifiers.valueAt(i), id);
        }

        return app;
    }

```
这里调用了 Instrumentation.newApplication 方法，创建 Application 的实例：
```java
    public Application newApplication(ClassLoader cl, String className, Context context)
            throws InstantiationException, IllegalAccessException, 
            ClassNotFoundException {
        
        // 继续调用 newApplication 方法！  
        return newApplication(cl.loadClass(className), context);
    }

    static public Application newApplication(Class<?> clazz, Context context)
            throws InstantiationException, IllegalAccessException, 
            ClassNotFoundException {

        // 创建 Application 对象！
        Application app = (Application)clazz.newInstance();

        // 将 context 和 Application 绑定，我们直接通过 getApplicationContext
        // 获得的就是这个 context 实例！
        app.attach(context);
        return app;
    }
```

### 14.2.3 Instrumentation.callApplicationOnCreate
这个方法很简单：
```java
    public void callApplicationOnCreate(Application app) {

        // 调用 onCreate 方法！
        app.onCreate();
    }
```
调用 Application 的 onCreate 方法！



到这里，bind 操作也完成了**，应用程序进程已经启动，并且完成了和 SystemServer 进程的双向 Binder 绑定，也创建了应用程序的 Application 对象，调用了Application.onCreate  方法**，接下来，就是启动指定的组件了！

回到 ActivityManagerService.attachApplicationLocked 方法中，bind 成功后，就会检测新进程中的组件了，组件的启动就在那里！！

关于组件的启动，以后在说！！


# 15 总结
其实，我们可以看到，进程的启动和创建，还是相当复杂的，其中涉及的细节很多，当时，阅读源码，最忌讳死扣细节，我们需要对框架有个整体的理解，下面，我们来通过几张图来总结一下，进程的启动！！

## 15.1 进程的创建和启动

下面我们用一张图来回顾下这个流程：

<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/23/Process 创建流程图-20221024234546526.png" alt="Process 创建流程图.png-33.2kB" style="zoom: 67%;" />


其中，check/start 指定组件，也存在系统进程和应用进程间的多次 Binder 通信，这里我们不重点讨论，在后面分析四大组件时，会详细分析他们的生命周期过程，会重点分析这个部分！

## 15.2 ActivityThread 关系图

