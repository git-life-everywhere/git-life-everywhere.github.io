# PMS 第 1 篇 - PackageManagerService 初始化
title: PMS 第 1 篇 - PackageManagerService 初始化
date: 2018/01/03
categories:
- AndroidFramework源码分析
- PackageManager包管理
tags: PackageManager包管理
---

[toc]

本文基于 Android 7.1.1 系统源码，分析 PackageManagerService 的架构和主要业务实现，本文是作者原创，转载请说明出处！

# 前言

PMS 用来管理所有的 package 信息，包括安装、卸载、更新以及解析 AndroidManifest.xml 以组织相应的数据结构，这些数据结构将会被 其他 service 和 application 使用到。

# 1 PMS 的启动

先来看 SystemServer 中 PackageManagerService 的启动代码：
```java
// 这里的 installer 也是一个系统服务
mPackageManagerService = PackageManagerService.main(mSystemContext, installer, 
                                                    mFactoryTestMode != FactoryTest.FACTORY_TEST_OFF, mOnlyCore);
mFirstBoot = mPackageManagerService.isFirstBoot();
mPackageManager = mSystemContext.getPackageManager();
```

main 函数很简单，只有短短几行代码，执行时间却较长，主要原因是 PKMS 在其构造函数中做了很多“重体力活”，这也是 Android 启动速度慢的主要原因之一。

```java
    // 先来看看PackageManangerService的main方法：
    public static PackageManagerService main(Context context, Installer installer,
            boolean factoryTest, boolean onlyCore) {
        // Self-check for initial settings.
        PackageManagerServiceCompilerMapping.checkProperties();

        PackageManagerService m = new PackageManagerService(context, installer,
                factoryTest, onlyCore);
        m.enableSystemUserPackages();
        ServiceManager.addService("package", m);
        return m;
    }
```

首先构造一个 PMS 对象，然后调用 ServiceManager 的 addService 注册 PackageManagerService 服务。

构造函数有如下 4 个参数：

- 第一个参数：系统进程的上下文实例；
- 第二个参数：Installer 对象，用于和 Installd 通信使用，我们后面分析 Installd 再来介绍；
- 第三个参数：factoryTest 为出厂测试，默认为 false；
- 第四个参数：onlyCore 表示是否只加载核心的应用，默认也为 false。

# 2 PMS 的初始化
PackageManagerService 的构造器代码结构如下：

```java
public PackageManagerService(Context context, Installer installer,
        boolean factoryTest, boolean onlyCore) {

    EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_PMS_START, SystemClock.uptimeMillis());

    ...
    
    synchronized (mInstallLock) {
        synchronized (mPackages) {
            
            ...
            
            EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_PMS_SYSTEM_SCAN_START, startTime);
            
            ...
            
            if (!mOnlyCore) {
                EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_PMS_DATA_SCAN_START, SystemClock.uptimeMillis());
            
            }
            
            ...
            
            EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_PMS_SCAN_END, SystemClock.uptimeMillis());
            
            ...
            
            EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_PMS_READY, SystemClock.uptimeMillis());
            
            ...
        } 
    }
    
    Runtime.getRuntime().gc();

    LocalServices.addService(PackageManagerInternal.class, new PackageManagerInternalImpl());
}
```
我们根据 EventLog 可以将 PMS 的初始化分为以下几个过程：

- 阶段1：BOOT_PROGRESS_PMS_START
- 阶段2：BOOT_PROGRESS_PMS_SYSTEM_SCAN_START
- 阶段3：BOOT_PROGRESS_PMS_DATA_SCAN_START
- 阶段4：BOOT_PROGRESS_PMS_SCAN_END
- 阶段5：BOOT_PROGRESS_PMS_READY

下面，来看看 PackageManangerService 的构造器方法代码，为了便于分析，我们把 PackageManagerService 构造器的代码分为如下几个部分：

## **2.1 PMS_START**
这个的阶段的代码，如下 ：
```java
       public PackageManagerService(Context context, Installer installer,
            boolean factoryTest, boolean onlyCore) {
        EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_PMS_START,
                SystemClock.uptimeMillis());

        if (mSdkVersion <= 0) {
            //【1】mSdkVersion 是 PKMS 的成员变量，定义的时候进行赋值，
            // 其值取自系统属性“ro.build.version.sdk”，即编译的 SDK 版本。
            Slog.w(TAG, "**** ro.build.version.sdk not set!");
        }

        mContext = context;
        mFactoryTest = factoryTest; // 默认为 false，即运行在非工厂模式下
        mOnlyCore = onlyCore; // 默认为 false,标记是否是只加载核心服务
        
        //【2】用于存储与显示屏相关的一些属性，例如屏幕的宽 / 高尺寸，分辨率等信息。
        mMetrics = new DisplayMetrics(); 

        //【3】在 Settings 中，创建 packages.xml、packages-backup.xml、packages.list 等文件对象
        // 并添加 system, phone, log, nfc, bluetooth, shell 这六种 shareUserId 到 mSettings，后面扫描和安装有用！
        mSettings = new Settings(mPackages);
        mSettings.addSharedUserLPw("android.uid.system", Process.SYSTEM_UID,
                ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
        mSettings.addSharedUserLPw("android.uid.phone", RADIO_UID,
                ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
        mSettings.addSharedUserLPw("android.uid.log", LOG_UID,
                ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
        mSettings.addSharedUserLPw("android.uid.nfc", NFC_UID,
                ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
        mSettings.addSharedUserLPw("android.uid.bluetooth", BLUETOOTH_UID,
                ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
        mSettings.addSharedUserLPw("android.uid.shell", SHELL_UID,
                ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);

        String separateProcesses = SystemProperties.get("debug.separate_processes");
        if (separateProcesses != null && separateProcesses.length() > 0) {
            if ("*".equals(separateProcesses)) {
                mDefParseFlags = PackageParser.PARSE_IGNORE_PROCESSES;
                mSeparateProcesses = null;
                Slog.w(TAG, "Running with debug.separate_processes: * (ALL)");
            } else {
                mDefParseFlags = 0;
                mSeparateProcesses = separateProcesses.split(",");
                Slog.w(TAG, "Running with debug.separate_processes: "
                        + separateProcesses);
            }
        } else {
            mDefParseFlags = 0;
            mSeparateProcesses = null;
        }

        //【3】初始化 mInstaller 对象，installer 对象和 Native 进程 installd 交互！
        mInstaller = installer;
        
        //【4】创建 PackageDexOptimizer 对象，用于 dex 优化！
        mPackageDexOptimizer = new PackageDexOptimizer(installer, mInstallLock, context,
                "*dexopt*");
                
        //【5】创建 MoveCallbacks 对象，用于操作回滚！
        mMoveCallbacks = new MoveCallbacks(FgThread.get().getLooper());
        
        //【6】创建 OnPermissionChangeListeners 对象，用于监听权限改变！
        mOnPermissionChangeListeners = new OnPermissionChangeListeners(
                FgThread.get().getLooper());

        getDefaultDisplayMetrics(context, mMetrics); // 获得默认的显示

        //【7】创建 SystemConfig 对象，用于获取系统配置信息
        // 主要有 /system/etc/ 目录和 /system/etc/sysconfig 目录下的 sysconfig 和 permissions 文件夹xml文件
        SystemConfig systemConfig = SystemConfig.getInstance();
        mGlobalGids = systemConfig.getGlobalGids();
        mSystemPermissions = systemConfig.getSystemPermissions();
        mAvailableFeatures = systemConfig.getAvailableFeatures();

        mProtectedPackages = new ProtectedPackages(mContext);

        synchronized (mInstallLock) {
        // writer
        synchronized (mPackages) {
        
            //【8】建立并启动一个名为 “PackageManager” 的 HandlerThread，类型是 ServiceThread，处理安装卸载的消息！
            mHandlerThread = new ServiceThread(TAG,
                    Process.THREAD_PRIORITY_BACKGROUND, true /*allowIo*/);
            mHandlerThread.start();
            
            //【9】创建一个 PackageHandler 对象，绑定前面创建的 HandlerThread！
            mHandler = new PackageHandler(mHandlerThread.getLooper());
            mProcessLoggingHandler = new ProcessLoggingHandler();
            Watchdog.getInstance().addThread(mHandler, WATCHDOG_TIMEOUT);

            //【10】创建 DefaultPermissionGrantPolicy 对象，用于给某些预制的apk，给予权限,也可以撤销
            mDefaultPermissionPolicy = new DefaultPermissionGrantPolicy(this);
            
            //【11】创建需要扫描检测的目录文件对象，为扫描做准备！
            File dataDir = Environment.getDataDirectory();
            mAppInstallDir = new File(dataDir, "app"); // 目录 /data/data，存放应用程序的数据；
            mAppLib32InstallDir = new File(dataDir, "app-lib");  // 目录 /data/app-lib；
            mEphemeralInstallDir = new File(dataDir, "app-ephemeral");  // 目录 /data/app-ephemeral；
            mAsecInternalPath = new File(dataDir, "app-asec").getPath();  // 目录 /data/app-asec；
            mDrmAppPrivateInstallDir = new File(dataDir, "app-private");  // 目录 /data/app-private，私有应用程序！
                
            //【11】创建用户管理服务！    
            sUserManager = new UserManagerService(context, this, mPackages);

            //【12】获得权限信息！
            ArrayMap<String, SystemConfig.PermissionEntry> permConfig
                    = systemConfig.getPermissions();
            for (int i=0; i < permConfig.size(); i++) {
                SystemConfig.PermissionEntry perm = permConfig.valueAt(i);
                BasePermission bp = mSettings.mPermissions.get(perm.name);

                if (bp == null) {
                    bp = new BasePermission(perm.name, "android", BasePermission.TYPE_BUILTIN);
                    mSettings.mPermissions.put(perm.name, bp);
                }

                if (perm.gids != null) {
                    bp.setGids(perm.gids, perm.perUser);
                }
            }

            //【13】处理共享库！
            ArrayMap<String, String> libConfig = systemConfig.getSharedLibraries();
            for (int i=0; i<libConfig.size(); i++) {
                mSharedLibraries.put(libConfig.keyAt(i),
                        new SharedLibraryEntry(libConfig.valueAt(i), null));
            }

            mFoundPolicyFile = SELinuxMMAC.readInstallPolicy();
            
            // 判断是否是第一次开机！
            // 并获得上一次的安装信息！
            mFirstBoot = !mSettings.readLPw(sUserManager.getUsers(false));

            // 移除哪些 codePath 无效的 Package，恢复处于 system 目录下的同名 package
            final int packageSettingCount = mSettings.mPackages.size();
            for (int i = packageSettingCount - 1; i >= 0; i--) {
                PackageSetting ps = mSettings.mPackages.valueAt(i);

                if (!isExternal(ps) && (ps.codePath == null || !ps.codePath.exists())
                        && mSettings.getDisabledSystemPkgLPr(ps.name) != null) {
                    mSettings.mPackages.removeAt(i);
                    mSettings.enableSystemPackageLPw(ps.name);
                }
            }

            if (mFirstBoot) {
                // 如果是第一次开机，从另外一个系统拷贝 odex 文件到当前系统的 data 分区
                // Android 7.1 引进了 AB 升级，这个是 AB 升级的特性，可以先不看！
                requestCopyPreoptedFiles();
            }

            String customResolverActivity = Resources.getSystem().getString(
                    R.string.config_customResolverActivity);
            if (TextUtils.isEmpty(customResolverActivity)) {
                customResolverActivity = null;
            } else {
                mCustomResolverComponentName = ComponentName.unflattenFromString(
                        customResolverActivity);
            }
      
            ... ... ... ...// 见，第二阶段
```
第一阶段，主要是做一些初始化工作，为后续的扫描做准备！

### **1.2.1 主要流程**

- 创建 Settings 对象，创建 packages.xml、packages-backup.xml、packages.list 等文件对象并添加 system, phone, log, nfc, bluetooth, shell 这六种 shareUserId 到 mSettings，后面扫描和安装有用；
- 初始化 mInstaller 对象，用于和 installed 交互；
- 创建 PackageDexOptimizer 对象，用于 dex 优化；
- 创建 OnPermissionChangeListeners 对象，用于监听权限改变；
- 创建 SystemConfig 对象，用于获取系统配置信息；
- 创建用户管理服务！  
- 获得权限信息！
- 获得系统共享库！


## **2.2 PMS_SYSTEM_SCAN_START**
下面是系统扫描阶段：
```java
            ... ... ... ...// 接上面
            
            long startTime = SystemClock.uptimeMillis();
             
            EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_PMS_SYSTEM_SCAN_START,
                    startTime);

            // Set flag to monitor and not change apk file paths when
            // scanning install directories.
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
                // NOTE: For now, we're compiling these system "shared libraries"
                // (and framework jars) into all available architectures. It's possible
                // to compile them only when we come across an app that uses them (there's
                // already logic for that in scanPackageLI) but that adds some complexity.

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
            // 判断是否是 OTA 升级！
            mIsUpgrade = !Build.FINGERPRINT.equals(ver.fingerprint);

            // 判断是否是从 Android 6.0 升级过来的，如果是就需要把系统 app 的权限从安装时提高到运行时
            mPromoteSystemApps =
                    mIsUpgrade && ver.sdkVersion <= Build.VERSION_CODES.LOLLIPOP_MR1;

            // When upgrading from pre-N, we need to handle package extraction like first boot,
            // as there is no profiling data available.
            // 判断是否是从 Android 7.0 升级过来的！
            mIsPreNUpgrade = mIsUpgrade && ver.sdkVersion < Build.VERSION_CODES.N;

            mIsPreNMR1Upgrade = mIsUpgrade && ver.sdkVersion < Build.VERSION_CODES.N_MR1;

            // save off the names of pre-existing system packages prior to scanning; we don't
            // want to automatically grant runtime permissions for new system apps
            // 保存从 Android 6.0 升级前已经存在的系统应用包，并对他们进行优先扫描！
            if (mPromoteSystemApps) {
                Iterator<PackageSetting> pkgSettingIter = mSettings.mPackages.values().iterator();
                while (pkgSettingIter.hasNext()) {
                    PackageSetting ps = pkgSettingIter.next();
                    if (isSystemApp(ps)) {
                        mExistingSystemPackages.add(ps.name);
                    }
                }
            }

            // Collect vendor overlay packages.
            // (Do this before scanning any apps.)
            // For security and version matching reason, only consider
            // overlay packages if they reside in VENDOR_OVERLAY_DIR.
            // 扫描收集目录 /vendor/overlay 下的供应商应用包！
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

            // 删除已经不存在的系统应用包！
            final List<String> possiblyDeletedUpdatedSystemApps = new ArrayList<String>();
            if (!mOnlyCore) {
                Iterator<PackageSetting> psit = mSettings.mPackages.values().iterator();
                while (psit.hasNext()) {
                    PackageSetting ps = psit.next();

                    // 如果不是系统应用，跳过！
                    if ((ps.pkgFlags & ApplicationInfo.FLAG_SYSTEM) == 0) {
                        continue;
                    }

                    // 如果系统应用包已经被扫描到了（scannedPkg ！= null），就不会被删掉！
                    final PackageParser.Package scannedPkg = mPackages.get(ps.name);
                    if (scannedPkg != null) {
                        /*
                         * If the system app is both scanned and in the
                         * disabled packages list, then it must have been
                         * added via OTA. Remove it from the currently
                         * scanned package so the previously user-installed
                         * application can be scanned.
                         */
                        // 如果系统应用包不仅被扫描过（在mPackages中），并且在不可用列表中！
                        // 说明一定是通过覆盖安装的，移除之前扫描的结果，保证之前用户安装的应用能够被扫描！
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
总结一下第二阶段：

### **2.2.1 主要流程**

- 获得环境变量：BOOTCLASSPATH 和 SYSTEMSERVERCLASSPATH！
- 对共享库进行 odex 优化操作！
- 判断是否是通过 OTA 升级的
    - 如果是从 6.0 升级过来的，保存从 Android 6.0 升级前已经存在的系统应用包，并对他们进行优先扫描！
- 扫描收集以下目录中的供应商应用包！
     - /vendor/overlay 	
     - /system/framework 	
     - /system/priv-app 	
     - /system/app
     - /vendor/app 	
     - /oem/app
- 删除已经不存在的系统应用包！
- 清理所有安装不完全的应用包！
- 移除临时文件
- 移除没有和应用程序包相关联的共享用户 id

### **2.2.2 细节分析**
#### **2.2.2.1 环境变量**

我们可以通过 adb shell env 来查看系统所有的环境变量及相应值。也可通过命令：adb shell echo $var 来看指定的环境变量的值！

- **SYSTEMSERVERCLASSPATH**：
  - 主要是系统服务 jar 包的路径！
  - 主要文件如下：
        - /system/framework/services.jar
        - /system/framework/ethernet-service.jar
        - /system/framework/wifi-service.jar

- **BOOTCLASSPATH**：
   - 该环境变量内容较多，第三方定制的系统可能有所不同，原生内容包含 /system/framework目录下的framework.jar，ext.jar，core-libart.jar，telephony-common.jar，ims-common.jar，core-junit.jar等文件！
   - 主要文件如下：
         - /system/framework/core-libart.jar 
         - /system/framework/conscrypt.jar
         - /system/framework/okhttp.jar 
         - /system/framework/core-junit.jar
         - /system/framework/bouncycastle.jar 
         - /system/framework/ext.jar
         - /system/framework/framework.jar
         - /system/framework/telephony-common.jar
         - /system/framework/voip-common.jar 
         - /system/framework/ims-common.jar
         - /system/framework/apache-xml.jar
         - /system/framework/org.apache.http.legacy.boot.jar
         - /system/framework/ifaamanager.jar 
         - /system/framework/tcmiface.jar
         - /system/framework/WfdCommon.jar
         - /system/framework/com.qti.dpmframework.jar
         - /system/framework/dpmapi.jar
         - /system/framework/com.qti.location.sdk.jar
         - /system/framework/qcom.fmradio.jar
         - /system/framework/qcmediaplayer.jar

#### **2.2.2.2  dexopt 优化**

执行 dex 优化操作的文件有以下几类：

- mSharedLibraries：该共享库下的所有文件，是由 SystemConfig 构造函数中赋值的；
- /system/framework：该目录的所有 apk 和 jar 文件；

#### **2.2.2.3 scanDirLI 扫描系统目录**
扫描指定目录下的 apk 文件，最终调用 PackageParser.parseBaseApk 来完成 AndroidManifest.xml 文件的解析，生成 Application，activity，service，broadcast，provider 等信息。

- /vendor/overlay
- /system/framework
- /system/priv-app
- /system/app
- /vendor/app
- /oem/app


## 2.3 PMS_DATA_SCAN_START

接下来，进入第三阶段：

```java
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

                /**
                 * Remove disable package settings for any updated system
                 * apps that were removed via an OTA. If they're not a
                 * previously-updated app, remove them completely.
                 * Otherwise, just revoke their system-level permissions.
                 */
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

                // 确保所有在用户 data 分区的应用都显示出来了，如果无法显示
                // 那就回滚，显示 system 分区的！
                for (int i = 0; i < mExpectingBetter.size(); i++) {
                    final String packageName = mExpectingBetter.keyAt(i);
                    if (!mPackages.containsKey(packageName)) {
                        final File scanFile = mExpectingBetter.valueAt(i);

                        logCriticalInfo(Log.WARN, "Expected better " + packageName
                                + " but never showed up; reverting to system");
                        
                        // 设置扫描参数，不同的目录，扫描参数不同！
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
                        // 设置系统 package 可用！
                        mSettings.enableSystemPackageLPw(packageName);
                        // 重新解析 system 分区的 package！
                        try {
                            scanPackageTracedLI(scanFile, reparseFlags, scanFlags, 0, null);
                        } catch (PackageManagerException e) {
                            Slog.e(TAG, "Failed to parse original system package: "
                                    + e.getMessage());
                        }
                    }
                }
            }
            mExpectingBetter.clear(); // 清空 mExpectingBetter 列表！

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

            // 这里，我们已经找到了所有的共享库文件！
            // 我们需要更新所有的应用，保证他们有正确的共享库路径。
            updateAllSharedLibrariesLPw();

            for (SharedUserSetting setting : mSettings.getAllSharedUsersLPw()) {
                // NOTE: We ignore potential failures here during a system scan (like
                // the rest of the commands above) because there's precious little we
                // can do about it. A settings error is reported, though.
                adjustCpuAbisForSharedUserLPw(setting.packages, null /* scanned package */,
                        false /* boot complete */);
            }

            // 到这里，系统中所有的 package 都被扫描刀了，这里是更新他们上一次的使用信息！
            mPackageUsage.read(mPackages);
            mCompilerStats.read();
            
            ... ... ... ...// 见，第四阶段
```
当mOnlyCore = false时，则 scanDirLI 还会收集如下目录中的 apk 的信息！

- /data/app
- /data/app-private

### 2.3.1 主要流程

## 2.4 PMS_SCAN_END
下面是第四阶段：
```java
            ... ... ... ...// 第四阶段

            EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_PMS_SCAN_END,
                    SystemClock.uptimeMillis());
            Slog.i(TAG, "Time to scan packages: "
                    + ((SystemClock.uptimeMillis()-startTime)/1000f)
                    + " seconds");

            // If the platform SDK has changed since the last time we booted,
            // we need to re-grant app permission to catch any new ones that
            // appear.  This is really a hack, and means that apps can in some
            // cases get permissions that the user didn't initially explicitly
            // allow...  it would be nice to have some better way to handle
            // this situation.
            // 如果从我们上次启动，SDK 平台被改变了，我们需要重新授予应用程序权限。
            int updateFlags = UPDATE_PERMISSIONS_ALL;
            if (ver.sdkVersion != mSdkVersion) {
                Slog.i(TAG, "Platform changed from " + ver.sdkVersion + " to "
                        + mSdkVersion + "; regranting permissions for internal storage");
                updateFlags |= UPDATE_PERMISSIONS_REPLACE_PKG | UPDATE_PERMISSIONS_REPLACE_ALL;
            }
            
            // 赋予 package 相应请求的权限 
            updatePermissionsLPw(null, null, StorageManager.UUID_PRIVATE_INTERNAL, updateFlags);
            ver.sdkVersion = mSdkVersion;

            // If this is the first boot or an update from pre-M, and it is a normal
            // boot, then we need to initialize the default preferred apps across
            // all defined users.
            // 如果这是第一次开机或前-M的更新，这是一个正常的启动，然后我们需要初始化默认的首选应用程序给所有已经定义的用户。
            if (!onlyCore && (mPromoteSystemApps || mFirstBoot)) {
                for (UserInfo user : sUserManager.getUsers(true)) {
                    mSettings.applyDefaultPreferredAppsLPw(this, user.id);
                    applyFactoryDefaultBrowserLPw(user.id);
                    primeDomainVerificationsLPw(user.id);
                }
            }

            // Prepare storage for system user really early during boot,
            // since core system apps like SettingsProvider and SystemUI
            // can't wait for user to start
            final int storageFlags;
            if (StorageManager.isFileEncryptedNativeOrEmulated()) {
                storageFlags = StorageManager.FLAG_STORAGE_DE;
            } else {
                storageFlags = StorageManager.FLAG_STORAGE_DE | StorageManager.FLAG_STORAGE_CE;
            }
            reconcileAppsDataLI(StorageManager.UUID_PRIVATE_INTERNAL, UserHandle.USER_SYSTEM,
                    storageFlags);

            // If this is first boot after an OTA, and a normal boot, then
            // we need to clear code cache directories.
            // Note that we do *not* clear the application profiles. These remain valid
            // across OTAs and are used to drive profile verification (post OTA) and
            // profile compilation (without waiting to collect a fresh set of profiles).
            // 如果这是在OTA升级后第一启动，这是正常的启动，然后我们需要清除代码缓存目录。
            if (mIsUpgrade && !onlyCore) {
                Slog.i(TAG, "Build fingerprint changed; clearing code caches");
                for (int i = 0; i < mSettings.mPackages.size(); i++) {
                    final PackageSetting ps = mSettings.mPackages.valueAt(i);
                    if (Objects.equals(StorageManager.UUID_PRIVATE_INTERNAL, ps.volumeUuid)) {
                        // No apps are running this early, so no need to freeze
                        clearAppDataLIF(ps.pkg, UserHandle.USER_ALL,
                                StorageManager.FLAG_STORAGE_DE | StorageManager.FLAG_STORAGE_CE
                                        | Installer.FLAG_CLEAR_CODE_CACHE_ONLY);
                    }
                }
                ver.fingerprint = Build.FINGERPRINT;
            }

            checkDefaultBrowser();

            // clear only after permissions and other defaults have been updated
            // 当权限和其他默认设置被更新后，执行清除操作。
            mExistingSystemPackages.clear();
            mPromoteSystemApps = false;

            // All the changes are done during package scanning.
            ver.databaseVersion = Settings.CURRENT_DATABASE_VERSION;

            // can downgrade to reader
            // 信息写回 packages.xml 文件
            mSettings.writeLPr();

            // Perform dexopt on all apps that mark themselves as coreApps. We do this pretty
            // early on (before the package manager declares itself as early) because other
            // components in the system server might ask for package contexts for these apps.
            //
            // Note that "onlyCore" in this context means the system is encrypted or encrypting
            // (i.e, that the data partition is unavailable).
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
暂时先写到这里！

### 2.4.1 主要流程

## 2.5 PMS_READY
```java
            ... ... ... ...// 第五阶段
            EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_PMS_READY,
                    SystemClock.uptimeMillis());

            if (!mOnlyCore) {
                mRequiredVerifierPackage = getRequiredButNotReallyRequiredVerifierLPr();
                mRequiredInstallerPackage = getRequiredInstallerLPr();
                mRequiredUninstallerPackage = getRequiredUninstallerLPr();
                mIntentFilterVerifierComponent = getIntentFilterVerifierComponentNameLPr();
                mIntentFilterVerifier = new IntentVerifierProxy(mContext,
                        mIntentFilterVerifierComponent);
                mServicesSystemSharedLibraryPackageName = getRequiredSharedLibraryLPr(
                        PackageManager.SYSTEM_SHARED_LIBRARY_SERVICES);
                mSharedSystemSharedLibraryPackageName = getRequiredSharedLibraryLPr(
                        PackageManager.SYSTEM_SHARED_LIBRARY_SHARED);
            } else {
                mRequiredVerifierPackage = null;
                mRequiredInstallerPackage = null;
                mRequiredUninstallerPackage = null;
                mIntentFilterVerifierComponent = null;
                mIntentFilterVerifier = null;
                mServicesSystemSharedLibraryPackageName = null;
                mSharedSystemSharedLibraryPackageName = null;
            }
            
            // 建立 PackageInstallerService 服务对象
            mInstallerService = new PackageInstallerService(context, this);

            final ComponentName ephemeralResolverComponent = getEphemeralResolverLPr();
            final ComponentName ephemeralInstallerComponent = getEphemeralInstallerLPr();
            // both the installer and resolver must be present to enable ephemeral
            if (ephemeralInstallerComponent != null && ephemeralResolverComponent != null) {
                if (DEBUG_EPHEMERAL) {
                    Slog.i(TAG, "Ephemeral activated; resolver: " + ephemeralResolverComponent
                            + " installer:" + ephemeralInstallerComponent);
                }
                mEphemeralResolverComponent = ephemeralResolverComponent;
                mEphemeralInstallerComponent = ephemeralInstallerComponent;
                setUpEphemeralInstallerActivityLP(mEphemeralInstallerComponent);
                mEphemeralResolverConnection =
                        new EphemeralResolverConnection(mContext, mEphemeralResolverComponent);
            } else {
                if (DEBUG_EPHEMERAL) {
                    final String missingComponent =
                            (ephemeralResolverComponent == null)
                            ? (ephemeralInstallerComponent == null)
                                    ? "resolver and installer"
                                    : "resolver"
                            : "installer";
                    Slog.i(TAG, "Ephemeral deactivated; missing " + missingComponent);
                }
                mEphemeralResolverComponent = null;
                mEphemeralInstallerComponent = null;
                mEphemeralResolverConnection = null;
            }

            mEphemeralApplicationRegistry = new EphemeralApplicationRegistry(this);
        } // synchronized (mPackages)
        } // synchronized (mInstallLock)

        // Now after opening every single application zip, make sure they
        // are all flushed.  Not really needed, but keeps things nice and
        // tidy.
        Runtime.getRuntime().gc();

        // The initial scanning above does many calls into installd while
        // holding the mPackages lock, but we're mostly interested in yelling
        // once we have a booted system.
        mInstaller.setWarnIfHeld(mPackages);

        // Expose private service for system components to use.
        LocalServices.addService(PackageManagerInternal.class, new PackageManagerInternalImpl());
        
    }
```
PKMS 初始化完成阶段，还会创建一个 PackageInstaller 服务。


# 3 总结
PackageManagerService 的初始化工作都是在它的构造函数中完成的，主要完成一下任务：

1、  添加一些用户 id，如 system、phone 等；
2、  解析 /system/etc/permission 下的 xml 文件，主要 是 platform.xml，建立 permission 和 gid 之间的关系，可以指定一个权限与几个组对应，当一个 apk 被授予这个权限时它也同时属于这几个组，readPermission(parser， perm)；
        给一些底层用户分配一些权限，如 shell 授予各种 permission，把一个权限赋予一个 uid，当 apk 使用这个 uid 运行时，就具备了这个权限系统增加的一些应用需要 link 的扩展的 jar 库，系统每增加一个硬件，都要添加相应的 featrue，将解析结果放入 mAvailableFeatures;
3、  建立并启动 PackageHandler 消息循环，用于处理 apk 安装请求如 adb install，package installer 安装 apk 时就会发送消息；
4、  检查 /data/system/packages.xml 是否存在，里面记录了系统的 permission，以及每个apk的 name，codePath，flags，ts，version，userid 等，这些信息主要是通过 apk 安装的时候解析 AndroidManifest.xml 获取到的，解析完 apk 后将更新信息写入这个文件并保存到 flash，下次开机直接从里面读取相关信息添加到内存相关列表中，当有 apk 安装，升级，删除时会更新这个文件；
5、  检查 BootClassPath，mSharedLibraries 及 /system/framework 下的jar是否需要 dexopt ，需要则通过 dexopt 进行优化，这里面主要是调用 mInstaller.dexopt 进行相应的优化；
6、  建立 java 层的 installer 与 c 层的 installd 的 socket 联接，使得在上层的 install，remove，dexopt 等功能最终由 installd 在底层实现；
7、  启动 AppDirObserver 线程往中监测 /system/framework，/system/app，/data/app/data/app-private 目录的事件，主要监听 add 和 remove 事件，对于目录监听底层通过 inotify 机制实现，inotify 是一种文件系统的变化通知机制如文件增加、删除等事件可以立刻让用户态得知，它为用户态监视文件系统的变化提供了强大的支持，当有 add event 时调用 scanPackageLI(File，in，int) 处理，当有 remove event 时调用 removePackageLI 处理；
8、  调用 scanDirLI 启动 apk 解析，解析目录包括：/system/framework、/system/app、/vendor/app、/data/app、/data/app-private；
9、  移除临时文件；
10、 赋予 package 相应请求的权限；
11、 将解析出的 Package 的相关信息保存到相关全局变量，还有文件。


后面我们会分析每个阶段的主要流程，不要着急！























