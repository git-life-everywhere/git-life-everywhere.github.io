# PMS 第 3 篇 - PMS_SYSTEM_SCAN_START 阶段

title: PMS 第 3 篇 - PMS_SYSTEM_SCAN_START 阶段
date: 2018/02/05 20:46:25
categories:
- AndroidFramework源码分析
- PackageManager包管理
tags: PackageManager包管理
---

[toc]

基于 Android 7.1.1 源码分析 PackageManagerService 的架构和逻辑实现，本文是作者原创，转载请说明出处！

# 0 综述
我们进入第二阶段系统目录扫描来分析，代码比较长，我们来回顾下该阶段的流程：
```java
            ... ... ... ...// 接上面
            long startTime = SystemClock.uptimeMillis();
             
            EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_PMS_SYSTEM_SCAN_START,
                    startTime);

            // 设置扫描参数！
            final int scanFlags = SCAN_NO_PATHS | SCAN_DEFER_DEX | SCAN_BOOTING | SCAN_INITIAL;
            
            // 获得环境变量：BOOTCLASSPATH 和 SYSTEMSERVERCLASSPATH！
            final String bootClassPath = System.getenv("BOOTCLASSPATH");
            final String systemServerClassPath = System.getenv("SYSTEMSERVERCLASSPATH");

            if (bootClassPath == null) {
                Slog.w(TAG, "No BOOTCLASSPATH found!");
            }

            if (systemServerClassPath == null) {
                Slog.w(TAG, "No SYSTEMSERVERCLASSPATH found!");
            }
            
            // 获得系统指令集合！
            final List<String> allInstructionSets = InstructionSets.getAllInstructionSets();
            final String[] dexCodeInstructionSets =
                    getDexCodeInstructionSets(
                            allInstructionSets.toArray(new String[allInstructionSets.size()]));

            // 对所有的共享库执行 odex 操作！
            if (mSharedLibraries.size() > 0) {
                for (String dexCodeInstructionSet : dexCodeInstructionSets) {
                    for (SharedLibraryEntry libEntry : mSharedLibraries.values()) {
                        final String lib = libEntry.path;
                        if (lib == null) {
                            continue;
                        }

                        try {
                            // Shared libraries do not have profiles so we perform a full
                            // AOT compilation (if needed).
                            int dexoptNeeded = DexFile.getDexOptNeeded(
                                    lib, dexCodeInstructionSet,
                                    getCompilerFilterForReason(REASON_SHARED_APK),
                                    false /* newProfile */);

                            if (dexoptNeeded != DexFile.NO_DEXOPT_NEEDED) {
                                mInstaller.dexopt(lib, Process.SYSTEM_UID, dexCodeInstructionSet,
                                        dexoptNeeded, DEXOPT_PUBLIC /*dexFlags*/,
                                        getCompilerFilterForReason(REASON_SHARED_APK),
                                        StorageManager.UUID_PRIVATE_INTERNAL,
                                        SKIP_SHARED_LIBRARY_CHECK);
                            }
                        } catch (FileNotFoundException e) {
                            Slog.w(TAG, "Library not found: " + lib);
                        } catch (IOException | InstallerException e) {
                            Slog.w(TAG, "Cannot dexopt " + lib + "; is it an APK or JAR? "
                                    + e.getMessage());
                        }
                    }
                }
            }

            // 获得目录 /system/framework，对其目录下的文件进行优化 !
            File frameworkDir = new File(Environment.getRootDirectory(), "framework");

            // 获得系统版本信息
            final VersionInfo ver = mSettings.getInternalVersion();

            // 判断是否是 OTA 升级，如果当前版本的指纹与历史版本的指纹信息不一致，表示当前版本是一次 OTA 升级上来更新版本！
            mIsUpgrade = !Build.FINGERPRINT.equals(ver.fingerprint);

            // 判断是否是从 Android M 之前的版本升级过来的，如果是就需要把系统 app 的权限从安装时提高到运行时！
            mPromoteSystemApps =
                    mIsUpgrade && ver.sdkVersion <= Build.VERSION_CODES.LOLLIPOP_MR1;

            // When upgrading from pre-N, we need to handle package extraction like first boot,
            // as there is no profiling data available.
            // 判断是否是从 Android 7.0 升级过来的！
            mIsPreNUpgrade = mIsUpgrade && ver.sdkVersion < Build.VERSION_CODES.N;

            mIsPreNMR1Upgrade = mIsUpgrade && ver.sdkVersion < Build.VERSION_CODES.N_MR1;

            // 保存从 Android 6.0 升级前已经存在的系统应用，并对他们进行优先扫描！
            // 扫描过程会将安装时权限变为运行时权限！
            if (mPromoteSystemApps) {
                Iterator<PackageSetting> pkgSettingIter = mSettings.mPackages.values().iterator();
                while (pkgSettingIter.hasNext()) {
                    PackageSetting ps = pkgSettingIter.next();
                    if (isSystemApp(ps)) {
                        mExistingSystemPackages.add(ps.name);
                    }
                }
            }

            //【1】扫描收集目录 /vendor/overlay 下的供应商应用包！
            File vendorOverlayDir = new File(VENDOR_OVERLAY_DIR);
            scanDirTracedLI(vendorOverlayDir, mDefParseFlags
                    | PackageParser.PARSE_IS_SYSTEM
                    | PackageParser.PARSE_IS_SYSTEM_DIR
                    | PackageParser.PARSE_TRUSTED_OVERLAY, scanFlags | SCAN_TRUSTED_OVERLAY, 0);

            // Find base frameworks (resource packages without code).
            //【2】扫描收集目录 /system/framework 下的应用包！ 
            scanDirTracedLI(frameworkDir, mDefParseFlags
                    | PackageParser.PARSE_IS_SYSTEM
                    | PackageParser.PARSE_IS_SYSTEM_DIR
                    | PackageParser.PARSE_IS_PRIVILEGED,
                    scanFlags | SCAN_NO_DEX, 0);

            //【3】扫描收集目录 /system/priv-app 下的应用包！ 
            final File privilegedAppDir = new File(Environment.getRootDirectory(), "priv-app");
            scanDirTracedLI(privilegedAppDir, mDefParseFlags
                    | PackageParser.PARSE_IS_SYSTEM
                    | PackageParser.PARSE_IS_SYSTEM_DIR
                    | PackageParser.PARSE_IS_PRIVILEGED, scanFlags, 0);

            //【4】扫描收集目录 /system/app 下的应用包！ 
            final File systemAppDir = new File(Environment.getRootDirectory(), "app");
            scanDirTracedLI(systemAppDir, mDefParseFlags
                    | PackageParser.PARSE_IS_SYSTEM
                    | PackageParser.PARSE_IS_SYSTEM_DIR, scanFlags, 0);

            //【5】扫描收集目录 /vendor/app 下的应用包！ 
            File vendorAppDir = new File("/vendor/app");
            try {
                vendorAppDir = vendorAppDir.getCanonicalFile();
            } catch (IOException e) {
                // failed to look up canonical path, continue with original one
            }
            scanDirTracedLI(vendorAppDir, mDefParseFlags
                    | PackageParser.PARSE_IS_SYSTEM
                    | PackageParser.PARSE_IS_SYSTEM_DIR, scanFlags, 0);

            //【6】扫描收集目录 /oem/app 下的应用包！ 
            final File oemAppDir = new File(Environment.getOemDirectory(), "app");
            scanDirTracedLI(oemAppDir, mDefParseFlags
                    | PackageParser.PARSE_IS_SYSTEM
                    | PackageParser.PARSE_IS_SYSTEM_DIR, scanFlags, 0);

            // 收集可能已经被删掉的系统应用包！
            final List<String> possiblyDeletedUpdatedSystemApps = new ArrayList<String>();

            if (!mOnlyCore) {
                // 遍历上一次安装的信息！
                Iterator<PackageSetting> psit = mSettings.mPackages.values().iterator();
                while (psit.hasNext()) {
                    PackageSetting ps = psit.next();

                    // 如果不是系统应用，跳过！
                    if ((ps.pkgFlags & ApplicationInfo.FLAG_SYSTEM) == 0) {
                        continue;
                    }

                    final PackageParser.Package scannedPkg = mPackages.get(ps.name);
                    if (scannedPkg != null) {
                        // 如果系统应用包不仅被扫描过（在mPackages中），并且在不可用列表中！
                        // 说明一定是通过覆盖更新的，移除之前扫描的结果，保证之前用户安装的应用能够被扫描！
                        if (mSettings.isDisabledSystemPackageLPr(ps.name)) {
                            logCriticalInfo(Log.WARN, "Expecting better updated system app for "
                                    + ps.name + "; removing system app.  Last known codePath="
                                    + ps.codePathString + ", installStatus=" + ps.installStatus
                                    + ", versionCode=" + ps.versionCode + "; scanned versionCode="
                                    + scannedPkg.mVersionCode);
                            
                            // 将之前的扫描结果移除！
                            removePackageLI(scannedPkg, true);
                            // 将这包添加到 mExpectingBetter 列表中！
                            mExpectingBetter.put(ps.name, ps.codePath);
                        }
                        
                        // 跳出循环，确保不会被删掉！
                        continue;
                    }

                    // 如果系统应用包没有被扫描，并且他也不在不可用的列表中，移除它，这个包不存在！
                    if (!mSettings.isDisabledSystemPackageLPr(ps.name)) {
                        psit.remove();
                        logCriticalInfo(Log.WARN, "System package " + ps.name
                                + " no longer exists; it's data will be wiped");
                        // Actual deletion of code and data will be handled by later
                        // reconciliation step

                    } else { 
                        
                        // 如果系统应用包没有被扫描，却在不可用的列表中，就将他加入到 
                        // possiblyDeletedUpdatedSystemApps 集合中，需要被删除！
                        final PackageSetting disabledPs = mSettings.getDisabledSystemPkgLPr(ps.name);
                        if (disabledPs.codePath == null || !disabledPs.codePath.exists()) {
                            possiblyDeletedUpdatedSystemApps.add(ps.name);
                        }

                    }
                }
            }

            // 清理所有安装不完全的应用包！
            ArrayList<PackageSetting> deletePkgsList = mSettings.getListOfIncompleteInstallPackagesLPr();
            for (int i = 0; i < deletePkgsList.size(); i++) {
                // Actual deletion of code and data will be handled by later
                // reconciliation step
                final String packageName = deletePkgsList.get(i).name;
                logCriticalInfo(Log.WARN, "Cleaning up incompletely installed app: " + packageName);
                synchronized (mPackages) {
                    mSettings.removePackageLPw(packageName);
                }
            }

            // 移除临时文件
            deleteTempPackageFiles();

            // 移除没有和应用程序包相关联的共享用户 id！
            mSettings.pruneSharedUsersLPw();
 
            ... ... ... ...// 见，第三阶段
```

主要流程如下：

- 对所有的共享库执行 odex 操作！
- 扫描手机系统目录信息！
- 收集那些可能已经不存在的系统应用包，在扫描完 data 分区后再处理！
- 清理所有安装不完全的应用包！

我们接下来继续分析：

# 1 对共享库执行 odex 操作

主要代码块如下：
```java
            // 对所有的共享库执行 odex 操作！
            if (mSharedLibraries.size() > 0) {
                for (String dexCodeInstructionSet : dexCodeInstructionSets) {
                    for (SharedLibraryEntry libEntry : mSharedLibraries.values()) {
                        final String lib = libEntry.path;
                        if (lib == null) {
                            continue;
                        }
                        try {
                            // Shared libraries do not have profiles so we perform a full
                            // AOT compilation (if needed).

                            // 判断共享库是否需要执行 odex 操作
                            int dexoptNeeded = DexFile.getDexOptNeeded(
                                    lib, dexCodeInstructionSet,
                                    getCompilerFilterForReason(REASON_SHARED_APK),
                                    false /* newProfile */);
                            
                            // 如果需要 odex 操作，对共享库进行一次预编译（AOT）
                            if (dexoptNeeded != DexFile.NO_DEXOPT_NEEDED) {
                                mInstaller.dexopt(lib, Process.SYSTEM_UID, dexCodeInstructionSet,
                                        dexoptNeeded, DEXOPT_PUBLIC /*dexFlags*/,
                                        getCompilerFilterForReason(REASON_SHARED_APK),
                                        StorageManager.UUID_PRIVATE_INTERNAL,
                                        SKIP_SHARED_LIBRARY_CHECK);
                            }
                        } catch (FileNotFoundException e) {
                            Slog.w(TAG, "Library not found: " + lib);
                        } catch (IOException | InstallerException e) {
                            Slog.w(TAG, "Cannot dexopt " + lib + "; is it an APK or JAR? "
                                    + e.getMessage());
                        }
                    }
                }
            }
```
主要流程：

- 判断共享库是否需要执行 odex 操作;
- 如果需要执行 odex 操作，就对共享库进行预（AOT）处理;

# 2 系统目录扫描 - 开始阶段

主要代码如下：
```java
            // 设置扫描参数！
            final int scanFlags = SCAN_NO_PATHS | SCAN_DEFER_DEX | SCAN_BOOTING | SCAN_INITIAL;

            ... ... ... ...

            // 扫描收集目录 /vendor/overlay 下的供应商应用包，用于资源替换！！
            File vendorOverlayDir = new File(VENDOR_OVERLAY_DIR);
            scanDirTracedLI(vendorOverlayDir, mDefParseFlags
                    | PackageParser.PARSE_IS_SYSTEM
                    | PackageParser.PARSE_IS_SYSTEM_DIR
                    | PackageParser.PARSE_TRUSTED_OVERLAY, scanFlags | SCAN_TRUSTED_OVERLAY, 0);

            // Find base frameworks (resource packages without code).
            // 扫描收集目录 /system/framework 下的应用包！ 
            scanDirTracedLI(frameworkDir, mDefParseFlags
                    | PackageParser.PARSE_IS_SYSTEM
                    | PackageParser.PARSE_IS_SYSTEM_DIR
                    | PackageParser.PARSE_IS_PRIVILEGED,
                    scanFlags | SCAN_NO_DEX, 0);

            // 扫描收集目录 /system/priv-app 下的应用包！ 
            final File privilegedAppDir = new File(Environment.getRootDirectory(), "priv-app");
            scanDirTracedLI(privilegedAppDir, mDefParseFlags
                    | PackageParser.PARSE_IS_SYSTEM
                    | PackageParser.PARSE_IS_SYSTEM_DIR
                    | PackageParser.PARSE_IS_PRIVILEGED, scanFlags, 0);

            // 扫描收集目录 /system/app 下的应用包！ 
            final File systemAppDir = new File(Environment.getRootDirectory(), "app");
            scanDirTracedLI(systemAppDir, mDefParseFlags
                    | PackageParser.PARSE_IS_SYSTEM
                    | PackageParser.PARSE_IS_SYSTEM_DIR, scanFlags, 0);

            // 扫描收集目录 /vendor/app 下的应用包！ 
            File vendorAppDir = new File("/vendor/app");
            try {
                vendorAppDir = vendorAppDir.getCanonicalFile();
            } catch (IOException e) {
                // failed to look up canonical path, continue with original one
            }
            scanDirTracedLI(vendorAppDir, mDefParseFlags
                    | PackageParser.PARSE_IS_SYSTEM
                    | PackageParser.PARSE_IS_SYSTEM_DIR, scanFlags, 0);
                    
            // 扫描收集目录 /oem/app 下的应用包！ 
            final File oemAppDir = new File(Environment.getOemDirectory(), "app");
            scanDirTracedLI(oemAppDir, mDefParseFlags
                    | PackageParser.PARSE_IS_SYSTEM
                    | PackageParser.PARSE_IS_SYSTEM_DIR, scanFlags, 0);
```

扫描参数的设置为：int scanFlags = SCAN_NO_PATHS | SCAN_DEFER_DEX | SCAN_BOOTING | SCAN_INITIAL;

按照顺序扫描的目录有：

- **/vendor/overlay**
- **/system/framework**
- **/system/priv-app**
- **/system/app**
- **/vendor/app**
- **/oem/app**

调用 PMS.scanDirTracedLI 进行扫描，需要注意的是被扫描目录的顺序，这个顺序意味着：先被扫描到的文件，就是最终被用到的文件。

下面我们以 /system/app 目录为例，跟踪系统路径扫描的全过程！！

## 2.1 PMS.scanDirTracedLI

调用 scanDirTracedLI 方法进行目录扫描：
```java
    private void scanDirTracedLI(File dir, final int parseFlags, int scanFlags, long currentTime) {
        Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "scanDir");
        try {
            
            //【2.2】进一步调用 scanDirLI 方法！
            scanDirLI(dir, parseFlags, scanFlags, currentTime);

        } finally {
            Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
        }
    }
```
## 2.2 PMS.scanDirLI
```java
    private void scanDirLI(File dir, final int parseFlags, int scanFlags, long currentTime) {
        
        // 获得 dir 目录中的文件集合！
        final File[] files = dir.listFiles();
        if (ArrayUtils.isEmpty(files)) {
            Log.d(TAG, "No files in app dir " + dir);
            return;
        }

        if (DEBUG_PACKAGE_SCANNING) {
            Log.d(TAG, "Scanning app dir " + dir + " scanFlags=" + scanFlags
                    + " flags=0x" + Integer.toHexString(parseFlags));
        }
        
        // 遍历 dir 目录下的所有文件！
        for (File file : files) {
            //【1】判断 file 是否是 Package，必须要同时满足下面 2 个条件：
            // 1、file 以 ".apk" 结尾或者 file 是一个目录；
            // 2、file 不是存储类型的文件；
            final boolean isPackage = (isApkFile(file) || file.isDirectory())
                    && !PackageInstallerService.isStageName(file.getName());
            if (!isPackage) {
                continue;
            }
            try {
            
                //【2.3】如果 file 是 package 类型，继续调用 scanPackageTracedLI 进行扫描！
                // 解析参数增加：必须是 apk！
                scanPackageTracedLI(file, parseFlags | PackageParser.PARSE_MUST_BE_APK,
                        scanFlags, currentTime, null);
 
            } catch (PackageManagerException e) {
                Slog.w(TAG, "Failed to parse " + file + ": " + e.getMessage());

                 // 如果扫描 data 分区的 Apk 失败，则删除 data 分区扫描失败的文件
                if ((parseFlags & PackageParser.PARSE_IS_SYSTEM) == 0 &&
                        e.error == PackageManager.INSTALL_FAILED_INVALID_APK) {
                    logCriticalInfo(Log.WARN, "Deleting invalid package at " + file);
                    removeCodePathLI(file);
                }
            }
        }
    }
```
注意这里的 try catch 语句，后面方法抛出的异常都是在这个进行 catch 的！

判断 file 是否是 Package，必须要同时满足下面 2 个条件：

- file 以 ".apk" 结尾或者 file 是一个目录；
- file 不是存储类型的文件：
   - file 不以 vmdl 开头且不以 .tmp 结尾；
   - file 不以 smdl 开头且不以 .tmp 结尾；
   - file 不以 smdl2tmp 开头；
   
## 2.3 PMS.scanPackageTracedLI
```java
    private PackageParser.Package scanPackageTracedLI(File scanFile, final int parseFlags,
            int scanFlags, long currentTime, UserHandle user) throws PackageManagerException {
        Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "scanPackage");
        try {

            //【2.4】继续调用 scanPackageLI，执行扫描解析！
            return scanPackageLI(scanFile, parseFlags, scanFlags, currentTime, user);
        } finally {
            Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
        }
    }
```

继续：

## 2.4 PMS.scanPackageLI
```java
    private PackageParser.Package scanPackageLI(File scanFile, int parseFlags, int scanFlags,
            long currentTime, UserHandle user) throws PackageManagerException {
        if (DEBUG_INSTALL) Slog.d(TAG, "Parsing: " + scanFile);
        
        //【1】创建一个 PackageParser 对象，用来执行解析操作！
        PackageParser pp = new PackageParser();
        pp.setSeparateProcesses(mSeparateProcesses);
        pp.setOnlyCoreApps(mOnlyCore); // mOnlyCore 为 false！
        pp.setDisplayMetrics(mMetrics);

        if ((scanFlags & SCAN_TRUSTED_OVERLAY) != 0) {
            parseFlags |= PackageParser.PARSE_TRUSTED_OVERLAY;
        }

        Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "parsePackage");
        
        //【2】PackageParser.Package 用来保存解析结果！
        final PackageParser.Package pkg;
        try {
            
            //【3.1】开始扫描解析 scanFile，返回解析结果！
            pkg = pp.parsePackage(scanFile, parseFlags);
            
        } catch (PackageParserException e) {
            throw PackageManagerException.from(e);
        } finally {
            Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
        }
        
        //【3.2】继续扫描！
        return scanPackageLI(pkg, scanFile, paraseFlags, scanFlags, currentTime, user);
    }
```

下面我们来分析扫描的过程！

# 3 系统目录扫描 - 扫描阶段

下面，我把 PackageParser 简称为 PParser！

## 3.1 PParser.parsePackage

参数 flags 为 parseFlags，不同的目录取值不同：

```java
    public Package parsePackage(File packageFile, int flags) throws PackageParserException {
        if (packageFile.isDirectory()) {
            //【3.1.1】Android 5.0 以后进入下面的分支！
            return parseClusterPackage(packageFile, flags);
        } else {  
            //【3.1.2】Android 5.0 以前进入下面的分支！
            return parseMonolithicPackage(packageFile, flags);
        }
    }
          
```
这里有 2 中解析方式：

- 如果 packageFile 是一个目录的话，比如 /system/app/TimeService，那就采用第一种解析方式！
- 如果 packageFile 是一个非目录文件的话，比如 /system/app/MyService.apk，那就采用第二种解析方式！

这里给大家解释一下：

Android 5.0 以前，apk 都是直接位于 app 目录下的，比如：/system/app/MyService.apk；

从 5.0 以后，谷歌引入了 apk 拆分机制，就是支持将一个 apk 拆分成很多个具有相同签名的子 apk，为了支持 apk 拆分，谷歌增加了一级目录：/system/app/MyService/，这个目录里会存放一到多个 apk，比如 base.apk，split.apk；

pms 在解析 package 时，会把多个 apk 的数据封装成一个 Package，加载到内存中！！

### 3.1.1 PParser.parseClusterPackage

```java
    private Package parseClusterPackage(File packageDir, int flags) throws PackageParserException {
    
        //【3.1.1.1】继续调用 parseClusterPackageLite 对 packageDir 目录下的！
        final PackageLite lite = parseClusterPackageLite(packageDir, 0);

        // 根据前面的解析结果，判断是否有核心的应用，如果 lite 中没有核心应用，退出！
        if (mOnlyCoreApps && !lite.coreApp) {
            throw new PackageParserException(INSTALL_PARSE_FAILED_MANIFEST_MALFORMED,
                    "Not a coreApp: " + packageDir);
        }

        final AssetManager assets = new AssetManager();
        try {

            // 装载应用程序的资源到 AssetManager！
            loadApkIntoAssetManager(assets, lite.baseCodePath, flags);

            if (!ArrayUtils.isEmpty(lite.splitCodePaths)) { // 如果 apk 被拆分了，加载子 apk 的资源！
                for (String path : lite.splitCodePaths) {
                    loadApkIntoAssetManager(assets, path, flags);
                }
            }
            
            //【3.1.1.2】对核心 apk 解析，返回封装核心 apk 信息的 Package 对象 pkg!
            final File baseApk = new File(lite.baseCodePath);
            final Package pkg = parseBaseApk(baseApk, assets, flags);

            if (pkg == null) {
                throw new PackageParserException(INSTALL_PARSE_FAILED_NOT_APK,
                        "Failed to parse base APK: " + baseApk);
            }

            // 如果 apk 被拆分了，那就对非核心 apk 也进行处理！
            if (!ArrayUtils.isEmpty(lite.splitNames)) {
                final int num = lite.splitNames.length;
                pkg.splitNames = lite.splitNames;
                pkg.splitCodePaths = lite.splitCodePaths;
                pkg.splitRevisionCodes = lite.splitRevisionCodes;
                pkg.splitFlags = new int[num];
                pkg.splitPrivateFlags = new int[num];

                for (int i = 0; i < num; i++) {
                    //【3.1.1.3】对拆分后的非核心 apk 也进行解析！
                    parseSplitApk(pkg, i, assets, flags);
                }
            }

            pkg.setCodePath(packageDir.getAbsolutePath()); // 设置 package 的 codePath！
            pkg.setUse32bitAbi(lite.use32bitAbi);
            
            // 返回解析对象！
            return pkg;
        } finally {
            IoUtils.closeQuietly(assets);
        }
    }

```
这里我们分为 3 个阶段来看：

- 对目录下的所有 apk 进行整体解析；
- 对目录中的核心 apk 进行解析；
- 如果 apk 被拆分了，对非核心应用进行解析；


#### 3.1.1.1 PParser.parseClusterPackageLite
首先，对目录下的所有 apk 进行整体解析：
```java
    private static PackageLite parseClusterPackageLite(File packageDir, int flags)
            throws PackageParserException {

        final File[] files = packageDir.listFiles();
        if (ArrayUtils.isEmpty(files)) {
            throw new PackageParserException(INSTALL_PARSE_FAILED_NOT_APK,
                    "No packages found in split");
        }

        String packageName = null;
        int versionCode = 0;

        final ArrayMap<String, ApkLite> apks = new ArrayMap<>();
        
        //【1】遍历目录下的所有 apk 文件，包括主 apk 和子 apk！
        for (File file : files) {
            if (isApkFile(file)) {
                
                //【3.1.1.1.1】解析 apk，将解析数据保存到 ApkLite 中！
                final ApkLite lite = parseApkLite(file, flags);

                if (packageName == null) {
                    packageName = lite.packageName;
                    versionCode = lite.versionCode;

                } else {
                    if (!packageName.equals(lite.packageName)) {
                        throw new PackageParserException(INSTALL_PARSE_FAILED_BAD_MANIFEST,
                                "Inconsistent package " + lite.packageName + " in " + file
                                + "; expected " + packageName);
                    }
                    if (versionCode != lite.versionCode) {
                        throw new PackageParserException(INSTALL_PARSE_FAILED_BAD_MANIFEST,
                                "Inconsistent version " + lite.versionCode + " in " + file
                                + "; expected " + versionCode);
                    }
                }

                // 将解析得到的 ApkLite 对象添加到一个 ArrayMap 集合中！
                // 注意：对于核心 apk，splitName 为 null！
                if (apks.put(lite.splitName, lite) != null) {
                    throw new PackageParserException(INSTALL_PARSE_FAILED_BAD_MANIFEST,
                            "Split name " + lite.splitName
                            + " defined more than once; most recent was " + file);
                }
            }
        }

        //【2】从列表中移除核心 apk，保存到 baseApk 中，之所以 remove null，是因为 base apk 的 splitName 为 null！
        // 如果没有 base apk，那就报错！！
        final ApkLite baseApk = apks.remove(null);
        if (baseApk == null) {
            throw new PackageParserException(INSTALL_PARSE_FAILED_BAD_MANIFEST,
                    "Missing base APK in " + packageDir);
        }

        //【3】处理非核心的 apk！
        final int size = apks.size();
        String[] splitNames = null;
        String[] splitCodePaths = null;
        int[] splitRevisionCodes = null;
        if (size > 0) {
        
            //【3.1】获得非核心 apk 的 splitName，codePath 和 splitRevisionCodes！
            splitNames = new String[size];
            splitCodePaths = new String[size];
            splitRevisionCodes = new int[size];

            splitNames = apks.keySet().toArray(splitNames);
            Arrays.sort(splitNames, sSplitNameComparator);

            for (int i = 0; i < size; i++) {
                splitCodePaths[i] = apks.get(splitNames[i]).codePath;
                splitRevisionCodes[i] = apks.get(splitNames[i]).revisionCode;
            }
        }

        final String codePath = packageDir.getAbsolutePath();
        
        //【3.1.1.1.2】创建所有应用的 PackageLite 解析对象，返回！
        return new PackageLite(codePath, baseApk, splitNames, splitCodePaths,
                splitRevisionCodes);
    }
```
这里调用了 parseApkLite 方法，解析 dir 下的核心 apk 和非核心 apk（如果有），然后获得非核心 apk 的 splitNames、splitCodePaths 和 splitRevisionCodes，最后创建 package 的 PackageLite 对象，并返回！

##### 3.1.1.1.1 PParser.parseApkLite\[2]
解析 apk 文件！
```java
    public static ApkLite parseApkLite(File apkFile, int flags)
            throws PackageParserException {
        final String apkPath = apkFile.getAbsolutePath();

        AssetManager assets = null;
        XmlResourceParser parser = null;

        try {
        
            // 创建资源管理器
            assets = new AssetManager(); 
            assets.setConfiguration(0, 0, null, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
                    Build.VERSION.RESOURCES_SDK_INT);

            // 首先将 apk 中的资源加载到资源管理器 asserts 中。 
            int cookie = assets.addAssetPath(apkPath);
            if (cookie == 0) {
                throw new PackageParserException(INSTALL_PARSE_FAILED_NOT_APK,
                        "Failed to parse " + apkPath);
            }

            final DisplayMetrics metrics = new DisplayMetrics();
            metrics.setToDefaults();

            final Resources res = new Resources(assets, metrics, null);
            
            //【1】创建解析器，用来解析 apk 的 AndroidMenifest.xml 文件。
            parser = assets.openXmlResourceParser(cookie, ANDROID_MANIFEST_FILENAME);

            final Signature[] signatures;
            final Certificate[][] certificates;

            if ((flags & PARSE_COLLECT_CERTIFICATES) != 0) {
                // TODO: factor signature related items out of Package object
                final Package tempPkg = new Package(null);
                Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "collectCertificates");
                try {
                    
                    // 进行签名验证，并将签名保存在 signatures 中。
                    collectCertificates(tempPkg, apkFile, 0 /*parseFlags*/);
                } finally {
                    Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
                }
                signatures = tempPkg.mSignatures;
                certificates = tempPkg.mCertificates;
            } else {
                signatures = null;
                certificates = null;
            }

            // 将 <manifest> 标签的属性付给 attrs！
            final AttributeSet attrs = parser;
            
            //【3.1.1.1.1.1】接着调用 parseApkLite 继续解析！
            return parseApkLite(apkPath, res, parser, attrs, flags, signatures, certificates);

        } catch (XmlPullParserException | IOException | RuntimeException e) {
            throw new PackageParserException(INSTALL_PARSE_FAILED_UNEXPECTED_EXCEPTION,
                    "Failed to parse " + apkPath, e);
        } finally {
            IoUtils.closeQuietly(parser);
            IoUtils.closeQuietly(assets);
        }
    }

```

调用重载函数 parseApkLite 函数，继续解析，参数传递：

###### 3.1.1.1.1.1 PParser.parseApkLite\[7]

```java
    private static ApkLite parseApkLite(String codePath, Resources res, XmlPullParser parser,
                    AttributeSet attrs, int flags, Signature[] signatures, Certificate[][] certificates)
                    throws IOException, XmlPullParserException, PackageParserException {
        //【3.1.1.1.1.2】解析获得应用的 
        final Pair<String, String> packageSplit = parsePackageSplitNames(parser, attrs);

        int installLocation = PARSE_DEFAULT_INSTALL_LOCATION;
        int versionCode = 0;
        int revisionCode = 0;
        boolean coreApp = false;
        boolean multiArch = false;
        boolean use32bitAbi = false;
        boolean extractNativeLibs = true;

        //【1】解析 <manifest> 标签的属性！
        for (int i = 0; i < attrs.getAttributeCount(); i++) {
            final String attr = attrs.getAttributeName(i);
            // 解析 android:installLocation 属性，表示安装位置，取值为 auto|internalOnly|preferExternal
            if (attr.equals("installLocation")) {
                installLocation = attrs.getAttributeIntValue(i,
                        PARSE_DEFAULT_INSTALL_LOCATION);
    
            } else if (attr.equals("versionCode")) { // 解析版本号 versionCode！
                versionCode = attrs.getAttributeIntValue(i, 0);

            } else if (attr.equals("revisionCode")) {
                revisionCode = attrs.getAttributeIntValue(i, 0);

            } else if (attr.equals("coreApp")) { // 是否是核心 apk
                coreApp = attrs.getAttributeBooleanValue(i, false);

            }
        }

        //【2】接着，解析 manifest 标签的直接子标签！
        int type;
        final int searchDepth = parser.getDepth() + 1;

        final List<VerifierInfo> verifiers = new ArrayList<VerifierInfo>();

        while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
                && (type != XmlPullParser.END_TAG || parser.getDepth() >= searchDepth)) {
            if (type == XmlPullParser.END_TAG || type == XmlPullParser.TEXT) {
                continue;
            }

            //【2.1】解析 <package-verifier> 标签的属性！
            if (parser.getDepth() == searchDepth && "package-verifier".equals(parser.getName())) {
                final VerifierInfo verifier = parseVerifier(res, parser, attrs, flags);
                if (verifier != null) {
                    verifiers.add(verifier);
                }
            }

            //【2.2】解析 <application> 标签的属性！
            if (parser.getDepth() == searchDepth && "application".equals(parser.getName())) {
                for (int i = 0; i < attrs.getAttributeCount(); ++i) {
                    final String attr = attrs.getAttributeName(i);
                    if ("multiArch".equals(attr)) {
                        multiArch = attrs.getAttributeBooleanValue(i, false);
                    }
                    if ("use32bitAbi".equals(attr)) {
                        use32bitAbi = attrs.getAttributeBooleanValue(i, false);
                    }
                    if ("extractNativeLibs".equals(attr)) {
                        extractNativeLibs = attrs.getAttributeBooleanValue(i, true);
                    }
                }
            }
        }

        //【3.1.1.1.1.3】创建 apk 对应的 ApkLite 对象！！
        return new ApkLite(codePath, packageSplit.first, packageSplit.second, versionCode,
                revisionCode, installLocation, verifiers, signatures, certificates, coreApp,
                multiArch, use32bitAbi, extractNativeLibs);
    }

```
###### 3.1.1.1.1.2 PParser.parsePackageSplitNames

该方法用于解析 apk 的 “manifest” 标签的 package 属性和 split 属性！
```java
    private static Pair<String, String> parsePackageSplitNames(XmlPullParser parser,
            AttributeSet attrs) throws IOException, XmlPullParserException,
            PackageParserException {

        int type;
        while ((type = parser.next()) != XmlPullParser.START_TAG
                && type != XmlPullParser.END_DOCUMENT) {
        }

        if (type != XmlPullParser.START_TAG) {
            throw new PackageParserException(INSTALL_PARSE_FAILED_MANIFEST_MALFORMED,
                    "No start tag found");
        }
        if (!parser.getName().equals(TAG_MANIFEST)) { // 如果不是 manifest 标签！
            throw new PackageParserException(INSTALL_PARSE_FAILED_MANIFEST_MALFORMED,
                    "No <manifest> tag");
        }

        //【1】解析 manifest 标签的 package 属性！
        final String packageName = attrs.getAttributeValue(null, "package");
        if (!"android".equals(packageName)) {
             // 如果 package 不是 android，那需要校验格式！
            final String error = validateName(packageName, true, true);
            if (error != null) {
                throw new PackageParserException(INSTALL_PARSE_FAILED_BAD_PACKAGE_NAME,
                        "Invalid manifest package: " + error);
            }
        }

        //【2】解析 manifest 标签的 split 属性！
        String splitName = attrs.getAttributeValue(null, "split");
        if (splitName != null) {
            if (splitName.length() == 0) {
                splitName = null;
            } else {
                final String error = validateName(splitName, false, false);
                if (error != null) {
                    throw new PackageParserException(INSTALL_PARSE_FAILED_BAD_PACKAGE_NAME,
                            "Invalid manifest split: " + error);
                }
            }
        }
        //【3】返回 packageName 和 splitName 的 Pair 对象！
        return Pair.create(packageName.intern(),
                (splitName != null) ? splitName.intern() : splitName);
    }

```

接着创建一个 ApkLite 对象，用来封装 apk 的解析信息！

###### 3.1.1.1.1.3 new ApkLite

最后，将结果封装为一个 ApkLite 对象，并返回！
```java
    public static class ApkLite {
        public final String codePath;     // apk 路径，就是 apk 文件的绝对路径！
        public final String packageName;  // 应用程序包名
        public final String splitName;    // 非核心 apk 包名
        public final int versionCode;     // 版本号
        public final int revisionCode;    // 修订版本号
        public final int installLocation; // 安装位置
        public final VerifierInfo[] verifiers; // 校验信息
        public final Signature[] signatures;  // 签名信息
        public final Certificate[][] certificates; // 证书信息
        public final boolean coreApp;     // 是否是核心应用
        public final boolean multiArch;   // 是否支持多软件架构
        public final boolean use32bitAbi; // 是否使用 32 位系统指令集
        public final boolean extractNativeLibs; // 是否依赖额外的本地库

        public ApkLite(String codePath, String packageName, String splitName, int versionCode,
                int revisionCode, int installLocation, List<VerifierInfo> verifiers,
                Signature[] signatures, Certificate[][] certificates, boolean coreApp,
                boolean multiArch, boolean use32bitAbi, boolean extractNativeLibs) {
            this.codePath = codePath;
            this.packageName = packageName;
            this.splitName = splitName;
            this.versionCode = versionCode;
            this.revisionCode = revisionCode;
            this.installLocation = installLocation;
            this.verifiers = verifiers.toArray(new VerifierInfo[verifiers.size()]);
            this.signatures = signatures;
            this.certificates = certificates;
            this.coreApp = coreApp;
            this.multiArch = multiArch;
            this.use32bitAbi = use32bitAbi;
            this.extractNativeLibs = extractNativeLibs;
        }
    }
```
然后返回，回到 PP.parseClusterPackageLite 函数中，根据解析的结果，创建 PackageLite 对象：


##### 3.1.1.1.2 new PackageLite
PackageLite 用来封装一个应用程序包的信息，PackageLite 的构造器如下：

```java
    public static class PackageLite {
        public final String packageName; // base apk 应用程序包名；
        public final int versionCode;    // base apk 的版本号；
        public final int installLocation;  // base apk 的安装位置；
        public final VerifierInfo[] verifiers; // base apk 的安装位置；
        
        public final String[] splitNames; // 非核心 apk 的名称；

        public final String codePath;  // 应用程序包的绝对路径，/system/app/com.coolqi.papapa

        public final String baseCodePath;   // base apk 的路径，/system/app/com.coolqi.papapa/base.apk
        public final String[] splitCodePaths;  //非核心 apk 的路径

        public final int baseRevisionCode; // 核心 apk 的修订版版本号
        public final int[] splitRevisionCodes; // 非核心 apk 的修订版版本号

        public final boolean coreApp; // 是否有核心 apk！
        public final boolean multiArch; // 是否支持多软件平台架构！
        public final boolean use32bitAbi; // 是否使用 23 位的系统指令集！
        public final boolean extractNativeLibs; // 是否支持！

        public PackageLite(String codePath, ApkLite baseApk, String[] splitNames,
                String[] splitCodePaths, int[] splitRevisionCodes) {

            this.packageName = baseApk.packageName;
            this.versionCode = baseApk.versionCode;
            this.installLocation = baseApk.installLocation;
            this.verifiers = baseApk.verifiers;
            this.splitNames = splitNames;
            this.codePath = codePath;
            this.baseCodePath = baseApk.codePath;
            this.splitCodePaths = splitCodePaths;
            this.baseRevisionCode = baseApk.revisionCode;
            this.splitRevisionCodes = splitRevisionCodes;
            this.coreApp = baseApk.coreApp;
            this.multiArch = baseApk.multiArch;
            this.use32bitAbi = baseApk.use32bitAbi;
            this.extractNativeLibs = baseApk.extractNativeLibs;
        }
    }

```
这里我们不在过多分析！！

#### 3.1.1.2 PParser.parseBaseApk\[3]
解析 base apk：
```java
    private Package parseBaseApk(File apkFile, AssetManager assets, int flags)
            throws PackageParserException {
        final String apkPath = apkFile.getAbsolutePath();

        String volumeUuid = null;
        if (apkPath.startsWith(MNT_EXPAND)) { // 解析 /mnt/expand/
            final int end = apkPath.indexOf('/', MNT_EXPAND.length());
            volumeUuid = apkPath.substring(MNT_EXPAND.length(), end);
        }

        mParseError = PackageManager.INSTALL_SUCCEEDED;
        mArchiveSourcePath = apkFile.getAbsolutePath();

        if (DEBUG_JAR) Slog.d(TAG, "Scanning base APK: " + apkPath);

        final int cookie = loadApkIntoAssetManager(assets, apkPath, flags);

        Resources res = null;
        XmlResourceParser parser = null;
        try {
            res = new Resources(assets, mMetrics, null);
            assets.setConfiguration(0, 0, null, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
                    Build.VERSION.RESOURCES_SDK_INT);

            //【1】用于解析 AndroidManifest.xml 文件！      
            parser = assets.openXmlResourceParser(cookie, ANDROID_MANIFEST_FILENAME);

            final String[] outError = new String[1];
            
            //【3.1.1.2】解析核心 apk！
            final Package pkg = parseBaseApk(res, parser, flags, outError);
            if (pkg == null) {
                throw new PackageParserException(mParseError,
                        apkPath + " (at " + parser.getPositionDescription() + "): " + outError[0]);
            }

            pkg.setVolumeUuid(volumeUuid);
            pkg.setApplicationVolumeUuid(volumeUuid);
            pkg.setBaseCodePath(apkPath);
            pkg.setSignatures(null);

            return pkg;

        } catch (PackageParserException e) {
            throw e;
        } catch (Exception e) {
            throw new PackageParserException(INSTALL_PARSE_FAILED_UNEXPECTED_EXCEPTION,
                    "Failed to read manifest from " + apkPath, e);
        } finally {
            IoUtils.closeQuietly(parser);
        }
    }
```
接着，调用重载 parseBaseApk 方法：


##### 3.1.1.2.1 PParser.parseBaseApk\[4]

我们继续来看：
```java
    private Package parseBaseApk(Resources res, XmlResourceParser parser, int flags,
            String[] outError) throws XmlPullParserException, IOException {
        final String splitName;
        final String pkgName;

        try {
            //【3.1.1.1.1.2】解析 <manifest> 标签，获得 packageName 和 splitName，
            // 以 Pair<packageName, splitName> 的形式返回！
            Pair<String, String> packageSplit = parsePackageSplitNames(parser, parser);
            pkgName = packageSplit.first;
            splitName = packageSplit.second;

            if (!TextUtils.isEmpty(splitName)) {
                outError[0] = "Expected base APK, but found split " + splitName;
                mParseError = PackageManager.INSTALL_PARSE_FAILED_BAD_PACKAGE_NAME;
                return null;
            }

        } catch (PackageParserException e) {

            mParseError = PackageManager.INSTALL_PARSE_FAILED_BAD_PACKAGE_NAME;
            return null;

        }

        //【3.1.3】创建一个 Package 对象，用于保存最终的解析信息！
        final Package pkg = new Package(pkgName);

        // 解析 <manifest> 标签，获得 versionCode、revisionCode、versionName、coreApp 值！
        TypedArray sa = res.obtainAttributes(parser,
                com.android.internal.R.styleable.AndroidManifest);
                
        pkg.mVersionCode = pkg.applicationInfo.versionCode = sa.getInteger(
                com.android.internal.R.styleable.AndroidManifest_versionCode, 0);
                
        pkg.baseRevisionCode = sa.getInteger(
                com.android.internal.R.styleable.AndroidManifest_revisionCode, 0);
                
        pkg.mVersionName = sa.getNonConfigurationString(
                com.android.internal.R.styleable.AndroidManifest_versionName, 0);
                
        if (pkg.mVersionName != null) {
            pkg.mVersionName = pkg.mVersionName.intern();
        }

        pkg.coreApp = parser.getAttributeBooleanValue(null, "coreApp", false);

        sa.recycle();

        //【3.1.1.2.2】接着，调用 parseBaseApkCommon 方法，继续解析！
        return parseBaseApkCommon(pkg, null, res, parser, flags, outError);
    }

```
##### 3.1.1.2.2 PParser.parseBaseApkCommon [6]
进入最终的解析方法中：

- Package pkg：应用程序的信息封装对象；
- Set<String> acceptedTags：需要解析的标签，如果传入 null ，表示解析所有，这里传入 null；
- Resources res：
- XmlResourceParser parser：
- int flags：
- String[] outError：用于保存解析过程中的错误信息；

```java
    private Package parseBaseApkCommon(Package pkg, Set<String> acceptedTags, Resources res,
            XmlResourceParser parser, int flags, String[] outError) throws XmlPullParserException,
            IOException {
        mParseInstrumentationArgs = null;
        mParseActivityArgs = null;
        mParseServiceArgs = null;
        mParseProviderArgs = null;

        int type;
        boolean foundApp = false;

        TypedArray sa = res.obtainAttributes(parser,
                com.android.internal.R.styleable.AndroidManifest);
        
        //【1】获得 sharedUserId 属性！
        String str = sa.getNonConfigurationString(
                com.android.internal.R.styleable.AndroidManifest_sharedUserId, 0);
        if (str != null && str.length() > 0) {
            String nameError = validateName(str, true, false);
            if (nameError != null && !"android".equals(pkg.packageName)) {
                outError[0] = "<manifest> specifies bad sharedUserId name \""
                    + str + "\": " + nameError;
                mParseError = PackageManager.INSTALL_PARSE_FAILED_BAD_SHARED_USER_ID;
                return null;
            }
            pkg.mSharedUserId = str.intern();
            pkg.mSharedUserLabel = sa.getResourceId(
                    com.android.internal.R.styleable.AndroidManifest_sharedUserLabel, 0);
        }

        //【2】获得 installLocation 属性！
        pkg.installLocation = sa.getInteger(
                com.android.internal.R.styleable.AndroidManifest_installLocation,
                PARSE_DEFAULT_INSTALL_LOCATION);
        pkg.applicationInfo.installLocation = pkg.installLocation;


        /* Set the global "forward lock" flag */
        if ((flags & PARSE_FORWARD_LOCK) != 0) {
            pkg.applicationInfo.privateFlags |= ApplicationInfo.PRIVATE_FLAG_FORWARD_LOCK;
        }

        /* Set the global "on SD card" flag */
        if ((flags & PARSE_EXTERNAL_STORAGE) != 0) {
            pkg.applicationInfo.flags |= ApplicationInfo.FLAG_EXTERNAL_STORAGE;
        }

        if ((flags & PARSE_IS_EPHEMERAL) != 0) {
            pkg.applicationInfo.privateFlags |= ApplicationInfo.PRIVATE_FLAG_EPHEMERAL;
        }

        // Resource boolean are -1, so 1 means we don't know the value.
        int supportsSmallScreens = 1;
        int supportsNormalScreens = 1;
        int supportsLargeScreens = 1;
        int supportsXLargeScreens = 1;
        int resizeable = 1;
        int anyDensity = 1;

        int outerDepth = parser.getDepth();
        while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
                && (type != XmlPullParser.END_TAG || parser.getDepth() > outerDepth)) {
            if (type == XmlPullParser.END_TAG || type == XmlPullParser.TEXT) {
                continue;
            }

            String tagName = parser.getName();

            if (acceptedTags != null && !acceptedTags.contains(tagName)) {
                Slog.w(TAG, "Skipping unsupported element under <manifest>: "
                        + tagName + " at " + mArchiveSourcePath + " "
                        + parser.getPositionDescription());
                XmlUtils.skipCurrentTag(parser);
                continue;
            }

            if (tagName.equals(TAG_APPLICATION)) { // 解析 "application" 标签
                if (foundApp) { 
                    if (RIGID_PARSER) {
                        outError[0] = "<manifest> has more than one <application>";
                        mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
                        return null;
                    } else {
                        Slog.w(TAG, "<manifest> has more than one <application>");
                        XmlUtils.skipCurrentTag(parser);
                        continue;
                    }
                }

                foundApp = true;
                
                //【3.1.1.2.3】接着调用 parseBaseApplication 方法解析 "application" 标签的属性以及四大组件的信息！
                if (!parseBaseApplication(pkg, res, parser, flags, outError)) {
                    return null;
                }

            } else if (tagName.equals(TAG_OVERLAY)) { // 解析 "overlay" 标签
                sa = res.obtainAttributes(parser,
                        com.android.internal.R.styleable.AndroidManifestResourceOverlay);
                pkg.mOverlayTarget = sa.getString(
                        com.android.internal.R.styleable.AndroidManifestResourceOverlay_targetPackage);
                pkg.mOverlayPriority = sa.getInt(
                        com.android.internal.R.styleable.AndroidManifestResourceOverlay_priority,
                        -1);
                sa.recycle();

                if (pkg.mOverlayTarget == null) {
                    outError[0] = "<overlay> does not specify a target package";
                    mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
                    return null;
                }
                if (pkg.mOverlayPriority < 0 || pkg.mOverlayPriority > 9999) {
                    outError[0] = "<overlay> priority must be between 0 and 9999";
                    mParseError =
                        PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
                    return null;
                }
                XmlUtils.skipCurrentTag(parser);

            } else if (tagName.equals(TAG_KEY_SETS)) { // 解析 'key-sets" 标签
                if (!parseKeySets(pkg, res, parser, outError)) {
                    return null;
                }
                
            } else if (tagName.equals(TAG_PERMISSION_GROUP)) { //【3.1.1.2.2.1】解析 'permission-group" 标签
                if (parsePermissionGroup(pkg, flags, res, parser, outError) == null) {
                    return null;
                }
                
            } else if (tagName.equals(TAG_PERMISSION)) { //【3.1.1.2.2.2】解析 'permission" 标签
                if (parsePermission(pkg, res, parser, outError) == null) {
                    return null;
                }
            } else if (tagName.equals(TAG_PERMISSION_TREE)) { //【3.1.1.2.2.3】解析 'permission-tree" 标签
                if (parsePermissionTree(pkg, res, parser, outError) == null) {
                    return null;
                }
                
            } else if (tagName.equals(TAG_USES_PERMISSION)) { //【3.1.1.2.2.4】解析 'uses-permission" 标签
                if (!parseUsesPermission(pkg, res, parser)) {
                    return null;
                }
                
            } else if (tagName.equals(TAG_USES_PERMISSION_SDK_M)
                    || tagName.equals(TAG_USES_PERMISSION_SDK_23)) { 
                    
                // 解析 'uses-permission-sdk-m" 或 "uses-permission-sdk-23" 标签
                if (!parseUsesPermission(pkg, res, parser)) {
                    return null;
                }
                
            } else if (tagName.equals(TAG_USES_CONFIGURATION)) { // 解析 "uses-configuration" 标签
                ConfigurationInfo cPref = new ConfigurationInfo();
                sa = res.obtainAttributes(parser,
                        com.android.internal.R.styleable.AndroidManifestUsesConfiguration);
                cPref.reqTouchScreen = sa.getInt(
                        com.android.internal.R.styleable.AndroidManifestUsesConfiguration_reqTouchScreen,
                        Configuration.TOUCHSCREEN_UNDEFINED);
                cPref.reqKeyboardType = sa.getInt(
                        com.android.internal.R.styleable.AndroidManifestUsesConfiguration_reqKeyboardType,
                        Configuration.KEYBOARD_UNDEFINED);
                if (sa.getBoolean(
                        com.android.internal.R.styleable.AndroidManifestUsesConfiguration_reqHardKeyboard,
                        false)) {
                    cPref.reqInputFeatures |= ConfigurationInfo.INPUT_FEATURE_HARD_KEYBOARD;
                }
                cPref.reqNavigation = sa.getInt(
                        com.android.internal.R.styleable.AndroidManifestUsesConfiguration_reqNavigation,
                        Configuration.NAVIGATION_UNDEFINED);
                if (sa.getBoolean(
                        com.android.internal.R.styleable.AndroidManifestUsesConfiguration_reqFiveWayNav,
                        false)) {
                    cPref.reqInputFeatures |= ConfigurationInfo.INPUT_FEATURE_FIVE_WAY_NAV;
                }
                sa.recycle();
                pkg.configPreferences = ArrayUtils.add(pkg.configPreferences, cPref);

                XmlUtils.skipCurrentTag(parser);

            } else if (tagName.equals(TAG_USES_FEATURE)) { // 解析 "uses-feature" 标签
                FeatureInfo fi = parseUsesFeature(res, parser);
                pkg.reqFeatures = ArrayUtils.add(pkg.reqFeatures, fi);

                if (fi.name == null) {
                    ConfigurationInfo cPref = new ConfigurationInfo();
                    cPref.reqGlEsVersion = fi.reqGlEsVersion;
                    pkg.configPreferences = ArrayUtils.add(pkg.configPreferences, cPref);
                }

                XmlUtils.skipCurrentTag(parser);

            } else if (tagName.equals(TAG_FEATURE_GROUP)) {  // 解析 "feature-group" 标签
                FeatureGroupInfo group = new FeatureGroupInfo();
                ArrayList<FeatureInfo> features = null;
                final int innerDepth = parser.getDepth();
                while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
                        && (type != XmlPullParser.END_TAG || parser.getDepth() > innerDepth)) {
                    if (type == XmlPullParser.END_TAG || type == XmlPullParser.TEXT) {
                        continue;
                    }

                    final String innerTagName = parser.getName();
                    if (innerTagName.equals("uses-feature")) {
                        FeatureInfo featureInfo = parseUsesFeature(res, parser);
                        // FeatureGroups are stricter and mandate that
                        // any <uses-feature> declared are mandatory.
                        featureInfo.flags |= FeatureInfo.FLAG_REQUIRED;
                        features = ArrayUtils.add(features, featureInfo);
                    } else {
                        Slog.w(TAG, "Unknown element under <feature-group>: " + innerTagName +
                                " at " + mArchiveSourcePath + " " +
                                parser.getPositionDescription());
                    }
                    XmlUtils.skipCurrentTag(parser);
                }

                if (features != null) {
                    group.features = new FeatureInfo[features.size()];
                    group.features = features.toArray(group.features);
                }
                pkg.featureGroups = ArrayUtils.add(pkg.featureGroups, group);

            } else if (tagName.equals(TAG_USES_SDK)) {  // 解析 "uses-sdk" 标签
                if (SDK_VERSION > 0) {
                    sa = res.obtainAttributes(parser,
                            com.android.internal.R.styleable.AndroidManifestUsesSdk);

                    int minVers = 1;
                    String minCode = null;
                    int targetVers = 0;
                    String targetCode = null;

                    TypedValue val = sa.peekValue(
                            com.android.internal.R.styleable.AndroidManifestUsesSdk_minSdkVersion);
                    if (val != null) {
                        if (val.type == TypedValue.TYPE_STRING && val.string != null) {
                            targetCode = minCode = val.string.toString();
                        } else {
                            // If it's not a string, it's an integer.
                            targetVers = minVers = val.data;
                        }
                    }

                    val = sa.peekValue(
                            com.android.internal.R.styleable.AndroidManifestUsesSdk_targetSdkVersion);
                    if (val != null) {
                        if (val.type == TypedValue.TYPE_STRING && val.string != null) {
                            targetCode = val.string.toString();
                            if (minCode == null) {
                                minCode = targetCode;
                            }
                        } else {
                            // If it's not a string, it's an integer.
                            targetVers = val.data;
                        }
                    }

                    sa.recycle();

                    if (minCode != null) {
                        boolean allowedCodename = false;
                        for (String codename : SDK_CODENAMES) {
                            if (minCode.equals(codename)) {
                                allowedCodename = true;
                                break;
                            }
                        }
                        if (!allowedCodename) {
                            if (SDK_CODENAMES.length > 0) {
                                outError[0] = "Requires development platform " + minCode
                                        + " (current platform is any of "
                                        + Arrays.toString(SDK_CODENAMES) + ")";
                            } else {
                                outError[0] = "Requires development platform " + minCode
                                        + " but this is a release platform.";
                            }
                            mParseError = PackageManager.INSTALL_FAILED_OLDER_SDK;
                            return null;
                        }
                        pkg.applicationInfo.minSdkVersion =
                                android.os.Build.VERSION_CODES.CUR_DEVELOPMENT;
                    } else if (minVers > SDK_VERSION) {
                        outError[0] = "Requires newer sdk version #" + minVers
                                + " (current version is #" + SDK_VERSION + ")";
                        mParseError = PackageManager.INSTALL_FAILED_OLDER_SDK;
                        return null;
                    } else {
                        pkg.applicationInfo.minSdkVersion = minVers;
                    }

                    if (targetCode != null) {
                        boolean allowedCodename = false;
                        for (String codename : SDK_CODENAMES) {
                            if (targetCode.equals(codename)) {
                                allowedCodename = true;
                                break;
                            }
                        }
                        if (!allowedCodename) {
                            if (SDK_CODENAMES.length > 0) {
                                outError[0] = "Requires development platform " + targetCode
                                        + " (current platform is any of "
                                        + Arrays.toString(SDK_CODENAMES) + ")";
                            } else {
                                outError[0] = "Requires development platform " + targetCode
                                        + " but this is a release platform.";
                            }
                            mParseError = PackageManager.INSTALL_FAILED_OLDER_SDK;
                            return null;
                        }
                        // If the code matches, it definitely targets this SDK.
                        pkg.applicationInfo.targetSdkVersion
                                = android.os.Build.VERSION_CODES.CUR_DEVELOPMENT;
                    } else {
                        pkg.applicationInfo.targetSdkVersion = targetVers;
                    }
                }

                XmlUtils.skipCurrentTag(parser);

            } else if (tagName.equals(TAG_SUPPORT_SCREENS)) {  // 解析 "supports-screens" 标签
                sa = res.obtainAttributes(parser,
                        com.android.internal.R.styleable.AndroidManifestSupportsScreens);

                pkg.applicationInfo.requiresSmallestWidthDp = sa.getInteger(
                        com.android.internal.R.styleable.AndroidManifestSupportsScreens_requiresSmallestWidthDp,
                        0);
                pkg.applicationInfo.compatibleWidthLimitDp = sa.getInteger(
                        com.android.internal.R.styleable.AndroidManifestSupportsScreens_compatibleWidthLimitDp,
                        0);
                pkg.applicationInfo.largestWidthLimitDp = sa.getInteger(
                        com.android.internal.R.styleable.AndroidManifestSupportsScreens_largestWidthLimitDp,
                        0);

                // 和屏幕相关的参数解析，屏幕大小等等！
                supportsSmallScreens = sa.getInteger(
                        com.android.internal.R.styleable.AndroidManifestSupportsScreens_smallScreens,
                        supportsSmallScreens);
                supportsNormalScreens = sa.getInteger(
                        com.android.internal.R.styleable.AndroidManifestSupportsScreens_normalScreens,
                        supportsNormalScreens);
                supportsLargeScreens = sa.getInteger(
                        com.android.internal.R.styleable.AndroidManifestSupportsScreens_largeScreens,
                        supportsLargeScreens);
                supportsXLargeScreens = sa.getInteger(
                        com.android.internal.R.styleable.AndroidManifestSupportsScreens_xlargeScreens,
                        supportsXLargeScreens);
                resizeable = sa.getInteger(
                        com.android.internal.R.styleable.AndroidManifestSupportsScreens_resizeable,
                        resizeable);
                anyDensity = sa.getInteger(
                        com.android.internal.R.styleable.AndroidManifestSupportsScreens_anyDensity,
                        anyDensity);

                sa.recycle();

                XmlUtils.skipCurrentTag(parser);

            } else if (tagName.equals(TAG_PROTECTED_BROADCAST)) { // 解析 "protected-broadcast" 标签
                sa = res.obtainAttributes(parser,
                        com.android.internal.R.styleable.AndroidManifestProtectedBroadcast);

                // Note: don't allow this value to be a reference to a resource
                // that may change.
                String name = sa.getNonResourceString(
                        com.android.internal.R.styleable.AndroidManifestProtectedBroadcast_name);

                sa.recycle();

                if (name != null && (flags&PARSE_IS_SYSTEM) != 0) {
                    if (pkg.protectedBroadcasts == null) {
                        pkg.protectedBroadcasts = new ArrayList<String>();
                    }
                    if (!pkg.protectedBroadcasts.contains(name)) {
                        pkg.protectedBroadcasts.add(name.intern());
                    }
                }

                XmlUtils.skipCurrentTag(parser);

            } else if (tagName.equals(TAG_INSTRUMENTATION)) {  // 解析 "instrumentation" 标签
                if (parseInstrumentation(pkg, res, parser, outError) == null) {
                    return null;
                }
            
            } else if (tagName.equals(TAG_ORIGINAL_PACKAGE)) { // 解析 "original-package" 标签
                sa = res.obtainAttributes(parser,
                        com.android.internal.R.styleable.AndroidManifestOriginalPackage);

                String orig =sa.getNonConfigurationString(
                        com.android.internal.R.styleable.AndroidManifestOriginalPackage_name, 0);
                if (!pkg.packageName.equals(orig)) {
                    if (pkg.mOriginalPackages == null) {
                        pkg.mOriginalPackages = new ArrayList<String>();
                        pkg.mRealPackage = pkg.packageName;
                    }
                    pkg.mOriginalPackages.add(orig);
                }

                sa.recycle();

                XmlUtils.skipCurrentTag(parser);

            } else if (tagName.equals(TAG_ADOPT_PERMISSIONS)) { // 解析 "adopt-permissions" 标签
                sa = res.obtainAttributes(parser,
                        com.android.internal.R.styleable.AndroidManifestOriginalPackage);

                String name = sa.getNonConfigurationString(
                        com.android.internal.R.styleable.AndroidManifestOriginalPackage_name, 0);

                sa.recycle();

                if (name != null) {
                    if (pkg.mAdoptPermissions == null) {
                        pkg.mAdoptPermissions = new ArrayList<String>();
                    }
                    pkg.mAdoptPermissions.add(name);
                }

                XmlUtils.skipCurrentTag(parser);

            } else if (tagName.equals(TAG_USES_GL_TEXTURE)) {  // 跳过一下标签！
                // Just skip this tag
                XmlUtils.skipCurrentTag(parser);
                continue;

            } else if (tagName.equals(TAG_COMPATIBLE_SCREENS)) {
                // Just skip this tag
                XmlUtils.skipCurrentTag(parser);
                continue;
                
            } else if (tagName.equals(TAG_SUPPORTS_INPUT)) { 
                XmlUtils.skipCurrentTag(parser);
                continue;

            } else if (tagName.equals(TAG_EAT_COMMENT)) { 
                // Just skip this tag
                XmlUtils.skipCurrentTag(parser);
                continue;

            } else if (tagName.equals(TAG_PACKAGE)) { // 解析 "package" 标签，该标签表示应用的子包！
                if (!MULTI_PACKAGE_APK_ENABLED) {
                    XmlUtils.skipCurrentTag(parser);
                    continue;
                }
                // 所有的子包都会解析保存到 ArrayList<Package> childPackages 集合中！
                if (!parseBaseApkChild(pkg, res, parser, flags, outError)) {
                    // If parsing a child failed the error is already set
                    return null;
                }

            } else if (tagName.equals(TAG_RESTRICT_UPDATE)) {  // 解析 "restrict-update" 标签
                if ((flags & PARSE_IS_SYSTEM_DIR) != 0) {
                    sa = res.obtainAttributes(parser,
                            com.android.internal.R.styleable.AndroidManifestRestrictUpdate);
                    final String hash = sa.getNonConfigurationString(
                            com.android.internal.R.styleable.AndroidManifestRestrictUpdate_hash, 0);
                    sa.recycle();

                    pkg.restrictUpdateHash = null;
                    if (hash != null) {
                        final int hashLength = hash.length();
                        final byte[] hashBytes = new byte[hashLength / 2];
                        for (int i = 0; i < hashLength; i += 2){
                            hashBytes[i/2] = (byte) ((Character.digit(hash.charAt(i), 16) << 4)
                                    + Character.digit(hash.charAt(i + 1), 16));
                        }
                        pkg.restrictUpdateHash = hashBytes;
                    }
                }

                XmlUtils.skipCurrentTag(parser);

            } else if (RIGID_PARSER) {
                outError[0] = "Bad element under <manifest>: "
                    + parser.getName();
                mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
                return null;

            } else {
                Slog.w(TAG, "Unknown element under <manifest>: " + parser.getName()
                        + " at " + mArchiveSourcePath + " "
                        + parser.getPositionDescription());
                XmlUtils.skipCurrentTag(parser);
                continue;
            }
        }

        if (!foundApp && pkg.instrumentation.size() == 0) {
            outError[0] = "<manifest> does not contain an <application> or <instrumentation>";
            mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_EMPTY;
        }

        final int NP = PackageParser.NEW_PERMISSIONS.length;
        StringBuilder implicitPerms = null;
        for (int ip=0; ip<NP; ip++) {
            final PackageParser.NewPermissionInfo npi
                    = PackageParser.NEW_PERMISSIONS[ip];
            if (pkg.applicationInfo.targetSdkVersion >= npi.sdkVersion) {
                break;
            }
            if (!pkg.requestedPermissions.contains(npi.name)) {
                if (implicitPerms == null) {
                    implicitPerms = new StringBuilder(128);
                    implicitPerms.append(pkg.packageName);
                    implicitPerms.append(": compat added ");
                } else {
                    implicitPerms.append(' ');
                }
                implicitPerms.append(npi.name);
                pkg.requestedPermissions.add(npi.name);
            }
        }
        if (implicitPerms != null) {
            Slog.i(TAG, implicitPerms.toString());
        }

        final int NS = PackageParser.SPLIT_PERMISSIONS.length;
        for (int is=0; is<NS; is++) {
            final PackageParser.SplitPermissionInfo spi
                    = PackageParser.SPLIT_PERMISSIONS[is];
            if (pkg.applicationInfo.targetSdkVersion >= spi.targetSdk
                    || !pkg.requestedPermissions.contains(spi.rootPerm)) {
                continue;
            }
            for (int in=0; in<spi.newPerms.length; in++) {
                final String perm = spi.newPerms[in];
                if (!pkg.requestedPermissions.contains(perm)) {
                    pkg.requestedPermissions.add(perm);
                }
            }
        }

        // 解析屏幕大小、像素密度相关的属性！
        if (supportsSmallScreens < 0 || (supportsSmallScreens > 0
                && pkg.applicationInfo.targetSdkVersion
                        >= android.os.Build.VERSION_CODES.DONUT)) {
            pkg.applicationInfo.flags |= ApplicationInfo.FLAG_SUPPORTS_SMALL_SCREENS;
        }
        if (supportsNormalScreens != 0) {
            pkg.applicationInfo.flags |= ApplicationInfo.FLAG_SUPPORTS_NORMAL_SCREENS;
        }
        if (supportsLargeScreens < 0 || (supportsLargeScreens > 0
                && pkg.applicationInfo.targetSdkVersion
                        >= android.os.Build.VERSION_CODES.DONUT)) {
            pkg.applicationInfo.flags |= ApplicationInfo.FLAG_SUPPORTS_LARGE_SCREENS;
        }
        if (supportsXLargeScreens < 0 || (supportsXLargeScreens > 0
                && pkg.applicationInfo.targetSdkVersion
                        >= android.os.Build.VERSION_CODES.GINGERBREAD)) {
            pkg.applicationInfo.flags |= ApplicationInfo.FLAG_SUPPORTS_XLARGE_SCREENS;
        }
        if (resizeable < 0 || (resizeable > 0
                && pkg.applicationInfo.targetSdkVersion
                        >= android.os.Build.VERSION_CODES.DONUT)) {
            pkg.applicationInfo.flags |= ApplicationInfo.FLAG_RESIZEABLE_FOR_SCREENS;
        }
        if (anyDensity < 0 || (anyDensity > 0
                && pkg.applicationInfo.targetSdkVersion
                        >= android.os.Build.VERSION_CODES.DONUT)) {
            pkg.applicationInfo.flags |= ApplicationInfo.FLAG_SUPPORTS_SCREEN_DENSITIES;
        }
        
        // 返回 Package 对象！
        return pkg;
    }

```
在这个方法中，解析 apk 中的四大组件，权限等信息，核心 apk 解析到此为止！

###### 3.1.1.2.2.1 PParser.parsePermissionGroup - 解析 permission-group

parsePermissionGroup 用于解析该应用所使用到的 permission-group：

```java
    private PermissionGroup parsePermissionGroup(Package owner, int flags, Resources res,
            XmlResourceParser parser, String[] outError)
            throws XmlPullParserException, IOException {
        //【1】对于权限组，创建了 PermissionGroup 对象！
        PermissionGroup perm = new PermissionGroup(owner);

        TypedArray sa = res.obtainAttributes(parser,
                com.android.internal.R.styleable.AndroidManifestPermissionGroup);
        if (!parsePackageItemInfo(owner, perm.info, outError,
                "<permission-group>", sa, true /*nameRequired*/,
                com.android.internal.R.styleable.AndroidManifestPermissionGroup_name,
                com.android.internal.R.styleable.AndroidManifestPermissionGroup_label,
                com.android.internal.R.styleable.AndroidManifestPermissionGroup_icon,
                com.android.internal.R.styleable.AndroidManifestPermissionGroup_roundIcon,
                com.android.internal.R.styleable.AndroidManifestPermissionGroup_logo,
                com.android.internal.R.styleable.AndroidManifestPermissionGroup_banner)) {
            sa.recycle();
            mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
            return null;
        }
        //【2】解析相关的属性
        perm.info.descriptionRes = sa.getResourceId(
                com.android.internal.R.styleable.AndroidManifestPermissionGroup_description,
                0);
        perm.info.flags = sa.getInt(
                com.android.internal.R.styleable.AndroidManifestPermissionGroup_permissionGroupFlags, 0);
        perm.info.priority = sa.getInt(
                com.android.internal.R.styleable.AndroidManifestPermissionGroup_priority, 0);
        if (perm.info.priority > 0 && (flags&PARSE_IS_SYSTEM) == 0) {
            perm.info.priority = 0;
        }

        sa.recycle();

        if (!parseAllMetaData(res, parser, "<permission-group>", perm,
                outError)) {
            mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
            return null;
        }
         //【3】保存到 owner.permissionGroups 中！
        owner.permissionGroups.add(perm);

        return perm;
    }

```
这里就不多说了！

###### 3.1.1.2.2.2 PParser.parsePermission - 解析 permission

parsePermission 解析该应用中定义的权限！
```java
    private Permission parsePermission(Package owner, Resources res,
            XmlResourceParser parser, String[] outError)
        throws XmlPullParserException, IOException {
        //【1】创建 Permission 对象
        Permission perm = new Permission(owner);

        TypedArray sa = res.obtainAttributes(parser,
                com.android.internal.R.styleable.AndroidManifestPermission);

        if (!parsePackageItemInfo(owner, perm.info, outError,
                "<permission>", sa, true /*nameRequired*/,
                com.android.internal.R.styleable.AndroidManifestPermission_name,
                com.android.internal.R.styleable.AndroidManifestPermission_label,
                com.android.internal.R.styleable.AndroidManifestPermission_icon,
                com.android.internal.R.styleable.AndroidManifestPermission_roundIcon,
                com.android.internal.R.styleable.AndroidManifestPermission_logo,
                com.android.internal.R.styleable.AndroidManifestPermission_banner)) {
            sa.recycle();
            mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
            return null;
        }

        // 解析 android:permissionGroup 属性
        perm.info.group = sa.getNonResourceString(
                com.android.internal.R.styleable.AndroidManifestPermission_permissionGroup);
        if (perm.info.group != null) {
            perm.info.group = perm.info.group.intern();
        }

        perm.info.descriptionRes = sa.getResourceId(
                com.android.internal.R.styleable.AndroidManifestPermission_description,
                0);
        // 解析 android:protectionLevel 属性
        perm.info.protectionLevel = sa.getInt(
                com.android.internal.R.styleable.AndroidManifestPermission_protectionLevel,
                PermissionInfo.PROTECTION_NORMAL);
        // 解析 android:permissionFlags 属性
        perm.info.flags = sa.getInt(
                com.android.internal.R.styleable.AndroidManifestPermission_permissionFlags, 0);

        sa.recycle();

        if (perm.info.protectionLevel == -1) {
            outError[0] = "<permission> does not specify protectionLevel";
            mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
            return null;
        }

        perm.info.protectionLevel = PermissionInfo.fixProtectionLevel(perm.info.protectionLevel);

        if ((perm.info.protectionLevel&PermissionInfo.PROTECTION_MASK_FLAGS) != 0) {
            if ((perm.info.protectionLevel&PermissionInfo.PROTECTION_MASK_BASE) !=
                    PermissionInfo.PROTECTION_SIGNATURE) {
                outError[0] = "<permission>  protectionLevel specifies a flag but is "
                        + "not based on signature type";
                mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
                return null;
            }
        }

        if (!parseAllMetaData(res, parser, "<permission>", perm, outError)) {
            mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
            return null;
        }
        // 将解析的权限信息保存到 owner.permissions 中！
        owner.permissions.add(perm);

        return perm;
    }
```
解析很简单，不多说了！

###### 3.1.1.2.2.3 PParser.parsePermissionTree - 解析 permission-tree

parsePermissionTree 用于解析该应用所使用到的 permission-tree：

```java
    private Permission parsePermissionTree(Package owner, Resources res,
            XmlResourceParser parser, String[] outError)
        throws XmlPullParserException, IOException { 
        //【1】可以看到对于 permission-tree，依然是会创建一个 Permission 对象！
        Permission perm = new Permission(owner);

        TypedArray sa = res.obtainAttributes(parser,
                com.android.internal.R.styleable.AndroidManifestPermissionTree);

        if (!parsePackageItemInfo(owner, perm.info, outError,
                "<permission-tree>", sa, true /*nameRequired*/,
                com.android.internal.R.styleable.AndroidManifestPermissionTree_name,
                com.android.internal.R.styleable.AndroidManifestPermissionTree_label,
                com.android.internal.R.styleable.AndroidManifestPermissionTree_icon,
                com.android.internal.R.styleable.AndroidManifestPermissionTree_roundIcon,
                com.android.internal.R.styleable.AndroidManifestPermissionTree_logo,
                com.android.internal.R.styleable.AndroidManifestPermissionTree_banner)) {
            sa.recycle();
            mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
            return null;
        }

        sa.recycle();

        //【2】解析权限的属性；
        int index = perm.info.name.indexOf('.');
        if (index > 0) {
            index = perm.info.name.indexOf('.', index+1);
        }
        if (index < 0) {
            outError[0] = "<permission-tree> name has less than three segments: "
                + perm.info.name;
            mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
            return null;
        }

        perm.info.descriptionRes = 0;
        perm.info.protectionLevel = PermissionInfo.PROTECTION_NORMAL;
        perm.tree = true;

        if (!parseAllMetaData(res, parser, "<permission-tree>", perm,
                outError)) { // 解析 meta-data，这里不关注！
            mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
            return null;
        }
        //【3】将该 Permission 对象添加到 owner.permissions 中！
        owner.permissions.add(perm);

        return perm;
    }
```


###### 3.1.1.2.2.4 PParser.parseUsesPermission - 解析 uses-permission

parseUsesPermission 解析该应用所使用的权限！

```java
    private boolean parseUsesPermission(Package pkg, Resources res, XmlResourceParser parser)
            throws XmlPullParserException, IOException {
        TypedArray sa = res.obtainAttributes(parser,
                com.android.internal.R.styleable.AndroidManifestUsesPermission);

        String name = sa.getNonResourceString(
                com.android.internal.R.styleable.AndroidManifestUsesPermission_name);

        int maxSdkVersion = 0;
        TypedValue val = sa.peekValue(
                com.android.internal.R.styleable.AndroidManifestUsesPermission_maxSdkVersion);
        if (val != null) {
            if (val.type >= TypedValue.TYPE_FIRST_INT && val.type <= TypedValue.TYPE_LAST_INT) {
                maxSdkVersion = val.data;
            }
        }

        sa.recycle();

        if ((maxSdkVersion == 0) || (maxSdkVersion >= Build.VERSION.RESOURCES_SDK_INT)) {
            if (name != null) {
                int index = pkg.requestedPermissions.indexOf(name);
                if (index == -1) {
                    //【1】将权限添加到 pkg.requestedPermissions 中去！
                    pkg.requestedPermissions.add(name.intern());
                } else {
                    Slog.w(TAG, "Ignoring duplicate uses-permissions/uses-permissions-sdk-m: "
                            + name + " in package: " + pkg.packageName + " at: "
                            + parser.getPositionDescription());
                }
            }
        }

        XmlUtils.skipCurrentTag(parser);
        return true;
    }
```
解析很简单，不多说了！

##### 3.1.1.2.3 PParser.parseBaseApplication

调用 parseBaseApplication 方法来解析核心 apk 的 applicaiton 标签和内部的四大组件信息！！
```java
    private boolean parseBaseApplication(Package owner, Resources res,
            XmlResourceParser parser, int flags, String[] outError)
        throws XmlPullParserException, IOException {
        //【1】解析 application 标签的属性！
        final ApplicationInfo ai = owner.applicationInfo;
        final String pkgName = owner.applicationInfo.packageName;

        TypedArray sa = res.obtainAttributes(parser,
                com.android.internal.R.styleable.AndroidManifestApplication);
       
        // 解析 android:name android:label android:icon android:roundIcon android:logo android:banner 属性！
        if (!parsePackageItemInfo(owner, ai, outError,
                "<application>", sa, false /*nameRequired*/,
                com.android.internal.R.styleable.AndroidManifestApplication_name,
                com.android.internal.R.styleable.AndroidManifestApplication_label,
                com.android.internal.R.styleable.AndroidManifestApplication_icon,
                com.android.internal.R.styleable.AndroidManifestApplication_roundIcon,
                com.android.internal.R.styleable.AndroidManifestApplication_logo,
                com.android.internal.R.styleable.AndroidManifestApplication_banner)) {
            sa.recycle();
            mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
            return false;
        }

        if (ai.name != null) {
            ai.className = ai.name;
        }
        // 解析 android:manageSpaceActivity 属性！
        String manageSpaceActivity = sa.getNonConfigurationString(
                com.android.internal.R.styleable.AndroidManifestApplication_manageSpaceActivity,
                Configuration.NATIVE_CONFIG_VERSION);
        if (manageSpaceActivity != null) {
            ai.manageSpaceActivityName = buildClassName(pkgName, manageSpaceActivity,
                    outError);
        }

        //【1】解析 android:allowBackup 属性，为 true 表示应用允许备份！
        boolean allowBackup = sa.getBoolean(
                com.android.internal.R.styleable.AndroidManifestApplication_allowBackup, true);
        if (allowBackup) {
            ai.flags |= ApplicationInfo.FLAG_ALLOW_BACKUP; // 如果为 true，设置 FLAG_ALLOW_BACKUP 标志位
            
            // 解析 android:backupAgent 属性！
            String backupAgent = sa.getNonConfigurationString(
                    com.android.internal.R.styleable.AndroidManifestApplication_backupAgent,
                    Configuration.NATIVE_CONFIG_VERSION);
            if (backupAgent != null) {
                ai.backupAgentName = buildClassName(pkgName, backupAgent, outError);
                if (DEBUG_BACKUP) {
                    Slog.v(TAG, "android:backupAgent = " + ai.backupAgentName
                            + " from " + pkgName + "+" + backupAgent);
                }

                if (sa.getBoolean(
                        com.android.internal.R.styleable.AndroidManifestApplication_killAfterRestore,
                        true)) {
                    // 解析 android:killAfterRestore 属性，为 true 设置 FLAG_KILL_AFTER_RESTORE 标志位
                    ai.flags |= ApplicationInfo.FLAG_KILL_AFTER_RESTORE;
                }
                if (sa.getBoolean(
                        com.android.internal.R.styleable.AndroidManifestApplication_restoreAnyVersion,
                        false)) {
                    // 解析 android:restoreAnyVersion 属性，为 true 设置 FLAG_RESTORE_ANY_VERSION 标志位
                    ai.flags |= ApplicationInfo.FLAG_RESTORE_ANY_VERSION;
                }
                if (sa.getBoolean(
                        com.android.internal.R.styleable.AndroidManifestApplication_fullBackupOnly,
                        false)) {
                    // 解析 android:fullBackupOnly 属性，为 true 设置 FLAG_FULL_BACKUP_ONLY 标志位
                    ai.flags |= ApplicationInfo.FLAG_FULL_BACKUP_ONLY;
                }
                if (sa.getBoolean(
                        com.android.internal.R.styleable.AndroidManifestApplication_backupInForeground,
                        false)) {
                    // 解析 android:backupInForeground 属性，为 true 设置 PRIVATE_FLAG_BACKUP_IN_FOREGROUND 标志位
                    ai.privateFlags |= ApplicationInfo.PRIVATE_FLAG_BACKUP_IN_FOREGROUND;
                }
            }
            
            // 解析 android:fullBackupContent 属性！
            TypedValue v = sa.peekValue(
                    com.android.internal.R.styleable.AndroidManifestApplication_fullBackupContent);
            if (v != null && (ai.fullBackupContent = v.resourceId) == 0) {
                if (DEBUG_BACKUP) {
                    Slog.v(TAG, "fullBackupContent specified as boolean=" +
                            (v.data == 0 ? "false" : "true"));
                }
                // "false" => -1, "true" => 0
                ai.fullBackupContent = (v.data == 0 ? -1 : 0);
            }
            if (DEBUG_BACKUP) {
                Slog.v(TAG, "fullBackupContent=" + ai.fullBackupContent + " for " + pkgName);
            }
        }

        // 解析 android:theme android:description 属性！
        ai.theme = sa.getResourceId(
                com.android.internal.R.styleable.AndroidManifestApplication_theme, 0);
        ai.descriptionRes = sa.getResourceId(
                com.android.internal.R.styleable.AndroidManifestApplication_description, 0);

        // 如果设置了 PARSE_IS_SYSTEM，并且 android:persistent 为 true
        // 那么 ai.flags 设置 ApplicationInfo.FLAG_PERSISTENT 标志位，其为 persistent 属性！
        if ((flags&PARSE_IS_SYSTEM) != 0) {
            if (sa.getBoolean(
                    com.android.internal.R.styleable.AndroidManifestApplication_persistent,
                    false)) {
                ai.flags |= ApplicationInfo.FLAG_PERSISTENT;
            }
        }
        // 解析 android:requiredForAllUsers 属性，为true，表示该应用在所有 user 下都可用！
        if (sa.getBoolean(
                com.android.internal.R.styleable.AndroidManifestApplication_requiredForAllUsers,
                false)) {
            owner.mRequiredForAllUsers = true;
        }

        // 解析 android:requiredAccountType android:restrictedAccountType 的属性！
        String restrictedAccountType = sa.getString(com.android.internal.R.styleable
                .AndroidManifestApplication_restrictedAccountType);
        if (restrictedAccountType != null && restrictedAccountType.length() > 0) {
            owner.mRestrictedAccountType = restrictedAccountType;
        }

        String requiredAccountType = sa.getString(com.android.internal.R.styleable
                .AndroidManifestApplication_requiredAccountType);
        if (requiredAccountType != null && requiredAccountType.length() > 0) {
            owner.mRequiredAccountType = requiredAccountType;
        }

        // 解析 android:debuggable android:vmSafeMode 属性！
        if (sa.getBoolean(
                com.android.internal.R.styleable.AndroidManifestApplication_debuggable,
                false)) {
            ai.flags |= ApplicationInfo.FLAG_DEBUGGABLE;
        }

        if (sa.getBoolean(
                com.android.internal.R.styleable.AndroidManifestApplication_vmSafeMode,
                false)) {
            ai.flags |= ApplicationInfo.FLAG_VM_SAFE_MODE;
        }
        // 解析 android:hardwareAccelerated 属性！
        owner.baseHardwareAccelerated = sa.getBoolean(
                com.android.internal.R.styleable.AndroidManifestApplication_hardwareAccelerated,
                owner.applicationInfo.targetSdkVersion >= Build.VERSION_CODES.ICE_CREAM_SANDWICH);
        if (owner.baseHardwareAccelerated) {
            ai.flags |= ApplicationInfo.FLAG_HARDWARE_ACCELERATED;
        }
        // 解析 android:hasCode 属性！
        if (sa.getBoolean(
                com.android.internal.R.styleable.AndroidManifestApplication_hasCode,
                true)) {
            ai.flags |= ApplicationInfo.FLAG_HAS_CODE;
        }

        // 解析 android:allowTaskReparenting 属性，这个属性很重要，决定了 acivity 和 task 的关系
        // 在 application 标签上设置该属性对内部的所有 activity 生效！
        if (sa.getBoolean(
                com.android.internal.R.styleable.AndroidManifestApplication_allowTaskReparenting,
                false)) {
            ai.flags |= ApplicationInfo.FLAG_ALLOW_TASK_REPARENTING;
        }
        
        // 解析 android:allowClearUserData 属性！
        if (sa.getBoolean(
                com.android.internal.R.styleable.AndroidManifestApplication_allowClearUserData,
                true)) {
            ai.flags |= ApplicationInfo.FLAG_ALLOW_CLEAR_USER_DATA;
        }

        // The parent package controls installation, hence specify test only installs.
        if (owner.parentPackage == null) {
            if (sa.getBoolean(
                    com.android.internal.R.styleable.AndroidManifestApplication_testOnly,
                    false)) {
                ai.flags |= ApplicationInfo.FLAG_TEST_ONLY;
            }
        }
        // 解析 android:largeHeap 属性！
        if (sa.getBoolean(
                com.android.internal.R.styleable.AndroidManifestApplication_largeHeap,
                false)) {
            ai.flags |= ApplicationInfo.FLAG_LARGE_HEAP;
        }
        // 解析 android:usesCleartextTraffic 属性！
        if (sa.getBoolean(
                com.android.internal.R.styleable.AndroidManifestApplication_usesCleartextTraffic,
                true)) {
            ai.flags |= ApplicationInfo.FLAG_USES_CLEARTEXT_TRAFFIC;
        }
        // 解析 android:supportsRtl 属性！
        if (sa.getBoolean(
                com.android.internal.R.styleable.AndroidManifestApplication_supportsRtl,
                false /* default is no RTL support*/)) {
            ai.flags |= ApplicationInfo.FLAG_SUPPORTS_RTL;
        }
        // 解析 android:multiArch 属性！
        if (sa.getBoolean(
                com.android.internal.R.styleable.AndroidManifestApplication_multiArch,
                false)) {
            ai.flags |= ApplicationInfo.FLAG_MULTIARCH;
        }
        // 解析 android:extractNativeLibs 属性！
        if (sa.getBoolean(
                com.android.internal.R.styleable.AndroidManifestApplication_extractNativeLibs,
                true)) {
            ai.flags |= ApplicationInfo.FLAG_EXTRACT_NATIVE_LIBS;
        }
        // 解析 android:defaultToDeviceProtectedStorage 属性！
        if (sa.getBoolean(
                R.styleable.AndroidManifestApplication_defaultToDeviceProtectedStorage,
                false)) {
            ai.privateFlags |= ApplicationInfo.PRIVATE_FLAG_DEFAULT_TO_DEVICE_PROTECTED_STORAGE;
        }
        // 解析 android:directBootAware 属性！
        if (sa.getBoolean(
                R.styleable.AndroidManifestApplication_directBootAware,
                false)) {
            ai.privateFlags |= ApplicationInfo.PRIVATE_FLAG_DIRECT_BOOT_AWARE;
        }
        // 解析 android:resizeableActivity 属性！
        if (sa.getBoolean(R.styleable.AndroidManifestApplication_resizeableActivity,
                owner.applicationInfo.targetSdkVersion >= Build.VERSION_CODES.N)) {
            ai.privateFlags |= PRIVATE_FLAG_RESIZEABLE_ACTIVITIES;
        }
        // 解析 android:networkSecurityConfig 属性！
        ai.networkSecurityConfigRes = sa.getResourceId(
                com.android.internal.R.styleable.AndroidManifestApplication_networkSecurityConfig,
                0);
        // 解析 android:permission 属性！
        String str;
        str = sa.getNonConfigurationString(
                com.android.internal.R.styleable.AndroidManifestApplication_permission, 0);
        ai.permission = (str != null && str.length() > 0) ? str.intern() : null;

        // 解析 android:taskAffinity 属性，这个属性很重要，决定了 acivity 和 task 的关系
        // 在 application 标签上设置该属性对内部的所有 activity 生效！
        if (owner.applicationInfo.targetSdkVersion >= Build.VERSION_CODES.FROYO) {
            str = sa.getNonConfigurationString(
                    com.android.internal.R.styleable.AndroidManifestApplication_taskAffinity,
                    Configuration.NATIVE_CONFIG_VERSION);
        } else {
            str = sa.getNonResourceString(
                    com.android.internal.R.styleable.AndroidManifestApplication_taskAffinity);
        }
        ai.taskAffinity = buildTaskAffinityName(ai.packageName, ai.packageName,
                str, outError);

        if (outError[0] == null) {
            CharSequence pname;
            // 解析 android:process 属性，这个属性很重要，决定了组件运行所在的进程名
            // 在 application 标签上设置该属性对内部的所有组件生效！
            if (owner.applicationInfo.targetSdkVersion >= Build.VERSION_CODES.FROYO) {
                pname = sa.getNonConfigurationString(
                        com.android.internal.R.styleable.AndroidManifestApplication_process,
                        Configuration.NATIVE_CONFIG_VERSION);
            } else {
             
                pname = sa.getNonResourceString(
                        com.android.internal.R.styleable.AndroidManifestApplication_process);
            }
            ai.processName = buildProcessName(ai.packageName, null, pname,
                    flags, mSeparateProcesses, outError);

            // 解析 android:isGame android:enabled 属性，
            ai.enabled = sa.getBoolean(
                    com.android.internal.R.styleable.AndroidManifestApplication_enabled, true);
            if (sa.getBoolean(
                    com.android.internal.R.styleable.AndroidManifestApplication_isGame, false)) {
                ai.flags |= ApplicationInfo.FLAG_IS_GAME;
            }

            if (false) {
                // 解析 android:cantSaveState 属性，这里由于 if 为 false，所以不会解析！
                // 主要用于 height-weight 类型的应用！
                if (sa.getBoolean(
                        com.android.internal.R.styleable.AndroidManifestApplication_cantSaveState,
                        false)) {
                    ai.privateFlags |= ApplicationInfo.PRIVATE_FLAG_CANT_SAVE_STATE;

                    // 对于 height-weight 类型的应用，其所在进程的进程名只能是包名，我们无法自定义其进程名！
                    if (ai.processName != null && ai.processName != ai.packageName) {
                        outError[0] = "cantSaveState applications can not use custom processes";
                    }
                }
            }
        }
        // 解析 android:uiOptions 属性！
        ai.uiOptions = sa.getInt(
                com.android.internal.R.styleable.AndroidManifestApplication_uiOptions, 0);

        sa.recycle();

        if (outError[0] != null) {
            mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
            return false;
        }
        //【2】解析四大组件！
        final int innerDepth = parser.getDepth();
        int type;
        while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
                && (type != XmlPullParser.END_TAG || parser.getDepth() > innerDepth)) {
            if (type == XmlPullParser.END_TAG || type == XmlPullParser.TEXT) {
                continue;
            }

            String tagName = parser.getName();

            //【2.1】解析 "activity" 组件
            if (tagName.equals("activity")) {
                Activity a = parseActivity(owner, res, parser, flags, outError, false,
                        owner.baseHardwareAccelerated);
                if (a == null) {
                    mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
                    return false;
                }

                owner.activities.add(a);

            //【2.2】解析 "receiver" 组件
            } else if (tagName.equals("receiver")) {
                Activity a = parseActivity(owner, res, parser, flags, outError, true, false);
                if (a == null) {
                    mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
                    return false;
                }

                owner.receivers.add(a);

            //【2.3】解析 "service" 组件
            } else if (tagName.equals("service")) {
                Service s = parseService(owner, res, parser, flags, outError);
                if (s == null) {
                    mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
                    return false;
                }

                owner.services.add(s);

            //【2.4】解析 "provider" 组件
            } else if (tagName.equals("provider")) {
                Provider p = parseProvider(owner, res, parser, flags, outError);
                if (p == null) {
                    mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
                    return false;
                }

                owner.providers.add(p);

            } else if (tagName.equals("activity-alias")) { 
                //【2.5】解析 "activity-alias" 组件
                Activity a = parseActivityAlias(owner, res, parser, flags, outError);
                if (a == null) {
                    mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
                    return false;
                }

                owner.activities.add(a);

            } else if (parser.getName().equals("meta-data")) {
                //【2.6】解析 "meta-data" 组件
                if ((owner.mAppMetaData = parseMetaData(res, parser, owner.mAppMetaData,
                        outError)) == null) {
                    mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
                    return false;
                }

            } else if (tagName.equals("library")) {
                //【2.6】解析 "library" 组件
                sa = res.obtainAttributes(parser,
                        com.android.internal.R.styleable.AndroidManifestLibrary);
                String lname = sa.getNonResourceString(
                        com.android.internal.R.styleable.AndroidManifestLibrary_name);

                sa.recycle();

                if (lname != null) {
                    lname = lname.intern();
                    if (!ArrayUtils.contains(owner.libraryNames, lname)) {
                        owner.libraryNames = ArrayUtils.add(owner.libraryNames, lname);
                    }
                }

                XmlUtils.skipCurrentTag(parser);

            } else if (tagName.equals("uses-library")) {
                //【2.7】解析 "uses-library" 组件
                sa = res.obtainAttributes(parser,
                        com.android.internal.R.styleable.AndroidManifestUsesLibrary);

                // Note: don't allow this value to be a reference to a resource
                // that may change.
                String lname = sa.getNonResourceString(
                        com.android.internal.R.styleable.AndroidManifestUsesLibrary_name);
                boolean req = sa.getBoolean(
                        com.android.internal.R.styleable.AndroidManifestUsesLibrary_required,
                        true);

                sa.recycle();

                if (lname != null) {
                    lname = lname.intern();
                    if (req) {
                        owner.usesLibraries = ArrayUtils.add(owner.usesLibraries, lname);
                    } else {
                        owner.usesOptionalLibraries = ArrayUtils.add(
                                owner.usesOptionalLibraries, lname);
                    }
                }

                XmlUtils.skipCurrentTag(parser);

            } else if (tagName.equals("uses-package")) {
                // Dependencies for app installers; we don't currently try to
                // enforce this.
                XmlUtils.skipCurrentTag(parser);

            } else {
                if (!RIGID_PARSER) {
                    Slog.w(TAG, "Unknown element under <application>: " + tagName
                            + " at " + mArchiveSourcePath + " "
                            + parser.getPositionDescription());
                    XmlUtils.skipCurrentTag(parser);
                    continue;
                } else {
                    outError[0] = "Bad element under <application>: " + tagName;
                    mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
                    return false;
                }
            }
        }

        modifySharedLibrariesForBackwardCompatibility(owner);

        if (hasDomainURLs(owner)) {
            owner.applicationInfo.privateFlags |= ApplicationInfo.PRIVATE_FLAG_HAS_DOMAIN_URLS;
        } else {
            owner.applicationInfo.privateFlags &= ~ApplicationInfo.PRIVATE_FLAG_HAS_DOMAIN_URLS;
        }

        return true;
    }

```
这里重点的是四大组件的解析：

###### 3.1.1.2.3.1 PParser.parseActivity - 解析 activity 和 receiver

我们来看看 activity 的解析过程：

```java
    private Activity parseActivity(Package owner, Resources res,
            XmlResourceParser parser, int flags, String[] outError,
            boolean receiver, boolean hardwareAccelerated)
            throws XmlPullParserException, IOException {
        TypedArray sa = res.obtainAttributes(parser, R.styleable.AndroidManifestActivity);
        // 解析 android:name android:label android:icon android:roundIcon android:logo android:banner 属性！
        if (mParseActivityArgs == null) {
            mParseActivityArgs = new ParseComponentArgs(owner, outError,
                    R.styleable.AndroidManifestActivity_name,
                    R.styleable.AndroidManifestActivity_label,
                    R.styleable.AndroidManifestActivity_icon,
                    R.styleable.AndroidManifestActivity_roundIcon,
                    R.styleable.AndroidManifestActivity_logo,
                    R.styleable.AndroidManifestActivity_banner,
                    mSeparateProcesses,
                    R.styleable.AndroidManifestActivity_process,
                    R.styleable.AndroidManifestActivity_description,
                    R.styleable.AndroidManifestActivity_enabled);
        }
        // 由于 activty 和 receiver 使用的同一个方法，所以需要区分解析的是哪个，此时 receiver 为 false！
        mParseActivityArgs.tag = receiver ? "<receiver>" : "<activity>";
        mParseActivityArgs.sa = sa;
        mParseActivityArgs.flags = flags;

        //【2】创建了一个 Activity 对象！
        Activity a = new Activity(mParseActivityArgs, new ActivityInfo());
        if (outError[0] != null) {
            sa.recycle();
            return null;
        }
        // 解析 android:exported 属性！
        boolean setExported = sa.hasValue(R.styleable.AndroidManifestActivity_exported);
        if (setExported) {
            a.info.exported = sa.getBoolean(R.styleable.AndroidManifestActivity_exported, false);
        }
        // 解析 android:theme 属性！
        a.info.theme = sa.getResourceId(R.styleable.AndroidManifestActivity_theme, 0);
        // 解析 android:uiOptions 属性！
        a.info.uiOptions = sa.getInt(R.styleable.AndroidManifestActivity_uiOptions,
                a.info.applicationInfo.uiOptions);
        // 解析 android:parentActivityName 属性！
        String parentName = sa.getNonConfigurationString(
                R.styleable.AndroidManifestActivity_parentActivityName,
                Configuration.NATIVE_CONFIG_VERSION);
        if (parentName != null) {
            String parentClassName = buildClassName(a.info.packageName, parentName, outError);
            if (outError[0] == null) {
                a.info.parentActivityName = parentClassName;
            } else {
                Log.e(TAG, "Activity " + a.info.name + " specified invalid parentActivityName " +
                        parentName);
                outError[0] = null;
            }
        }
        // 解析 android:permission 属性！
        String str;
        str = sa.getNonConfigurationString(R.styleable.AndroidManifestActivity_permission, 0);
        if (str == null) {
            a.info.permission = owner.applicationInfo.permission;
        } else {
            a.info.permission = str.length() > 0 ? str.toString().intern() : null;
        }
        // 解析 android:taskAffinity 属性，这个属性真重要，能决定该 acitivty 和 task 的关系！！
        str = sa.getNonConfigurationString(
                R.styleable.AndroidManifestActivity_taskAffinity,
                Configuration.NATIVE_CONFIG_VERSION);
        a.info.taskAffinity = buildTaskAffinityName(owner.applicationInfo.packageName,
                owner.applicationInfo.taskAffinity, str, outError);

        a.info.flags = 0;
        // 解析 android:multiprocess 属性
        if (sa.getBoolean(
                R.styleable.AndroidManifestActivity_multiprocess, false)) {
            a.info.flags |= ActivityInfo.FLAG_MULTIPROCESS;
        }
        // 解析 android:finishOnTaskLaunch 属性，这个属性真重要，能决定该 acitivty 和 task 的关系！！
        if (sa.getBoolean(R.styleable.AndroidManifestActivity_finishOnTaskLaunch, false)) {
            a.info.flags |= ActivityInfo.FLAG_FINISH_ON_TASK_LAUNCH;
        }
        // 解析 android:clearTaskOnLaunch 属性，这个属性真重要，能决定该 acitivty 和 task 的关系！
        if (sa.getBoolean(R.styleable.AndroidManifestActivity_clearTaskOnLaunch, false)) {
            a.info.flags |= ActivityInfo.FLAG_CLEAR_TASK_ON_LAUNCH;
        }
        // 解析 android:noHistory 属性，这个属性真重要，能决定该 acitivty 和 task 的关系！
        if (sa.getBoolean(R.styleable.AndroidManifestActivity_noHistory, false)) {
            a.info.flags |= ActivityInfo.FLAG_NO_HISTORY;
        }
        // 解析 android:alwaysRetainTaskState 属性，这个属性真重要，能决定该 acitivty 和 task 的关系！
        if (sa.getBoolean(R.styleable.AndroidManifestActivity_alwaysRetainTaskState, false)) {
            a.info.flags |= ActivityInfo.FLAG_ALWAYS_RETAIN_TASK_STATE;
        }
        // 解析 android:stateNotNeeded 属性
        if (sa.getBoolean(R.styleable.AndroidManifestActivity_stateNotNeeded, false)) {
            a.info.flags |= ActivityInfo.FLAG_STATE_NOT_NEEDED;
        }
        // 解析 android:excludeFromRecents 属性，不再最近任务显示
        if (sa.getBoolean(R.styleable.AndroidManifestActivity_excludeFromRecents, false)) {
            a.info.flags |= ActivityInfo.FLAG_EXCLUDE_FROM_RECENTS;
        }
        // 解析 android:allowTaskReparenting 属性，这个属性真重要，能决定该 acitivty 和 task 的关系！
        if (sa.getBoolean(R.styleable.AndroidManifestActivity_allowTaskReparenting,
                (owner.applicationInfo.flags&ApplicationInfo.FLAG_ALLOW_TASK_REPARENTING) != 0)) {
            a.info.flags |= ActivityInfo.FLAG_ALLOW_TASK_REPARENTING;
        }
        // 解析 android:finishOnCloseSystemDialogs 属性
        if (sa.getBoolean(R.styleable.AndroidManifestActivity_finishOnCloseSystemDialogs, false)) {
            a.info.flags |= ActivityInfo.FLAG_FINISH_ON_CLOSE_SYSTEM_DIALOGS;
        }
        // 解析 android:showOnLockScreen 属性
        if (sa.getBoolean(R.styleable.AndroidManifestActivity_showOnLockScreen, false)
                || sa.getBoolean(R.styleable.AndroidManifestActivity_showForAllUsers, false)) {
            a.info.flags |= ActivityInfo.FLAG_SHOW_FOR_ALL_USERS;
        }
        // 解析 android:immersive 属性
        if (sa.getBoolean(R.styleable.AndroidManifestActivity_immersive, false)) {
            a.info.flags |= ActivityInfo.FLAG_IMMERSIVE;
        }
        // 解析 android:systemUserOnly 属性
        if (sa.getBoolean(R.styleable.AndroidManifestActivity_systemUserOnly, false)) {
            a.info.flags |= ActivityInfo.FLAG_SYSTEM_USER_ONLY;
        }
        // 接下来，对于 activity 和 receiver 解析不同的属性！
        if (!receiver) {
            // 对于 activity 进入这个！
            // 解析 android:hardwareAccelerated 属性
            if (sa.getBoolean(R.styleable.AndroidManifestActivity_hardwareAccelerated,
                    hardwareAccelerated)) {
                a.info.flags |= ActivityInfo.FLAG_HARDWARE_ACCELERATED;
            }
            // 解析 android:launchMode 属性
            a.info.launchMode = sa.getInt(
                    R.styleable.AndroidManifestActivity_launchMode, ActivityInfo.LAUNCH_MULTIPLE);
            // 解析 android:documentLaunchMode 属性
            a.info.documentLaunchMode = sa.getInt(
                    R.styleable.AndroidManifestActivity_documentLaunchMode,
                    ActivityInfo.DOCUMENT_LAUNCH_NONE);
            // 解析 android:maxRecents 属性
            a.info.maxRecents = sa.getInt(
                    R.styleable.AndroidManifestActivity_maxRecents,
                    ActivityManager.getDefaultAppRecentsLimitStatic());
            // 解析 android:configChanges 属性
            a.info.configChanges = sa.getInt(R.styleable.AndroidManifestActivity_configChanges, 0);
            // 解析 android:windowSoftInputMode 属性
            a.info.softInputMode = sa.getInt(
                    R.styleable.AndroidManifestActivity_windowSoftInputMode, 0);
            // 解析 android:persistableMode 属性
            a.info.persistableMode = sa.getInteger(
                    R.styleable.AndroidManifestActivity_persistableMode,
                    ActivityInfo.PERSIST_ROOT_ONLY);
            // 解析 android:allowEmbedded 属性
            if (sa.getBoolean(R.styleable.AndroidManifestActivity_allowEmbedded, false)) {
                a.info.flags |= ActivityInfo.FLAG_ALLOW_EMBEDDED;
            }
            // 解析 android:autoRemoveFromRecents 属性
            if (sa.getBoolean(R.styleable.AndroidManifestActivity_autoRemoveFromRecents, false)) {
                a.info.flags |= ActivityInfo.FLAG_AUTO_REMOVE_FROM_RECENTS;
            }
            // 解析 android:relinquishTaskIdentity 属性
            if (sa.getBoolean(R.styleable.AndroidManifestActivity_relinquishTaskIdentity, false)) {
                a.info.flags |= ActivityInfo.FLAG_RELINQUISH_TASK_IDENTITY;
            }
            // 解析 android:resumeWhilePausing 属性
            if (sa.getBoolean(R.styleable.AndroidManifestActivity_resumeWhilePausing, false)) {
                a.info.flags |= ActivityInfo.FLAG_RESUME_WHILE_PAUSING;
            }
            // 解析 android:screenOrientation 属性
            a.info.screenOrientation = sa.getInt(
                    R.styleable.AndroidManifestActivity_screenOrientation,
                    SCREEN_ORIENTATION_UNSPECIFIED);
            // 解析 android:resizeableActivity android:supportsPictureInPicture 属性
            a.info.resizeMode = RESIZE_MODE_UNRESIZEABLE;
            final boolean appDefault = (owner.applicationInfo.privateFlags
                    & PRIVATE_FLAG_RESIZEABLE_ACTIVITIES) != 0;
            final boolean resizeableSetExplicitly
                    = sa.hasValue(R.styleable.AndroidManifestActivity_resizeableActivity);
            final boolean resizeable = sa.getBoolean(
                    R.styleable.AndroidManifestActivity_resizeableActivity, appDefault);

            if (resizeable) {
                if (sa.getBoolean(R.styleable.AndroidManifestActivity_supportsPictureInPicture,
                        false)) {
                    a.info.resizeMode = RESIZE_MODE_RESIZEABLE_AND_PIPABLE;
                } else {
                    a.info.resizeMode = RESIZE_MODE_RESIZEABLE;
                }
            } else if (owner.applicationInfo.targetSdkVersion >= Build.VERSION_CODES.N
                    || resizeableSetExplicitly) {
                a.info.resizeMode = RESIZE_MODE_UNRESIZEABLE;
            } else if (!a.info.isFixedOrientation() && (a.info.flags & FLAG_IMMERSIVE) == 0) {
                a.info.resizeMode = RESIZE_MODE_FORCE_RESIZEABLE;
            }
            // 解析 android:alwaysFocusable 属性
            if (sa.getBoolean(R.styleable.AndroidManifestActivity_alwaysFocusable, false)) {
                a.info.flags |= FLAG_ALWAYS_FOCUSABLE;
            }
            // 解析 android:lockTaskMode 属性
            a.info.lockTaskLaunchMode =
                    sa.getInt(R.styleable.AndroidManifestActivity_lockTaskMode, 0);
            // 解析 android:directBootAware 属性
            a.info.encryptionAware = a.info.directBootAware = sa.getBoolean(
                    R.styleable.AndroidManifestActivity_directBootAware,
                    false);
            // 解析 android:enableVrMode 属性
            a.info.requestedVrComponent =
                sa.getString(R.styleable.AndroidManifestActivity_enableVrMode);
        } else {
            // 对于 receiver，会进入这里！
            a.info.launchMode = ActivityInfo.LAUNCH_MULTIPLE;
            a.info.configChanges = 0;
            // 解析 android:singleUser 属性
            if (sa.getBoolean(R.styleable.AndroidManifestActivity_singleUser, false)) {
                a.info.flags |= ActivityInfo.FLAG_SINGLE_USER;
                if (a.info.exported && (flags & PARSE_IS_PRIVILEGED) == 0) {
                    Slog.w(TAG, "Activity exported request ignored due to singleUser: "
                            + a.className + " at " + mArchiveSourcePath + " "
                            + parser.getPositionDescription());
                    a.info.exported = false;
                    setExported = true;
                }
            }
            // 解析 android:directBootAware 属性
            a.info.encryptionAware = a.info.directBootAware = sa.getBoolean(
                    R.styleable.AndroidManifestActivity_directBootAware,
                    false);
        }

        if (a.info.directBootAware) {
            owner.applicationInfo.privateFlags |=
                    ApplicationInfo.PRIVATE_FLAG_PARTIALLY_DIRECT_BOOT_AWARE;
        }

        sa.recycle();

        // 如果解析的是 receiver，针对 height-weight 类型的进程做处理
        if (receiver && (owner.applicationInfo.privateFlags
                &ApplicationInfo.PRIVATE_FLAG_CANT_SAVE_STATE) != 0) {
            if (a.info.processName == owner.packageName) {
                outError[0] = "Heavy-weight applications can not have receivers in main process";
            }
        }

        if (outError[0] != null) {
            return null;
        }

        int outerDepth = parser.getDepth();
        int type;
        while ((type=parser.next()) != XmlPullParser.END_DOCUMENT
               && (type != XmlPullParser.END_TAG
                       || parser.getDepth() > outerDepth)) {
            if (type == XmlPullParser.END_TAG || type == XmlPullParser.TEXT) {
                continue;
            }

            if (parser.getName().equals("intent-filter")) { // 解析 "intent-filter"
                ActivityIntentInfo intent = new ActivityIntentInfo(a);
                if (!parseIntent(res, parser, true, true, intent, outError)) {
                    return null;
                }
                if (intent.countActions() == 0) {
                    Slog.w(TAG, "No actions in intent filter at "
                            + mArchiveSourcePath + " "
                            + parser.getPositionDescription());
                } else {
                    a.intents.add(intent);
                }
            } else if (!receiver && parser.getName().equals("preferred")) { // 解析 "preferred"
                ActivityIntentInfo intent = new ActivityIntentInfo(a);
                if (!parseIntent(res, parser, false, false, intent, outError)) {
                    return null;
                }
                if (intent.countActions() == 0) {
                    Slog.w(TAG, "No actions in preferred at "
                            + mArchiveSourcePath + " "
                            + parser.getPositionDescription());
                } else {
                    if (owner.preferredActivityFilters == null) {
                        owner.preferredActivityFilters = new ArrayList<ActivityIntentInfo>();
                    }
                    owner.preferredActivityFilters.add(intent);
                }
            } else if (parser.getName().equals("meta-data")) { // 解析 "meta-data"
                if ((a.metaData = parseMetaData(res, parser, a.metaData,
                        outError)) == null) {
                    return null;
                }
            } else if (!receiver && parser.getName().equals("layout")) { // 解析 "layout"
                parseLayout(res, parser, a);
            } else {
                if (!RIGID_PARSER) {
                    Slog.w(TAG, "Problem in package " + mArchiveSourcePath + ":");
                    if (receiver) {
                        Slog.w(TAG, "Unknown element under <receiver>: " + parser.getName()
                                + " at " + mArchiveSourcePath + " "
                                + parser.getPositionDescription());
                    } else {
                        Slog.w(TAG, "Unknown element under <activity>: " + parser.getName()
                                + " at " + mArchiveSourcePath + " "
                                + parser.getPositionDescription());
                    }
                    XmlUtils.skipCurrentTag(parser);
                    continue;
                } else {
                    if (receiver) {
                        outError[0] = "Bad element under <receiver>: " + parser.getName();
                    } else {
                        outError[0] = "Bad element under <activity>: " + parser.getName();
                    }
                    return null;
                }
            }
        }

        if (!setExported) {
            a.info.exported = a.intents.size() > 0;
        }

        return a;
    }
```

整个解析过程很清晰，就是不断读取对应标签的属性，然后设置 Activity 的属性和标志位！

###### 3.1.1.2.3.2 PParser.parseService - 解析 service

```java
    private Service parseService(Package owner, Resources res,
            XmlResourceParser parser, int flags, String[] outError)
            throws XmlPullParserException, IOException {
        TypedArray sa = res.obtainAttributes(parser,
                com.android.internal.R.styleable.AndroidManifestService);
        // 解析 android:name android:label android:icon android:roundIcon android:logo android:banner 属性！
        if (mParseServiceArgs == null) {
            mParseServiceArgs = new ParseComponentArgs(owner, outError,
                    com.android.internal.R.styleable.AndroidManifestService_name,
                    com.android.internal.R.styleable.AndroidManifestService_label,
                    com.android.internal.R.styleable.AndroidManifestService_icon,
                    com.android.internal.R.styleable.AndroidManifestService_roundIcon,
                    com.android.internal.R.styleable.AndroidManifestService_logo,
                    com.android.internal.R.styleable.AndroidManifestService_banner,
                    mSeparateProcesses,
                    com.android.internal.R.styleable.AndroidManifestService_process,
                    com.android.internal.R.styleable.AndroidManifestService_description,
                    com.android.internal.R.styleable.AndroidManifestService_enabled);
            mParseServiceArgs.tag = "<service>";
        }

        mParseServiceArgs.sa = sa;
        mParseServiceArgs.flags = flags;
        // 创建 Service 对象！
        Service s = new Service(mParseServiceArgs, new ServiceInfo());
        if (outError[0] != null) {
            sa.recycle();
            return null;
        }
        // 解析 android:exported 属性！
        boolean setExported = sa.hasValue(
                com.android.internal.R.styleable.AndroidManifestService_exported);
        if (setExported) {
            s.info.exported = sa.getBoolean(
                    com.android.internal.R.styleable.AndroidManifestService_exported, false);
        }
        // 解析 android:permission 属性！
        String str = sa.getNonConfigurationString(
                com.android.internal.R.styleable.AndroidManifestService_permission, 0);
        if (str == null) {
            s.info.permission = owner.applicationInfo.permission;
        } else {
            s.info.permission = str.length() > 0 ? str.toString().intern() : null;
        }
        // 解析 android:stopWithTask 属性！
        s.info.flags = 0;
        if (sa.getBoolean(
                com.android.internal.R.styleable.AndroidManifestService_stopWithTask,
                false)) {
            s.info.flags |= ServiceInfo.FLAG_STOP_WITH_TASK;
        }
        // 解析 android:isolatedProcess 属性！
        if (sa.getBoolean(
                com.android.internal.R.styleable.AndroidManifestService_isolatedProcess,
                false)) {
            s.info.flags |= ServiceInfo.FLAG_ISOLATED_PROCESS;
        }
        // 解析 android:externalService 属性！
        if (sa.getBoolean(
                com.android.internal.R.styleable.AndroidManifestService_externalService,
                false)) {
            s.info.flags |= ServiceInfo.FLAG_EXTERNAL_SERVICE;
        }
        // 解析 android:singleUser 属性！
        if (sa.getBoolean(
                com.android.internal.R.styleable.AndroidManifestService_singleUser,
                false)) {
            s.info.flags |= ServiceInfo.FLAG_SINGLE_USER;
            if (s.info.exported && (flags & PARSE_IS_PRIVILEGED) == 0) {
                Slog.w(TAG, "Service exported request ignored due to singleUser: "
                        + s.className + " at " + mArchiveSourcePath + " "
                        + parser.getPositionDescription());
                s.info.exported = false;
                setExported = true;
            }
        }
        // 解析 android:directBootAware 属性！
        s.info.encryptionAware = s.info.directBootAware = sa.getBoolean(
                R.styleable.AndroidManifestService_directBootAware,
                false);
        if (s.info.directBootAware) {
            owner.applicationInfo.privateFlags |=
                    ApplicationInfo.PRIVATE_FLAG_PARTIALLY_DIRECT_BOOT_AWARE;
        }

        sa.recycle();
        // 如果应用属于 height-weight 类型的进程，要对进程名做处理！
        if ((owner.applicationInfo.privateFlags&ApplicationInfo.PRIVATE_FLAG_CANT_SAVE_STATE)
                != 0) {
            if (s.info.processName == owner.packageName) {
                outError[0] = "Heavy-weight applications can not have services in main process";
                return null;
            }
        }

        int outerDepth = parser.getDepth();
        int type;
        while ((type=parser.next()) != XmlPullParser.END_DOCUMENT
               && (type != XmlPullParser.END_TAG
                       || parser.getDepth() > outerDepth)) {
            if (type == XmlPullParser.END_TAG || type == XmlPullParser.TEXT) {
                continue;
            }

            if (parser.getName().equals("intent-filter")) { // 解析 "intent-filter"！
                ServiceIntentInfo intent = new ServiceIntentInfo(s);
                if (!parseIntent(res, parser, true, false, intent, outError)) {
                    return null;
                }

                s.intents.add(intent);
            } else if (parser.getName().equals("meta-data")) {
                if ((s.metaData=parseMetaData(res, parser, s.metaData,
                        outError)) == null) {
                    return null;
                }
            } else {
                if (!RIGID_PARSER) {
                    Slog.w(TAG, "Unknown element under <service>: "
                            + parser.getName() + " at " + mArchiveSourcePath + " "
                            + parser.getPositionDescription());
                    XmlUtils.skipCurrentTag(parser);
                    continue;
                } else {
                    outError[0] = "Bad element under <service>: " + parser.getName();
                    return null;
                }
            }
        }

        if (!setExported) {
            s.info.exported = s.intents.size() > 0;
        }

        return s;
    }
```




###### 3.1.1.2.3.3 PParser.parseProvider - 解析 parseProvider

接下来是解析 provider：
```java
    private Provider parseProvider(Package owner, Resources res,
            XmlResourceParser parser, int flags, String[] outError)
            throws XmlPullParserException, IOException {
        TypedArray sa = res.obtainAttributes(parser,
                com.android.internal.R.styleable.AndroidManifestProvider);
        // 解析 android:name android:label android:icon android:roundIcon android:logo android:banner 属性！
        if (mParseProviderArgs == null) {
            mParseProviderArgs = new ParseComponentArgs(owner, outError,
                    com.android.internal.R.styleable.AndroidManifestProvider_name,
                    com.android.internal.R.styleable.AndroidManifestProvider_label,
                    com.android.internal.R.styleable.AndroidManifestProvider_icon,
                    com.android.internal.R.styleable.AndroidManifestProvider_roundIcon,
                    com.android.internal.R.styleable.AndroidManifestProvider_logo,
                    com.android.internal.R.styleable.AndroidManifestProvider_banner,
                    mSeparateProcesses,
                    com.android.internal.R.styleable.AndroidManifestProvider_process,
                    com.android.internal.R.styleable.AndroidManifestProvider_description,
                    com.android.internal.R.styleable.AndroidManifestProvider_enabled);
            mParseProviderArgs.tag = "<provider>";
        }

        mParseProviderArgs.sa = sa;
        mParseProviderArgs.flags = flags;
        // 创建一个 Provider 对象！
        Provider p = new Provider(mParseProviderArgs, new ProviderInfo());
        if (outError[0] != null) {
            sa.recycle();
            return null;
        }

        boolean providerExportedDefault = false;

        if (owner.applicationInfo.targetSdkVersion < Build.VERSION_CODES.JELLY_BEAN_MR1) {
            // For compatibility, applications targeting API level 16 or lower
            // should have their content providers exported by default, unless they
            // specify otherwise.
            providerExportedDefault = true;
        }
        // 解析 android:exported 属性！
        p.info.exported = sa.getBoolean(
                com.android.internal.R.styleable.AndroidManifestProvider_exported,
                providerExportedDefault);
        // 解析 android:authorities 属性！
        String cpname = sa.getNonConfigurationString(
                com.android.internal.R.styleable.AndroidManifestProvider_authorities, 0);
        // 解析 android:syncable 属性！
        p.info.isSyncable = sa.getBoolean(
                com.android.internal.R.styleable.AndroidManifestProvider_syncable,
                false);
        // 解析 android:permission 属性！
        String permission = sa.getNonConfigurationString(
                com.android.internal.R.styleable.AndroidManifestProvider_permission, 0);
        // 解析 android:readPermission 属性！
        String str = sa.getNonConfigurationString(
                com.android.internal.R.styleable.AndroidManifestProvider_readPermission, 0);
        if (str == null) {
            str = permission;
        }
        if (str == null) {
            p.info.readPermission = owner.applicationInfo.permission;
        } else {
            p.info.readPermission =
                str.length() > 0 ? str.toString().intern() : null;
        }
        // 解析 android:writePermission 属性！
        str = sa.getNonConfigurationString(
                com.android.internal.R.styleable.AndroidManifestProvider_writePermission, 0);
        if (str == null) {
            str = permission;
        }
        if (str == null) {
            p.info.writePermission = owner.applicationInfo.permission;
        } else {
            p.info.writePermission =
                str.length() > 0 ? str.toString().intern() : null;
        }
        // 解析 android:grantUriPermissions 属性！
        p.info.grantUriPermissions = sa.getBoolean(
                com.android.internal.R.styleable.AndroidManifestProvider_grantUriPermissions,
                false);
        // 解析 android:multiprocess 属性！
        p.info.multiprocess = sa.getBoolean(
                com.android.internal.R.styleable.AndroidManifestProvider_multiprocess,
                false);
        // 解析 android:initOrder 属性！
        p.info.initOrder = sa.getInt(
                com.android.internal.R.styleable.AndroidManifestProvider_initOrder,
                0);

        p.info.flags = 0;
        // 解析 android:singleUser 属性！
        if (sa.getBoolean(
                com.android.internal.R.styleable.AndroidManifestProvider_singleUser,
                false)) {
            p.info.flags |= ProviderInfo.FLAG_SINGLE_USER;
            if (p.info.exported && (flags & PARSE_IS_PRIVILEGED) == 0) {
                Slog.w(TAG, "Provider exported request ignored due to singleUser: "
                        + p.className + " at " + mArchiveSourcePath + " "
                        + parser.getPositionDescription());
                p.info.exported = false;
            }
        }
        // 解析 android:directBootAware 属性！
        p.info.encryptionAware = p.info.directBootAware = sa.getBoolean(
                R.styleable.AndroidManifestProvider_directBootAware,
                false);
        if (p.info.directBootAware) {
            owner.applicationInfo.privateFlags |=
                    ApplicationInfo.PRIVATE_FLAG_PARTIALLY_DIRECT_BOOT_AWARE;
        }

        sa.recycle();
        // 如果应用设置为 height-weight 类型的应用，对进程名要做校验！
        if ((owner.applicationInfo.privateFlags&ApplicationInfo.PRIVATE_FLAG_CANT_SAVE_STATE)
                != 0) {
            if (p.info.processName == owner.packageName) {
                outError[0] = "Heavy-weight applications can not have providers in main process";
                return null;
            }
        }

        if (cpname == null) {
            outError[0] = "<provider> does not include authorities attribute";
            return null;
        }
        if (cpname.length() <= 0) {
            outError[0] = "<provider> has empty authorities attribute";
            return null;
        }
        p.info.authority = cpname.intern();

        if (!parseProviderTags(res, parser, p, outError)) {
            return null;
        }

        return p;
    }

```
对 provider 的解析就分析到这里！

###### 3.1.1.2.3.4 PParser.parseIntent - 解析 intent-filter

接下来，我们来看看对 activity receiver provider 的 intent-filter 的解析！

```java
    private boolean parseIntent(Resources res, XmlResourceParser parser,
            boolean allowGlobs, boolean allowAutoVerify, IntentInfo outInfo, String[] outError)
            throws XmlPullParserException, IOException {

        TypedArray sa = res.obtainAttributes(parser,
                com.android.internal.R.styleable.AndroidManifestIntentFilter);
        // 解析 android:priority 属性！
        int priority = sa.getInt(
                com.android.internal.R.styleable.AndroidManifestIntentFilter_priority, 0);
        outInfo.setPriority(priority);
        // 解析 android:label 属性！
        TypedValue v = sa.peekValue(
                com.android.internal.R.styleable.AndroidManifestIntentFilter_label);
        if (v != null && (outInfo.labelRes=v.resourceId) == 0) {
            outInfo.nonLocalizedLabel = v.coerceToString();
        }
        // 解析 android:useRoundIcon 属性！
        final boolean useRoundIcon =
                Resources.getSystem().getBoolean(com.android.internal.R.bool.config_useRoundIcon);
        // 解析 android:roundIcon android:icon 属性！
        int roundIconVal = useRoundIcon ? sa.getResourceId(
                com.android.internal.R.styleable.AndroidManifestIntentFilter_roundIcon, 0) : 0;
        if (roundIconVal != 0) {
            outInfo.icon = roundIconVal;
        } else {
            outInfo.icon = sa.getResourceId(
                    com.android.internal.R.styleable.AndroidManifestIntentFilter_icon, 0);
        }
        // 解析 android:logo 属性！
        outInfo.logo = sa.getResourceId(
                com.android.internal.R.styleable.AndroidManifestIntentFilter_logo, 0);
        // 解析 android:banner 属性！
        outInfo.banner = sa.getResourceId(
                com.android.internal.R.styleable.AndroidManifestIntentFilter_banner, 0);
        // 解析 android:autoVerify 属性！
        if (allowAutoVerify) {
            outInfo.setAutoVerify(sa.getBoolean(
                    com.android.internal.R.styleable.AndroidManifestIntentFilter_autoVerify,
                    false));
        }

        sa.recycle();

        int outerDepth = parser.getDepth();
        int type;
        while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
                && (type != XmlPullParser.END_TAG || parser.getDepth() > outerDepth)) {
            if (type == XmlPullParser.END_TAG || type == XmlPullParser.TEXT) {
                continue;
            }

            String nodeName = parser.getName();
            if (nodeName.equals("action")) { // 解析 "action" 标签！
                String value = parser.getAttributeValue(
                        ANDROID_RESOURCES, "name");
                if (value == null || value == "") {
                    outError[0] = "No value supplied for <android:name>";
                    return false;
                }
                XmlUtils.skipCurrentTag(parser);

                outInfo.addAction(value);
            } else if (nodeName.equals("category")) { // 解析 "category" 标签！
                String value = parser.getAttributeValue(
                        ANDROID_RESOURCES, "name");
                if (value == null || value == "") {
                    outError[0] = "No value supplied for <android:name>";
                    return false;
                }
                XmlUtils.skipCurrentTag(parser);

                outInfo.addCategory(value);

            } else if (nodeName.equals("data")) {  // 解析 "data" 标签！
                sa = res.obtainAttributes(parser,
                        com.android.internal.R.styleable.AndroidManifestData);
                // 解析 android:mimeType 属性！
                String str = sa.getNonConfigurationString(
                        com.android.internal.R.styleable.AndroidManifestData_mimeType, 0);
                if (str != null) {
                    try {
                        outInfo.addDataType(str);
                    } catch (IntentFilter.MalformedMimeTypeException e) {
                        outError[0] = e.toString();
                        sa.recycle();
                        return false;
                    }
                }
                // 解析 android:scheme 属性！
                str = sa.getNonConfigurationString(
                        com.android.internal.R.styleable.AndroidManifestData_scheme, 0);
                if (str != null) {
                    outInfo.addDataScheme(str);
                }
                // 解析 android:ssp 属性！
                str = sa.getNonConfigurationString(
                        com.android.internal.R.styleable.AndroidManifestData_ssp, 0);
                if (str != null) {
                    outInfo.addDataSchemeSpecificPart(str, PatternMatcher.PATTERN_LITERAL);
                }
                // 解析 android:sspPrefix 属性！
                str = sa.getNonConfigurationString(
                        com.android.internal.R.styleable.AndroidManifestData_sspPrefix, 0);
                if (str != null) {
                    outInfo.addDataSchemeSpecificPart(str, PatternMatcher.PATTERN_PREFIX);
                }
                // 解析 android:sspPattern 属性！
                str = sa.getNonConfigurationString(
                        com.android.internal.R.styleable.AndroidManifestData_sspPattern, 0);
                if (str != null) {
                    if (!allowGlobs) {
                        outError[0] = "sspPattern not allowed here; ssp must be literal";
                        return false;
                    }
                    outInfo.addDataSchemeSpecificPart(str, PatternMatcher.PATTERN_SIMPLE_GLOB);
                }
                // 解析 android:host android:port 属性！
                String host = sa.getNonConfigurationString(
                        com.android.internal.R.styleable.AndroidManifestData_host, 0);
                String port = sa.getNonConfigurationString(
                        com.android.internal.R.styleable.AndroidManifestData_port, 0);
                if (host != null) {
                    outInfo.addDataAuthority(host, port);
                }
                // 解析 android:path 属性！
                str = sa.getNonConfigurationString(
                        com.android.internal.R.styleable.AndroidManifestData_path, 0);
                if (str != null) {
                    outInfo.addDataPath(str, PatternMatcher.PATTERN_LITERAL);
                }
                // 解析 android:pathPrefix 属性！
                str = sa.getNonConfigurationString(
                        com.android.internal.R.styleable.AndroidManifestData_pathPrefix, 0);
                if (str != null) {
                    outInfo.addDataPath(str, PatternMatcher.PATTERN_PREFIX);
                }
                // 解析 android:pathPattern 属性！
                str = sa.getNonConfigurationString(
                        com.android.internal.R.styleable.AndroidManifestData_pathPattern, 0);
                if (str != null) {
                    if (!allowGlobs) {
                        outError[0] = "pathPattern not allowed here; path must be literal";
                        return false;
                    }
                    outInfo.addDataPath(str, PatternMatcher.PATTERN_SIMPLE_GLOB);
                }

                sa.recycle();
                XmlUtils.skipCurrentTag(parser);
            } else if (!RIGID_PARSER) {
                Slog.w(TAG, "Unknown element under <intent-filter>: "
                        + parser.getName() + " at " + mArchiveSourcePath + " "
                        + parser.getPositionDescription());
                XmlUtils.skipCurrentTag(parser);
            } else {
                outError[0] = "Bad element under <intent-filter>: " + parser.getName();
                return false;
            }
        }

        outInfo.hasDefault = outInfo.hasCategory(Intent.CATEGORY_DEFAULT);

        if (DEBUG_PARSER) {
            final StringBuilder cats = new StringBuilder("Intent d=");
            cats.append(outInfo.hasDefault);
            cats.append(", cat=");

            final Iterator<String> it = outInfo.categoriesIterator();
            if (it != null) {
                while (it.hasNext()) {
                    cats.append(' ');
                    cats.append(it.next());
                }
            }
            Slog.d(TAG, cats.toString());
        }

        return true;
    }

```
对 intent-filter 的解析就到这里！

#### 3.1.1.3 PParser.parseSplitApk\[4]

核心 apk 解析完成后，会返回一个 Package 对象，传入 parseSplitApk，用于解析非核心 apk：
```java
    private void parseSplitApk(Package pkg, int splitIndex, AssetManager assets, int flags)
            throws PackageParserException {
        final String apkPath = pkg.splitCodePaths[splitIndex];

        mParseError = PackageManager.INSTALL_SUCCEEDED;
        mArchiveSourcePath = apkPath;

        if (DEBUG_JAR) Slog.d(TAG, "Scanning split APK: " + apkPath);

        final int cookie = loadApkIntoAssetManager(assets, apkPath, flags);

        Resources res = null;
        XmlResourceParser parser = null;

        try {
            res = new Resources(assets, mMetrics, null);
            assets.setConfiguration(0, 0, null, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
                    Build.VERSION.RESOURCES_SDK_INT);

            // 用于解析 AndroidManifest.xml 文件！
            parser = assets.openXmlResourceParser(cookie, ANDROID_MANIFEST_FILENAME);

            final String[] outError = new String[1];
            
            //【3.1.1.3.1】解析非核心 apk！
            pkg = parseSplitApk(pkg, res, parser, flags, splitIndex, outError);

            if (pkg == null) {
                throw new PackageParserException(mParseError,
                        apkPath + " (at " + parser.getPositionDescription() + "): " + outError[0]);
            }

        } catch (PackageParserException e) {
            throw e;
        } catch (Exception e) {
            throw new PackageParserException(INSTALL_PARSE_FAILED_UNEXPECTED_EXCEPTION,
                    "Failed to read manifest from " + apkPath, e);
        } finally {
            IoUtils.closeQuietly(parser);
        }
    }
```
这个过程个解析 base apk 一样，我们不多关注！

##### 3.1.1.3.1 PParser.parseSplitApk\[5]
```java
    private Package parseSplitApk(Package pkg, Resources res, XmlResourceParser parser, int flags,
            int splitIndex, String[] outError) throws XmlPullParserException, IOException,
            PackageParserException {

        AttributeSet attrs = parser;

        //【3.1.1.1.1.2】解析 package 和 sliptName 属性！
        parsePackageSplitNames(parser, attrs);

        mParseInstrumentationArgs = null;
        mParseActivityArgs = null;
        mParseServiceArgs = null;
        mParseProviderArgs = null;

        int type;

        boolean foundApp = false;

        int outerDepth = parser.getDepth();
        while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
                && (type != XmlPullParser.END_TAG || parser.getDepth() > outerDepth)) {
            if (type == XmlPullParser.END_TAG || type == XmlPullParser.TEXT) {
                continue;
            }

            String tagName = parser.getName();
            if (tagName.equals("application")) { // 解析非核心 apk 的 "application" 标签！
                if (foundApp) {
                    if (RIGID_PARSER) {
                        outError[0] = "<manifest> has more than one <application>";
                        mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
                        return null;
                    } else {
                        Slog.w(TAG, "<manifest> has more than one <application>");
                        XmlUtils.skipCurrentTag(parser);
                        continue;
                    }
                }

                foundApp = true;
                
                //【3.1.1.3.2】调用 parseSplitApplication 方法继续解析！
                if (!parseSplitApplication(pkg, res, parser, flags, splitIndex, outError)) {
                    return null;
                }

            } else if (RIGID_PARSER) {
                outError[0] = "Bad element under <manifest>: "
                    + parser.getName();
                mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
                return null;

            } else {
                Slog.w(TAG, "Unknown element under <manifest>: " + parser.getName()
                        + " at " + mArchiveSourcePath + " "
                        + parser.getPositionDescription());
                XmlUtils.skipCurrentTag(parser);
                continue;
            }
        }

        if (!foundApp) {
            outError[0] = "<manifest> does not contain an <application>";
            mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_EMPTY;
        }
        
        //【3】返回最终的 Package 对象！
        return pkg;
    }

```

进入 parseSplitApplication 方法！

##### 3.1.1.3.2 PParser.parseSplitApplication
```java
    private boolean parseSplitApplication(Package owner, Resources res, XmlResourceParser parser,
            int flags, int splitIndex, String[] outError)
            throws XmlPullParserException, IOException {
            
        // 获得 "application" 标签的属性，保存到 TypedArray 对象 sa 中！
        TypedArray sa = res.obtainAttributes(parser,
                com.android.internal.R.styleable.AndroidManifestApplication);

        if (sa.getBoolean(
                com.android.internal.R.styleable.AndroidManifestApplication_hasCode, true)) {
            owner.splitFlags[splitIndex] |= ApplicationInfo.FLAG_HAS_CODE;
        }

        final int innerDepth = parser.getDepth();
        int type;
        while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
                && (type != XmlPullParser.END_TAG || parser.getDepth() > innerDepth)) {
            if (type == XmlPullParser.END_TAG || type == XmlPullParser.TEXT) {
                continue;
            }

            String tagName = parser.getName();
            if (tagName.equals("activity")) { // 解析非核心 apk 的 "activity"
            
                //【1】调用 parseActivity
                Activity a = parseActivity(owner, res, parser, flags, outError, false,
                        owner.baseHardwareAccelerated);
                if (a == null) {
                    mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
                    return false;
                }

                owner.activities.add(a);

            } else if (tagName.equals("receiver")) { // 解析非核心 apk 的 "receiver"
            
                //【2】调用 parseReceiver
                Activity a = parseActivity(owner, res, parser, flags, outError, true, false);
                if (a == null) {
                    mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
                    return false;
                }

                owner.receivers.add(a);

            } else if (tagName.equals("service")) {  // 解析非核心 apk 的 "service"
            
                //【3】调用 parseService
                Service s = parseService(owner, res, parser, flags, outError);
                if (s == null) {
                    mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
                    return false;
                }

                owner.services.add(s);

            } else if (tagName.equals("provider")) {  // 解析非核心 apk 的 "provider"
                //【4】调用 parseProvider
                Provider p = parseProvider(owner, res, parser, flags, outError);
                if (p == null) {
                    mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
                    return false;
                }

                owner.providers.add(p);

            } else if (tagName.equals("activity-alias")) {  // 解析非核心 apk 的 "activity-alias"
                //【5】调用 parseActivityAlias
                Activity a = parseActivityAlias(owner, res, parser, flags, outError);
                if (a == null) {
                    mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
                    return false;
                }

                owner.activities.add(a);

            } else if (parser.getName().equals("meta-data")) {  // 解析非核心 apk 的 "meta-data"
                //【6】调用 parseMetaData
                if ((owner.mAppMetaData = parseMetaData(res, parser, owner.mAppMetaData,
                        outError)) == null) {
                    mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
                    return false;
                }

            } else if (tagName.equals("uses-library")) {  // 解析非核心 apk 的 "uses-library"
                sa = res.obtainAttributes(parser,
                        com.android.internal.R.styleable.AndroidManifestUsesLibrary);

                // lib 库名称！
                String lname = sa.getNonResourceString(
                        com.android.internal.R.styleable.AndroidManifestUsesLibrary_name);
                // lib 库是否强制被使用！
                boolean req = sa.getBoolean(
                        com.android.internal.R.styleable.AndroidManifestUsesLibrary_required,
                        true);

                sa.recycle();
                
                // 将 lib 库信息添加到 usesLibraries 或者 usesOptionalLibraries 中！
                if (lname != null) {
                    lname = lname.intern();
                    if (req) {
                        // Upgrade to treat as stronger constraint
                        owner.usesLibraries = ArrayUtils.add(owner.usesLibraries, lname);
                        owner.usesOptionalLibraries = ArrayUtils.remove(
                                owner.usesOptionalLibraries, lname);
                    } else {
                        // Ignore if someone already defined as required
                        if (!ArrayUtils.contains(owner.usesLibraries, lname)) {
                            owner.usesOptionalLibraries = ArrayUtils.add(
                                    owner.usesOptionalLibraries, lname);
                        }
                    }
                }

                XmlUtils.skipCurrentTag(parser);

            } else if (tagName.equals("uses-package")) { // 解析非核心 apk 的 "uses-package"，不处理！
                // Dependencies for app installers; we don't currently try to
                // enforce this.
                XmlUtils.skipCurrentTag(parser);

            } else {
                if (!RIGID_PARSER) {
                    Slog.w(TAG, "Unknown element under <application>: " + tagName
                            + " at " + mArchiveSourcePath + " "
                            + parser.getPositionDescription());
                    XmlUtils.skipCurrentTag(parser);
                    continue;
                } else {
                    outError[0] = "Bad element under <application>: " + tagName;
                    mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
                    return false;
                }
            }
        }

        return true;
    }

```
到这里，应用程序包就已经被解析完了，最终，所有 apk 的数据都会封装到一个 Package 对象中返回！


### 3.1.2 PParser.parseMonolithicPackage

对于不支持 apk 拆分的 package，PMS 使用 parseMonolithicPackage 进行解析，典型的不支持拆分的 apk，是 /system/framework/framework-res.apk，下面我们来看看这个方法：

```java
    @Deprecated
    public Package parseMonolithicPackage(File apkFile, int flags) throws PackageParserException {
        
        //【3.1.2.1】同样的，先对 apk 进行一次整体解析！
        final PackageLite lite = parseMonolithicPackageLite(apkFile, flags);
        if (mOnlyCoreApps) {
            if (!lite.coreApp) {
                throw new PackageParserException(INSTALL_PARSE_FAILED_MANIFEST_MALFORMED,
                        "Not a coreApp: " + apkFile);
            }
        }

        final AssetManager assets = new AssetManager();
        try {
            
            //【3.1.1.2】再次调用 parseBaseApk 进行二次解析！
            final Package pkg = parseBaseApk(apkFile, assets, flags);
            pkg.setCodePath(apkFile.getAbsolutePath());
            pkg.setUse32bitAbi(lite.use32bitAbi);
            return pkg;
        } finally {
            IoUtils.closeQuietly(assets);
        }
    }
```
#### 3.1.2.1 PParser.parseMonolithicPackageLite
继续调用 parseMonolithicPackageLite 方法：
```java
    private static PackageLite parseMonolithicPackageLite(File packageFile, int flags)
            throws PackageParserException {
        
        // 同样的，调用 parseApkLite 直接对 apk 进行解析，返回 ApkLite 对象！
        final ApkLite baseApk = parseApkLite(packageFile, flags);
        final String packagePath = packageFile.getAbsolutePath();
        
        // 创建 PackageLite 对象！
        return new PackageLite(packagePath, baseApk, null, null, null);
    }

```
最后，同样的，还是返回一个 PackageParser.Package 对象！

这里就不多说了！

### 3.1.3 Package 
我们来看看 Pacakge 结构体，他用来分装一个应用程序包的完整信息：
```java
    public final static class Package {

        public String packageName; // 应用程序包名
        public String[] splitNames; // 非核心 apk 的名称

        public String volumeUuid;

        public String codePath; // 应用程序包的路径
        public String baseCodePath; // 核心 apk 的路径
        public String[] splitCodePaths; // 非核心 apk 的路径

        public int baseRevisionCode; // 核心 apk 
        public int[] splitRevisionCodes;

        public int[] splitFlags;

        public int[] splitPrivateFlags;

        public boolean baseHardwareAccelerated;

        // For now we only support one application per package.
        public final ApplicationInfo applicationInfo = new ApplicationInfo(); // 核心 apk 的 application 对象！

        public final ArrayList<Permission> permissions = new ArrayList<Permission>(0);
        public final ArrayList<PermissionGroup> permissionGroups = new ArrayList<PermissionGroup>(0);
        public final ArrayList<Activity> activities = new ArrayList<Activity>(0);
        public final ArrayList<Activity> receivers = new ArrayList<Activity>(0);
        public final ArrayList<Provider> providers = new ArrayList<Provider>(0);
        public final ArrayList<Service> services = new ArrayList<Service>(0);
        public final ArrayList<Instrumentation> instrumentation = new ArrayList<Instrumentation>(0);

        public final ArrayList<String> requestedPermissions = new ArrayList<String>();

        public ArrayList<String> protectedBroadcasts;

        public Package parentPackage;
        public ArrayList<Package> childPackages;

        public ArrayList<String> libraryNames = null;
        public ArrayList<String> usesLibraries = null;
        public ArrayList<String> usesOptionalLibraries = null;
        public String[] usesLibraryFiles = null;

        public ArrayList<ActivityIntentInfo> preferredActivityFilters = null;

        public ArrayList<String> mOriginalPackages = null;
        public String mRealPackage = null;
        public ArrayList<String> mAdoptPermissions = null;

        // We store the application meta-data independently to avoid multiple unwanted references
        public Bundle mAppMetaData = null;

        // The version code declared for this package.
        public int mVersionCode;

        // The version name declared for this package.
        public String mVersionName;

        // The shared user id that this package wants to use.
        public String mSharedUserId;

        // The shared user label that this package wants to use.
        public int mSharedUserLabel;

        // Signatures that were read from the package.
        public Signature[] mSignatures;
        public Certificate[][] mCertificates;

        // For use by package manager service for quick lookup of
        // preferred up order.
        public int mPreferredOrder = 0;

        // For use by package manager to keep track of when a package was last used.
        public long[] mLastPackageUsageTimeInMills =
                new long[PackageManager.NOTIFY_PACKAGE_USE_REASONS_COUNT];

        // // User set enabled state.
        // public int mSetEnabled = PackageManager.COMPONENT_ENABLED_STATE_DEFAULT;
        //
        // // Whether the package has been stopped.
        // public boolean mSetStopped = false;

        // Additional data supplied by callers.
        public Object mExtras;

        // Applications hardware preferences
        public ArrayList<ConfigurationInfo> configPreferences = null;

        // Applications requested features
        public ArrayList<FeatureInfo> reqFeatures = null;

        // Applications requested feature groups
        public ArrayList<FeatureGroupInfo> featureGroups = null;

        public int installLocation;

        public boolean coreApp;

        /* An app that's required for all users and cannot be uninstalled for a user */
        public boolean mRequiredForAllUsers;

        /* The restricted account authenticator type that is used by this application */
        public String mRestrictedAccountType;

        /* The required account type without which this application will not function */
        public String mRequiredAccountType;

        public String mOverlayTarget;
        public int mOverlayPriority;
        public boolean mTrustedOverlay;

        public ArraySet<PublicKey> mSigningKeys;
        public ArraySet<String> mUpgradeKeySets;
        public ArrayMap<String, ArraySet<PublicKey>> mKeySetMapping;

        public String cpuAbiOverride;

        public boolean use32bitAbi;

        public byte[] restrictUpdateHash;

        public Package(String packageName) {
            this.packageName = packageName;
            applicationInfo.packageName = packageName;
            applicationInfo.uid = -1;
        }

        ... ... ... ...

    }
```
通过 PParser.parsePackage 方法，最终会返回一个 Package 对象，封装了应用程序包的所有信息！！

## 3.2 PMS.scanPackageLI

参数 policyFlags 为 paraseFlags！

前一个阶段，通过解析获得了 package 的信息，接着继续扫描：
```java
    private PackageParser.Package scanPackageLI(PackageParser.Package pkg, File scanFile,
            final int policyFlags, int scanFlags, long currentTime, UserHandle user)
            throws PackageManagerException {

        //【1】SCAN_CHECK_ONLY 标签是为了检测是否所有的包（parent 和 child）都可以被成功的扫描到！
        if ((scanFlags & SCAN_CHECK_ONLY) == 0) { 
            if (pkg.childPackages != null && pkg.childPackages.size() > 0) {
                scanFlags |= SCAN_CHECK_ONLY;  // 设置 SCAN_CHECK_ONLY 位！
            }

        } else {
            scanFlags &= ~SCAN_CHECK_ONLY; // 取消 SCAN_CHECK_ONLY 位！
        }

        //【3.2.1】继续扫描当前 package，返回扫描结果 scannedPkg！
        PackageParser.Package scannedPkg = scanPackageInternalLI(pkg, scanFile, policyFlags,
                scanFlags, currentTime, user);

        // 扫描当前 package 的子 pacakge（如果有）
        final int childCount = (pkg.childPackages != null) ? pkg.childPackages.size() : 0;
        for (int i = 0; i < childCount; i++) {
            PackageParser.Package childPackage = pkg.childPackages.get(i);
            // 解析子包！
            scanPackageInternalLI(childPackage, scanFile, policyFlags, scanFlags,
                    currentTime, user);
        }


        if ((scanFlags & SCAN_CHECK_ONLY) != 0) {
            // 如果设置了 SCAN_CHECK_ONLY 位， 就调用自身，再次处理！
            return scanPackageLI(pkg, scanFile, policyFlags, scanFlags, currentTime, user);
        }

        return scannedPkg;
    }

```
如果被扫描的 package 有 child package，并且是第一次进入该方法的话，就需要检测是否所有的包（parent 和 child）都可以被成功的扫描到，scanFlags 本来是没有 SCAN_CHECK_ONLY 位的，所以这里会将其 SCAN_CHECK_ONLY 置为 1，这样在方法的最后，又会调用自身，这次又会将 SCAN_CHECK_ONLY 位置为 0！



继续看：

### 3.2.1 PMS.scanPackageInternalLI

我们继续来看，通过前面的扫描解析，我们获得了应用程序的 PackageParser.Package 对象，同时，我们也已经获得了上一次的安装信息，接下来，就是要处理解析获得的数据：
```java
    private PackageParser.Package scanPackageInternalLI(PackageParser.Package pkg, File scanFile,
            int policyFlags, int scanFlags, long currentTime, UserHandle user)
            throws PackageManagerException {

        PackageSetting ps = null;
        PackageSetting updatedPkg;

        synchronized (mPackages) {
            // package 是否被重命名过，有的话，获得其 oldName！
            String oldName = mSettings.mRenamedPackages.get(pkg.packageName);
            
            //【3.2.1.1】如果 package 有源包，并且源包的名字是当前扫描的 package 的旧名字，
            // 那就用源包的 PackageSetting 作为当前 package 的数据！
            if (pkg.mOriginalPackages != null && pkg.mOriginalPackages.contains(oldName)) {
                ps = mSettings.peekPackageLPr(oldName);
            }

            // 如果没有源包，就用当前 package 的包名查找一个已的 PackageSetting！
            if (ps == null) {
                ps = mSettings.peekPackageLPr(pkg.packageName);
            }
        
            //【3.2.1.2】如果当前系统 package 被更新过，就查找到被更新之前的 PackageSetting！
            updatedPkg = mSettings.getDisabledSystemPkgLPr(ps != null ? ps.name : pkg.packageName);
            if (DEBUG_INSTALL && updatedPkg != null) Slog.d(TAG, "updatedPkg = " + updatedPkg);
            
            // 如果系统 package，并且被更新过，需要处理新旧 child package 的差异！
            // 比较更新后的 child package 和更新前的 child package，如果更新前的 child package 不存在了；
            // 就要移除！
            if ((policyFlags & PackageParser.PARSE_IS_SYSTEM) != 0) { 
                PackageSetting disabledPs = mSettings.getDisabledSystemPkgLPr(pkg.packageName);
                if (disabledPs != null) {
                    // 当前 package 的 child package 个数！
                    final int scannedChildCount = (pkg.childPackages != null)
                            ? pkg.childPackages.size() : 0;
                    // 更新之前的 child package 个数！
                    final int disabledChildCount = disabledPs.childPackageNames != null
                            ? disabledPs.childPackageNames.size() : 0;

                    for (int i = 0; i < disabledChildCount; i++) {
                        String disabledChildPackageName = disabledPs.childPackageNames.get(i);
                        boolean disabledPackageAvailable = false;
                        
                        for (int j = 0; j < scannedChildCount; j++) {
                            PackageParser.Package childPkg = pkg.childPackages.get(j);
                            if (childPkg.packageName.equals(disabledChildPackageName)) {
                            
                                disabledPackageAvailable = true;
                                break;
                            }
                         }
        
                         if (!disabledPackageAvailable) {
                             //【3.2.1.3】更新前的 child package，不包含在更新后的 child package 中，无效，
                             // 就从 mDisabledSysPackages 中删除掉！
                             mSettings.removeDisabledSystemPackageLPw(disabledChildPackageName);
                         }
                    }
                }
            }
        }

        boolean updatedPkgBetter = false; 

        //【A】处理覆盖更新的情况！
        // 如果当前解析的系统 apk，并且他之前被覆盖更新过！
        // 注意：ps 是上次安装的信息，updatedPkg 是由于覆盖安装更新前的信息！，pkg 则是本次扫描的 system app 的信息！
        if (updatedPkg != null && (policyFlags & PackageParser.PARSE_IS_SYSTEM) != 0) {
            if (locationIsPrivileged(scanFile)) { 
                // 对于 /system/priv-app 目录下的 app，需要增加 ApplicationInfo.PRIVATE_FLAG_PRIVILEGED 的 flag！
                updatedPkg.pkgPrivateFlags |= ApplicationInfo.PRIVATE_FLAG_PRIVILEGED;
            } else {
                updatedPkg.pkgPrivateFlags &= ~ApplicationInfo.PRIVATE_FLAG_PRIVILEGED;
            }
            
            // 上次更新后的应用路径和这次扫描解析的路径不一样，上次更新到了 data 分区，而这次扫描的是 system 分区，
            // 就需要比较一下 versionCode 的大小，
            if (ps != null && !ps.codePath.equals(scanFile)) {
                if (DEBUG_INSTALL) Slog.d(TAG, "Path changing from " + ps.codePath);
                // pkg 的 versioncode 小于等于上次更新后的 versioncode，说明 data 分区的 apk 仍然是最新的
                // 那就用本次扫描解析的数据，来更新上次更新前的旧数据！
                if (pkg.mVersionCode <= ps.versionCode) {

                    if (DEBUG_INSTALL) Slog.i(TAG, "Package " + ps.name + " at " + scanFile
                            + " ignored: updated version " + ps.versionCode
                            + " better than this " + pkg.mVersionCode);

                    // updatedPkg.codePath 不等于当前的扫描目录，说明该 system app 目录发生了变化！
                    // 而该 system apk 是被覆盖更新了，所以也需要更新 updatedPkg 的数据～
                    if (!updatedPkg.codePath.equals(scanFile)) {
                        Slog.w(PackageManagerService.TAG, "Code path for hidden system pkg "
                                + ps.name + " changing from " + updatedPkg.codePathString
                                + " to " + scanFile);

                        updatedPkg.codePath = scanFile;
                        updatedPkg.codePathString = scanFile.toString();
                        updatedPkg.resourcePath = scanFile;
                        updatedPkg.resourcePathString = scanFile.toString();
                    }
                    
                    updatedPkg.pkg = pkg;
                    updatedPkg.versionCode = pkg.mVersionCode;

                    final int childCount = updatedPkg.childPackageNames != null
                            ? updatedPkg.childPackageNames.size() : 0;

                    for (int i = 0; i < childCount; i++) {
                        String childPackageName = updatedPkg.childPackageNames.get(i);
                        PackageSetting updatedChildPkg = mSettings.getDisabledSystemPkgLPr(
                                childPackageName);

                        if (updatedChildPkg != null) {
                            updatedChildPkg.pkg = pkg;
                            updatedChildPkg.versionCode = pkg.mVersionCode;
                        }
                    }

                    throw new PackageManagerException(Log.WARN, "Package " + ps.name + " at "
                            + scanFile + " ignored: updated version " + ps.versionCode
                            + " better than this " + pkg.mVersionCode);
                } else {
                    // pkg 的 versioncode 大于上次更新后的 versioncode，
                    // 说明当前在 system 分区的 apk 要比 data 分区的新，这说明 system app 又通过 OTA 升级更新了
                    // 那就要保留本次扫描解析的数据，删除上一次更新的数据！
                    synchronized (mPackages) {
                        // 从 PMS.mPackages 中删除上一次的扫描数据！
                        mPackages.remove(ps.name);
                    }

                    logCriticalInfo(Log.WARN, "Package " + ps.name + " at " + scanFile
                            + " reverting from " + ps.codePathString
                            + ": new version " + pkg.mVersionCode
                            + " better than installed " + ps.versionCode);
                    
                    // 创建一个安装参数对象，封装了上次安装的信息！！
                    InstallArgs args = createInstallArgsForExisting(packageFlagsToInstallFlags(ps),
                            ps.codePathString, ps.resourcePathString, getAppDexInstructionSets(ps));

                    synchronized (mInstallLock) {
                        // 移除 data 分区的 apk 和 相应的 dex 文件！
                        args.cleanUpResourcesLI();
                    }
                    
                    synchronized (mPackages) {
                        //【3.2.1.4】将更新前的旧数据 PackageSetting ，从 mDisabledSysPackages 中移除，
                        // 并复用旧数据，创建一个新的PackageSetting，添加到 Setting 的 mPackage 中！
                        mSettings.enableSystemPackageLPw(ps.name);
                    }
                    
                    updatedPkgBetter = true; // 设置 updatedPkgBetter 为 true！
                }
            }
        }

        if (updatedPkg != null) { // 对于被更新的系统 app 的 flag 进行设置！
            policyFlags |= PackageParser.PARSE_IS_SYSTEM;
            if ((updatedPkg.pkgPrivateFlags & ApplicationInfo.PRIVATE_FLAG_PRIVILEGED) != 0) {
                policyFlags |= PackageParser.PARSE_IS_PRIVILEGED;
            }
        }

        collectCertificatesLI(ps, pkg, scanFile, policyFlags);

        //【B】处理没有覆盖，但是 data 和 system 分区出现了相同的 apk 的情况！
        // package 并没有发生覆盖更新，但之前 apk 是安装在 data 分区，而后来出现了相同包名的新 apk 安装在了 system 分区
        //（比如系统 OTA 升级导致的 apk 分区改变，或者 root 后 push 一个相同 apk 到 system 分区然后重启）！
        // 或者还有一种是之前安装在 data 分区，然后应用移动到了 system 分区！
        // 那就要判断是否隐藏 system 分区的这个 apk 了！
        boolean shouldHideSystemApp = false; 
        if (updatedPkg == null && ps != null
                && (policyFlags & PackageParser.PARSE_IS_SYSTEM_DIR) != 0 && !isSystemApp(ps)) {
            // 校验签名，如果不匹配的话，就冻结掉安装在 system 分区的 apk，并清除数据！
            if (compareSignatures(ps.signatures.mSignatures, pkg.mSignatures)
                    != PackageManager.SIGNATURE_MATCH) {
                    
                logCriticalInfo(Log.WARN, "Package " + ps.name + " appeared on system, but"
                        + " signatures don't match existing userdata copy; removing");
                
                // 创建一个 PackageFreezer 用来冻结指定的 package！
                try (PackageFreezer freezer = freezePackage(pkg.packageName,
                        "scanPackageInternalLI")) {
                    deletePackageLIF(pkg.packageName, null, true, null, 0, null, false, null);
                }
                ps = null;

            } else {
                // 如果新添加的位于 system 分区的 apk 的版本号小于等于位于 data 分区的 apk；
                // 那就要隐藏 system 分区的 apk；
                if (pkg.mVersionCode <= ps.versionCode) {
                    shouldHideSystemApp = true; // 设置 shouldHideSystemApp 为 true！
                    logCriticalInfo(Log.INFO, "Package " + ps.name + " appeared at " + scanFile
                            + " but new version " + pkg.mVersionCode + " better than installed "
                            + ps.versionCode + "; hiding system");
                } else {
                    // system 分区的应用版本更高，删除 data 分区的 apk，同时保留数据！
                    logCriticalInfo(Log.WARN, "Package " + ps.name + " at " + scanFile
                            + " reverting from " + ps.codePathString + ": new version "
                            + pkg.mVersionCode + " better than installed " + ps.versionCode);
                            
                    // 创建一个安装参数！        
                    InstallArgs args = createInstallArgsForExisting(packageFlagsToInstallFlags(ps),
                            ps.codePathString, ps.resourcePathString, getAppDexInstructionSets(ps));
                            
                    synchronized (mInstallLock) {
                        // 删除 data 分区的 apk，并保留用户数据！
                        args.cleanUpResourcesLI();
                    }
                }
            }
        }

        // 系统 apk 不进入！
        if ((policyFlags & PackageParser.PARSE_IS_SYSTEM_DIR) == 0) {
            if (ps != null && !ps.codePath.equals(ps.resourcePath)) {
                policyFlags |= PackageParser.PARSE_FORWARD_LOCK;
            }
        }

        String resourcePath = null;
        String baseResourcePath = null;
        if ((policyFlags & PackageParser.PARSE_FORWARD_LOCK) != 0 && !updatedPkgBetter) {
            if (ps != null && ps.resourcePathString != null) {
                resourcePath = ps.resourcePathString;
                baseResourcePath = ps.resourcePathString;

            } else {
                Slog.e(TAG, "Resource path not set for package " + pkg.packageName);

            }
        } else {
            resourcePath = pkg.codePath;
            baseResourcePath = pkg.baseCodePath;
        }

        // 设置 Package.ApplicationInfo 对象的 path 属性！
        pkg.setApplicationVolumeUuid(pkg.volumeUuid);
        pkg.setApplicationInfoCodePath(pkg.codePath);
        pkg.setApplicationInfoBaseCodePath(pkg.baseCodePath);
        pkg.setApplicationInfoSplitCodePaths(pkg.splitCodePaths);
        pkg.setApplicationInfoResourcePath(resourcePath);
        pkg.setApplicationInfoBaseResourcePath(baseResourcePath);
        pkg.setApplicationInfoSplitResourcePaths(pkg.splitCodePaths);

        //【3.2.1.7】接着调用 scanPackageLI，继续处理扫描数据！
        PackageParser.Package scannedPkg = scanPackageLI(pkg, policyFlags, scanFlags
                | SCAN_UPDATE_SIGNATURE, currentTime, user);

        //【3.2.1.8】如果确认 data 分区的 apk 版本比 system 分区的版本高
        // 就隐藏 system 分区的 app，让 data 分区的 apk 显示出来！
        if (shouldHideSystemApp) {
            synchronized (mPackages) {
                mSettings.disableSystemPackageLPw(pkg.packageName, true);
            }
        }

        return scannedPkg;
    }

```
到这里，我们先来总结一下，对于一个系统 app 而言， app 升级有如下两种途径：

- 覆盖安装：这种方式 PMS 会动态修改 packages.xml 文件！
- OTA 升级方式：这种方式并不会动态修改 package.xml 文件！

下面我们来分开分析一下：

- 覆盖安装
   - 对于系统 app，覆盖安装的话，新的 app 会安装到 data 分区，packages.xml 中相应数据会发生如下变化：
```xml
 <package name="com.android.pic" codePath="/data/app/com.android.pic-1" nativeLibraryPath="/data/app/com.android.pic-1/lib"
          primaryCpuAbi="arm64-v8a" publicFlags="944258757" privateFlags="0" ft="15c38695c40" it="15baf6278f0" 
          ut="15c38695e98" version="3002" sharedUserId="1000" installer="com.android.packageinstaller" isOrphaned="true">
 </package>
        
<updated-package name="com.android.pic" codePath="/system/app/pic" 
          ft="15baf6278f0" it="15baf6278f0" ut="15baf6278f0" version="3002" 
          nativeLibraryPath="/system/app/pic/lib" primaryCpuAbi="arm64-v8a" sharedUserId="1000" />
```
可以看到，package 的 codePath、nativeLibraryPath 都发生了变化，根据上面的解析过程，data 分区的覆盖安装的 apk 将被保存到 mPackages 中，system 分区原来的旧 apk 将被保存到 mDisablePackage 中！

- OTA 升级方式
  - 这种方式是通过进入 Recovery 升级（或者是 A/B 升级方式），通过 patch 和文件覆盖方式来升级，这种方式不会即时更改 Packages.xml 文件，真正的修改是在重启后的 PMS 中去做的！



#### 3.2.1.1 Settings.peekPackageLPr
```java
    PackageSetting peekPackageLPr(String name) {
        return mPackages.get(name);
    }
```
该方法用于获得 packageName 对应的 PackageSetting 对象！

#### 3.2.1.2 Settings.getDisabledSystemPkgLPr
```java
    public PackageSetting getDisabledSystemPkgLPr(String name) {
        PackageSetting ps = mDisabledSysPackages.get(name);
        return ps;
    }

```
我们知道 mDisabledSysPackages 中的数据来自 updated-package 标签！

#### 3.2.1.3 Settings.removeDisabledSystemPackageLPw
```java
    void removeDisabledSystemPackageLPw(String name) {
        mDisabledSysPackages.remove(name);
    }
```

#### 3.2.1.4 Settings.enableSystemPackageLPw

这里用于恢复一个 system pacakge！

```java
    PackageSetting enableSystemPackageLPw(String name) {
        // 从 mDisabledSysPackages 中获得被更新前的 PackageSettings！
        PackageSetting p = mDisabledSysPackages.get(name);
        if(p == null) {
            Log.w(PackageManagerService.TAG, "Package " + name + " is not disabled");
            return null;
        }
        // 去掉 ApplicationInfo.FLAG_UPDATED_SYSTEM_APP 标志位！
        if((p.pkg != null) && (p.pkg.applicationInfo != null)) {
            p.pkg.applicationInfo.flags &= ~ApplicationInfo.FLAG_UPDATED_SYSTEM_APP;
        }
        // 复用数据，创建一个新的 PackageSetting，并根据 uid 添加到指定集合中！ 
        PackageSetting ret = addPackageLPw(name, p.realName, p.codePath, p.resourcePath,
                p.legacyNativeLibraryPathString, p.primaryCpuAbiString,
                p.secondaryCpuAbiString, p.cpuAbiOverrideString,
                p.appId, p.versionCode, p.pkgFlags, p.pkgPrivateFlags,
                p.parentPackageName, p.childPackageNames);

        // 从 mDisabledSysPackages 移除这个 package
        mDisabledSysPackages.remove(name);
        return ret;
    }
```
方法的流程很简单，不多说了！

#### 3.2.1.6 PMS.deletePackageLIF 

对于之前 apk 是安装在 data 分区，而后来出现了相同包名的新 apk 安装在了 system 分区这种情况，需要校验签名，如果不匹配的话，就要删掉 data 分区的 apk！

首先创建了一个 PackageFreezer 对象，用来冻结这个应用！
```java
        public PackageFreezer(String packageName, int userId, String killReason) {
            synchronized (mPackages) {
                // 添加到 mFrozenPackages 集合中！
                mPackageName = packageName;
                mWeFroze = mFrozenPackages.add(mPackageName);

                final PackageSetting ps = mSettings.mPackages.get(mPackageName);
                if (ps != null) {
                    // 杀掉应用的进程
                    killApplication(ps.name, ps.appId, userId, killReason);
                }

                // 对子包进行相同的操作！
                final PackageParser.Package p = mPackages.get(packageName);
                if (p != null && p.childPackages != null) {

                    final int N = p.childPackages.size();
                    mChildren = new PackageFreezer[N];
                    for (int i = 0; i < N; i++) {
                        mChildren[i] = new PackageFreezer(p.childPackages.get(i).packageName,
                                userId, killReason);
                    }
                } else {
                    mChildren = null;
                }
            }
            mCloseGuard.open("close");
        }
```
接着，调用 deletePackageLIF 方法来处理这个 package，传入参数：

- String packageName：传入 pkg.packageName；
- UserHandle user：传入 null；
- boolean deleteCodeAndResources：传入 true；
- int[] allUserHandles：传入 null；
- int flags：传入 0；
- PackageRemovedInfo outInfo：传入 null；
- boolean writeSettings：传入 false；
- PackageParser.Package replacingPackage：传入 null;

```java
    private boolean deletePackageLIF(String packageName, UserHandle user,
            boolean deleteCodeAndResources, int[] allUserHandles, int flags,
            PackageRemovedInfo outInfo, boolean writeSettings,
            PackageParser.Package replacingPackage) {

        if (packageName == null) {
            Slog.w(TAG, "Attempt to delete null packageName.");
            return false;
        }

        if (DEBUG_REMOVE) Slog.d(TAG, "deletePackageLI: " + packageName + " user " + user);

        PackageSetting ps;

        synchronized (mPackages) {
            
            // 获得之前安装的信息，如果为 null，谁明这个
            ps = mSettings.mPackages.get(packageName);
            if (ps == null) {
                Slog.w(TAG, "Package named '" + packageName + "' doesn't exist.");
                return false;
            }
            
            // 如果当前的 package 是 child package，且不是系统 apk 的话，进入这个分支！
            if (ps.parentPackageName != null && (!isSystemApp(ps)
                    || (flags & PackageManager.DELETE_SYSTEM_APP) != 0)) {

                if (DEBUG_REMOVE) {
                    Slog.d(TAG, "Uninstalled child package:" + packageName + " for user:"
                            + ((user == null) ? UserHandle.USER_ALL : user));
                }
                
                // 根据参数传递，removedUserId 的值为 UserHandle.USER_ALL；
                final int removedUserId = (user != null) ? user.getIdentifier()
                        : UserHandle.USER_ALL;

                if (!clearPackageStateForUserLIF(ps, removedUserId, outInfo)) {
                    return false;
                }
                
                markPackageUninstalledForUserLPw(ps, user);
                scheduleWritePackageRestrictionsLocked(user);
                
                return true;
            }
        }

        // 当 user 不为 null 才会进入这里，表示为指定的设备用户删除该 apk！
        if (((!isSystemApp(ps) || (flags&PackageManager.DELETE_SYSTEM_APP) != 0) && user != null
                && user.getIdentifier() != UserHandle.USER_ALL)) {
            ... ... ... ...
        }

        // 这里也不进入！
        if (ps.childPackageNames != null && outInfo != null) {
            ... ... ... ...
        }

        boolean ret = false;

        if (isSystemApp(ps)) {
            if (DEBUG_REMOVE) Slog.d(TAG, "Removing system package: " + ps.name);
            // When an updated system application is deleted we delete the existing resources
            // as well and fall back to existing code in system partition
            ret = deleteSystemPackageLIF(ps.pkg, ps, allUserHandles, flags, outInfo, writeSettings);
        } else {
            if (DEBUG_REMOVE) Slog.d(TAG, "Removing non-system package: " + ps.name);
            
            // 进入这个分支，因为 data 分区的 apk 已经存在，所以进入这个分支，删除掉非系统 apk！
            ret = deleteInstalledPackageLIF(ps, deleteCodeAndResources, flags, allUserHandles,
                    outInfo, writeSettings, replacingPackage);
        }

        // 根据参数，不会进入这个分支
        if (outInfo != null) {
            ... ... ... 
        }

        return ret;
    }

```
关于 apk 删除的逻辑，我们在另开一篇讲解，这个的逻辑是会删掉之前已经安装的 data 分区的 apk 的！


继续看：

#### 3.2.1.7 PMS.scanPackageLI

接下来，进一步的扫描：
```java
    private PackageParser.Package scanPackageLI(PackageParser.Package pkg, final int policyFlags,
            int scanFlags, long currentTime, UserHandle user) throws PackageManagerException {
        boolean success = false;
        try {
        
            //【3.2.1.7.1】继续调用 scanPackageDirtyLI 处理扫描数据！
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
下面，我们继续看 scanPackageDirtyLI 方法中的逻辑！



##### 3.2.1.7.1 *PMS.scanPackageDirtyLI

最终调用 scanPackageDirtyLI 方法处理扫描得到的数据：

因为我们本次扫描的是 system app，所以 PackageParser.Package pkg 分装了扫描的结果信息！
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

        //【1】处理系统 apk 标志位；
        if ((policyFlags & PackageParser.PARSE_IS_SYSTEM) != 0) { 
            // 对于系统 app，增加 ApplicationInfo.FLAG_SYSTEM 的 flags，表示是系统 apk！
            //【1.1】注意系统和非系统的区别是，apk 所在的目录！
            pkg.applicationInfo.flags |= ApplicationInfo.FLAG_SYSTEM;
            // 处理 Direct Boot Mode 属性，设备启动后进入的一个新模式，直到用户解锁（unlock）设备此阶段结束！
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
            pkg.coreApp = false;
            pkg.applicationInfo.privateFlags &=
                    ~ApplicationInfo.PRIVATE_FLAG_DEFAULT_TO_DEVICE_PROTECTED_STORAGE;
            pkg.applicationInfo.privateFlags &=
                    ~ApplicationInfo.PRIVATE_FLAG_DIRECT_BOOT_AWARE;
        }

        pkg.mTrustedOverlay = (policyFlags&PackageParser.PARSE_TRUSTED_OVERLAY) != 0;

        //【2】设置 ApplicationInfo.PRIVATE_FLAG_PRIVILEGED 标记位，说明这个 apk 是特权 apk！
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

        //【3】处理包名为 "android" 的 apk，也就是 framework-res.apk，属于系统平台包！
        if (pkg.packageName.equals("android")) {
            synchronized (mPackages) {
                if (mAndroidApplication != null) {
                    Slog.w(TAG, "*************************************************");
                    Slog.w(TAG, "Core android package being redefined.  Skipping.");
                    Slog.w(TAG, " file=" + scanFile);
                    Slog.w(TAG, "*************************************************");
                    throw new PackageManagerException(INSTALL_FAILED_DUPLICATE_PACKAGE,
                            "Core android package being redefined.  Skipping.");
                }

                if ((scanFlags & SCAN_CHECK_ONLY) == 0) { // 进入该分支，设置系统平台包相关的属性！
                    // Set up information for our fall-back user intent resolution activity.
                    mPlatformPackage = pkg;
                    pkg.mVersionCode = mSdkVersion;
                    mAndroidApplication = pkg.applicationInfo;

                    if (!mResolverReplaced) { // 建立 ResolverActivity 的内存对象。
                        mResolveActivity.applicationInfo = mAndroidApplication;
                        mResolveActivity.name = ResolverActivity.class.getName();
                        mResolveActivity.packageName = mAndroidApplication.packageName;
                        mResolveActivity.processName = "system:ui";
                        mResolveActivity.launchMode = ActivityInfo.LAUNCH_MULTIPLE;
                        mResolveActivity.documentLaunchMode = ActivityInfo.DOCUMENT_LAUNCH_NEVER;
                        mResolveActivity.flags = ActivityInfo.FLAG_EXCLUDE_FROM_RECENTS;
                        mResolveActivity.theme = R.style.Theme_Material_Dialog_Alert;
                        mResolveActivity.exported = true;
                        mResolveActivity.enabled = true;
                        mResolveActivity.resizeMode = ActivityInfo.RESIZE_MODE_RESIZEABLE;
                        mResolveActivity.configChanges = ActivityInfo.CONFIG_SCREEN_SIZE
                                | ActivityInfo.CONFIG_SMALLEST_SCREEN_SIZE
                                | ActivityInfo.CONFIG_SCREEN_LAYOUT
                                | ActivityInfo.CONFIG_ORIENTATION
                                | ActivityInfo.CONFIG_KEYBOARD
                                | ActivityInfo.CONFIG_KEYBOARD_HIDDEN;
                        mResolveInfo.activityInfo = mResolveActivity;
                        mResolveInfo.priority = 0;
                        mResolveInfo.preferredOrder = 0;
                        mResolveInfo.match = 0;
                        mResolveComponentName = new ComponentName(
                                mAndroidApplication.packageName, mResolveActivity.name);
                    }
                }
            }
        }

        if (DEBUG_PACKAGE_SCANNING) {
            if ((policyFlags & PackageParser.PARSE_CHATTY) != 0)
                Log.d(TAG, "Scanning package " + pkg.packageName);
        }

        synchronized (mPackages) {
            //【4】如果 PMS.mPackages 已经包含当前的 Package，说明这个包已经被扫描过了，抛异常，不继续处理！
            if (mPackages.containsKey(pkg.packageName)
                    || mSharedLibraries.containsKey(pkg.packageName)) {
                throw new PackageManagerException(INSTALL_FAILED_DUPLICATE_PACKAGE,
                        "Application package " + pkg.packageName
                                + " already installed.  Skipping duplicate.");
            }


            //【5】系统 apk 不进入这个分支，SCAN_REQUIRE_KNOWN 只有在扫描 data 分区是才会被设置！！
            if ((scanFlags & SCAN_REQUIRE_KNOWN) != 0) {
                //【5.1】扫描 data 分区时，会判断 mExpectingBetter 中是否包含该 package！
                if (mExpectingBetter.containsKey(pkg.packageName)) {
                    logCriticalInfo(Log.WARN,
                            "Relax SCAN_REQUIRE_KNOWN requirement for package " + pkg.packageName);
                } else {
                    PackageSetting known = mSettings.peekPackageLPr(pkg.packageName);
                    if (known != null) {
                        if (DEBUG_PACKAGE_SCANNING) {
                            Log.d(TAG, "Examining " + pkg.codePath
                                    + " and requiring known paths " + known.codePathString
                                    + " & " + known.resourcePathString);
                        }
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

        //【6】获得扫描到的 apk 和资源的路径，下面会用到！
        File destCodeFile = new File(pkg.applicationInfo.getCodePath());
        File destResourceFile = new File(pkg.applicationInfo.getResourcePath());

        SharedUserSetting suid = null;
        PackageSetting pkgSetting = null;

        //【7】系统 apk 才能有 mOriginalPackages，mRealPackage 和 mAdoptPermissions！
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
            //【8】当前的系统（三方） package 是共享 uid 的，要判断其对应的共享 uid 是否存在，
            // 不存在就抛出异常！
            if (pkg.mSharedUserId != null) {
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


            //【9】对于系统 package，如果有源包，那就要尝试将 package 的名字改为源包的名字！
            // 需要找到一个合适的源包，用来改名！
            PackageSetting origPackage = null;
            String realName = null
            if (pkg.mOriginalPackages != null) {
                final String renamed = mSettings.mRenamedPackages.get(pkg.mRealPackage);
                //【9.1】如果当前 package 被重命名过，并且有源包的名字是重命名前的名字，就将 package 的名字改为以前的！
                if (pkg.mOriginalPackages.contains(renamed)) {
                    realName = pkg.mRealPackage;
                    if (!pkg.packageName.equals(renamed)) {
                        pkg.setPackageName(renamed);
                    }

                } else {
                    for (int i=pkg.mOriginalPackages.size()-1; i>=0; i--) {
                        if ((origPackage = mSettings.peekPackageLPr(
                                pkg.mOriginalPackages.get(i))) != null) {
                            //【9.2】对当前 package 和其源包进行校验
                            // 如果源包是非系统 package，不是同一分区，无法重命名为源包名，返回 false；
                            // 或者源包是系统 package，但是 PMS.mPackage 中仍然有其扫描数据，源包仍然存在，返回 false；
                            if (!verifyPackageUpdateLPr(origPackage, pkg)) {
                                origPackage = null;
                                continue;
                                
                            } else if (origPackage.sharedUser != null) {
                                //【9.3】源包是系统 package，并且共享用户 uid；
                                // 如果当前的 package 和源包的共享 uid 不匹配，也会返回 false！
                                // 表示无法迁移数据；
                                if (!origPackage.sharedUser.name.equals(pkg.mSharedUserId)) {
                                    Slog.w(TAG, "Unable to migrate data from " + origPackage.name
                                            + " to " + pkg.packageName + ": old uid "
                                            + origPackage.sharedUser.name
                                            + " differs from " + pkg.mSharedUserId);
                                    origPackage = null;
                                    continue;
                                }
                            } else {
                                // 可以看到，如果找到用于重命名的源包，origPackage 不会为 null！
                                if (DEBUG_UPGRADE) Log.v(TAG, "Renaming new package "
                                        + pkg.packageName + " to old name " + origPackage.name);
                            }
                            break;
                        }
                    }
                }
            }
       
            // mTransferedPackages 用于保存那些自身数据已经被转移到其他 package 的 package！
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
            
            //【3.2.1.7.1.1】获得当前扫描的这个 package 对应的 packageSetting 对象，如果已经存在就直接返回，
            // 不存在就创建！如果 origPackage 不为 null，创建新的需要重命名为源包的名字！
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

            //【11】对于系统 package，如果 origPackage 不为 null，改名为源包的名字！
            if (pkgSetting.origPackage != null) {
                pkg.setPackageName(origPackage.name);

                String msg = "New package " + pkgSetting.realName
                        + " renamed to replace old package " + pkgSetting.name;
                reportSettingsProblem(Log.WARN, msg);

                if ((scanFlags & SCAN_CHECK_ONLY) == 0) { 
                    // 如果没有设置 SCAN_CHECK_ONLY，并且源包是存在的，就将源包添加到 mTransferedPackages 中！
                    mTransferedPackages.add(origPackage.name);
                }

                // 清空 originPackage 属性！
                pkgSetting.origPackage = null;
            }

            if ((scanFlags & SCAN_CHECK_ONLY) == 0 && realName != null) {
                mTransferedPackages.add(pkg.packageName);
            }

            //【12】如果当前的系统 apk 被覆盖更新过，就添加 ApplicationInfo.FLAG_UPDATED_SYSTEM_APP 标签！
            if (mSettings.isDisabledSystemPackageLPr(pkg.packageName)) { 
                pkg.applicationInfo.flags |= ApplicationInfo.FLAG_UPDATED_SYSTEM_APP;
            }

            if ((policyFlags & PackageParser.PARSE_IS_SYSTEM_DIR) == 0) {
                updateSharedLibrariesLPw(pkg, null);
            }

            if (mFoundPolicyFile) {
                //【13】给当前的 Package 分配一个标签，用于 seLinux！
                SELinuxMMAC.assignSeinfoValue(pkg);
            }

            pkg.applicationInfo.uid = pkgSetting.appId;
            pkg.mExtras = pkgSetting;

            //【13】处理 keySet 更新和签名校验！
            if (shouldCheckUpgradeKeySetLP(pkgSetting, scanFlags)) {
                if (checkUpgradeKeySetLP(pkgSetting, pkg)) {
                    // 签名正确，更新本地的签名信息！
                    pkgSetting.signatures.mSignatures = pkg.mSignatures;
                } else {
                    // 签名异常处理
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
                // 如果不检查 KetSet 更新的话，就直接校验签名！
                try {
                    verifySignaturesLP(pkgSetting, pkg);
                    pkgSetting.signatures.mSignatures = pkg.mSignatures;
                    
                } catch (PackageManagerException e) {
                    if ((policyFlags & PackageParser.PARSE_IS_SYSTEM_DIR) == 0) {
                        throw e;
                    }
                    // 如果签名校验出现问题，这里会先恢复成本次解析的签名！
                    pkgSetting.signatures.mSignatures = pkg.mSignatures;
                    // 但是如果是 sharedUser 的情况，就会报错！
                    if (pkgSetting.sharedUser != null) {
                        if (compareSignatures(pkgSetting.sharedUser.signatures.mSignatures,
                                              pkg.mSignatures) != PackageManager.SIGNATURE_MATCH) {
                            throw new PackageManagerException(
                                    INSTALL_PARSE_FAILED_INCONSISTENT_CERTIFICATES,
                                            "Signature mismatch for shared user: "
                                            + pkgSetting.sharedUser);
                        }
                    }
                    String msg = "System package " + pkg.packageName
                        + " signature changed; retaining data.";
                    reportSettingsProblem(Log.WARN, msg);
                }
            }

            //【14】安装的时候才会进入这个分支，这里不看！
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

            //【15】同样的，只有系统 apk 才能进入该分支，用于权限继承！
            if ((scanFlags & SCAN_CHECK_ONLY) == 0 && pkg.mAdoptPermissions != null) {
                for (int i = pkg.mAdoptPermissions.size() - 1; i >= 0; i--) {
                    final String origName = pkg.mAdoptPermissions.get(i);
                    final PackageSetting orig = mSettings.peekPackageLPr(origName);
                    if (orig != null) {
                        //【15.1】校验要被继承权限的 package 一是否存在，二是否是系统应用！
                        // 条件不满足，无法权限继承！
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
        // 从 base.apk 和其 split.apk（如果有）中选择修改时间最晚的作为扫描时间！
        final long scanFileTime = getLastModifiedTime(pkg, scanFile);
        final boolean forceDex = (scanFlags & SCAN_FORCE_DEX) != 0; // 没用到，应该是废弃代码！
        
        //【16】设置 pacakge 对应的进程名，如果 processName 为 null，默认进程名为包名！
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

        //【17】下面是设置本地库和系统平台相关的属性！！
        if ((scanFlags & SCAN_NEW_INSTALL) == 0) {
            derivePackageAbi(pkg, scanFile, cpuAbiOverride, true /* extract libs */);
            if (isSystemApp(pkg) && !pkg.isUpdatedSystemApp() &&
                    pkg.applicationInfo.primaryCpuAbi == null) {
                setBundledAppAbisAndRoots(pkg, pkgSetting);
                setNativeLibraryPaths(pkg);
            }
        } else {
            if ((scanFlags & SCAN_MOVE) != 0) {
                pkg.applicationInfo.primaryCpuAbi = pkgSetting.primaryCpuAbiString;
                pkg.applicationInfo.secondaryCpuAbi = pkgSetting.secondaryCpuAbiString;
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

        //【18】处理特权 apk 的子包，只有特权 apk 才能添加子包；
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
            //【19】如果是系统 package，并且他之前更新过，就要尝试对共享库 lib 进行更新！
            // 只有系统 package 才能添加共享 lib
            if ((pkg.applicationInfo.flags & ApplicationInfo.FLAG_SYSTEM) != 0) {
                if (pkg.libraryNames != null) {
                    for (int i=0; i<pkg.libraryNames.size(); i++) {
                        String name = pkg.libraryNames.get(i);
                        boolean allowed = false;
                        if (pkg.isUpdatedSystemApp()) {
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
                        // 如果不是开机扫描，我们需要更新下该应用的共享库，对于开机扫描的情况，我们会在扫描完成后自动更新
                        clientLibPkgs = updateAllSharedLibrariesLPw(pkg);
                    }
                }
            }
        }

        //【20】处理和冻结相关的逻辑
        if ((scanFlags & SCAN_BOOTING) != 0) {
            // 如果是开机扫描，不需要冻结，因为没有应用在此时可以运行！
        } else if ((scanFlags & SCAN_DONT_KILL_APP) != 0) {
            // 如果扫描过程中不允许 kill app，那就不会冻结！
        } else if ((scanFlags & SCAN_IGNORE_FROZEN) != 0) {
            // 如果扫描过程中显式指定忽略冻结，那就不会冻结！
        } else {
            // 其他情况，我们会默认冻结该应用，防止其启动！
            checkPackageFrozen(pkgName);
        }

        //【21】杀掉所以依赖于库文件的应用，因为库发生了更新！
        if (clientLibPkgs != null) {
            for (int i=0; i<clientLibPkgs.size(); i++) {
                PackageParser.Package clientPkg = clientLibPkgs.get(i);
                killApplication(clientPkg.applicationInfo.packageName,
                        clientPkg.applicationInfo.uid, "update lib");
            }
        }

        KeySetManagerService ksms = mSettings.mKeySetManagerService; // 校验 keyset 的真实性
        ksms.assertScannedPackageValid(pkg);

        Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "updateSettings"); // 记录 trace 事件！

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

            //【3.2.1.7.1.2 important】将新创建或者更新后的 PackageSetting 重新添加到 mSettings 中！
            mSettings.insertPackageSettingLPw(pkgSetting, pkg);
            
            //【important】将本次扫描的 Package 添加到 PMS.mPackages 中去！
            mPackages.put(pkg.applicationInfo.packageName, pkg);
            
            //【22】将当前的 package 从 mSettings.mPackagesToBeCleaned 中移除，防止被清除！
            final Iterator<PackageCleanItem> iter = mSettings.mPackagesToBeCleaned.iterator();
            while (iter.hasNext()) {
                PackageCleanItem item = iter.next();
                if (pkgName.equals(item.packageName)) {
                    iter.remove();
                }
            }

            //【23】处理 package 的第一次安装时间和最近更新时间！
            if (currentTime != 0) {
                if (pkgSetting.firstInstallTime == 0) {
                    pkgSetting.firstInstallTime = pkgSetting.lastUpdateTime = currentTime;

                } else if ((scanFlags & SCAN_UPDATE_TIME) != 0) {
                    pkgSetting.lastUpdateTime = currentTime;

                }
            } else if (pkgSetting.firstInstallTime == 0) {
                pkgSetting.firstInstallTime = pkgSetting.lastUpdateTime = scanFileTime;
                
            } else if ((policyFlags & PackageParser.PARSE_IS_SYSTEM_DIR) != 0) {
                if (scanFileTime != pkgSetting.timeStamp) {
                    pkgSetting.lastUpdateTime = scanFileTime;
                }
            }

            ksms.addScannedPackageLPw(pkg); // 将 package 的 KeySets 添加到 KeySetManagerService 中！
           
            //【24】处理四大组件！ 
            //【24.1】处理该 Package 中 的 Provider 信息，添加到 PMS.mProviders 中！
            int N = pkg.providers.size();
            StringBuilder r = null;
            int i;
            for (i=0; i<N; i++) {
                PackageParser.Provider p = pkg.providers.get(i);
                p.info.processName = fixProcessName(pkg.applicationInfo.processName,
                        p.info.processName, pkg.applicationInfo.uid);
                mProviders.addProvider(p); // 添加到 PMS.mProviders 中！
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

            //【24.2】处理该 Package 中的 Service 信息，添加到 PMS.mService 中！
            N = pkg.services.size();
            r = null;
            for (i=0; i<N; i++) {
                PackageParser.Service s = pkg.services.get(i);
                s.info.processName = fixProcessName(pkg.applicationInfo.processName,
                        s.info.processName, pkg.applicationInfo.uid);
                mServices.addService(s); // 添加到 PMS.mServices 中！
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

            //【24.3】处理该 Package 中的 BroadcastReceiver 信息,添加到 PMS.mReceivers 中。
            N = pkg.receivers.size();
            r = null;
            for (i=0; i<N; i++) {
                PackageParser.Activity a = pkg.receivers.get(i);
                a.info.processName = fixProcessName(pkg.applicationInfo.processName,
                        a.info.processName, pkg.applicationInfo.uid);
                mReceivers.addActivity(a, "receiver"); // 添加到 PMS.mReceivers 中！
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
            
            //【24.4】处理该 Package 中的 activity 信息，添加到 PMS.mActivities 中！
            N = pkg.activities.size();
            r = null;
            for (i=0; i<N; i++) {
                PackageParser.Activity a = pkg.activities.get(i);
                a.info.processName = fixProcessName(pkg.applicationInfo.processName,
                        a.info.processName, pkg.applicationInfo.uid);
                mActivities.addActivity(a, "activity"); // 添加到 PMS.mActivities 中！
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

            //【24.5】处理该 Package 中的 PermissionGroups 信息，添加到 PMS.mPermissionGroups 中！
            N = pkg.permissionGroups.size();
            r = null;
            for (i=0; i<N; i++) {
                PackageParser.PermissionGroup pg = pkg.permissionGroups.get(i);
                PackageParser.PermissionGroup cur = mPermissionGroups.get(pg.info.name);
                final String curPackageName = cur == null ? null : cur.info.packageName;
                //【5.1】如果 isPackageUpdate 为 true，说明要更新权限组的信息！
                // 如果 cur 为 null，说明是新添加的权限组信息！
                final boolean isPackageUpdate = pg.info.packageName.equals(curPackageName);
                if (cur == null || isPackageUpdate) {
                    mPermissionGroups.put(pg.info.name, pg); // 添加到 PMS.mPermissionGroups 中！
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
            
            //【24.6】处理该 Package 中的定义 Permission 和 Permission-tree 信息！
            N = pkg.permissions.size();
            r = null;
            for (i=0; i<N; i++) {
                PackageParser.Permission p = pkg.permissions.get(i);
                // 默认取消掉 PermissionInfo.FLAG_INSTALLED 标志位！
                p.info.flags &= ~PermissionInfo.FLAG_INSTALLED;

                //【6.1】设置权限所属的 group，前提是只有 Android5.1 以后才支持 Permission Groups！
                if (pkg.applicationInfo.targetSdkVersion > Build.VERSION_CODES.LOLLIPOP_MR1) {
                    p.group = mPermissionGroups.get(p.info.group);
                    if (p.info.group != null && p.group == null) {
                        Slog.w(TAG, "Permission " + p.info.name + " from package "
                                + p.info.packageName + " in an unknown group " + p.info.group);
                    }
                }

                //【6.2】从 Settings 中获得权限管理集合，如果该权限是一个 Permission-tree，
                // 那就返回 mSettings.mPermissionTrees；否则，返回 mSettings.mPermissions！
                ArrayMap<String, BasePermission> permissionMap =
                        p.tree ? mSettings.mPermissionTrees
                                : mSettings.mPermissions;

                //【6.3】尝试获得上次安装时该权限对应的 BasePermission 对象！
                BasePermission bp = permissionMap.get(p.info.name);

                //【6.4】允许系统应用来重新定义非系统权限！
                // 如果上次安装时的 BasePermission 不为 null，但是当前解析的 package 不是上次安装时该权限的定义者！
                if (bp != null && !Objects.equals(bp.sourcePackage, p.info.packageName)) {
                    //【6.4.1】判断上一次安装时，定义该权限的 package 是否是系统应用！
                    // 如果是 currentOwnerIsSystem 为 true！
                    final boolean currentOwnerIsSystem = (bp.perm != null
                            && isSystemApp(bp.perm.owner));
                    
                    // 如果当前解析的定义了该权限的 package 是系统 app，那么进入这里！     
                    if (isSystemApp(p.owner)) {
                        // 如果上次安装时，该权限是一个 BasePermission.TYPE_BUILTIN 系统权限，且 bp.perm 为 null，
                        // 即：拥有者未知，那么这里我们将这个 system package 分配给这个权限！
                        if (bp.type == BasePermission.TYPE_BUILTIN && bp.perm == null) {
                            bp.packageSetting = pkgSetting;
                            bp.perm = p;
                            bp.uid = pkg.applicationInfo.uid;
                            bp.sourcePackage = p.info.packageName;
                            p.info.flags |= PermissionInfo.FLAG_INSTALLED;
                        
                        // 判断上一次安装时，定义该权限的 package 不是系统应用，而定义相同权限的
                        // 本次解析的 package 是系统应用，那么该权限会被系统应用重新定义！
                        } else if (!currentOwnerIsSystem) {
                            String msg = "New decl " + p.owner + " of permission  "
                                    + p.info.name + " is system; overriding " + bp.sourcePackage;
                            reportSettingsProblem(Log.WARN, msg);
                            // 设置 bp 为 null！
                            bp = null;
                        }
                    }
                }

                // 因为 bp 为 null，所谓我们会使用系统应用重新定义权限，如果是 permission-tree，会被添加到 
                // mSettings.mPermissionTrees 中！
                if (bp == null) {
                    bp = new BasePermission(p.info.name, p.info.packageName,
                            BasePermission.TYPE_NORMAL);
                    permissionMap.put(p.info.name, bp);
                }
                // 重新分配拥有者为该系统应用！
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
                // 如果上次安装时的权限的定义者，就是本次解析的 package，设置 protectionLevel！
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
                ... ... ... ...// 这里省略掉，暂时不看！
            }
            if (r != null) {
                if (DEBUG_PACKAGE_SCANNING) Log.d(TAG, "  Instrumentation: " + r);
            }
            
            //【25】处理该 Package 中的 protectedBroadcasts 信息！
            if (pkg.protectedBroadcasts != null) {
                N = pkg.protectedBroadcasts.size();
                for (i=0; i<N; i++) {
                    mProtectedBroadcasts.add(pkg.protectedBroadcasts.get(i));
                }
            }
            //【27】更新 PackageSettings 中的时间戳！
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
到这里，对于扫描的 package 的数据就处理完了！

###### 3.2.1.7.1.1 Settings.getPackageLPw

上面又再次调用了 getPackageLPw 方法，注意根据参数传递：

- boolean add：传入的值为 false！
- boolean allowInstall：传入的值为 true；

该方法传入的参数的是本次扫描解析获得的 apk 的数据！

```java
    // 最终进入这个方法！
    private PackageSetting getPackageLPw(String name, PackageSetting origPackage,
            String realName, SharedUserSetting sharedUser, File codePath, File resourcePath,
            String legacyNativeLibraryPathString, String primaryCpuAbiString,
            String secondaryCpuAbiString, int vc, int pkgFlags, int pkgPrivateFlags,
            UserHandle installUser, boolean add, boolean allowInstall, String parentPackage,
            List<String> childPackageNames) {
        
        //【1】尝试获得之前的安装数据！
        PackageSetting p = mPackages.get(name);
        UserManagerService userManager = UserManagerService.getInstance();
        
        //【2】如果 p 不为 null，说明这个 apk 之前就存在！
        // 下面就要比较 codePath！
        if (p != null) {
            p.primaryCpuAbiString = primaryCpuAbiString;
            p.secondaryCpuAbiString = secondaryCpuAbiString;
            if (childPackageNames != null) {
                p.childPackageNames = new ArrayList<>(childPackageNames);
            }

            if (!p.codePath.equals(codePath)) {
                if ((p.pkgFlags & ApplicationInfo.FLAG_SYSTEM) != 0) {
                    // 这是被更新的 system app 的情况，此时在 system 分区和 app 分区都会有该 apk，我们只需要
                    // 将高 versionCode 的 apk 显示出来即可！
                    Slog.w(PackageManagerService.TAG, "Trying to update system app code path from "
                            + p.codePathString + " to " + codePath.toString());
                } else {
                    // 这种情况是说明这个 system apk 的路径发生了变化！
                    // 或者说这个应用是一个三方应用！
                    Slog.i(PackageManagerService.TAG, "Package " + name + " codePath changed from "
                            + p.codePath + " to " + codePath + "; Retaining data and using new");
                    // 如果当前扫描的是 system apk，并且其不是被覆盖安装的，即 getDisabledSystemPkgLPr 为 null；
                    // 更新所有用户下的安装状态为 true！
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
                // 如果共享 uid 匹配，或者没有使用共享，进入这里！
                // 如果我们扫描的是一个 system 分区的 apk,  无论其之前是否在 data 分区存在！
                // 我们都会给上一次的安装信息设置 FLAG_SYSTEM 和 PRIVATE_FLAG_PRIVILEGED 标志位；
                // 这里和【3.2.1】【B】有关联
                p.pkgFlags |= pkgFlags & ApplicationInfo.FLAG_SYSTEM;
                p.pkgPrivateFlags |= pkgPrivateFlags & ApplicationInfo.PRIVATE_FLAG_PRIVILEGED;
            }
        }
        
        //【3】处理需要重新创建 PackageSetting 的情况！
        // 如果 p 为 null，说明这个系统 apk 是通过 OTA 升级方式新添加的，或者 shareUserId 不匹配！
        // 如果有源包，就用源包的命名，创建新的 PackageSetting 对象，并源包的数据来初始化！
        // 没有源包，就用当前包的命名，创建新的 PackageSetting 对象！
        if (p == null) {
            if (origPackage != null) {
                //【3.1】源包不为 null 的情况，进入这里，使用本次扫描的信息创建新的 PackageSetting！
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
                //【3.2】源包为 null 的情况，进入这里，使用本次扫描的信息创建新的 PackageSetting！
                p = new PackageSetting(name, realName, codePath, resourcePath,
                        legacyNativeLibraryPathString, primaryCpuAbiString, secondaryCpuAbiString,
                        null /* cpuAbiOverrideString */, vc, pkgFlags, pkgPrivateFlags,
                        parentPackage, childPackageNames);

                p.setTimeStamp(codePath.lastModified());
                p.sharedUser = sharedUser;

                // 系统 apk 不会进入这个分支！
                if ((pkgFlags&ApplicationInfo.FLAG_SYSTEM) == 0) {
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
        
        //【4】返回查到或者是创建的新的系统 package 对应的 PackageSetting 对象！
        return p;
    }

```

###### 3.2.1.7.1.2 Settings.insertPackageSettingLPw

该方法会将 PackageSetting 保存到 mUserIds 或 mOtherIds 中去，同时用本次扫描的的 Package 更新 PackageSetting！

```java
    void insertPackageSettingLPw(PackageSetting p, PackageParser.Package pkg) {
        p.pkg = pkg; //【1】建立引用关系！
        // pkg.mSetEnabled = p.getEnabled(userId);
        // pkg.mSetStopped = p.getStopped(userId);
        final String volumeUuid = pkg.applicationInfo.volumeUuid;
        final String codePath = pkg.applicationInfo.getCodePath();
        final String resourcePath = pkg.applicationInfo.getResourcePath();
        final String legacyNativeLibraryPath = pkg.applicationInfo.nativeLibraryRootDir;
        // Update volume if needed
        if (!Objects.equals(volumeUuid, p.volumeUuid)) {
            Slog.w(PackageManagerService.TAG, "Volume for " + p.pkg.packageName +
                    " changing from " + p.volumeUuid + " to " + volumeUuid);
            p.volumeUuid = volumeUuid;
        }
        // Update code path if needed
        if (!Objects.equals(codePath, p.codePathString)) {
            Slog.w(PackageManagerService.TAG, "Code path for " + p.pkg.packageName +
                    " changing from " + p.codePathString + " to " + codePath);
            p.codePath = new File(codePath);
            p.codePathString = codePath;
        }
        //Update resource path if needed
        if (!Objects.equals(resourcePath, p.resourcePathString)) {
            Slog.w(PackageManagerService.TAG, "Resource path for " + p.pkg.packageName +
                    " changing from " + p.resourcePathString + " to " + resourcePath);
            p.resourcePath = new File(resourcePath);
            p.resourcePathString = resourcePath;
        }
        // Update the native library paths if needed
        if (!Objects.equals(legacyNativeLibraryPath, p.legacyNativeLibraryPathString)) {
            p.legacyNativeLibraryPathString = legacyNativeLibraryPath;
        }

        // Update the required Cpu Abi
        p.primaryCpuAbiString = pkg.applicationInfo.primaryCpuAbi;
        p.secondaryCpuAbiString = pkg.applicationInfo.secondaryCpuAbi;
        p.cpuAbiOverrideString = pkg.cpuAbiOverride;
        // Update version code if needed
        if (pkg.mVersionCode != p.versionCode) {
            p.versionCode = pkg.mVersionCode;
        }
        // Update signatures if needed.
        if (p.signatures.mSignatures == null) {
            p.signatures.assignSignatures(pkg.mSignatures);
        }
        // Update flags if needed.
        if (pkg.applicationInfo.flags != p.pkgFlags) {
            p.pkgFlags = pkg.applicationInfo.flags;
        }
        // If this app defines a shared user id initialize
        // the shared user signatures as well.
        if (p.sharedUser != null && p.sharedUser.signatures.mSignatures == null) {
            p.sharedUser.signatures.assignSignatures(pkg.mSignatures);
        }
        //【2】将 PackageSetting 保存到 mUserIds 或则会 mOtherIds 中去
        addPackageSettingLPw(p, pkg.packageName, p.sharedUser);
    }
```
这里就不多说了！

###### 3.2.1.7.1.3 ActivityIntentResolver.addActivity - 添加 activity

这里先不关注～～



#### 3.2.1.8 Settings.disableSystemPackageLPw

当需要隐藏掉 system app 的时候，会调用 disableSystemPackageLPw 方法：

参数传递：

- boolean replaced：true！

```java
    boolean disableSystemPackageLPw(String name, boolean replaced) {
        // 判断一下，这个系统 apk 是否已经安装，没有安装就不用 disable！
        final PackageSetting p = mPackages.get(name);
        if(p == null) {
            Log.w(PackageManagerService.TAG, "Package " + name + " is not an installed package");
            return false;
        }
        
        // 系统 apk 已经安装了，并且之前没有被覆盖更新过，
        // 且之前没有添加 ApplicationInfo.FLAG_UPDATED_SYSTEM_APP 标签！
        final PackageSetting dp = mDisabledSysPackages.get(name);

        if (dp == null && p.pkg != null && p.pkg.isSystemApp() && !p.pkg.isUpdatedSystemApp()) {

            if((p.pkg != null) && (p.pkg.applicationInfo != null)) {
                p.pkg.applicationInfo.flags |= ApplicationInfo.FLAG_UPDATED_SYSTEM_APP;
            }
            // 将其添加到 mDisabledSysPackages 列表中！
            mDisabledSysPackages.put(name, p);

            if (replaced) { // 这里为 true
                //【3.2.1.8.1】为更新的后的应用重新创建一个 PackageSetting，添加到 mPackages 中
                // 因为旧的 PackageSetting 被添加到了 mDisabledSysPackages！
                PackageSetting newp = new PackageSetting(p);
                replacePackageLPw(name, newp);
            }
            return true;
        }
        return false;
    }
```
这里面会设置到 pkgFlags 和 pkgPrivateFlags 的设置：
```java
    void setFlags(int pkgFlags) {
        this.pkgFlags = pkgFlags
                & (ApplicationInfo.FLAG_SYSTEM
                        | ApplicationInfo.FLAG_EXTERNAL_STORAGE);
    }

    void setPrivateFlags(int pkgPrivateFlags) {
        this.pkgPrivateFlags = pkgPrivateFlags
                & (ApplicationInfo.PRIVATE_FLAG_PRIVILEGED
                | ApplicationInfo.PRIVATE_FLAG_FORWARD_LOCK
                | ApplicationInfo.PRIVATE_FLAG_REQUIRED_FOR_SYSTEM_USER);
    }
```

到这里 system 分区的扫描流程就结束了！
##### 3.2.1.8.1 Settings.replacePackageLPw

```java
    private void replacePackageLPw(String name, PackageSetting newp) {
        final PackageSetting p = mPackages.get(name);
        if (p != null) {
            if (p.sharedUser != null) {
                p.sharedUser.removePackage(p);
                p.sharedUser.addPackage(newp);
            } else {
                replaceUserIdLPw(p.appId, newp);
            }
        }
        mPackages.put(name, newp);
    }
```
更新 mPackages 中的数据！

###### 3.2.1.8.1.1 Settings.replaceUserIdLPw

```java
    private void replaceUserIdLPw(int uid, Object obj) {
        if (uid >= Process.FIRST_APPLICATION_UID) {
            final int N = mUserIds.size();
            final int index = uid - Process.FIRST_APPLICATION_UID;
            if (index < N) mUserIds.set(index, obj);
        } else {
            mOtherUserIds.put(uid, obj);
        }
    }

```
更新 mOtherUserIds 和 mUserIds 中的数据！

## 3.3 扫描总结

下面我们来总结下扫描过程：

### 3.3.1 扫描过程总结
接下来，用一张图来总结一下，扫描的过程：
![扫描系统目录.png-116.5kB](https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/23/扫描系统目录.png)

流程很清晰了，不多说。

### 3.3.2 数据关系总结

下面我们来看看解析获得的数据之间的关系：
![扫描数据关系图.png-354.6kB](https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/23/扫描数据关系图.png) 

这个图展示了解析得到的一个 Package 对象在内存中的结果。

- 组件的基类是 Component，AndroidManifest.xml 中定义的组件 activity，service，provider，permission等，都继承了 Component；
- activity，service 这样的组件，都可以配置 <intent-filter> 属性，在 pms 中体现在 IntentInfo 这个数据结构中，从上图可以看到，每一种组件所依赖的 intentInfo 不同，Component 和 IntentInfo 之间的依赖关系采用了桥接模式，通过泛型实现！


# 4 系统目录扫描 - 收尾阶段

接下来，是收集可能被删掉的系统应用包！
```java
            //【×】用于收集可能已经被删掉的系统应用包！
            final List<String> possiblyDeletedUpdatedSystemApps = new ArrayList<String>();

            if (!mOnlyCore) {
                // 遍历上次安装的所有的 package！
                Iterator<PackageSetting> psit = mSettings.mPackages.values().iterator();
                while (psit.hasNext()) {
                    PackageSetting ps = psit.next();
                    //【×】如果是非系统应用，跳过，如果是被覆盖安装！
                    if ((ps.pkgFlags & ApplicationInfo.FLAG_SYSTEM) == 0) {
                        continue;
                    }

                    final PackageParser.Package scannedPkg = mPackages.get(ps.name);
                    if (scannedPkg != null) {
                        //【4.1】如果系统应用包不仅被扫描过（在mPackages中），并且在不可用列表中！
                        // 说明一定是通过覆盖安装的，移除之前扫描的结果，保证之前用户安装的应用能够被扫描！
                        if (mSettings.isDisabledSystemPackageLPr(ps.name)) {
                            logCriticalInfo(Log.WARN, "Expecting better updated system app for "
                                    + ps.name + "; removing system app.  Last known codePath="
                                    + ps.codePathString + ", installStatus=" + ps.installStatus
                                    + ", versionCode=" + ps.versionCode + "; scanned versionCode="
                                    + scannedPkg.mVersionCode);
                            
                            //【4.2】将之前的扫描结果移除！
                            removePackageLI(scannedPkg, true);
                            // 将这包添加到 mExpectingBetter 列表中！
                            mExpectingBetter.put(ps.name, ps.codePath);
                        }
                        
                        // 跳出循环，确保不会被删掉！
                        continue;
                    }

                    if (!mSettings.isDisabledSystemPackageLPr(ps.name)) {
                        // 如果系统应用包没有被扫描，并且他也不在不可用的列表中，说明其没有被更新过，
                        // 可能被删除了，也可能被移动到了 data 分区，那就清楚其前面的数据！
                        // 如果移动到了 data 分区，那就在扫描 data 的过程中处理！
                        psit.remove();
                        logCriticalInfo(Log.WARN, "System package " + ps.name
                                + " no longer exists; it's data will be wiped");

                    } else { 
                        // 如果系统应用包没有被扫描，却在不可用的列表中，说明其被更新过了，但是 apk 被删除了
                        // 就将他加入到 possiblyDeletedUpdatedSystemApps 集合中！
                        final PackageSetting disabledPs = mSettings.getDisabledSystemPkgLPr(ps.name);
                        if (disabledPs.codePath == null || !disabledPs.codePath.exists()) {
                            possiblyDeletedUpdatedSystemApps.add(ps.name);
                        }
                    }
                }
            }

            //【×】清理所有安装不完全的应用包！
            ArrayList<PackageSetting> deletePkgsList = mSettings.getListOfIncompleteInstallPackagesLPr();
            for (int i = 0; i < deletePkgsList.size(); i++) {
                final String packageName = deletePkgsList.get(i).name;
                logCriticalInfo(Log.WARN, "Cleaning up incompletely installed app: " + packageName);
                synchronized (mPackages) {
                    mSettings.removePackageLPw(packageName);
                }
            }

            // 移除临时文件
            deleteTempPackageFiles();

            // 移除没有和应用程序包相关联的共享用户 id！
            mSettings.pruneSharedUsersLPw();
```
对于已经被扫描到的系统 app，如果发现他存在于被更新过的列表中，说明这个系统 apk 在 data 分区有更新的版本了，那就需要把之前扫描的清楚掉，这样我们要确保在扫描 data 分区时，data 分区的 app 能够显示出来！！

## 4.1 PMS.isDisabledSystemPackageLPr
```java
    boolean isDisabledSystemPackageLPr(String name) {
        return mDisabledSysPackages.containsKey(name);
    }
```
## 4.2 PMS.removePackageLI

removePackageLI 会移除指定 pacakge 的扫描结果！

```java
    private void removePackageLI(PackageParser.Package pkg, boolean chatty) {
        // Remove the parent package setting
        PackageSetting ps = (PackageSetting) pkg.mExtras;
        if (ps != null) {
            removePackageLI(ps, chatty);
        }
        // Remove the child package setting
        final int childCount = (pkg.childPackages != null) ? pkg.childPackages.size() : 0;
        for (int i = 0; i < childCount; i++) {
            PackageParser.Package childPkg = pkg.childPackages.get(i);
            ps = (PackageSetting) childPkg.mExtras;
            if (ps != null) {
                removePackageLI(ps, chatty);
            }
        }
    }

    void removePackageLI(PackageSetting ps, boolean chatty) {
        if (DEBUG_INSTALL) {
            if (chatty)
                Log.d(TAG, "Removing package " + ps.name);
        }

        // writer
        synchronized (mPackages) {
            mPackages.remove(ps.name);
            final PackageParser.Package pkg = ps.pkg;
            if (pkg != null) {
                cleanPackageDataStructuresLILPw(pkg, chatty);
            }
        }
    }
```

# 5 总结

我们用一张流程图来总结一下系统 apk 的扫描解析处理过程：

<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/23/系统 apk 的扫描流程.png" alt="系统 apk 的扫描流程" style="zoom: 33%;" /> 


- 需要隐藏的系统 apk 都会被添加到 mExpectingBetter 列表中：
1、系统 apk 被覆盖安装过，且 system 分区的 versionCode 的是不高于 data 分区的；
2、之前已经有一相同包名的 apk 安装在 data 分区中了，这次有同样的包名的 app 出现在了 system 分区， versionCode 不高于 data 分区的；

![解析过程](https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/23/解析过程.png)
