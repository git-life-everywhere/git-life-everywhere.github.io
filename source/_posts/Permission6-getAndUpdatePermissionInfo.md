# Permission第 6 篇 - permission info 的获取和更新
title: Permission第 6 篇 - permission info 的获取和更新
date: 2017/12/35 20:46:25
catalog: true
categories:
- AndroidFramework源码分析
- Permission权限管理
tags: Permission权限管理
---
[toc]

# 0 综述

基于 Android 7.1.1，分析权限管理相关知识，本文权限信息的获取！

PackageManagerService 提供了很多个接口用于获取权限的信息！

# 1 获取权限组相关的信息！

PackageManagerService.mPermissionGroups 保存了从所有 Application 中解析到的权限组信息！

PackageManagerService 提供了如下的两个接口来获得权限组的信息！

## 1.1 PackageManagerS.getAllPermissionGroups

获得所有的权限组
```java
    @Override
    public @NonNull ParceledListSlice<PermissionGroupInfo> getAllPermissionGroups(int flags) {
        synchronized (mPackages) {
            final int N = mPermissionGroups.size();
            ArrayList<PermissionGroupInfo> out
                    = new ArrayList<PermissionGroupInfo>(N);
            for (PackageParser.PermissionGroup pg : mPermissionGroups.values()) {
                //【*1.2.1】调用了 PackageParser.generatePermissionGroupInfo 方法
                out.add(PackageParser.generatePermissionGroupInfo(pg, flags));
            }
            return new ParceledListSlice<>(out);
        }
    }
```

## 1.2 PackageManagerS.getPermissionGroupInfo

获得指定的权限组信息

```java
    @Override
    public PermissionGroupInfo getPermissionGroupInfo(String name, int flags) {
        synchronized (mPackages) {
            //【*1.2.1】调用了 PackageParser.generatePermissionGroupInfo 方法
            return PackageParser.generatePermissionGroupInfo(
                    mPermissionGroups.get(name), flags);
        }
    }
```

### 1.2.1 PackageParser.generatePermissionGroupInfo

该方法会新创建的 PermissionGroupInfo 对象，作为解析数据 PermissionGroup.PermissionGroupInfo 的拷贝！！

```
    public static final PermissionGroupInfo generatePermissionGroupInfo(
            PermissionGroup pg, int flags) {
        if (pg == null) return null;
        // 如果 flags 没有设置 PackageManager.GET_META_DATA，直接返回 PermissionGroup.PermissionGroupInfo
        if ((flags & PackageManager.GET_META_DATA) == 0) {
            return pg.info;
        }
        // 如果 flags 设置了 PackageManager.GET_META_DATA，我们会新建一个 PermissionGroupInfo 对象，
        // 将解析的数据拷贝进来！
        PermissionGroupInfo pgi = new PermissionGroupInfo(pg.info);
        pgi.metaData = pg.metaData;
        return pgi;
    }
```

# 2 获取权限相关的信息！

mSettings.mPermissions 保存了系统和应用定义的所有的权限信息！

PackageManagerService 提供了如下的两个接口来获得权限组的信息！

## 2.1 PackageParser.getPermissionInfo

获得指定 name 的权限信息！

```java
    @Override
    public PermissionInfo getPermissionInfo(String name, int flags) {
        synchronized (mPackages) {
            final BasePermission p = mSettings.mPermissions.get(name);
            if (p != null) {
                //【2.2.1】调用了 PackageParser.generatePermissionInfo 方法
                return generatePermissionInfo(p, flags);
            }
            return null;
        }
    }
```


## 2.2 PackageParser.queryPermissionsByGroup

获得同一个 group 中的所有权限信息！

```java
    @Override
    public @Nullable ParceledListSlice<PermissionInfo> queryPermissionsByGroup(String group,
            int flags) {
        synchronized (mPackages) {
            //【1】校验权限组是否存在！
            if (group != null && !mPermissionGroups.containsKey(group)) {
                // This is thrown as NameNotFoundException
                return null;
            }
            //【2.2.1】调用了 PackageParser.generatePermissionInfo 方法
            ArrayList<PermissionInfo> out = new ArrayList<PermissionInfo>(10);
            for (BasePermission p : mSettings.mPermissions.values()) {
                if (group == null) { // 如果参数 group 为 null，只收集无 group 的权限！
                    if (p.perm == null || p.perm.info.group == null) {
                        out.add(generatePermissionInfo(p, flags));
                    }
                } else {
                    if (p.perm != null && group.equals(p.perm.info.group)) {
                        out.add(PackageParser.generatePermissionInfo(p.perm, flags));
                    }
                }
            }
            return new ParceledListSlice<>(out);
        }
    }
```

### 2.2.1 PackageParser.generatePermissionInfo

该方法会新创建的 PermissionInfo 对象，拷贝 Permission 的数据！！

```java
    public static final PermissionInfo generatePermissionInfo(
            Permission p, int flags) {
        if (p == null) return null;
        if ((flags & PackageManager.GET_META_DATA) == 0) {
            return p.info;
        }
        PermissionInfo pi = new PermissionInfo(p.info);
        pi.metaData = p.metaData;
        return pi;
    }
```
如果不需要 GET_META_DATA，那就直接返回解析得到的 PermissionInfo 对象！


# 3 获取权限标志位的信息！

PackageManagerService 提供了如下接口来获得权限标志位组的信息！

PackageManagerService.mPackages 用于保存所有解析过的 Pacakge 信息！

mSettings.mPermissions 中保存了系统中所有的权限信息！

## 3.1 PackageManagerService.getPermissionFlags

```java
    @Override
    public int getPermissionFlags(String name, String packageName, int userId) {
        if (!sUserManager.exists(userId)) {
            return 0;
        }

        enforceGrantRevokeRuntimePermissionPermissions("getPermissionFlags");

        enforceCrossUserPermission(Binder.getCallingUid(), userId,
                true /* requireFullPermission */, false /* checkShell */,
                "getPermissionFlags");

        synchronized (mPackages) {
            //【1】如果 packageName 不存在，返回 0；
            final PackageParser.Package pkg = mPackages.get(packageName);
            if (pkg == null) {
                return 0;
            }
            //【2】如果权限不存在，返回 0；
            final BasePermission bp = mSettings.mPermissions.get(name);
            if (bp == null) {
                return 0;
            }
            //【3】获得该 package 对应的 PackageSettings 或者 SharedUserSetting 对象！
            SettingBase sb = (SettingBase) pkg.mExtras;
            if (sb == null) {
                return 0;
            }
            //【4】获得权限的 flags
            PermissionsState permissionsState = sb.getPermissionsState();
            return permissionsState.getPermissionFlags(name, userId);
        }
    }
```
方法很简单，不多说了！

# 4 更新权限标志位的信息！

PackageManagerService 提供了如下接口来更新权限标志位的信息！

## 4.1 PackageManagerService.updatePermissionFlags

该方法更新指定权限的 flags，flagMask 表示的是 flags 的位掩码，用来屏蔽某些位；flagValues 表示新的标志位值！

```java
    @Override
    public void updatePermissionFlags(String name, String packageName, int flagMask,
            int flagValues, int userId) {
        if (!sUserManager.exists(userId)) {
            return;
        }

        enforceGrantRevokeRuntimePermissionPermissions("updatePermissionFlags");

        enforceCrossUserPermission(Binder.getCallingUid(), userId,
                true /* requireFullPermission */, true /* checkShell */,
                "updatePermissionFlags");

        //【1】如果调用者不是 system uid，不能改变以下标志位，flagMask  和 flagValues 需去掉对应标志位：
        if (getCallingUid() != Process.SYSTEM_UID) {
            flagMask &= ~PackageManager.FLAG_PERMISSION_SYSTEM_FIXED;
            flagValues &= ~PackageManager.FLAG_PERMISSION_SYSTEM_FIXED;
            flagMask &= ~PackageManager.FLAG_PERMISSION_GRANTED_BY_DEFAULT;
            flagValues &= ~PackageManager.FLAG_PERMISSION_GRANTED_BY_DEFAULT;
            flagValues &= ~PackageManager.FLAG_PERMISSION_REVIEW_REQUIRED;
        }

        synchronized (mPackages) {
            //【2】如果该 package 不存在，抛出异常！
            final PackageParser.Package pkg = mPackages.get(packageName);
            if (pkg == null) {
                throw new IllegalArgumentException("Unknown package: " + packageName);
            }
            //【3】如果权限 name 不存在，抛出异常！
            final BasePermission bp = mSettings.mPermissions.get(name);
            if (bp == null) {
                throw new IllegalArgumentException("Unknown permission: " + name);
            }
            //【4】如果该 package 没有安装记录，抛出异常！
            SettingBase sb = (SettingBase) pkg.mExtras;
            if (sb == null) {
                throw new IllegalArgumentException("Unknown package: " + packageName);
            }
            //【5】获得该 package 的权限状态管理对象！
            PermissionsState permissionsState = sb.getPermissionsState();

            //【6】获得该应用程序的运行时权限状态信息，返回不为 null，说明其有运行时权限！
            boolean hadState = permissionsState.getRuntimePermissionState(name, userId) != null;

            //【*4.1.1】更新该权限的标志位！
            if (permissionsState.updatePermissionFlags(bp, userId, flagMask, flagValues)) {
                // 安装时权限和运行时权限保存在不同的目录下，所以要更新不同的文件
                if (permissionsState.getInstallPermissionState(name) != null) {
                    scheduleWriteSettingsLocked(); // 更新运行时权限！
                    
                } else if (permissionsState.getRuntimePermissionState(name, userId) != null
                        || hadState) { // 更新安装时权限！
                    mSettings.writeRuntimePermissionsForUserLPr(userId, false);
                }
            }
        }
    }
```
先会更新 flags，更新成功后，会根据权限的类型去，去更新对应的持久化文件！

### 4.1.1 PermissionsState.updatePermissionFlags

PermissionsState 用于管理 package 的权限状态!

```java
    public boolean updatePermissionFlags(BasePermission permission, int userId,
            int flagMask, int flagValues) {
        enforceValidUserId(userId);
        //【1】如果 flagValues 和 flagMask 有一个不为 0，那就需要更新 flags！
        final boolean mayChangeFlags = flagValues != 0 || flagMask != 0;

        if (mPermissions == null) {
            if (!mayChangeFlags) {
                return false;
            }
            ensurePermissionData(permission);
        }

        PermissionData permissionData = mPermissions.get(permission.name);
        if (permissionData == null) {
            if (!mayChangeFlags) {
                return false;
            }
            permissionData = ensurePermissionData(permission);
        }
        //【2】获得旧的 flags！
        final int oldFlags = permissionData.getFlags(userId);
        
        //【*4.1.1.1】调用 PermissionData.updatePermissionFlags 更新权限的标志位：
        final boolean updated = permissionData.updateFlags(userId, flagMask, flagValues);

        //【3】如果 flags 发生了更新，比较下，更新后是否需要再次 review！
        if (updated) {
            final int newFlags = permissionData.getFlags(userId); // 获得新的 flags！

            if ((oldFlags & PackageManager.FLAG_PERMISSION_REVIEW_REQUIRED) == 0
                    && (newFlags & PackageManager.FLAG_PERMISSION_REVIEW_REQUIRED) != 0) {

                if (mPermissionReviewRequired == null) {
                    mPermissionReviewRequired = new SparseBooleanArray();
                }
                mPermissionReviewRequired.put(userId, true);

            } else if ((oldFlags & PackageManager.FLAG_PERMISSION_REVIEW_REQUIRED) != 0
                    && (newFlags & PackageManager.FLAG_PERMISSION_REVIEW_REQUIRED) == 0) {

                if (mPermissionReviewRequired != null) {
                    mPermissionReviewRequired.delete(userId);
                    if (mPermissionReviewRequired.size() <= 0) {
                        mPermissionReviewRequired = null;
                    }
                }
            }
        }
        return updated;
    }
```

整个方法很简单，无需多说，前面分析过了，这里就不多说了！！

#### 4.1.1.1 PermissionData.updateFlags

PermissionData 用于封装指定权限的状态信息！

```java
        public boolean updateFlags(int userId, int flagMask, int flagValues) {
            if (isInstallPermission()) {
                userId = UserHandle.USER_ALL;
            }

            if (!isCompatibleUserId(userId)) {
                return false;
            }

            //【1】新的 newFlags 取 flagValues 和 flagMask 相同的位值！
            // 就是说，新的 flags 要么是 0，要么只能取和 flagMask 相同的位值！
            final int newFlags = flagValues & flagMask;

            PermissionState userState = mUserStates.get(userId);
            if (userState != null) {
                final int oldFlags = userState.mFlags;
                //【2】最新的权限 flags 设置如下：
                // 先取 oldFlags 和 ~flagMask 相同的位值，然后加上 newFlags！
                userState.mFlags = (userState.mFlags & ~flagMask) | newFlags;
                if (userState.isDefault()) {
                    mUserStates.remove(userId);
                }
                return userState.mFlags != oldFlags; // 判断标志位是否变化！

            } else if (newFlags != 0) {
                userState = new PermissionState(mPerm.name);
                userState.mFlags = newFlags;
                mUserStates.put(userId, userState);
                return true;
            }

            return false;
        }
```
这里首先，通过 flagValues & flagMask 取其相同的位值为 newFlags！

设置最新的 flags 的时候，先是 oldFlags & ~flagMask 取 oldFlags 和 ～flagMask 相同的位值，然后加上 newFlags！

## 4.2 PackageManagerService.updatePermissionFlagsForAllApps

该方法更新指定所有权限的 flags！

```java
    @Override
    public void updatePermissionFlagsForAllApps(int flagMask, int flagValues, int userId) {
        if (!sUserManager.exists(userId)) {
            return;
        }

        enforceGrantRevokeRuntimePermissionPermissions("updatePermissionFlagsForAllApps");

        enforceCrossUserPermission(Binder.getCallingUid(), userId,
                true /* requireFullPermission */, true /* checkShell */,
                "updatePermissionFlagsForAllApps");

        //【1】如果不是 system uid，不能修改 system fixed flags，从 flagMask 和 flagValues 中去掉该标志位！
        // 那么下面的调整中就不会涉及到 system fix 标志位！
        if (getCallingUid() != Process.SYSTEM_UID) {
            flagMask &= ~PackageManager.FLAG_PERMISSION_SYSTEM_FIXED;
            flagValues &= ~PackageManager.FLAG_PERMISSION_SYSTEM_FIXED;
        }

        synchronized (mPackages) {
            boolean changed = false;
            final int packageCount = mPackages.size();
            for (int pkgIndex = 0; pkgIndex < packageCount; pkgIndex++) {
                final PackageParser.Package pkg = mPackages.valueAt(pkgIndex);
                SettingBase sb = (SettingBase) pkg.mExtras;
                if (sb == null) {
                    continue;
                }
                PermissionsState permissionsState = sb.getPermissionsState();
                //【*4.2.1】调用了 updatePermissionFlagsForAllPermissions 方法，更新 flags！
                changed |= permissionsState.updatePermissionFlagsForAllPermissions(
                        userId, flagMask, flagValues);
            }
            if (changed) { 
                //【2】如果发生了改变，更新 rumtime-permissions.xml 文件！
                mSettings.writeRuntimePermissionsForUserLPr(userId, false);
            }
        }
    }
```

### 4.2.1 PermissionsState.updatePermissionFlagsForAllPermissions

```java
    public boolean updatePermissionFlagsForAllPermissions(
            int userId, int flagMask, int flagValues) {
        enforceValidUserId(userId);

        if (mPermissions == null) {
            return false;
        }
        boolean changed = false;
        final int permissionCount = mPermissions.size();
        //【*4.1.1.1】更新 PermissionsState 管理的所有权限的 flags！
        for (int i = 0; i < permissionCount; i++) {
            PermissionData permissionData = mPermissions.valueAt(i);
            changed |= permissionData.updateFlags(userId, flagMask, flagValues);
        }
        return changed;
    }
```
流程很简单，不多说了！