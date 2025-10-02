# PMS 第 4 篇 - PMS_DATA_SCAN_START 阶段
title: PMS 第 4 篇 - PMS_DATA_SCAN_START 阶段
date: 2018/02/17
categories:
- AndroidFramework源码分析
- PackageManager包管理
tags: PackageManager包管理
---

[toc]

基于 Android 7.1.1 源码分析 PackageManagerService 的架构和逻辑实现，本文是作者原创，转载请说明出处！

# 0 综述

通过扫描 system 分区阶段，我们得到如下的几个集合：

- **mExpectingBetter**：用来存放那些在 data 分区有更高版本的系统 app！
- **possiblyDeletedUpdatedSystemApps**：用来存储那些不存在的被更新过的系统 app！


```java
            // Set flag to monitor and not change apk file paths when
            // scanning install directories.
            final int scanFlags = SCAN_NO_PATHS | SCAN_DEFER_DEX | SCAN_BOOTING | SCAN_INITIAL;

            ... ... ... ...// 第三阶段
            
            if (!mOnlyCore) {
                EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_PMS_DATA_SCAN_START,
                        SystemClock.uptimeMillis());
                        
                // 同理，扫描 /data/app 目录和 /data/app-private 目录！
                scanDirTracedLI(mAppInstallDir, 0, scanFlags | SCAN_REQUIRE_KNOWN, 0);

                scanDirTracedLI(mDrmAppPrivateInstallDir, mDefParseFlags
                        | PackageParser.PARSE_FORWARD_LOCK,
                        scanFlags | SCAN_REQUIRE_KNOWN, 0);

                scanDirLI(mEphemeralInstallDir, mDefParseFlags
                        | PackageParser.PARSE_IS_EPHEMERAL,
                        scanFlags | SCAN_REQUIRE_KNOWN, 0);

                // 处理那些不存在的系统 app！
                for (String deletedAppName : possiblyDeletedUpdatedSystemApps) {
                    PackageParser.Package deletedPkg = mPackages.get(deletedAppName);
                    mSettings.removeDisabledSystemPackageLPw(deletedAppName);

                    String msg;
                    if (deletedPkg == null) {
                        msg = "Updated system package " + deletedAppName
                                + " no longer exists; it's data will be wiped";
                        // Actual deletion of code and data will be handled by later
                        // reconciliation step
                    } else {
                        msg = "Updated system app + " + deletedAppName
                                + " no longer present; removing system privileges for "
                                + deletedAppName;

                        deletedPkg.applicationInfo.flags &= ~ApplicationInfo.FLAG_SYSTEM;

                        PackageSetting deletedPs = mSettings.mPackages.get(deletedAppName);
                        deletedPs.pkgFlags &= ~ApplicationInfo.FLAG_SYSTEM;
                    }

                    logCriticalInfo(Log.WARN, msg);
                }

                // 确保所有在用户 data 分区的应用都显示出来了，如果 data 分区的无法显示，就显示 system 分区的！
                for (int i = 0; i < mExpectingBetter.size(); i++) {
                    final String packageName = mExpectingBetter.keyAt(i);

                    // 如果 PMS 仍然没有扫描到 mExpectingBetter 列表中的 apk，说明 data 分区的 apk 无法显示
                    // 那就要显示原来 system 分区的 apk！
                    if (!mPackages.containsKey(packageName)) {
                        final File scanFile = mExpectingBetter.valueAt(i);

                        logCriticalInfo(Log.WARN, "Expected better " + packageName
                                + " but never showed up; reverting to system");

                        // 根据目录，初始化解析标志！
                        int reparseFlags = mDefParseFlags;
                        if (FileUtils.contains(privilegedAppDir, scanFile)) {
                            reparseFlags = PackageParser.PARSE_IS_SYSTEM
                                    | PackageParser.PARSE_IS_SYSTEM_DIR
                                    | PackageParser.PARSE_IS_PRIVILEGED;
                                    
                        } else if (FileUtils.contains(systemAppDir, scanFile)) {
                            reparseFlags = PackageParser.PARSE_IS_SYSTEM
                                    | PackageParser.PARSE_IS_SYSTEM_DIR;
                                    
                        } else if (FileUtils.contains(vendorAppDir, scanFile)) {
                            reparseFlags = PackageParser.PARSE_IS_SYSTEM
                                    | PackageParser.PARSE_IS_SYSTEM_DIR;
                                    
                        } else if (FileUtils.contains(oemAppDir, scanFile)) {
                            reparseFlags = PackageParser.PARSE_IS_SYSTEM
                                    | PackageParser.PARSE_IS_SYSTEM_DIR;
                                    
                        } else {
                            Slog.e(TAG, "Ignoring unexpected fallback path " + scanFile);
                            continue;
                        }

                        // 使得 system 的 apk 可用；
                        mSettings.enableSystemPackageLPw(packageName);

                        try {
                        
                            // 重新扫描 system 的 apk！
                            scanPackageTracedLI(scanFile, reparseFlags, scanFlags, 0, null);
                        } catch (PackageManagerException e) {
                            Slog.e(TAG, "Failed to parse original system package: "
                                    + e.getMessage());
                        }
                    }
                }
            }
            mExpectingBetter.clear();

            // Resolve the storage manager.
            mStorageManagerPackage = getStorageManagerPackageName();

            // Resolve protected action filters. Only the setup wizard is allowed to
            // have a high priority filter for these actions.
            mSetupWizardPackage = getSetupWizardPackageName();
            if (mProtectedFilters.size() > 0) {
                if (DEBUG_FILTERS && mSetupWizardPackage == null) {
                    Slog.i(TAG, "No setup wizard;"
                        + " All protected intents capped to priority 0");
                }
                for (ActivityIntentInfo filter : mProtectedFilters) {
                    if (filter.activity.info.packageName.equals(mSetupWizardPackage)) {
                        if (DEBUG_FILTERS) {
                            Slog.i(TAG, "Found setup wizard;"
                                + " allow priority " + filter.getPriority() + ";"
                                + " package: " + filter.activity.info.packageName
                                + " activity: " + filter.activity.className
                                + " priority: " + filter.getPriority());
                        }
                        // skip setup wizard; allow it to keep the high priority filter
                        continue;
                    }
                    Slog.w(TAG, "Protected action; cap priority to 0;"
                            + " package: " + filter.activity.info.packageName
                            + " activity: " + filter.activity.className
                            + " origPrio: " + filter.getPriority());
                    filter.setPriority(0);
                }
            }
            mDeferProtectedFilters = false;
            mProtectedFilters.clear();

            // 我们需要更新所有的应用，保证他们有正确的共享库路径。
            updateAllSharedLibrariesLPw();

            for (SharedUserSetting setting : mSettings.getAllSharedUsersLPw()) {
                adjustCpuAbisForSharedUserLPw(setting.packages, null /* scanned package */,
                        false /* boot complete */);
            }

            // 更新所有 package 的最新使用时间！
            mPackageUsage.read(mPackages);
            mCompilerStats.read();
            
            ... ... ... ...// 见，第四阶段
```

# 1 扫描 data 分区

data 分区比较特殊，data 分区的 apk 有两种：

- 一种是非系统 apk；
- 另外一种是系统 apk 通过覆盖安装的方式，安装到了 data 分区；

下面我们来看看 data 分区的扫描过程！
```java
            final int scanFlags = SCAN_NO_PATHS | SCAN_DEFER_DEX | SCAN_BOOTING | SCAN_INITIAL;
            
                // 同理，扫描 /data/app 目录和 /data/app-private 目录！
                scanDirTracedLI(mAppInstallDir, 0, scanFlags | SCAN_REQUIRE_KNOWN, 0);

                scanDirTracedLI(mDrmAppPrivateInstallDir, mDefParseFlags
                        | PackageParser.PARSE_FORWARD_LOCK,
                        scanFlags | SCAN_REQUIRE_KNOWN, 0);

                scanDirLI(mEphemeralInstallDir, mDefParseFlags
                        | PackageParser.PARSE_IS_EPHEMERAL,
                        scanFlags | SCAN_REQUIRE_KNOWN, 0);
```
扫描的目录如下：

- /data/app；
   - **parseFlags**：0
   - **scanFlags**：scanFlags | SCAN_REQUIRE_KNOWN
- /data/app-private；
   - **parseFlags**：PackageParser.PARSE_FORWARD_LOCK
   - **scanFlags**：scanFlags | SCAN_REQUIRE_KNOWN
- /data/app-ephemeral；
   - **parseFlags**：PackageParser.PARSE_IS_EPHEMERAL
   - **scanFlags**：scanFlags | SCAN_REQUIRE_KNOWN



这里的步骤和扫描 system 分区是一样的，但是因为 flag 不同，所以会涉及不同的逻辑，这里我们分析一下重点方法，根据之前的扫描顺序，最后会调用一下方法：scanPackageInternalLI 方法！

## 1.1 PMS.scanPackageInternalLI

我们继续来看，通过前面的扫描解析，我们获得了应用程序的 PackageParser.Package 对象，接下来，就是要处理解析获得的数据，policyFlags 的值就是最开始传入的 parseFlags！
```java
    private PackageParser.Package scanPackageInternalLI(PackageParser.Package pkg, File scanFile,
            int policyFlags, int scanFlags, long currentTime, UserHandle user)
            throws PackageManagerException {

        PackageSetting ps = null;
        PackageSetting updatedPkg;

        synchronized (mPackages) {
            // 对于系统 apk 来说，才有源包！！
            String oldName = mSettings.mRenamedPackages.get(pkg.packageName);
            if (pkg.mOriginalPackages != null && pkg.mOriginalPackages.contains(oldName)) {
                ps = mSettings.peekPackageLPr(oldName);
            }

            // 尝试获得 PackageSetting 对象！
            if (ps == null) {
                ps = mSettings.peekPackageLPr(pkg.packageName);
            }
        
            // 对于非系统的 apk，updatedPkg 是为 null 的！
            // 但对于覆盖安装的系统 apk 来说，updatedPkg 不为 null 的！
            updatedPkg = mSettings.getDisabledSystemPkgLPr(ps != null ? ps.name : pkg.packageName);
            if (DEBUG_INSTALL && updatedPkg != null) Slog.d(TAG, "updatedPkg = " + updatedPkg);

            // 因为扫描的是 data 分区，不进入这个分支！
            if ((policyFlags & PackageParser.PARSE_IS_SYSTEM) != 0) { 
                ... ... ... ...

            }
        }

        // 目前，不进入这个分支！
        boolean updatedPkgBetter = false;
        if (updatedPkg != null && (policyFlags & PackageParser.PARSE_IS_SYSTEM) != 0) {
            ... ... ... ...
        
        }

        // 非系统 apk，不进入这个分支！
        // 但是对于覆盖安装的系统 apk，updatedPkg 不为 null，所以要添加指定的扫描位，表示当前解析的是一个系统 apk！
        if (updatedPkg != null) { 
            policyFlags |= PackageParser.PARSE_IS_SYSTEM;

            if ((updatedPkg.pkgPrivateFlags & ApplicationInfo.PRIVATE_FLAG_PRIVILEGED) != 0) {
                policyFlags |= PackageParser.PARSE_IS_PRIVILEGED;
            }
        }

        collectCertificatesLI(ps, pkg, scanFile, policyFlags);

        // 同样的， policyFlags 并没有置 PackageParser.PARSE_IS_SYSTEM_DIR 位为 1，所以不进入改分支！
        boolean shouldHideSystemApp = false;
        if (updatedPkg == null && ps != null
                && (policyFlags & PackageParser.PARSE_IS_SYSTEM_DIR) != 0 && !isSystemApp(ps)) {
           
           ... ... ... ...
           
        }

        // 对于非系统的 apk，可能存在 apk 路径和 data 路径不一样的情况！
        if ((policyFlags & PackageParser.PARSE_IS_SYSTEM_DIR) == 0) {
            if (ps != null && !ps.codePath.equals(ps.resourcePath)) {
            
                // 如果不一样，就加上 PackageParser.PARSE_FORWARD_LOCK 位！
                policyFlags |= PackageParser.PARSE_FORWARD_LOCK;
            }
        }

        
        // 下面是处理 data apk 的 apk 路径和资源路径！
        String resourcePath = null;
        String baseResourcePath = null;

        if ((policyFlags & PackageParser.PARSE_FORWARD_LOCK) != 0 && !updatedPkgBetter) {
            if (ps != null && ps.resourcePathString != null) {
                resourcePath = ps.resourcePathString;
                baseResourcePath = ps.resourcePathString;

            } else {
                // Should not happen at all. Just log an error.
                Slog.e(TAG, "Resource path not set for package " + pkg.packageName);
            }
        } else {
            resourcePath = pkg.codePath;
            baseResourcePath = pkg.baseCodePath;

        }

        // 设置 ApplicationInfo 对象的属性！
        pkg.setApplicationVolumeUuid(pkg.volumeUuid);
        pkg.setApplicationInfoCodePath(pkg.codePath);
        pkg.setApplicationInfoBaseCodePath(pkg.baseCodePath);
        pkg.setApplicationInfoSplitCodePaths(pkg.splitCodePaths);
        pkg.setApplicationInfoResourcePath(resourcePath);
        pkg.setApplicationInfoBaseResourcePath(baseResourcePath);
        pkg.setApplicationInfoSplitResourcePaths(pkg.splitCodePaths);

        //【1.2】接着调用 scanPackageLI，继续处理扫描数据！
        PackageParser.Package scannedPkg = scanPackageLI(pkg, policyFlags, scanFlags
                | SCAN_UPDATE_SIGNATURE, currentTime, user);

        // 对于非系统的 app，shouldHideSystemApp 为 false，不走这个分支！
        if (shouldHideSystemApp) {
            synchronized (mPackages) {
                mSettings.disableSystemPackageLPw(pkg.packageName, true);
            }
        }

        return scannedPkg;
    }

```
对于非系统 apk，有两种情况：

- apk 之前已经安装到了 data 分区，所以肯定有对应的 PackageSetting 对象；
- apk 之前并没有安装，这次通过 OTA 升级的方式，通过 data 分区内置 app 的方式安装 apk；

下面，我们继续来分析：

## 1.2 PMS.scanPackageLI

继续来看：
```java
    private PackageParser.Package scanPackageLI(PackageParser.Package pkg, final int policyFlags,
            int scanFlags, long currentTime, UserHandle user) throws PackageManagerException {
        boolean success = false;
        try {
            //【1.3】继续调用 scanPackageDirtyLI 处理扫描数据！
            final PackageParser.Package res = scanPackageDirtyLI(pkg, policyFlags, scanFlags,
                    currentTime, user);
                    
            success = true;
            return res;
        } finally {
            if (!success && (scanFlags & SCAN_DELETE_DATA_ON_FAILURES) != 0) {
            
                // DELETE_DATA_ON_FAILURES is only used by frozen paths
                // 失败了就清楚掉数据！
                destroyAppDataLIF(pkg, UserHandle.USER_ALL,
                        StorageManager.FLAG_STORAGE_DE | StorageManager.FLAG_STORAGE_CE);
                destroyAppProfilesLIF(pkg, UserHandle.USER_ALL);
            }
        }
    }
```
继续看：

## 1.3 PMS.scanPackageDirtyLI

最终，同样调用 scanPackageDirtyLI 方法处理扫描得到的数据，传入参数：

- **PackageParser.Package pkg**：非系统目录下解析的 apk 数据；

```java
    private PackageParser.Package scanPackageDirtyLI(PackageParser.Package pkg,
            final int policyFlags, final int scanFlags, long currentTime, UserHandle user)
            throws PackageManagerException {

        final File scanFile = new File(pkg.codePath);
        if (pkg.applicationInfo.getCodePath() == null ||
                pkg.applicationInfo.getResourcePath() == null) {

            throw new PackageManagerException(INSTALL_FAILED_INVALID_APK,
                    "Code and resource paths haven't been set correctly");
        }
        //【×】扫描 data 分区不会进入这里，那么 package 也就不会被设置 ApplicationInfo.FLAG_SYSTEM 标志位！
        if ((policyFlags&PackageParser.PARSE_IS_SYSTEM) != 0) { 
            pkg.applicationInfo.flags |= ApplicationInfo.FLAG_SYSTEM;
            if (pkg.applicationInfo.isDirectBootAware()) {

                for (PackageParser.Service s : pkg.services) {
                    s.info.encryptionAware = s.info.directBootAware = true;
                }
                for (PackageParser.Provider p : pkg.providers) {
                    p.info.encryptionAware = p.info.directBootAware = true;
                }
                for (PackageParser.Activity a : pkg.activities) {
                    a.info.encryptionAware = a.info.directBootAware = true;
                }
                for (PackageParser.Activity r : pkg.receivers) {
                    r.info.encryptionAware = r.info.directBootAware = true;
                }
            }
        } else {
            //【×】data 分区 apk 进入这个分支，coreApp 的值为 false，并清除一些 flag；
            pkg.coreApp = false;
            pkg.applicationInfo.privateFlags &=
                    ~ApplicationInfo.PRIVATE_FLAG_DEFAULT_TO_DEVICE_PROTECTED_STORAGE;
            pkg.applicationInfo.privateFlags &=
                    ~ApplicationInfo.PRIVATE_FLAG_DIRECT_BOOT_AWARE;
        }

        // 下面这部分扫描 data 分区也不会遇到！
        pkg.mTrustedOverlay = (policyFlags&PackageParser.PARSE_TRUSTED_OVERLAY) != 0;

        if ((policyFlags&PackageParser.PARSE_IS_PRIVILEGED) != 0) {
            pkg.applicationInfo.privateFlags |= ApplicationInfo.PRIVATE_FLAG_PRIVILEGED;
        }

        if ((policyFlags & PackageParser.PARSE_ENFORCE_CODE) != 0) {
            enforceCodePolicy(pkg);
        }

        if (mCustomResolverComponentName != null &&
                mCustomResolverComponentName.getPackageName().equals(pkg.packageName)) {
            setUpCustomResolverActivity(pkg);
        }

        // 扫描 data 不会进入这里！！ 
        if (pkg.packageName.equals("android")) {
            ... ... ... ...
        }

        if (DEBUG_PACKAGE_SCANNING) {
            if ((policyFlags & PackageParser.PARSE_CHATTY) != 0)
                Log.d(TAG, "Scanning package " + pkg.packageName);
        }

        synchronized (mPackages) {
            // 如果 PMS 已经扫描并处理了这个 apk，就抛异常，跳过！
            if (mPackages.containsKey(pkg.packageName)
                    || mSharedLibraries.containsKey(pkg.packageName)) {
                throw new PackageManagerException(INSTALL_FAILED_DUPLICATE_PACKAGE,
                        "Application package " + pkg.packageName
                                + " already installed.  Skipping duplicate.");
            }


            //【A】对于 data 分区的 apk，进入该分支！
            if ((scanFlags & SCAN_REQUIRE_KNOWN) != 0) {
                // 如果 mExpectingBetter 中有该应用的包名，说明该应用是覆盖了 system apk！
                // 这种情况，我们后续会选择 versionCode 更高的显示出来！
                if (mExpectingBetter.containsKey(pkg.packageName)) {
                    logCriticalInfo(Log.WARN,
                            "Relax SCAN_REQUIRE_KNOWN requirement for package " + pkg.packageName);
                        
                } else {
                    // 获得上一次安装的信息!
                    PackageSetting known = mSettings.peekPackageLPr(pkg.packageName);
                    if (known != null) {
                        if (DEBUG_PACKAGE_SCANNING) {
                            Log.d(TAG, "Examining " + pkg.codePath
                                    + " and requiring known paths " + known.codePathString
                                    + " & " + known.resourcePathString);
                        }

                        //【B】如果本次扫描和之前安装后的的 codePath 或者 ResourcePath 不一样，抛出异常，结束处理！
                        // 这种情况，我们会忽视这个 apk！！
                        if (!pkg.applicationInfo.getCodePath().equals(known.codePathString)
                                || !pkg.applicationInfo.getResourcePath().equals(
                                known.resourcePathString)) {

                            throw new PackageManagerException(INSTALL_FAILED_PACKAGE_CHANGED,
                                    "Application package " + pkg.packageName
                                            + " found at " + pkg.applicationInfo.getCodePath()
                                            + " but expected at " + known.codePathString
                                            + "; ignoring.");
                        }
                    }
                }
            }
        }

        // 初始化资源路径！
        File destCodeFile = new File(pkg.applicationInfo.getCodePath());
        File destResourceFile = new File(pkg.applicationInfo.getResourcePath());

        SharedUserSetting suid = null;
        PackageSetting pkgSetting = null;

        // 非系统 apk ，要取消以下属性！！，判断是否是 system app 的依据是 
        // 是否有这个 ApplicationInfo.FLAG_SYSTEM 标志位！
        if (!isSystemApp(pkg)) {
            pkg.mOriginalPackages = null;
            pkg.mRealPackage = null;
            pkg.mAdoptPermissions = null;
        }

        // Getting the package setting may have a side-effect, so if we
        // are only checking if scan would succeed, stash a copy of the
        // old setting to restore at the end.
        PackageSetting nonMutatedPs = null;

        synchronized (mPackages) {
            // 如果 apk 是共享 uid 的！
            if (pkg.mSharedUserId != null) {
                // 获得 SharedUserSetting 对象！
                suid = mSettings.getSharedUserLPw(pkg.mSharedUserId, 0, 0, true);
                if (suid == null) {
                    throw new PackageManagerException(INSTALL_FAILED_INSUFFICIENT_STORAGE,
                            "Creating application package " + pkg.packageName
                            + " for shared user failed");
                }

                if (DEBUG_PACKAGE_SCANNING) {
                    if ((policyFlags & PackageParser.PARSE_CHATTY) != 0)
                        Log.d(TAG, "Shared UserID " + pkg.mSharedUserId + " (uid=" + suid.userId
                                + "): packages=" + suid.packages);
                }
            }

            // 非系统 apk 无源包，不进入这个分支！
            // 覆盖安装的系统 apk，可以进入这个分支，这里的目的是找到合适的源包，用来设置包名，已经复用源包的数据！
            PackageSetting origPackage = null;
            String realName = null;
            if (pkg.mOriginalPackages != null) {
                
                final String renamed = mSettings.mRenamedPackages.get(pkg.mRealPackage);
                
                if (pkg.mOriginalPackages.contains(renamed)) {
                
                    realName = pkg.mRealPackage;
                    if (!pkg.packageName.equals(renamed)) {
                    
                        pkg.setPackageName(renamed);
                    }

                } else {
    
                    for (int i=pkg.mOriginalPackages.size()-1; i>=0; i--) {
                        if ((origPackage = mSettings.peekPackageLPr(
                                pkg.mOriginalPackages.get(i))) != null) {
                        
                            if (!verifyPackageUpdateLPr(origPackage, pkg)) {
                                // New package is not compatible with original.
                                origPackage = null;
                                continue;
                                
                            } else if (origPackage.sharedUser != null) {
                                if (!origPackage.sharedUser.name.equals(pkg.mSharedUserId)) {
                                    Slog.w(TAG, "Unable to migrate data from " + origPackage.name
                                            + " to " + pkg.packageName + ": old uid "
                                            + origPackage.sharedUser.name
                                            + " differs from " + pkg.mSharedUserId);
                                    origPackage = null;
                                    continue;
                                }
                                // TODO: Add case when shared user id is added [b/28144775]
                            } else {
                            
                                if (DEBUG_UPGRADE) Log.v(TAG, "Renaming new package "
                                        + pkg.packageName + " to old name " + origPackage.name);
                            }
                            break;
                        }
                    }
                }
            }
    

            if (mTransferedPackages.contains(pkg.packageName)) {
                Slog.w(TAG, "Package " + pkg.packageName
                        + " was transferred to another, but its .apk remains");
            }

            // See comments in nonMutatedPs declaration
            if ((scanFlags & SCAN_CHECK_ONLY) != 0) {
                PackageSetting foundPs = mSettings.peekPackageLPr(pkg.packageName);
                if (foundPs != null) {
                    nonMutatedPs = new PackageSetting(foundPs);
                }
            }
            
            //【1.3.1】获得当前扫描的这个 package 对应的 packageSetting 对象，如果已经存在就直接返回，不存在就创建！
            // 如果 origPackage 不为 null，创建新的需要重命名为源包的名字！
            pkgSetting = mSettings.getPackageLPw(pkg, origPackage, realName, suid, destCodeFile,
                    destResourceFile, pkg.applicationInfo.nativeLibraryRootDir,
                    pkg.applicationInfo.primaryCpuAbi,
                    pkg.applicationInfo.secondaryCpuAbi,
                    pkg.applicationInfo.flags, pkg.applicationInfo.privateFlags,
                    user, false);
    
            if (pkgSetting == null) {
                throw new PackageManagerException(INSTALL_FAILED_INSUFFICIENT_STORAGE,
                        "Creating application package " + pkg.packageName + " failed");
            }

            if (pkgSetting.origPackage != null) { // 非系统 apk 不会进入该分支!
            
                pkg.setPackageName(origPackage.name);

                String msg = "New package " + pkgSetting.realName
                        + " renamed to replace old package " + pkgSetting.name;
                reportSettingsProblem(Log.WARN, msg);

                if ((scanFlags & SCAN_CHECK_ONLY) == 0) {
                    mTransferedPackages.add(origPackage.name);
                }

                pkgSetting.origPackage = null;
            }

            if ((scanFlags & SCAN_CHECK_ONLY) == 0 && realName != null) {
                mTransferedPackages.add(pkg.packageName);
            }

            // 如果是被覆盖安装的系统 apk，需要添加 ApplicationInfo.FLAG_UPDATED_SYSTEM_APP 标签
            if (mSettings.isDisabledSystemPackageLPr(pkg.packageName)) {
                pkg.applicationInfo.flags |= ApplicationInfo.FLAG_UPDATED_SYSTEM_APP;
            }

            if ((policyFlags&PackageParser.PARSE_IS_SYSTEM_DIR) == 0) {
                updateSharedLibrariesLPw(pkg, null);
            }

            // 设置 Polisy 标签！
            if (mFoundPolicyFile) {
                SELinuxMMAC.assignSeinfoValue(pkg);
            }

            pkg.applicationInfo.uid = pkgSetting.appId;
            pkg.mExtras = pkgSetting;

            // 处理 keySet 更新和签名校验！
            if (shouldCheckUpgradeKeySetLP(pkgSetting, scanFlags)) {
                if (checkUpgradeKeySetLP(pkgSetting, pkg)) {
                    // We just determined the app is signed correctly, so bring
                    // over the latest parsed certs.
                    pkgSetting.signatures.mSignatures = pkg.mSignatures;
                } else {
                    if ((policyFlags & PackageParser.PARSE_IS_SYSTEM_DIR) == 0) {
                        throw new PackageManagerException(INSTALL_FAILED_UPDATE_INCOMPATIBLE,
                                "Package " + pkg.packageName + " upgrade keys do not match the "
                                + "previously installed version");
                    } else {
                        pkgSetting.signatures.mSignatures = pkg.mSignatures;
                        String msg = "System package " + pkg.packageName
                            + " signature changed; retaining data.";
                        reportSettingsProblem(Log.WARN, msg);
                    }
                }
            } else {
                try {
                    verifySignaturesLP(pkgSetting, pkg);

                    pkgSetting.signatures.mSignatures = pkg.mSignatures;
                    
                } catch (PackageManagerException e) {
                    if ((policyFlags & PackageParser.PARSE_IS_SYSTEM_DIR) == 0) {
                        throw e;
                    }

                    pkgSetting.signatures.mSignatures = pkg.mSignatures;

                    if (pkgSetting.sharedUser != null) {
                        if (compareSignatures(pkgSetting.sharedUser.signatures.mSignatures,
                                              pkg.mSignatures) != PackageManager.SIGNATURE_MATCH) {
                            throw new PackageManagerException(
                                    INSTALL_PARSE_FAILED_INCONSISTENT_CERTIFICATES,
                                            "Signature mismatch for shared user: "
                                            + pkgSetting.sharedUser);
                        }
                    }
                    // File a report about this.
                    String msg = "System package " + pkg.packageName
                        + " signature changed; retaining data.";
                    reportSettingsProblem(Log.WARN, msg);
                }
            }
            

            // 安装的时候才会进入这个分支，这里不看！
            // 判断这个 package 使用的 content providers 是否和已经存在 package 冲突！
            if ((scanFlags & SCAN_NEW_INSTALL) != 0) { 
                final int N = pkg.providers.size();
                int i;
                for (i=0; i<N; i++) {
                    PackageParser.Provider p = pkg.providers.get(i);
                    if (p.info.authority != null) {
                        String names[] = p.info.authority.split(";");
                        for (int j = 0; j < names.length; j++) {
                            if (mProvidersByAuthority.containsKey(names[j])) {
                                PackageParser.Provider other = mProvidersByAuthority.get(names[j]);
                                final String otherPackageName =
                                        ((other != null && other.getComponentName() != null) ?
                                                other.getComponentName().getPackageName() : "?");
                                throw new PackageManagerException(
                                        INSTALL_FAILED_CONFLICTING_PROVIDER,
                                                "Can't install because provider name " + names[j]
                                                + " (in package " + pkg.applicationInfo.packageName
                                                + ") is already used by " + otherPackageName);
                            }
                        }
                    }
                }
            }

            // 同样的，只有系统 apk 才能进入该分支！
            if ((scanFlags & SCAN_CHECK_ONLY) == 0 && pkg.mAdoptPermissions != null) {

                // This package wants to adopt ownership of permissions from
                // another package.
                for (int i = pkg.mAdoptPermissions.size() - 1; i >= 0; i--) {
                    final String origName = pkg.mAdoptPermissions.get(i);
                    final PackageSetting orig = mSettings.peekPackageLPr(origName);
                    if (orig != null) {
                        if (verifyPackageUpdateLPr(orig, pkg)) {
                            Slog.i(TAG, "Adopting permissions from " + origName + " to "
                                    + pkg.packageName);

                            mSettings.transferPermissionsLPw(origName, pkg.packageName);
                        }
                    }
                }
            }
        }

        final String pkgName = pkg.packageName;

        final long scanFileTime = getLastModifiedTime(pkg, scanFile);
        final boolean forceDex = (scanFlags & SCAN_FORCE_DEX) != 0;
        
        // 设置 pacakge 对应的进程名！
        pkg.applicationInfo.processName = fixProcessName(
                pkg.applicationInfo.packageName,
                pkg.applicationInfo.processName,
                pkg.applicationInfo.uid);

        if (pkg != mPlatformPackage) {

            // Get all of our default paths setup
            pkg.applicationInfo.initForUser(UserHandle.USER_SYSTEM);
        }

        final String path = scanFile.getPath();
        final String cpuAbiOverride = deriveAbiOverride(pkg.cpuAbiOverride, pkgSetting);

        // 下面是设置本地库和系统平台相关的属性！
        if ((scanFlags & SCAN_NEW_INSTALL) == 0) {
            derivePackageAbi(pkg, scanFile, cpuAbiOverride, true /* extract libs */);

            if (isSystemApp(pkg) && !pkg.isUpdatedSystemApp() &&
                    pkg.applicationInfo.primaryCpuAbi == null) {
                setBundledAppAbisAndRoots(pkg, pkgSetting);
                setNativeLibraryPaths(pkg);
            }

        } else {
            if ((scanFlags & SCAN_MOVE) != 0) {
            
            }

            setNativeLibraryPaths(pkg);
        }

        if (mPlatformPackage == pkg) {
            pkg.applicationInfo.primaryCpuAbi = VMRuntime.getRuntime().is64Bit() ?
                    Build.SUPPORTED_64_BIT_ABIS[0] : Build.SUPPORTED_32_BIT_ABIS[0];
        }

        if ((scanFlags & SCAN_NO_DEX) == 0 && (scanFlags & SCAN_NEW_INSTALL) != 0) {
            if (cpuAbiOverride == null && pkgSetting.cpuAbiOverrideString != null) {
                Slog.w(TAG, "Ignoring persisted ABI override " + cpuAbiOverride +
                        " for package " + pkg.packageName);
            }
        }

        pkgSetting.primaryCpuAbiString = pkg.applicationInfo.primaryCpuAbi;
        pkgSetting.secondaryCpuAbiString = pkg.applicationInfo.secondaryCpuAbi;
        pkgSetting.cpuAbiOverrideString = cpuAbiOverride;

        pkg.cpuAbiOverride = cpuAbiOverride;

        if (DEBUG_ABI_SELECTION) {
            Slog.d(TAG, "Resolved nativeLibraryRoot for " + pkg.applicationInfo.packageName
                    + " to root=" + pkg.applicationInfo.nativeLibraryRootDir + ", isa="
                    + pkg.applicationInfo.nativeLibraryRootRequiresIsa);
        }

        pkgSetting.legacyNativeLibraryPathString = pkg.applicationInfo.nativeLibraryRootDir;

        if (DEBUG_ABI_SELECTION) {
            Log.d(TAG, "Abis for package[" + pkg.packageName + "] are" +
                    " primary=" + pkg.applicationInfo.primaryCpuAbi +
                    " secondary=" + pkg.applicationInfo.secondaryCpuAbi);
        }

        if ((scanFlags & SCAN_BOOTING) == 0 && pkgSetting.sharedUser != null) {
            adjustCpuAbisForSharedUserLPw(pkgSetting.sharedUser.packages,
                    pkg, true /* boot complete */);
        }

        if (mFactoryTest && pkg.requestedPermissions.contains(
                android.Manifest.permission.FACTORY_TEST)) {
            pkg.applicationInfo.flags |= ApplicationInfo.FLAG_FACTORY_TEST;
        }

        // 如果是系统 package，设置 isOrphaned 的属性为 true！
        if (isSystemApp(pkg)) {
            pkgSetting.isOrphaned = true;
        }

        ArrayList<PackageParser.Package> clientLibPkgs = null;

        if ((scanFlags & SCAN_CHECK_ONLY) != 0) {
            if (nonMutatedPs != null) {
                synchronized (mPackages) {
                    mSettings.mPackages.put(nonMutatedPs.name, nonMutatedPs);
                }
            }
            return pkg;
        }

        // 处理特权 apk 的子包，只有特权 apk 才能添加子包；
        // 特权 apk 包括两部分：
        // 1、特定 uid 的 app
        // 2、framework-res.apk 和 system/priv-app 目录下的 apk！
        if (pkg.childPackages != null && !pkg.childPackages.isEmpty()) {
            if ((policyFlags & PARSE_IS_PRIVILEGED) == 0) {
                throw new PackageManagerException("Only privileged apps and updated "
                        + "privileged apps can add child packages. Ignoring package "
                        + pkg.packageName);
            }
            final int childCount = pkg.childPackages.size();
            for (int i = 0; i < childCount; i++) {
                PackageParser.Package childPkg = pkg.childPackages.get(i);
                if (mSettings.hasOtherDisabledSystemPkgWithChildLPr(pkg.packageName,
                        childPkg.packageName)) {
                    throw new PackageManagerException("Cannot override a child package of "
                            + "another disabled system app. Ignoring package " + pkg.packageName);
                }
            }
        }

        synchronized (mPackages) {
            if ((pkg.applicationInfo.flags&ApplicationInfo.FLAG_SYSTEM) != 0) {

                // 如果是系统 package，并且他之前更新过，就要尝试对共享库 lib 进行更新！
                // 只有系统 package 才能添加共享 lib
                if (pkg.libraryNames != null) {
                    for (int i=0; i<pkg.libraryNames.size(); i++) {
                        String name = pkg.libraryNames.get(i);
                        boolean allowed = false;
                        if (pkg.isUpdatedSystemApp()) {
                        
                            // New library entries can only be added through the
                            // system image.  This is important to get rid of a lot
                            // of nasty edge cases: for example if we allowed a non-
                            // system update of the app to add a library, then uninstalling
                            // the update would make the library go away, and assumptions
                            // we made such as through app install filtering would now
                            // have allowed apps on the device which aren't compatible
                            // with it.  Better to just have the restriction here, be
                            // conservative, and create many fewer cases that can negatively
                            // impact the user experience.
                            final PackageSetting sysPs = mSettings
                                    .getDisabledSystemPkgLPr(pkg.packageName);
                            if (sysPs.pkg != null && sysPs.pkg.libraryNames != null) {
                                for (int j=0; j<sysPs.pkg.libraryNames.size(); j++) {
                                    if (name.equals(sysPs.pkg.libraryNames.get(j))) {
                                        allowed = true;
                                        break;
                                    }
                                }
                            }
                        } else {
                            allowed = true;
                        }
                        if (allowed) {
                            if (!mSharedLibraries.containsKey(name)) {
                                mSharedLibraries.put(name, new SharedLibraryEntry(null, pkg.packageName));
                                
                            } else if (!name.equals(pkg.packageName)) {
                                Slog.w(TAG, "Package " + pkg.packageName + " library "
                                        + name + " already exists; skipping");
                            }
                        } else {
                            Slog.w(TAG, "Package " + pkg.packageName + " declares lib "
                                    + name + " that is not declared on system image; skipping");
                        }
                    }
                    if ((scanFlags & SCAN_BOOTING) == 0) {

                        // If we are not booting, we need to update any applications
                        // that are clients of our shared library.  If we are booting,
                        // this will all be done once the scan is complete.
                        clientLibPkgs = updateAllSharedLibrariesLPw(pkg);
                    }
                }
            }
        }

        if ((scanFlags & SCAN_BOOTING) != 0) {
        } else if ((scanFlags & SCAN_DONT_KILL_APP) != 0) {
        } else if ((scanFlags & SCAN_IGNORE_FROZEN) != 0) {
        } else {
            // 尝试去冻结 apk！
            checkPackageFrozen(pkgName);
        }

        if (clientLibPkgs != null) {
            for (int i=0; i<clientLibPkgs.size(); i++) {
                PackageParser.Package clientPkg = clientLibPkgs.get(i);
                killApplication(clientPkg.applicationInfo.packageName,
                        clientPkg.applicationInfo.uid, "update lib");
            }
        }

        // Make sure we're not adding any bogus keyset info
        KeySetManagerService ksms = mSettings.mKeySetManagerService;
        ksms.assertScannedPackageValid(pkg);

        // writer
        Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "updateSettings");

        boolean createIdmapFailed = false;
        synchronized (mPackages) {

            // We don't expect installation to fail beyond this point
            if (pkgSetting.pkg != null) {

                // Note that |user| might be null during the initial boot scan. If a codePath
                // for an app has changed during a boot scan, it's due to an app update that's
                // part of the system partition and marker changes must be applied to all users.
                maybeRenameForeignDexMarkers(pkgSetting.pkg, pkg,
                    (user != null) ? user : UserHandle.ALL);
            }

            //【×】将新创建的 PackageSetting 添加到 mSettings 中！
            mSettings.insertPackageSettingLPw(pkgSetting, pkg);
            
            // 将新创建的 PackageSetting 添加到 PMS.mPackages 中！
            mPackages.put(pkg.applicationInfo.packageName, pkg);
            
            // 将当前的 package 从 mSettings.mPackagesToBeCleaned 中移除，防止数据清除！
            final Iterator<PackageCleanItem> iter = mSettings.mPackagesToBeCleaned.iterator();
            while (iter.hasNext()) {
                PackageCleanItem item = iter.next();
                if (pkgName.equals(item.packageName)) {
                    iter.remove();
                }
            }

            // 处理 package 的第一次安装时间和最近更新时间！
            if (currentTime != 0) {
                if (pkgSetting.firstInstallTime == 0) {
                    pkgSetting.firstInstallTime = pkgSetting.lastUpdateTime = currentTime;

                } else if ((scanFlags&SCAN_UPDATE_TIME) != 0) {
                    pkgSetting.lastUpdateTime = currentTime;

                }
            } else if (pkgSetting.firstInstallTime == 0) {
            
                // We need *something*.  Take time time stamp of the file.
                pkgSetting.firstInstallTime = pkgSetting.lastUpdateTime = scanFileTime;
                
            } else if ((policyFlags&PackageParser.PARSE_IS_SYSTEM_DIR) != 0) {
                if (scanFileTime != pkgSetting.timeStamp) {

                    // A package on the system image has changed; consider this
                    // to be an update.
                    pkgSetting.lastUpdateTime = scanFileTime;
                }
            }

            // Add the package's KeySets to the global KeySetManagerService
            ksms.addScannedPackageLPw(pkg);
            
            // 处理该 Package 中 的 Provider 信息，添加到 mProviders 中！
            int N = pkg.providers.size();
            StringBuilder r = null;
            int i;
            for (i=0; i<N; i++) {
                PackageParser.Provider p = pkg.providers.get(i);
                p.info.processName = fixProcessName(pkg.applicationInfo.processName,
                        p.info.processName, pkg.applicationInfo.uid);
                mProviders.addProvider(p);
                p.syncable = p.info.isSyncable;
                if (p.info.authority != null) {
                    String names[] = p.info.authority.split(";");
                    p.info.authority = null;
                    for (int j = 0; j < names.length; j++) {
                        if (j == 1 && p.syncable) {
                        
                            // We only want the first authority for a provider to possibly be
                            // syncable, so if we already added this provider using a different
                            // authority clear the syncable flag. We copy the provider before
                            // changing it because the mProviders object contains a reference
                            // to a provider that we don't want to change.
                            // Only do this for the second authority since the resulting provider
                            // object can be the same for all future authorities for this provider.
                            p = new PackageParser.Provider(p);
                            p.syncable = false;
                        }
                        if (!mProvidersByAuthority.containsKey(names[j])) {
                            mProvidersByAuthority.put(names[j], p);
                            if (p.info.authority == null) {
                                p.info.authority = names[j];
                            } else {
                                p.info.authority = p.info.authority + ";" + names[j];
                            }
                            if (DEBUG_PACKAGE_SCANNING) {
                                if ((policyFlags & PackageParser.PARSE_CHATTY) != 0)
                                    Log.d(TAG, "Registered content provider: " + names[j]
                                            + ", className = " + p.info.name + ", isSyncable = "
                                            + p.info.isSyncable);
                            }
                        } else {
                            PackageParser.Provider other = mProvidersByAuthority.get(names[j]);
                            Slog.w(TAG, "Skipping provider name " + names[j] +
                                    " (in package " + pkg.applicationInfo.packageName +
                                    "): name already used by "
                                    + ((other != null && other.getComponentName() != null)
                                            ? other.getComponentName().getPackageName() : "?"));
                        }
                    }
                }
                if ((policyFlags&PackageParser.PARSE_CHATTY) != 0) {
                    if (r == null) {
                        r = new StringBuilder(256);
                    } else {
                        r.append(' ');
                    }
                    r.append(p.info.name);
                }
            }
            if (r != null) {
                if (DEBUG_PACKAGE_SCANNING) Log.d(TAG, "  Providers: " + r);
            }

            // 处理该 Package 中的 Service 信息，添加到 PMS.mService 中！
            N = pkg.services.size();
            r = null;
            for (i=0; i<N; i++) {
                PackageParser.Service s = pkg.services.get(i);
                s.info.processName = fixProcessName(pkg.applicationInfo.processName,
                        s.info.processName, pkg.applicationInfo.uid);
                mServices.addService(s);
                if ((policyFlags&PackageParser.PARSE_CHATTY) != 0) {
                    if (r == null) {
                        r = new StringBuilder(256);
                    } else {
                        r.append(' ');
                    }
                    r.append(s.info.name);
                }
            }
            if (r != null) {
                if (DEBUG_PACKAGE_SCANNING) Log.d(TAG, "  Services: " + r);
            }

            // 处理该 Package 中的 BroadcastReceiver 信息,添加到 PMS.mReceivers 中。
            N = pkg.receivers.size();
            r = null;
            for (i=0; i<N; i++) {
                PackageParser.Activity a = pkg.receivers.get(i);
                a.info.processName = fixProcessName(pkg.applicationInfo.processName,
                        a.info.processName, pkg.applicationInfo.uid);
                mReceivers.addActivity(a, "receiver");
                if ((policyFlags&PackageParser.PARSE_CHATTY) != 0) {
                    if (r == null) {
                        r = new StringBuilder(256);
                    } else {
                        r.append(' ');
                    }
                    r.append(a.info.name);
                }
            }
            if (r != null) {
                if (DEBUG_PACKAGE_SCANNING) Log.d(TAG, "  Receivers: " + r);
            }
            
            // 处理该 Package 中的 activity 信息，添加到 PMS.mActivities 中！
            N = pkg.activities.size();
            r = null;
            for (i=0; i<N; i++) {
                PackageParser.Activity a = pkg.activities.get(i);
                a.info.processName = fixProcessName(pkg.applicationInfo.processName,
                        a.info.processName, pkg.applicationInfo.uid);
                mActivities.addActivity(a, "activity");
                if ((policyFlags&PackageParser.PARSE_CHATTY) != 0) {
                    if (r == null) {
                        r = new StringBuilder(256);
                    } else {
                        r.append(' ');
                    }
                    r.append(a.info.name);
                }
            }
            if (r != null) {
                if (DEBUG_PACKAGE_SCANNING) Log.d(TAG, "  Activities: " + r);
            }

            // 处理该 Package 中的 PermissionGroups 信息
            N = pkg.permissionGroups.size();
            r = null;
            for (i=0; i<N; i++) {
                PackageParser.PermissionGroup pg = pkg.permissionGroups.get(i);
                PackageParser.PermissionGroup cur = mPermissionGroups.get(pg.info.name);
                final String curPackageName = cur == null ? null : cur.info.packageName;
                final boolean isPackageUpdate = pg.info.packageName.equals(curPackageName);
                if (cur == null || isPackageUpdate) {
                    mPermissionGroups.put(pg.info.name, pg);
                    if ((policyFlags&PackageParser.PARSE_CHATTY) != 0) {
                        if (r == null) {
                            r = new StringBuilder(256);
                        } else {
                            r.append(' ');
                        }
                        if (isPackageUpdate) {
                            r.append("UPD:");
                        }
                        r.append(pg.info.name);
                    }
                } else {
                    Slog.w(TAG, "Permission group " + pg.info.name + " from package "
                            + pg.info.packageName + " ignored: original from "
                            + cur.info.packageName);
                    if ((policyFlags&PackageParser.PARSE_CHATTY) != 0) {
                        if (r == null) {
                            r = new StringBuilder(256);
                        } else {
                            r.append(' ');
                        }
                        r.append("DUP:");
                        r.append(pg.info.name);
                    }
                }
            }
            if (r != null) {
                if (DEBUG_PACKAGE_SCANNING) Log.d(TAG, "  Permission Groups: " + r);
            }
            
            // 处理该 Package 中的 Permissions 信息！
            N = pkg.permissions.size();
            r = null;
            for (i=0; i<N; i++) {
                PackageParser.Permission p = pkg.permissions.get(i);

                // Assume by default that we did not install this permission into the system.
                p.info.flags &= ~PermissionInfo.FLAG_INSTALLED;

                // Now that permission groups have a special meaning, we ignore permission
                // groups for legacy apps to prevent unexpected behavior. In particular,
                // permissions for one app being granted to someone just becase they happen
                // to be in a group defined by another app (before this had no implications).
                if (pkg.applicationInfo.targetSdkVersion > Build.VERSION_CODES.LOLLIPOP_MR1) {
                    p.group = mPermissionGroups.get(p.info.group);
                    // Warn for a permission in an unknown group.
                    if (p.info.group != null && p.group == null) {
                        Slog.w(TAG, "Permission " + p.info.name + " from package "
                                + p.info.packageName + " in an unknown group " + p.info.group);
                    }
                }

                // 从 Settings 中获得权限管理集合！
                ArrayMap<String, BasePermission> permissionMap =
                        p.tree ? mSettings.mPermissionTrees
                                : mSettings.mPermissions;

                // 获得 package 的权限对应的 BasePermission 对象！
                BasePermission bp = permissionMap.get(p.info.name);

                // Allow system apps to redefine non-system permissions
                // BasePermission 不为 null，且当前解析的 package 也不是权限的定义者！
                if (bp != null && !Objects.equals(bp.sourcePackage, p.info.packageName)) {
                
                    final boolean currentOwnerIsSystem = (bp.perm != null
                            && isSystemApp(bp.perm.owner));
                            
                    if (isSystemApp(p.owner)) 
                    
                        // 如果当前 package 申请的权限拥有者是系统 app
                        if (bp.type == BasePermission.TYPE_BUILTIN && bp.perm == null) {
                            // It's a built-in permission and no owner, take ownership now
                            bp.packageSetting = pkgSetting;
                            bp.perm = p;
                            bp.uid = pkg.applicationInfo.uid;
                            bp.sourcePackage = p.info.packageName;
                            p.info.flags |= PermissionInfo.FLAG_INSTALLED;
                            
                        } else if (!currentOwnerIsSystem) {
                            String msg = "New decl " + p.owner + " of permission  "
                                    + p.info.name + " is system; overriding " + bp.sourcePackage;
                            reportSettingsProblem(Log.WARN, msg);
                            bp = null;
                        }
                    }
                }

                if (bp == null) {
                    bp = new BasePermission(p.info.name, p.info.packageName,
                            BasePermission.TYPE_NORMAL);
                    permissionMap.put(p.info.name, bp);
                }

                if (bp.perm == null) {
                    if (bp.sourcePackage == null
                            || bp.sourcePackage.equals(p.info.packageName)) {
                        BasePermission tree = findPermissionTreeLP(p.info.name);
                        if (tree == null
                                || tree.sourcePackage.equals(p.info.packageName)) {
                            bp.packageSetting = pkgSetting;
                            bp.perm = p;
                            bp.uid = pkg.applicationInfo.uid;
                            bp.sourcePackage = p.info.packageName;
                            p.info.flags |= PermissionInfo.FLAG_INSTALLED;
                            if ((policyFlags&PackageParser.PARSE_CHATTY) != 0) {
                                if (r == null) {
                                    r = new StringBuilder(256);
                                } else {
                                    r.append(' ');
                                }
                                r.append(p.info.name);
                            }
                        } else {
                            Slog.w(TAG, "Permission " + p.info.name + " from package "
                                    + p.info.packageName + " ignored: base tree "
                                    + tree.name + " is from package "
                                    + tree.sourcePackage);
                        }
                    } else {
                        Slog.w(TAG, "Permission " + p.info.name + " from package "
                                + p.info.packageName + " ignored: original from "
                                + bp.sourcePackage);
                    }
                } else if ((policyFlags&PackageParser.PARSE_CHATTY) != 0) {
                    if (r == null) {
                        r = new StringBuilder(256);
                    } else {
                        r.append(' ');
                    }
                    r.append("DUP:");
                    r.append(p.info.name);
                }
                if (bp.perm == p) {
                    bp.protectionLevel = p.info.protectionLevel;
                }
            }

            if (r != null) {
                if (DEBUG_PACKAGE_SCANNING) Log.d(TAG, "  Permissions: " + r);
            }

            N = pkg.instrumentation.size();
            r = null;
            for (i=0; i<N; i++) {
                PackageParser.Instrumentation a = pkg.instrumentation.get(i);
                a.info.packageName = pkg.applicationInfo.packageName;
                a.info.sourceDir = pkg.applicationInfo.sourceDir;
                a.info.publicSourceDir = pkg.applicationInfo.publicSourceDir;
                a.info.splitSourceDirs = pkg.applicationInfo.splitSourceDirs;
                a.info.splitPublicSourceDirs = pkg.applicationInfo.splitPublicSourceDirs;
                a.info.dataDir = pkg.applicationInfo.dataDir;
                a.info.deviceProtectedDataDir = pkg.applicationInfo.deviceProtectedDataDir;
                a.info.credentialProtectedDataDir = pkg.applicationInfo.credentialProtectedDataDir;

                a.info.nativeLibraryDir = pkg.applicationInfo.nativeLibraryDir;
                a.info.secondaryNativeLibraryDir = pkg.applicationInfo.secondaryNativeLibraryDir;
                mInstrumentation.put(a.getComponentName(), a);
                if ((policyFlags&PackageParser.PARSE_CHATTY) != 0) {
                    if (r == null) {
                        r = new StringBuilder(256);
                    } else {
                        r.append(' ');
                    }
                    r.append(a.info.name);
                }
            }
            if (r != null) {
                if (DEBUG_PACKAGE_SCANNING) Log.d(TAG, "  Instrumentation: " + r);
            }
            
            // 处理该 Package 中的 protectedBroadcasts 信息！
            if (pkg.protectedBroadcasts != null) {
                N = pkg.protectedBroadcasts.size();
                for (i=0; i<N; i++) {
                    mProtectedBroadcasts.add(pkg.protectedBroadcasts.get(i));
                }
            }

            pkgSetting.setTimeStamp(scanFileTime);

            // Create idmap files for pairs of (packages, overlay packages).
            // Note: "android", ie framework-res.apk, is handled by native layers.
            if (pkg.mOverlayTarget != null) {
                // This is an overlay package.
                if (pkg.mOverlayTarget != null && !pkg.mOverlayTarget.equals("android")) {
                    if (!mOverlays.containsKey(pkg.mOverlayTarget)) {
                        mOverlays.put(pkg.mOverlayTarget,
                                new ArrayMap<String, PackageParser.Package>());
                    }
                    ArrayMap<String, PackageParser.Package> map = mOverlays.get(pkg.mOverlayTarget);
                    map.put(pkg.packageName, pkg);
                    PackageParser.Package orig = mPackages.get(pkg.mOverlayTarget);
                    if (orig != null && !createIdmapForPackagePairLI(orig, pkg)) {
                        createIdmapFailed = true;
                    }
                }
            } else if (mOverlays.containsKey(pkg.packageName) &&
                    !pkg.packageName.equals("android")) {
                // This is a regular package, with one or more known overlay packages.
                createIdmapsForPackageLI(pkg);
            }
        }

        Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);

        if (createIdmapFailed) {
            throw new PackageManagerException(INSTALL_FAILED_UPDATE_INCOMPATIBLE,
                    "scanPackageLI failed to createIdmap");
        }
        return pkg;
    }

```
### 1.3.1 Settings.getPackageLPw
上面又再次调用了 getPackageLPw 方法，注意根据参数传递：

- boolean add：传入的值为 false！
- boolean allowInstall：传入的值为 true；

```java
    // 最终进入这个方法！
    private PackageSetting getPackageLPw(String name, PackageSetting origPackage,
            String realName, SharedUserSetting sharedUser, File codePath, File resourcePath,
            String legacyNativeLibraryPathString, String primaryCpuAbiString,
            String secondaryCpuAbiString, int vc, int pkgFlags, int pkgPrivateFlags,
            UserHandle installUser, boolean add, boolean allowInstall, String parentPackage,
            List<String> childPackageNames) {
        
        // 尝试获得之前的安装数据！
        // 如果是非系统 apk，这个安装就是他自身的；如果是覆盖安装的系统 app，那么这个数据是 data 分区下的；
        PackageSetting p = mPackages.get(name);
        UserManagerService userManager = UserManagerService.getInstance();
        
        //【×】如果 p 不为 null，说明这个 data 分区的 apk 之前就存在，它可能是一个非系统 apk；
        // 也可能是一个系统的 apk 覆盖安装到了 data 分区！
        if (p != null) {
            p.primaryCpuAbiString = primaryCpuAbiString;
            p.secondaryCpuAbiString = secondaryCpuAbiString;
            if (childPackageNames != null) {
                p.childPackageNames = new ArrayList<>(childPackageNames);
            }

            // 对于扫描 data 分区的情况，如果 codePath 不匹配，会在【1.3】【B】的位置抛出异常，也就是说会忽略掉这个包！
            // 这里是不会进入的！！
            if (!p.codePath.equals(codePath)) {
                if ((p.pkgFlags & ApplicationInfo.FLAG_SYSTEM) != 0) {
                    Slog.w(PackageManagerService.TAG, "Trying to update system app code path from "
                            + p.codePathString + " to " + codePath.toString());
                } else {
                    Slog.i(PackageManagerService.TAG, "Package " + name + " codePath changed from "
                            + p.codePath + " to " + codePath + "; Retaining data and using new");

                    if ((pkgFlags & ApplicationInfo.FLAG_SYSTEM) != 0 &&
                            getDisabledSystemPkgLPr(name) == null) {
                        List<UserInfo> allUserInfos = getAllUsers();
                        if (allUserInfos != null) {
                            for (UserInfo userInfo : allUserInfos) {
                                p.setInstalled(true, userInfo.id);
                            }
                        }
                    }
                    p.legacyNativeLibraryPathString = legacyNativeLibraryPathString;
                }
            }
            
            if (p.sharedUser != sharedUser) { 
                // 如果共享 uid 不匹配，就需要创建新的 PackageSetting 替换以前的！
                PackageManagerService.reportSettingsProblem(Log.WARN,
                        "Package " + name + " shared user changed from "
                        + (p.sharedUser != null ? p.sharedUser.name : "<nothing>")
                        + " to "
                        + (sharedUser != null ? sharedUser.name : "<nothing>")
                        + "; replacing with new");
                p = null;
                
            } else {
                // 如果不是共享 uid，或者共享 uid 匹配，会进入这里：
                // 这里要注意：pkgFlags 是没有 FLAG_SYSTEM 标志位的，所以 pkgFlags & ApplicationInfo.FLAG_SYSTEM 
                // 会置空该标志，但是由于 | 的特性，并没有影响上一次安装信息的 pkgFlags 位！
                // 就是说如果是覆盖更新的 system app，即使扫描了 data 分区，也不会取消 FLAG_SYSTEM 位！
                p.pkgFlags |= pkgFlags & ApplicationInfo.FLAG_SYSTEM;
                p.pkgPrivateFlags |= pkgPrivateFlags & ApplicationInfo.PRIVATE_FLAG_PRIVILEGED;
            }
        }

        //【×】如果 p == null，
        // 说明这个 data 分区的 apk 是新增的，比如一些 OTA 版本会在 data 分区内置一些应用！
        // 或者是之前的数据和最新的扫描数据比较，共享 uid 不匹配，那就需要创建新的 PackageSetting！
        // 或者是 system 的 app 移动到了 data 分区！
        if (p == null) {
            if (origPackage != null) {
                p = new PackageSetting(origPackage.name, name, codePath, resourcePath,
                        legacyNativeLibraryPathString, primaryCpuAbiString, secondaryCpuAbiString,
                        null /* cpuAbiOverrideString */, vc, pkgFlags, pkgPrivateFlags,
                        parentPackage, childPackageNames);
    
                if (PackageManagerService.DEBUG_UPGRADE) Log.v(PackageManagerService.TAG, "Package "
                        + name + " is adopting original package " + origPackage.name);

                PackageSignatures s = p.signatures;
                p.copyFrom(origPackage);
                p.signatures = s;
                p.sharedUser = origPackage.sharedUser;
                p.appId = origPackage.appId;
                p.origPackage = origPackage;
                p.getPermissionsState().copyFrom(origPackage.getPermissionsState());
                
                // 重命名为源包的名字后，将新旧名字加入 mRenamedPackages 集合！
                mRenamedPackages.put(name, origPackage.name);
                name = origPackage.name;
    
                p.setTimeStamp(codePath.lastModified());

            } else {
                // 非系统 app 会进入这里，使用本次扫描的信息创建 PackageSetting！
                p = new PackageSetting(name, realName, codePath, resourcePath,
                        legacyNativeLibraryPathString, primaryCpuAbiString, secondaryCpuAbiString,
                        null /* cpuAbiOverrideString */, vc, pkgFlags, pkgPrivateFlags,
                        parentPackage, childPackageNames);

                p.setTimeStamp(codePath.lastModified());
                p.sharedUser = sharedUser;

                // 系统 apk 不会进入这个分支！
                if ((pkgFlags & ApplicationInfo.FLAG_SYSTEM) == 0) {
                    if (DEBUG_STOPPED) {
                        RuntimeException e = new RuntimeException("here");
                        e.fillInStackTrace();
                        Slog.i(PackageManagerService.TAG, "Stopping package " + name, e);
                    }
                    List<UserInfo> users = getAllUsers();
                    final int installUserId = installUser != null ? installUser.getIdentifier() : 0;
                    if (users != null && allowInstall) {
                        for (UserInfo user : users) {
                            final boolean installed = installUser == null
                                    || (installUserId == UserHandle.USER_ALL
                                        && !isAdbInstallDisallowed(userManager, user.id))
                                    || installUserId == user.id;
                            p.setUserState(user.id, 0, COMPONENT_ENABLED_STATE_DEFAULT,
                                    installed,
                                    true, // stopped,
                                    true, // notLaunched
                                    false, // hidden
                                    false, // suspended
                                    null, null, null,
                                    false, // blockUninstall
                                    INTENT_FILTER_DOMAIN_VERIFICATION_STATUS_UNDEFINED, 0);
                            writePackageRestrictionsLPr(user.id);
                        }
                    }
                }
                
                // 设置系统 package 的 appid
                // 如果是共享用户 id，那 appId 就是共享用户 id；
                // 如果不是共享用户 id，就看系统 app 是否被覆盖更新过，如果有，就克隆更新前的旧数据，初始化 appId，权限等等
                // 如果系统 app 没有被覆盖更新过，就创建新的 uid！
                if (sharedUser != null) {
                    p.appId = sharedUser.userId;
                    
                } else {
                    PackageSetting dis = mDisabledSysPackages.get(name);
                    if (dis != null) {
                    
                        if (dis.signatures.mSignatures != null) {
                            p.signatures.mSignatures = dis.signatures.mSignatures.clone();
                        }
                        p.appId = dis.appId;
                        
                        p.getPermissionsState().copyFrom(dis.getPermissionsState());
                        
                        List<UserInfo> users = getAllUsers();
                        if (users != null) {
                            for (UserInfo user : users) {
                                int userId = user.id;
                                p.setDisabledComponentsCopy(
                                        dis.getDisabledComponents(userId), userId);
                                p.setEnabledComponentsCopy(
                                        dis.getEnabledComponents(userId), userId);
                            }
                        }
                        
                        // 将这个新的 PackageSetting 根据 uid 添加到 mUserIds 或者 mOtherUserIds 中！
                        addUserIdLPw(p.appId, p, name);
                    } else {
                    
                        // 分配一个新的 uid，并将映射关系保存到 mUserIds 中！
                        p.appId = newUserIdLPw(p);
                    }
                }
            }

            if (p.appId < 0) {
                PackageManagerService.reportSettingsProblem(Log.WARN,
                        "Package " + name + " could not be assigned a valid uid");
                return null;
            }

            if (add) { // add 为 false，这里不添加，添加的操作在 PMS.scanPackageDirtyLI 方法中！
                addPackageSettingLPw(p, name, sharedUser);
            }
        } else {
            if (installUser != null && allowInstall) {
                // The caller has explicitly specified the user they want this
                // package installed for, and the package already exists.
                // Make sure it conforms to the new request.
                List<UserInfo> users = getAllUsers();
                if (users != null) {
                    for (UserInfo user : users) {
                        if ((installUser.getIdentifier() == UserHandle.USER_ALL
                                    && !isAdbInstallDisallowed(userManager, user.id))
                                || installUser.getIdentifier() == user.id) {
                            boolean installed = p.getInstalled(user.id);
                            if (!installed) {
                                p.setInstalled(true, user.id);
                                writePackageRestrictionsLPr(user.id);
                            }
                        }
                    }
                }
            }
        }
        
        // 返回查到或者是创建的新的系统 package 对应的 PackageSetting 对象！
        return p;
    }
```
整个过程其实还是很复杂的，要忽视很多的细节，抓住脉络才行！


# 2 Data 分区扫描结尾工作

下面我们来看看 Data 分区扫描的结尾工作！

```java
                //【1】处理那些已经不存在的被更新过的系统 app！
                for (String deletedAppName : possiblyDeletedUpdatedSystemApps) {
                    PackageParser.Package deletedPkg = mPackages.get(deletedAppName);
                    
                    mSettings.removeDisabledSystemPackageLPw(deletedAppName);

                    String msg;
                    if (deletedPkg == null) {
                        // 如果无扫描结果，那么系统中一已经没有该 apk 了，那就删掉它！
                        msg = "Updated system package " + deletedAppName
                                + " no longer exists; it's data will be wiped";
                    } else {
                        // 如果有扫描结果，说明 data 分区的 apk 还在，那么这个系统 apk 此时就已经是一个非系统 apk 了
                        // 那就要去掉 FLAG_SYSTEM 标志位！
                        msg = "Updated system app + " + deletedAppName
                                + " no longer present; removing system privileges for "
                                + deletedAppName;

                        deletedPkg.applicationInfo.flags &= ~ApplicationInfo.FLAG_SYSTEM;
                        PackageSetting deletedPs = mSettings.mPackages.get(deletedAppName);
                        deletedPs.pkgFlags &= ~ApplicationInfo.FLAG_SYSTEM;
                    }

                    logCriticalInfo(Log.WARN, msg);
                }

                //【2】确保所有在用户 data 分区的应用都显示出来了，如果 data 分区的无法显示，就显示 system 分区的！
                for (int i = 0; i < mExpectingBetter.size(); i++) {
                    final String packageName = mExpectingBetter.keyAt(i);
                    // 如果 PMS 仍然没有扫描到 mExpectingBetter 列表中的 apk，说明 data 分区的 apk 无法显示
                    // 那就要显示原来 system 分区的 apk！
                    if (!mPackages.containsKey(packageName)) {
                        final File scanFile = mExpectingBetter.valueAt(i);

                        logCriticalInfo(Log.WARN, "Expected better " + packageName
                                + " but never showed up; reverting to system");
                        //【2.1】根据目录，初始化解析标志！！！
                        int reparseFlags = mDefParseFlags;
                        if (FileUtils.contains(privilegedAppDir, scanFile)) {
                            reparseFlags = PackageParser.PARSE_IS_SYSTEM
                                    | PackageParser.PARSE_IS_SYSTEM_DIR
                                    | PackageParser.PARSE_IS_PRIVILEGED;
                                    
                        } else if (FileUtils.contains(systemAppDir, scanFile)) {
                            reparseFlags = PackageParser.PARSE_IS_SYSTEM
                                    | PackageParser.PARSE_IS_SYSTEM_DIR;
                                    
                        } else if (FileUtils.contains(vendorAppDir, scanFile)) {
                            reparseFlags = PackageParser.PARSE_IS_SYSTEM
                                    | PackageParser.PARSE_IS_SYSTEM_DIR;
                                    
                        } else if (FileUtils.contains(oemAppDir, scanFile)) {
                            reparseFlags = PackageParser.PARSE_IS_SYSTEM
                                    | PackageParser.PARSE_IS_SYSTEM_DIR;
                                    
                        } else {
                            Slog.e(TAG, "Ignoring unexpected fallback path " + scanFile);
                            continue;
                        }
                        //【2.2】使得 system 的 apk 可用；
                        mSettings.enableSystemPackageLPw(packageName);

                        try {
                            //【2.3】重新扫描 system 的 apk！
                            scanPackageTracedLI(scanFile, reparseFlags, scanFlags, 0, null);
                        } catch (PackageManagerException e) {
                            Slog.e(TAG, "Failed to parse original system package: "
                                    + e.getMessage());
                        }
                    }
                }
            }
            mExpectingBetter.clear(); // 清空 mExpectingBetter 因为已经处理完了！

            // 获得存储管理对象！
            mStorageManagerPackage = getStorageManagerPackageName();

            // Resolve protected action filters. Only the setup wizard is allowed to
            // have a high priority filter for these actions.
            // 获得开机向导应用
            mSetupWizardPackage = getSetupWizardPackageName();
            if (mProtectedFilters.size() > 0) {
                if (DEBUG_FILTERS && mSetupWizardPackage == null) {
                    Slog.i(TAG, "No setup wizard;"
                        + " All protected intents capped to priority 0");
                }
                for (ActivityIntentInfo filter : mProtectedFilters) {
                    if (filter.activity.info.packageName.equals(mSetupWizardPackage)) {
                        if (DEBUG_FILTERS) {
                            Slog.i(TAG, "Found setup wizard;"
                                + " allow priority " + filter.getPriority() + ";"
                                + " package: " + filter.activity.info.packageName
                                + " activity: " + filter.activity.className
                                + " priority: " + filter.getPriority());
                        }
                        // skip setup wizard; allow it to keep the high priority filter
                        continue;
                    }
                    Slog.w(TAG, "Protected action; cap priority to 0;"
                            + " package: " + filter.activity.info.packageName
                            + " activity: " + filter.activity.className
                            + " origPrio: " + filter.getPriority());
                    filter.setPriority(0);
                }
            }
            mDeferProtectedFilters = false;
            mProtectedFilters.clear();

            //【3】我们需要更新所有的应用，保证他们有正确的共享库路径。
            updateAllSharedLibrariesLPw();

            //【4】调整所有共享 uid 的 package 的指令集！
            for (SharedUserSetting setting : mSettings.getAllSharedUsersLPw()) {
                adjustCpuAbisForSharedUserLPw(setting.packages, null /* scanned package */,
                        false /* boot complete */);
            }

            //【5】更新所有 package 的最新使用时间！
            mPackageUsage.read(mPackages);
            mCompilerStats.read();
```

到这里，data 分区的扫描就结束了！！

接下来我们重点分析下：

```java
    //【5】更新所有 package 的最新使用时间！
    mPackageUsage.read(mPackages);
    mCompilerStats.read();
```

#3 PackageUsage 和 CompilerStats

在 PMS 的启动时，会创建 PackageUsage 和 CompilerStats 对象！
```java
    private final PackageUsage mPackageUsage = new PackageUsage();
    private final CompilerStats mCompilerStats = new CompilerStats();
```
PackageUsage 用来统计 Package 的使用信息！

```java
class PackageUsage extends AbstractStatsBase<Map<String, PackageParser.Package>> {
    PackageUsage() {
        super("package-usage.list", "PackageUsage_DiskWriter", /* lock */ true);
    }
    ... ... ...
}
```

PackageUsage 继承了抽象类 AbstractStatsBase，他会从 data/system/package-usage.list 中读取 package 的使用信息！


```
class CompilerStats extends AbstractStatsBase<Void> {
    public CompilerStats() {
        super("package-cstats.list", "CompilerStats_DiskWriter", /* lock */ false);
        packageStats = new HashMap<>();
    }
    ... ... ...
}
```
CompilerStats 同样继承了抽象类 AbstractStatsBase，他会从 data/system/package-cstats.list 中读取 package 的编译信息！

##3.1 PackageUsage.read

package-usage.list 的格式如下：包名 +　8 个时间戳
```xml
PACKAGE_USAGE__VERSION_1
com.android.providers.telephony 0 0 0 1531174607101 1531246438563 0 1531246438760 0
com.android.providers.calendar 0 1531246463197 0 1531246463181 1531246461258 0 0 0
```

我们来看看他是如何解析的！

read 方法会调用自身的 readInternal 方法！

###3.1.1 PackageUsage.readInternal

我们来看下 readInternal 的方法调用！
```java
    @Override
    protected void readInternal(Map<String, PackageParser.Package> packages) {
        AtomicFile file = getFile();
        BufferedInputStream in = null;
        try {
            in = new BufferedInputStream(file.openRead());
            StringBuffer sb = new StringBuffer();

            String firstLine = readLine(in, sb);
            if (firstLine == null) {
            } else if (USAGE_FILE_MAGIC_VERSION_1.equals(firstLine)) {
                //【3.1.2】根据第一行，我们会进入这里！
                readVersion1LP(packages, in, sb);
            } else {
                readVersion0LP(packages, in, sb, firstLine);
            }
        } catch (FileNotFoundException expected) {
            mIsHistoricalPackageUsageAvailable = false;
        } catch (IOException e) {
            Log.w(PackageManagerService.TAG, "Failed to read package usage times", e);
        } finally {
            IoUtils.closeQuietly(in);
        }
    }
```
###3.1.2 PackageUsage.readVersion1LP

我们来看下 readVersion1LP 的方法调用！
```java
    private void readVersion1LP(Map<String, PackageParser.Package> packages, InputStream in,
            StringBuffer sb) throws IOException {
        String line;
        while ((line = readLine(in, sb)) != null) {
            //【1】从 " " 将每一行分割开！
            String[] tokens = line.split(" ");
            if (tokens.length != PackageManager.NOTIFY_PACKAGE_USE_REASONS_COUNT + 1) {
                throw new IOException("Failed to parse " + line + " as a timestamp array.");
            }

            String packageName = tokens[0];
            PackageParser.Package pkg = packages.get(packageName);
            if (pkg == null) {
                continue;
            }
            
            //【2】将这 8 个时间戳保存到 PackageParser.Package.mLastPackageUsageTimeInMills 数组中！
            for (int reason = 0;
                    reason < PackageManager.NOTIFY_PACKAGE_USE_REASONS_COUNT;
                    reason++) {
                pkg.mLastPackageUsageTimeInMills[reason] = parseAsLong(tokens[reason + 1]);
            }
        }
    }
```
整个流程很简单！

##3.2 CompilerStats.read

package-cstats.list 的格式如下：
```xml
PACKAGE_MANAGER__COMPILER_STATS__1
com.android.storagemanager
-StorageManager.apk:1362
com.android.printspooler
-PrintSpooler.apk:659
```

我们来看看他是如何解析的，CompilerStats.read[0] 会调用 CompilerStats.read(Reader r) 方法！

###3.2.1 CompilerStats.read

CompilerStats 内部有一个 map 表 packageStats 保存了 packageName 和对应的 PackageStats 的映射关系！
```java
 private final Map<String, PackageStats> packageStats;
```

我们来继续看 read 方法：

```java
   public boolean read(Reader r) {
        synchronized (packageStats) {
            //【1】清空内部的 packageStats！
            packageStats.clear();
            try {
                BufferedReader in = new BufferedReader(r);

                //【2】版本号校验！
                String versionLine = in.readLine();
                if (versionLine == null) {
                    throw new IllegalArgumentException("No version line found.");
                } else {
                    if (!versionLine.startsWith(COMPILER_STATS_VERSION_HEADER)) {
                        throw new IllegalArgumentException("Invalid version line: " + versionLine);
                    }
                    int version = Integer.parseInt(
                            versionLine.substring(COMPILER_STATS_VERSION_HEADER.length()));
                    if (version != COMPILER_STATS_VERSION) {
                        // TODO: Upgrade older formats? For now, just reject and regenerate.
                        throw new IllegalArgumentException("Unexpected version: " + version);
                    }
                }

                //【3】创建第一行 package 对应的 PackageStats 对象！
                // 开始读取文件内容！
                PackageStats currentPackage = new PackageStats("fake package");

                String s = null;
                while ((s = in.readLine()) != null) {
                    //【3.1】"-" 开头的 line 是 PackageStats 要封装的信息！
                    if (s.startsWith("-")) {
                        int colonIndex = s.indexOf(':'); // 使用 ":" 将每一行分割开！
                        if (colonIndex == -1 || colonIndex == 1) {
                            throw new IllegalArgumentException("Could not parse data " + s);
                        }
                        String codePath = s.substring(1, colonIndex);
                        long time = Long.parseLong(s.substring(colonIndex + 1));
                        //【3.1.1】设置编译文件名和编译时间属性！
                        currentPackage.setCompileTime(codePath, time);
                    } else {
                        //【3.2.2】创建或者获得下一个 PackageStats 实例！
                        currentPackage = getOrCreatePackageStats(s);
                    }
                }
            } catch (Exception e) {
                Log.e(PackageManagerService.TAG, "Error parsing compiler stats", e);
                return false;
            }

            return true;
        }
    }
```
我们来看看 PackageStats 类：
```java
    static class PackageStats {
        private final String packageName;
        private final Map<String, Long> compileTimePerCodePath;

        public PackageStats(String packageName) {
            this.packageName = packageName;
            // We expect at least one element in here, but let's make it minimal.
            compileTimePerCodePath = new ArrayMap<>(2);
        }
```
packageName 是包名，compileTimePerCodePath 用来保存被编译的 code 文件名和编译的时间！


###3.2.2 CompilerStats.getOrCreatePackageStats

getOrCreatePackageStats 方法用来获得或者创建一个新的 PackageStats 对象！
```java
    public PackageStats getOrCreatePackageStats(String packageName) {
        synchronized (packageStats) {
            PackageStats existingStats = packageStats.get(packageName);
            if (existingStats != null) {
                return existingStats;
            }

            return createPackageStats(packageName);
        }
    }
    
    public PackageStats createPackageStats(String packageName) {
        synchronized (packageStats) {
            PackageStats newStats = new PackageStats(packageName);
            packageStats.put(packageName, newStats);
            return newStats;
        }
    }
```
整个方法流程很简单！