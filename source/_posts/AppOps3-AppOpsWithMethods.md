# AppOps 第 3 篇 - AppOps 权限相关方法
title: AppOps 第 3 篇 - AppOps 权限相关方法
date: 2017/09/10 20:46:25
catalog: true
categories:
- AndroidFramework源码分析
- AppOps应用操作管理
tags: AppOps应用操作管理
---

本文基于 Android 7.1.1 源码分析，如有错误，欢迎指正，谢谢！

[toc]

# 0 综述

前面分析了 AppOpsManager 中的一些接口，最终都会调用 AppOpservice 中的相应接口，下面我们来看下 AppOpservice 中的权限操作！

涉及的类如下：

- Settings 上层应用入口：

```java
packages/apps/Settings/src/com/android/settings/applications/AppOpsCategory.java
packages/apps/Settings/src/com/android/settings/applications/AppOpsDetails.java
packages/apps/Settings/src/com/android/settings/applications/AppOpsState.java
packages/apps/Settings/src/com/android/settings/applications/AppOpsSummary.java
```
- appops 的可执行文件：

位于 /system/bin/ 目录下！

```
frameworks/native/libs/binder/AppOpsManager.cpp
frameworks/native/include/binder/AppOpsManager.h
```

- 框架层 AppOps 核心实现类：

AppOpsService 是功能具体实现；AppOpsManager 是 AppOpsService 提供给其他进程的一个入口； AppOpsPolicy大概意思是，系统出厂时预制的系统 App(System-app)和用户 app（User-app）的一些默认的权限！
```
frameworks/base/services/core/java/com/android/server/AppOpsService.java
frameworks/base/core/java/android/app/AppOpsManager.java 
```

这里我们重点关注核心服务 AppOpsService  的实现，下面分析下一些核心的方法接口！

# 1 AppOpsService.checkOperation - 检查操作权限

checkOperation 用于检查指定 package 是否有权限执行指定操作！

```java
    @Override
    public int checkOperation(int code, int uid, String packageName) {
        //【×1.1】校验 uid 是否有 UPDATE_APP_OPS_STATS 权限；
        verifyIncomingUid(uid);
        //【×1.2】校验 op code 是否正确！
        verifyIncomingOp(code);
        //【×1.3】处理包名！
        String resolvedPackageName = resolvePackageName(uid, packageName);
        if (resolvedPackageName == null) {
            return AppOpsManager.MODE_IGNORED; // 如果 package 为 null，返回 MODE_IGNORED！
        }
        synchronized (this) {
            //【×1.4】判断该 operation 是否收到用户限制，如果有，返回 MODE_IGNORED！
            if (isOpRestrictedLocked(uid, code, resolvedPackageName)) {
                return AppOpsManager.MODE_IGNORED;
            }
            //【2】获得决定 code 指定 op 的模式的 op，调用 AppOpsManager.opToSwitch 方法！
            code = AppOpsManager.opToSwitch(code);

            //【×1.5】判断该 uid 是否允许执行该操作，如果不允许，返回已有的模式状态；
            UidState uidState = getUidStateLocked(uid, false);
            if (uidState != null && uidState.opModes != null) {
                final int uidMode = uidState.opModes.get(code);
                if (uidMode != AppOpsManager.MODE_ALLOWED) {
                    return uidMode;
                }
            }
            //【×1.6】如果该 uid 允许执行该操作，那就进一步判断该 package 是否允许执行该操作，
            // 如果没有定义，返回默认状态；否则返回已有的模式状态；
            Op op = getOpLocked(code, uid, resolvedPackageName, false);
            if (op == null) {
                return AppOpsManager.opToDefaultMode(code);
            }
            return op.mode;
        }
    }
```
其实我们知道，该方法返回值有如下几种：MODE_ALLOWED，MODE_IGNORED，MODE_ERRORED 和 MODE_DEFAULT！

## 1.1 verifyIncomingUid

校验 uid 是否具有 android.Manifest.permission.UPDATE_APP_OPS_STATS 权限！

```java
    private void verifyIncomingUid(int uid) {
        //【1】如果是 uid 自身调用，不需要检验！
        if (uid == Binder.getCallingUid()) {
            return;
        }
        //【2】如果是当前进程自己调用的，najiu
        if (Binder.getCallingPid() == Process.myPid()) {
            return;
        }
        mContext.enforcePermission(android.Manifest.permission.UPDATE_APP_OPS_STATS,
                Binder.getCallingPid(), Binder.getCallingUid(), null);
    }
```
如果 uid 没有 android.Manifest.permission.UPDATE_APP_OPS_STATS 权限，抛出异常！

## 1.2 verifyIncomingOp

校验 op 是否正确！
```java
    private void verifyIncomingOp(int op) {
        if (op >= 0 && op < AppOpsManager._NUM_OP) {
            return;
        }
        throw new IllegalArgumentException("Bad operation #" + op);
    }
```
如果 op 不在 [0， AppOpsManager._NUM_OP) 范围内，抛出异常；

## 1.3 resolvePackageName

处理一些特殊 uid 所属包名！

```java
    private static String resolvePackageName(int uid, String packageName)  {
        if (uid == 0) {
            return "root";
        } else if (uid == Process.SHELL_UID) {
            return "com.android.shell";
        } else if (uid == Process.SYSTEM_UID && packageName == null) {
            return "android";
        }
        return packageName;
    }
```
不多说！

## 1.4 isOpRestrictedLocked

判断该操作是否被用户限制！
```java
    private boolean isOpRestrictedLocked(int uid, int code, String packageName) {
        //【1】获得该 uid 所属的 userId！
        int userHandle = UserHandle.getUserId(uid);
        final int restrictionSetCount = mOpUserRestrictions.size();

        for (int i = 0; i < restrictionSetCount; i++) {
            //【2】遍历每一个 ClientRestrictionState，检查该 op 是否在 uid 所在的用户下被限制，同时也检查
            // 属于该 uid 的 package 是否在白名单中，如果在白名单中，那就不受限制！
            ClientRestrictionState restrictionState = mOpUserRestrictions.valueAt(i);
            //【×1.4.1】校验该 package 的 op 在指定的 userid 下是否收到限制，如果有返回 true！
            if (restrictionState.hasRestriction(code, packageName, userHandle)) {
                //【3】如果确实在该 userId 下受到限制，那就在判断下是否允许 system/system ui 绕过限制！
                if (AppOpsManager.opAllowSystemBypassRestriction(code)) {
                    synchronized (this) {
                        //【*1.4.2】获得该 package 的所有 op 操作
                        Ops ops = getOpsRawLocked(uid, packageName, true);
                        if ((ops != null) && ops.isPrivileged) {
                            //【3.1】如果 ops.isPrivileged 为 true，说明该 package 是特权应用，要绕过用户限制！
                            return false;
                        }
                    }
                }
                //【3.2】受到限制但是不允许绕过！
                return true;
            }
        }
        return false; // 不受限制返回 false；
    }
```
AppOpsService 用一个 ArrayMap 保存所有的用户限制状态信息！
```java
    private final ArrayMap<IBinder, ClientRestrictionState> mOpUserRestrictions = new ArrayMap<>();
```
### 1.4.1 ClientRestrictionState.hasRestriction

该方法用于判断指定 package 的指定 op 是否在 userId 下收到限制！
```java
        public boolean hasRestriction(int restriction, String packageName, int userId) {
            //【1】如果 perUserRestrictions 为 null，不受限制！
            if (perUserRestrictions == null) {
                return false;
            }
            //【2】如果在 userId 下没有限制信息，不受限制！
            boolean[] restrictions = perUserRestrictions.get(userId);
            if (restrictions == null) {
                return false;
            }
            //【3】如果在 userId 下，restrictions[restriction] 为 false，不受限制！
            if (!restrictions[restriction]) {
                return false;
            }
            //【4】如果在 userId 下，restrictions[restriction] 为 true，说明在该用户下，该 op 是受限制的
            // 那就要看下 package 是否在白名单中，如果没有白名单，那么就是首先的！
            if (perUserExcludedPackages == null) {
                return true;
            }
            String[] perUserExclusions = perUserExcludedPackages.get(userId);
            if (perUserExclusions == null) {
                return true;
            }
            //【5】如果该 package 是在白名单中，那么就是不受限的，返回 false；
            return !ArrayUtils.contains(perUserExclusions, packageName);
        }
```
方法很简单，不多说了！

### 1.4.2 getOpsRawLocked

```java
    private Ops getOpsRawLocked(int uid, String packageName, boolean edit) {
        //【*1.5】获得 uid 的 UidState 对象！
        UidState uidState = getUidStateLocked(uid, edit);
        if (uidState == null) {
            return null;
        }

        if (uidState.pkgOps == null) {
            if (!edit) {
                return null;
            }
            uidState.pkgOps = new ArrayMap<>();
        }
        //【1】获得 packageName 指定的应用的 op 限制！
        Ops ops = uidState.pkgOps.get(packageName);
        if (ops == null) {
            if (!edit) {
                return null;
            }
            boolean isPrivileged = false;
            //【2】下面会判断 package 和 uid 的对应关系是否有效！
            if (uid != 0) {
                final long ident = Binder.clearCallingIdentity();
                try {
                    int pkgUid = -1;
                    try {
                        //【2.1】获得 packageName 对应的应用程序信息 ApplicationInfo！
                        ApplicationInfo appInfo = ActivityThread.getPackageManager()
                                .getApplicationInfo(packageName,
                                        PackageManager.MATCH_DEBUG_TRIAGED_MISSING,
                                        UserHandle.getUserId(uid));
                        if (appInfo != null) {
                            //【2.2】获得 package 的 uid，判断其是否是特权应用；
                            pkgUid = appInfo.uid;
                            isPrivileged = (appInfo.privateFlags
                                    & ApplicationInfo.PRIVATE_FLAG_PRIVILEGED) != 0;
                        } else {
                            //【2.3】针对一些特殊没有安装信息的 package，这里做一个处理！
                            if ("media".equals(packageName)) {
                                pkgUid = Process.MEDIA_UID;
                                isPrivileged = false;
                            } else if ("audioserver".equals(packageName)) {
                                pkgUid = Process.AUDIOSERVER_UID;
                                isPrivileged = false;
                            } else if ("cameraserver".equals(packageName)) {
                                pkgUid = Process.CAMERASERVER_UID;
                                isPrivileged = false;
                            }
                        }
                    } catch (RemoteException e) {
                        Slog.w(TAG, "Could not contact PackageManager", e);
                    }
                    //【2.4】判读 package 和 uid 的对应关系是否有效，如果 package 现在的 uid 不是 AppOps
                    // 中记录的 uid，那么输出异常，并返回；
                    if (pkgUid != uid) {
                        RuntimeException ex = new RuntimeException("here");
                        ex.fillInStackTrace();
                        Slog.w(TAG, "Bad call: specified package " + packageName
                                + " under uid " + uid + " but it is really " + pkgUid, ex);
                        return null;
                    }
                } finally {
                    Binder.restoreCallingIdentity(ident);
                }
            }
            //【3】对于 edit 为 true 的情况，会做初始化操作！！
            ops = new Ops(packageName, uidState, isPrivileged);
            uidState.pkgOps.put(packageName, ops);
        }
        return ops;
    }
```
不多说了！

## 1.5 getUidStateLocked

获得 uid 的 op 管理信息：

```java
    private UidState getUidStateLocked(int uid, boolean edit) {
        UidState uidState = mUidStates.get(uid);
        if (uidState == null) {
            if (!edit) {
                return null;
            }
            uidState = new UidState(uid);
            //【1】从 mUidStates 获取！
            mUidStates.put(uid, uidState);
        }
        return uidState;
    }
```
不多说！

## 1.6 getOpLocked

getOpLocked 用于获得指定 package 的特定 op 对应的 Op 对象，该 Op 对象封装了操作的模式等相关信息，参数 boolean edit 这里传入的是 true！

```java
    private Op getOpLocked(int code, int uid, String packageName, boolean edit) {
       //【1】获得该 package 的所有的 op 信息，封装在 Ops 对象中！
        Ops ops = getOpsRawLocked(uid, packageName, edit);
        if (ops == null) {
            return null;
        }
        //【2】返回指定 op 对应的信息！
        return getOpLocked(ops, code, edit);
    }
```
调用了三参数函数 getOpLocked！
```java
    private Op getOpLocked(Ops ops, int code, boolean edit) {
        //【1】获得 code 对应的 Op 对象！
        Op op = ops.get(code);
        if (op == null) {
            if (!edit) {
                return null;
            }
            // 如果没有，由于 edit 为 true，这里会默认初始化创建一个 Op 对象！
            op = new Op(ops.uidState.uid, ops.packageName, code);
            ops.put(code, op);
        }
        if (edit) { // 同时更新本地持久化文件！
            scheduleWriteLocked();
        }
        return op; // 返回！
    }
```

# 2 AppOpsService.noteOperation - 检查操作权限

检查指定的 package 是否允许执行 op 对应的操作，同时会更新 op 相关的信息！

noteOperation 用于短期权限的检查。

```java
    @Override
    public int noteOperation(int code, int uid, String packageName) {
        verifyIncomingUid(uid);
        verifyIncomingOp(code);
        String resolvedPackageName = resolvePackageName(uid, packageName);
        if (resolvedPackageName == null) {
            return AppOpsManager.MODE_IGNORED;
        }
        //【*2.1】调用了 5 参函数！
        return noteOperationUnchecked(code, uid, resolvedPackageName, 0, null);
    }
```
继续分析：

## 2.1 noteOperationUnchecked

我们去看看 5 参函数：

```java
    private int noteOperationUnchecked(int code, int uid, String packageName,
            int proxyUid, String proxyPackageName) {
        synchronized (this) {
            //【*1.4.2】获得该 package 的所有 Op 信息！
            Ops ops = getOpsRawLocked(uid, packageName, true);
            if (ops == null) {
                if (DEBUG) Log.d(TAG, "noteOperation: no op for code " + code + " uid " + uid
                        + " package " + packageName);
                return AppOpsManager.MODE_ERRORED;
            }
            //【*1.5】获得 code 指定的 Op 信息！
            Op op = getOpLocked(ops, code, true);

            //【*1.4】如果该 op 在 uid 所在的用户下受限，返回 MODE_IGNORED ！
            if (isOpRestrictedLocked(uid, code, packageName)) {
                return AppOpsManager.MODE_IGNORED;
            }
            if (op.duration == -1) {
                Slog.w(TAG, "Noting op not finished: uid " + uid + " pkg " + packageName
                        + " code " + code + " time=" + op.time + " duration=" + op.duration);
            }

            op.duration = 0;

            //【2】获得决定 code 指定 op 模式的 switchCode，其实是另外一个 op！
            final int switchCode = AppOpsManager.opToSwitch(code);
            //【3】获得该 package 所属 uid 的 op 限制！
            UidState uidState = ops.uidState;

            if (uidState.opModes != null && uidState.opModes.indexOfKey(switchCode) >= 0) {
                //【3.1】如果该 uid 有 switchCode，那么 uid 的 switchCode 模式优先级更高！
                // 如果 uid 不允许执行该操作，返回的是该 uid 的 switchCode 模式！
                final int uidMode = uidState.opModes.get(switchCode);
                if (uidMode != AppOpsManager.MODE_ALLOWED) {
                    if (DEBUG) Log.d(TAG, "noteOperation: reject #" + op.mode + " for code "
                            + switchCode + " (" + code + ") uid " + uid + " package "
                            + packageName);

                    //【3.1.1】uid 不被允许的执行该 switchCode 在指定的操作，
                    // 那么更新 code 对应 op 的 rejectTime 值！
                    op.rejectTime = System.currentTimeMillis();
                    return uidMode;
                }
            } else {
                //【3.2】如果该 uid 没有该 switchCode，那就读取 package 的 switchCode 模式！
                //【×1.6】通过 getOpLocked 获得指定的 op 信息！
                final Op switchOp = switchCode != code ? getOpLocked(ops, switchCode, true) : op;
                if (switchOp.mode != AppOpsManager.MODE_ALLOWED) {
                    if (DEBUG) Log.d(TAG, "noteOperation: reject #" + op.mode + " for code "
                            + switchCode + " (" + code + ") uid " + uid + " package "
                            + packageName);
                    //【3.2.1】package 不被允许的执行该 switchCode 在指定的操作，
                    // 那么更新 code 对应 op 的 rejectTime 值！
                    return switchOp.mode;
                }
            }

            if (DEBUG) Log.d(TAG, "noteOperation: allowing code " + code + " uid " + uid
                    + " package " + packageName);

            //【4】更新 code 对应的 op 的相关信息！
            op.time = System.currentTimeMillis();
            op.rejectTime = 0;
            op.proxyUid = proxyUid;
            op.proxyPackageName = proxyPackageName;
            return AppOpsManager.MODE_ALLOWED;
        }
    }
```
方法很简单，不多说了！

# 3 AppOpsService.setUidMode - 设置 uid 的 op 模式

设置指定 uid 的相应 op 的模式，如果 uid 的 op 模式发生后，那么属于该 uid 的所有 package 也会受到影响！！

```java
    @Override
    public void setUidMode(int code, int uid, int mode) {
        if (Binder.getCallingPid() != Process.myPid()) {
            mContext.enforcePermission(android.Manifest.permission.UPDATE_APP_OPS_STATS,
                    Binder.getCallingPid(), Binder.getCallingUid(), null);
        }
        //【×1.2】校验 op 是否有效！
        verifyIncomingOp(code);
        //【1】获得决定 code 指定 op 模式的 switchCode，其实是另外一个 op！
        code = AppOpsManager.opToSwitch(code);

        synchronized (this) {
            //【2】首先，获得该 op 的默认模式 defaultMode！
            final int defaultMode = AppOpsManager.opToDefaultMode(code);

            //【×1.5】获得该 uid 对应的 UidState 对象！
            UidState uidState = getUidStateLocked(uid, false);
            if (uidState == null) {
                //【2.1】如果该 uid 没有相关的记录，且本次设置的模式是默认模式，不处理。直接返回！
                if (mode == defaultMode) {
                    return;
                }
                // 否则，创建 UidState 对象，将 code 和 mode 的关系添加进去！
                uidState = new UidState(uid);
                uidState.opModes = new SparseIntArray();
                uidState.opModes.put(code, mode);
                mUidStates.put(uid, uidState);
                // 更新本地文件；
                scheduleWriteLocked();
                
            } else if (uidState.opModes == null) {
                //【2.2】如果该 uid 没有 op 相关的记录，且本次设置的模式是默认模式，不处理！
                // 否则初始化 uidState.opModes，将 code 和 mode 的关系添加进去！
                if (mode != defaultMode) {
                    uidState.opModes = new SparseIntArray();
                    uidState.opModes.put(code, mode);
                    // 更新本地文件；
                    scheduleWriteLocked();
                }
            } else {
                //【2.3】如果该 uid 下的 op 已有模式和本次设置的一样，不处理，直接返回！
                if (uidState.opModes.get(code) == mode) {
                    return;
                }
                // 如果该 uid 下的 op 已有模式不是默认模式，但是要设置为默认模式，直接从 uidState.opModes 中删除 op！
                // 否则，更新 mode 的值！
                if (mode == defaultMode) {
                    uidState.opModes.delete(code);
                    if (uidState.opModes.size() <= 0) {
                        uidState.opModes = null;
                    }
                } else {
                    uidState.opModes.put(code, mode);
                }
                scheduleWriteLocked();  // 更新本地文件；
            }
        }
        //【3】获得该 uid 下的所有 package！
        String[] uidPackageNames = getPackagesForUid(uid);
        ArrayMap<Callback, ArraySet<String>> callbackSpecs = null;

        synchronized (this) {
            //【3.1】收集所有监听该 uid 下该 op 模式变化的 Callback，用于触发回调！！
            ArrayList<Callback> callbacks = mOpModeWatchers.get(code);
            if (callbacks != null) {
                final int callbackCount = callbacks.size();
                for (int i = 0; i < callbackCount; i++) {
                    Callback callback = callbacks.get(i);
                    ArraySet<String> changedPackages = new ArraySet<>();
                    
                    //【3.1.1】将该 uid 下的所有 package 保存到 changedPackages 中，然后将
                    // 每个 Callback 和 changedPackages 的映射关系保存到 callbackSpecs 中！
                    Collections.addAll(changedPackages, uidPackageNames);
                    callbackSpecs = new ArrayMap<>();
                    callbackSpecs.put(callback, changedPackages);
                }
            }
            //【3.2】收集所有监听该 uid 下 package 的 op 模式变化的 Callback！
            for (String uidPackageName : uidPackageNames) {
                callbacks = mPackageModeWatchers.get(uidPackageName);
                if (callbacks != null) {
                    if (callbackSpecs == null) {
                        callbackSpecs = new ArrayMap<>();
                    }
                    final int callbackCount = callbacks.size();
                    for (int i = 0; i < callbackCount; i++) {
                        Callback callback = callbacks.get(i);
                        //【3.2.1】获得之前已经添加的 packageName！！
                        ArraySet<String> changedPackages = callbackSpecs.get(callback);
                        if (changedPackages == null) {
                            changedPackages = new ArraySet<>();
                            callbackSpecs.put(callback, changedPackages);
                        }
                        //【3.2.2】将 package 添加到对应关系中！
                        changedPackages.add(uidPackageName);
                    }
                }
            }
        }

        if (callbackSpecs == null) {
            return;
        }
        //【4】处理回调，通知该 uid 下的 op 发生了变化，如果 uid 下有 pkg，也会将 pkg 传递过去！！
        final long identity = Binder.clearCallingIdentity();
        try {
            for (int i = 0; i < callbackSpecs.size(); i++) {
                Callback callback = callbackSpecs.keyAt(i);
                ArraySet<String> reportedPackageNames = callbackSpecs.valueAt(i);
                try {
                    if (reportedPackageNames == null) {
                        callback.mCallback.opChanged(code, uid, null);
                    } else {
                        final int reportedPackageCount = reportedPackageNames.size();
                        for (int j = 0; j < reportedPackageCount; j++) {
                            String reportedPackageName = reportedPackageNames.valueAt(j);
                            //【4.1】触发回调！
                            callback.mCallback.opChanged(code, uid, reportedPackageName);
                        }
                    }
                } catch (RemoteException e) {
                    Log.w(TAG, "Error dispatching op op change", e);
                }
            }
        } finally {
            Binder.restoreCallingIdentity(identity);
        }
    }
```
设置完 uid 的 op 后，会触发那些监听 op 变化的回调！

# 4 AppOpsService.setMode - 设置 package 的 op 模式

设置指定 package 的指定 op 的模式值，参数 int uid 是该 package 所属的 uid！
```java
    @Override
    public void setMode(int code, int uid, String packageName, int mode) {
        //【1】校验权限，因为 setMode 方法只能是系统调用，系统是默认有 UPDATE_APP_OPS_STATS 权限的！
        if (Binder.getCallingPid() != Process.myPid()) {
            mContext.enforcePermission(android.Manifest.permission.UPDATE_APP_OPS_STATS,
                    Binder.getCallingPid(), Binder.getCallingUid(), null);
        }
        verifyIncomingOp(code);
        ArrayList<Callback> repCbs = null;
        //【2】获得决定 code 指定的 op 的模式的 op，调用 AppOpsManager.opToSwitch 方法！
        code = AppOpsManager.opToSwitch(code);
        
        synchronized (this) {
            //【3】获得 uid 对应的 UidState 对象！
            UidState uidState = getUidStateLocked(uid, false);
            //【×1.5】获得 code（switch） 对应的 Op 对象！
            Op op = getOpLocked(code, uid, packageName, true);
            if (op != null) {
                //【5】如果 op 的现有模式和本次要设置的模式不同，更新模式值！
                if (op.mode != mode) {
                    op.mode = mode;
                    //【5.1】更新了模式值后，收集所有监听该 op 模式变化的观察者！
                    ArrayList<Callback> cbs = mOpModeWatchers.get(code);
                    if (cbs != null) {
                        if (repCbs == null) {
                            repCbs = new ArrayList<Callback>();
                        }
                        repCbs.addAll(cbs);
                    }
                    //【5.2】更新了模式值后，收集所有监听该 packageName 操作模式变化的观察者！
                    cbs = mPackageModeWatchers.get(packageName);
                    if (cbs != null) {
                        if (repCbs == null) {
                            repCbs = new ArrayList<Callback>();
                        }
                        repCbs.addAll(cbs);
                    }
                    //【×4.1】如果本次设置是恢复默认模式，那么就移除该 op；
                    if (mode == AppOpsManager.opToDefaultMode(op.op)) {
                        pruneOp(op, uid, packageName);
                    }
                    //【×4.2】更新本地文件；
                    scheduleFastWriteLocked();
                }
            }
        }
        //【6】当 repCbs 不为 null，通知所有监听者！
        if (repCbs != null) {
            // 将远程调用者的 uid/pid 转为系统进程的 uid/pid 防止权限相关问题！
            final long identity = Binder.clearCallingIdentity();
            try {
                for (int i = 0; i < repCbs.size(); i++) {
                    try {
                        //【6.1】触发 callback 的 opChanged 回调！
                        repCbs.get(i).mCallback.opChanged(code, uid, packageName);
                    } catch (RemoteException e) {
                    }
                }
            } finally {
                Binder.restoreCallingIdentity(identity);
            }
        }
    }
```
方法流程很简单，不多说了！

## 4.1 pruneOp

移除指定的 Op！
```java
    private void pruneOp(Op op, int uid, String packageName) {
        if (op.time == 0 && op.rejectTime == 0) {
            //【1】返回该 uid 下该 package 的所有 Op 信息！
            Ops ops = getOpsRawLocked(uid, packageName, false);
            if (ops != null) {
                //【2】从中移除该 op！
                ops.remove(op.op);
                //【3】如果该 package 已经没有任何 Op，那就从所属于的 UidState 中移除相关记录！
                if (ops.size() <= 0) {
                    UidState uidState = ops.uidState;
                    ArrayMap<String, Ops> pkgOps = uidState.pkgOps;
                    if (pkgOps != null) {
                        pkgOps.remove(ops.packageName);
                        if (pkgOps.isEmpty()) {
                            uidState.pkgOps = null;
                        }
                        if (uidState.isDefault()) {
                            mUidStates.remove(uid);
                        }
                    }
                }
            }
        }
    }
```
整个过程很简单，不多说了！


## 4.2 scheduleFastWriteLocked
```java
    private void scheduleFastWriteLocked() {
        if (!mFastWriteScheduled) {
            mWriteScheduled = true;
            mFastWriteScheduled = true;
            mHandler.removeCallbacks(mWriteRunner);
            mHandler.postDelayed(mWriteRunner, 10*1000);
        }
    }
```
更新本地持久化文件，执行一个 Runnable 对象 mWriteRunner，在启动篇的时候，我们看过了，这里就不再多说了！

# 5 AppOpsService.startWatchingMode - 监听 op 变化

startWatchingMode 用于启动一个监听，他能够监听指定 package 的指定 op 操作的模式变化，当发生会变化后，会回调 callback！
```java
    @Override
    public void startWatchingMode(int op, String packageName, IAppOpsCallback callback) {
        if (callback == null) {
            return;
        }
        synchronized (this) {
            //【1】获得决定 op 模式的 op 操作！
            op = (op != AppOpsManager.OP_NONE) ? AppOpsManager.opToSwitch(op) : op;
            //【3.1】将 callback 转为客户端代理，封装为一个 Callback 对象，添加到 mModeWatchers 中！
            Callback cb = mModeWatchers.get(callback.asBinder());
            if (cb == null) {
                // 创建了一个 Callback 对象，用于监听远程 Binder 对象的死亡布告对象
                // 这样当 IAppOpsCallback.Stub 死亡后，可以停止监听！
                cb = new Callback(callback);
                // 将 IAppOpsCallback.Stub 和 Callback 的映射保存到 mModeWatchers 中！
                mModeWatchers.put(callback.asBinder(), cb);
            }
            //【3.2】将要监听的 op 操作和对应的监听回调 Callback 的映射关系保存到 mOpModeWatchers 中！
            if (op != AppOpsManager.OP_NONE) {
                ArrayList<Callback> cbs = mOpModeWatchers.get(op);
                if (cbs == null) {
                    cbs = new ArrayList<Callback>();
                    mOpModeWatchers.put(op, cbs);
                }
                cbs.add(cb);
            }
            //【3.3】将要监听的 package 和对应的监听回调 Callback 的映射关系保存到 mPackageModeWatchers 中！
            if (packageName != null) {
                ArrayList<Callback> cbs = mPackageModeWatchers.get(packageName);
                if (cbs == null) {
                    cbs = new ArrayList<Callback>();
                    mPackageModeWatchers.put(packageName, cbs);
                }
                cbs.add(cb);
            }
        }
    }
```

AppOpsService 内部有多个集合：
```java
    //【1】用与 IBinder 作和对应的回调 Callback 的映射关系！
    final ArrayMap<IBinder, Callback> mModeWatchers
            = new ArrayMap<IBinder, Callback>();
```
用于保存远程回调 IAppOpsCallback.Stub 和其对应的死亡仆告对象 Callback 的映射关系，当 IAppOpsCallback.Stub 死亡后，Callback.binderDied 会被触发！！

```java
    //【2】用与保存 op 操作和对应的回调 Callback 的映射关系！
    final SparseArray<ArrayList<Callback>> mOpModeWatchers
            = new SparseArray<ArrayList<Callback>>();
    //【3】用与保存 package 和对应的回调 Callback 的映射关系！
    final ArrayMap<String, ArrayList<Callback>> mPackageModeWatchers
            = new ArrayMap<String, ArrayList<Callback>>();
```
而 mOpModeWatchers 和 mPackageModeWatchers 则是从不同的角度来建立监听关系：mOpModeWatchers 是从具体 op 的角度，而 mPackageModeWatchers 则是从 package 的角度！

可以看到 Callback 的作用是作为远程回调的死亡仆告对象，用于停止监听。

## 5.1 new Callback

我们来看下 Callback 对象的创建，Callback 是一个回调对象，用于处理 op 变化后的操作；同时又是一个 DeathRecipient 对象，用来监听远程 IAppOpsCallback 桩对象是否死亡！
```java
    public final class Callback implements DeathRecipient {
        final IAppOpsCallback mCallback;

        public Callback(IAppOpsCallback callback) {
            mCallback = callback;
            try {
                //【1】将自身注册为一个死亡仆告对象！
                mCallback.asBinder().linkToDeath(this, 0);
            } catch (RemoteException e) {
            }
        }

        public void unlinkToDeath() { //【2】解除注册！
            mCallback.asBinder().unlinkToDeath(this, 0);
        }

        @Override
        public void binderDied() { //【1.6】当远程桩对象死亡后，停止监听！
            stopWatchingMode(mCallback);
        }
    }
```
当远程的 IAppOpsCallback.Stub 死亡后，AppOpsService.stopWatchingMode 会被执行！

# 6 AppOpsService.stopWatchingMode - 停止 op 监听

startWatchingMode 用于停止一个监听！
```java
    @Override
    public void stopWatchingMode(IAppOpsCallback callback) {
        if (callback == null) {
            return;
        }
        synchronized (this) {
            //【1】从 mModeWatchers 中移除 Callback！
            Callback cb = mModeWatchers.remove(callback.asBinder());
            if (cb != null) {
                // 解除死亡仆告对象注册！
                cb.unlinkToDeath();
                //【2】从 mOpModeWatchers 移除该 Callback！
                for (int i=mOpModeWatchers.size()-1; i>=0; i--) {
                    ArrayList<Callback> cbs = mOpModeWatchers.valueAt(i);
                    cbs.remove(cb);
                    if (cbs.size() <= 0) {
                        mOpModeWatchers.removeAt(i);
                    }
                }
                //【3】从 mPackageModeWatchers 移除该 Callback！
                for (int i=mPackageModeWatchers.size()-1; i>=0; i--) {
                    ArrayList<Callback> cbs = mPackageModeWatchers.valueAt(i);
                    cbs.remove(cb);
                    if (cbs.size() <= 0) {
                        mPackageModeWatchers.removeAt(i);
                    }
                }
            }
        }
    }
```
方法很简单，就不多说了！

# 7 AppOpsService.startOperation - 开始监视操作

startOperation 用于监控一些长时间工作的操作，比如 Gps，像 Vibrator 等等，可以使用 startOperation 开始监视权限，使用 finishOperation 结束监视，要监视 op 前提是被监视者有权限执行 op 操作！！

```java
    @Override
    public int startOperation(IBinder token, int code, int uid, String packageName) {
        verifyIncomingUid(uid);
        verifyIncomingOp(code);
        //【1】处理包名！
        String resolvedPackageName = resolvePackageName(uid, packageName);
        if (resolvedPackageName == null) {
            return  AppOpsManager.MODE_IGNORED;
        }
        ClientState client = (ClientState)token;
        synchronized (this) {
            //【*1.4.1】获得该 package 的所有 Op 信息！
            Ops ops = getOpsRawLocked(uid, resolvedPackageName, true);
            if (ops == null) {
                if (DEBUG) Log.d(TAG, "startOperation: no op for code " + code + " uid " + uid
                        + " package " + resolvedPackageName);
                return AppOpsManager.MODE_ERRORED;
            }
            //【*1.5】获得 code 指定的 Op 信息！
            Op op = getOpLocked(ops, code, true);

            //【*1.4】如果该 op 在 uid 所在的用户下受限，返回 MODE_IGNORED ！
            if (isOpRestrictedLocked(uid, code, resolvedPackageName)) {
                return AppOpsManager.MODE_IGNORED;
            }
            //【2】获得决定 code 指定 op 模式的 switchCode，其实是另外一个 op！
            final int switchCode = AppOpsManager.opToSwitch(code);
            UidState uidState = ops.uidState;
            if (uidState.opModes != null) {
                //【3.1】如果该 uid 有 switchCode 对应的操作，那么 uid 的 switchCode 模式优先级更高！
                // 所以，最后返回的是该 uid 的 switchCode 模式！
                final int uidMode = uidState.opModes.get(switchCode);
                if (uidMode != AppOpsManager.MODE_ALLOWED) {
                    if (DEBUG) Log.d(TAG, "noteOperation: reject #" + op.mode + " for code "
                            + switchCode + " (" + code + ") uid " + uid + " package "
                            + resolvedPackageName);
                    //【3.1.1】uid 不被允许的执行该 switchCode 指定的操作，
                    // 那么更新 code 对应 op 的 rejectTime 值！
                    op.rejectTime = System.currentTimeMillis();
                    return uidMode;
                }
            }
            //【3.2】如果该 uid 没有该 switchCode 对应的操作，那就读取 package 的 switchCode 操作模式！
            // 所以，最后返回的是该 package 的 switchCode 模式！
            final Op switchOp = switchCode != code ? getOpLocked(ops, switchCode, true) : op;
            if (switchOp.mode != AppOpsManager.MODE_ALLOWED) {
                if (DEBUG) Log.d(TAG, "startOperation: reject #" + op.mode + " for code "
                        + switchCode + " (" + code + ") uid " + uid + " package "
                        + resolvedPackageName);
                //【3.2.1】package 不被允许的执行该 switchCode 在指定的操作，
                // 那么更新 code 对应 op 的 rejectTime 值！
                op.rejectTime = System.currentTimeMillis();
                return switchOp.mode;
            }
            if (DEBUG) Log.d(TAG, "startOperation: allowing code " + code + " uid " + uid
                    + " package " + resolvedPackageName);
            
            // 进入到这里，表示该操作是被允许的，这里会更新 op 的数据，然后返回 MODE_ALLOWED！
            if (op.nesting == 0) {
                op.time = System.currentTimeMillis(); // 更新允许时间
                op.rejectTime = 0; // 设置拒绝时间为 0；
                op.duration = -1;
            }
            op.nesting++; // start 次数加 1；
            if (client.mStartedOps != null) {
                client.mStartedOps.add(op); // 将 start 的 Op 添加到 ClientState.mStartedOps 中监控！
            }
            return AppOpsManager.MODE_ALLOWED;
        }
    }
```
startOperation 的第一个参数是一个 IBinder token，这个 token 通过如下方式获得：

```java
AppOpsManager.getToken(mAppOpsService)
```
在 AppOpsManager 中我们有分析过这个方法，其返回的是一个 ClientState 对象！

## 7.1 AppOpsManager.getToken

我们来看看 getToken 方法，传入的是 mService。mService 是 AppOpsManager 的成员变量：
```java
IAppOpsService mService;
```
其实 mService 就是 AppOpsService 的 Proxy 对象！

```hava
    /** @hide */
    public static IBinder getToken(IAppOpsService service) {
        synchronized (AppOpsManager.class) {
            if (sToken != null) {
                return sToken;
            }
            try {
                //【1】这里创建了一个 new Binder 对象，同时调用 AppOpsService.getToken 方法！
                // 该方法会返回一个 Binder 对象，保存到 AppOpsManager.sToken 变量中;
                sToken = service.getToken(new Binder());
            } catch (RemoteException e) {
                throw e.rethrowFromSystemServer();
            }
            return sToken;
        }
    }
```
这里会创建一个 Binder 对象，表示当前进程实体！

sToken 也是一个 Binder 对象：
```java
    static IBinder sToken;
```
用于保存 AppOpsService 返回的 ClientState 对象！

## 7.2 AppOpsService.getToken

通过 Binder 通信，AppOpsManager 创建的 Binder 对象，传递到了 AppOpsService.getToken 方法中！
```java
    @Override
    public IBinder getToken(IBinder clientToken) {
        synchronized (this) {
            ClientState cs = mClients.get(clientToken);
            if (cs == null) {
                //【×7.2.1】创建一个 ClientState 对象，保存到 mClients 中！
                cs = new ClientState(clientToken);
                mClients.put(clientToken, cs);
            }
            return cs;
        }
    }
```
这里的 ClientToken，就是前面 new Binder 创建的实体!

AppOpsService 内部有一个 mClients 变量，用于保存调用方进程的 Binder 对象和对应的 ClientState 的映射关系！
```java
    final ArrayMap<IBinder, ClientState> mClients = new ArrayMap<IBinder, ClientState>();
```
这个 ClientState 会记录调用方进程的 op 状态，同时其也实现了 DeathRecipient 接口，能够监听调用方进程的 Binder 对象是否死亡！

### 7.2.1 new ClientState - 被监控的实体信息

我们来看下 ClientState 的结构！

```java
    public final class ClientState extends Binder implements DeathRecipient {
        final IBinder mAppToken; // 调用方进程 token！
        final int mPid; // 调用方进程的 pid
        final ArrayList<Op> mStartedOps; // 用于保存调用方被 start 的所有 Op 操作！

        public ClientState(IBinder appToken) {
            mAppToken = appToken; // 保存调用方的 Binder 对象！
            mPid = Binder.getCallingPid(); // 获得调用方进程的 pid！
            if (appToken instanceof Binder) {
                // 如果调用方传来的 appToken 是 Binder 的实例，那说明调用方就是系统进程内部的某个 client！
                // 这种中情况，无需监控被 start 的 op 操作！
                mStartedOps = null;
            } else {
                // 如果调用方处于非系统进程，那么我们需要监控其 start 的 op 操作！
                mStartedOps = new ArrayList<Op>();
                try {
                    mAppToken.linkToDeath(this, 0);
                } catch (RemoteException e) {
                }
            }
        }

        @Override
        public void binderDied() {
            // 当远程调用进程死亡后，会调用 ClientState.binderDied 方法，该方法会 finish 掉远程进程
            // 持有的所有被 start 的 op 操作！
            synchronized (AppOpsService.this) {
                for (int i=mStartedOps.size()-1; i>=0; i--) {
                    //【*8.1】调用了 finishOperation 方法！
                    finishOperationLocked(mStartedOps.get(i));
                }
                mClients.remove(mAppToken);
            }
        }
    }
```
对于 ClientState 的分析就到这里！

# 8 AppOpsService.finishOperation - 结束监视操作

finishOperation

第一个参数 IBinder token，必须是 ClientState 对象！也就是说，必须在调用了 startOperation 后，finishOperation 才有意义！

```java
    @Override
    public void finishOperation(IBinder token, int code, int uid, String packageName) {
        verifyIncomingUid(uid);
        verifyIncomingOp(code);
        String resolvedPackageName = resolvePackageName(uid, packageName);
        if (resolvedPackageName == null) {
            return;
        }
        if (!(token instanceof ClientState)) {
            return;
        }
        ClientState client = (ClientState) token;
        synchronized (this) {
            //【×1.5】获得 uid 下该 package 的 code 操作对应的 Op 实例！
            Op op = getOpLocked(code, uid, resolvedPackageName, true);
            if (op == null) {
                return;
            }
            //【2】从 ClientState.mStartedOps 中移除该 Op！！
            if (client.mStartedOps != null) {
                if (!client.mStartedOps.remove(op)) {
                    throw new IllegalStateException("Operation not started: uid" + op.uid
                            + " pkg=" + op.packageName + " op=" + op.op);
                }
            }
            //【×8.1】调用了 finishOperationLocked 方法，继续处理！
            finishOperationLocked(op);
        }
    }
```
整个方法很简单，不所说了！

## 8.1 finishOperationLocked

finishOperationLocked 方法中会对该 Op 的属性进行更新！
```java
    void finishOperationLocked(Op op) {
        if (op.nesting <= 1) {
            if (op.nesting == 1) {
                //【1】更新该操作的执行时长！
                op.duration = (int)(System.currentTimeMillis() - op.time);
                //【2】更新操作 op 的最新允许时间！
                op.time += op.duration;
            } else {
                Slog.w(TAG, "Finishing op nesting under-run: uid " + op.uid + " pkg "
                        + op.packageName + " code " + op.op + " time=" + op.time
                        + " duration=" + op.duration + " nesting=" + op.nesting);
            }
            // 将 op.nesting 置为 0，表示 Op 的 start 操作完全结束！
            op.nesting = 0;
        } else {
            op.nesting--; // start 次数减 1；
        }
    }
```

# 9 AppOpsService.setUserRestriction - 设置用户限制

setUserRestriction 主要是在 UserManagerService 方法中使用！

setUserRestriction 方法用于设置用户限制，他有 2 个重载函数，第一个 setUserRestriction 方法会设置所有 op 的用户限制！

参数 IBinder token 是一个 Binder 对象，用于表示远程调用者，UserManagerService 中直接传入的是一个 new Bindler 对象！
```java
    @Override
    public void setUserRestrictions(Bundle restrictions, IBinder token, int userHandle) {
        checkSystemUid("setUserRestrictions"); / 如果不是系统进程调用会抛出异常！
        Preconditions.checkNotNull(restrictions);
        Preconditions.checkNotNull(token);
        //【1】处理定义的每一个 operation，获得其所受的用户限制！
        for (int i = 0; i < AppOpsManager._NUM_OP; i++) {
            String restriction = AppOpsManager.opToRestriction(i);
            // 如果该 op 确实会受到用户限制，继续处理！
            if (restriction != null) {
                //【×9.1】调用了 setUserRestrictionNoCheck 方法，进一步设置用户限制！
                setUserRestrictionNoCheck(i, restrictions.getBoolean(restriction, false), token,
                        userHandle, null);
            }
        }
    }
```
第二个 setUserRestriction 方法，第一个方法会设置指定 op 是否受到用户限制，同时额外传入了一个 String[] exceptionPackages，表示白名单，在该白名单中的 package 可以不受用户限制：
```java
    @Override
    public void setUserRestriction(int code, boolean restricted, IBinder token, int userHandle,
            String[] exceptionPackages) {
        //【1】当不是当前进程调用，强制校验是否有 MANAGE_APP_OPS_RESTRICTIONS 权限！
        if (Binder.getCallingPid() != Process.myPid()) {
            mContext.enforcePermission(Manifest.permission.MANAGE_APP_OPS_RESTRICTIONS,
                    Binder.getCallingPid(), Binder.getCallingUid(), null);
        }
        //【2】如果是跨用户调用，要校验是否有 INTERACT_ACROSS_USERS_FULL 和 INTERACT_ACROSS_USERS 权限！
        if (userHandle != UserHandle.getCallingUserId()) {
            if (mContext.checkCallingOrSelfPermission(Manifest.permission
                    .INTERACT_ACROSS_USERS_FULL ) != PackageManager.PERMISSION_GRANTED
                && mContext.checkCallingOrSelfPermission(Manifest.permission
                    .INTERACT_ACROSS_USERS) != PackageManager.PERMISSION_GRANTED) {
                throw new SecurityException("Need INTERACT_ACROSS_USERS_FULL or"
                        + " INTERACT_ACROSS_USERS to interact cross user ");
            }
        }
        verifyIncomingOp(code);
        Preconditions.checkNotNull(token);
        //【×9.1】调用了 setUserRestrictionNoCheck 方法，进一步设置用户限制！
        setUserRestrictionNoCheck(code, restricted, token, userHandle, exceptionPackages);
    }
```

## 9.1 setUserRestrictionNoCheck

参数 boolean restricted 表示是否限制，为 true 的话，表示打开限制！

这里的 token 来自 UserManagerService 中：

```java
private static final IBinder mUserRestriconToken = new Binder();
```
本质上是一个 Binder 对象！！
```java
    private void setUserRestrictionNoCheck(int code, boolean restricted, IBinder token,
            int userHandle, String[] exceptionPackages) {
        boolean notifyChange = false;

        synchronized (AppOpsService.this) {
            //【×9.2】从 mOpUserRestrictions 获得 IBinder 对应的 ClientRestrictionState 对象！
            // 如果没有，就会创建一个新的实例，加入 mOpUserRestrictions！
            ClientRestrictionState restrictionState = mOpUserRestrictions.get(token);
            if (restrictionState == null) {
                try {
                    restrictionState = new ClientRestrictionState(token);
                } catch (RemoteException e) {
                    return;
                }
                mOpUserRestrictions.put(token, restrictionState);
            }
            //【×9.3】设置用户限制，如果有 op 的限制发生了变化，或者不受限制白名单发生了变化！
            if (restrictionState.setRestriction(code, restricted, exceptionPackages, userHandle)) {
                notifyChange = true;
            }
            //【×9.3.1】如果该 bindler 对应的 ClientRestrictionState 恢复了默认状态，那就移除！
            if (restrictionState.isDefault()) {
                mOpUserRestrictions.remove(token);
                restrictionState.destroy();
            }
        }
        //【×9.4】通知所有的监听者，该操作的用户限制状态发生了变化！
        if (notifyChange) {
            notifyWatchersOfChange(code);
        }
    }
```
AppOpsService 有一个 mOpUserRestrictions 的哈希表，用与保存所有的用于限制信息！
```java
private final ArrayMap<IBinder, ClientRestrictionState> mOpUserRestrictions = new ArrayMap<>();
```

## 9.2 new ClientRestrictionState

当第一次设置用户限制时，会创建 ClientRestrictionState 对象。保存用户限制状态！
```java
    private final class ClientRestrictionState implements DeathRecipient {
        private final IBinder token;
        SparseArray<boolean[]> perUserRestrictions; // 见下面；
        SparseArray<String[]> perUserExcludedPackages; // 见下面；

        public ClientRestrictionState(IBinder token)
                throws RemoteException {
            //【1】绑定远程进程的 Binder 对象！
            token.linkToDeath(this, 0);
            this.token = token;
        }
        
        @Override
        public void binderDied() {
            synchronized (AppOpsService.this) {
                //【2】从 mOpUserRestrictions 中该 token 相应关系，解除了用户限制！
                mOpUserRestrictions.remove(token);
                if (perUserRestrictions == null) {
                    return;
                }
                final int userCount = perUserRestrictions.size();
                for (int i = 0; i < userCount; i++) {
                    final boolean[] restrictions = perUserRestrictions.valueAt(i);
                    final int restrictionCount = restrictions.length;
                    for (int j = 0; j < restrictionCount; j++) {
                        //【1】如果有 op 之前是开启了用户限制，那么需要通知所有监听 op 状态变化的回调
                        //【1.9.4】调用了 notifyWatchersOfChange 方法！
                        if (restrictions[j]) {
                            final int changedCode = j;
                            mHandler.post(() -> notifyWatchersOfChange(changedCode));
                        }
                    }
                }
                destroy();
            }
        }
    }
```
ClientRestrictionState 实现了 DeathRecipient 接口，当和其绑定的远程 Binder 对象死亡后会触发其 binderDied 方法！

- **perUserRestrictions** 是一个 SparseArray，下标为 userId，而 SparseArray[i] 则是一个 boolean[] 数组，该数组长度为 AppOpsManager._NUM_OP 表示在 userId 下，每个 op 的限制情况，为 true 表示限制，为 false 表示不限制！

- **perUserExcludedPackages** 也是一个 SparseArray，下标为 userId，而 SparseArray[i] 则是一个 String[] 数组，该数字保存白名单，在该名单中的应用不会受到用户限制！

当 token 对应的远程 Binder 对象死亡后，会触发 ClientRestrictionState.binderDied 方法

## 9.3 ClientRestrictionState.setRestriction

设置用户限制，restricted 表示是否限制！
```java
        public boolean setRestriction(int code, boolean restricted, String[] excludedPackages, int userId) {
            boolean changed = false;
            if (perUserRestrictions == null && restricted) {
                perUserRestrictions = new SparseArray<>();
            }

            if (perUserRestrictions != null) {
                //【1】获得 userId 下，所有 op 的限制情况！
                boolean[] userRestrictions = perUserRestrictions.get(userId);
                if (userRestrictions == null && restricted) {
                    //【1.1】如果 userRestrictions 为 null，且当前要设置为限制，那就会初始化 userRestrictions
                    // 为一个 AppOpsManager._NUM_OP 大小的数组，添加到 perUserRestrictions 中！
                    userRestrictions = new boolean[AppOpsManager._NUM_OP];
                    perUserRestrictions.put(userId, userRestrictions);
                }
                //【2】尝试更新 userRestrictions[code] 的值，userRestrictions[code] 表示的是 op 的限制状态！
                // 只有当限制状态变化时，才会更新！
                if (userRestrictions != null && userRestrictions[code] != restricted) {
                    // 更新 userRestrictions[code]；
                    userRestrictions[code] = restricted;
                    //【×9.3.1】如果本次是取消用户限制，并且 userId 下的所有的 op 都是不开启限制的！
                    // 那么从 perUserRestrictions 中移除该 userId 的记录！
                    if (!restricted && isDefault(userRestrictions)) {
                        perUserRestrictions.remove(userId);
                        userRestrictions = null;
                    }
                    changed = true; // 表示发生了更新！
                }

                //【3】userRestrictions 不为 null，说明在该 userId 下对某些 op 进行了用户限制！
                if (userRestrictions != null) {
                    final boolean noExcludedPackages = ArrayUtils.isEmpty(excludedPackages);
                    if (perUserExcludedPackages == null && !noExcludedPackages) {
                        //【3.1】如果本次指定了白名单，并且 perUserExcludedPackages 为 null
                        // 那么会进行初始化；
                        perUserExcludedPackages = new SparseArray<>();
                    }
                    //【3.2】本次指定的白名单 excludedPackages 和该 userId 下已有的名单不一样！
                    // 如果本次未指定白名单，那就移除该 userId 下已有的名单；
                    // 如果本次指定了白名单，那就更新该 userId 下已有的名单；
                    if (perUserExcludedPackages != null && !Arrays.equals(excludedPackages,
                            perUserExcludedPackages.get(userId))) {
                        if (noExcludedPackages) {
                            perUserExcludedPackages.remove(userId);
                            if (perUserExcludedPackages.size() <= 0) {
                                perUserExcludedPackages = null;
                            }
                        } else {
                            perUserExcludedPackages.put(userId, excludedPackages);
                        }
                        //【3.1】名单更新了，设置 changed 为 true！
                        changed = true;
                    }
                }
            }
            return changed;
        }
```
继续分析！

### 9.3.1 ClientRestrictionState.isDefault

有两个 default 方法：
```java
        public boolean isDefault() {
            return perUserRestrictions == null || perUserRestrictions.size() <= 0;
        }
```
无参数 isDefault 方法用来判断该 ClientRestrictionState 对象是否是默认状态！

```java
        private boolean isDefault(boolean[] array) {
            if (ArrayUtils.isEmpty(array)) {
                return true;
            }
            for (boolean value : array) {
                if (value) {
                    return false;
                }
            }
            return true;
        }
```
一参数 isDefault 方法用来判断指定 ClientRestrictionState.perUserRestrictions 中指定 userId 下的用户限制是否是默认状态！

## 9.4 notifyWatchersOfChange

当用户限制状态发生变化后，AppOpsService 会调用 notifyWatchersOfChange 通知所有监听 op 变化的监听者！

int code 参数表示用户限制状态发生变化的 op code！

```java
    private void notifyWatchersOfChange(int code) {
        final ArrayList<Callback> clonedCallbacks;
        //【1】从 mOpModeWatchers 中获得该 op 变化后需要出发的回调列表！
        synchronized (this) {
            ArrayList<Callback> callbacks = mOpModeWatchers.get(code);
            if (callbacks == null) {
                return;
            }
            clonedCallbacks = new ArrayList<>(callbacks);
        }

        final long identity = Binder.clearCallingIdentity();
        try {
            final int callbackCount = clonedCallbacks.size();
            for (int i = 0; i < callbackCount; i++) {
                Callback callback = clonedCallbacks.get(i);
                try {
                    //【2】执行回调的 opChanged 方法！
                    callback.mCallback.opChanged(code, -1, null);
                } catch (RemoteException e) {
                    Log.w(TAG, "Error dispatching op op change", e);
                }
            }
        } finally {
            Binder.restoreCallingIdentity(identity);
        }
    }
```