# PMS 第 11 篇 - 通过 adb 指令分析 hide/unhide 过程
title: PMS 第 11 篇 - 通过 adb 指令分析 hide/unhide 过程
date: 2018/09/11
categories:
- AndroidFramework源码分析
- PackageManager包管理
tags: PackageManager包管理
---

[toc]

基于 Android7.1.1 分析 PackageManagerService 的架构设计！
  
# 0 综述  

本文来分析下 pms hide 相关的操作：

- adb shell pm hide
- adb shell pm unhide

这个指令可以让一个 package 被 hide，无法被找到，同样的，我们从 Pm 中看起！


# 1 Pm

## 1.1 run

和其他方法的调用逻辑一样，进入 run 方法：

```java
    public int run(String[] args) throws RemoteException {
        boolean validCommand = false;
        if (args.length < 1) {
            return showUsage();
        }
        mAm = IAccountManager.Stub.asInterface(ServiceManager.getService(Context.ACCOUNT_SERVICE));
        mUm = IUserManager.Stub.asInterface(ServiceManager.getService(Context.USER_SERVICE));
        mPm = IPackageManager.Stub.asInterface(ServiceManager.getService("package"));

        if (mPm == null) {
            System.err.println(PM_NOT_RUNNING_ERR);
            return 1;
        }
        mInstaller = mPm.getPackageInstaller();

        mArgs = args;
        String op = args[0];
        mNextArg = 1;

        ... ... ...

        if ("hide".equals(op)) {
            //【*1.2】调用自身的另一个方法！
            return runSetHiddenSetting(true);
        }

        if ("unhide".equals(op)) {
            //【*1.2】调用自身的另一个方法！
            return runSetHiddenSetting(false);
        }

        ... ... ...
    }
```


## 1.2 runSetHiddenSetting

我们来看下 runSetHiddenSetting 的调用逻辑：

```java
    private int runSetHiddenSetting(boolean state) {
        //【1】默认要 hide 所在的用户，为 USER_SYSTEM(0)
        int userId = UserHandle.USER_SYSTEM;
        String option = nextOption();
        //【2】如果通过 --user 指定了 hide 的 user。那就初始化为该 user！
        if (option != null && option.equals("--user")) {
            String optionData = nextOptionData();
            if (optionData == null || !isNumber(optionData)) {
                System.err.println("Error: no USER_ID specified");
                return showUsage();
            } else {
                userId = Integer.parseInt(optionData);
            }
        }
        //【3】获得要 hide 的应用包名；
        String pkg = nextArg();
        if (pkg == null) {
            System.err.println("Error: no package or component specified");
            return showUsage();
        }
        try {
            //【*2.1】进入 pms！
            mPm.setApplicationHiddenSettingAsUser(pkg, state, userId);
            System.out.println("Package " + pkg + " new hidden state: "
                    + mPm.getApplicationHiddenSettingAsUser(pkg, userId));
            return 0;
        } catch (RemoteException e) {
            System.err.println(e.toString());
            System.err.println(PM_NOT_RUNNING_ERR);
            return 1;
        }
    }
```
不多说了！

# 2 PackageManagerService

进入 pms 中去：

## 2.1 setApplicationHiddenSettingAsUser

```java
    @Override
    public boolean setApplicationHiddenSettingAsUser(String packageName, boolean hidden,
            int userId) {
        //【1】首先校验下是否有 MANAGE_USERS 权限以及跨用户的权限；
        mContext.enforceCallingOrSelfPermission(android.Manifest.permission.MANAGE_USERS, null);
        PackageSetting pkgSetting;
        final int uid = Binder.getCallingUid();
        enforceCrossUserPermission(uid, userId,
                true /* requireFullPermission */, true /* checkShell */,
                "setApplicationHiddenSetting for user " + userId);
        //【2】如果要 hide 的 package 是 device admin，禁止 hide！
        if (hidden && isPackageDeviceAdmin(packageName, userId)) {
            Slog.w(TAG, "Not hiding package " + packageName + ": has active device admin");
            return false;
        }

        long callingId = Binder.clearCallingIdentity();
        try {
            boolean sendAdded = false;
            boolean sendRemoved = false;
            // writer
            synchronized (mPackages) {
                //【3】获得该 package 对应的 PackageSetting 实例；
                pkgSetting = mSettings.mPackages.get(packageName);
                if (pkgSetting == null) {
                    return false;
                }
                //【4】不允许 "android" 被 hide！
                if ("android".equals(packageName)) {
                    Slog.w(TAG, "Cannot hide package: android");
                    return false;
                }
                //【5】只允许受保护的 package hide 他们自己！
                if (hidden && !UserHandle.isSameApp(uid, pkgSetting.appId)
                        && mProtectedPackages.isPackageStateProtected(userId, packageName)) {
                    Slog.w(TAG, "Not hiding protected package: " + packageName);
                    return false;
                }

                //【6】处理 hide 状态，如果本次设置的和上一次的 hide 状态不一样，那么就需要更新 hide 状态！
                //【*2.1.1】获得该 package 在 userId 下的 PackageUserState 实例，然后获得其 hide 值；
                if (pkgSetting.getHidden(userId) != hidden) {
                    //【*2.1.2】设置该 package 在 userId 下的 hide 状态！
                    pkgSetting.setHidden(hidden, userId);
                    //【6.1】更新偏好设置的本地化文件；
                    mSettings.writePackageRestrictionsLPr(userId);
                    //【6.2】如果本次是 hide，sendRemoved 为 true；如果本次是 unhide，sendAdded 为 true；
                    if (hidden) {
                        sendRemoved = true;
                    } else {
                        sendAdded = true;
                    }
                }
            }
            //【7】如果是 unhide，进入这里；
            if (sendAdded) {
                //【*2.2】发送 pkg add 的广播！
                sendPackageAddedForUser(packageName, pkgSetting, userId);
                return true;
            }
            //【8】如果是 hide，进入这里；
            if (sendRemoved) {
                //【*2.3】杀掉 pkg 进程；
                killApplication(packageName, UserHandle.getUid(userId, pkgSetting.appId),
                        "hiding pkg");
                //【*2.4】发送 pkg add 的广播！
                sendApplicationHiddenForUser(packageName, pkgSetting, userId);
                return true;
            }
        } finally {
            Binder.restoreCallingIdentity(callingId);
        }
        return false;
    }

```

### 2.1.1 PackageSetting.getHidden

```java
    boolean getHidden(int userId) {
        //【*2.1.1.1】获得对应的 PackageUserState.hide 值；
        return readUserState(userId).hidden;
    }
```
继续来看：

#### 2.1.1.1 readUserState

```java
    public PackageUserState readUserState(int userId) {
        //【1】在 userId 下的 PackageUserState 实例！ 
        PackageUserState state = userState.get(userId);
        if (state != null) {
            return state;
        }
        return DEFAULT_USER_STATE;
    }
```

### 2.1.2 PackageSetting.setHidden

```java
    void setHidden(boolean hidden, int userId) {
        //【*2.1.2.1】设置对应 userId 下的 PackageUserState 的 hide 状态；
        modifyUserState(userId).hidden = hidden;
    }
```

#### 2.1.2.1 modifyUserState

该方法其实很简单，不多说了！

```java
    private PackageUserState modifyUserState(int userId) {
        PackageUserState state = userState.get(userId);
        if (state == null) {
            state = new PackageUserState();
            userState.put(userId, state);
        }
        return state;
    }
```

## 2.2 sendPackageAddedForUser[3]

给指定的 user 发送 Intent.ACTION_PACKAGE_ADDED 广播：

```java
    private void sendPackageAddedForUser(String packageName, PackageSetting pkgSetting,
            int userId) {
        //【1】是不是 sys app;
        final boolean isSystem = isSystemApp(pkgSetting) || isUpdatedSystemApp(pkgSetting);
        //【*2.2.1】调用另一方法：
        sendPackageAddedForUser(packageName, isSystem, pkgSetting.appId, userId);
    }
```
继续来看：

### 2.2.1 sendPackageAddedForUser[4]


```java
    private void sendPackageAddedForUser(String packageName, boolean isSystem,
            int appId, int userId) {
        Bundle extras = new Bundle(1);
        //【1】获得 inhide 的应用所在的 user！
        extras.putInt(Intent.EXTRA_UID, UserHandle.getUid(userId, appId));
        //【*2.2.1.1】发送 Intent.ACTION_PACKAGE_ADDED 广播；
        sendPackageBroadcast(Intent.ACTION_PACKAGE_ADDED,
                packageName, extras, 0, null, null, new int[] {userId});
        try {
            IActivityManager am = ActivityManagerNative.getDefault();

            //【2】如果 unhide 的是 sys app，并且 userId 正在运行中；
            if (isSystem && am.isUserRunning(userId, 0)) {
                // The just-installed/enabled app is bundled on the system, so presumed
                // to be able to run automatically without needing an explicit launch.
                // Send it a BOOT_COMPLETED if it would ordinarily have gotten one.
                //【2.1】这里会发送一个 boot completed 广播给这个 pkg，给它一个引导；
                Intent bcIntent = new Intent(Intent.ACTION_BOOT_COMPLETED)
                        .addFlags(Intent.FLAG_INCLUDE_STOPPED_PACKAGES)
                        .setPackage(packageName);
                am.broadcastIntent(null, bcIntent, null, null, 0, null, null, null,
                        android.app.AppOpsManager.OP_NONE, null, false, false, userId);
            }
        } catch (RemoteException e) {
            // shouldn't happen
            Slog.w(TAG, "Unable to bootstrap installed package", e);
        }
    }
```
不多说了！

#### 2.2.1.1 sendPackageBroadcast

发送广播的代码如下：

```java
    final void sendPackageBroadcast(final String action, final String pkg, final Bundle extras,
            final int flags, final String targetPkg, final IIntentReceiver finishedReceiver,
            final int[] userIds) {
        mHandler.post(new Runnable() {
            @Override
            public void run() {
                try {
                    final IActivityManager am = ActivityManagerNative.getDefault();
                    if (am == null) return;
                    //【1】要发送的目标 userIds；
                    final int[] resolvedUserIds;
                    if (userIds == null) {
                        resolvedUserIds = am.getRunningUserIds();
                    } else {
                        resolvedUserIds = userIds;
                    }
                    //【2】开始发送广播：
                    for (int id : resolvedUserIds) {
                        final Intent intent = new Intent(action,
                                pkg != null ? Uri.fromParts(PACKAGE_SCHEME, pkg, null) : null);
                        if (extras != null) {
                            intent.putExtras(extras);
                        }
                        if (targetPkg != null) {
                            intent.setPackage(targetPkg);
                        }
                        //【2.1】计算 package 在目标 userId 下的 uid！
                        int uid = intent.getIntExtra(Intent.EXTRA_UID, -1);
                        if (uid > 0 && UserHandle.getUserId(uid) != id) {
                            uid = UserHandle.getUid(id, UserHandle.getAppId(uid));
                            intent.putExtra(Intent.EXTRA_UID, uid);
                        }
                        //【2.2】计算目标 userId！
                        intent.putExtra(Intent.EXTRA_USER_HANDLE, id);
                        intent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY_BEFORE_BOOT | flags);
                        if (DEBUG_BROADCASTS) {
                            RuntimeException here = new RuntimeException("here");
                            here.fillInStackTrace();
                            Slog.d(TAG, "Sending to user " + id + ": "
                                    + intent.toShortString(false, true, false, false)
                                    + " " + intent.getExtras(), here);
                        }
                        //【2.3】发送广播！
                        am.broadcastIntent(null, intent, null, finishedReceiver,
                                0, null, null, null, android.app.AppOpsManager.OP_NONE,
                                null, finishedReceiver != null, false, id);
                    }
                } catch (RemoteException ex) {
                }
            }
        });
    }
```
这就不多说了！

## 2.3 killApplication[3]

杀掉应用的进程：
```java
    private void killApplication(String pkgName, int appId, String reason) {
        //【*2.3.1】调用另一个方法：
        killApplication(pkgName, appId, UserHandle.USER_ALL, reason);
    }
```
### 2.3.1 killApplication[4]

另一个 4 参数的 kill：
```java
    private void killApplication(String pkgName, int appId, int userId, String reason) {
        // Request the ActivityManager to kill the process(only for existing packages)
        // so that we do not end up in a confused state while the user is still using the older
        // version of the application while the new one gets installed.
        final long token = Binder.clearCallingIdentity();
        try {
            IActivityManager am = ActivityManagerNative.getDefault();
            if (am != null) {
                try {
                    //【1】杀掉进程；
                    am.killApplication(pkgName, appId, userId, reason);
                } catch (RemoteException e) {
                }
            }
        } finally {
            Binder.restoreCallingIdentity(token);
        }
    }
```
对于 killApplication 杀进程的流程，这里就不再分析了！

## 2.4 sendApplicationHiddenForUser

给指定的 user 发送 Intent.ACTION_PACKAGE_REMOVED 广播：

```java
    private void sendApplicationHiddenForUser(String packageName, PackageSetting pkgSetting,
            int userId) {
        //【*2.4.1】创建了一个 PackageRemovedInfo！
        final PackageRemovedInfo info = new PackageRemovedInfo();
        //【1】初始化 removedPackage，removedUsers 和 uid；
        info.removedPackage = packageName;
        info.removedUsers = new int[] {userId};
        info.uid = UserHandle.getUid(userId, pkgSetting.appId);
        //【*2.4.2】调用了 PackageRemovedInfo 的 sendPackageRemovedBroadcasts 方法！
        info.sendPackageRemovedBroadcasts(true /*killApp*/);
    }
```
这里不多数说了！

### 2.4.1 new PackageRemovedInfo

对于 PackageRemovedInfo 这里简单看下：

```java
    class PackageRemovedInfo {
        String removedPackage;  // 要 hide 的 packageName
        int uid = -1;    // 该 pkg 在指定的 userId 下的 uid
        int removedAppId = -1;  // 该 pkg 的 appId
        int[] origUsers;
        int[] removedUsers = null; // 要移除的 userId
        boolean isRemovedPackageSystemUpdate = false;
        boolean isUpdate;
        boolean dataRemoved; // 是否移除数据，默认 false 不移除
        boolean removedForAllUsers; // 是否从所有 user 下移除，默认 false；
        // Clean up resources deleted packages.
        InstallArgs args = null;
        ArrayMap<String, PackageRemovedInfo> removedChildPackages;
        ArrayMap<String, PackageInstalledInfo> appearedChildPackages;
        ... ... ...
    }
```

### 2.4.2 PackageRemovedInfo.sendPackageRemovedBroadcasts

```java
        void sendPackageRemovedBroadcasts(boolean killApp) {
            //【*2.4.2.1】调用另一个 send 方法；
            sendPackageRemovedBroadcastInternal(killApp);
            //【1】对于 child packages 执行相同的操作！
            final int childCount = removedChildPackages != null ? removedChildPackages.size() : 0;
            for (int i = 0; i < childCount; i++) {
                PackageRemovedInfo childInfo = removedChildPackages.valueAt(i);
                //【*2.4.2.1】调用另一个 send 方法；
                childInfo.sendPackageRemovedBroadcastInternal(killApp);
            }
        }
```

#### 2.4.1.1 PackageRemovedInfo.sendPackageRemovedBroadcastInternal

```java
        private void sendPackageRemovedBroadcastInternal(boolean killApp) {
            //【1】创建了一个 Bundle 对象；
            Bundle extras = new Bundle(2);
            //【2】保存要发送的数据：uid，data_moved，kill_app 等；
            extras.putInt(Intent.EXTRA_UID, removedAppId >= 0  ? removedAppId : uid);
            extras.putBoolean(Intent.EXTRA_DATA_REMOVED, dataRemoved);
            extras.putBoolean(Intent.EXTRA_DONT_KILL_APP, !killApp);

            //【3】显然，对于 hide 是不进入这里的；
            if (isUpdate || isRemovedPackageSystemUpdate) {
                extras.putBoolean(Intent.EXTRA_REPLACING, true);
            }
            extras.putBoolean(Intent.EXTRA_REMOVED_FOR_ALL_USERS, removedForAllUsers);

            //【4】准备发送广播：
            if (removedPackage != null) {
                //【*2.2.1.1】发送 Intent.ACTION_PACKAGE_REMOVED 广播；
                sendPackageBroadcast(Intent.ACTION_PACKAGE_REMOVED, removedPackage,
                        extras, 0, null, null, removedUsers);
                        
                //【4.1】如果需要移除数据，同时 remove 的是 sys app 自身，说明这是一个完全移除；
                //（hide 是不会进入这个逻辑的！）
                if (dataRemoved && !isRemovedPackageSystemUpdate) {
                    //【*2.2.1.1】发送 Intent.ACTION_PACKAGE_FULLY_REMOVED 广播；
                    sendPackageBroadcast(Intent.ACTION_PACKAGE_FULLY_REMOVED,
                            removedPackage, extras, 0, null, null, removedUsers);
                }
            }
            //【5】如果 removedAppId >= 0，说明应用对应的 uid 被移除了！
            // 然而这里 app 被 hide 了，其 app id 依然存在，所以不会发送对应的广播；
            //（hide 是不会进入这个逻辑的！）
            if (removedAppId >= 0) {
                //【*2.2.1.1】发送 Intent.ACTION_UID_REMOVED 广播；
                sendPackageBroadcast(Intent.ACTION_UID_REMOVED, null, extras, 0, null, null,
                        removedUsers);
            }
        }
```
这里就不多说了！！