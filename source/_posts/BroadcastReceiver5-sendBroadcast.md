# BroadcastReceiver篇 5 - sendBroadcast 流程分析
title: BroadcastReceiver篇 5 - sendBroadcast 流程分析
date: 2016/05/10 20:46:25
catalog: true
categories:
- AndroidFramework源码分析
- BroadcastReceiver广播接收者
tags: BroadcastReceiver广播接收者
---

[toc]

本文基于 Android 7.1.1，分析发送广播的流程，转载请说明出处！

写本文的目的：

- 了解广播发送的流程；
- 了解 AMS 对广播接收者组件的管理；
- 记录自己的研究和分析；

# 0 综述

在 Android 系统中，有如下种类的广播，他们的发送方式各不一样，我们先来简单的了解一下：

**1. 普通广播**

发送普通广播的方法如下：
```java
sendBroadcast(...)
sendBroadcastMultiplePermissions(...)
sendBroadcastAsUser(...)
```

普通广播的发送参数 serialized 为 `false`，`sticky` 也为 `false`，表示普通广播既不是粘性的，也不是无序的；

对于普通广播，`AMS` 会针对其接收者的类型做不同的处理：

- 对于动态注册的广播接收者，`AMS` 会遍历对应的目标接收者集合，依次发送广播；
- 对于静态注册的广播接收者，`AMS` 在发送普通广播时，会按照有序的方式来进行分发；

由此可见，对于普通广播来说，动态注册的广播接收者会先接收到广播！

</br>

**2. 有序广播**

发送有序广播的方法如下：
```java
sendOrderedBroadcast(...)
sendOrderedBroadcastAsUser(...)
```
有序广播的发送参数 `serialized` 为 `true`，`sticky` 为 `false`；

对于有序广播，AMS 会收集能够匹配的静态注册和动态注册的广播接收者，按照优先级们，依次有序的发送广播！

AMS 收到上一个 `BroadcastReceiver` 处理完毕的消息返回后，才会将广播发送给下一个 `BroadcastReceiver`；其中，任意一个 `BroadcastReceiver`，都可以中止后续的广播分发流程；同时，上一个 `BroadcastReceiver` 可以将额外的信息添加到广播中。

对于发送给静态注册的广播接收者的普通广播，`AMS` 是将其按照发送有序广播的方式来进行发送的，因为静态注册的接收者由于其注册方式的特殊性，其所在进程可能没有被启动，所以如果采用并发的方式发送，那么系统需要在同一时间启动大量的进程，这显然不是合理的设计！

</br>

**3. 粘性广播**

发送粘性广播的方法如下：
```java
sendStickyBroadcast(...)
sendStickyBroadcastAsUser(...)
```
粘性广播的发送参数 `serialized` 为 `false`，`sticky` 为 `true`；

粘性广播的特性是系统会保存这个广播，当系统中又新注册了一个广播接收者时，该接收者会立刻接收到这个广播，粘性广播默认属于普通广播！

</br>

**4. 粘性有序广播**

发送粘性有序广播的方法如下：
```java
sendStickyOrderedBroadcast(...)
```
粘性广播的发送参数 `serialized` 为 `true`，`sticky` 为 `true`；

粘性有序广播本质上属于有序广播，只不过其具有 “粘性”！

</br>

本篇文章，会主要分析下广播的发送和处理过程：


# 1 发送者进程

## 1.1 ContextWapper.sendBroadcast
```java
    @Override
    public void sendBroadcast(Intent intent) {
        //【1.2】继续调用
        mBase.sendBroadcast(intent);
    }
    
    ... ... ... ...
```
这里的 `mBase` 对象是 `ContextImpl` 实例，其他的方法的调用方式也是一样的！

## 1.2 ContextImpl.sendBroadcast

进入 `ContextImpl`，我们看看不同的 `send` 方法都做了什么操作：
```java
    @Override
    public void sendBroadcast(Intent intent) {
        warnIfCallingFromSystemProcess();
        String resolvedType = intent.resolveTypeIfNeeded(getContentResolver());
        try {
            intent.prepareToLeaveProcess(this);
            //【1.3】开始发送广播！
            ActivityManagerNative.getDefault().broadcastIntent(
                    mMainThread.getApplicationThread(), intent, resolvedType, null,
                    Activity.RESULT_OK, null, null, null, AppOpsManager.OP_NONE, null, false, false,
                    getUserId());
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
```
可以看到，这里最终调用了 `AMS` 的 `broadcastIntent` 方法！

```java
    /** {@hide} */
    @Override
    public int getUserId() {
        return mUser.getIdentifier();
    }
```
getUserId 方法用来获得进程所在的设备用户 `id`，一般我们在应用程序进程中，只能获得当前的设备用户 `id`，这个 `id` 是进程启动时获得的！

## 1.3 ActivityManagerProxy.broadcastIntent

我们来看一下，发送普通广播的参数传递：

- `IApplicationThread caller`：发送者进程的 `ApplicationThread` 对象，用于跨进程通信！
- `Intent intent`：广播 `Intent`
- `String resolvedType`：
- `IIntentReceiver resultTo`：广播发送后的结果反馈对象，这里传入 `null`！
- `int resultCode`：广播发送后的结果反馈码，这里传入 `Activity.RESULT_OK`！
- `String resultData`： 广播发送后的结果反馈数据，这里传入 `null`！
- `Bundle map`：传入 null；
- `String[] requiredPermissions`：广播接受者需要的权限，传入 `null` 或者指定的权限数组！
- `int appOp`：发送广播相关的应用操作，传入 `AppOpsManager.OP_NONE` 或者指定的操作！
- `Bundle options`：广播携带的参数信息，传入 `null` 或者指定的 `Bundle` 对象！
- `boolean serialized`：广播是否是序列化的，普通的广播传入 `false`；
- `boolean sticky`： 广播是否是粘性的，传入 `false`；
- `int userId`：当前的设备用户 `id`；


```java
    public int broadcastIntent(IApplicationThread caller,
            Intent intent, String resolvedType, IIntentReceiver resultTo,
            int resultCode, String resultData, Bundle map,
            String[] requiredPermissions, int appOp, Bundle options, boolean serialized,
            boolean sticky, int userId) throws RemoteException {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IActivityManager.descriptor);
        data.writeStrongBinder(caller != null ? caller.asBinder() : null);
        intent.writeToParcel(data, 0);
        data.writeString(resolvedType);
        data.writeStrongBinder(resultTo != null ? resultTo.asBinder() : null);
        data.writeInt(resultCode);
        data.writeString(resultData);
        data.writeBundle(map);
        data.writeStringArray(requiredPermissions);
        data.writeInt(appOp);
        data.writeBundle(options);
        data.writeInt(serialized ? 1 : 0);
        data.writeInt(sticky ? 1 : 0);
        data.writeInt(userId);
        //【1】BROADCAST_INTENT_TRANSACTION binder 消息！
        mRemote.transact(BROADCAST_INTENT_TRANSACTION, data, reply, 0);
        reply.readException();
        int res = reply.readInt();
        reply.recycle();
        data.recycle();
        return res;
    }
```

下面，就是进入系统进程！

# 2 系统进程

首先，进入 `ActivityManagerNative` 中：
```java
    @Override
    public boolean onTransact(int code, Parcel data, Parcel reply, int flags)
            throws RemoteException {
        switch (code) {
        case BROADCAST_INTENT_TRANSACTION:
        {
            data.enforceInterface(IActivityManager.descriptor);
            IBinder b = data.readStrongBinder();

            //【1】获得代理对象 ApplicationThreadProxy！
            IApplicationThread app =
                b != null ? ApplicationThreadNative.asInterface(b) : null;
            Intent intent = Intent.CREATOR.createFromParcel(data);
            String resolvedType = data.readString();

            //【2】显然这里的 resultTo = null;
            b = data.readStrongBinder();
            IIntentReceiver resultTo =
                b != null ? IIntentReceiver.Stub.asInterface(b) : null;
            
            //【3】得到 Activity.RESULT_OK
            int resultCode = data.readInt();
            
            //【4】均为 null！
            String resultData = data.readString();
            Bundle resultExtras = data.readBundle();
            String[] perms = data.readStringArray();
            int appOp = data.readInt();
            Bundle options = data.readBundle();
            boolean serialized = data.readInt() != 0;
            boolean sticky = data.readInt() != 0;
            int userId = data.readInt();
            
            //【2.1】调用 AMS 的 broadcastIntent 方法继续发送广播！
            int res = broadcastIntent(app, intent, resolvedType, resultTo,
                    resultCode, resultData, resultExtras, perms, appOp,
                    options, serialized, sticky, userId);

            reply.writeNoException();
            reply.writeInt(res);
            return true;
        }
    }
```

我们进入 `AMS` 的方法中去看看：

## 2.1 AMService.broadcastIntent

这里的参数和之前保持一致，不用多说！

```java
    public final int broadcastIntent(IApplicationThread caller,
            Intent intent, String resolvedType, IIntentReceiver resultTo,
            int resultCode, String resultData, Bundle resultExtras,
            String[] requiredPermissions, int appOp, Bundle bOptions,
            boolean serialized, boolean sticky, int userId) {

        // 首先判断进程是否隔离！ 
        enforceNotIsolatedCaller("broadcastIntent");
        synchronized(this) {
            intent = verifyBroadcastLocked(intent);
            
            //【1】获得发送者所在进程！
            final ProcessRecord callerApp = getRecordForAppLocked(caller);
            final int callingPid = Binder.getCallingPid();
            final int callingUid = Binder.getCallingUid();
            final long origId = Binder.clearCallingIdentity();
            
            //【×2.2】调用 broadcastIntentLocked 方法继续处理广播的分发！
            int res = broadcastIntentLocked(callerApp,
                    callerApp != null ? callerApp.info.packageName : null,
                    intent, resolvedType, resultTo, resultCode, resultData, resultExtras,
                    requiredPermissions, appOp, bOptions, serialized, sticky,
                    callingPid, callingUid, userId);

            Binder.restoreCallingIdentity(origId);
            return res;
        }
    }
```

## 2.2 AMService.broadcastIntentLocked

这里我们再来看看参数传递：

- **`ProcessRecord callerApp`**：发送者所在的进程对象
- **`String callerPackage`**：发送者所在的应用包名
- **`Intent intent`**：广播 `Intent`
- **`String resolvedType`**：
- **`IIntentReceiver resultTo`**： 广播发送后的结果反馈对象；
- **`int resultCode`**：广播发送后的结果反馈码，这里传入 `Activity.RESULT_OK`；
- **`String resultData`**：广播发送后的结果反馈数据，这里传入 `null`；
- **`Bundle resultExtras`**： 传入 `null`；
- **`String[] requiredPermissions`**： 广播接受者需要的权限，传入 `null` 或者指定的权限数组；
- **`int appOp`**：发送广播相关的应用操作，传入 `AppOpsManager.OP_NONE` 或者指定的操作；
- **`Bundle bOptions`**：广播携带的参数信息，传入 `null` 或者指定的 `Bundle` 对象；
- **`boolean ordered`**：广播是否是有序广播还是普通广播；
- **`boolean sticky`**：广播是否是粘性的；
- **`int callingPid`**：发送者所在进程的 `pid`；
- **`int callingUid`**：发送者所在进程的 `uid`；
- **`int userId`**：目标设备用户 `id`；

```java
    final int broadcastIntentLocked(ProcessRecord callerApp,
            String callerPackage, Intent intent, String resolvedType,
            IIntentReceiver resultTo, int resultCode, String resultData,
            Bundle resultExtras, String[] requiredPermissions, int appOp, Bundle bOptions,
            boolean ordered, boolean sticky, int callingPid, int callingUid, int userId) {

        intent = new Intent(intent);

        //【1】给 intent 添加 Intent.FLAG_EXCLUDE_STOPPED_PACKAGES 标志位！
        // 禁止广播发送给被强制停止的应用！
        intent.addFlags(Intent.FLAG_EXCLUDE_STOPPED_PACKAGES);

        //【2】如果系统还没有完成启动，并且广播没有设置 FLAG_RECEIVER_BOOT_UPGRADE 标志位！
        // 那就给广播 Intent 添加 Intent.FLAG_RECEIVER_REGISTERED_ONLY 标志位
        // 这样，该广播就不会发送给静态注册的广播接收者了，也就不会拉起其所在的进程！
        if (!mProcessesReady && (intent.getFlags()&Intent.FLAG_RECEIVER_BOOT_UPGRADE) == 0) {
            intent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY);
        }

        if (DEBUG_BROADCAST_LIGHT) Slog.v(TAG_BROADCAST,
                (sticky ? "Broadcast sticky: ": "Broadcast: ") + intent
                + " ordered=" + ordered + " userid=" + userId);
        if ((resultTo != null) && !ordered) {
            Slog.w(TAG, "Broadcast " + intent + " not ordered but result callback requested!");
        }

        userId = mUserController.handleIncomingUser(callingPid, callingUid, userId, true,
                ALLOW_NON_FULL, "broadcast", callerPackage);

        //【3】判断接受该广播的设备用户是否运行，如果没有运行，就停止发送广播！
        if (userId != UserHandle.USER_ALL && !mUserController.isUserRunningLocked(userId, 0)) {
            if ((callingUid != Process.SYSTEM_UID
                    || (intent.getFlags() & Intent.FLAG_RECEIVER_BOOT_UPGRADE) == 0)
                    && !Intent.ACTION_SHUTDOWN.equals(intent.getAction())) {

                Slog.w(TAG, "Skipping broadcast of " + intent
                        + ": user " + userId + " is stopped");

                return ActivityManager.BROADCAST_FAILED_USER_STOPPED;
            }
        }
        
        //【4】如果广播带有附加参数，用于修改 doze 临时白名单，那么就要校验是否有
        // CHANGE_DEVICE_IDLE_TEMP_WHITELIST 权限！
        BroadcastOptions brOptions = null;
        if (bOptions != null) {
            brOptions = new BroadcastOptions(bOptions);
            if (brOptions.getTemporaryAppWhitelistDuration() > 0) {
                if (checkComponentPermission(
                        android.Manifest.permission.CHANGE_DEVICE_IDLE_TEMP_WHITELIST,
                        Binder.getCallingPid(), Binder.getCallingUid(), -1, true)
                        != PackageManager.PERMISSION_GRANTED) {
                    String msg = "Permission Denial: " + intent.getAction()
                            + " broadcast from " + callerPackage + " (pid=" + callingPid
                            + ", uid=" + callingUid + ")"
                            + " requires "
                            + android.Manifest.permission.CHANGE_DEVICE_IDLE_TEMP_WHITELIST;
                    Slog.w(TAG, msg);
                    throw new SecurityException(msg);
                }
            }
        }

        //【5】判断该广播是否是被保护的广播，被保护的广播只能由系统发送！
        final String action = intent.getAction();
        final boolean isProtectedBroadcast;

        try {
            //【×2.2.2】判断是否是保护广播！
            isProtectedBroadcast = AppGlobals.getPackageManager().isProtectedBroadcast(action);
        } catch (RemoteException e) {

            Slog.w(TAG, "Remote exception", e);
            return ActivityManager.BROADCAST_SUCCESS;
        }
        
        //【6】判断发送者进程是否是系统进程；
        final boolean isCallerSystem;
        switch (UserHandle.getAppId(callingUid)) {
            case Process.ROOT_UID:
            case Process.SYSTEM_UID:
            case Process.PHONE_UID:
            case Process.BLUETOOTH_UID:
            case Process.NFC_UID:
                isCallerSystem = true;
                break;
            default:
                isCallerSystem = (callerApp != null) && callerApp.persistent;
                break;
        }

        //【7】安全校验，禁止非系统的 app 发送受保护的广播！
        if (!isCallerSystem) {
            if (isProtectedBroadcast) {
                //【7.1】如果应用是非系统应用，但是广播是受保护的，那就抛出异常！
                String msg = "Permission Denial: not allowed to send broadcast "
                        + action + " from pid="
                        + callingPid + ", uid=" + callingUid;
                Slog.w(TAG, msg);
                throw new SecurityException(msg);

            } else if (AppWidgetManager.ACTION_APPWIDGET_CONFIGURE.equals(action)
                    || AppWidgetManager.ACTION_APPWIDGET_UPDATE.equals(action)) {
                //【7.2】如果不是受保护的广播，那么会对一些特殊广播做限制，只能发送给应用自身！
                if (callerPackage == null) {
                    String msg = "Permission Denial: not allowed to send broadcast "
                            + action + " from unknown caller.";
                    Slog.w(TAG, msg);
                    throw new SecurityException(msg);
                } else if (intent.getComponent() != null) {
                    // 如果指定了组件名，那么会校验下组件所属的包名是否和发送者一样！
                    if (!intent.getComponent().getPackageName().equals(
                            callerPackage)) {
                        String msg = "Permission Denial: not allowed to send broadcast "
                                + action + " to "
                                + intent.getComponent().getPackageName() + " from "
                                + callerPackage;
                        Slog.w(TAG, msg);
                        throw new SecurityException(msg);
                    }
                } else {
                    // 设置包名，限制目标包为自身！
                    intent.setPackage(callerPackage);
                }
            }
        }
        
        //【8】针对一些特殊 action 的广播进行处理，比如 uid 被移除，package 相关的广播，外置应用是否可用！
        if (action != null) {
            switch (action) {
                case Intent.ACTION_UID_REMOVED:
                case Intent.ACTION_PACKAGE_REMOVED:
                case Intent.ACTION_PACKAGE_CHANGED:
                case Intent.ACTION_EXTERNAL_APPLICATIONS_UNAVAILABLE:
                case Intent.ACTION_EXTERNAL_APPLICATIONS_AVAILABLE:
                case Intent.ACTION_PACKAGES_SUSPENDED:
                case Intent.ACTION_PACKAGES_UNSUSPENDED:
                    //【8.1】首先判断发送者是否有 android.Manifest.permission.BROADCAST_PACKAGE_REMOVED 权限！
                    // 没有的话，抛出异常！
                    if (checkComponentPermission(
                            android.Manifest.permission.BROADCAST_PACKAGE_REMOVED,
                            callingPid, callingUid, -1, true)
                            != PackageManager.PERMISSION_GRANTED) {
                        String msg = "Permission Denial: " + intent.getAction()
                                + " broadcast from " + callerPackage + " (pid=" + callingPid
                                + ", uid=" + callingUid + ")"
                                + " requires "
                                + android.Manifest.permission.BROADCAST_PACKAGE_REMOVED;
                        Slog.w(TAG, msg);
                        throw new SecurityException(msg);
                    }

                    switch (action) {
                        case Intent.ACTION_UID_REMOVED:
                            //【8.2】如果是 Intent.ACTION_UID_REMOVED，就从 mBatteryStatsService 
                            // 和 mAppOpsService 中移除这个 uid 的信息！
                            final Bundle intentExtras = intent.getExtras();
                            final int uid = intentExtras != null
                                    ? intentExtras.getInt(Intent.EXTRA_UID) : -1;
                            if (uid >= 0) {
                                mBatteryStatsService.removeUid(uid);
                                mAppOpsService.uidRemoved(uid);
                            }
                            break;

                        case Intent.ACTION_EXTERNAL_APPLICATIONS_UNAVAILABLE:
                            //【8.3】如果是扩展应用不可用的广播，那就清楚该应用的相关数据！
                            String list[] =
                                    intent.getStringArrayExtra(Intent.EXTRA_CHANGED_PACKAGE_LIST);

                            if (list != null && list.length > 0) {
                                for (int i = 0; i < list.length; i++) {
                                    forceStopPackageLocked(list[i], -1, false, true, true,
                                            false, false, userId, "storage unmount");
                                }
                                mRecentTasks.cleanupLocked(UserHandle.USER_ALL);
                                sendPackageBroadcastLocked(
                                        IApplicationThread.EXTERNAL_STORAGE_UNAVAILABLE, list,
                                        userId);
                            }
                            break;

                        case Intent.ACTION_EXTERNAL_APPLICATIONS_AVAILABLE:
                            //【8.4】如果是扩展应用可用的广播，那就只清楚最近任务中的数据！
                            mRecentTasks.cleanupLocked(UserHandle.USER_ALL);
                            break;

                        case Intent.ACTION_PACKAGE_REMOVED:
                        case Intent.ACTION_PACKAGE_CHANGED:
                            //【8.5】如果是 package 移除或改变的广播，进入这里！！
                            // CHANGED 广播一般是在某个组件 enabled or disabled 的时候！
                            Uri data = intent.getData();
                            String ssp;
                            if (data != null && (ssp=data.getSchemeSpecificPart()) != null) {
                                //【8.5.1】判断是否是覆盖安装，卸载！
                                boolean removed = Intent.ACTION_PACKAGE_REMOVED.equals(action);
                                final boolean replacing =
                                        intent.getBooleanExtra(Intent.EXTRA_REPLACING, false);
                                final boolean killProcess =
                                        !intent.getBooleanExtra(Intent.EXTRA_DONT_KILL_APP, false);

                                // 这里的 fullUninstall 表示是否完全卸载，卸载掉覆盖安装的 apk，replacing 为 true！
                                final boolean fullUninstall = removed && !replacing;
                                //【8.5.2】应用包被移除了，进入该分支！
                                if (removed) {
                                    if (killProcess) {
                                        //【8.5.2.1】杀掉进程！
                                        forceStopPackageLocked(ssp, UserHandle.getAppId(
                                                intent.getIntExtra(Intent.EXTRA_UID, -1)),
                                                false, true, true, false, fullUninstall, userId,
                                                removed ? "pkg removed" : "pkg changed");
                                    }

                                    final int cmd = killProcess
                                            ? IApplicationThread.PACKAGE_REMOVED
                                            : IApplicationThread.PACKAGE_REMOVED_DONT_KILL;

                                    sendPackageBroadcastLocked(cmd,
                                            new String[] {ssp}, userId);
                                    //【8.5.2.2】如果是卸载应用的话，移除该应用的所有信息：包括 uid 信息，uri 权限信息
                                    // 最近任务，以及电池使用信息！
                                    if (fullUninstall) {
                                        mAppOpsService.packageRemoved(
                                                intent.getIntExtra(Intent.EXTRA_UID, -1), ssp);
                                        removeUriPermissionsForPackageLocked(ssp, userId, true);

                                        removeTasksByPackageNameLocked(ssp, userId);

                                        if (mUnsupportedDisplaySizeDialog != null && ssp.equals(
                                                mUnsupportedDisplaySizeDialog.getPackageName())) {
                                            mUnsupportedDisplaySizeDialog.dismiss();
                                            mUnsupportedDisplaySizeDialog = null;
                                        }
                                        mCompatModePackages.handlePackageUninstalledLocked(ssp);
                                        mBatteryStatsService.notePackageUninstalled(ssp);
                                    }
                                } else {
                                    //【8.5.3】应用包没有被移除，进入该分支，如果需要是杀掉进程，那就执行 kill 操作！
                                    if (killProcess) {
                                        killPackageProcessesLocked(ssp, UserHandle.getAppId(
                                                intent.getIntExtra(Intent.EXTRA_UID, -1)),
                                                userId, ProcessList.INVALID_ADJ,
                                                false, true, true, false, "change " + ssp);
                                    }
                                    //【8.5.4】清除掉 disable 的包组件！
                                    cleanupDisabledPackageComponentsLocked(ssp, userId, killProcess,
                                            intent.getStringArrayExtra(
                                                    Intent.EXTRA_CHANGED_COMPONENT_NAME_LIST));
                                }
                            }
                            break;

                        case Intent.ACTION_PACKAGES_SUSPENDED:
                        case Intent.ACTION_PACKAGES_UNSUSPENDED:
                            //【8.6】如果是 SUSPENDED 或 UNSUSPENDED 的广播，进入这里！！
                            final boolean suspended = Intent.ACTION_PACKAGES_SUSPENDED.equals(
                                    intent.getAction());
                            final String[] packageNames = intent.getStringArrayExtra(
                                    Intent.EXTRA_CHANGED_PACKAGE_LIST);
                            final int userHandle = intent.getIntExtra(
                                    Intent.EXTRA_USER_HANDLE, UserHandle.USER_NULL);

                            synchronized(ActivityManagerService.this) {
                                mRecentTasks.onPackagesSuspendedChanged(
                                        packageNames, suspended, userHandle);
                            }
                            break;
                    }
                    break;

                case Intent.ACTION_PACKAGE_REPLACED:
                {
                    //【8.7】如果是 REPLACED （覆盖安装的时候）的广播，进入这里！！
                    final Uri data = intent.getData();
                    final String ssp;
                    if (data != null && (ssp = data.getSchemeSpecificPart()) != null) {
                        final ApplicationInfo aInfo =
                                getPackageManagerInternalLocked().getApplicationInfo(
                                        ssp,
                                        userId);
                        if (aInfo == null) {
                            Slog.w(TAG, "Dropping ACTION_PACKAGE_REPLACED for non-existent pkg:"
                                    + " ssp=" + ssp + " data=" + data);
                            return ActivityManager.BROADCAST_SUCCESS;
                        }
                        mStackSupervisor.updateActivityApplicationInfoLocked(aInfo);
                        sendPackageBroadcastLocked(IApplicationThread.PACKAGE_REPLACED,
                                new String[] {ssp}, userId);
                    }
                    break;
                }

                case Intent.ACTION_PACKAGE_ADDED:
                {
                    //【8.8】如果是 ADDED （全新安装的时候）的广播，进入这里！！
                    Uri data = intent.getData();
                    String ssp;
                    if (data != null && (ssp = data.getSchemeSpecificPart()) != null) {
                        final boolean replacing =
                                intent.getBooleanExtra(Intent.EXTRA_REPLACING, false);
                        mCompatModePackages.handlePackageAddedLocked(ssp, replacing);

                        try {
                            ApplicationInfo ai = AppGlobals.getPackageManager().
                                    getApplicationInfo(ssp, 0, 0);
                            mBatteryStatsService.notePackageInstalled(ssp,
                                    ai != null ? ai.versionCode : 0);
                        } catch (RemoteException e) {
                        }
                    }
                    break;
                }

                case Intent.ACTION_PACKAGE_DATA_CLEARED:
                {
                    //【8.9】如果是 PACKAGE_DATA_CLEARED （清楚数据）的广播，进入这里！！
                    Uri data = intent.getData();
                    String ssp;
                    if (data != null && (ssp = data.getSchemeSpecificPart()) != null) {
                        // Hide the "unsupported display" dialog if necessary.
                        if (mUnsupportedDisplaySizeDialog != null && ssp.equals(
                                mUnsupportedDisplaySizeDialog.getPackageName())) {
                            mUnsupportedDisplaySizeDialog.dismiss();
                            mUnsupportedDisplaySizeDialog = null;
                        }
                        mCompatModePackages.handlePackageDataClearedLocked(ssp);
                    }
                    break;
                }

                case Intent.ACTION_TIMEZONE_CHANGED:
                    // If this is the time zone changed action, queue up a message that will reset
                    // the timezone of all currently running processes. This message will get
                    // queued up before the broadcast happens.
                    mHandler.sendEmptyMessage(UPDATE_TIME_ZONE);
                    break;

                case Intent.ACTION_TIME_CHANGED:
                    // If the user set the time, let all running processes know.
                    final int is24Hour =
                            intent.getBooleanExtra(Intent.EXTRA_TIME_PREF_24_HOUR_FORMAT, false) ? 1
                                    : 0;
                    mHandler.sendMessage(mHandler.obtainMessage(UPDATE_TIME, is24Hour, 0));
                    BatteryStatsImpl stats = mBatteryStatsService.getActiveStatistics();
                    synchronized (stats) {
                        stats.noteCurrentTimeChangedLocked();
                    }
                    break;

                case Intent.ACTION_CLEAR_DNS_CACHE:
                    mHandler.sendEmptyMessage(CLEAR_DNS_CACHE_MSG);
                    break;

                case Proxy.PROXY_CHANGE_ACTION:
                    ProxyInfo proxy = intent.getParcelableExtra(Proxy.EXTRA_PROXY_INFO);
                    mHandler.sendMessage(mHandler.obtainMessage(UPDATE_HTTP_PROXY_MSG, proxy));
                    break;

                case android.hardware.Camera.ACTION_NEW_PICTURE:
                case android.hardware.Camera.ACTION_NEW_VIDEO:
                    // These broadcasts are no longer allowed by the system, since they can
                    // cause significant thrashing at a crictical point (using the camera).
                    // Apps should use JobScehduler to monitor for media provider changes.
                    Slog.w(TAG, action + " no longer allowed; dropping from "
                            + UserHandle.formatUid(callingUid));
                    if (resultTo != null) {
                        final BroadcastQueue queue = broadcastQueueForIntent(intent);
                        try {
                            queue.performReceiveLocked(callerApp, resultTo, intent,
                                    Activity.RESULT_CANCELED, null, null,
                                    false, false, userId);
                        } catch (RemoteException e) {
                            Slog.w(TAG, "Failure ["
                                    + queue.mQueueName + "] sending broadcast result of "
                                    + intent, e);

                        }
                    }
                    // Lie; we don't want to crash the app.
                    return ActivityManager.BROADCAST_SUCCESS;
            }
        }

        //【9】如果 sticky 为 true，表示该广播为粘性广播，那就要收集到指定的集合中！
        if (sticky) {
            //【9.1】权限校验，发送粘性广播必须配置 android.Manifest.permission.BROADCAST_STICKY 权限！
            // 否则会抛出异常！
            if (checkPermission(android.Manifest.permission.BROADCAST_STICKY,
                    callingPid, callingUid) != PackageManager.PERMISSION_GRANTED) {
                String msg = "Permission Denial: broadcastIntent() requesting a sticky broadcast from pid="
                        + callingPid + ", uid=" + callingUid
                        + " requires " + android.Manifest.permission.BROADCAST_STICKY;

                Slog.w(TAG, msg);
                throw new SecurityException(msg);
            }
            
            //【9.2】对于粘性广播，不能强制广播接收者应具有的权限！
            if (requiredPermissions != null && requiredPermissions.length > 0) {
                Slog.w(TAG, "Can't broadcast sticky intent " + intent
                        + " and enforce permissions " + Arrays.toString(requiredPermissions));
                return ActivityManager.BROADCAST_STICKY_CANT_HAVE_PERMISSION;
            }
            
            //【9.3】如果粘性广播设置具体的组件，抛出异常！
            if (intent.getComponent() != null) {
                throw new SecurityException(
                        "Sticky broadcasts can't target a specific component");
            }

            //【9.4】如果我们的目标设备用户 id 不是 UserHandle.USER_ALL
            // 那就要判断是否已经有一个相同的全局粘性广播，如果有，产生冲突，抛出异常！
            if (userId != UserHandle.USER_ALL) {
                ArrayMap<String, ArrayList<Intent>> stickies = mStickyBroadcasts.get(
                        UserHandle.USER_ALL);

                if (stickies != null) {
                    ArrayList<Intent> list = stickies.get(intent.getAction());
                    if (list != null) {
                        int N = list.size();
                        int i;
                        for (i=0; i<N; i++) {
                            if (intent.filterEquals(list.get(i))) {
                                throw new IllegalArgumentException(
                                        "Sticky broadcast " + intent + " for user "
                                        + userId + " conflicts with existing global broadcast");
                            }
                        }
                    }
                }
            }

            //【9.5】将该粘性广播 Intent 添加到设备用户 userId 对应的广播列表中！
            // 如果之前已经添加过，就取代之前的相同粘性广播！
            ArrayMap<String, ArrayList<Intent>> stickies = mStickyBroadcasts.get(userId);
            if (stickies == null) {
                stickies = new ArrayMap<>();
                mStickyBroadcasts.put(userId, stickies);
            }

            ArrayList<Intent> list = stickies.get(intent.getAction());
            if (list == null) {
                list = new ArrayList<>();
                stickies.put(intent.getAction(), list);
            }
            final int stickiesCount = list.size();
            int i;
            for (i = 0; i < stickiesCount; i++) {
                if (intent.filterEquals(list.get(i))) {
                    //【9.5.1】如果有重复，就取代旧的粘性广播！
                    list.set(i, new Intent(intent));
                    break;
                }
            }

            //【9.5.2】没有重复，就将这个广播添加到该粘性广播列表中！
            if (i >= stickiesCount) {
                list.add(new Intent(intent));
            }
        }
        
        //【10】根据传入的设备用户 id，获得广播的目标设备用户数组！
        int[] users;
        if (userId == UserHandle.USER_ALL) {
            // 如果 userId 为 UserHandle.USER_ALL，说明这个广播要发送给所有的设备用户，
            // 就调用 getStartedUserArrayLocked 方法获得当前被启动的所有的设备用户数组！
            users = mUserController.getStartedUserArrayLocked();
        } else {
            // 如果 userId 不为 UserHandle.USER_ALL，说明这个广播是发送给指定的设备用户的！
            users = new int[] {userId};
        }

        //【11】收集能够接收和处理该广播的广播接收者！
        // receivers 保存静态注册的广播接受者，registeredReceivers 保存动态注册的广播接收者！
        List receivers = null;
        List<BroadcastFilter> registeredReceivers = null;

        // 判断 Intent 是否设置了 Intent.FLAG_RECEIVER_REGISTERED_ONLY 标志位，如果设置了该标志位
        // 那么，静态注册的广播接收者无法接受这个广播！
        if ((intent.getFlags()&Intent.FLAG_RECEIVER_REGISTERED_ONLY) == 0) {
            //【×2.2.1】收集系统中匹配该广播的静态注册的广播接收者！
            receivers = collectReceiverComponents(intent, resolvedType, callingUid, users);
        }
        
        //【12】如果广播 Intent 没有设置组件信息，那就从 mReceiverResolver 中进行匹配动态注册的接收者！
        // mReceiverResolver 用来保存动态注册的广播接收者信息，这个我们在前面讲过！
        if (intent.getComponent() == null) {
            if (userId == UserHandle.USER_ALL && callingUid == Process.SHELL_UID) {
                // 遍历每一个设备用户！
                for (int i = 0; i < users.length; i++) {
                    if (mUserController.hasUserRestriction(
                            UserManager.DISALLOW_DEBUGGING_FEATURES, users[i])) {
                        continue;
                    }

                    List<BroadcastFilter> registeredReceiversForUser =
                            mReceiverResolver.queryIntent(intent,
                                    resolvedType, false, users[i]);

                    if (registeredReceivers == null) {
                        registeredReceivers = registeredReceiversForUser;

                    } else if (registeredReceiversForUser != null) {
                        registeredReceivers.addAll(registeredReceiversForUser);

                    }
                }
            } else {
                registeredReceivers = mReceiverResolver.queryIntent(intent,
                        resolvedType, false, userId);
            }
        }

        //【13】判断 intent 是否设置了 FLAG_RECEIVER_REPLACE_PENDING 标志位，如果设置了。
        // 那么新的 intent 会取代前面正在处于等待状态的 intent；
        final boolean replacePending =
                (intent.getFlags() & Intent.FLAG_RECEIVER_REPLACE_PENDING) != 0;

        if (DEBUG_BROADCAST) Slog.v(TAG_BROADCAST, "Enqueing broadcast: " + intent.getAction()
                + " replacePending=" + replacePending);
        
        int NR = registeredReceivers != null ? registeredReceivers.size() : 0; // 动态接受者的广播！
        
        //【14】如果该广播是普通广播，并且存在匹配的动态注册的接收器，将广播添加到队列的并行集合中！
        if (!ordered && NR > 0) {
            if (isCallerSystem) {
                checkBroadcastFromSystem(intent, callerApp, callerPackage, callingUid,
                        isProtectedBroadcast, registeredReceivers);
            }
            
            //【×2.3.1】根据 Intent 是否设置了 Intent.FLAG_RECEIVER_FOREGROUND 标志位，选择指定的前台队列或者后台队列！
            final BroadcastQueue queue = broadcastQueueForIntent(intent);
            
            //【×2.3.2】创建广播对应的 BroadcastRecord 对象，发送给动态注册的广播接收者 registeredReceivers！
            BroadcastRecord r = new BroadcastRecord(queue, intent, callerApp,
                    callerPackage, callingPid, callingUid, resolvedType, requiredPermissions,
                    appOp, brOptions, registeredReceivers, resultTo, resultCode, resultData,
                    resultExtras, ordered, sticky, false, userId);

            if (DEBUG_BROADCAST) Slog.v(TAG_BROADCAST, "Enqueueing parallel broadcast " + r);
            
            //【×2.3.3】如果在队列的 mParallelBroadcasts 集合中能够找到相同的无序广播，就取代之前的旧的无序广播！
            final boolean replaced = replacePending && queue.replaceParallelBroadcastLocked(r);

            if (!replaced) {
                //【×2.3.4】将该广播添加到队列的并行集合 mParallelBroadcasts 中！
                queue.enqueueParallelBroadcastLocked(r);
                //【×2.3.5】启动队列的广播分发任务！
                queue.scheduleBroadcastsLocked();
            }
            
            //【14.1】将动态注册的广播接收者集合 registeredReceivers 清空！
            registeredReceivers = null;
            NR = 0;
        }

        //【15】将查询到的广播接收者合并到一个列表中！
        int ir = 0;
        // 如果收集到静态注册的广播接收者，才会进入该分支！
        // 对于普通广播来说，动态注册的广播接收者在前面已经处理完了，这只剩下了静态注册的广播接收者了！
        // 对于有序广播来说，静态和动态的广播接收者都还没有处理！
        if (receivers != null) {
            //【15.1】这里对一些特殊的 action 做了一些处理，目的是防止刚安装的应用包通过静态接收这些广播实现自启动！
            // 首先收集这些这些广播中携带的 packageName！
            String skipPackages[] = null;
            if (Intent.ACTION_PACKAGE_ADDED.equals(intent.getAction())
                    || Intent.ACTION_PACKAGE_RESTARTED.equals(intent.getAction())
                    || Intent.ACTION_PACKAGE_DATA_CLEARED.equals(intent.getAction())) {
                Uri data = intent.getData();
                if (data != null) {
                    //【15.1.1】上面的广播会携带 Scheme 属性，表示包名，这里会收集到 skipPackages 中！
                    String pkgName = data.getSchemeSpecificPart();
                    if (pkgName != null) {
                        skipPackages = new String[] { pkgName };
                    }
                }

            } else if (Intent.ACTION_EXTERNAL_APPLICATIONS_AVAILABLE.equals(intent.getAction())) {
                skipPackages = intent.getStringArrayExtra(Intent.EXTRA_CHANGED_PACKAGE_LIST);
            }

            //【15.2】从静态注册的广播接收者中删除属于这些 package 的广播接收者，防止应用自启动！
            if (skipPackages != null && (skipPackages.length > 0)) {
                for (String skipPackage : skipPackages) {
                    if (skipPackage != null) {
                        int NT = receivers.size();
                        for (int it=0; it<NT; it++) {
                            ResolveInfo curt = (ResolveInfo)receivers.get(it);
                            if (curt.activityInfo.packageName.equals(skipPackage)) {
                                receivers.remove(it);
                                it--;
                                NT--;
                            }
                        }
                    }
                }
            }

            int NT = receivers != null ? receivers.size() : 0; // 静态接收者的数量！
            int it = 0;
            ResolveInfo curt = null;
            BroadcastFilter curr = null;

            //【15.3】注意，这里的 while 循环的目的是将所有的广播接收者合并到同一个集合中！
            // 如果是有序广播，这里会进入 while 循环，将 registeredReceivers 中的动态广播接收者，
            // 根据优先级合并到静态注册的广播接收者集合 receivers 中！
            // 对于普通广播，动态注册的广播接收者在前面已经处理完了，所以只剩下静态注册的接收者，所以不会进入循环！
            while (it < NT && nr < NR) {
                if (curt == null) {
                    curt = (ResolveInfo)receivers.get(it);
                }
                if (curr == null) {
                    curr = registeredReceivers.get(ir);
                }
                if (curr.getPriority() >= curt.priority) { // 按照优先级！
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
        }

        while (ir < NR) {
            if (receivers == null) {
                receivers = new ArrayList();
            }
            //【15.3】将剩余的动态接收者收集进来。当然如果没有静态接收者
            // 这里就只收集动态接收者；
            receivers.add(registeredReceivers.get(ir));
            ir++;
        }
        //【16】如果是系统调用，那么这里会做广播检查；
        if (isCallerSystem) {
            checkBroadcastFromSystem(intent, callerApp, callerPackage, callingUid,
                    isProtectedBroadcast, receivers);
        }
        
        //【17】最后，将广播分发给已经合并到同一个集合中的广播接收者！
        // 如果没有收集到任何的广播接收者，且这个广播是一个隐式广播，那就把这个广播的状态记录下来！
        if ((receivers != null && receivers.size() > 0)
                || resultTo != null) {

            // 这里的逻辑和上面的类似，不多说了！
            // 注意这里的 ordered，对于目标为静态接收者的普通广播，为 false，对于有序广播，则为 true；
            //【×2.3.1】选择合适的队列！
            BroadcastQueue queue = broadcastQueueForIntent(intent);
            //【×2.3.2】创建广播对象！
            BroadcastRecord r = new BroadcastRecord(queue, intent, callerApp,
                    callerPackage, callingPid, callingUid, resolvedType,
                    requiredPermissions, appOp, brOptions, receivers, resultTo, resultCode,
                    resultData, resultExtras, ordered, sticky, false, userId);

            if (DEBUG_BROADCAST) Slog.v(TAG_BROADCAST, "Enqueueing ordered broadcast " + r
                    + ": prev had " + queue.mOrderedBroadcasts.size());
            if (DEBUG_BROADCAST) Slog.i(TAG_BROADCAST,
                    "Enqueueing broadcast " + r.intent.getAction());
            //【×2.3.3】判读是否要取代已经存在的广播；
            boolean replaced = replacePending && queue.replaceOrderedBroadcastLocked(r);

            if (!replaced) {

                //【×2.3.4】将广播添加到有序列表中！
                queue.enqueueOrderedBroadcastLocked(r);
                //【×2.3.5】触发广播分发！
                queue.scheduleBroadcastsLocked();
            }
        } else {
            if (intent.getComponent() == null && intent.getPackage() == null
                    && (intent.getFlags()&Intent.FLAG_RECEIVER_REGISTERED_ONLY) == 0) {

                // 将广播的状态记录下来！
                addBroadcastStatLocked(intent.getAction(), callerPackage, 0, 0, 0);
            }
        }

        return ActivityManager.BROADCAST_SUCCESS;
    }

```

我们来总结一下，这个方法方法流程：

- **1.** 给 intent 添加 `Intent.FLAG_EXCLUDE_STOPPED_PACKAGES` 标志位，禁止广播发送给被强制停止的应用！
  - **1.1** 如果系统没有启动完成，就给广播添加 `Intent.FLAG_RECEIVER_REGISTERED_ONLY` 标志位，禁止启动静态广播接收者！
<br />

- **2.** 针对一些特殊 `action` 的广播进行处理，比如 `uid` 被移除，`package` 相关的广播等等！
<br />

- **3.** 如果是粘性广播，就需要将广播收集到 `mStickyBroadcasts` 集合中！
   - **3.1** 权限校验：发送粘性广播必须配置 `android.Manifest.permission.BROADCAST_STICKY` 权限！
   - **3.2** 对于粘性广播，不能强制广播接收者应具有的权限！
   - **3.3** 如果有相同的全局粘性广播，会产生冲突！
<br />

- **4.** 收集能够匹配当前广播的广播接收者：
   - **4.1** 尝试收集能够匹配当前广播的静态注册的广播接收者（如果 `intent` 设置了 `Intent.FLAG_RECEIVER_REGISTERED_ONLY`，不收集）
   - **4.2** 收集能够匹配当前广播动态注册的广播接收者
        - **4.2.1** 如果广播没有设置组件信息，那就从 `mReceiverResolver` 对象中收集动态广播接收者！
<br />	

- **5.** 如果广播是普通广播，且存在匹配的动态注册的广播接收者 `registeredReceivers`！
   - **5.1** 创建对应的 `BroadcastRecord(registeredReceivers)` 对象，这里的 `BroadcastRecord.receivers` 都是动态注册的广播接收者！
   - **5.2** 并添加到指定的队列（前台后台由 `Intent.FLAG_RECEIVER_FOREGROUND` 决定）的无序并行集合 `mParallelBroadcasts` 中！
   - **5.3** 触发广播发送任务，清空 `registeredReceivers`！
<br />

- **6.** 将收集到的静态广播接收者 `receivers` 和动态广播接收者 `registeredReceivers` 合并到同一个列表 `receivers` 中，只有在收集到静态注册的广播接收者才会合并！
   - **6.1** 这首先会针对一些特殊 `action` 做了处理，防止应用自启动！
   - **6.2** 对于普通广播，到这里就只剩下静态广播接收者了，无需合并；
             对于有序广播，到这里静态广播接收者和动态广播接收者都没有处理，那就按照优先级，将二者合并到静态广播接收者 receivers 中！
<br />   

- **7.** 处理 `6` 中获得的合并广播接收者集合！
    - **7.1** 创建对应的 `BroadcastRecord(receivers)` 对象，注意这里的 `BroadcastRecord.receivers` 可能只有静态广播接收者，可能静态动态都有！
    - **7.2** 并添加到指定的队列（前台后台由 `Intent.FLAG_RECEIVER_FOREGROUND` 决定）的无序并行集合 `mParallelBroadcasts` / 有序集合 `mOrderedBroadcasts` 中！
    - **7.3** 触发广播发送任务；
<br />   

接下来，我们来看看这个方法中的一些细节：

### 2.2.1 AMService.collectReceiverComponents

`collectReceiverComponents` 方法用来收集指定设备用户下的静态注册的广播接收者：
```java
    private List<ResolveInfo> collectReceiverComponents(Intent intent, String resolvedType,
            int callingUid, int[] users) {
        int pmFlags = STOCK_PM_FLAGS | MATCH_DEBUG_TRIAGED_MISSING;
        
        // 用来保存查询结果！
        List<ResolveInfo> receivers = null;
        try {

            // 用来保存在所有设备用户下都只有一个实例的广播接收者！
            HashSet<ComponentName> singleUserReceivers = null;
            boolean scannedFirstReceivers = false;

            for (int user : users) {

                // Skip users that have Shell restrictions, with exception of always permitted
                // Shell broadcasts
                if (callingUid == Process.SHELL_UID
                        && mUserController.hasUserRestriction(
                                UserManager.DISALLOW_DEBUGGING_FEATURES, user)
                        && !isPermittedShellBroadcast(intent)) {
                    continue;
                }
                
                // 查询指定设备用户下的静态广播接收者！
                List<ResolveInfo> newReceivers = AppGlobals.getPackageManager()
                        .queryIntentReceivers(intent, resolvedType, pmFlags, user).getList();
                
                // 如果不是系统用户，那就要检测该设备用户下的所有静态广播接收者，
                // 过滤掉设置了 systemUserOnly 属性的接收者！
                if (user != UserHandle.USER_SYSTEM && newReceivers != null) {
                    for (int i=0; i<newReceivers.size(); i++) {
                        ResolveInfo ri = newReceivers.get(i);
                        if ((ri.activityInfo.flags&ActivityInfo.FLAG_SYSTEM_USER_ONLY) != 0) {
                            newReceivers.remove(i);
                            i--;
                        }
                    }
                }

                if (newReceivers != null && newReceivers.size() == 0) {
                    newReceivers = null;
                }
                
                // 保存查询结果，查询第一个设备用户下的广播接收者进入该分支！
                if (receivers == null) {
                    receivers = newReceivers;

                } else if (newReceivers != null) {
                    // 后续设备用户下的查询会进入该分支！
                    if (!scannedFirstReceivers) {

                        // 收集那些在所有的设备用户下都只有一个实例的广播接收者！
                        scannedFirstReceivers = true;
                        for (int i=0; i<receivers.size(); i++) {
                            ResolveInfo ri = receivers.get(i);
                            if ((ri.activityInfo.flags&ActivityInfo.FLAG_SINGLE_USER) != 0) {
                                ComponentName cn = new ComponentName(
                                        ri.activityInfo.packageName, ri.activityInfo.name);

                                if (singleUserReceivers == null) {
                                    singleUserReceivers = new HashSet<ComponentName>();
                                }

                                singleUserReceivers.add(cn);
                            }
                        }
                    }

                    for (int i=0; i<newReceivers.size(); i++) {
                        ResolveInfo ri = newReceivers.get(i);
                        if ((ri.activityInfo.flags&ActivityInfo.FLAG_SINGLE_USER) != 0) {
                            ComponentName cn = new ComponentName(
                                    ri.activityInfo.packageName, ri.activityInfo.name);
                            if (singleUserReceivers == null) {
                                singleUserReceivers = new HashSet<ComponentName>();
                            }
                            
                            // 这里保证了在所有的设备用户下都只有一个实例的广播接收者只会被添加一次！
                            if (!singleUserReceivers.contains(cn)) {
                                singleUserReceivers.add(cn);
                                receivers.add(ri);
                            }
                        } else {
                            receivers.add(ri);
                        }
                    }
                }
            }
        } catch (RemoteException ex) {
            // pm is in same process, this will never happen.
        }
        return receivers;
    }
```
这里要注意：

如果我们在 `AndroidManifest.xml` 中给静态广播接收者设置了 `android:singleUser="true"` 属性的话，那么他对应的 `ActivityInfo.flags` 会被设置 `ActivityInfo.FLAG_SINGLE_USER` 位，这样该接收者在所有的设备用户下都只会有一个实例！


具体的关于静态广播接收者的查询过程，请去看另一篇文章!

### 2.2.2 PackageManagerS.isProtectedBroadcast
```java
    @Override
    public boolean isProtectedBroadcast(String actionName) {
        synchronized (mPackages) {
            if (mProtectedBroadcasts.contains(actionName)) {
                return true;
            } else if (actionName != null) {
                // TODO: remove these terrible hacks
                if (actionName.startsWith("android.net.netmon.lingerExpired")
                        || actionName.startsWith("com.android.server.sip.SipWakeupTimer")
                        || actionName.startsWith("com.android.internal.telephony.data-reconnect")
                        || actionName.startsWith("android.net.netmon.launchCaptivePortalApp")) {
                    return true;
                }
            }
        }
        return false;
    }
```
我们来看看如何判断一个广播是否是受保护的广播：

- 该广播在 `mProtectedBroadcasts` 列表中；
- 该广播是指定的一些广播；

二者满足任何一个条件就行了！

其中 mProtectedBroadcasts 是在 PMS 里面对 protected-broadcast 类型的标签进行解析得到的，这里我们不细看！

## 2.3 广播的分发处理

接下来，我们去看看广播队列是如何处理其内部的广播分发的，我们先回到 ActivityManagerS 中去！
```java
... ... ...
            BroadcastQueue queue = broadcastQueueForIntent(intent);

            BroadcastRecord r = new BroadcastRecord(queue, intent, callerApp,
                    callerPackage, callingPid, callingUid, resolvedType,
                    requiredPermissions, appOp, brOptions, receivers, resultTo, resultCode,
                    resultData, resultExtras, ordered, sticky, false, userId);

            if (DEBUG_BROADCAST) Slog.v(TAG_BROADCAST, "Enqueueing ordered broadcast " + r
                    + ": prev had " + queue.mOrderedBroadcasts.size());
            if (DEBUG_BROADCAST) Slog.i(TAG_BROADCAST,
                    "Enqueueing broadcast " + r.intent.getAction());

            boolean replaced = replacePending && queue.replaceOrderedBroadcastLocked(r);

            if (!replaced) {
                queue.enqueueOrderedBroadcastLocked(r);
                queue.scheduleBroadcastsLocked();
            }
... ... ...
```

这里是分发广播的核心代码段，我们一个一个来分析！

### 2.3.1 ActivityManagerS.broadcastQueueForIntent

```java
    BroadcastQueue broadcastQueueForIntent(Intent intent) {
        final boolean isFg = (intent.getFlags() & Intent.FLAG_RECEIVER_FOREGROUND) != 0;
        if (DEBUG_BROADCAST_BACKGROUND) Slog.i(TAG_BROADCAST,
                "Broadcast intent " + intent + " on "
                + (isFg ? "foreground" : "background") + " queue");
        return (isFg) ? mFgBroadcastQueue : mBgBroadcastQueue;
    }

```
用于判断将这个 `Intent` 插入哪个队列，插入的依据是 `Intent` 是否设置了 `Intent.FLAG_RECEIVER_FOREGROUND` 标志位！

这里我们可以看到 `ActivityManagerS` 通过 2 个队列来管理内部的广播：

- `mFgBroadcastQueue`：前台广播队列；
- `mBgBroadcastQueue`：后台广播队列；

### 2.3.2 new BroadcastRecord
```java
    BroadcastRecord(BroadcastQueue _queue,
            Intent _intent, ProcessRecord _callerApp, String _callerPackage,
            int _callingPid, int _callingUid, String _resolvedType, String[] _requiredPermissions,
            int _appOp, BroadcastOptions _options, List _receivers, IIntentReceiver _resultTo,
            int _resultCode, String _resultData, Bundle _resultExtras, boolean _serialized,
            boolean _sticky, boolean _initialSticky,
            int _userId) {

        if (_intent == null) {
            throw new NullPointerException("Can't construct with a null intent");
        }

        queue = _queue; // 该广播位于的广播队列
        intent = _intent; // 广播的 Intent
        targetComp = _intent.getComponent(); // 广播的目标组件
        callerApp = _callerApp; // 发送者的进程对象
        callerPackage = _callerPackage; // 发送者的包名
        callingPid = _callingPid; // 发送者的 pid
        callingUid = _callingUid; // 发送者的 uid
        resolvedType = _resolvedType;
        requiredPermissions = _requiredPermissions; // 广播接收者需要具备的权限！
        appOp = _appOp;
        options = _options;
        receivers = _receivers; // 广播接收者集合，包含静态广播接收者和动态广播接收者
        delivery = new int[_receivers != null ? _receivers.size() : 0]; // 广播的分发情况！

        resultTo = _resultTo;
        resultCode = _resultCode;
        resultData = _resultData;
        resultExtras = _resultExtras;

        ordered = _serialized; // 该广播是有序广播吗？
        sticky = _sticky; // 该广播是粘性广播吗？
        initialSticky = _initialSticky; 
        userId = _userId; // 广播的目标设备用户
        nextReceiver = 0; // 下一个要接受该广播的接收者序号！
        state = IDLE; // 广播的状态！
    }
```
这些属性其实很简单！

### 2.3.3 BroadcastQueue.replaceXXXXX

替换掉 BroadcastQueue 中 mParallelBroadcasts 已有的正在等待分发的 intent：

- replace ParallelBroadcast

```java
    public final boolean replaceParallelBroadcastLocked(BroadcastRecord r) {
        for (int i = mParallelBroadcasts.size() - 1; i >= 0; i--) {
            final Intent curIntent = mParallelBroadcasts.get(i).intent;
            //【1】匹配 filter，如果有匹配的 intent，替换掉！
            if (r.intent.filterEquals(curIntent)) {
                if (DEBUG_BROADCAST) Slog.v(TAG_BROADCAST,
                        "***** DROPPING PARALLEL ["
                + mQueueName + "]: " + r.intent);
                mParallelBroadcasts.set(i, r);
                return true;
            }
        }
        return false;
    }
```

同样的，如下操作：

- **replace Ordered Broadcast**

```JAVA
    public final boolean replaceOrderedBroadcastLocked(BroadcastRecord r) {
        for (int i = mOrderedBroadcasts.size() - 1; i > 0; i--) {
            //【1】匹配 filter，如果有匹配的 intent，替换掉！
            if (r.intent.filterEquals(mOrderedBroadcasts.get(i).intent)) {
                if (DEBUG_BROADCAST) Slog.v(TAG_BROADCAST,
                        "***** DROPPING ORDERED ["
                        + mQueueName + "]: " + r.intent);
                mOrderedBroadcasts.set(i, r);
                return true;
            }
        }
        return false;
    }
```

不多说了！

### 2.3.4 BroadcastQueue.enqueueOrderedBroadcastLocked

这里是将广播添加到广播队列的并行集合或者是有序集合中：

```java
    public void enqueueParallelBroadcastLocked(BroadcastRecord r) {
        mParallelBroadcasts.add(r);
        r.enqueueClockTime = System.currentTimeMillis();
    }

    public void enqueueOrderedBroadcastLocked(BroadcastRecord r) {
        mOrderedBroadcasts.add(r);
        r.enqueueClockTime = System.currentTimeMillis();
    }
```

通过前面的分析，我们可以看出：

- 对于普通广播来说
  - 先会被添加到 `mParallelBroadcasts` 中，`BroadcastRecord.receivers` 为动态注册的广播接收者集合，采用并发的发送方式；
  - 再被添加到 `mOrderedBroadcasts` 中，`BroadcastRecord.receivers` 为静态注册的广播接收者集合，采用有序的发送方式；

</br>

- 对于有序广播来说：
  - 会被直接添加到 `mOrderedBroadcasts` 中， `BroadcastRecord.receivers` 为静态注册和动态注册的广播接收者集合，按照优先级排序，采用有序的发送方式；

</br>

### 2.3.5 BroadcastQueue.scheduleBroadcastsLocked

接下来，就是触发广播的分发调度！
```java
    public void scheduleBroadcastsLocked() {
        if (DEBUG_BROADCAST) Slog.v(TAG_BROADCAST, "Schedule broadcasts ["
                + mQueueName + "]: current="
                + mBroadcastsScheduled);

        if (mBroadcastsScheduled) {
            return;
        }
        
        //【×2.4】发送 `BROADCAST_INTENT_MSG` 消息！
        mHandler.sendMessage(mHandler.obtainMessage(BROADCAST_INTENT_MSG, this));
        mBroadcastsScheduled = true;
    }
```

下面就进入了 BroadcastHandler 中！

## 2.4 BroadcastHandler.handleMessage

```java
    private final class BroadcastHandler extends Handler {
        public BroadcastHandler(Looper looper) {
            super(looper, null, true);
        }

        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                //【1】处理 BROADCAST_INTENT_MSG 消息！
                case BROADCAST_INTENT_MSG: {
                    if (DEBUG_BROADCAST) Slog.v(
                            TAG_BROADCAST, "Received BROADCAST_INTENT_MSG");
                    //【×2.5】进行广播分发，参数传入 true；
                    processNextBroadcast(true);
                } break;

                case BROADCAST_TIMEOUT_MSG: {
                    synchronized (mService) {
                        broadcastTimeoutLocked(true);
                    }
                } break;

                case SCHEDULE_TEMP_WHITELIST_MSG: {
                    DeviceIdleController.LocalService dic = mService.mLocalDeviceIdleController;
                    if (dic != null) {
                        dic.addPowerSaveTempWhitelistAppDirect(UserHandle.getAppId(msg.arg1),
                                msg.arg2, true, (String)msg.obj);
                    }
                } break;
            }
        }
    }
```
收到 `BROADCAST_INTENT_MSG` 消息后，调用 `processNextBroadcast` 方法，执行广播分发！

## 2.5 BroadcastQueue.processNextBroadcast

processNextBroadcast 方法比较长，需要我们仔细分析，参数传递：

- **boolean fromMsg**：传入 true，表示触发是来自消息调用！

```java
    final void processNextBroadcast(boolean fromMsg) {
        synchronized(mService) {
            BroadcastRecord r;

            if (DEBUG_BROADCAST) Slog.v(TAG_BROADCAST, "processNextBroadcast ["
                    + mQueueName + "]: "
                    + mParallelBroadcasts.size() + " broadcasts, "
                    + mOrderedBroadcasts.size() + " ordered broadcasts");

            //【1】更新 cpu 的状态信息！
            mService.updateCpuStats();

            if (fromMsg) {
                //【2】如果 fromMsg 为 true，会将 mBroadcastsScheduled 置为 false；
                // 这样之前又可以继续发送消息触发分发了！
                mBroadcastsScheduled = false;
            }

            //【3】立刻分发 mParallelBroadcasts 集合中的普通广播！
            // 根据前面的分析，普通广播会先被加入到 mParallelBroadcasts 中并且目标接收者是动态注册的广播接受者！
            while (mParallelBroadcasts.size() > 0) {
                r = mParallelBroadcasts.remove(0);
                //【3.1】更新普通广播的分发时间！
                r.dispatchTime = SystemClock.uptimeMillis();
                r.dispatchClockTime = System.currentTimeMillis();
                final int N = r.receivers.size();

                if (DEBUG_BROADCAST_LIGHT) Slog.v(TAG_BROADCAST, "Processing parallel broadcast ["
                        + mQueueName + "] " + r);

                for (int i=0; i<N; i++) {
                    Object target = r.receivers.get(i);
                    if (DEBUG_BROADCAST)  Slog.v(TAG_BROADCAST,
                            "Delivering non-ordered on [" + mQueueName + "] to registered "
                            + target + ": " + r);
                    
                    //【×2.5.1.1】将该普通广播分发给动态注册的每个接收者！
                    deliverToRegisteredReceiverLocked(r, (BroadcastFilter)target, false, i);
                }
                
                //【×2.5.1.2】分发成功后，将该普通广播添加到历史集合中！
                addBroadcastToHistoryLocked(r);

                if (DEBUG_BROADCAST_LIGHT) Slog.v(TAG_BROADCAST, "Done with parallel broadcast ["
                        + mQueueName + "] " + r);
            }

            //【4】处理正在等待目标进程启动的广播，如果 mPendingBroadcast 不为 null，说明该广播仍然未处理！
            if (mPendingBroadcast != null) {
                if (DEBUG_BROADCAST_LIGHT) Slog.v(TAG_BROADCAST,
                        "processNextBroadcast [" + mQueueName + "]: waiting for "
                        + mPendingBroadcast.curApp);

                boolean isDead;
                synchronized (mService.mPidsSelfLocked) {
                    //【4.1】获得目标进程的 ProcessRecord 对象！
                    ProcessRecord proc = mService.mPidsSelfLocked.get(mPendingBroadcast.curApp.pid);
                    isDead = proc == null || proc.crashing;
                }
                //【4.2】如果目标进程没有死亡，就继续的等待，return！
                if (!isDead) {
                    return;
                } else {
                    Slog.w(TAG, "pending app  ["
                            + mQueueName + "]" + mPendingBroadcast.curApp
                            + " died before responding to broadcast");
                    
                    //【4.3】目标进程死亡，将 mPendingBroadcast 初始化为 null！
                    mPendingBroadcast.state = BroadcastRecord.IDLE;
                    mPendingBroadcast.nextReceiver = mPendingBroadcastRecvIndex;
                    mPendingBroadcast = null;
                }
            }

            boolean looped = false;

            //【5】 while 循环用于移除 mOrderedBroadcasts 列表中的无需发送的广播，并找到下一个需要发送的广播！
            do {
                //【5.1】如果 mOrderedBroadcasts 的广播已经处理完了，就直接返回！
                if (mOrderedBroadcasts.size() == 0) {
                    mService.scheduleAppGcsLocked();
                    if (looped) {
                        //【5.1.1】更新系统 oomAdj 的值！
                        mService.updateOomAdjLocked();
                    }
                    return;
                }
                
                //【5.2】获得有序队列中的第一个要分发的广播！
                r = mOrderedBroadcasts.get(0);
                boolean forceReceive = false;

                int numReceivers = (r.receivers != null) ? r.receivers.size() : 0;
 
                if (mService.mProcessesReady && r.dispatchTime > 0) {
                    //【5.3】判断是否广播超时，超时则调用 broadcastTimeoutLocked(false) 强行结束广播；
                    // 并设置 forceReceive 为 true，设置广播状态 r.state 为 BroadcastRecord.IDLE！  
                    long now = SystemClock.uptimeMillis();
                    if ((numReceivers > 0) &&
                            (now > r.dispatchTime + (2*mTimeoutPeriod*numReceivers))) {

                        Slog.w(TAG, "Hung broadcast ["
                                + mQueueName + "] discarded after timeout failure:"
                                + " now=" + now
                                + " dispatchTime=" + r.dispatchTime
                                + " startTime=" + r.receiverTime
                                + " intent=" + r.intent
                                + " numReceivers=" + numReceivers
                                + " nextReceiver=" + r.nextReceiver
                                + " state=" + r.state);

                        //【5.4】处理广播超时
                        broadcastTimeoutLocked(false); 
                        //【5.5】广播超时了，所以我们视其为接收，forceReceive 为 true；
                        forceReceive = true;
                        //【5.6】更新广播的状态为 idle；
                        r.state = BroadcastRecord.IDLE;
                    }
                }

                //【5.7】判断广播的状态是否是 BroadcastRecord.IDLE，对于 mOrderedBroadcasts 集合中的广播来说，
                // 只有广播的状态为 BroadcastRecord.IDLE 时才能继续下一次的分发，这里就直接返回了！！
                if (r.state != BroadcastRecord.IDLE) {
                    if (DEBUG_BROADCAST) Slog.d(TAG_BROADCAST,
                            "processNextBroadcast("
                            + mQueueName + ") called when not idle (state="
                            + r.state + ")");
                    return;
                }
                
                //【5.8】如果该广播没有接收者，或者该广播的所有接收者都接收到了广播（nextReceiver 大于接收者数）
                // 或者该广播被终止继续传递（有序广播特性），或者该广播超时了被视为强制接收了，进入该分支；
                if (r.receivers == null || r.nextReceiver >= numReceivers
                        || r.resultAbort || forceReceive) {
                        
                    if (r.resultTo != null) {
                        try {
                            if (DEBUG_BROADCAST) Slog.i(TAG_BROADCAST,
                                    "Finishing broadcast [" + mQueueName + "] "
                                    + r.intent.getAction() + " app=" + r.callerApp);
                            
                            //【×2.5.1.3】如果该广播的 r.resultTo 不为 null，resultTo 也是一个广播接收者，
                            // 那就调用 performReceiveLocked() 将该广播发送给 resultTo！
                            performReceiveLocked(r.callerApp, r.resultTo,
                                new Intent(r.intent), r.resultCode,
                                r.resultData, r.resultExtras, false, false, r.userId);

                            //【5.8.1】将广播的 r.resultTo 置为 null；
                            r.resultTo = null;
                            
                        } catch (RemoteException e) {
                            r.resultTo = null;
                            Slog.w(TAG, "Failure ["
                                    + mQueueName + "] sending broadcast result of "
                                    + r.intent, e);

                        }
                    }

                    if (DEBUG_BROADCAST) Slog.v(TAG_BROADCAST, "Cancelling BROADCAST_TIMEOUT_MSG");
                    //【5.8.2】取消广播超时的消息！
                    cancelBroadcastTimeoutLocked();

                    if (DEBUG_BROADCAST_LIGHT) Slog.v(TAG_BROADCAST,
                            "Finished with ordered broadcast " + r);

                    //【×2.5.1.2】将广播添加到历史集合中！
                    addBroadcastToHistoryLocked(r);

                    //【5.8.3】如果该广播是一个隐式广播，将该广播添加到 mCurBroadcastStats 中记录下来！
                    if (r.intent.getComponent() == null && r.intent.getPackage() == null
                            && (r.intent.getFlags()&Intent.FLAG_RECEIVER_REGISTERED_ONLY) == 0) {
                        mService.addBroadcastStatLocked(r.intent.getAction(), r.callerPackage,
                                r.manifestCount, r.manifestSkipCount, r.finishTime-r.dispatchTime);
                    }
                    
                    //【5.8.4】从 mOrderedBroadcasts 中移除该广播！
                    mOrderedBroadcasts.remove(0);
                    
                    //【5.8.5】将 r 设为 null，进行下一次循环！
                    r = null;
                    looped = true;
                    continue;
                }
            } while (r == null);

            //【6】分发 mOrderedBroadcasts 集合中的广播，通过前面的分析 mOrderedBroadcasts 会存储 2 种类型的广播
            // 普通广播，接收者为静态注册的；有序广播，接收者为的动态和静态注册的合并；
            // 首先，计算广播的下一个接收者的序号。同时，增加目标接收者计数！！ 
            int recIdx = r.nextReceiver++;

            //【7】初始化该广播的接收时间，该时间会不断更新！！
            r.receiverTime = SystemClock.uptimeMillis();
            //【8】如果是第一个接收者，就记录广播的开始分发时间；
            if (recIdx == 0) {
                r.dispatchTime = r.receiverTime;
                r.dispatchClockTime = System.currentTimeMillis();
                if (DEBUG_BROADCAST_LIGHT) Slog.v(TAG_BROADCAST, "Processing ordered broadcast ["
                        + mQueueName + "] " + r);
            }
            //【9】如果没有设置广播超时消息，那就设置超时消息！
            if (!mPendingBroadcastTimeoutMessage) {
                long timeoutTime = r.receiverTime + mTimeoutPeriod;
                if (DEBUG_BROADCAST) Slog.v(TAG_BROADCAST,
                        "Submitting BROADCAST_TIMEOUT_MSG ["
                        + mQueueName + "] for " + r + " at " + timeoutTime);
                
                //【9.1】设置广播处理的超时时间为 10s
                // 这里会将 mPendingBroadcastTimeoutMessage 设置为 true；
                setBroadcastTimeoutLocked(timeoutTime);
            }

            //【10】获得广播的额外数据；
            final BroadcastOptions brOptions = r.options;

            //【11】获得广播的下一个接收者；
            final Object nextReceiver = r.receivers.get(recIdx);
            
            //【12】如果下一个广播接收者是动态注册的广播接收者！（这里只可能是有序广播）
            if (nextReceiver instanceof BroadcastFilter) {
                BroadcastFilter filter = (BroadcastFilter)nextReceiver;
    
                if (DEBUG_BROADCAST)  Slog.v(TAG_BROADCAST,
                        "Delivering ordered ["
                        + mQueueName + "] to registered "
                        + filter + ": " + r);
                
                //【×2.5.1.1】将该有序广播分发给动态注册的广播接收者，该方法中会进一步过滤！！
                deliverToRegisteredReceiverLocked(r, filter, r.ordered, recIdx);

                //【12.1】当所有的接收者都接收到了广播时，那么 r.receiver 为 null；
                // 或者说该广播是普通广播，那么这里会继续下一次分发！
                if (r.receiver == null || !r.ordered) {
                    if (DEBUG_BROADCAST) Slog.v(TAG_BROADCAST, "Quick finishing ["
                            + mQueueName + "]: ordered="
                            + r.ordered + " receiver=" + r.receiver);
                    
                    //【12.2】将广播的状态设置为 idle；
                    r.state = BroadcastRecord.IDLE;
                    //【×2.3.5】重新发送BROADCAST_INTENT_MSG，触发下一次发送广播的流程！
                    scheduleBroadcastsLocked();

                } else {
                    // 如果广播没有分发完成，并且是有序广播，并且其携带了额外数据，为了防止其在 doze 模式下
                    // 能够收到广播，所以如果额外数据中指定了 temp duration，那么会将其加入到 doze 临时白名单中；
                    if (brOptions != null && brOptions.getTemporaryAppWhitelistDuration() > 0) {
                        scheduleTempWhitelistLocked(filter.owningUid,
                                brOptions.getTemporaryAppWhitelistDuration(), r);
                    }
                }
                
                //【12.3】这里直接 return 掉，因为对于有序广播而言，
                // 已经通知了一个 BroadcastReceiver，需要等待其处理结果，因此返回！
                return;
            }

            //【13】如果下一个广播接收者是静态注册的广播接收者（对于有序/普通广播）
            ResolveInfo info =
                (ResolveInfo)nextReceiver;
            ComponentName component = new ComponentName(
                    info.activityInfo.applicationInfo.packageName,
                    info.activityInfo.name);
            
            //【14】判断是否跳过该静态接收者！
            boolean skip = false;
            
            //【15】如果 sdk 不匹配，跳过该接收者！
            if (brOptions != null &&
                    (info.activityInfo.applicationInfo.targetSdkVersion
                            < brOptions.getMinManifestReceiverApiLevel() ||
                    info.activityInfo.applicationInfo.targetSdkVersion
                            > brOptions.getMaxManifestReceiverApiLevel())) {
                skip = true;
            }

            //【16】检查广播发送者是否具有接收者要求的权限！
            int perm = mService.checkComponentPermission(info.activityInfo.permission,
                    r.callingPid, r.callingUid, info.activityInfo.applicationInfo.uid,
                    info.activityInfo.exported);

            if (!skip && perm != PackageManager.PERMISSION_GRANTED) {
                //【16.1】如果发送者不被授予该权限，那就跳过该接收者！
                if (!info.activityInfo.exported) {
                    Slog.w(TAG, "Permission Denial: broadcasting "
                            + r.intent.toString()
                            + " from " + r.callerPackage + " (pid=" + r.callingPid
                            + ", uid=" + r.callingUid + ")"
                            + " is not exported from uid " + info.activityInfo.applicationInfo.uid
                            + " due to receiver " + component.flattenToShortString());
                } else {
                    Slog.w(TAG, "Permission Denial: broadcasting "
                            + r.intent.toString()
                            + " from " + r.callerPackage + " (pid=" + r.callingPid
                            + ", uid=" + r.callingUid + ")"
                            + " requires " + info.activityInfo.permission
                            + " due to receiver " + component.flattenToShortString());
                }
                skip = true;

            } else if (!skip && info.activityInfo.permission != null) {
                //【16.2】权限校验通过，那就要判断发送者是否能执行接收者指定的权限对应的操作！
                // 一般情况下，如果有权限有对应的 op，那么权限在授予的情况下，也会授予 op！
                final int opCode = AppOpsManager.permissionToOpCode(info.activityInfo.permission);

                if (opCode != AppOpsManager.OP_NONE
                        && mService.mAppOpsService.noteOperation(opCode, r.callingUid,
                                r.callerPackage) != AppOpsManager.MODE_ALLOWED) {
                    Slog.w(TAG, "Appop Denial: broadcasting "
                            + r.intent.toString()
                            + " from " + r.callerPackage + " (pid="
                            + r.callingPid + ", uid=" + r.callingUid + ")"
                            + " requires appop " + AppOpsManager.permissionToOp(
                                    info.activityInfo.permission)
                            + " due to registered receiver "
                            + component.flattenToShortString());
                    skip = true;
                }
            }
            
            //【17】如果接收者不是 system uid 的，而广播发送者指定了权限，那就要校验接收者是否具有该权限！
            if (!skip && info.activityInfo.applicationInfo.uid != Process.SYSTEM_UID &&
                r.requiredPermissions != null && r.requiredPermissions.length > 0) {
                for (int i = 0; i < r.requiredPermissions.length; i++) {
                    String requiredPermission = r.requiredPermissions[i];
                    try {
                        perm = AppGlobals.getPackageManager().
                                checkPermission(requiredPermission,
                                        info.activityInfo.applicationInfo.packageName,
                                        UserHandle.getUserId(info.activityInfo.applicationInfo.uid));
                    } catch (RemoteException e) {
                        perm = PackageManager.PERMISSION_DENIED;
                    }
                    
                    //【17.1】接收者不具有该权限，跳过该接收者！
                    if (perm != PackageManager.PERMISSION_GRANTED) {
                        Slog.w(TAG, "Permission Denial: receiving "
                                + r.intent + " to "
                                + component.flattenToShortString()
                                + " requires " + requiredPermission
                                + " due to sender " + r.callerPackage
                                + " (uid " + r.callingUid + ")");
                        skip = true;
                        break;
                    }
                    
                    //【17.2】判断接收者是否能执行发送者指定的权限所对应的操作！
                    int appOp = AppOpsManager.permissionToOpCode(requiredPermission);
                    if (appOp != AppOpsManager.OP_NONE && appOp != r.appOp
                            && mService.mAppOpsService.noteOperation(appOp,
                            info.activityInfo.applicationInfo.uid, info.activityInfo.packageName)
                            != AppOpsManager.MODE_ALLOWED) {
                        Slog.w(TAG, "Appop Denial: receiving "
                                + r.intent + " to "
                                + component.flattenToShortString()
                                + " requires appop " + AppOpsManager.permissionToOp(
                                requiredPermission)
                                + " due to sender " + r.callerPackage
                                + " (uid " + r.callingUid + ")");
                        skip = true;
                        break;
                    }
                }
            }
            //【18】如果广播无需权限，只需要执行某个 op 的话，这里会判断是否有 op 权限！
            if (!skip && r.appOp != AppOpsManager.OP_NONE
                    && mService.mAppOpsService.noteOperation(r.appOp,
                    info.activityInfo.applicationInfo.uid, info.activityInfo.packageName)
                    != AppOpsManager.MODE_ALLOWED) {
                Slog.w(TAG, "Appop Denial: receiving "
                        + r.intent + " to "
                        + component.flattenToShortString()
                        + " requires appop " + AppOpsManager.opToName(r.appOp)
                        + " due to sender " + r.callerPackage
                        + " (uid " + r.callingUid + ")");
                skip = true;
            }
            
            //【19】判断 Intent 是否满足 AMS 的 IntentFirewall 防火墙要求，不满足，跳过！
            if (!skip) {
                skip = !mService.mIntentFirewall.checkBroadcast(r.intent, r.callingUid,
                        r.callingPid, r.resolvedType, info.activityInfo.applicationInfo.uid);
            }
            
            //【20】判断该接收者是否是单例！
            boolean isSingleton = false;
            try {
                isSingleton = mService.isSingleton(info.activityInfo.processName,
                        info.activityInfo.applicationInfo,
                        info.activityInfo.name, info.activityInfo.flags);
            } catch (SecurityException e) {
                Slog.w(TAG, e.getMessage());
                skip = true;
            }
            
            //【20.1】如果接收者是单例模式，那就要校验接收者是否有
            // android.Manifest.permission.INTERACT_ACROSS_USERS 权限，如果没有，跳过该接收者！
            if ((info.activityInfo.flags&ActivityInfo.FLAG_SINGLE_USER) != 0) {
                if (ActivityManager.checkUidPermission(
                        android.Manifest.permission.INTERACT_ACROSS_USERS,
                        info.activityInfo.applicationInfo.uid)
                                != PackageManager.PERMISSION_GRANTED) {
                    Slog.w(TAG, "Permission Denial: Receiver " + component.flattenToShortString()
                            + " requests FLAG_SINGLE_USER, but app does not hold "
                            + android.Manifest.permission.INTERACT_ACROSS_USERS);
                    skip = true;
                }
            }
            //【21】记录处理的静态接收者数量！
            if (!skip) {
                r.manifestCount++;
            } else {
                r.manifestSkipCount++;
            }
            
            //【22】如果接收者所在进程 crash 了，跳过该接收者！
            if (r.curApp != null && r.curApp.crashing) {
                Slog.w(TAG, "Skipping deliver ordered [" + mQueueName + "] " + r
                        + " to " + r.curApp + ": process crashing");
                skip = true;
            }
            
            //【23】判断该静态接收者所在的 package 是否可用，不可用就跳过！
            if (!skip) {
                boolean isAvailable = false;
                try {
                    isAvailable = AppGlobals.getPackageManager().isPackageAvailable(
                            info.activityInfo.packageName,
                            UserHandle.getUserId(info.activityInfo.applicationInfo.uid));
                } catch (Exception e) {
                    Slog.w(TAG, "Exception getting recipient info for "
                            + info.activityInfo.packageName, e);
                }
                if (!isAvailable) { // 不可用，跳过！
                    if (DEBUG_BROADCAST) Slog.v(TAG_BROADCAST,
                            "Skipping delivery to " + info.activityInfo.packageName + " / "
                            + info.activityInfo.applicationInfo.uid
                            + " : package no longer available");
                    skip = true;
                }
            }

            //【24】如果在任何组件运行时都要重新校验权限，那就执权限校验，校验失败，跳过！
            if (Build.PERMISSIONS_REVIEW_REQUIRED && !skip) {
                if (!requestStartTargetPermissionsReviewIfNeededLocked(r,
                        info.activityInfo.packageName, UserHandle.getUserId(
                                info.activityInfo.applicationInfo.uid))) {
                    skip = true;
                }
            }

            final int receiverUid = info.activityInfo.applicationInfo.uid;

            //【25】如果发送者不是系统进程，且接收者是单例，并且本次调用是有效的
            // 那就始终返回默认设备用户下的接收者对象，保持单例有效性！
            if (r.callingUid != Process.SYSTEM_UID && isSingleton
                    && mService.isValidSingletonCall(r.callingUid, receiverUid)) {
                info.activityInfo = mService.getActivityInfoForUser(info.activityInfo, 0);
            }
            
            //【26】获得目标广播接收者的进程名；
            String targetProcess = info.activityInfo.processName;

            //【27】获得目标广播接收者所在进程的 ProcessRecord 对象！
            ProcessRecord app = mService.getProcessRecordLocked(targetProcess,
                    info.activityInfo.applicationInfo.uid, false);
            
            if (!skip) {
                //【28】判断该静态接收者所在进程是否允许后台启动或者在后台接受广播，不允许就跳过！
                final int allowed = mService.checkAllowBackgroundLocked(
                        info.activityInfo.applicationInfo.uid, info.activityInfo.packageName, -1,
                        false);
                if (allowed != ActivityManager.APP_START_MODE_NORMAL) {
                    if (allowed == ActivityManager.APP_START_MODE_DISABLED) {
                        Slog.w(TAG, "Background execution disabled: receiving "
                                + r.intent + " to "
                                + component.flattenToShortString());
                        skip = true;

                    } else if (((r.intent.getFlags()&Intent.FLAG_RECEIVER_EXCLUDE_BACKGROUND) != 0)
                            || (r.intent.getComponent() == null
                                && r.intent.getPackage() == null
                                && ((r.intent.getFlags()
                                        & Intent.FLAG_RECEIVER_INCLUDE_BACKGROUND) == 0))) {
                        //【28.1】这里针对 intent 的属性做了判断：
                        // 如果 intent 设置了 FLAG_RECEIVER_EXCLUDE_BACKGROUND，那就不能发给后台接收者；
                        // 如果是没有设置 FLAG_RECEIVER_INCLUDE_BACKGROUND 的隐式广播，那么也不能发给后台接收者；
                        Slog.w(TAG, "Background execution not allowed: receiving "
                                + r.intent + " to "
                                + component.flattenToShortString());

                        skip = true;

                    }
                }
            }
            
            //【29】如果 skip 的值为 true，说明要跳过当前的静态接收者，继续下次广播分发！！
            if (skip) {
                if (DEBUG_BROADCAST)  Slog.v(TAG_BROADCAST,
                        "Skipping delivery of ordered [" + mQueueName + "] "
                        + r + " for whatever reason");

                //【29.1】设置序号为 recIdx 的接收者的分发状态为 BroadcastRecord.DELIVERY_SKIPPED 跳过！
                r.delivery[recIdx] = BroadcastRecord.DELIVERY_SKIPPED;
                r.receiver = null;
                r.curFilter = null;
                r.state = BroadcastRecord.IDLE;

                //【×2.3.5】继续下次广播分发，结束方法！
                scheduleBroadcastsLocked();

                return;
            }

            //【30】如果该静态接收者不会跳过，那就设置其属性，准备发送广播！
            r.delivery[recIdx] = BroadcastRecord.DELIVERY_DELIVERED;
            r.state = BroadcastRecord.APP_RECEIVE; // 广播状态；
            r.curComponent = component;
            r.curReceiver = info.activityInfo;

            if (DEBUG_MU && r.callingUid > UserHandle.PER_USER_RANGE) {
                Slog.v(TAG_MU, "Updated broadcast record activity info for secondary user, "
                        + info.activityInfo + ", callingUid = " + r.callingUid + ", uid = "
                        + info.activityInfo.applicationInfo.uid);
            }
            //【31】如果额外参数指定了 doze 白名单参数，那么就设置临时白名单；
            if (brOptions != null && brOptions.getTemporaryAppWhitelistDuration() > 0) {
                scheduleTempWhitelistLocked(receiverUid,
                        brOptions.getTemporaryAppWhitelistDuration(), r);
            }

            try {

                //【32】设置接收者所在的包 stoped 状态为 false！
                AppGlobals.getPackageManager().setPackageStoppedState(
                        r.curComponent.getPackageName(), false, UserHandle.getUserId(r.callingUid));

            } catch (RemoteException e) {

            } catch (IllegalArgumentException e) {
                Slog.w(TAG, "Failed trying to unstop package "
                        + r.curComponent.getPackageName() + ": " + e);
            }

            //【33】如果静态广播接收者所在的进程已经启动，那就要直接发送广播！
            if (app != null && app.thread != null) {
                try {
                    app.addPackage(info.activityInfo.packageName,
                            info.activityInfo.applicationInfo.versionCode, mService.mProcessStats);
                            
                    //【×2.5.2.1.1】发送广播，并等待结果返回！
                    processCurBroadcastLocked(r, app);
                    return;

                } catch (RemoteException e) {
                    Slog.w(TAG, "Exception when sending broadcast to "
                          + r.curComponent, e);
                } catch (RuntimeException e) {
                    Slog.wtf(TAG, "Failed sending broadcast to "
                            + r.curComponent + " with " + r.intent, e);

                    logBroadcastReceiverDiscardLocked(r);

                    //【×4.1.2】发送失败，结束本次发送！
                    finishReceiverLocked(r, r.resultCode, r.resultData,
                            r.resultExtras, r.resultAbort, false);

                    //【×2.3.5】继续发送后续的广播！
                    scheduleBroadcastsLocked();
                    r.state = BroadcastRecord.IDLE; // 设置状态！
                    return;
                }
            }

            if (DEBUG_BROADCAST)  Slog.v(TAG_BROADCAST,
                    "Need to start app ["
                    + mQueueName + "] " + targetProcess + " for broadcast " + r);

            //【34】如果静态广播接收者所在的进程没有启动，那就要先启动其所在进程！
            if ((r.curApp=mService.startProcessLocked(targetProcess,
                    info.activityInfo.applicationInfo, true,
                    r.intent.getFlags() | Intent.FLAG_FROM_BACKGROUND,
                    "broadcast", r.curComponent,
                    (r.intent.getFlags()&Intent.FLAG_RECEIVER_BOOT_UPGRADE) != 0, false, false))
                            == null) {

                Slog.w(TAG, "Unable to launch app "
                        + info.activityInfo.applicationInfo.packageName + "/"
                        + info.activityInfo.applicationInfo.uid + " for broadcast "
                        + r.intent + ": process is bad");
                
                logBroadcastReceiverDiscardLocked(r);
                //【×4.1.2】目标进程启动失败，结束本次发送！
                finishReceiverLocked(r, r.resultCode, r.resultData,
                        r.resultExtras, r.resultAbort, false);
                        
                //【×2.3.5】继续发送后续的广播！
                scheduleBroadcastsLocked();
                r.state = BroadcastRecord.IDLE;
                return;
            }
            
            //【35】将当前的广播保存到 mPendingBroadcast 中，表示正在等待目标进程启动的广播！
            mPendingBroadcast = r;
            // 广播的目标接收者的序号！
            mPendingBroadcastRecvIndex = recIdx;
        }
    }
```
这个方法的逻辑很长，我们来总结一下他的业务流程：

- 1 立刻分发 `mParallelBroadcasts` 集合中的目标为动态接收者的普通广播，方式方式为并行！
  - 1.1 调用【`deliverToRegisteredReceiverLocked`】发送广播！

</br>

- 2 处理正在等待目标进程启动的广播 `mPendingBroadcast`，`mPendingBroadcast` 是有序的发送方式！
  - 2.1 如果进程没有启动成功，就等待进程启动后处理改广播，`return`；

</br>

- 3 移除 `mOrderedBroadcasts` 列表中的无需发送的广播，并找到下一个需要发送的广播，`mOrderedBroadcasts` 集合中的所有广播都是有序发送！
  - 3.1 如果某个广播超时了，就强行结束广播，并设置广播的状态为 `BroadcastRecord.IDLE`;
  - 3.2 如果某个广播的状态不是 `BroadcastRecord.IDLE`，说明上一个接收者没有结束广播处理，`return`；
  - 3.3 如果某个广播没有接收者，或者已经分发给了所有的目标接受者，或者被终止传递，或者广播超时了，那就移除该广播！
      如果指定了 `r.resultTo`，那还需要将该广播发送给 `resultTo` 接受者！ 

</br>

- 4 有序分发 `mOrderedBroadcasts` 的广播，根据下一个接收者的不同，做不同的处理！
  - 4.1 如果是动态接收者，调用【`deliverToRegisteredReceiverLocked`】发送广播，然后等待处理结果，`return`！
  - 4.2 如果是静态接收者，先要判断是否 skip 该接收者；
      - 4.2.1 跳过的触发条件：
		    - sdk 不匹配，跳过；
		    - 发送者不具有接收者要求的权限，跳过；
		    - 接收者不具有发送者要求的权限，跳过；
		    - 广播不满足 `AMS` 的 `IntentFirewall` 防火墙要求，跳过；
		    - 接收者是单例，但是没有申明 `android.Manifest.permission.INTERACT_ACROSS_USERS`，跳过；
		    - 接收者所在进程 `crash` 了，跳过；
		    - 接收者所在 `package` 不可用，跳过；
		    - 接收者所在进程不允许后台启动，跳过；
	   
            </br>

	  - 4.2.2 如果需要跳过该接受者，执行 `scheduleBroadcastsLocked`，进行下一次广播分发；
	        - `r.delivery[recIdx] = BroadcastRecord.DELIVERY_SKIPPED`;
            - `r.receiver = null`;
            - `r.curFilter = null`;
            - `r.state = BroadcastRecord.IDLE`;
            
            </br>

	  - 4.2.3 如果不跳过，就执行发送操作，这里需要判断静态接收者的进程是否启动；
	        - 4.2.3.1 静态广播接收者所在的进程已经启动，调用【`processCurBroadcastLocked`】 发送广播，并等待结果，`return`；
			- 4.2.3.2 静态广播接收者所在的进程没有启动，调用 `startProcessLocked` 启动进程：
			        - 启动成功，将当前广播保存到 `mPendingBroadcast` 中，等待处理！
					- 启动失败，那就 执行 `scheduleBroadcastsLocked`，进行下一次广播分发；
				

</br>
            
梳理后，整个方法的业务逻辑很清楚了，下面我们来关注一些细节：

### 2.5.1 发送给动态注册的接收者

#### 2.5.1.1 BroadcastQueue.deliverToRegisteredReceiverLocked


对于目标是动态接收者的普通广播（`ordered` 为 `false`），和目标是动态接收者的有序广播（`ordered` 为 `true`），都是通过 `deliverToRegisteredReceiverLocked` 直接发送的！

下面，我们来看看如何给动态注册的接收者发送广播！
```java
    private void deliverToRegisteredReceiverLocked(BroadcastRecord r,
            BroadcastFilter filter, boolean ordered, int index) {

        //【1】判断是否跳过这个接收者，这个和前面的流程很类似！
        boolean skip = false;
        
        //【2】首先是权限校验，校验发送者是否有接收者定义的权限，没有，跳过该接收者！
        // 权限如果是授予的，那还要再校验下 appOps！
        if (filter.requiredPermission != null) {
            int perm = mService.checkComponentPermission(filter.requiredPermission,
                    r.callingPid, r.callingUid, -1, true);
            if (perm != PackageManager.PERMISSION_GRANTED) {
                Slog.w(TAG, "Permission Denial: broadcasting "
                        + r.intent.toString()
                        + " from " + r.callerPackage + " (pid="
                        + r.callingPid + ", uid=" + r.callingUid + ")"
                        + " requires " + filter.requiredPermission
                        + " due to registered receiver " + filter);
                skip = true;
            } else {
                final int opCode = AppOpsManager.permissionToOpCode(filter.requiredPermission);
                if (opCode != AppOpsManager.OP_NONE
                        && mService.mAppOpsService.noteOperation(opCode, r.callingUid,
                                r.callerPackage) != AppOpsManager.MODE_ALLOWED) {
                    Slog.w(TAG, "Appop Denial: broadcasting "
                            + r.intent.toString()
                            + " from " + r.callerPackage + " (pid="
                            + r.callingPid + ", uid=" + r.callingUid + ")"
                            + " requires appop " + AppOpsManager.permissionToOp(
                                    filter.requiredPermission)
                            + " due to registered receiver " + filter);
                    skip = true;
                }
            }
        }
        //【3】校验接收者者是否有发送者定义的权限，没有，跳过该接收者！
        // 权限如果是授予的，那还要再校验下 appOps！
        if (!skip && r.requiredPermissions != null && r.requiredPermissions.length > 0) {
            for (int i = 0; i < r.requiredPermissions.length; i++) {
                String requiredPermission = r.requiredPermissions[i];
                int perm = mService.checkComponentPermission(requiredPermission,
                        filter.receiverList.pid, filter.receiverList.uid, -1, true);

                if (perm != PackageManager.PERMISSION_GRANTED) {
                    Slog.w(TAG, "Permission Denial: receiving "
                            + r.intent.toString()
                            + " to " + filter.receiverList.app
                            + " (pid=" + filter.receiverList.pid
                            + ", uid=" + filter.receiverList.uid + ")"
                            + " requires " + requiredPermission
                            + " due to sender " + r.callerPackage
                            + " (uid " + r.callingUid + ")");

                    skip = true;
                    break;
                }
                int appOp = AppOpsManager.permissionToOpCode(requiredPermission);
                if (appOp != AppOpsManager.OP_NONE && appOp != r.appOp
                        && mService.mAppOpsService.noteOperation(appOp,
                        filter.receiverList.uid, filter.packageName)
                        != AppOpsManager.MODE_ALLOWED) {

                    Slog.w(TAG, "Appop Denial: receiving "
                            + r.intent.toString()
                            + " to " + filter.receiverList.app
                            + " (pid=" + filter.receiverList.pid
                            + ", uid=" + filter.receiverList.uid + ")"
                            + " requires appop " + AppOpsManager.permissionToOp(
                            requiredPermission)
                            + " due to sender " + r.callerPackage
                            + " (uid " + r.callingUid + ")");

                    skip = true;
                    break;
                }
            }
        }
        //【4】校验接收者权限，这里跟踪下代码，由于传入的权限为 null，方法里面只针对 uid 做了判断
        // 看是不是 root 或者 system，是不是隔离进程 uid；
        if (!skip && (r.requiredPermissions == null || r.requiredPermissions.length == 0)) {
            int perm = mService.checkComponentPermission(null,
                    filter.receiverList.pid, filter.receiverList.uid, -1, true);
            if (perm != PackageManager.PERMISSION_GRANTED) {
                Slog.w(TAG, "Permission Denial: security check failed when receiving "
                        + r.intent.toString()
                        + " to " + filter.receiverList.app
                        + " (pid=" + filter.receiverList.pid
                        + ", uid=" + filter.receiverList.uid + ")"
                        + " due to sender " + r.callerPackage
                        + " (uid " + r.callingUid + ")");
                skip = true;
            }
        }
        //【5】校验 ops！
        if (!skip && r.appOp != AppOpsManager.OP_NONE
                && mService.mAppOpsService.noteOperation(r.appOp,
                filter.receiverList.uid, filter.packageName)
                != AppOpsManager.MODE_ALLOWED) {
            Slog.w(TAG, "Appop Denial: receiving "
                    + r.intent.toString()
                    + " to " + filter.receiverList.app
                    + " (pid=" + filter.receiverList.pid
                    + ", uid=" + filter.receiverList.uid + ")"
                    + " requires appop " + AppOpsManager.opToName(r.appOp)
                    + " due to sender " + r.callerPackage
                    + " (uid " + r.callingUid + ")");
            skip = true;
        }
        
        //【6】判断是否允许后台发送广播，不允许就跳过！
        if (!skip) {
            final int allowed = mService.checkAllowBackgroundLocked(filter.receiverList.uid,
                    filter.packageName, -1, true);
            if (allowed == ActivityManager.APP_START_MODE_DISABLED) {
                Slog.w(TAG, "Background execution not allowed: receiving "
                        + r.intent
                        + " to " + filter.receiverList.app
                        + " (pid=" + filter.receiverList.pid
                        + ", uid=" + filter.receiverList.uid + ")");
                skip = true;
            }
        }
        
        //【7】广播不通过防火墙校验，跳过该接收者！
        if (!mService.mIntentFirewall.checkBroadcast(r.intent, r.callingUid,
                r.callingPid, r.resolvedType, filter.receiverList.uid)) {
            skip = true;
        }
        
        //【8】接收者所在进程 crash 了，就跳过接收者！
        if (!skip && (filter.receiverList.app == null || filter.receiverList.app.crashing)) {
            Slog.w(TAG, "Skipping deliver [" + mQueueName + "] " + r
                    + " to " + filter.receiverList + ": process crashing");
            skip = true;
        }
        
        //【9】如果跳过该接收者，那就设置该接收者为 DELIVERY_SKIPPED；
        if (skip) {
            r.delivery[index] = BroadcastRecord.DELIVERY_SKIPPED;
            return;
        }

        //【10】如果需要 review 权限，那就拉起 review！
        if (Build.PERMISSIONS_REVIEW_REQUIRED) {
            if (!requestStartTargetPermissionsReviewIfNeededLocked(r, filter.packageName,
                    filter.owningUserId)) {
                r.delivery[index] = BroadcastRecord.DELIVERY_SKIPPED;
                return;
            }
        }
        
        //【11】开始发送广播！
        r.delivery[index] = BroadcastRecord.DELIVERY_DELIVERED;

        //【12】如果是有序广播，下面的分支！！
        if (ordered) {
            //【12.1】设置广播的目标接收者 IIntentReceier.Proxy 对象，用于挂进程拉起 onReceive 方法！
            r.receiver = filter.receiverList.receiver.asBinder();
            //【12.2】设置一系列相互引用；
            r.curFilter = filter;
            filter.receiverList.curBroadcast = r;
            //【12.2】设置广播状态；
            r.state = BroadcastRecord.CALL_IN_RECEIVE;

            //【12.3】如果接收者所在进程已经启动！
            if (filter.receiverList.app != null) {
                r.curApp = filter.receiverList.app;
                filter.receiverList.app.curReceiver = r;
                mService.updateOomAdjLocked(r.curApp);
            }
        }

        try {
            if (DEBUG_BROADCAST_LIGHT) Slog.i(TAG_BROADCAST,
                    "Delivering to " + filter + " : " + r);

            if (filter.receiverList.app != null && filter.receiverList.app.inFullBackup) {
                //【13】如果接收者的进程正在进行数据的备份和还原操作，那就跳过当前接收者！
                // 发送给下一个接收者，先 finishReceiverLocked，再 scheduleBroadcastsLocked！！
                if (ordered) {
                    skipReceiverLocked(r);
                }
            } else {
                //【×2.5.1.3】发送广播，one way 异步传输，无需等待返回！
                performReceiveLocked(filter.receiverList.app, filter.receiverList.receiver,
                        new Intent(r.intent), r.resultCode, r.resultData,
                        r.resultExtras, r.ordered, r.initialSticky, r.userId);
            }
            
            //【14】如果是有序广播，将广播状态置为 CALL_DONE_RECEIVE 已经接受！
            if (ordered) {
                //【14.1】修改广播状态为 CALL_DONE_RECEIVE！
                r.state = BroadcastRecord.CALL_DONE_RECEIVE;
            }
        } catch (RemoteException e) {
            Slog.w(TAG, "Failure sending broadcast " + r.intent, e);
            if (ordered) {
                r.receiver = null;
                r.curFilter = null;
                filter.receiverList.curBroadcast = null;
                if (filter.receiverList.app != null) {
                    filter.receiverList.app.curReceiver = null;
                }
            }
        }
    }

```
整个方法的逻辑很简单！

#### 2.5.1.2 BroadcastQueue.addBroadcastToHistoryLocked

将分发的广播加入历史集合中：

```java
    private final void addBroadcastToHistoryLocked(BroadcastRecord r) {
        if (r.callingUid < 0) {
            // This was from a registerReceiver() call; ignore it.
            return;
        }
        r.finishTime = SystemClock.uptimeMillis();

        mBroadcastHistory[mHistoryNext] = r;
        mHistoryNext = ringAdvance(mHistoryNext, 1, MAX_BROADCAST_HISTORY);

        mBroadcastSummaryHistory[mSummaryHistoryNext] = r.intent;
        mSummaryHistoryEnqueueTime[mSummaryHistoryNext] = r.enqueueClockTime;
        mSummaryHistoryDispatchTime[mSummaryHistoryNext] = r.dispatchClockTime;
        mSummaryHistoryFinishTime[mSummaryHistoryNext] = System.currentTimeMillis();
        mSummaryHistoryNext = ringAdvance(mSummaryHistoryNext, 1, MAX_BROADCAST_SUMMARY_HISTORY);
    }
```

#### 2.5.1.3 BroadcastQueue.performReceiveLocked

```java
    void performReceiveLocked(ProcessRecord app, IIntentReceiver receiver,
            Intent intent, int resultCode, String data, Bundle extras,
            boolean ordered, boolean sticky, int sendingUser) throws RemoteException {
        if (app != null) {
            if (app.thread != null) {
                try {
                    //【×2.5.1.4】one-way 通过 Binder 通信，进入接收者所在的进程！
                    app.thread.scheduleRegisteredReceiver(receiver, intent, resultCode,
                            data, extras, ordered, sticky, sendingUser, app.repProcState);

                } catch (RemoteException ex) {
                    synchronized (mService) {
                        Slog.w(TAG, "Can't deliver broadcast to " + app.processName
                                + " (pid " + app.pid + "). Crashing it.");
                        app.scheduleCrash("can't deliver broadcast");
                    }
                    throw ex;
                }
            } else {
                throw new RemoteException("app.thread must not be null");
            }
        } else {
            //【1】如果进程 app 为 null，这里会直接通过 binder 拉起相应方法！
            receiver.performReceive(intent, resultCode, data, extras, ordered,
                    sticky, sendingUser);
        }
    }
```
这里会调用 `ApplicationThreadProxy` 对象的 `scheduleRegisteredReceiver`！

这里就要进入接收者所在的进程了！

#### 2.5.1.4 ApplicationThreadP.scheduleRegisteredReceiver
```java
    public void scheduleRegisteredReceiver(IIntentReceiver receiver, Intent intent,
            int resultCode, String dataStr, Bundle extras, boolean ordered,
            boolean sticky, int sendingUser, int processState) throws RemoteException {
        Parcel data = Parcel.obtain();
        data.writeInterfaceToken(IApplicationThread.descriptor);

        // 这里的 receiver 是动态注册的接收者在 AMS 中的 IIntentReceiver.Proxy 对象！
        data.writeStrongBinder(receiver.asBinder());
        intent.writeToParcel(data, 0);
        data.writeInt(resultCode);
        data.writeString(dataStr);
        data.writeBundle(extras);
        data.writeInt(ordered ? 1 : 0);
        data.writeInt(sticky ? 1 : 0);
        data.writeInt(sendingUser);
        data.writeInt(processState);
        
        // Binder 通信：SCHEDULE_REGISTERED_RECEIVER_TRANSACTION
        // 这里是 FLAG_ONEWAY，异步传输，无需等待返回！
        mRemote.transact(SCHEDULE_REGISTERED_RECEIVER_TRANSACTION, data, null,
                IBinder.FLAG_ONEWAY);
        data.recycle();
    }
```

接下来就会进入接收者所在进程！

### 2.5.2 发送给静态注册的接收者

对于静态注册的广播接收者（有序广播/普通广播），因为他的进程未必已经被启动，所以要分情况！

#### 2.5.2.1 进程已启动

如果接收者所在的进程被启动了：
```java
           // Is this receiver's application already running?
            if (app != null && app.thread != null) {
                try {
                    app.addPackage(info.activityInfo.packageName,
                            info.activityInfo.applicationInfo.versionCode, mService.mProcessStats);
                    
                    //【×2.5.2.1.1】那就调用 processCurBroadcastLocked 处理广播发送！
                    processCurBroadcastLocked(r, app);
                    return;
                } catch (RemoteException e) {
                   
                } catch (RuntimeException e) {
                   
                }
            }
```
我们继续看：

##### 2.5.2.1.1 BroadcastQueue.processCurBroadcastLocked

发送当前的广播！

```java
    private final void processCurBroadcastLocked(BroadcastRecord r,
            ProcessRecord app) throws RemoteException {
        if (DEBUG_BROADCAST)  Slog.v(TAG_BROADCAST,
                "Process cur broadcast " + r + " for app " + app);
        if (app.thread == null) {
            throw new RemoteException();
        }

        //【1】进程如果在进行备份还原操作，跳过该静态接收者！
        if (app.inFullBackup) {
            skipReceiverLocked(r);
            return;
        }
        
        //【2】将广播的 r.receiver 设置为接收者进程的 ApplicationThreadProxy 对象！
        r.receiver = app.thread.asBinder();
        r.curApp = app;
        app.curReceiver = r;

        app.forceProcessStateUpTo(ActivityManager.PROCESS_STATE_RECEIVER);
        mService.updateLruProcessLocked(app, false, null);
        mService.updateOomAdjLocked();

        //【3】设置组件信息，由于前面 r.receiver 设置的是 ApplicationThreadProxy，
        // 所以为了拉起接收者，这里必须设置组件名！
        r.intent.setComponent(r.curComponent);

        boolean started = false;
        try {
            if (DEBUG_BROADCAST_LIGHT) Slog.v(TAG_BROADCAST,
                    "Delivering to component " + r.curComponent
                    + ": " + r);

            mService.notifyPackageUse(r.intent.getComponent().getPackageName(),
                                      PackageManager.NOTIFY_PACKAGE_USE_BROADCAST_RECEIVER);
                                      
            //【×3.2.1】通过 Binder 通信，进入应用进程，拉起接收者的 onReceive 方法！
            // one ways！
            app.thread.scheduleReceiver(new Intent(r.intent), r.curReceiver,
                    mService.compatibilityInfoForPackageLocked(r.curReceiver.applicationInfo),
                    r.resultCode, r.resultData, r.resultExtras, r.ordered, r.userId,
                    app.repProcState);

            if (DEBUG_BROADCAST)  Slog.v(TAG_BROADCAST,
                    "Process cur broadcast " + r + " DELIVERED for app " + app);

            started = true;

        } finally {
            if (!started) {
                if (DEBUG_BROADCAST)  Slog.v(TAG_BROADCAST,
                        "Process cur broadcast " + r + ": NOT STARTED!");
                r.receiver = null;
                r.curApp = null;
                app.curReceiver = null;
            }
        }
    }
```
这里就会进入接收者所在的进程，我们后面再看！

##### 2.5.2.1.2 ApplicationThreadP.processCurBroadcastLocked

发送当前的广播！
```java
    public final void scheduleReceiver(Intent intent, ActivityInfo info,
            CompatibilityInfo compatInfo, int resultCode, String resultData,
            Bundle map, boolean sync, int sendingUser, int processState) throws RemoteException {
        Parcel data = Parcel.obtain();
        data.writeInterfaceToken(IApplicationThread.descriptor);
        intent.writeToParcel(data, 0);
        info.writeToParcel(data, 0);
        compatInfo.writeToParcel(data, 0);
        data.writeInt(resultCode);
        data.writeString(resultData);
        data.writeBundle(map);
        data.writeInt(sync ? 1 : 0);
        data.writeInt(sendingUser);
        data.writeInt(processState);
        
        //【1】发送 SCHEDULE_RECEIVER_TRANSACTION 消息，one way！！
        mRemote.transact(SCHEDULE_RECEIVER_TRANSACTION, data, null,
                IBinder.FLAG_ONEWAY);
        data.recycle();
    }
```

下面就进入了应用进程：

#### 2.5.2.2 进程未启动

对于进程没有启动这种情况，我们需要主动的拉起接收者所在的进程：

```java
            if ((r.curApp=mService.startProcessLocked(targetProcess,
                    info.activityInfo.applicationInfo, true,
                    r.intent.getFlags() | Intent.FLAG_FROM_BACKGROUND,
                    "broadcast", r.curComponent,
                    (r.intent.getFlags()&Intent.FLAG_RECEIVER_BOOT_UPGRADE) != 0, false, false))
                            == null) {
                ... ... ... ...
            }
            //【1】将广播保存到 mPendingBroadcast 中！
            mPendingBroadcast = r;
            mPendingBroadcastRecvIndex = recIdx;
```

对于进程的启动，我们这里不详细的介绍，请看其他的博文，这里我就直接上结果：

##### 2.5.2.2.1 ActivityManagerService.attachApplicationLocked

```java
    private final boolean attachApplicationLocked(IApplicationThread thread,
            int pid) {
        
        ... ... ... ...
        
        //【×2.5.2.2.1.1 】判读是否有等待该进程启动的广播
        if (!badApp && isPendingBroadcastProcessLocked(pid)) {
            try {

                //【×2.5.2.2.1.2】发送广播！
                didSomething |= sendPendingBroadcastsLocked(app);
            } catch (Exception e) {
                // If the app died trying to launch the receiver we declare it 'bad'
                Slog.wtf(TAG, "Exception thrown dispatching broadcasts in " + app, e);
                badApp = true;
            }
        }
        
        ... ... ... ...
        
    }
```
我们继续来看：

###### 2.5.2.2.1.1 ActivityManagerS.isPendingBroadcastProcessLocked

进程启动后，会判断该进程中是否有有需要分发的广播：
```java
    boolean isPendingBroadcastProcessLocked(int pid) {
        //【×2.5.2.2.1.2】判读是否有等待该进程启动的广播
        return mFgBroadcastQueue.isPendingBroadcastProcessLocked(pid)
                || mBgBroadcastQueue.isPendingBroadcastProcessLocked(pid);
    }
```

其实判断的依据很简单：
```java
    public boolean isPendingBroadcastProcessLocked(int pid) {
        return mPendingBroadcast != null && mPendingBroadcast.curApp.pid == pid;
    }
```
如果 `BroadcastQueue` 内部的 `mPendingBroadcast` 不为 `null`，且 `mPendingBroadcast.curApp.pid` 等于当前进程的 `pid`，就说明有目标为该进程的广播！

###### 2.5.2.2.1.2 ActivityManagerS.sendPendingBroadcastsLocked
```java
    boolean sendPendingBroadcastsLocked(ProcessRecord app) {
        boolean didSomething = false;

        //【1】遍历前台和后台广播队列！
        for (BroadcastQueue queue : mBroadcastQueues) {
            //【×2.5.2.2.1.3】发送 pending broadcast！！
            didSomething |= queue.sendPendingBroadcastsLocked(app);
        }
        return didSomething;
    }
```
下面我们去 `BroadcastQueue` 中去看看！

###### 2.5.2.2.1.3 BroadcastQueue.sendPendingBroadcastsLocked
```java
    public boolean sendPendingBroadcastsLocked(ProcessRecord app) {
        boolean didSomething = false;
        
        //【1】将 mPendingBroadcast 赋给 br！
        final BroadcastRecord br = mPendingBroadcast;
        if (br != null && br.curApp.pid == app.pid) {
        
            //【1】校验进程是否匹配！
            if (br.curApp != app) {
                Slog.e(TAG, "App mismatch when sending pending broadcast to "
                        + app.processName + ", intended target is " + br.curApp.processName);
                return false;
            }

            try {
                //【2】将 mPendingBroadcast 赋值为 null；
                mPendingBroadcast = null;

                //【×2.5.2.1.1】发送广播！
                processCurBroadcastLocked(br, app);
                didSomething = true;

            } catch (Exception e) {
                Slog.w(TAG, "Exception in new application when starting receiver "
                        + br.curComponent.flattenToShortString(), e);

                logBroadcastReceiverDiscardLocked(br);
                finishReceiverLocked(br, br.resultCode, br.resultData,
                        br.resultExtras, br.resultAbort, false);
                scheduleBroadcastsLocked();

                // We need to reset the state if we failed to start the receiver.
                br.state = BroadcastRecord.IDLE;
                throw new RuntimeException(e.getMessage());
            }
        }
        return didSomething;
    }

```
这里的 `processCurBroadcastLocked` 又回到了进程已启动的发送流程了！！

# 3 接收者进程

这里我们按照广播接收者的不同来分别看看：

## 3.1 动态接收者进程

发送给动态注册的接收者有 `2` 种广播，首先会进入 `ApplicationThreadN` 中：

```java
        case SCHEDULE_REGISTERED_RECEIVER_TRANSACTION: {
            data.enforceInterface(IApplicationThread.descriptor);
            
            IIntentReceiver receiver = IIntentReceiver.Stub.asInterface(
                    data.readStrongBinder());
            
            // 这里是将 IIntentReceiver Binder 对象转为 InnerReceiver 对象！
            Intent intent = Intent.CREATOR.createFromParcel(data);
            int resultCode = data.readInt();
            String dataStr = data.readString();
            Bundle extras = data.readBundle();
            boolean ordered = data.readInt() != 0;
            boolean sticky = data.readInt() != 0;
            int sendingUser = data.readInt();
            int processState = data.readInt();
            
            //【1】进入 ApplicationThread，调用 scheduleRegisteredReceiver 方法！
            scheduleRegisteredReceiver(receiver, intent,
                    resultCode, dataStr, extras, ordered, sticky, sendingUser, processState);
                    
            return true;
        }
```

### 3.1.1 ApplicationThread.scheduleRegisteredReceiver
```java
        public void scheduleRegisteredReceiver(IIntentReceiver receiver, Intent intent,
                int resultCode, String dataStr, Bundle extras, boolean ordered,
                boolean sticky, int sendingUser, int processState) throws RemoteException {
            updateProcessState(processState, false);
            
            //【×3.1.2】进入 InnerReceiver！
            receiver.performReceive(intent, resultCode, dataStr, extras, ordered,
                    sticky, sendingUser);
        }
```

### 3.1.2 InnerReceiver.performReceive

继续看：
```java
        @Override
        public void performReceive(Intent intent, int resultCode, String data,
                Bundle extras, boolean ordered, boolean sticky, int sendingUser) {
            final LoadedApk.ReceiverDispatcher rd;
            if (intent == null) {
                Log.wtf(TAG, "Null intent received");
                rd = null;
            } else {
                //【1】如果广播不为 null，就获得对应的 ReceiverDispatcher 对象，用于分发广播！
                rd = mDispatcher.get();
            }

            if (ActivityThread.DEBUG_BROADCAST) {
                int seq = intent.getIntExtra("seq", -1);
                Slog.i(ActivityThread.TAG, "Receiving broadcast " + intent.getAction()
                        + " seq=" + seq + " to " + (rd != null ? rd.mReceiver : null));
            }

            if (rd != null) {
                //【×3.1.3】调用 ReceiverDispatcher 的 performReceive 方法，分发广播！
                rd.performReceive(intent, resultCode, data, extras,
                        ordered, sticky, sendingUser);
            } else {
                //【3】如果 rd 为 null，那就调用 AMS 的方法，立刻结束广播的发送！
                if (ActivityThread.DEBUG_BROADCAST) Slog.i(ActivityThread.TAG,
                        "Finishing broadcast to unregistered receiver");
                IActivityManager mgr = ActivityManagerNative.getDefault();

                try {
                    if (extras != null) {
                        extras.setAllowFds(false);
                    }
                    //【4】调用了 ams 的 finish receiver 方法，这个方法我们在取消注册的时候会看！
                    mgr.finishReceiver(this, resultCode, data, extras, false, intent.getFlags());
                } catch (RemoteException e) {
                    throw e.rethrowFromSystemServer();
                }
            }
        }
```

### 3.1.3 ReceiverDispatcher.performReceive

接着，进入 `ReceiverDispatcher`，我们都知道 `ReceiverDispatcher` 保存了 `BroadcastReceiver` 和 `InnerReceiver` 的映射关系：
```java
        public void performReceive(Intent intent, int resultCode, String data,
                Bundle extras, boolean ordered, boolean sticky, int sendingUser) {
            //【*3.1.3.1】创建了一个 Args 对象，用于封装广播参数信息！Args 继承了 Runnable 对象!
            final Args args = new Args(intent, resultCode, data, extras, ordered,
                    sticky, sendingUser);

            if (intent == null) {
                Log.wtf(TAG, "Null intent received");
            } else {
                if (ActivityThread.DEBUG_BROADCAST) {
                    int seq = intent.getIntExtra("seq", -1);
                    Slog.i(ActivityThread.TAG, "Enqueueing broadcast " + intent.getAction()
                            + " seq=" + seq + " to " + mReceiver);
                }
            }
            
            //【×3.1.4】调用 mActivityThread.post 执行 Args 任务！post 如果返回 false，说明任务执行失败！
            if (intent == null || !mActivityThread.post(args)) {
                //【1】任务执行失败，如果该广播是通过有序的方式发送的，还要通知系统，广播已经分发完成！
                // 然后系统就会进行下一个广播的有序分发！
                if (mRegistered && ordered) {
                    IActivityManager mgr = ActivityManagerNative.getDefault();
                    if (ActivityThread.DEBUG_BROADCAST) Slog.i(ActivityThread.TAG,
                            "Finishing sync broadcast to " + mReceiver);
                    //【×3.3.2】调用了 args 父类 PendingResult 的 sendFinished 方法！
                    args.sendFinished(mgr);
                }
            }
        }
```

这里我们回顾一下，采用有序的分发方式的广播有 `2` 种类型：

- 目标是静态注册的接收者的普通广播；
- 另外一种是目标是静态或者动态注册的接收者的有序广播；

这里的 `ordered` 如果是普通广播，那么为 `false`，如果是有序广播，那么为 `true`！

#### 3.1.3.1 new Args

创建了一个 Args 对象，其继承了 PendingResult：

```java
        final class Args extends BroadcastReceiver.PendingResult implements Runnable {

            private Intent mCurIntent;
            private final boolean mOrdered;
            private boolean mDispatched;

            public Args(Intent intent, int resultCode, String resultData, Bundle resultExtras,
                    boolean ordered, boolean sticky, int sendingUser) {
                
                //【1】这里的 mIIntentReceiver 是 ReceiverDispatcher 的内部变量：InnerReceiver！
                // 显然，对于动态注册的接收者，mType 等于 TYPE_REGISTERED 或者 TYPE_UNREGISTERED；
                super(resultCode, resultData, resultExtras,
                        mRegistered ? TYPE_REGISTERED : TYPE_UNREGISTERED, ordered,
                        sticky, mIIntentReceiver.asBinder(), sendingUser, intent.getFlags());

                mCurIntent = intent;
                mOrdered = ordered;
            }
            ... ... ... ...
        }
```
这里的 `mRegistered` 是应用在动态注册接收者时，创建 `ReceiverDispatcher` 时设置为 `true` 的，这里显然为 `true`！







### 3.1.4 Args.run

`mActivityThread.post` 方法，会在主线程执行任务 `Args`，下面我们来看看 `Args.run` 方法：

```java
            public void run() {
                final BroadcastReceiver receiver = mReceiver;
                final boolean ordered = mOrdered; // 是否有序！
                
                if (ActivityThread.DEBUG_BROADCAST) {
                    int seq = mCurIntent.getIntExtra("seq", -1);
                    Slog.i(ActivityThread.TAG, "Dispatching broadcast " + mCurIntent.getAction()
                            + " seq=" + seq + " to " + mReceiver);
                    Slog.i(ActivityThread.TAG, "  mRegistered=" + mRegistered
                            + " mOrderedHint=" + ordered);
                }
                
                final IActivityManager mgr = ActivityManagerNative.getDefault();
                final Intent intent = mCurIntent;
                if (intent == null) {
                    Log.wtf(TAG, "Null intent being dispatched, mDispatched=" + mDispatched);
                }

                mCurIntent = null;
                mDispatched = true;
                
                //【1】如果动态注册的接收者为 null，或者广播为 null，或者已经通过 unregisterReceiver
                // 方法动态取消注册了接收者，并且如果是有序广播，就要通知系统，继续下一次分发！
                if (receiver == null || intent == null || mForgotten) {
                    if (mRegistered && ordered) {
                        if (ActivityThread.DEBUG_BROADCAST) Slog.i(ActivityThread.TAG,
                                "Finishing null broadcast to " + mReceiver);
                        //【×3.3.2】调用了 args 父类 PendingResult 的 sendFinished 方法！
                        sendFinished(mgr);
                    }
                    return;
                }

                Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "broadcastReceiveReg");
    
                try {
                
                    //【2】通过反射的方式，拉起 onReceive 方法！
                    ClassLoader cl =  mReceiver.getClass().getClassLoader();
                    intent.setExtrasClassLoader(cl);
                    intent.prepareToEnterProcess();
                    setExtrasClassLoader(cl);
                    
                    //【3】设置 PendingResult 属性
                    receiver.setPendingResult(this);
                    
                    //【4】拉起 onReceive 方法！
                    receiver.onReceive(mContext, intent);

                } catch (Exception e) {
                    if (mRegistered && ordered) {
                        if (ActivityThread.DEBUG_BROADCAST) Slog.i(ActivityThread.TAG,
                                "Finishing failed broadcast to " + mReceiver);
                        //【×3.3.2】这里出现了异常，如果是有序广播，需要通知系统！
                        // 调用了 args 父类 PendingResult 的 sendFinished 方法！
                        sendFinished(mgr);
                    }
                    if (mInstrumentation == null ||
                            !mInstrumentation.onException(mReceiver, e)) {
                        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                        throw new RuntimeException(
                            "Error receiving broadcast " + intent
                            + " in " + mReceiver, e);
                    }
                }
                
                //【3】显然，这里不为 null，调用 finish 结束广播！
                if (receiver.getPendingResult() != null) {
                    //【×3.3.1】调用了父类的 finish 方法！
                    finish();
                }
                Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
            }
```
`mForgotten` 属性是在 `unregisterReceiver` 的时候被置为 `true` 的，这里终于进入了 `BroadcastReceiver` 的 `onReceive` 方法！

## 3.2 静态接收者进程

发送给静态注册的接收者也有 `2` 种广播，首先会进入 `ApplicationThreadN` 中：

```java
        case SCHEDULE_RECEIVER_TRANSACTION:
        {
            data.enforceInterface(IApplicationThread.descriptor);
            Intent intent = Intent.CREATOR.createFromParcel(data);
            ActivityInfo info = ActivityInfo.CREATOR.createFromParcel(data);
            CompatibilityInfo compatInfo = CompatibilityInfo.CREATOR.createFromParcel(data);
            int resultCode = data.readInt();
            String resultData = data.readString();
            Bundle resultExtras = data.readBundle();
            boolean sync = data.readInt() != 0;
            int sendingUser = data.readInt();
            int processState = data.readInt();
            
            //【1】进入 ApplicationThread 的 scheduleReceiver 方法！
            scheduleReceiver(intent, info, compatInfo, resultCode, resultData,
                    resultExtras, sync, sendingUser, processState);
            return true;
        }
```

### 3.2.1 ApplicationThread.scheduleReceiver
```java
        public final void scheduleReceiver(Intent intent, ActivityInfo info,
                CompatibilityInfo compatInfo, int resultCode, String data, Bundle extras,
                boolean sync, int sendingUser, int processState) {
            updateProcessState(processState, false);

            //【×3.2.1.1】创建了一个 ReceiverData 对象！
            // mAppThread 是当前进程的 ApplicationThread 对象，是一个 Binder 对象！
            ReceiverData r = new ReceiverData(intent, resultCode, data, extras,
                    sync, false, mAppThread.asBinder(), sendingUser);
            r.info = info;
            r.compatInfo = compatInfo;
            
            //【×3.2.2】发送 H.RECEIVER 给主线程！
            sendMessage(H.RECEIVER, r);
        }
```
这里创建了一个 `ReceiverData` 对象，用来封装接收者的数据信息！

#### 3.2.1.1 new ReceiverData

```java
    static final class ReceiverData extends BroadcastReceiver.PendingResult {
        public ReceiverData(Intent intent, int resultCode, String resultData, Bundle resultExtras,
                boolean ordered, boolean sticky, IBinder token, int sendingUser) {
            
            // 这个地方，传入的是 TYPE_COMPONENT，即 mType 是 TYPE_COMPONENT！
            super(resultCode, resultData, resultExtras, TYPE_COMPONENT, ordered, sticky,
                    token, sendingUser, intent.getFlags());
            this.intent = intent;
        }
        
        // 广播 Intent 对象！
        Intent intent;
        // 接收者的信息 ActivityInfo 对象！
        ActivityInfo info;
        CompatibilityInfo compatInfo;
    }
```
`ReceiverData` 继承了 `BroadcastReceiver.PendingResult` 对象，用来封装静态接收者的数据信息！

这里就和 Args 类似了！

### 3.2.2 H.handleMessage

我们去看看主线程 Handler H 是如何处理 H.RECEIVER 消息的：

```java
    private class H extends Handler {
    
        public void handleMessage(Message msg) {
            if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
            switch (msg.what) {
                case RECEIVER:
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "broadcastReceiveComp");
                    
                    //【×3.2.3】调用 handleReceiver 处理广播！
                    handleReceiver((ReceiverData)msg.obj);

                    maybeSnapshot();
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                    break;
            }
        }
    }
```

### 3.2.3 ActivityThread.handleReceiver
```java
    private void handleReceiver(ReceiverData data) {
        unscheduleGcIdler();

        //【1】获得组件名，用于反射！
        String component = data.intent.getComponent().getClassName();

        LoadedApk packageInfo = getPackageInfoNoCheck(
                data.info.applicationInfo, data.compatInfo);

        IActivityManager mgr = ActivityManagerNative.getDefault();
 
        BroadcastReceiver receiver;
        
        try {

            //【2】通过反射获得 BroadcastReceiver 对象的实例！
            java.lang.ClassLoader cl = packageInfo.getClassLoader();
            data.intent.setExtrasClassLoader(cl);
            data.intent.prepareToEnterProcess();
            data.setExtrasClassLoader(cl);
            receiver = (BroadcastReceiver)cl.loadClass(component).newInstance();

        } catch (Exception e) {
            if (DEBUG_BROADCAST) Slog.i(TAG,
                    "Finishing failed broadcast to " + data.intent.getComponent());
            data.sendFinished(mgr);
            throw new RuntimeException(
                "Unable to instantiate receiver " + component
                + ": " + e.toString(), e);
        }

        try {
            //【3】获得 Application 的上下文运行环境！
            Application app = packageInfo.makeApplication(false, mInstrumentation);

            if (localLOGV) Slog.v(
                TAG, "Performing receive of " + data.intent
                + ": app=" + app
                + ", appName=" + app.getPackageName()
                + ", pkg=" + packageInfo.getPackageName()
                + ", comp=" + data.intent.getComponent().toShortString()
                + ", dir=" + packageInfo.getAppDir());

            ContextImpl context = (ContextImpl)app.getBaseContext();
            sCurrentBroadcastIntent.set(data.intent);
            
            //【4】设置接收者的 PendingResult 属性！
            receiver.setPendingResult(data);
            
            //【5】拉起广播接收者的 onReceive 方法！！
            receiver.onReceive(context.getReceiverRestrictedContext(),
                    data.intent);

        } catch (Exception e) {
            if (DEBUG_BROADCAST) Slog.i(TAG,
                    "Finishing failed broadcast to " + data.intent.getComponent());
            //【×3.3.2】异常情况直接返回！     
            data.sendFinished(mgr);
            if (!mInstrumentation.onException(receiver, e)) {
                throw new RuntimeException(
                    "Unable to start receiver " + component
                    + ": " + e.toString(), e);
            }
        } finally {
            sCurrentBroadcastIntent.set(null);
        }
        
        //【6】调用 ReceiverData 的 finish 函数，通知 AMS 广播处理完成！
        if (receiver.getPendingResult() != null) {
            //【×3.3.1】实际上调用了 PendingResult 的 finish 方法！
            data.finish();
        }
    }
```
下面，我们进入到 `ReceiverData` 中去！！

## 3.3 PendingResult

### 3.3.1 PendingResult.finish

接着，调用 `PendingResult .finish` 结束广播的分发，通知系统继续广播的分发！
```java
        public final void finish() {
            // 如果 mType 为 TYPE_COMPONENT，静态接收者，进入 IF 分支！
            if (mType == TYPE_COMPONENT) {
                final IActivityManager mgr = ActivityManagerNative.getDefault();
                
                // 若主线程的 QueuedWork 中有事情还未处理完，则必须让事情做完后，才通知 AMS 结果！
                if (QueuedWork.hasPendingWork()) {

                    // 创建一个任务，通过 singleThreadExecutor 单线程依次执行！
                    QueuedWork.singleThreadExecutor().execute( new Runnable() {
                        @Override public void run() {
                            if (ActivityThread.DEBUG_BROADCAST) Slog.i(ActivityThread.TAG,
                                    "Finishing broadcast after work to component " + mToken);
                            sendFinished(mgr);
                        }
                    });
                } else {
                    if (ActivityThread.DEBUG_BROADCAST) Slog.i(ActivityThread.TAG,
                            "Finishing broadcast to component " + mToken);
                            
                            
                    //【×3.3.2】如果主线程无等待处理的事件，直接通知 AMS 结果！
                    sendFinished(mgr);
                }
            } else if (mOrderedHint && mType != TYPE_UNREGISTERED) {
                //【1】如果 mType 为 TYPE_REGISTERED : TYPE_UNREGISTERED，动态接收者，
                // 所以会进入 ELSE IF， mOrderedHint 为 true，表示为有序广播，此时我们需要通知系统！！
                if (ActivityThread.DEBUG_BROADCAST) Slog.i(ActivityThread.TAG,
                        "Finishing broadcast to " + mToken);

                final IActivityManager mgr = ActivityManagerNative.getDefault();

                //【×3.3.2】发送结果！
                sendFinished(mgr);
            }
        }
```

`mOrderedHint` 是创建 `Args` 时赋值初始化的，他是父类 `PendingResult` 的成员变量，表示该广播是否是有序的！


### 3.3.2 PendingResult.sendFinished

无论是 `Args`，还是 `ReceiverData` 他们都继承了 `PendingResult` 类，`sendFinished` 的实现是在 `PendingResult` 类中！

通过上面的方法可以看到，静态接收者由于都是有序分发，所以都要回调通知；但是动态接收者，只有广播是有序广播，才会回调通知系统，普通广播是不需要回调系统的！

```java
        /** @hide */
        public void sendFinished(IActivityManager am) {
            synchronized (this) {
                //【1】判断是否已经 finish 过，会抛出异常！
                if (mFinished) {
                    throw new IllegalStateException("Broadcast already finished");
                }
                mFinished = true;
            
                try {
                    if (mResultExtras != null) {
                        mResultExtras.setAllowFds(false);
                    }
                    
                    //【1】有序发送的广播，mOrderedHint 为 true，进入该分支！
                    // mAbortBroadcast 表示是否终止该广播的继续分发！
                    // mOrderedHint 表示该广播是否是有序发送的！
                    if (mOrderedHint) {
                        am.finishReceiver(mToken, mResultCode, mResultData, mResultExtras,
                                mAbortBroadcast, mFlags);
                    } else {
                        // 普通广播但是是发送给静态接收者的，也会回调系统！
                        am.finishReceiver(mToken, 0, null, null, false, mFlags);
                    }
                } catch (RemoteException ex) {
                }
            }
        }
```
这里的 `mToken` 要注意一下：

- 如果是动态注册的接收者，`mToken` 是 `InnerReceiver` 对象；
- 如果是静态注册的接收者，`mToken` 是 `ApplicationThread` 对象；

这里的 `mAbortBroadcast` 表示是否终止该广播的继续分发，对于有序发送的广播来说，当一个接收者接收到了该广播，可以通过以下方式来设置是否终止该广播的继续传递：

```java
    public final void abortBroadcast() {
        checkSynchronousHint();
        mPendingResult.mAbortBroadcast = true;
    }
```
以及：
```
    public final void clearAbortBroadcast() {
        if (mPendingResult != null) {
            mPendingResult.mAbortBroadcast = false;
        }
    }
```

下面就会进入系统进程！


# 4 系统进程

通过前面的分析，我们知道以下 2 种广播是通过有序的方式发送的：

- 目标是静态接收者的普通广播；
- 有序广播；

下面我们来看看有序发送的广播的后事处理：

## 4.1 ActivityManagerS.finishReceiver

参数 IBinder who 表示的是句柄，代表我们广播的目标客户端！

```java
    public void finishReceiver(IBinder who, int resultCode, String resultData,
            Bundle resultExtras, boolean resultAbort, int flags) {
        if (DEBUG_BROADCAST) Slog.v(TAG_BROADCAST, "Finish receiver: " + who);

        //【1】不能传递文件描述符！
        if (resultExtras != null && resultExtras.hasFileDescriptors()) {
            throw new IllegalArgumentException("File descriptors passed in Bundle");
        }

        final long origId = Binder.clearCallingIdentity();
        try {
            boolean doNext = false;
            BroadcastRecord r;

            synchronized(this) {
                //【2】根据广播的 Intent 是否设置了 FLAG_RECEIVER_FOREGROUND 标志位，选择前台/后台队列！
                BroadcastQueue queue = (flags & Intent.FLAG_RECEIVER_FOREGROUND) != 0
                        ? mFgBroadcastQueue : mBgBroadcastQueue;
                
                //【×4.1.1】接着是在 mOrderedBroadcasts 匹配到 who 对应的广播，也就是我们这次分发的广播！
                r = queue.getMatchingOrderedReceiver(who);
                
                if (r != null) {
                    //【×4.1.2】判断是否需要继续分发该广播！
                    doNext = r.queue.finishReceiverLocked(r, resultCode,
                        resultData, resultExtras, resultAbort, true);
                }
            }
            
            //【3】如果 doNext 为 true，继续下一次分发！
            if (doNext) {
                r.queue.processNextBroadcast(false);
            }
            
            // 回收资源！
            trimApplications();
        } finally {
            Binder.restoreCallingIdentity(origId);
        }
    }
```
流程很简单，不多说了！

### 4.1.1 BroadcastQueue.getMatchingOrderedReceiver
```java
    public BroadcastRecord getMatchingOrderedReceiver(IBinder receiver) {
        if (mOrderedBroadcasts.size() > 0) {
            final BroadcastRecord r = mOrderedBroadcasts.get(0);
            //【1】匹配接收者处理的广播，并返回！
            if (r != null && r.receiver == receiver) {
                return r;
            }
        }
        return null;
    }
```
`getMatchingOrderedReceiver` 逻辑很简单，这里就不多说了！

### 4.1.2 BroadcastQueue.finishReceiverLocked

接下来看看 finishReceiverLocked 方法做了什么！
```java
    public boolean finishReceiverLocked(BroadcastRecord r, int resultCode,
            String resultData, Bundle resultExtras, boolean resultAbort, boolean waitForServices) {
        //【1】保存之前的状态！
        final int state = r.state;
        final ActivityInfo receiver = r.curReceiver;
        
        //【2】初始化广播的一些变量！
        r.state = BroadcastRecord.IDLE;
        if (state == BroadcastRecord.IDLE) {
            Slog.w(TAG, "finishReceiver [" + mQueueName + "] called but state is IDLE");
        }
        
        //【3】将 r.receiver 置为 null，因为该接收者已经处理完了广播！
        r.receiver = null;
        r.intent.setComponent(null);

        if (r.curApp != null && r.curApp.curReceiver == r) {
            r.curApp.curReceiver = null;
        }
        if (r.curFilter != null) {
            r.curFilter.receiverList.curBroadcast = null;
        }
        r.curFilter = null;
        r.curReceiver = null;
        r.curApp = null;
        
        //【4】将 mPendingBroadcast 置为 null;
        mPendingBroadcast = null;

        r.resultCode = resultCode;
        r.resultData = resultData;
        r.resultExtras = resultExtras;
        
        //【5】判断是否需要终止该广播的继续分发！
        // 当 resultAbort 为 true，且广播没有设置 Intent.FLAG_RECEIVER_NO_ABORT 标志位，
        // 那么该广播就会终止继续传递！
        if (resultAbort && (r.intent.getFlags()&Intent.FLAG_RECEIVER_NO_ABORT) == 0) {
            r.resultAbort = resultAbort;
        } else {
            r.resultAbort = false;
        }
        
        //【6】这里的 waitForServices 传入的是 true，对于后台队列来说 mDelayBehindServices 为 true！
        if (waitForServices && r.curComponent != null && r.queue.mDelayBehindServices
                && r.queue.mOrderedBroadcasts.size() > 0
                && r.queue.mOrderedBroadcasts.get(0) == r) {

            ActivityInfo nextReceiver;
            if (r.nextReceiver < r.receivers.size()) {
                //【6.1】计算下一个接收者！
                Object obj = r.receivers.get(r.nextReceiver);
                nextReceiver = (obj instanceof ActivityInfo) ? (ActivityInfo)obj : null;
            } else {
                nextReceiver = null;
            }
            // 如果本次接受广播的接收者和下一个接收者不在同一个进程，或者下一个接收者为 null
            if (receiver == null || nextReceiver == null
                    || receiver.applicationInfo.uid != nextReceiver.applicationInfo.uid
                    || !receiver.processName.equals(nextReceiver.processName)) {

                // 对于后台队列中的有序发送的广播，如果广播的目标设别用户下有需要后台延迟启动的服务
                // 那就不能继续分发，这里会将广播的状态改为 BroadcastRecord.WAITING_SERVICES，返回 false！
                if (mService.mServices.hasBackgroundServices(r.userId)) {
                    Slog.i(TAG, "Delay finish: " + r.curComponent.flattenToShortString());。。。。
                    r.state = BroadcastRecord.WAITING_SERVICES;
                    return false;
                }
            }
        }

        r.curComponent = null;

        //【7】前面我们知道，由于发送广播是异步操作，所以正常情况发送前，状态就改为了下面的状态；
        // 这是返回的是 true，那么 doNext 就是 true；如果需要等待 Service 启动，那么状态就为 WAITING_SERVICES；
        // 此时 doNext 就为 false；
        return state == BroadcastRecord.APP_RECEIVE
                || state == BroadcastRecord.CALL_DONE_RECEIVE;
    }
```
对于静态注册的接收者 r.state 被置为 BroadcastRecord.APP_RECEIVE；
对于动态注册的接收者 r.state 被置为 BroadcastRecord.CALL_DONE_RECEIVE；


这里的 `r.queue.mDelayBehindServices` 要解释一下，他是在创建前台和后台队列时初始化的：
```java
    mFgBroadcastQueue = new BroadcastQueue(this, mHandler, "foreground", BROADCAST_FG_TIMEOUT, false);
    mBgBroadcastQueue = new BroadcastQueue(this, mHandler, "background", BROADCAST_BG_TIMEOUT, true);
```
可以看到，对于后台队列，其 mDelayBehindServices 为 true！表示如果有后台延迟启动的服务，就延迟广播的发送！

那么我们如何判断目标设备用户下是否有后台延迟发送的广播，这就要调用 ActiveServices 的 hasBackgroundServices 方法了：
```java
    boolean hasBackgroundServices(int callingUser) {
        ServiceMap smap = mServiceMap.get(callingUser);
        return smap != null ? smap.mStartingBackground.size() >= mMaxStartingBackground : false;
    }
```
和 `Service` 的生命周期相关的内容，请去看其他的博客！


最后又再次回到了 `processNextBroadcast` 方法中了！

# 5 总结

到这里，`sendBroadcast` 方法的流程就分析完了，下面我们来总结一下：

## 5.1 广播的处理流程

在发送广播之前，都需要收集其目标接收者，包括静态接收者和动态接收者！

如果广播设置了 `Intent.FLAG_RECEIVER_FOREGROUND` 标志位，会被添加到前台队列 `mFgBroadcastQueue` 中，否则就被添加到后台队列 `mBgBroadcastQueue` 中！

如果广播设置了 `Intent.FLAG_RECEIVER_NO_ABORT` 标志位，有序发送的情况下不能被强制中断分发！

### 5.1.1 普通广播的处理流程

- **1** 先发送给**动态接收者**，并行分发，广播会被加入到 `mParallelBroadcasts` 集合中，通过一个 `while` 循环，不断遍历 `mParallelBroadcasts`，直到为空 。
    - **1.1** 设置广播开始分发时间：`r.dispatchTime = SystemClock.uptimeMillis();` 和 `r.dispatchClockTime = System.currentTimeMillis();`
    - **1.2** 设置当前接收者的分发状态：`r.delivery[index] = BroadcastRecord.DELIVERY_DELIVERED`
    - **1.3** 对于动态接收者来说，系统进程保存着其 `IIntentReceiver.Proxy` 对象！
    - **1.4** 通过 `app.thread.scheduleRegisteredReceiver`，进入应用进程，调用 `InnerReceiver.performReceive` 拉起广播接收者的 `onReceive` 方法！如果 `app = null`，就直接通过 `IIntentReceiver.Proxy.performReceive`，进入应用进程！
	      - **1.4.1** 这时会创建一个 `Args` 对象（继承了 `BroadcastReceiver.PendingResult` 实现了 `Runnable`），通过主线程执行 `Args.run` 方法！
	      - **1.4.2** `PendingResult` 有一个内部成员变量 `mType`，用来保存接收者的类型，对于动态接收者：`mType = TYPE_REGISTERED/TYPE_UNREGISTERED`！
          - **1.4.3** 最后通过反射，拉起 `onReceive` 方法！
	
</br>	

- **2** 然后发送给**静态接收者**，广播会被加入到 `mOrderedBroadcasts` 集合中，有序发送！
    - **2.1** 如果已经有一个正在等待目标进程启动的广播 `mPendingBroadcast`，那就等待对应进程启动后处理！
    - **2.2** 否则，计算本次该广播要分发的接收者的序号：`recIdx`，和下一个接收者的序号：`r.nextReceiver`，二者满足：`int recIdx = r.nextReceiver++`
    - **2.3** 设置广播开始分发时间（当 `recIdx` 为第一个接收者的时候）`r.dispatchTime = r.receiverTime;　r.dispatchClockTime = System.currentTimeMillis();`
       - 设置当前接收者的分发状态：`r.delivery[recIdx] = BroadcastRecord.DELIVERY_DELIVERED;`
       - 设置广播的当前状态：`r.state = BroadcastRecord.APP_RECEIVE;`
       - 设置广播当前的目标组件：`r.curComponent = component;`
       - 设置广播当前的目标接收者：`r.curReceiver = info.activityInfo;`
		
	- **2.4** 对于静态接收者，发送广播时候，其进程未必启动，所以要分情况处理：
		  - **2.4.1** 如果进程未启动，就先启动其进程，并将广播添加到 `mPendingBroadcast` 中，等待进程启动后处理！
			   进程启动后，会判断当前进程是否有 `mPendingBroadcast`，有的话，就将 `mPendingBroadcast` 发送目标接收者（2.4.2），并设置 `mPendingBroadcast = null;`！

		    </br>

		  - **2.4.2** 如果进程已启动，那就直接发送广播！
			   - 设置广播的目标接收者通信对象：`r.receiver = app.thread.asBinder();`，即目标进程的 `ApplicationThreadProxy` 对象！
			   - 设置广播的目标进程：`r.curApp = app;`
			   - 设置目标进程的当前要接收广播的接收者：`app.curReceiver = r;`
			   - 设置广播的组件信息；`r.intent.setComponent(r.curComponent);`
			   
			   - 通过 `app.thread.scheduleReceiver`，跨进程拉起接收者的 `onReceive` 方法！
			       - 进入应用进程后，会创建一个 `ReceiverData` 对象（其继承了 `BroadcastReceiver.PendingResult`）
			       - `PendingResult` 有一个内部成员变量 `mType`，用来保存接收者的类型，对于静态接收者：`mType =  TYPE_COMPONENT`！
                   - 最后通过反射，拉起 `onReceive` 方法！
	- **2.5** 最后，调用 `ActivityManagerService.finishReceiver`，通知系统进程，接收者完成了广播处理，继续分发!！
       - 初始化广播的状态：`r.state = BroadcastRecord.IDLE;`
		- 清空广播的目标接收者：`Binder` 对象：`r.receiver = null;`
		- 清空广播组件信息：`r.intent.setComponent(null);`
		- 清空广播目标进程的当前接收者：`r.curApp.curReceiver = null;`
		- 清空广播过滤器的当前广播：`r.curFilter.receiverList.curBroadcast = null;`
								  
		- 清空广播的过滤器：`r.curFilter = null;`
		- 清空广播接收者信息：`r.curReceiver = null;`
		- 清空广播的目标进程：`r.curApp = null;`
		- 清空广播的组件信息：`r.curComponent = null;`

### 5.1.2 有序广播的处理流程

对于有序广播来说，会将收集到的静态接收者和动态接收者通过优先级的不同合并到同一个集合中去，然后将该集合添加到指定队列的 `mOrderedBroadcasts` 集合中！

- **1** 如果已经有一个正在等待目标进程启动的广播 `mPendingBroadcast`，那就等待对应进程启动后处理，此时不能继续有序发送！

</br>

- **2** 从 `mOrderedBroadcasts` 中移除不需要发送的广播，比如：没有接收者，或者所有接收者都已接收了该广播或者被终止继续传递，或者超时了，找到下一个需要分发的广播 r！

</br>

- **3** 准备工作：
   - 计算本次该广播的目标接收者的序号：`recIdx`，和下一个接收者的序号：`r.nextReceiver`，二者满足：`int recIdx = r.nextReceiver++`；
   - 初始化该广播的接收时间： `r.receiverTime = SystemClock.uptimeMillis();`，根据该事件设置超时任务！
   - 设置广播开始分发时间： `r.dispatchTime = r.receiverTime;　r.dispatchClockTime = System.currentTimeMillis();`（当 `recIdx` 为第一个接收者的时候）

</br>

- **4** 从广播的接收者列表中找到 `recIdx` 对应的目标接收者，接下来就是分发广播了：
   - **4.1** 如果是**动态注册的接收者**，`deliverToRegisteredReceiverLocked`！
     - **4.1.1** 设置目标接收者的分发状态： `r.delivery[index] = BroadcastRecord.DELIVERY_DELIVERED`
	 - **4.1.2** 设置广播的属性：
	     - 设置广播接收者对应的 `Binder` 通信对象：`r.receiver = filter.receiverList.receiver.asBinder();`，即 `IIntentReceiver.Proxy` 对象！
	     - 设置广播的过滤器对象： `r.curFilter = filter;`
	     - 设置过滤器处理的广播：  `filter.receiverList.curBroadcast = r;`
	     - 设置广播的状态： `r.state = BroadcastRecord.CALL_IN_RECEIVE;`
		 - 设置广播的目标进程：  `r.curApp = filter.receiverList.app;`
		 - 设置目标进程当前处理的广播：  `filter.receiverList.app.curReceiver = r;`
		- **4.1.3** 立刻设置广播的状态：`r.state = BroadcastRecord.CALL_DONE_RECEIVE;` 表示接收完毕！
		- **4.1.4** 发送广播！
		  - **4.1.4.1** 通过 `app.thread.scheduleRegisteredReceiver`，进入应用进程，调用 `InnerReceiver.performReceive` 拉起广播接收者的 `onReceive` 方法！如果 `app = null`，就直接通过 `IIntentReceiver.Proxy.performReceive`，进入应用进程！
		  - **4.1.4.2** 调用 `ReceiverDispatcher.performReceive` 方法，处理广播！      
			 - 这时会创建一个 `Args` 对象（继承了 `BroadcastReceiver.PendingResult` 实现了 `Runnable`），通过主线程执行 `Args.run` 方法！
			 - `PendingResult` 有一个内部成员变量 `mType`，用来保存接收者的类型，对于动态接收者：`mType = TYPE_REGISTERED/TYPE_UNREGISTERED`！
			 - 最后通过反射，拉起 `onReceive` 方法！
			 - 调用 `ActivityManagerService.finishReceiver`，通知系统进程，接收者完成了广播处理，继续分发！
				 - 初始化广播的状态：`r.state = BroadcastRecord.IDLE;`
				 - 清空广播的目标接收者：Binder 对象：`r.receiver = null;`
				 - 清空广播组件信息：`r.intent.setComponent(null);`
			 	 - 清空广播目标进程的当前接收者：`r.curApp.curReceiver = null;`
				 - 清空广播过滤器的当前广播：`r.curFilter.receiverList.curBroadcast = null;`
						  
				 - 清空广播的过滤器：`r.curFilter = null;`
				 - 清空广播接收者信息：`r.curReceiver = null;`
                 - 清空广播的目标进程：`r.curApp = null;`
				 - 清空广播的组件信息：`r.curComponent = null;`



	- **4.2** 如果是**静态注册的接收者**，`deliverToRegisteredReceiverLocked`！
		- **4.1.1** 设置目标接收者的分发状态：  `r.delivery[index] = BroadcastRecord.DELIVERY_DELIVERED`
		- **4.1.2** 设置广播的属性：
		     - 设置广播的状态：`r.state = BroadcastRecord.APP_RECEIVE;`
             - 设置广播的目标组件信息：`r.curComponent = component;`
             - 设置广播的目标接收者信息：`r.curReceiver = info.activityInfo;`
					
		- **4.1.3** 根据静态接收者的进程状态来做不同处理：
		    - **4.1.3.1** 如果进程未启动，就先启动其进程，并将广播添加到 `mPendingBroadcast` 中，等待进程启动后处理！进程启动后，会判断当前进程是否有 `mPendingBroadcast`，有的话，就将 `mPendingBroadcast` 发送目标接收者（4.1.3.2），并设置 `mPendingBroadcast = null;`！
				 
			- **4.1.3.2** 如果进程已启动，那就直接发送广播！
			 - 设置广播的目标接收者通信对象：`r.receiver = app.thread.asBinder();`，即目标进程的 `ApplicationThreadProxy` 对象！
			 - 设置广播的目标进程：`r.curApp = app;`
			 - 设置目标进程的当前要接收广播的接收者：`app.curReceiver = r;`
			 - 设置广播的组件信息；`r.intent.setComponent(r.curComponent);`
			   
			 - 通过 `app.thread.scheduleReceiver`，跨进程拉起接收者的 `onReceive` 方法！
			 - 进入应用进程后，会创建一个 `ReceiverData` 对象（其继承了 `BroadcastReceiver.PendingResult`）
			 - `PendingResult` 有一个内部成员变量 `mType`，用来保存接收者的类型，对于静态接收者：`mType =  TYPE_COMPONENT`！
             - 最后通过反射，拉起 `onReceive` 方法！
						   
			 - 调用 `ActivityManagerService.finishReceiver`，通知系统进程，接收者完成了广播处理，继续广播分发！
				    - 初始化广播的状态：`r.state = BroadcastRecord.IDLE;`
					- 清空广播的目标接收者：`Binder` 对象：`r.receiver = null;`
					- 清空广播组件信息：`r.intent.setComponent(null);`
					- 清空广播目标进程的当前接收者：`r.curApp.curReceiver = null;`
					- 清空广播过滤器的当前广播：`r.curFilter.receiverList.curBroadcast = null;`
								  
					- 清空广播的过滤器：`r.curFilter = null;`
					- 清空广播接收者信息：`r.curReceiver = null;`
					- 清空广播的目标进程：`r.curApp = null;`
					- 清空广播的组件信息：`r.curComponent = null;`

### 5.1.3 粘性广播的处理流程

粘性是一个附加的属性，和有序，普通不冲突，普通广播和有序广播都可以是粘性的，粘性广播的一个特性是，系统会将该粘性广播保存下来，以便分发给以后注册的接收者！

保存的集合为：

```java
    final SparseArray<ArrayMap<String, ArrayList<Intent>>> mStickyBroadcasts =
            new SparseArray<ArrayMap<String, ArrayList<Intent>>>();
```

除去这个特性之外，粘性广播的处理方式和普通广播、有序广播是一样的，这里就不多说了！

- 如果要发送粘性广播，应用必须配置 android.Manifest.permission.BROADCAST_STICKY 权限！
- 对于粘性广播，不能强制广播接收者应该具有某些权限！
- 对于粘性广播，如果有相同的全局粘性广播，会产生冲突，抛出异常！
- Android 7.1.1 上已经不推荐使用粘性广播了，因为会产生安全问题！

## 5.2 广播的状态周期

对于广播来说，会有如下的几种状态：
```java
    static final int IDLE = 0;
    static final int APP_RECEIVE = 1;
    static final int CALL_IN_RECEIVE = 2;
    static final int CALL_DONE_RECEIVE = 3;
    static final int WAITING_SERVICES = 4;
```

- 普通广播：

如果是静态注册的接收者：
```java
BroadcastReocrd.IDLE -> BroadcastReocrd.IDLE.APP_RECEIVE -> BroadcastReocrd.CALL_DONE_RECEIVE ->（BroadcastReocrd.WAITING_SERVICES）-> BroadcastReocrd.IDLE
```
如果是动态注册的接收者：
```java
BroadcastReocrd.IDLE
```

- 有序广播：

如果是静态注册的接收者：
```java
BroadcastReocrd.IDLE -> BroadcastReocrd.APP_RECEIVE -> BroadcastReocrd.CALL_DONE_RECEIVE ->（BroadcastReocrd.WAITING_SERVICES）-> BroadcastReocrd.IDLE
```
如果是动态注册的接收者：
```java
BroadcastReocrd.IDLE -> BroadcastReocrd.CALL_IN_RECEIVE -> BroadcastReocrd.CALL_DONE_RECEIVE ->（BroadcastReocrd.WAITING_SERVICES）-> BroadcastReocrd.IDLE
```
这里的 `BroadcastReocrd.WAITING_SERVICES` 只会对后台队列中有序发送的广播有效，因为这些广播可能在等待一些后台服务的启动！

当后台服务启动后，`ActiveServices` 会通过 `rescheduleDelayedStarts` 方法拉起这些服务！这里不多说了！

## 5.3 接收者的分发状态

对于接收者来说，下面这些状态用来表示广播的分发状态：

```java
    static final int DELIVERY_PENDING = 0; // 没有用到！
    static final int DELIVERY_DELIVERED = 1; // 广播已经分发给该广播接收者
    static final int DELIVERY_SKIPPED = 2; // 广播跳过了该广播接收者
    static final int DELIVERY_TIMEOUT = 3; // 广播超时了
```

关于广播的超时，我们会单独一篇博文中介绍！