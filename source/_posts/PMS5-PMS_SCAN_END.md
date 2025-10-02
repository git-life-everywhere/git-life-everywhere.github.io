# PMS 第 5 篇 - PMS_SCAN_END 阶段
title: PMS 第 5 篇 - PMS_SCAN_END 阶段
date: 2018/03/03
categories:
- AndroidFramework源码分析
- PackageManager包管理
tags: PackageManager包管理
---

[toc]

基于 Android 7.1.1 源码分析 PackageManagerService 的架构和逻辑实现，本文是作者原创，转载请说明出处！

# 0 综述

system 分区和 data 分区的扫描过程到这里就结束了，下面我们来分析下：

```java
            ... ... ... ...// 第四阶段

            EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_PMS_SCAN_END,
                    SystemClock.uptimeMillis());

            Slog.i(TAG, "Time to scan packages: "
                    + ((SystemClock.uptimeMillis()-startTime)/1000f)
                    + " seconds");

            // 如果从我们上次启动，SDK 平台被改变了，我们需要重新授予应用程序权限。
            // 这里会有一些安全问题，就是可能会有应用通过这种方式获取那些用户没有显式允许的权限
            // 这里可能后续 google 会改善！
            int updateFlags = UPDATE_PERMISSIONS_ALL;
            if (ver.sdkVersion != mSdkVersion) {
                Slog.i(TAG, "Platform changed from " + ver.sdkVersion + " to "
                        + mSdkVersion + "; regranting permissions for internal storage");
                updateFlags |= UPDATE_PERMISSIONS_REPLACE_PKG | UPDATE_PERMISSIONS_REPLACE_ALL;
            }
            
            //【1】更新系统中的权限和权限树，移除无效的权限和权限树，同时更新应用的权限授予情况！
            updatePermissionsLPw(null, null, StorageManager.UUID_PRIVATE_INTERNAL, updateFlags);
            ver.sdkVersion = mSdkVersion;

            //【2】如果这是第一次开机或从 Anroid M之前的版本升级上来的，然后我们需要初始化默认应用程序给所有的系统用户！
            if (!onlyCore && (mPromoteSystemApps || mFirstBoot)) {
                for (UserInfo user : sUserManager.getUsers(true)) {
                    mSettings.applyDefaultPreferredAppsLPw(this, user.id);
                    applyFactoryDefaultBrowserLPw(user.id);
                    primeDomainVerificationsLPw(user.id);
                }
            }

            //【3】在启动完成前，为系统用户准备文件存储，因为很多核心的系统比如设置，系统界面等等会提前启动！
            final int storageFlags;
            if (StorageManager.isFileEncryptedNativeOrEmulated()) {
                storageFlags = StorageManager.FLAG_STORAGE_DE;
            } else {
                storageFlags = StorageManager.FLAG_STORAGE_DE | StorageManager.FLAG_STORAGE_CE;
            }
            reconcileAppsDataLI(StorageManager.UUID_PRIVATE_INTERNAL, UserHandle.USER_SYSTEM,
                    storageFlags);

            //【4】如果这是在 OTA 升级后第一次正常启动，然后我们需要清除代码缓存目录。
            // 我们这里使用了 Installer.FLAG_CLEAR_CODE_CACHE_ONLY 标志位；
            if (mIsUpgrade && !onlyCore) {
                Slog.i(TAG, "Build fingerprint changed; clearing code caches");
                for (int i = 0; i < mSettings.mPackages.size(); i++) {
                    final PackageSetting ps = mSettings.mPackages.valueAt(i);
                    if (Objects.equals(StorageManager.UUID_PRIVATE_INTERNAL, ps.volumeUuid)) {
                        clearAppDataLIF(ps.pkg, UserHandle.USER_ALL,
                                StorageManager.FLAG_STORAGE_DE | StorageManager.FLAG_STORAGE_CE
                                        | Installer.FLAG_CLEAR_CODE_CACHE_ONLY);
                    }
                }
                ver.fingerprint = Build.FINGERPRINT;
            }

            checkDefaultBrowser(); // 检查默认的浏览器

            // 当权限和其他默认设置被更新后，执行清除操作！
            mExistingSystemPackages.clear();
            mPromoteSystemApps = false;

            // 更新系统数据库版本号！
            ver.databaseVersion = Settings.CURRENT_DATABASE_VERSION;

            //【5】信息写回 packages.xml 等文件！
            mSettings.writeLPr();

            //【6】如果是第一次开机，或者是系统升级，对核心的系统应用执行 odex 操作！
            if ((isFirstBoot() || isUpgrade() || VMRuntime.didPruneDalvikCache()) && !onlyCore) {
                long start = System.nanoTime();
                List<PackageParser.Package> coreApps = new ArrayList<>();
                for (PackageParser.Package pkg : mPackages.values()) {
                    if (pkg.coreApp) {
                        coreApps.add(pkg);
                    }
                }

                int[] stats = performDexOptUpgrade(coreApps, false,
                        getCompilerFilterForReason(REASON_CORE_APP));

                final int elapsedTimeSeconds =
                        (int) TimeUnit.NANOSECONDS.toSeconds(System.nanoTime() - start);
                MetricsLogger.histogram(mContext, "opt_coreapps_time_s", elapsedTimeSeconds);

                if (DEBUG_DEXOPT) {
                    Slog.i(TAG, "Dex-opt core apps took : " + elapsedTimeSeconds + " seconds (" +
                            stats[0] + ", " + stats[1] + ", " + stats[2] + ")");
                }


                // TODO: Should we log these stats to tron too ?
                // MetricsLogger.histogram(mContext, "opt_coreapps_num_dexopted", stats[0]);
                // MetricsLogger.histogram(mContext, "opt_coreapps_num_skipped", stats[1]);
                // MetricsLogger.histogram(mContext, "opt_coreapps_num_failed", stats[2]);
                // MetricsLogger.histogram(mContext, "opt_coreapps_num_total", coreApps.size());
            }
            
            ... ... ... ...// 见，第五阶段
```
下面我们来分析下这个过程！


# 1 PackageMS.updatePermissionsLPw - 更新权限信息

updatePermissionsLPw 方法用于更新应用的权限信息，首先我们来看看 updateFlags！

```java
            int updateFlags = UPDATE_PERMISSIONS_ALL; // 表示更新所有的权限信息；
            if (ver.sdkVersion != mSdkVersion) {
                // 如果 sdkVersion 发生了变化，还会额外的设置以下的 2 个权限！
                // sdkVersion 发生变化一般是在大版本系统升级时，比如 7.1 升级 8.0！
                Slog.i(TAG, "Platform changed from " + ver.sdkVersion + " to "
                        + mSdkVersion + "; regranting permissions for internal storage");
                updateFlags |= UPDATE_PERMISSIONS_REPLACE_PKG | UPDATE_PERMISSIONS_REPLACE_ALL;
            }
```

updatePermissionsLPw 的逻辑如下，这里的 changingPkg 和 pkgInfo 均为 null；

replaceVolumeUuid 的值为：StorageManager.UUID_PRIVATE_INTERNAL，表示的是安装位置，这里传入的值表示内置存储；

（这里简单的说一下参数的意思：String changingPkg 指定了权限发生变化的包名，PackageParser.Package pkgInfo 表示的是 changingPkg 对应的应用的信息）

```java
    private void updatePermissionsLPw(String changingPkg,
            PackageParser.Package pkgInfo, String replaceVolumeUuid, int flags) {
        //【1】确保系统中没有废弃的权限树；
        Iterator<BasePermission> it = mSettings.mPermissionTrees.values().iterator();
        while (it.hasNext()) {
            final BasePermission bp = it.next();
            if (bp.packageSetting == null) {
                //【1.1】如果 packageSetting 为 null，尝试查找定义的 package！
                bp.packageSetting = mSettings.mPackages.get(bp.sourcePackage);
            }

            if (bp.packageSetting == null) {
                //【1.2】如果找不到定义该权限的 package，那就移除该 permission-tree！
                Slog.w(TAG, "Removing dangling permission tree: " + bp.name
                        + " from package " + bp.sourcePackage);
                it.remove();

            } else if (changingPkg != null && changingPkg.equals(bp.sourcePackage)) {
                //【1.3】根据参数，不进入这里，但是我们来分析下这部分代码的意思
                // 如果权限的定义者不再定义这个权限了，那就移除该权限的上一次信息！
                if (pkgInfo == null || !hasPermission(pkgInfo, bp.name)) {
                    Slog.i(TAG, "Removing old permission tree: " + bp.name
                            + " from package " + bp.sourcePackage);
                    //【1.3.1】同时 flags 会被设置 UPDATE_PERMISSIONS_ALL 标志位！
                    flags |= UPDATE_PERMISSIONS_ALL;
                    it.remove();
                }
            }
        }

        //【2】确保系统中所有的动态权限都已经分配给对应的 package，同时系统中也没有废弃的权限!
        it = mSettings.mPermissions.values().iterator();
        while (it.hasNext()) {
            final BasePermission bp = it.next();
            //【2.1】如果是动态权限，尝试找到其所属的 permission-tree，并绑定到定义 tree 的 package 上！
            if (bp.type == BasePermission.TYPE_DYNAMIC) {
                if (DEBUG_SETTINGS) Log.v(TAG, "Dynamic permission: name="
                        + bp.name + " pkg=" + bp.sourcePackage
                        + " info=" + bp.pendingInfo);
    
                if (bp.packageSetting == null && bp.pendingInfo != null) {
                    final BasePermission tree = findPermissionTreeLP(bp.name);
                    if (tree != null && tree.perm != null) {
                        bp.packageSetting = tree.packageSetting;
                        bp.perm = new PackageParser.Permission(tree.perm.owner,
                                new PermissionInfo(bp.pendingInfo));
                        bp.perm.info.packageName = tree.perm.info.packageName;
                        bp.perm.info.name = bp.name;
                        bp.uid = tree.uid;
                    }
                }
            }

            //【2.2】如果权限的定义者为 null，尝试在系统中找到定义者！
            if (bp.packageSetting == null) {
                bp.packageSetting = mSettings.mPackages.get(bp.sourcePackage);
            }

            //【2.3】如果依然找不到定义该权限的 package，那就移除该 permission ！
            if (bp.packageSetting == null) { // 该权限无效，移除它！
                Slog.w(TAG, "Removing dangling permission: " + bp.name
                        + " from package " + bp.sourcePackage);
                it.remove();

            } else if (changingPkg != null && changingPkg.equals(bp.sourcePackage)) {
                //【2.4】根据参数，不进入这里，但是我们来分析下这部分代码的意思
                // 如果权限的定义者不在定义这个权限了，那就移除该权限的上一次信息！
                if (pkgInfo == null || !hasPermission(pkgInfo, bp.name)) {
                    Slog.i(TAG, "Removing old permission: " + bp.name
                            + " from package " + bp.sourcePackage);
                    //【2.4.1】同时 flags 会被设置 UPDATE_PERMISSIONS_ALL 标志位！
                    flags |= UPDATE_PERMISSIONS_ALL;
                    it.remove();
                }
            }
        }

        //【3】如果 flags 设置了 UPDATE_PERMISSIONS_ALL 标志位，那么就更新除了 changingPkg 以外的其他
        // 所有的 package 的权限授予信息 ，特别是替换系统包的授予权限！
        if ((flags & UPDATE_PERMISSIONS_ALL) != 0) {
            for (PackageParser.Package pkg : mPackages.values()) {
                if (pkg != pkgInfo) {
                    //【3.1】获得该应用的安装位置，我们只会修改安装在 replaceVolumeUuid 指定的位置的应用的权限信息！
                    // 当然默认是在内置存储，所以返回的是 UUID_PRIVATE_INTERNAL！
                    final String volumeUuid = getVolumeUuidForPackage(pkg);
                    // 如果 sdk 发生了变化，replace 为 true！
                    final boolean replace = ((flags & UPDATE_PERMISSIONS_REPLACE_ALL) != 0)
                            && Objects.equals(replaceVolumeUuid, volumeUuid);
                    //【×1.2】重新授予其他 pkg 权限！
                    grantPermissionsLPw(pkg, replace, changingPkg);
                }
            }
        }

        //【4】因为这里 pkgInfo 为 null，所以不进入这里！
        // 这里的逻辑是，如果 changingPkg 不为 null，最后也会更新其权限的授予信息！
        if (pkgInfo != null) { 
            //【4.1】获得该应用的安装位置，我们只会修改安装在 replaceVolumeUuid 指定的位置的应用的权限信息！
            // 当然默认是在内置存储，所以返回的是 UUID_PRIVATE_INTERNAL！
            final String volumeUuid = getVolumeUuidForPackage(pkgInfo);
            // 如果 sdk 发生了变化，replace 为 true！
            final boolean replace = ((flags & UPDATE_PERMISSIONS_REPLACE_PKG) != 0)
                    && Objects.equals(replaceVolumeUuid, volumeUuid);
            //【×1.2】重新授予发生 change 的 pkg 权限！
            grantPermissionsLPw(pkgInfo, replace, changingPkg);
        }
    }
```
由于这里的 changingPkg 和 pkgInfo 均为 null，所以不仅移除了无效的权限和权限树，而且更新了系统中所有应用的权限信息！

这里涉及到了 flags 标志位，可以取以下值：

```java
static final int UPDATE_PERMISSIONS_ALL = 1<<0; // 更新所有应用的权限信息；
static final int UPDATE_PERMISSIONS_REPLACE_PKG = 1<<1;
static final int UPDATE_PERMISSIONS_REPLACE_ALL = 1<<2;
```
下面的分析过程中会遇到：

我们继续分析！

## 1.1 PackageMS.findPermissionTreeLP

找到权限名所属的权限树！
```java
    private BasePermission findPermissionTreeLP(String permName) {
        for(BasePermission bp : mSettings.mPermissionTrees.values()) {
            if (permName.startsWith(bp.name) &&
                    permName.length() > bp.name.length() &&
                    permName.charAt(bp.name.length()) == '.') {
                return bp;
            }
        }
        return null;
    }
```


## 1.2 PackageMS.grantPermissionsLPw

在更新过权限后，会再次授予应用权限，因为可能有些权限已经失效并且被移除！

Android 系统有两种类型的权限：安装和运行时，dangerous 属于运行时权限，normal 和 signiture 属于安装时权限。 

正常和签名保护级别的权限是安装时权限。 而危险级别的权限是运行时权限！

如果应用的目标 SDK 为 Lollipop MR1 或更早版本，则运行时权限默认按照安装时权限处理。根据上面的代码跟踪

这里的 replace 为 true，由于我们是更新所有的 pacakge，所以 pkg 和 packageOfInterest 均为 null，参数 pkg 是需要更新权限授予信息的应用！

```java
    private void grantPermissionsLPw(PackageParser.Package pkg, boolean replace,
            String packageOfInterest) {
        //【1】如果该应用的 PackageSetting 没有，那么不处理！
        final PackageSetting ps = (PackageSetting) pkg.mExtras;
        if (ps == null) {
            return;
        }

        Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "grantPermissions");
        
        //【2】获得该 package 的权限管理对象，此时这个权限管理对象是解析上次安装信息时恢复的！
        // 我们将其保存到 origPermissions 中，作为原始数据！
        PermissionsState permissionsState = ps.getPermissionsState();
        PermissionsState origPermissions = permissionsState;

        final int[] currentUserIds = UserManagerService.getInstance().getUserIds();

        boolean runtimePermissionsRevoked = false;
        int[] changedRuntimePermissionUserIds = EMPTY_INT_ARRAY;

        boolean changedInstallPermission = false;

        //【3】如果 replace 的值为 true，进入这里！
        if (replace) {
            // 设置 installPermissionsFixed 为 false，因为下面会对权限信息调整！
            ps.installPermissionsFixed = false;
            if (!ps.isSharedUser()) {
                //【3.1】如果该 package 不是共享 uid 的，那就重置 permissionsState，但是会将
                // 原始信息 copy到 origPermissions 中！
                origPermissions = new PermissionsState(permissionsState);
                permissionsState.reset();

            } else {
                //【3.2】如果该 package 是共享 uid 的，那么我们会拒绝那些不用的共享 uid 相关权限信息！
                // （我们只需要知道运行时权限更改，因为调用代码总是写入安装权限状态，但是运行时权限只有在更改时才写入。
                // 这里更改运行时权限的唯一情况是将安装提升到运行时并从共享用户撤消运行时。）
                // 上面这段文字是根据注释翻译的，生涩，目前我自己也不是很明白。。。
                changedRuntimePermissionUserIds = revokeUnusedSharedUserPermissionsLPw(
                        ps.sharedUser, UserManagerService.getInstance().getUserIds());
                if (!ArrayUtils.isEmpty(changedRuntimePermissionUserIds)) {
                    runtimePermissionsRevoked = true;
                }
            }
        }

        permissionsState.setGlobalGids(mGlobalGids);

        //【4】处理该 package 请求的所有权限！
        final int N = pkg.requestedPermissions.size();
        for (int i=0; i < N; i++) {
            final String name = pkg.requestedPermissions.get(i);
            final BasePermission bp = mSettings.mPermissions.get(name);

            if (DEBUG_INSTALL) {
                Log.i(TAG, "Package " + pkg.packageName + " checking " + name + ": " + bp);
            }
            //【4.1】如果该权限没有定义，那么跳过！！
            if (bp == null || bp.packageSetting == null) {
                if (packageOfInterest == null || packageOfInterest.equals(pkg.packageName)) {
                    Slog.w(TAG, "Unknown permission " + name
                            + " in package " + pkg.packageName);
                }
                continue;
            }

            final String perm = bp.name;
            boolean allowedSig = false;
            int grant = GRANT_DENIED; // 用于保存每个权限的授予情况！

            //【4.2】如果申请的权限设置了 PROTECTION_FLAG_APPOP 标志位，
            // 我们会将其添加到 mAppOpPermissionPackages 进行监控！
            if ((bp.protectionLevel & PermissionInfo.PROTECTION_FLAG_APPOP) != 0) {
                ArraySet<String> pkgs = mAppOpPermissionPackages.get(bp.name);
                if (pkgs == null) {
                    pkgs = new ArraySet<>();
                    mAppOpPermissionPackages.put(bp.name, pkgs);
                }
                pkgs.add(pkg.packageName);
            }

            //【4.3】计算权限的基本类型，并判断应用是否支持运行时权限！！！
            final int level = bp.protectionLevel & PermissionInfo.PROTECTION_MASK_BASE;
            final boolean appSupportsRuntimePermissions = pkg.applicationInfo.targetSdkVersion
                    >= Build.VERSION_CODES.M;

            //【4.4】根据权限的基本类型，判断权限的授予情况！！
            switch (level) {
                case PermissionInfo.PROTECTION_NORMAL: { 
                    //【4.4.1】基本类型为 normal，为安装时权限，默认授予！
                    grant = GRANT_INSTALL;
                } break;

                case PermissionInfo.PROTECTION_DANGEROUS: {
                    //【4.4.2】基本类型为 dangerous，那么我们要分不同的情况！！
                    if (!appSupportsRuntimePermissions && !Build.PERMISSIONS_REVIEW_REQUIRED) {
                        //【4.4.2.1】如果 legacy apps，同时也不需要在组件启动之前重新 review 权限
                        // 说明应用的 targetSDKVersion < 23 是 legacy apps，那么运行时权限视为安装时权限；
                        grant = GRANT_INSTALL;
                        
                    } else if (origPermissions.hasInstallPermission(bp.name)) {
                        //【4.4.2.2】如果应用支持安装是权限，或者需要在组件启动之前重新 review 权限
                        // 并且该运行时权限之前是属于安装时权限，那么就要升级为运行时权限（系统升级会出现）
                        grant = GRANT_UPGRADE;
                        
                    } else if (mPromoteSystemApps
                            && isSystemApp(ps)
                            && mExistingSystemPackages.contains(ps.name)) {
                        //【4.4.2.3】如果是从 API 22 升级上来的话，对于 Promote System Apps，
                        // 那么安装时权限就要升级为运行时权限；
                        grant = GRANT_UPGRADE;
                        
                    } else {
                        //【4.4.2.4】其他情况，直接视为安装时权限！
                        grant = GRANT_RUNTIME;
                        
                    }
                } break;

                case PermissionInfo.PROTECTION_SIGNATURE: {
                    //【4.4.3】基本类型为 signature 的权限，需要校验签名！！
                    //【*1.2.2-important】校验签名是否匹配，如果是的话，是为安装时权限！
                    allowedSig = grantSignaturePermission(perm, pkg, bp, origPermissions);
                    if (allowedSig) {
                        // 如果签名校验通过，直接视为安装时权限！
                        grant = GRANT_INSTALL;
                    }
                } break;
            }

            if (DEBUG_INSTALL) {
                Log.i(TAG, "Package " + pkg.packageName + " granting " + perm);
            }

            //【4.5】处理授予情况，如果授予情况不是拒绝 GRANT_DENIED，进入 IF 分支；
            // 如果授予情况是拒绝 GRANT_DENIED，进入 ELSE 分支！
            if (grant != GRANT_DENIED) {
                if (!isSystemApp(ps) && ps.installPermissionsFixed) {
                    //【4.5.1】如果这是一个已经被安装了的非系统的应用，同时本次申请的权限之前不属于安装时权限
                    // 除非该权限属于 new platform permission 不然的话，我们是不会授予他这个权限的；
                    if (!allowedSig && !origPermissions.hasInstallPermission(perm)) {
                        if (!isNewPlatformPermissionForPackage(perm, pkg)) {
                            grant = GRANT_DENIED;
                        }
                    }
                }
                //【4.5.2】处理权限的授予状态！
                switch (grant) {
                    case GRANT_INSTALL: { // 处理安装时权限；
                        //【4.5.2.1.1】首先，判断该权限之前是否是运行时权限，如果是的话，那就要撤销之前的授予！
                        for (int userId : UserManagerService.getInstance().getUserIds()) {
                            if (origPermissions.getRuntimePermissionState(
                                    bp.name, userId) != null) {
                                // 撤销权限，并取消所有的标志位；
                                origPermissions.revokeRuntimePermission(bp, userId);
                                origPermissions.updatePermissionFlags(bp, userId,
                                      PackageManager.MASK_PERMISSION_FLAGS, 0);
                                // 将运行时权限授予状态发生变化的 userId 记录下来，后面用于持久化数据！
                                changedRuntimePermissionUserIds = ArrayUtils.appendInt(
                                        changedRuntimePermissionUserIds, userId);
                            }
                        }
                        //【4.5.2.1.2】授予安装时权限；
                        if (permissionsState.grantInstallPermission(bp) !=
                                PermissionsState.PERMISSION_OPERATION_FAILURE) {
                            changedInstallPermission = true;
                        }
                    } break;

                    case GRANT_RUNTIME: { // 处理运行时权限；
                        //【4.5.2.1.3】授予之前被授予的运行时权限；
                        for (int userId : UserManagerService.getInstance().getUserIds()) {
                            //【4.5.2.1.3.1】获得每个 userId 下的该运行时权限对象和其 flags；
                            PermissionState permissionState = origPermissions
                                    .getRuntimePermissionState(bp.name, userId);
                            int flags = permissionState != null
                                    ? permissionState.getFlags() : 0;
                            
                            //【4.5.2.1.3.2】如果本次授予的权限是已经申请的，那么我们就再次授予！
                            if (origPermissions.hasRuntimePermission(bp.name, userId)) {
                                //【4.5.2.1.3.2.1】授予运行时权限，如果失败的话，意味着本次无法授予，那就更新
                                // changedRuntimePermissionUserIds 列表；
                                if (permissionsState.grantRuntimePermission(bp, userId) ==
                                        PermissionsState.PERMISSION_OPERATION_FAILURE) {
                                    changedRuntimePermissionUserIds = ArrayUtils.appendInt(
                                            changedRuntimePermissionUserIds, userId);
                                }
                                //【4.5.2.1.3.2.2】如果应用支持运行时权限，该权限之前是需要 review 的
                                // 那么这里会取消 review 的标志位！
                                if (Build.PERMISSIONS_REVIEW_REQUIRED
                                        && appSupportsRuntimePermissions
                                        && (flags & PackageManager
                                                .FLAG_PERMISSION_REVIEW_REQUIRED) != 0) {
                                    flags &= ~PackageManager.FLAG_PERMISSION_REVIEW_REQUIRED;
                                    // 标志位变了，那就更新 changedRuntimePermissionUserIds 列表；
                                    changedRuntimePermissionUserIds = ArrayUtils.appendInt(
                                            changedRuntimePermissionUserIds, userId);
                                }
    
                            } else if (Build.PERMISSIONS_REVIEW_REQUIRED
                                    && !appSupportsRuntimePermissions) {
                                //【4.5.2.1.3.2】如果应用不支持运行时权限，但是系统需要权限 review
                                // 每一个被授予的新的运行时权限，都要被 review！
                                // 如果是系统权限，强制增加 FLAG_PERMISSION_REVIEW_REQUIRED 标志位
                                if (PLATFORM_PACKAGE_NAME.equals(bp.sourcePackage)) {
                                    if ((flags & FLAG_PERMISSION_REVIEW_REQUIRED) == 0) {
                                        flags |= FLAG_PERMISSION_REVIEW_REQUIRED;
                                        // 标志位变了，那就更新 changedRuntimePermissionUserIds 列表；
                                        changedRuntimePermissionUserIds = ArrayUtils.appendInt(
                                                changedRuntimePermissionUserIds, userId);
                                    }
                                }
                                // 授予这个新的运行时权限，如果授予失败，更新 changedRuntimePermissionUserIds 列表；
                                if (permissionsState.grantRuntimePermission(bp, userId)
                                        != PermissionsState.PERMISSION_OPERATION_FAILURE) {
                                    changedRuntimePermissionUserIds = ArrayUtils.appendInt(
                                            changedRuntimePermissionUserIds, userId);
                                }
                            }
                            //【4.5.2.1.3.2.1】更新权限的 flags！
                            permissionsState.updatePermissionFlags(bp, userId, flags, flags);

                        }
                    } break;

                    case GRANT_UPGRADE: { // 处理安装时权限升级为运行时权限
                        //【4.5.2.1.4】授予安装时升级为运行时的权限；
                        // 获得权限管理对象和 flags；
                        PermissionState permissionState = origPermissions
                                .getInstallPermissionState(bp.name);
                        final int flags = permissionState != null ? permissionState.getFlags() : 0;

                        //【4.5.2.1.4.1】撤销之前的安装时权限，并清空 flags；
                        if (origPermissions.revokeInstallPermission(bp)
                                != PermissionsState.PERMISSION_OPERATION_FAILURE) {
                            origPermissions.updatePermissionFlags(bp, UserHandle.USER_ALL,
                                    PackageManager.MASK_PERMISSION_FLAGS, 0);
                            changedInstallPermission = true;
                        }

                        //【4.5.2.1.4.2】如果该权限没有设置 FLAG_PERMISSION_REVOKE_ON_UPGRADE，
                        // 表示在升级后可以授予，那么我们就授予运行时权限；
                        if ((flags & PackageManager.FLAG_PERMISSION_REVOKE_ON_UPGRADE) == 0) {
                            for (int userId : currentUserIds) {
                                if (permissionsState.grantRuntimePermission(bp, userId) !=
                                        PermissionsState.PERMISSION_OPERATION_FAILURE) {
                                    // 将旧的 flags 更新到运行时权限；
                                    permissionsState.updatePermissionFlags(bp, userId,
                                            flags, flags);
                                    // 标志位变了，那就更新 changedRuntimePermissionUserIds 列表；
                                    changedRuntimePermissionUserIds = ArrayUtils.appendInt(
                                            changedRuntimePermissionUserIds, userId);
                                }
                            }
                        }

                    } break;

                    default: {
                        if (packageOfInterest == null
                                || packageOfInterest.equals(pkg.packageName)) {
                            Slog.w(TAG, "Not granting permission " + perm
                                    + " to package " + pkg.packageName
                                    + " because it was previously installed without");
                        }
                    } break;
                }
            } else {
                //【4.6】处理撤销情况，撤销该权限，同时清空所有的标志位；
                if (permissionsState.revokeInstallPermission(bp) !=
                        PermissionsState.PERMISSION_OPERATION_FAILURE) {
                    // 清空标志位；
                    permissionsState.updatePermissionFlags(bp, UserHandle.USER_ALL,
                            PackageManager.MASK_PERMISSION_FLAGS, 0);
                    changedInstallPermission = true;
                    Slog.i(TAG, "Un-granting permission " + perm
                            + " from package " + pkg.packageName
                            + " (protectionLevel=" + bp.protectionLevel
                            + " flags=0x" + Integer.toHexString(pkg.applicationInfo.flags)
                            + ")");
                } else if ((bp.protectionLevel&PermissionInfo.PROTECTION_FLAG_APPOP) == 0) {
                    //【4.6.1】如果撤销失败，但是该权限和一个 appOp 相关联，这种情况可以不处理。因为
                    // 对于这类权限，我们不处理，因为我们会通过 appOps 进行控制，通过 ui 给用户选择；
                    if (packageOfInterest == null || packageOfInterest.equals(pkg.packageName)) {
                        Slog.w(TAG, "Not granting permission " + perm
                                + " to package " + pkg.packageName
                                + " (protectionLevel=" + bp.protectionLevel
                                + " flags=0x" + Integer.toHexString(pkg.applicationInfo.flags)
                                + ")");
                    }
                }
            }
        }
        if ((changedInstallPermission || replace) && !ps.installPermissionsFixed &&
                !isSystemApp(ps) || isUpdatedSystemApp(ps)){
            // 这是我们第一次听说过这个包，所以我们现在选择的权限是固定的，直到明确更改为止。
            // （没看懂这段逻辑...）
            ps.installPermissionsFixed = true;
        }

        //【5】持久化变化调整后的运行时权限数据！
        for (int userId : changedRuntimePermissionUserIds) {
            mSettings.writeRuntimePermissionsForUserLPr(userId, runtimePermissionsRevoked);
        }

        Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
    }
```
### 1.2.1 PackageMS.revokeUnusedSharedUserPermissionsLPw

撤销不再使用的共享用户权限；

参数 SharedUserSetting su 表示的是共享 uid 的信息对象！
```java
    private int[] revokeUnusedSharedUserPermissionsLPw(SharedUserSetting su, int[] allUserIds) {
        //【1】收集该 shared user 下的所有被请求的权限！
        ArraySet<String> usedPermissions = new ArraySet<>();
        final int packageCount = su.packages.size();
        for (int i = 0; i < packageCount; i++) {
            PackageSetting ps = su.packages.valueAt(i);
            if (ps.pkg == null) {
                continue;
            }
            final int requestedPermCount = ps.pkg.requestedPermissions.size();
            for (int j = 0; j < requestedPermCount; j++) {
                String permission = ps.pkg.requestedPermissions.get(j);
                BasePermission bp = mSettings.mPermissions.get(permission);
                if (bp != null) {
                    usedPermissions.add(permission);
                }
            }
        }
        //【2】获得该 shared user 的权限管理对象！!
        PermissionsState permissionsState = su.getPermissionsState();

        //【3】撤销那些本次不再申请，但是之前申请了的安装时权限！!
        List<PermissionState> installPermStates = permissionsState.getInstallPermissionStates();
        final int installPermCount = installPermStates.size();
        for (int i = installPermCount - 1; i >= 0;  i--) {
            PermissionState permissionState = installPermStates.get(i);
            if (!usedPermissions.contains(permissionState.getName())) {
                BasePermission bp = mSettings.mPermissions.get(permissionState.getName());
                if (bp != null) {
                    //【3.1】撤销权限，移除所有标志位；
                    permissionsState.revokeInstallPermission(bp);
                    permissionsState.updatePermissionFlags(bp, UserHandle.USER_ALL,
                            PackageManager.MASK_PERMISSION_FLAGS, 0);
                }
            }
        }

        int[] runtimePermissionChangedUserIds = EmptyArray.INT;

        //【4】撤销那些本次不再申请，但是之前申请了的运行时权限！!
        for (int userId : allUserIds) {
            List<PermissionState> runtimePermStates = permissionsState
                    .getRuntimePermissionStates(userId);
            final int runtimePermCount = runtimePermStates.size();
            for (int i = runtimePermCount - 1; i >= 0; i--) {
                PermissionState permissionState = runtimePermStates.get(i);
                if (!usedPermissions.contains(permissionState.getName())) {
                    BasePermission bp = mSettings.mPermissions.get(permissionState.getName());
                    if (bp != null) {
                        //【4.1】撤销权限，移除所有标志位；
                        permissionsState.revokeRuntimePermission(bp, userId);
                        permissionsState.updatePermissionFlags(bp, userId,
                                PackageManager.MASK_PERMISSION_FLAGS, 0);
                        //【4.2】返回该运行时权限所在的 userId；
                        runtimePermissionChangedUserIds = ArrayUtils.appendInt(
                                runtimePermissionChangedUserIds, userId);
                    }
                }
            }
        }
        //【5】返回该运行时权限所在的 userId；
        return runtimePermissionChangedUserIds;
    }
```
runtimePermissionChangedUserIds 用于持久化运行时数据库；

###1.2.2 PackageMS.grantSignaturePermission

grantSignaturePermission 方法用于判断签名是否能校验通过，我们在这里先忽略掉签名校验的过程，假设要么校验成功，要么校验失败！

```java
    private boolean grantSignaturePermission(String perm, PackageParser.Package pkg,
            BasePermission bp, PermissionsState origPermissions) {
        boolean allowed;
        //【1】判断应用的签名和权限定义者签名或者平台签名是否匹配！
        allowed = (compareSignatures(
                bp.packageSetting.signatures.mSignatures, pkg.mSignatures)
                        == PackageManager.SIGNATURE_MATCH)
                || (compareSignatures(mPlatformPackage.mSignatures, pkg.mSignatures)
                        == PackageManager.SIGNATURE_MATCH);
                        
        //【2】如果和权限定义者签名或者平台签名不匹配，并且该权限是 PRIVILEGED 的，进行进一步的判断！
        if (!allowed && (bp.protectionLevel
                & PermissionInfo.PROTECTION_FLAG_PRIVILEGED) != 0) {
            
            if (isSystemApp(pkg)) { // 对于 PRIVILEGED 类型的权限，首先必须是系统应用！
                //【2.1】如果该应用是一个被更新过的系统应用，那么只有更新前的旧 apk 被授予了这个权限，
                // 新的 apk 才能被授予这个权限！
                if (pkg.isUpdatedSystemApp()) {
                    // 获得更新前的安装数据(system)！
                    final PackageSetting sysPs = mSettings
                            .getDisabledSystemPkgLPr(pkg.packageName);

                    if (sysPs != null && sysPs.getPermissionsState().hasInstallPermission(perm)) {
                        //【2.1.1】如果更新前，其已经被授予了该权限，并且其是一个特权应用，
                        // 那么本次也允许授予！！
                        if (sysPs.isPrivileged()) {
                            allowed = true;
                        }
                    } else {
                        //【2.1.2】如果更新前，并没有被授予该权限，该权限是更新后新申请的权限！
                        // 那就要看之前是否有请求该权限，只有请求了该权限，才允许授予；
                        if (sysPs != null && sysPs.pkg != null && sysPs.isPrivileged()) {
                            for (int j = 0; j < sysPs.pkg.requestedPermissions.size(); j++) {
                                if (perm.equals(sysPs.pkg.requestedPermissions.get(j))) {
                                    allowed = true;
                                    break;
                                }
                            }
                        }
    
                        // 此外，如果系统映像上的特权父包或其父包任何子代请求特权权限，则更新的子包也可以获得该许可。
                        if (pkg.parentPackage != null) {
                            final PackageSetting disabledSysParentPs = mSettings
                                    .getDisabledSystemPkgLPr(pkg.parentPackage.packageName);
                            if (disabledSysParentPs != null && disabledSysParentPs.pkg != null
                                    && disabledSysParentPs.isPrivileged()) {
                                if (isPackageRequestingPermission(disabledSysParentPs.pkg, perm)) {
                                    allowed = true;
                                } else if (disabledSysParentPs.pkg.childPackages != null) {
                                    final int count = disabledSysParentPs.pkg.childPackages.size();
                                    for (int i = 0; i < count; i++) {
                                        PackageParser.Package disabledSysChildPkg =
                                                disabledSysParentPs.pkg.childPackages.get(i);
                                        if (isPackageRequestingPermission(disabledSysChildPkg,
                                                perm)) {
                                            allowed = true;
                                            break;
                                        }
                                    }
                                }
                            }
                        }
                    }
                } else {
                    //【2.2】如果该应用是一个没有被更新过的系统应用，那就要判读其是否是特权应用！
                    allowed = isPrivilegedApp(pkg);
                }
            }
        }
        
        //【3】经过了上面的判断，依然不允许，接着进行下一步的判断，下面会根据额外的标志位继续判断！
        if (!allowed) {
            if (!allowed && (bp.protectionLevel
                    & PermissionInfo.PROTECTION_FLAG_PRE23) != 0
                    && pkg.applicationInfo.targetSdkVersion < Build.VERSION_CODES.M) {
                //【3.1】PROTECTION_FLAG_PRE23 指定此权限会自动授予低于 M（在引入运行时权限之前）的 API 级别的应用程序。
                // 如果应用的 targetSdkVersion 是小于 M 的，那么我们授予他签名权限！
                allowed = true;
            }

            if (!allowed && (bp.protectionLevel & PermissionInfo.PROTECTION_FLAG_INSTALLER) != 0
                    && pkg.packageName.equals(mRequiredInstallerPackage)) {
                //【3.2】PROTECTION_FLAG_INSTALLER 指定此权限可以自动授予包软件应用程序。
                // 而此时的 apk 就是 packageInstaller，那么授予他签名权限！
                allowed = true;
            }

            if (!allowed && (bp.protectionLevel & PermissionInfo.PROTECTION_FLAG_VERIFIER) != 0
                    && pkg.packageName.equals(mRequiredVerifierPackage)) {
                //【3.3】PROTECTION_FLAG_INSTALLER 指定此权限可以被自动授予给验证软件包的系统应用程序。
                // 而此时的 apk 就是 VerifierPackage，那么授予他签名权限！
                allowed = true;
            }

            if (!allowed && (bp.protectionLevel
                    & PermissionInfo.PROTECTION_FLAG_PREINSTALLED) != 0
                    && isSystemApp(pkg)) {
                //【3.4】PROTECTION_FLAG_PREINSTALLED 指定此权限可以自动授予 system 分区预先安装的任何应用程序
                // 而此时的 apk 就是 system app，那么授予他签名权限！
                allowed = true;
            }

            if (!allowed && (bp.protectionLevel
                    & PermissionInfo.PROTECTION_FLAG_DEVELOPMENT) != 0) {
                //【3.4】PROTECTION_FLAG_DEVELOPMENT 指定此权限可以授予（可选）给开发应用程序
                // 而上一次应用是被授予了这个权限，那么一次也不被授予
                allowed = origPermissions.hasInstallPermission(perm);
            }

            if (!allowed && (bp.protectionLevel & PermissionInfo.PROTECTION_FLAG_SETUP) != 0
                    && pkg.packageName.equals(mSetupWizardPackage)) {
                //【3.5】PROTECTION_FLAG_SETUP 指定此权限可以被自动授予给开机向导应用程序
                // 而此时的 apk 就是开机向导，那么授予他签名权限！.
                allowed = true;
            }
        }
        return allowed;
    }
```
可以看到，这里对 Signature 类型的权限做了详细的处理！

# 2 Settings.applyDefaultPreferredAppsLPw - 默认应用设置

该方法用于设置默认应用！

```java
    void applyDefaultPreferredAppsLPw(PackageManagerService service, int userId) {
        //【1】从之前安装过应用中设置默认应用；
        for (PackageSetting ps : mPackages.values()) {
            if ((ps.pkgFlags&ApplicationInfo.FLAG_SYSTEM) != 0 && ps.pkg != null
                    && ps.pkg.preferredActivityFilters != null) {
                ArrayList<PackageParser.ActivityIntentInfo> intents
                        = ps.pkg.preferredActivityFilters;
                for (int i=0; i<intents.size(); i++) {
                    PackageParser.ActivityIntentInfo aii = intents.get(i);
                    applyDefaultPreferredActivityLPw(service, aii, new ComponentName(
                            ps.name, aii.activity.className), userId);
                }
            }
        }

        //【2】从 /etc/preferred-apps 目录下读取默认应用的配置！
        File preferredDir = new File(Environment.getRootDirectory(), "etc/preferred-apps");
        if (!preferredDir.exists() || !preferredDir.isDirectory()) {
            return;
        }
        if (!preferredDir.canRead()) {
            Slog.w(TAG, "Directory " + preferredDir + " cannot be read");
            return;
        }

        // Iterate over the files in the directory and scan .xml files
        for (File f : preferredDir.listFiles()) {
            if (!f.getPath().endsWith(".xml")) {
                Slog.i(TAG, "Non-xml file " + f + " in " + preferredDir + " directory, ignoring");
                continue;
            }
            if (!f.canRead()) {
                Slog.w(TAG, "Preferred apps file " + f + " cannot be read");
                continue;
            }

            if (PackageManagerService.DEBUG_PREFERRED) Log.d(TAG, "Reading default preferred " + f);
            InputStream str = null;
            try {
                str = new BufferedInputStream(new FileInputStream(f));
                XmlPullParser parser = Xml.newPullParser();
                parser.setInput(str, null);

                int type;
                while ((type = parser.next()) != XmlPullParser.START_TAG
                        && type != XmlPullParser.END_DOCUMENT) {
                    ;
                }

                if (type != XmlPullParser.START_TAG) {
                    Slog.w(TAG, "Preferred apps file " + f + " does not have start tag");
                    continue;
                }
                if (!"preferred-activities".equals(parser.getName())) {
                    Slog.w(TAG, "Preferred apps file " + f
                            + " does not start with 'preferred-activities'");
                    continue;
                }
                //【1】读取默认应用的配置信息！
                readDefaultPreferredActivitiesLPw(service, parser, userId);
            } catch (XmlPullParserException e) {
                Slog.w(TAG, "Error reading apps file " + f, e);
            } catch (IOException e) {
                Slog.w(TAG, "Error reading apps file " + f, e);
            } finally {
                if (str != null) {
                    try {
                        str.close();
                    } catch (IOException e) {
                    }
                }
            }
        }
    }
```


# 3 PMS.reconcileAppsDataLI - 准备数据目录

reconcileAppsDataLI 方法用于调整所有应用的数据，该方法会清楚那些被卸载或者重新安装到其他分区上的应用的已有数据，同时也会校验应用对应的数据目录是否存在，不存在的话，会创建对应的目录！

StorageManager.isFileEncryptedNativeOrEmulated() 方法用于判断系统是否运行在文件加密模式，如果是文件加密模式的话，storageFlags 只有 StorageManager.FLAG_STORAGE_DE；

```java
            if (StorageManager.isFileEncryptedNativeOrEmulated()) {
                storageFlags = StorageManager.FLAG_STORAGE_DE;
            } else {
                storageFlags = StorageManager.FLAG_STORAGE_DE | StorageManager.FLAG_STORAGE_CE;
            }
```
下面我们来看看 reconcileAppsDataLI 方法：
```java
    private void reconcileAppsDataLI(String volumeUuid, int userId, int flags) {
        Slog.v(TAG, "reconcileAppsData for " + volumeUuid + " u" + userId + " 0x"
                + Integer.toHexString(flags));
        // cdDir 文件对应的目录是：/data/user/<userId>，在默认用户下是 /data/user/0
        // deDir 文件对应的目录是：/data/user_de/<userId>，在默认用户下是 /data/user_de/0
        final File ceDir = Environment.getDataUserCeDirectory(volumeUuid, userId);
        final File deDir = Environment.getDataUserDeDirectory(volumeUuid, userId);

        //【1】系统运行在文件加密模式下，是不会就进入这里的，正常情况会进入这里；
        if ((flags & StorageManager.FLAG_STORAGE_CE) != 0) {
            if (StorageManager.isFileEncryptedNativeOrEmulated()
                    && !StorageManager.isUserKeyUnlocked(userId)) { // 再次校验是否处于文件加密模式下！
                throw new RuntimeException(
                        "Yikes, someone asked us to reconcile CE storage while " + userId
                                + " was still locked; this would have caused massive data loss!");
            }
            //【1.1】遍历 /data/user/0 目录，处理该目录下的每个文件！
            final File[] files = FileUtils.listFilesOrEmpty(ceDir);
            for (File file : files) {
                final String packageName = file.getName();
                try {
                    //【1.2】校验所有已经安装的应用的数据
                    assertPackageKnownAndInstalled(volumeUuid, packageName, userId);
                } catch (PackageManagerException e) {
                    logCriticalInfo(Log.WARN, "Destroying " + file + " due to: " + e);
                    try {
                        // 通过 installd 删除该 package 的旧数据！
                        mInstaller.destroyAppData(volumeUuid, packageName, userId,
                                StorageManager.FLAG_STORAGE_CE, 0);
                    } catch (InstallerException e2) {
                        logCriticalInfo(Log.WARN, "Failed to destroy: " + e2);
                    }
                }
            }
        }
        //【2】遍历 /data/user_de/0 目录，处理该目录下的每个文件！
        if ((flags & StorageManager.FLAG_STORAGE_DE) != 0) {
            final File[] files = FileUtils.listFilesOrEmpty(deDir);
            for (File file : files) {
                final String packageName = file.getName();
                try {
                    assertPackageKnownAndInstalled(volumeUuid, packageName, userId);
                } catch (PackageManagerException e) {
                    logCriticalInfo(Log.WARN, "Destroying " + file + " due to: " + e);
                    try {
                        // 通过 installd 删除该 package 的旧数据！
                        mInstaller.destroyAppData(volumeUuid, packageName, userId,
                                StorageManager.FLAG_STORAGE_DE, 0);
                    } catch (InstallerException e2) {
                        logCriticalInfo(Log.WARN, "Failed to destroy: " + e2);
                    }
                }
            }
        }

        //【3】确定系统中所有应用都有对应的 data 目录！
        final List<PackageSetting> packages;
        synchronized (mPackages) {
            packages = mSettings.getVolumePackagesLPr(volumeUuid);
        }
        int preparedCount = 0;
        for (PackageSetting ps : packages) {
            final String packageName = ps.name;
            if (ps.pkg == null) {
                Slog.w(TAG, "Odd, missing scanned package " + packageName);
                continue;
            }
            //【3.1】如果 package 已经安装了，那么就会调用 prepareAppDataLIF 方法，准备数据目录！
            if (ps.getInstalled(userId)) {
                prepareAppDataLIF(ps.pkg, userId, flags);

                if (maybeMigrateAppDataLIF(ps.pkg, userId)) {
                    prepareAppDataLIF(ps.pkg, userId, flags);
                }
                preparedCount++;
            }
        }
        Slog.v(TAG, "reconcileAppsData finished " + preparedCount + " packages");
    }
```
这里想说下 /data/user/0 和 /data/user_de/0，其实它们是 /data/data 目录的 link 名称，本质上指向了一个目录！

这个过程如下：

- 校验所有已经安装的应用，删除那些异常情况下的应用的数据，如果应用改了包名等等的情况，删除之前的数据；
- 为没有数据目录的应用创建目录；

这个过程首先处理了 rename package 的情况，对于 rename package 的情况，重命名之前的数据是不可用的，所以这里会使用 assertPackageKnownAndInstalled 方法来进行处理！


```java
    private void assertPackageKnownAndInstalled(String volumeUuid, String packageName, int userId)
            throws PackageManagerException {
        synchronized (mPackages) {
            // 如果重命名过，会返回 oldName，不然则是当前的包名！
            packageName = normalizePackageNameLPr(packageName);
            // 尝试在上次安装信息中，查找这个 package 对应的安装信息；
            final PackageSetting ps = mSettings.mPackages.get(packageName);
            if (ps == null) {
                throw new PackageManagerException("Package " + packageName + " is unknown");
            } else if (!TextUtils.equals(volumeUuid, ps.volumeUuid)) {
                throw new PackageManagerException(
                        "Package " + packageName + " found on unknown volume " + volumeUuid
                                + "; expected volume " + ps.volumeUuid);
            } else if (!ps.getInstalled(userId)) {
                throw new PackageManagerException(
                        "Package " + packageName + " not installed for user " + userId);
            }
        }
    }
```

如果查不到安装数据，或者 volumeUuid 不匹配，或者该应用在当前设备用户 id 下不存在了，那就会抛出异常，然后 pms 会删除掉之前的数据！

normalizePackageNameLPr 会根据传入的 packageName，在 mRenamedPackages 找到重命名前的 oldName，如果该 package 之前重命名过，那么其一定会有 old name；

```java
    private String normalizePackageNameLPr(String packageName) {
        String normalizedPackageName = mSettings.mRenamedPackages.get(packageName);
        return normalizedPackageName != null ? normalizedPackageName : packageName;
    }
```
方法逻辑很简单！

## 3.1 PMS.prepareAppDataLIF

prepareAppDataLIF 方法为应用准备数据目录！

```java
    private void prepareAppDataLIF(PackageParser.Package pkg, int userId, int flags) {
        if (pkg == null) {
            Slog.wtf(TAG, "Package was null!", new Throwable());
            return;
        }

        //【3.2】为该应用准备数据目录！
        prepareAppDataLeafLIF(pkg, userId, flags);
        // 为 childPackage 创建数据目录！
        final int childCount = (pkg.childPackages != null) ? pkg.childPackages.size() : 0;
        for (int i = 0; i < childCount; i++) {
            prepareAppDataLeafLIF(pkg.childPackages.get(i), userId, flags);
        }
    }

```
## 3.2 PMS.prepareAppDataLeafLIF

prepareAppDataLeafLIF 方法会继续处理数据目录操作：

```java
    private void prepareAppDataLeafLIF(PackageParser.Package pkg, int userId, int flags) {
        if (DEBUG_APP_DATA) {
            Slog.v(TAG, "prepareAppData for " + pkg.packageName + " u" + userId + " 0x"
                    + Integer.toHexString(flags));
        }

        final String volumeUuid = pkg.volumeUuid;
        final String packageName = pkg.packageName;
        final ApplicationInfo app = pkg.applicationInfo;
        final int appId = UserHandle.getAppId(app.uid);

        Preconditions.checkNotNull(app.seinfo);

        try {
            // 通过 Installd 来创建应用的数据目录；
            mInstaller.createAppData(volumeUuid, packageName, userId, flags,
                    appId, app.seinfo, app.targetSdkVersion);
        } catch (InstallerException e) {
            // 如果创建失败，那么如果该应用是系统 apk，pms 会尝试重新创建一次，如果再次失败，那就不会处理！
            if (app.isSystemApp()) {
                logCriticalInfo(Log.ERROR, "Failed to create app data for " + packageName
                        + ", but trying to recover: " + e);
                destroyAppDataLeafLIF(pkg, userId, flags);
                try {
                    mInstaller.createAppData(volumeUuid, packageName, userId, flags,
                            appId, app.seinfo, app.targetSdkVersion);
                    logCriticalInfo(Log.DEBUG, "Recovery succeeded!");
                } catch (InstallerException e2) {
                    logCriticalInfo(Log.DEBUG, "Recovery failed!");
                }
            } else {
                Slog.e(TAG, "Failed to create app data for " + packageName + ": " + e);
            }
        }

        // 系统运行在文件加密模式下，不会进入这里！
        if ((flags & StorageManager.FLAG_STORAGE_CE) != 0) {
            try {
                // CE storage is unlocked right now, so read out the inode and
                // remember for use later when it's locked
                // TODO: mark this structure as dirty so we persist it!
                final long ceDataInode = mInstaller.getAppDataInode(volumeUuid, packageName, userId,
                        StorageManager.FLAG_STORAGE_CE);
                synchronized (mPackages) {
                    final PackageSetting ps = mSettings.mPackages.get(packageName);
                    if (ps != null) {
                        ps.setCeDataInode(ceDataInode, userId);
                    }
                }
            } catch (InstallerException e) {
                Slog.e(TAG, "Failed to find inode for " + packageName + ": " + e);
            }
        }
        //【3.3】继续调用 prepareAppDataContentsLeafLIF 进入下一步；
        prepareAppDataContentsLeafLIF(pkg, userId, flags);
    }
```

## 3.3 PMS.prepareAppDataContentsLeafLIF

```java
    private void prepareAppDataContentsLeafLIF(PackageParser.Package pkg, int userId, int flags) {
        final String volumeUuid = pkg.volumeUuid;
        final String packageName = pkg.packageName;
        final ApplicationInfo app = pkg.applicationInfo;
 
        // 系统运行在文件加密模式下，不会进入这里！
        if ((flags & StorageManager.FLAG_STORAGE_CE) != 0) {
            // 如果应用使用的是 32 位架构的话，为 23 位库建立本地库符号链接；
            if (app.primaryCpuAbi != null && !VMRuntime.is64BitAbi(app.primaryCpuAbi)) {
                final String nativeLibPath = app.nativeLibraryDir;
                try {
                    // 同样的，通过 installd 来建立 
                    mInstaller.linkNativeLibraryDirectory(volumeUuid, packageName,
                            nativeLibPath, userId);
                } catch (InstallerException e) {
                    Slog.e(TAG, "Failed to link native for " + packageName + ": " + e);
                }
            }
        }
    }
```

可以看到，PMS 对于应用的数据目录的一系列操作，都是通过 installd 来实现的，因为 pms 本身没有 selinux 权限，所以他必须将权限交给一个特权进程，也就是 installd 来做！


# 4 PackageManagerService.clearAppDataLIF

清楚缓存数据的前提是：ps.volumeUuid == StorageManager.UUID_PRIVATE_INTERNAL

参数传递：flags 为  StorageManager.FLAG_STORAGE_DE | StorageManager.FLAG_STORAGE_CE | Installer.FLAG_CLEAR_CODE_CACHE_ONLY
```java
    private void clearAppDataLIF(PackageParser.Package pkg, int userId, int flags) {
        if (pkg == null) {
            Slog.wtf(TAG, "Package was null!", new Throwable());
            return;
        }
        clearAppDataLeafLIF(pkg, userId, flags);
        final int childCount = (pkg.childPackages != null) ? pkg.childPackages.size() : 0;
        for (int i = 0; i < childCount; i++) {
            clearAppDataLeafLIF(pkg.childPackages.get(i), userId, flags);
        }
    }
```
继续调用：
```java
    private void clearAppDataLeafLIF(PackageParser.Package pkg, int userId, int flags) {
        final PackageSetting ps;
        synchronized (mPackages) {
            ps = mSettings.mPackages.get(pkg.packageName);
        }
        for (int realUserId : resolveUserIds(userId)) {
            final long ceDataInode = (ps != null) ? ps.getCeDataInode(realUserId) : 0;
            try {
                
                mInstaller.clearAppData(pkg.volumeUuid, pkg.packageName, realUserId, flags,
                        ceDataInode);
            } catch (InstallerException e) {
                Slog.w(TAG, String.valueOf(e));
            }
        }
    }
```
仍然是调用 mInstaller 来对用户的应用数据进行操作！

# 5 Settings.writeLPr - 持久化最新数据

将解析到的安装数据写回到 packages.xml 等文件中！
```java
    void writeLPr() {
        Debug.startMethodTracing("/data/system/packageprof", 8 * 1024 * 1024);
        //【1】下面会先将最新的数据写入到 packages-backup.xml 文件中！
        if (mSettingsFilename.exists()) {
            if (!mBackupSettingsFilename.exists()) {
                if (!mSettingsFilename.renameTo(mBackupSettingsFilename)) {
                    Slog.wtf(PackageManagerService.TAG,
                            "Unable to backup package manager settings, "
                            + " current changes will be lost at reboot");
                    return;
                }
            } else {
                mSettingsFilename.delete();
                Slog.w(PackageManagerService.TAG, "Preserving older settings backup");
            }
        }

        mPastSignatures.clear();

        try {
            FileOutputStream fstr = new FileOutputStream(mSettingsFilename);
            BufferedOutputStream str = new BufferedOutputStream(fstr);

            //XmlSerializer serializer = XmlUtils.serializerInstance();
            XmlSerializer serializer = new FastXmlSerializer();
            serializer.setOutput(str, StandardCharsets.UTF_8.name());
            serializer.startDocument(null, true);
            serializer.setFeature("http://xmlpull.org/v1/doc/features.html#indent-output", true);

            serializer.startTag(null, "packages"); //【2】写入 "packages"！

            for (int i = 0; i < mVersion.size(); i++) { //【3】写入 "version"
                final String volumeUuid = mVersion.keyAt(i);
                final VersionInfo ver = mVersion.valueAt(i);

                serializer.startTag(null, TAG_VERSION);
                XmlUtils.writeStringAttribute(serializer, ATTR_VOLUME_UUID, volumeUuid);
                XmlUtils.writeIntAttribute(serializer, ATTR_SDK_VERSION, ver.sdkVersion);
                XmlUtils.writeIntAttribute(serializer, ATTR_DATABASE_VERSION, ver.databaseVersion);
                XmlUtils.writeStringAttribute(serializer, ATTR_FINGERPRINT, ver.fingerprint);
                serializer.endTag(null, TAG_VERSION);
            }

            if (mVerifierDeviceIdentity != null) { //【4】写入 "verifier"
                serializer.startTag(null, "verifier");
                serializer.attribute(null, "device", mVerifierDeviceIdentity.toString());
                serializer.endTag(null, "verifier");
            }

            if (mReadExternalStorageEnforced != null) {
                serializer.startTag(null, TAG_READ_EXTERNAL_STORAGE);
                serializer.attribute(
                        null, ATTR_ENFORCEMENT, mReadExternalStorageEnforced ? "1" : "0");
                serializer.endTag(null, TAG_READ_EXTERNAL_STORAGE);
            }

            serializer.startTag(null, "permission-trees"); //【4.1】写入 "permission-trees"
            for (BasePermission bp : mPermissionTrees.values()) {
                writePermissionLPr(serializer, bp);
            }
            serializer.endTag(null, "permission-trees");

            serializer.startTag(null, "permissions"); //【4.1】写入 "permissions"
            for (BasePermission bp : mPermissions.values()) {
                writePermissionLPr(serializer, bp);
            }
            serializer.endTag(null, "permissions");

            for (final PackageSetting pkg : mPackages.values()) { // 写入 "package"
                writePackageLPr(serializer, pkg);
            }

            for (final PackageSetting pkg : mDisabledSysPackages.values()) { // 写入 "updated-package"
                writeDisabledSysPackageLPr(serializer, pkg);
            }

            for (final SharedUserSetting usr : mSharedUsers.values()) { // 写入 "shared-user"
                serializer.startTag(null, "shared-user");
                serializer.attribute(null, ATTR_NAME, usr.name);
                serializer.attribute(null, "userId",
                        Integer.toString(usr.userId));
                usr.signatures.writeXml(serializer, "sigs", mPastSignatures);
                // 写入 shared-user 的安装时权限！
                writePermissionsLPr(serializer, usr.getPermissionsState()
                        .getInstallPermissionStates());
                serializer.endTag(null, "shared-user");
            }

            if (mPackagesToBeCleaned.size() > 0) { // 写入 "cleaning-package"
                for (PackageCleanItem item : mPackagesToBeCleaned) {
                    final String userStr = Integer.toString(item.userId);
                    serializer.startTag(null, "cleaning-package");
                    serializer.attribute(null, ATTR_NAME, item.packageName);
                    serializer.attribute(null, ATTR_CODE, item.andCode ? "true" : "false");
                    serializer.attribute(null, ATTR_USER, userStr);
                    serializer.endTag(null, "cleaning-package");
                }
            }

            if (mRenamedPackages.size() > 0) { // 写入 "renamed-package"
                for (Map.Entry<String, String> e : mRenamedPackages.entrySet()) {
                    serializer.startTag(null, "renamed-package");
                    serializer.attribute(null, "new", e.getKey());
                    serializer.attribute(null, "old", e.getValue());
                    serializer.endTag(null, "renamed-package");
                }
            }

            final int numIVIs = mRestoredIntentFilterVerifications.size();
            if (numIVIs > 0) {
                if (DEBUG_DOMAIN_VERIFICATION) {
                    Slog.i(TAG, "Writing restored-ivi entries to packages.xml");
                }
                serializer.startTag(null, "restored-ivi");
                for (int i = 0; i < numIVIs; i++) {
                    IntentFilterVerificationInfo ivi = mRestoredIntentFilterVerifications.valueAt(i);
                    writeDomainVerificationsLPr(serializer, ivi);
                }
                serializer.endTag(null, "restored-ivi");
            } else {
                if (DEBUG_DOMAIN_VERIFICATION) {
                    Slog.i(TAG, "  no restored IVI entries to write");
                }
            }

            mKeySetManagerService.writeKeySetManagerServiceLPr(serializer);

            serializer.endTag(null, "packages"); // 结束 packages-backup.xml 文件的写入！

            serializer.endDocument();

            str.flush();
            FileUtils.sync(fstr);
            str.close();

            // New settings successfully written, old ones are no longer
            // needed.
            mBackupSettingsFilename.delete();
            FileUtils.setPermissions(mSettingsFilename.toString(),
                    FileUtils.S_IRUSR|FileUtils.S_IWUSR
                    |FileUtils.S_IRGRP|FileUtils.S_IWGRP,
                    -1, -1);

            writeKernelMappingLPr();
            writePackageListLPr(); //【4.4】写入 packageslist 文件！
            writeAllUsersPackageRestrictionsLPr(); //【4.5】写入偏好设置信息；
            writeAllRuntimePermissionsLPr(); //【4.6】写入运行时权限授予信息！
            return;

        } catch(XmlPullParserException e) {
            Slog.wtf(PackageManagerService.TAG, "Unable to write package manager settings, "
                    + "current changes will be lost at reboot", e);
        } catch(java.io.IOException e) {
            Slog.wtf(PackageManagerService.TAG, "Unable to write package manager settings, "
                    + "current changes will be lost at reboot", e);
        }
        // Clean up partially written files
        if (mSettingsFilename.exists()) {
            if (!mSettingsFilename.delete()) {
                Slog.wtf(PackageManagerService.TAG, "Failed to clean up mangled file: "
                        + mSettingsFilename);
            }
        }
        //Debug.stopMethodTracing();
    }
```

## 5.1 Settings.writePermissionLPr - 保存 "permission-trees" / "permission"
```java
    void writePermissionLPr(XmlSerializer serializer, BasePermission bp)
            throws XmlPullParserException, java.io.IOException {
        if (bp.sourcePackage != null) {
            serializer.startTag(null, TAG_ITEM); // item 标签
            serializer.attribute(null, ATTR_NAME, bp.name); // name 属性
            serializer.attribute(null, "package", bp.sourcePackage); // package 属性，表示定义者 package
            if (bp.protectionLevel != PermissionInfo.PROTECTION_NORMAL) { // protection 属性，权限级别！
                serializer.attribute(null, "protection", Integer.toString(bp.protectionLevel));
            }

            if (PackageManagerService.DEBUG_SETTINGS)
                Log.v(PackageManagerService.TAG, "Writing perm: name=" + bp.name + " type="
                        + bp.type);

            if (bp.type == BasePermission.TYPE_DYNAMIC) { // 如果是动态权限，这里是针对于权限树的操作！
                final PermissionInfo pi = bp.perm != null ? bp.perm.info : bp.pendingInfo;
                if (pi != null) {
                    serializer.attribute(null, "type", "dynamic");
                    if (pi.icon != 0) {
                        serializer.attribute(null, "icon", Integer.toString(pi.icon));
                    }
                    if (pi.nonLocalizedLabel != null) {
                        serializer.attribute(null, "label", pi.nonLocalizedLabel.toString());
                    }
                }
            }
            serializer.endTag(null, TAG_ITEM);
        }
    }
```
权限树以 `<permission-trees> </permission-trees> ` 开始和结束，权限树以 `<permissions> </permissions> ` 开始和结束！

## 5.2 Settings.writePackageLPr - 保存 "package"

```java

    void writePackageLPr(XmlSerializer serializer, final PackageSetting pkg)
            throws java.io.IOException {
        serializer.startTag(null, "package");
        serializer.attribute(null, ATTR_NAME, pkg.name); 
        if (pkg.realName != null) {
            serializer.attribute(null, "realName", pkg.realName);
        }
        serializer.attribute(null, "codePath", pkg.codePathString);
        if (!pkg.resourcePathString.equals(pkg.codePathString)) {
            serializer.attribute(null, "resourcePath", pkg.resourcePathString);
        }

        if (pkg.legacyNativeLibraryPathString != null) {
            serializer.attribute(null, "nativeLibraryPath", pkg.legacyNativeLibraryPathString);
        }
        if (pkg.primaryCpuAbiString != null) {
            serializer.attribute(null, "primaryCpuAbi", pkg.primaryCpuAbiString);
        }
        if (pkg.secondaryCpuAbiString != null) {
            serializer.attribute(null, "secondaryCpuAbi", pkg.secondaryCpuAbiString);
        }
        if (pkg.cpuAbiOverrideString != null) {
            serializer.attribute(null, "cpuAbiOverride", pkg.cpuAbiOverrideString);
        }

        serializer.attribute(null, "publicFlags", Integer.toString(pkg.pkgFlags));
        serializer.attribute(null, "privateFlags", Integer.toString(pkg.pkgPrivateFlags));
        serializer.attribute(null, "ft", Long.toHexString(pkg.timeStamp));
        serializer.attribute(null, "it", Long.toHexString(pkg.firstInstallTime));
        serializer.attribute(null, "ut", Long.toHexString(pkg.lastUpdateTime));
        serializer.attribute(null, "version", String.valueOf(pkg.versionCode));
        if (pkg.sharedUser == null) {
            serializer.attribute(null, "userId", Integer.toString(pkg.appId));
        } else {
            serializer.attribute(null, "sharedUserId", Integer.toString(pkg.appId));
        }
        if (pkg.uidError) {
            serializer.attribute(null, "uidError", "true");
        }
        if (pkg.installStatus == PackageSettingBase.PKG_INSTALL_INCOMPLETE) {
            serializer.attribute(null, "installStatus", "false");
        }
        if (pkg.installerPackageName != null) {
            serializer.attribute(null, "installer", pkg.installerPackageName);
        }
        if (pkg.isOrphaned) {
            serializer.attribute(null, "isOrphaned", "true");
        }
        if (pkg.volumeUuid != null) {
            serializer.attribute(null, "volumeUuid", pkg.volumeUuid);
        }
        if (pkg.parentPackageName != null) {
            serializer.attribute(null, "parentPackageName", pkg.parentPackageName);
        }

        writeChildPackagesLPw(serializer, pkg.childPackageNames);

        pkg.signatures.writeXml(serializer, "sigs", mPastSignatures);
        // 写入该 package 持有的安装时权限！
        writePermissionsLPr(serializer, pkg.getPermissionsState()
                    .getInstallPermissionStates());

        writeSigningKeySetLPr(serializer, pkg.keySetData);
        writeUpgradeKeySetsLPr(serializer, pkg.keySetData);
        writeKeySetAliasesLPr(serializer, pkg.keySetData);
        writeDomainVerificationsLPr(serializer, pkg.verificationInfo);

        serializer.endTag(null, "package");
    }
```
应用包信息以 `<package> </package> ` 开始和结束，不多说了！

## 5.3 Settings.writeDisabledSysPackageLPr - 保存 "updated-package"

```java

    void writeDisabledSysPackageLPr(XmlSerializer serializer, final PackageSetting pkg)
            throws java.io.IOException {
        serializer.startTag(null, "updated-package");
        serializer.attribute(null, ATTR_NAME, pkg.name);
        if (pkg.realName != null) {
            serializer.attribute(null, "realName", pkg.realName);
        }
        serializer.attribute(null, "codePath", pkg.codePathString);
        serializer.attribute(null, "ft", Long.toHexString(pkg.timeStamp));
        serializer.attribute(null, "it", Long.toHexString(pkg.firstInstallTime));
        serializer.attribute(null, "ut", Long.toHexString(pkg.lastUpdateTime));
        serializer.attribute(null, "version", String.valueOf(pkg.versionCode));
        if (!pkg.resourcePathString.equals(pkg.codePathString)) {
            serializer.attribute(null, "resourcePath", pkg.resourcePathString);
        }
        if (pkg.legacyNativeLibraryPathString != null) {
            serializer.attribute(null, "nativeLibraryPath", pkg.legacyNativeLibraryPathString);
        }
        if (pkg.primaryCpuAbiString != null) {
           serializer.attribute(null, "primaryCpuAbi", pkg.primaryCpuAbiString);
        }
        if (pkg.secondaryCpuAbiString != null) {
            serializer.attribute(null, "secondaryCpuAbi", pkg.secondaryCpuAbiString);
        }
        if (pkg.cpuAbiOverrideString != null) {
            serializer.attribute(null, "cpuAbiOverride", pkg.cpuAbiOverrideString);
        }

        if (pkg.sharedUser == null) {
            serializer.attribute(null, "userId", Integer.toString(pkg.appId));
        } else {
            serializer.attribute(null, "sharedUserId", Integer.toString(pkg.appId));
        }

        if (pkg.parentPackageName != null) {
            serializer.attribute(null, "parentPackageName", pkg.parentPackageName);
        }

        writeChildPackagesLPw(serializer, pkg.childPackageNames);

        // If this is a shared user, the permissions will be written there.
        if (pkg.sharedUser == null) {
            writePermissionsLPr(serializer, pkg.getPermissionsState()
                    .getInstallPermissionStates());
        }

        serializer.endTag(null, "updated-package");
    }
```
被更新过的应用的信息以 `<updated-package> </updated-package> ` 开始和结束，不多说了！

## 5.4 Settings.writePackageListLPr - 写入 "packages.list"

```java
    void writePackageListLPr() {
        writePackageListLPr(-1);
    }
```
最后，调用的是 writePackageListLPr 一参数方法：

```java
    void writePackageListLPr(int creatingUserId) {
        //【1】首先获得所有的设备用户信息！
        final List<UserInfo> users = UserManagerService.getInstance().getUsers(true);
        int[] userIds = new int[users.size()];
        for (int i = 0; i < userIds.length; i++) {
            userIds[i] = users.get(i).id;
        }
        if (creatingUserId != -1) { // 这里不进入！
            userIds = ArrayUtils.appendInt(userIds, creatingUserId);
        }

        // Write package list file now, use a JournaledFile.
        File tempFile = new File(mPackageListFilename.getAbsolutePath() + ".tmp");
        JournaledFile journal = new JournaledFile(mPackageListFilename, tempFile);

        final File writeTarget = journal.chooseForWrite();
        FileOutputStream fstr;
        BufferedWriter writer = null;
        try {
            fstr = new FileOutputStream(writeTarget);
            writer = new BufferedWriter(new OutputStreamWriter(fstr, Charset.defaultCharset()));
            FileUtils.setPermissions(fstr.getFD(), 0640, SYSTEM_UID, PACKAGE_INFO_GID);

            StringBuilder sb = new StringBuilder();
            //【2】遍历系统中所有被安装过的 package！
            for (final PackageSetting pkg : mPackages.values()) {
                //【2.1】跳过哪些异常的 package！
                if (pkg.pkg == null || pkg.pkg.applicationInfo == null
                        || pkg.pkg.applicationInfo.dataDir == null) {
                    if (!"android".equals(pkg.name)) {
                        Slog.w(TAG, "Skipping " + pkg + " due to missing metadata");
                    }
                    continue;
                }

                final ApplicationInfo ai = pkg.pkg.applicationInfo;
                final String dataPath = ai.dataDir;
                final boolean isDebug = (ai.flags & ApplicationInfo.FLAG_DEBUGGABLE) != 0;
                final int[] gids = pkg.getPermissionsState().computeGids(userIds);

                //【2.2】跳过哪些 dataPath 有空格的应用！
                if (dataPath.indexOf(' ') >= 0)
                    continue;

                // we store on each line the following information for now:
                //
                // pkgName    - package name
                // userId     - application-specific user id
                // debugFlag  - 0 or 1 if the package is debuggable.
                // dataPath   - path to package's data path
                // seinfo     - seinfo label for the app (assigned at install time)
                // gids       - supplementary gids this app launches with
                //
                // NOTE: We prefer not to expose all ApplicationInfo flags for now.
                //
                // DO NOT MODIFY THIS FORMAT UNLESS YOU CAN ALSO MODIFY ITS USERS
                // FROM NATIVE CODE. AT THE MOMENT, LOOK AT THE FOLLOWING SOURCES:
                //   frameworks/base/libs/packagelistparser
                //   system/core/run-as/run-as.c
                //
                //【2.3】根据指定的格式写入数据！
                sb.setLength(0);
                sb.append(ai.packageName);
                sb.append(" ");
                sb.append(ai.uid);
                sb.append(isDebug ? " 1 " : " 0 ");
                sb.append(dataPath);
                sb.append(" ");
                sb.append(ai.seinfo);
                sb.append(" ");
                if (gids != null && gids.length > 0) {
                    sb.append(gids[0]);
                    for (int i = 1; i < gids.length; i++) {
                        sb.append(",");
                        sb.append(gids[i]);
                    }
                } else {
                    sb.append("none");
                }
                sb.append("\n");
                writer.append(sb);
            }
            writer.flush();
            FileUtils.sync(fstr);
            writer.close();
            journal.commit();
        } catch (Exception e) {
            Slog.wtf(TAG, "Failed to write packages.list", e);
            IoUtils.closeQuietly(writer);
            journal.rollback();
        }
    }
```
我们可以看到，在 packages.list 文件中，存储的格式为：
```java
pkgName userId debugFlag dataPath  seinfo gids 
```
大家可以去看那个文件内容！！

## 5.5 Settings.writeAllUsersPackageRestrictionsLPr - 保存偏好设置

接下来，写入最新的偏好设置！
```java
    void writeAllUsersPackageRestrictionsLPr() {
        List<UserInfo> users = getAllUsers();
        if (users == null) return;

        for (UserInfo user : users) {
            writePackageRestrictionsLPr(user.id);
        }
    }
```
接着，调用 writePackageRestrictionsLPr 方法！这个方法我们前面分析过，这里不处理！


## 5.6 Settings.writeAllRuntimePermissionsLPr - 保存运行时权限授予信息

接下来，写入最新的运行时权限授予信息！
```java
    void writeAllRuntimePermissionsLPr() {
        for (int userId : UserManagerService.getInstance().getUserIds()) {
            //【5.6.1】对每一个设备用户，保存其运行时权限授予信息！
            mRuntimePermissionsPersistence.writePermissionsForUserAsyncLPr(userId);
        }
    }
```
### 5.6.1 RuntimePermissionsPersistence.writePermissionsForUserAsyncLPr 
```java
        public void writePermissionsForUserAsyncLPr(int userId) {
            final long currentTimeMillis = SystemClock.uptimeMillis();

            if (mWriteScheduled.get(userId)) {
                mHandler.removeMessages(userId);

                // If enough time passed, write without holding off anymore.
                final long lastNotWrittenMutationTimeMillis = mLastNotWrittenMutationTimesMillis
                        .get(userId);
                final long timeSinceLastNotWrittenMutationMillis = currentTimeMillis
                        - lastNotWrittenMutationTimeMillis;
                if (timeSinceLastNotWrittenMutationMillis >= MAX_WRITE_PERMISSIONS_DELAY_MILLIS) {
                    mHandler.obtainMessage(userId).sendToTarget();
                    return;
                }

                // Hold off a bit more as settings are frequently changing.
                final long maxDelayMillis = Math.max(lastNotWrittenMutationTimeMillis
                        + MAX_WRITE_PERMISSIONS_DELAY_MILLIS - currentTimeMillis, 0);
                final long writeDelayMillis = Math.min(WRITE_PERMISSIONS_DELAY_MILLIS,
                        maxDelayMillis);

                Message message = mHandler.obtainMessage(userId);
                mHandler.sendMessageDelayed(message, writeDelayMillis);
            } else {
                mLastNotWrittenMutationTimesMillis.put(userId, currentTimeMillis);
                Message message = mHandler.obtainMessage(userId);
                mHandler.sendMessageDelayed(message, WRITE_PERMISSIONS_DELAY_MILLIS);
                mWriteScheduled.put(userId, true);
            }
        }
```
就是发送消息给 MyHandler：

### 5.6.2 RuntimePermissionsPersistence.MyHandler.handleMessage 
```java
        private final class MyHandler extends Handler {
            public MyHandler() {
                super(BackgroundThread.getHandler().getLooper());
            }

            @Override
            public void handleMessage(Message message) {
                final int userId = message.what;
                Runnable callback = (Runnable) message.obj;
                //【4.6.3】writePermissionsSync！
                writePermissionsSync(userId);
                if (callback != null) {
                    callback.run();
                }
            }
        }
```

### 5.6.3 RuntimePermissionsPersistence.writePermissionsSync

writePermissionsSync 会将上一次运行时权限的授予情况保存到文件中！
```java
        private void writePermissionsSync(int userId) {
            AtomicFile destination = new AtomicFile(getUserRuntimePermissionsFile(userId));
            //【1】用于保存 package 和 SharedUser 运行时权限！
            ArrayMap<String, List<PermissionState>> permissionsForPackage = new ArrayMap<>();
            ArrayMap<String, List<PermissionState>> permissionsForSharedUser = new ArrayMap<>();

            synchronized (mLock) {
                mWriteScheduled.delete(userId);
                //【2】收集 package 运行时权限，保存到 permissionsForPackage！
                final int packageCount = mPackages.size();
                for (int i = 0; i < packageCount; i++) {
                    String packageName = mPackages.keyAt(i);
                    PackageSetting packageSetting = mPackages.valueAt(i);
                    if (packageSetting.sharedUser == null) {
                        PermissionsState permissionsState = packageSetting.getPermissionsState();
                        List<PermissionState> permissionsStates = permissionsState
                                .getRuntimePermissionStates(userId);
                        if (!permissionsStates.isEmpty()) {
                            permissionsForPackage.put(packageName, permissionsStates);
                        }
                    }
                }
                //【3】收集 SharedUser 运行时权限，保存到 permissionsForSharedUser！
                final int sharedUserCount = mSharedUsers.size();
                for (int i = 0; i < sharedUserCount; i++) {
                    String sharedUserName = mSharedUsers.keyAt(i);
                    SharedUserSetting sharedUser = mSharedUsers.valueAt(i);
                    PermissionsState permissionsState = sharedUser.getPermissionsState();
                    List<PermissionState> permissionsStates = permissionsState
                            .getRuntimePermissionStates(userId);
                    if (!permissionsStates.isEmpty()) {
                        permissionsForSharedUser.put(sharedUserName, permissionsStates);
                    }
                }
            }

            FileOutputStream out = null;
            try {
                out = destination.startWrite();

                XmlSerializer serializer = Xml.newSerializer();
                serializer.setOutput(out, StandardCharsets.UTF_8.name());
                serializer.setFeature(
                        "http://xmlpull.org/v1/doc/features.html#indent-output", true);
                serializer.startDocument(null, true);

                serializer.startTag(null, TAG_RUNTIME_PERMISSIONS); // 根标签 runtime-permissions！

                String fingerprint = mFingerprints.get(userId);
                if (fingerprint != null) {
                    serializer.attribute(null, ATTR_FINGERPRINT, fingerprint); // 属性 fingerprint！
                }
                // 写入 package 运行时权限！
                final int packageCount = permissionsForPackage.size();
                for (int i = 0; i < packageCount; i++) {
                    String packageName = permissionsForPackage.keyAt(i);
                    List<PermissionState> permissionStates = permissionsForPackage.valueAt(i);
                    serializer.startTag(null, TAG_PACKAGE); // pkg 标签
                    serializer.attribute(null, ATTR_NAME, packageName); // name 属性！
                    //【4.6.3.1】写入运行时权限信息
                    writePermissions(serializer, permissionStates);
                    serializer.endTag(null, TAG_PACKAGE);
                }

                // 写入 SharedUser 运行时权限！
                final int sharedUserCount = permissionsForSharedUser.size();
                for (int i = 0; i < sharedUserCount; i++) {
                    String packageName = permissionsForSharedUser.keyAt(i);
                    List<PermissionState> permissionStates = permissionsForSharedUser.valueAt(i);
                    serializer.startTag(null, TAG_SHARED_USER); // shared-user 标签；
                    serializer.attribute(null, ATTR_NAME, packageName); // name 属性；
                    //【4.6.3.1】写入运行时权限信息；
                    writePermissions(serializer, permissionStates);
                    serializer.endTag(null, TAG_SHARED_USER);
                }

                serializer.endTag(null, TAG_RUNTIME_PERMISSIONS);

                // Now any restored permission grants that are waiting for the apps
                // in question to be installed.  These are stored as per-package
                // TAG_RESTORED_RUNTIME_PERMISSIONS blocks, each containing some
                // number of individual permission grant entities.
                if (mRestoredUserGrants.get(userId) != null) {
                    ArrayMap<String, ArraySet<RestoredPermissionGrant>> restoredGrants =
                            mRestoredUserGrants.get(userId);
                    if (restoredGrants != null) {
                        final int pkgCount = restoredGrants.size();
                        for (int i = 0; i < pkgCount; i++) {
                            final ArraySet<RestoredPermissionGrant> pkgGrants =
                                    restoredGrants.valueAt(i);
                            if (pkgGrants != null && pkgGrants.size() > 0) {
                                final String pkgName = restoredGrants.keyAt(i);
                                serializer.startTag(null, TAG_RESTORED_RUNTIME_PERMISSIONS); // restored-perms 标签；
                                serializer.attribute(null, ATTR_PACKAGE_NAME, pkgName); // packageName 属性；

                                final int N = pkgGrants.size();
                                for (int z = 0; z < N; z++) {
                                    RestoredPermissionGrant g = pkgGrants.valueAt(z);
                                    serializer.startTag(null, TAG_PERMISSION_ENTRY); // perm 标签；
                                    serializer.attribute(null, ATTR_NAME, g.permissionName); // name 属性；

                                    if (g.granted) {
                                        serializer.attribute(null, ATTR_GRANTED, "true"); //  granted 属性
                                    }

                                    if ((g.grantBits&FLAG_PERMISSION_USER_SET) != 0) { //  set 属性
                                        serializer.attribute(null, ATTR_USER_SET, "true");
                                    }
                                    if ((g.grantBits&FLAG_PERMISSION_USER_FIXED) != 0) { //  fixed 属性
                                        serializer.attribute(null, ATTR_USER_FIXED, "true");
                                    }
                                    if ((g.grantBits&FLAG_PERMISSION_REVOKE_ON_UPGRADE) != 0) { //  rou 属性
                                        serializer.attribute(null, ATTR_REVOKE_ON_UPGRADE, "true");
                                    }
                                    serializer.endTag(null, TAG_PERMISSION_ENTRY);
                                }
                                serializer.endTag(null, TAG_RESTORED_RUNTIME_PERMISSIONS);
                            }
                        }
                    }
                }

                serializer.endDocument();
                destination.finishWrite(out);

                if (Build.FINGERPRINT.equals(fingerprint)) { // 如果
                    mDefaultPermissionsGranted.put(userId, true);
                }
            // Any error while writing is fatal.
            } catch (Throwable t) {
                Slog.wtf(PackageManagerService.TAG,
                        "Failed to write settings, restoring backup", t);
                destination.failWrite(out);
            } finally {
                IoUtils.closeQuietly(out);
            }
        }

```

#### 5.6.3.1 RuntimePermissionsPersistence.writePermissions
```java
        private void writePermissions(XmlSerializer serializer,
                List<PermissionState> permissionStates) throws IOException {
            for (PermissionState permissionState : permissionStates) {
                serializer.startTag(null, TAG_ITEM); // item 标签
                serializer.attribute(null, ATTR_NAME,permissionState.getName()); // name 属性
                serializer.attribute(null, ATTR_GRANTED,
                        String.valueOf(permissionState.isGranted())); // graned 属性
                serializer.attribute(null, ATTR_FLAGS,
                        Integer.toHexString(permissionState.getFlags())); // flags 属性
                serializer.endTag(null, TAG_ITEM);
            }
        }
```
这里就不多说了！