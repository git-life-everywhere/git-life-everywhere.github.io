# PMS 第 2 篇 - PMS_START 阶段

title: PMS 第 2 篇 - PMS_START 阶段
date: 2018/01/26
categories:
- AndroidFramework源码分析
- PackageManager包管理
tags: PackageManager包管理
---

[toc]

基于 Android 7.1.1 源码分析 PackageManagerService 的架构和逻辑实现，本文是作者原创，转载请说明出处！

# 0 综述
我们进入第一阶段来分析，代码比较长，我们先来个总体的阶段回顾：

```java
       public PackageManagerService(Context context, Installer installer,
            boolean factoryTest, boolean onlyCore) {
        EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_PMS_START,
                SystemClock.uptimeMillis());

        if (mSdkVersion <= 0) {
            // mSdkVersion 是 PKMS 的成员变量，定义的时候进行赋值，
            // 其值取自系统属性 ro.build.version.sdk，即编译的 SDK 版本。
            Slog.w(TAG, "**** ro.build.version.sdk not set!");
        }

        mContext = context; // 系统进程的上下文实例！
        mFactoryTest = factoryTest; // 默认为 false，即运行在非工厂模式下
        mOnlyCore = onlyCore; // 默认为 false，标记是否是只加载核心服务
        
        // 用于存储与显示屏相关的一些属性，例如屏幕的宽 / 高尺寸，分辨率等信息。
        mMetrics = new DisplayMetrics(); 

        //【1】在 Settings 中，创建 packages.xml、packages-backup.xml、packages.list 等文件对象
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

        // 初始化 mInstaller 对象，installer 对象和 Native 进程 installd 交互，以后分析 installd 时再来讨论它的作用！
        mInstaller = installer;
        
        //【2】创建 PackageDexOptimizer 对象，用于 dex 优化！
        mPackageDexOptimizer = new PackageDexOptimizer(installer, mInstallLock, context,
                "*dexopt*");
                
        //【3】创建 MoveCallbacks 对象，用于操作回滚！
        mMoveCallbacks = new MoveCallbacks(FgThread.get().getLooper());
        
        //【4】创建 OnPermissionChangeListeners 对象，用于监听权限改变！
        mOnPermissionChangeListeners = new OnPermissionChangeListeners(
                FgThread.get().getLooper());

        getDefaultDisplayMetrics(context, mMetrics);

        //【5】创建 SystemConfig 对象，用于获取系统配置信息
        // 主要有 /system/etc/ 目录和 /system/etc/ 目录下的 sysconfig 和 permissions 文件夹中的 xml 文件
        SystemConfig systemConfig = SystemConfig.getInstance();
        mGlobalGids = systemConfig.getGlobalGids();
        mSystemPermissions = systemConfig.getSystemPermissions();
        mAvailableFeatures = systemConfig.getAvailableFeatures();

        mProtectedPackages = new ProtectedPackages(mContext);

        synchronized (mInstallLock) {
        // writer
        synchronized (mPackages) {
            //【6】建立并启动一个名为 “PackageManager” 的 HandlerThread，类型是 ServiceThread，处理安装卸载的消息！
            mHandlerThread = new ServiceThread(TAG,
                    Process.THREAD_PRIORITY_BACKGROUND, true /*allowIo*/);
            mHandlerThread.start();
            
            //【7】创建一个 PackageHandler 对象，绑定前面创建的 HandlerThread！
            mHandler = new PackageHandler(mHandlerThread.getLooper());

            mProcessLoggingHandler = new ProcessLoggingHandler();
            Watchdog.getInstance().addThread(mHandler, WATCHDOG_TIMEOUT);
            
            //【8】创建默认权限管理对象，用于给某些预制的 apk 给予权限，也可以撤销！
            mDefaultPermissionPolicy = new DefaultPermissionGrantPolicy(this);
            
            // 创建需要扫描检测的目录文件对象，为扫描做准备！
            File dataDir = Environment.getDataDirectory();
            mAppInstallDir = new File(dataDir, "app"); // /data/data，存放应用程序的数据
            mAppLib32InstallDir = new File(dataDir, "app-lib");  // /data/app-lib，用于存放用户自己安装的第三方应用程序
            mEphemeralInstallDir = new File(dataDir, "app-ephemeral");  // /data/app-ephemeral
            mAsecInternalPath = new File(dataDir, "app-asec").getPath();  // /data/app-asec
            mDrmAppPrivateInstallDir = new File(dataDir, "app-private");  // /data/app-private，存放私有应用程序！

            //【9】创建用户管理服务！    
            sUserManager = new UserManagerService(context, this, mPackages);

            //【10】将前面 SystemConfig 获得的系统权限保存到 mSettings.mPermissions 中！
            ArrayMap<String, SystemConfig.PermissionEntry> permConfig
                    = systemConfig.getPermissions();
            for (int i=0; i<permConfig.size(); i++) {
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

            //【11】处理共享库！
            ArrayMap<String, String> libConfig = systemConfig.getSharedLibraries();
            for (int i=0; i<libConfig.size(); i++) {
                mSharedLibraries.put(libConfig.keyAt(i),
                        new SharedLibraryEntry(libConfig.valueAt(i), null));
            }
            
            //【12】从 /etc/security/mac_permissions.xml 读取 SELinux 信息！
            mFoundPolicyFile = SELinuxMMAC.readInstallPolicy();
            
            //【13】解析 packages.xml 文件，获得上一次的安装信息。返回值表示是否是第一次开机！
            // 如果是第一次开机，那么是读不到上次的数据的！
            mFirstBoot = !mSettings.readLPw(sUserManager.getUsers(false));

            //【14】如果应用是被更新过的系统应用，且位于 data 分区的 apk 文件不存在了
            // 那就移除上一次的 data 分区的安装信息，恢复为 system 分区的安装信息！
            // 这里貌似是为了解决一个 bug:32321269!
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
                //【15】如果是第一次开机，从另外一个系统拷贝 odex 文件到当前系统的 data 分区
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
下面我们来逐一分析这个阶段的过程：

# 1 Settings
首先，创建了 Settings 对象：！

参数传递：

- Object lock：mPackage 对象！

```java
    Settings(Object lock) {
        // 调用第二个构造器！
        this(Environment.getDataDirectory(), lock);
    }
```
接着调用第二个构造函数：
```java
    Settings(File dataDir, Object lock) {
        mLock = lock;
        // 用于处理运行时权限相关的操作！
        mRuntimePermissionsPersistence = new RuntimePermissionPersistence(mLock);
        
        // 创建 /data/system 目录，并设置该目录的权限！
        mSystemDir = new File(dataDir, "system");
        mSystemDir.mkdirs();
        FileUtils.setPermissions(mSystemDir.toString(),
                FileUtils.S_IRWXU|FileUtils.S_IRWXG
                |FileUtils.S_IROTH|FileUtils.S_IXOTH,
                -1, -1);
        
        // 创建 packages.xml 和 packages-backup.xml 文件对象
        mSettingsFilename = new File(mSystemDir, "packages.xml");
        mBackupSettingsFilename = new File(mSystemDir, "packages-backup.xml");

        // 创建 packages.list 文件对象，并设置权限信息！
        mPackageListFilename = new File(mSystemDir, "packages.list");
        FileUtils.setPermissions(mPackageListFilename, 0640, SYSTEM_UID, PACKAGE_INFO_GID);

        // 创建 sdcardfs 文件对象！
        final File kernelDir = new File("/config/sdcardfs");
        mKernelMappingFilename = kernelDir.exists() ? kernelDir : null;

        // 创建 packages-stopped.xml 和 packages-stopped-backup.xml 文件对象！
        mStoppedPackagesFilename = new File(mSystemDir, "packages-stopped.xml");
        mBackupStoppedPackagesFilename = new File(mSystemDir, "packages-stopped-backup.xml");
    }
```
Settings 对象的构造器很简单，这里要重点说明一下；

- packages.xml 和 packages-backup.xml 文件是一组，用于描述系统中所安装的所有 Package 信息，PMS 会先把数据写入 packages-backup.xml，信息写成功后，再改名为 packages.xml ！
- packages.list 文件同样保存了系统中所有的应用的信息！
- packages-stopped.xml 和 packages-stopped-backup.xml 文件是一组，用于描述被强行停止运行的 package 信息！

```java
com.android.settings 1000 0 /data/user_de/0/com.android.settings platform:privapp 2001,3009,3002,1023,1015,3003,3001,1004,2002,1007,3006,3188
```
## 1.1 new RuntimePermissionPersistence

创建运行时权限管理对象！


## 1.2 Settings.addSharedUserLPw
接着调用了 **addSharedUserLPw** 方法，添加了几个系统共享用户，参数传递：

- **String name：**共享用户名，下面是传入的用户名：
    - android.uid.system
    - android.uid.phone
    - android.uid.log
    - android.uid.nfc
    - android.uid.bluetooth
    - android.uid.shell
- **int uid：**共享用户 id，下面的用户名和上面的 id 一一对应！
    - Process.SYSTEM_UID
    - RADIO_UID
    - LOG_UID
    - NFC_UID
    - BLUETOOTH_UID
    - SHELL_UID
- **int pkgFlags：**
    - ApplicationInfo.FLAG_SYSTEM 
- **int pkgPrivateFlags：**
    - ApplicationInfo.PRIVATE_FLAG_PRIVILEGED
                
                
```java
    SharedUserSetting addSharedUserLPw(String name, int uid, int pkgFlags, int pkgPrivateFlags) {
        SharedUserSetting s = mSharedUsers.get(name);
        if (s != null) {
            if (s.userId == uid) {
                return s;
            }
            PackageManagerService.reportSettingsProblem(Log.ERROR,
                    "Adding duplicate shared user, keeping first: " + name);
            return null;
        }

        //【1.1.1】创建 SharedUserSetting 对象，封装 SharedUserId！
        s = new SharedUserSetting(name, pkgFlags, pkgPrivateFlags);
        s.userId = uid;
        
        //【1.1.2】根据 uid 的范围，保存到到 mUserIds 或 mOtherUserIds 中！
        if (addUserIdLPw(uid, s, name)) {
            //【1.1.2.1】添加到 mSharedUsers 中！
            mSharedUsers.put(name, s);
            return s;
        }
        return null;
    }
```

### 1.2.1 SharedUserSetting
我们来看看 SharedUserSetting 这个对象：

```java
final class SharedUserSetting extends SettingBase {
    // 共享 uid 的名称
    final String name;
    // uid 值
    int userId;
    // 和这个 sharedUserId 相关的 flags
    int uidFlags; // 取值为 ApplicationInfo.FLAG_SYSTEM
    int uidPrivateFlags; // 取值为 ApplicationInfo.FLAG_SYSTEM

    // 使用这个共享 uid 的所有应用程序 package！
    final ArraySet<PackageSetting> packages = new ArraySet<PackageSetting>();

    final PackageSignatures signatures = new PackageSignatures();

    SharedUserSetting(String _name, int _pkgFlags, int _pkgPrivateFlags) {
        super(_pkgFlags, _pkgPrivateFlags);
        uidFlags =  _pkgFlags;
        uidPrivateFlags = _pkgPrivateFlags;
        name = _name;
    }

    @Override
    public String toString() {
        return "SharedUserSetting{" + Integer.toHexString(System.identityHashCode(this)) + " "
                + name + "/" + userId + "}";
    }
    
    ... ... ...
}

```
代码很简单，这里就不多说了！

### 1.2.2 Settings.addUserIdLPw

addUserIdLPw 用于将创建的 sharedUserId 对象根据其 uid 是系统 uid 还是非系统 uid ，添加到指定的集合中！

```java
    private boolean addUserIdLPw(int uid, Object obj, Object name) {
        // uid 不能超过限制！
        if (uid > Process.LAST_APPLICATION_UID) {
            return false;
        }

        //【1】如果 uid 属于非系统应用，将其加入到 mUserIds 集合中！
        if (uid >= Process.FIRST_APPLICATION_UID) {
            int N = mUserIds.size();
            final int index = uid - Process.FIRST_APPLICATION_UID;
            while (index >= N) {
                mUserIds.add(null);
                N++;
            }
            if (mUserIds.get(index) != null) {
                PackageManagerService.reportSettingsProblem(Log.ERROR,
                        "Adding duplicate user id: " + uid
                        + " name=" + name);
                return false;
            }
            mUserIds.set(index, obj);
        } else {
        
            //【2】如果 uid 属于系统应用，将其加入到 mOtherUserIds 集合中！
            if (mOtherUserIds.get(uid) != null) {
                PackageManagerService.reportSettingsProblem(Log.ERROR,
                        "Adding duplicate shared id: " + uid
                                + " name=" + name);
                return false;
            }
            mOtherUserIds.put(uid, obj);
        }
        return true;
    }

```
这里来说明一下：

> 涉及的常量

- Process.FIRST_APPLICATION_UID：取值 10000，用来区分系统应用 uid 和非系统应用的 uid。非系统应用的 uid 大于等于 10000，小于等于 19999，系统应用的 uid 小于 10000；
- Process.LAST_APPLICATION_UID：取值 19999，表示应用程序的 uid 最大为 19999！


> 总结

- mSharedUsers 集合用来保存所有的共享用户id！
- mUserIds 集合中会保存非系统应用的 uid，包括共享和非共享！
- mOtherUserIds 集合中会保存系统应用的 uid，包括共享和非共享！


# 2 PackageDexOptimizer 

PackageDexOptimizer 用于进行 odex 优化，我们来先简单的看看代码，参数传递：

- Installer installer：installer 对象，用于安装程序；
- Object installLock：Object() 锁对象！
- Context context：系统进程上下文实例！
- String wakeLockTag：传入"*dexopt*"

```java
class PackageDexOptimizer {
    private static final String TAG = "PackageManager.DexOptimizer";
    static final String OAT_DIR_NAME = "oat";
    // TODO b/19550105 Remove error codes and use exceptions
    static final int DEX_OPT_SKIPPED = 0;
    static final int DEX_OPT_PERFORMED = 1;
    static final int DEX_OPT_FAILED = -1;

    private final Installer mInstaller;
    private final Object mInstallLock;

    private final PowerManager.WakeLock mDexoptWakeLock;
    private volatile boolean mSystemReady;

    // 调用的这个构造器！
    PackageDexOptimizer(Installer installer, Object installLock, Context context,
            String wakeLockTag) {
        this.mInstaller = installer;
        this.mInstallLock = installLock;

        PowerManager powerManager = (PowerManager)context.getSystemService(Context.POWER_SERVICE);
        mDexoptWakeLock = powerManager.newWakeLock(PowerManager.PARTIAL_WAKE_LOCK, wakeLockTag);
    }

    protected PackageDexOptimizer(PackageDexOptimizer from) {
        this.mInstaller = from.mInstaller;
        this.mInstallLock = from.mInstallLock;
        this.mDexoptWakeLock = from.mDexoptWakeLock;
        this.mSystemReady = from.mSystemReady;
    }

    ... ... ... ...
}
```

这里创建了 PackageDexOptimizer 对象，后续执行 odex 优化会用到！

# 3 MoveCallbacks  

用于监听 package 的 move 操作！！

```java
    private static class MoveCallbacks extends Handler {
        private static final int MSG_CREATED = 1;
        private static final int MSG_STATUS_CHANGED = 2;

        private final RemoteCallbackList<IPackageMoveObserver>
                mCallbacks = new RemoteCallbackList<>();

        private final SparseIntArray mLastStatus = new SparseIntArray();

        public MoveCallbacks(Looper looper) {
            super(looper);
        }

        public void register(IPackageMoveObserver callback) {
            mCallbacks.register(callback);
        }

        public void unregister(IPackageMoveObserver callback) {
            mCallbacks.unregister(callback);
        }

        @Override
        public void handleMessage(Message msg) {
            final SomeArgs args = (SomeArgs) msg.obj;
            final int n = mCallbacks.beginBroadcast();
            for (int i = 0; i < n; i++) {
                final IPackageMoveObserver callback = mCallbacks.getBroadcastItem(i);
                try {
                    invokeCallback(callback, msg.what, args);
                } catch (RemoteException ignored) {
                }
            }
            mCallbacks.finishBroadcast();
            args.recycle();
        }

        private void invokeCallback(IPackageMoveObserver callback, int what, SomeArgs args)
                throws RemoteException {
            switch (what) {
                case MSG_CREATED: {
                    callback.onCreated(args.argi1, (Bundle) args.arg2);
                    break;
                }
                case MSG_STATUS_CHANGED: {
                    callback.onStatusChanged(args.argi1, args.argi2, (long) args.arg3);
                    break;
                }
            }
        }

        private void notifyCreated(int moveId, Bundle extras) {
            Slog.v(TAG, "Move " + moveId + " created " + extras.toString());

            final SomeArgs args = SomeArgs.obtain();
            args.argi1 = moveId;
            args.arg2 = extras;
            obtainMessage(MSG_CREATED, args).sendToTarget();
        }

        private void notifyStatusChanged(int moveId, int status) {
            notifyStatusChanged(moveId, status, -1);
        }

        private void notifyStatusChanged(int moveId, int status, long estMillis) {
            Slog.v(TAG, "Move " + moveId + " status " + status);

            final SomeArgs args = SomeArgs.obtain();
            args.argi1 = moveId;
            args.argi2 = status;
            args.arg3 = estMillis;
            obtainMessage(MSG_STATUS_CHANGED, args).sendToTarget();

            synchronized (mLastStatus) {
                mLastStatus.put(moveId, status);
            }
        }
    }
```


# 4 OnPermissionChangeListeners
创建 OnPermissionChangeListeners 对象！
```java
        // 创建 OnPermissionChangeListeners 对象，用于监听权限改变！
        mOnPermissionChangeListeners = new OnPermissionChangeListeners(
                FgThread.get().getLooper());
```
OnPermissionChangeListeners 用于监听权限的改变：

```java
    private final static class OnPermissionChangeListeners extends Handler {
        private static final int MSG_ON_PERMISSIONS_CHANGED = 1;

        private final RemoteCallbackList<IOnPermissionsChangeListener> mPermissionListeners =
                new RemoteCallbackList<>();

        public OnPermissionChangeListeners(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                //【2】处理 MSG_ON_PERMISSIONS_CHANGED 消息！
                case MSG_ON_PERMISSIONS_CHANGED: {
                    final int uid = msg.arg1;
                    handleOnPermissionsChanged(uid);
                } break;
            }
        }

        // 添加权限改变监听器！
        public void addListenerLocked(IOnPermissionsChangeListener listener) {
            mPermissionListeners.register(listener);

        }
        
        // 移除权限改变监听器！
        public void removeListenerLocked(IOnPermissionsChangeListener listener) {
            mPermissionListeners.unregister(listener);
        }

        public void onPermissionsChanged(int uid) {
            //【1】uid 的权限改变后，会触发这个方法，发送 MSG_ON_PERMISSIONS_CHANGED 消息！
            if (mPermissionListeners.getRegisteredCallbackCount() > 0) {
                obtainMessage(MSG_ON_PERMISSIONS_CHANGED, uid, 0).sendToTarget();
            }
        }

        //【3】uid 对应的应用程序的权限发生了变化！
        private void handleOnPermissionsChanged(int uid) {
            final int count = mPermissionListeners.beginBroadcast();
            try {
                for (int i = 0; i < count; i++) {
                    IOnPermissionsChangeListener callback = mPermissionListeners
                            .getBroadcastItem(i);
                    try {
                        callback.onPermissionsChanged(uid);
                    } catch (RemoteException e) {
                        Log.e(TAG, "Permission listener is dead", e);
                    }
                }
            } finally {
                mPermissionListeners.finishBroadcast();
            }
        }
    }
```
当指定 uid 的权限发生变化后，会调用 IOnPermissionsChangeListener 的 onPermissionsChanged 方法，通过 Binder 机制，最终会调用 ApplicationPackageManager 的 OnPermissionsChangeListenerDelegate 中的 onPermissionsChanged 方法！
```java
    public class OnPermissionsChangeListenerDelegate extends IOnPermissionsChangeListener.Stub
            implements Handler.Callback{
        private static final int MSG_PERMISSIONS_CHANGED = 1;

        private final OnPermissionsChangedListener mListener;
        private final Handler mHandler;

        public OnPermissionsChangeListenerDelegate(OnPermissionsChangedListener listener,
                Looper looper) {
            mListener = listener;
            mHandler = new Handler(looper, this);
        }

        @Override
        public void onPermissionsChanged(int uid) {
            mHandler.obtainMessage(MSG_PERMISSIONS_CHANGED, uid, 0).sendToTarget();
        }

        @Override
        public boolean handleMessage(Message msg) {
            switch (msg.what) {
                case MSG_PERMISSIONS_CHANGED: {
                    final int uid = msg.arg1;
                    
                    // 最后调用 OnPermissionsChangedListener 的 onPermissionsChanged 方法！
                    mListener.onPermissionsChanged(uid);
                    return true;
                }
            }
            return false;
        }
    }

```

OnPermissionsChangedListener 是一个接口，定义在 PackageManager 中：
```java
    @SystemApi
    public interface OnPermissionsChangedListener {

        /**
         * Called when the permissions for a UID change.
         * @param uid The UID with a change.
         */
        public void onPermissionsChanged(int uid);
    }
```

具体的实现是在 LocationManagerService 中：
```java
    public void systemRunning() {
        synchronized (mLock) {
            if (D) Log.d(TAG, "systemRunning()");

            ... ... ... ...

            // 这里创建了一个 OnPermissionsChangedListener 对象，并添加到 PackageManagerervice 中！
            PackageManager.OnPermissionsChangedListener permissionListener
                    = new PackageManager.OnPermissionsChangedListener() {
                @Override
                public void onPermissionsChanged(final int uid) {
                    synchronized (mLock) {
                        applyAllProviderRequirementsLocked();
                    }
                }
            };
            mPackageManager.addOnPermissionsChangeListener(permissionListener);

            ... ... ...
            
        }

```

进入 PackageManagerService 中：
```java
    @Override
    public void addOnPermissionsChangeListener(IOnPermissionsChangeListener listener) {
        mContext.enforceCallingOrSelfPermission(
                Manifest.permission.OBSERVE_GRANT_REVOKE_PERMISSIONS,
                "addOnPermissionsChangeListener");

        synchronized (mPackages) {
            
            // 添加到 mOnPermissionChangeListeners 内部的集合中！
            mOnPermissionChangeListeners.addListenerLocked(listener);
        }
    }
```
这里先看到这里吧！


# 5 SystemConfig

SystemConfig 是用来存储系统全局的配置信息的数据结构，有系统配置信息，权限信息，Gid 等等！
```java
    // 单例模式！
    public static SystemConfig getInstance() {
        synchronized (SystemConfig.class) {
            if (sInstance == null) {
                sInstance = new SystemConfig();
            }
            return sInstance;
        }
    }
```
下面我们来看看 SystemConfig 构造器中的相关方法：
```java
    // 其构造器会调用方法读取系统配置信息！
    SystemConfig() {
        // 读取配置信息：/etc/sysconfig
        readPermissions(Environment.buildPath(
                Environment.getRootDirectory(), "etc", "sysconfig"), ALLOW_ALL);
                
        // 读取配置信息：/etc/permissions
        readPermissions(Environment.buildPath(
                Environment.getRootDirectory(), "etc", "permissions"), ALLOW_ALL);
                
        // 读取配置信息：/odm/etc/sysconfig，/odm/etc/permissions
        int odmPermissionFlag = ALLOW_LIBS | ALLOW_FEATURES | ALLOW_APP_CONFIGS;
        readPermissions(Environment.buildPath(
                Environment.getOdmDirectory(), "etc", "sysconfig"), odmPermissionFlag);
        readPermissions(Environment.buildPath(
                Environment.getOdmDirectory(), "etc", "permissions"), odmPermissionFlag);
                
        // 读取配置信息：/oem/etc/sysconfig，/oem/etc/permissions
        readPermissions(Environment.buildPath(
                Environment.getOemDirectory(), "etc", "sysconfig"), ALLOW_FEATURES);
        readPermissions(Environment.buildPath(
                Environment.getOemDirectory(), "etc", "permissions"), ALLOW_FEATURES);
    }

```
第一个参数，表示可以访问的目录，这些目录包括：

- /etc/sysconfig：ALLOW_ALL
- /etc/permissions：ALLOW_ALL
- /odm/etc/sysconfig：ALLOW_LIBS | ALLOW_FEATURES | ALLOW_APP_CONFIGS
- /odm/etc/permissions：ALLOW_LIBS | ALLOW_FEATURES | ALLOW_APP_CONFIGS
- /oem/etc/sysconfig：ALLOW_FEATURES
- /oem/etc/permissions：ALLOW_FEATURES

第二个参数，表示这个目录可以设置的配置信息的类型，同样的，PMS 在解析时，也只会解析和读取参数指定的配置属性：

- oem：只能设置 features！
- odm：只能设置 libs，features 和 app configs！
- /system/etc/sysconfig，/system/etc/permissions：可以设置所有的配置，libs，permissions，features，app configs 等等！

下面是 flag 的取值：
```java
    private static final int ALLOW_FEATURES = 0x01;
    private static final int ALLOW_LIBS = 0x02;
    private static final int ALLOW_PERMISSIONS = 0x04;
    private static final int ALLOW_APP_CONFIGS = 0x08;
    private static final int ALLOW_ALL = ~0;
```
继续看！

## 2.1 SystemConfig.readPermissions

我们来看看读取的过程：
```java
    void readPermissions(File libraryDir, int permissionFlag) {
        // 校验目录可读性！
        if (!libraryDir.exists() || !libraryDir.isDirectory()) {
            if (permissionFlag == ALLOW_ALL) {
                Slog.w(TAG, "No directory " + libraryDir + ", skipping");
            }
            return;
        }
        if (!libraryDir.canRead()) {
            Slog.w(TAG, "Directory " + libraryDir + " cannot be read");
            return;
        }

        // 遍历解析目录下的所有 .xml 文件！
        File platformFile = null;
        for (File f : libraryDir.listFiles()) {
            //【1】对于 etc/permissions/platform.xml 文件，最后再处理！
            if (f.getPath().endsWith("etc/permissions/platform.xml")) {
                platformFile = f;
                continue;
            }

            if (!f.getPath().endsWith(".xml")) {
                Slog.i(TAG, "Non-xml file " + f + " in " + libraryDir + " directory, ignoring");
                continue;
            }
            if (!f.canRead()) {
                Slog.w(TAG, "Permissions library file " + f + " cannot be read");
                continue;
            }
            
            //【2】进一步解析文件！
            readPermissionsFromXml(f, permissionFlag);
        }

        //【3】读取和解析 platform.xml 文件！
        if (platformFile != null) {
            readPermissionsFromXml(platformFile, permissionFlag);
        }
    }
```
这个方法的主要过程是：

- 先解析该目录下的其他的 xml 文件；
- 最后解析 etc/permissions/platform.xml 文件（如果有）；


## 2.2 SystemConfig.readPermissionsFromXml
继续调用 readPermissionsFromXml 方法，这里我们重点看一些 tag：
```java
    private void readPermissionsFromXml(File permFile, int permissionFlag) {
        FileReader permReader = null;
        try {
            permReader = new FileReader(permFile);
        } catch (FileNotFoundException e) {
            Slog.w(TAG, "Couldn't find or open permissions file " + permFile);
            return;
        }

        //【1】判读设备是否是低内存！
        final boolean lowRam = ActivityManager.isLowRamDeviceStatic();

        try {
            XmlPullParser parser = Xml.newPullParser();
            parser.setInput(permReader);

            int type;
            while ((type=parser.next()) != parser.START_TAG
                       && type != parser.END_DOCUMENT) {
                ;
            }

            if (type != parser.START_TAG) {
                throw new XmlPullParserException("No start tag found");
            }

            if (!parser.getName().equals("permissions") && !parser.getName().equals("config")) {
                throw new XmlPullParserException("Unexpected start tag in " + permFile
                        + ": found " + parser.getName() + ", expected 'permissions' or 'config'");
            }

            //【2】解析 Flag，指定二进制位为 1 ，即表示该条件满足，如果 flag 是 ALLOW_ALL，
            // 那么其他的条件均满足！
            boolean allowAll = permissionFlag == ALLOW_ALL;
            boolean allowLibs = (permissionFlag & ALLOW_LIBS) != 0;
            boolean allowFeatures = (permissionFlag & ALLOW_FEATURES) != 0;
            boolean allowPermissions = (permissionFlag & ALLOW_PERMISSIONS) != 0;
            boolean allowAppConfigs = (permissionFlag & ALLOW_APP_CONFIGS) != 0;

            while (true) {
                // 解析下一个标签！
                XmlUtils.nextElement(parser);
                if (parser.getEventType() == XmlPullParser.END_DOCUMENT) {
                    break;
                }

                String name = parser.getName();
                if ("group".equals(name) && allowAll) { 
                    //【1】allowAll 为 true，解析 "group" 标签；
                    // 解析系统中所有的 Gid 信息！
                    String gidStr = parser.getAttributeValue(null, "gid");
                    if (gidStr != null) {
                        int gid = android.os.Process.getGidForName(gidStr);
                        mGlobalGids = appendInt(mGlobalGids, gid);
                    } else {
                        Slog.w(TAG, "<group> without gid in " + permFile + " at "
                                + parser.getPositionDescription());
                    }

                    XmlUtils.skipCurrentTag(parser);
                    continue;
                    
                } else if ("permission".equals(name) && allowPermissions) {  
                    //【2】allowPermissions 为 true，解析 "permission" 标签
                    // 解析系统中所有权限和其所属 gid 的信息！
                    String perm = parser.getAttributeValue(null, "name");
                    if (perm == null) {
                        Slog.w(TAG, "<permission> without name in " + permFile + " at "
                                + parser.getPositionDescription());
                        XmlUtils.skipCurrentTag(parser);
                        continue;
                    }
                    perm = perm.intern();
                    readPermission(parser, perm); // 调用 readPermission 继续解析权限

                } else if ("assign-permission".equals(name) && allowPermissions) { 
                    //【3】如果 allowPermissions 为 true，解析 "assign-permission" 标签
                    // 解析系统中 uid 和其所持有的权限的关系！
                    String perm = parser.getAttributeValue(null, "name"); // 权限名
                    if (perm == null) {
                        Slog.w(TAG, "<assign-permission> without name in " + permFile + " at "
                                + parser.getPositionDescription());
                        XmlUtils.skipCurrentTag(parser);
                        continue;
                    }
                    String uidStr = parser.getAttributeValue(null, "uid"); // 拥有该权限的 uid
                    if (uidStr == null) {
                        Slog.w(TAG, "<assign-permission> without uid in " + permFile + " at "
                                + parser.getPositionDescription());
                        XmlUtils.skipCurrentTag(parser);
                        continue;
                    }
                    int uid = Process.getUidForName(uidStr); // 将 uid 转为 int 值！
                    if (uid < 0) {
                        Slog.w(TAG, "<assign-permission> with unknown uid \""
                                + uidStr + "  in " + permFile + " at "
                                + parser.getPositionDescription());
                        XmlUtils.skipCurrentTag(parser);
                        continue;
                    }
                    perm = perm.intern();
                    ArraySet<String> perms = mSystemPermissions.get(uid);
                    if (perms == null) {
                        perms = new ArraySet<String>();
                        mSystemPermissions.put(uid, perms);
                    }
                    perms.add(perm);
                    XmlUtils.skipCurrentTag(parser);

                } else if ("library".equals(name) && allowLibs) { 
                    //【4】如果 allowLibs 为 true，解析 "library" 标签
                    // 获得系统中所有共享库的信息！
                    String lname = parser.getAttributeValue(null, "name");
                    String lfile = parser.getAttributeValue(null, "file");
                    if (lname == null) {
                        Slog.w(TAG, "<library> without name in " + permFile + " at "
                                + parser.getPositionDescription());
                    } else if (lfile == null) {
                        Slog.w(TAG, "<library> without file in " + permFile + " at "
                                + parser.getPositionDescription());
                    } else {
                        //Log.i(TAG, "Got library " + lname + " in " + lfile);
                        mSharedLibraries.put(lname, lfile);
                    }
                    XmlUtils.skipCurrentTag(parser);
                    continue;

                } else if ("feature".equals(name) && allowFeatures) { 
                    //【5】如果 allowFeatures 为 true，解析 "feature" 标签
                    // 获得系统中所有可用特性的信息！
                    String fname = parser.getAttributeValue(null, "name");
                    int fversion = XmlUtils.readIntAttribute(parser, "version", 0);
                    boolean allowed;
                    
                    if (!lowRam) {
                        allowed = true;
                    } else {
                        // 内存不足时，配置了 notLowRam 的 feature 不会在加载！
                        String notLowRam = parser.getAttributeValue(null, "notLowRam");
                        allowed = !"true".equals(notLowRam);
                    }
                    if (fname == null) {
                        Slog.w(TAG, "<feature> without name in " + permFile + " at "
                                + parser.getPositionDescription());
                    } else if (allowed) {
                        // 将 feature 构造成 featureInfo，加入到 mAvailableFeatures 对象中！
                        addFeature(fname, fversion);
                    }
                    XmlUtils.skipCurrentTag(parser);
                    continue;

                } else if ("unavailable-feature".equals(name) && allowFeatures) { 
                    //【6】如果 allowFeatures 为 true，解析 "unavailable-feature" 标签
                    // 获得系统中所有不可用特性的信息！
                    String fname = parser.getAttributeValue(null, "name");
                    if (fname == null) {
                        Slog.w(TAG, "<unavailable-feature> without name in " + permFile + " at "
                                + parser.getPositionDescription());
                    } else {
                        mUnavailableFeatures.add(fname);
                    }
                    XmlUtils.skipCurrentTag(parser);
                    continue;

                } else if ("allow-in-power-save-except-idle".equals(name) && allowAll) {
                    //【7】解析 "allow-in-power-save-except-idle" 标签
                    // 用于获得系统中处于省电模式白名单，而没有在 idle 模式白名单中的应用信息！
                    // 这些应用可以在后台运行！
                    String pkgname = parser.getAttributeValue(null, "package");
                    if (pkgname == null) {
                        Slog.w(TAG, "<allow-in-power-save-except-idle> without package in "
                                + permFile + " at " + parser.getPositionDescription());
                    } else {
                        mAllowInPowerSaveExceptIdle.add(pkgname);
                    }
                    XmlUtils.skipCurrentTag(parser);
                    continue;

                } else if ("allow-in-power-save".equals(name) && allowAll) { 
                    //【8】解析 "allow-in-power-save" 标签
                    // 获得哪些处于 doze 模式白名单中的应用信息！
                    String pkgname = parser.getAttributeValue(null, "package");
                    if (pkgname == null) {
                        Slog.w(TAG, "<allow-in-power-save> without package in " + permFile + " at "
                                + parser.getPositionDescription());
                    } else {
                        mAllowInPowerSave.add(pkgname);
                    }
                    XmlUtils.skipCurrentTag(parser);
                    continue;

                } else if ("allow-in-data-usage-save".equals(name) && allowAll) {  
                    //【9】解析 "allow-in-data-usage-save" 标签，
                    // 获得哪些处于流量节省模式白名单中的应用的信息，在白名单中的应用，当系统处于
                    // 流量节省模式时，可以在后台运行！
                    String pkgname = parser.getAttributeValue(null, "package");
                    if (pkgname == null) {
                        Slog.w(TAG, "<allow-in-data-usage-save> without package in " + permFile
                                + " at " + parser.getPositionDescription());
                    } else {
                        mAllowInDataUsageSave.add(pkgname);
                    }
                    XmlUtils.skipCurrentTag(parser);
                    continue;

                } else if ("app-link".equals(name) && allowAppConfigs) {  
                    //【10】解析 "app-link" 标签
                    String pkgname = parser.getAttributeValue(null, "package");
                    if (pkgname == null) {
                        Slog.w(TAG, "<app-link> without package in " + permFile + " at "
                                + parser.getPositionDescription());
                    } else {
                        mLinkedApps.add(pkgname);
                    }
                    XmlUtils.skipCurrentTag(parser);

                } else if ("system-user-whitelisted-app".equals(name) && allowAppConfigs) { 
                    //【11】解析 "system-user-whitelisted-app" 标签
                    // 获得哪些在 system user 下可以运行的应用信息！
                    String pkgname = parser.getAttributeValue(null, "package");
                    if (pkgname == null) {
                        Slog.w(TAG, "<system-user-whitelisted-app> without package in "
                        + permFile + " at " + parser.getPositionDescription());
                    } else {
                        mSystemUserWhitelistedApps.add(pkgname);
                    }
                    XmlUtils.skipCurrentTag(parser);

                } else if ("system-user-blacklisted-app".equals(name) && allowAppConfigs) { 
                    //【12】解析 "system-user-blacklisted-app" 标签
                    // 获得哪些在 system user 下不可以运行的应用信息！
                    String pkgname = parser.getAttributeValue(null, "package");
                    if (pkgname == null) {
                        Slog.w(TAG, "<system-user-blacklisted-app without package in " + permFile
                                + " at " + parser.getPositionDescription());
                    } else {
                        mSystemUserBlacklistedApps.add(pkgname);
                    }
                    XmlUtils.skipCurrentTag(parser);

                } else if ("default-enabled-vr-app".equals(name) && allowAppConfigs) { 
                    //【13】解析 "default-enabled-vr-app" 标签
                    String pkgname = parser.getAttributeValue(null, "package");
                    String clsname = parser.getAttributeValue(null, "class");
                    if (pkgname == null) {
                        Slog.w(TAG, "<default-enabled-vr-app without package in " + permFile
                                + " at " + parser.getPositionDescription());
                    } else if (clsname == null) {
                        Slog.w(TAG, "<default-enabled-vr-app without class in " + permFile
                                + " at " + parser.getPositionDescription());
                    } else {
                        mDefaultVrComponents.add(new ComponentName(pkgname, clsname));
                    }
                    XmlUtils.skipCurrentTag(parser);

                } else if ("backup-transport-whitelisted-service".equals(name) && allowFeatures) {  
                    //【14】解析 "backup-transport-whitelisted-service" 标签
                    String serviceName = parser.getAttributeValue(null, "service");
                    if (serviceName == null) {
                        Slog.w(TAG, "<backup-transport-whitelisted-service> without service in "
                                + permFile + " at " + parser.getPositionDescription());
                    } else {
                        ComponentName cn = ComponentName.unflattenFromString(serviceName);
                        if (cn == null) {
                            Slog.w(TAG,
                                    "<backup-transport-whitelisted-service> with invalid service name "
                                    + serviceName + " in "+ permFile
                                    + " at " + parser.getPositionDescription());
                        } else {
                            mBackupTransportWhitelist.add(cn);
                        }
                    }
                    XmlUtils.skipCurrentTag(parser);

                } else if ("disabled-until-used-preinstalled-carrier-associated-app".equals(name)
                        && allowAppConfigs) {  
                    //【15】解析 "disabled-until-used-preinstalled-carrier-associated-app" 标签
                    String pkgname = parser.getAttributeValue(null, "package");
                    String carrierPkgname = parser.getAttributeValue(null, "carrierAppPackage");
                    if (pkgname == null || carrierPkgname == null) {
                        Slog.w(TAG, "<disabled-until-used-preinstalled-carrier-associated-app"
                                + " without package or carrierAppPackage in " + permFile + " at "
                                + parser.getPositionDescription());
                    } else {
                        List<String> associatedPkgs =
                                mDisabledUntilUsedPreinstalledCarrierAssociatedApps.get(
                                        carrierPkgname);
                        if (associatedPkgs == null) {
                            associatedPkgs = new ArrayList<>();
                            mDisabledUntilUsedPreinstalledCarrierAssociatedApps.put(
                                    carrierPkgname, associatedPkgs);
                        }
                        associatedPkgs.add(pkgname);
                    }
                    XmlUtils.skipCurrentTag(parser);
                } else {
                    XmlUtils.skipCurrentTag(parser);
                    continue;
                }
            }
        } catch (XmlPullParserException e) {
            Slog.w(TAG, "Got exception parsing permissions.", e);
        } catch (IOException e) {
            Slog.w(TAG, "Got exception parsing permissions.", e);
        } finally {
            IoUtils.closeQuietly(permReader);
        }

        // Some devices can be field-converted to FBE, so offer to splice in
        // those features if not already defined by the static config
        // 加密相关的 feature!
        if (StorageManager.isFileEncryptedNativeOnly()) {
            addFeature(PackageManager.FEATURE_FILE_BASED_ENCRYPTION, 0);
            addFeature(PackageManager.FEATURE_SECURELY_REMOVES_USERS, 0);
        }

        // 从 mAvailableFeatures 移除不支持的 feature！
        for (String featureName : mUnavailableFeatures) {
            removeFeature(featureName);
        }
    }

```
这个方法的流程如下：

- 解析 "group" 标签；
- 解析 "permission" 标签；
- 解析 "assign-permission" 标签；
- 解析 "library" 标签 
- 解析 "feature" 标签 
- 解析 "unavailable-feature" 标签
- 解析 "allow-in-power-save-except-idle" 标签
- 解析 "allow-in-power-save" 标签
- 解析 "allow-in-data-usage-save" 标签
- 解析 "app-link" 标签
- 解析 "system-user-whitelisted-app" 标签
- 解析 "system-user-blacklisted-app" 标签
- 解析 "default-enabled-vr-app" 标签
- 解析 "backup-transport-whitelisted-service" 标签
- 解析 "disabled-until-used-preinstalled-carrier-associated-app" 标签

这些解析后的数据都会保存到下面的集合中：

```
public class SystemConfig {

    // 系统定义的所有的 gid！
    int[] mGlobalGids;

    // 保存了 uid 和其所有的权限信息的映射关系！
    final SparseArray<ArraySet<String>> mSystemPermissions = new SparseArray<>();

    // 系统所有的共享库信息，key 是 lib name，value 是 lib path！
    final ArrayMap<String, String> mSharedLibraries  = new ArrayMap<>();

    // 系统中可用的 feature！
    final ArrayMap<String, FeatureInfo> mAvailableFeatures = new ArrayMap<>();

    // 系统中不可用的 feature！
    final ArraySet<String> mUnavailableFeatures = new ArraySet<>();

    // 保存了 gid 和其所拥有的权限的映射关系！
    final ArrayMap<String, PermissionEntry> mPermissions = new ArrayMap<>();

    final ArraySet<String> mAllowInPowerSaveExceptIdle = new ArraySet<>();
    final ArraySet<String> mAllowInPowerSave = new ArraySet<>();
    final ArraySet<String> mAllowInDataUsageSave = new ArraySet<>();

    final ArraySet<String> mLinkedApps = new ArraySet<>();
    final ArraySet<String> mSystemUserWhitelistedApps = new ArraySet<>();
    final ArraySet<String> mSystemUserBlacklistedApps = new ArraySet<>();

    // 默认的 VR 组件信息！
    final ArraySet<ComponentName> mDefaultVrComponents = new ArraySet<>();
    final ArraySet<ComponentName> mBackupTransportWhitelist = new ArraySet<>();
    final ArrayMap<String, List<String>> mDisabledUntilUsedPreinstalledCarrierAssociatedApps =
            new ArrayMap<>();
}
```
下面，我们重点分析几个配置解析，以 /system/etc/permissions 为例，该目录所有的 xml 文件的根节点都是：
```java
<permissions>

</permissions>
```
下面我们继续来看：

### 2.2.1 解析 "group" - 获得系统中的所有 gid

和 "group" 相关的 xml 内容如下，可以看到  "group" 并不是单独存在的，而是和  "permission" 一起配合使用！

```xml
<permissions>
    <permission name="android.permission.BLUETOOTH_ADMIN" >
        <!--解析 group 标签-->
        <group gid="net_bt_admin" />
    </permission>
</permissions>
```
下面是解析过程：
```java
        if ("group".equals(name) && allowAll) { 
            //【1】解析 "group" 标签
            String gidStr = parser.getAttributeValue(null, "gid");
            if (gidStr != null) {
                // 将 Gid 从字符串形式转为对应的 int 形式！
                int gid = android.os.Process.getGidForName(gidStr);
                mGlobalGids = appendInt(mGlobalGids, gid);
            } else {
                Slog.w(TAG, "<group> without gid in " + permFile + " at "
                            + parser.getPositionDescription());
            }
            XmlUtils.skipCurrentTag(parser);
            continue;
        }
```
根据组名称，调用 android.os.Process.getGidForName 转为 gid，保存到 mGlobalGids！


### 2.2.2 解析 "permission" - 获得系统权限和所属的 gid
xml 信息如下：
```xml
<permissions>
    <!--解析 permission 标签-->
    <permission name="android.permission.BLUETOOTH_ADMIN" >
        <group gid="net_bt_admin" />
    </permission>

    <permission name="android.permission.BLUETOOTH" >
        <group gid="net_bt" />
    </permission>
</permissions>
```
下面是解析过程：
```java
         } else if ("permission".equals(name) && allowPermissions) { 
            //【1】解析 "permission" 标签，获得权限的名称 perm！
            String perm = parser.getAttributeValue(null, "name");
            if (perm == null) {
                Slog.w(TAG, "<permission> without name in " + permFile + " at "
                         + parser.getPositionDescription());
                XmlUtils.skipCurrentTag(parser);
                continue;
            }

            perm = perm.intern();
                    
            //【2】调用 readPermission 继续解析权限
            readPermission(parser, perm);

        }
```
进一步调用 readPermission 方法：
```java
    void readPermission(XmlPullParser parser, String name)
            throws IOException, XmlPullParserException {
        //【1】如果 mPermissions 已经包含这个权限，解析冲突！
        if (mPermissions.containsKey(name)) {
            throw new IllegalStateException("Duplicate permission definition for " + name);
        }
        
        //【2】perUser 表示该权限的 gid 是否针对设备用户 id 进行调整，默认为 false！
        final boolean perUser = XmlUtils.readBooleanAttribute(parser, "perUser", false);
        
        //【3】创建 PermissionEntry，并添加到 mPermissions 中！
        final PermissionEntry perm = new PermissionEntry(name, perUser);
        mPermissions.put(name, perm);

        int outerDepth = parser.getDepth();
        int type;
        while ((type=parser.next()) != XmlPullParser.END_DOCUMENT
               && (type != XmlPullParser.END_TAG
                       || parser.getDepth() > outerDepth)) {
            if (type == XmlPullParser.END_TAG
                    || type == XmlPullParser.TEXT) {
                continue;
            }

            String tagName = parser.getName();

            //【4】解析内部 "group" 标签，获得权限所属的 gid，将其添加到 PermissionEntry 的 gids 数组中！
            if ("group".equals(tagName)) {
                String gidStr = parser.getAttributeValue(null, "gid");
                if (gidStr != null) {
                    int gid = Process.getGidForName(gidStr);
                    perm.gids = appendInt(perm.gids, gid);
                } else {
                    Slog.w(TAG, "<group> without gid at "
                            + parser.getPositionDescription());
                }
            }
            XmlUtils.skipCurrentTag(parser);
        }
    }

```
可以看到，解析 "permission" 标签以及其内部的标签 "group"，获得了 gid 和权限的映射关系，保存到了 mPermissions 中，一个权限可以对应多个 gid！

### 2.2.3 解析 "assign-permission" - 给指定的 uid 分配权限

"assign-permission" 的 xml 内容如下：
```xml
    <assign-permission name="android.permission.MODIFY_AUDIO_SETTINGS" uid="media" />
    <assign-permission name="android.permission.ACCESS_SURFACE_FLINGER" uid="media" />
    <assign-permission name="android.permission.WAKE_LOCK" uid="media" />
```
下面是解析过程：
```java
        } else if ("assign-permission".equals(name) && allowPermissions) { 
            //【1】解析 "name" 属性，获得权限的名称！
            String perm = parser.getAttributeValue(null, "name");
            if (perm == null) {
                Slog.w(TAG, "<assign-permission> without name in " + permFile + " at "
                                + parser.getPositionDescription());
                XmlUtils.skipCurrentTag(parser);
                continue;
            }
            
            //【2】获得权限所属用户的名称！
            String uidStr = parser.getAttributeValue(null, "uid");
            if (uidStr == null) {
                Slog.w(TAG, "<assign-permission> without uid in " + permFile + " at "
                            + parser.getPositionDescription());
                XmlUtils.skipCurrentTag(parser);
                continue;
            }
            
            //【3】获得用户 uid！
            int uid = Process.getUidForName(uidStr);
            if (uid < 0) {
                Slog.w(TAG, "<assign-permission> with unknown uid \""
                        + uidStr + "  in " + permFile + " at "
                        + parser.getPositionDescription());
                XmlUtils.skipCurrentTag(parser);
                continue;
            }
            perm = perm.intern();
            ArraySet<String> perms = mSystemPermissions.get(uid);
            if (perms == null) {
                perms = new ArraySet<String>();
                mSystemPermissions.put(uid, perms);
            }
            perms.add(perm);
            XmlUtils.skipCurrentTag(parser);

        }
```
可以看到，解析 "assign-permission" 标签，获得了 uid 和权限的映射关系，保存到了 mSystemPermissions 中，一个 uid 可以对应多个权限！

### 2.2.4 解析 "library" - 获得共享库信息
xml 信息如下：
```xml
    <library name="android.test.runner"
            file="/system/framework/android.test.runner.jar" />
    <library name="javax.obex"
            file="/system/framework/javax.obex.jar" />
    <library name="org.apache.http.legacy"
            file="/system/framework/org.apache.http.legacy.jar" />
```
下面是解析过程：

```java
                } else if ("library".equals(name) && allowLibs) { 
                    //【1】解析 "library" 标签
                    String lname = parser.getAttributeValue(null, "name");
                    String lfile = parser.getAttributeValue(null, "file");
                    if (lname == null) {
                        Slog.w(TAG, "<library> without name in " + permFile + " at "
                                + parser.getPositionDescription());
                    } else if (lfile == null) {
                        Slog.w(TAG, "<library> without file in " + permFile + " at "
                                + parser.getPositionDescription());
                    } else {
                        //Log.i(TAG, "Got library " + lname + " in " + lfile);
                        mSharedLibraries.put(lname, lfile);
                    }
                    XmlUtils.skipCurrentTag(parser);
                    continue;

                }
```
解析共享库信息，保存到 mSharedLibraries 中，key 为 共享库名称，value 为共享库路径！


### 2.2.5 解析 "feature" - 获得系统特性配置信息
"feature" 的 xml 内容如下：
```java
<permissions>
    <feature name="android.hardware.bluetooth" />
</permissions>
```
下面是解析过程：
``` java
                } else if ("feature".equals(name) && allowFeatures) { 

                    //【1】解析 "feature" 标签
                    // 获得 feature 的名称和版本号
                    String fname = parser.getAttributeValue(null, "name");
                    int fversion = XmlUtils.readIntAttribute(parser, "version", 0);
                    boolean allowed;
                    if (!lowRam) {
                        allowed = true;
                    } else {
                        String notLowRam = parser.getAttributeValue(null, "notLowRam");
                        allowed = !"true".equals(notLowRam);
                    }
                    if (fname == null) {
                        Slog.w(TAG, "<feature> without name in " + permFile + " at "
                                + parser.getPositionDescription());
                    } else if (allowed) {
                        addFeature(fname, fversion);
                    }
                    XmlUtils.skipCurrentTag(parser);
                    continue;

                } else if ("unavailable-feature".equals(name) && allowFeatures) {

                    //【2】解析 "unavailable-feature" 标签
                    String fname = parser.getAttributeValue(null, "name");
                    if (fname == null) {
                        Slog.w(TAG, "<unavailable-feature> without name in " + permFile + " at "
                                + parser.getPositionDescription());
                    } else {
                        mUnavailableFeatures.add(fname);
                    }
                    XmlUtils.skipCurrentTag(parser);
                    continue;

                }
```
上面会把配置文件中的 feature 解析出来，可用 feature 会被添加到 mAvailableFeatures 中，不可用 feature 会被添加到 mUnavailableFeatures 中！

### 2.2.6 解析其他的标签

其他的标签有如下几个:

- 解析 **"allow-in-power-save-except-idle"**，这个是省电模式的白名单，该白名单不包括 idle 白名单！
     - mAllowInPowerSaveExceptIdle.add(pkgname);


- 解析 **"allow-in-power-save"**，这个是省电模式的白名单！
     - mAllowInPowerSave.add(pkgname);

- 解析 **"allow-in-data-usage-save"**
     - mAllowInDataUsageSave.add(pkgname);

- 解析 **"app-link"**
     - mLinkedApps.add(pkgname);

- 解析 **"system-user-whitelisted-app"**
     - mSystemUserWhitelistedApps.add(pkgname);

- 解析 **"default-enabled-vr-app"**
     - mDefaultVrComponents.add(new ComponentName(pkgname, clsname));

- 解析 **"backup-transport-whitelisted-service"**
     - ComponentName cn = ComponentName.unflattenFromString(serviceName);  
     - mBackupTransportWhitelist.add(cn);

- 解析 **"disabled-until-used-preinstalled-carrier-associated-app"**
     - associatedPkgs = new ArrayList<>();
     - mDisabledUntilUsedPreinstalledCarrierAssociatedApps.put(carrierPkgname, associatedPkgs);
     - associatedPkgs.add(pkgname);


SystemConfig 内部的数据关系图：

![SystemConfig.png-199.2kB](https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/23/SystemConfig-20221024231541799.png )

## 2.3 PMS 对于 SystemConfig 的后续处理

然后，回到 PMS 的构造器中：
将 SystemConfig 中的 mGlobalGids，mSystemPermissions 和 mAvailableFeatures 拷贝一份保存到 PMS 中！
```java
        //【1】保存到 PMS 中！
        mGlobalGids = systemConfig.getGlobalGids();
        mSystemPermissions = systemConfig.getSystemPermissions();
        mAvailableFeatures = systemConfig.getAvailableFeatures();
```

接着，处理权限信息和共享库信息，将解析出来的 mPermissions 信息保存到 Setting 的 mPermissions 中，权限所属的包名为 android !

因为 SystemConfig.mPermissions 保存的是系统中定义的权限和对应的 Gid 的关系！！
```java
            //【2】从获得 SystemConfig 加载的系统权限集合 mPermissions！
            ArrayMap<String, SystemConfig.PermissionEntry> permConfig
                    = systemConfig.getPermissions();

            for (int i = 0; i < permConfig.size(); i++) {
                SystemConfig.PermissionEntry perm = permConfig.valueAt(i);
                BasePermission bp = mSettings.mPermissions.get(perm.name);
                
                //【2】加入到 Settings 的 mPermissions 中，SystemConfig 解析的权限的包名都为 “android”！
                if (bp == null) {
                    // BasePermission.TYPE_BUILTIN 表示的是在编译时期就确定好的，系统要提供的权限！
                    bp = new BasePermission(perm.name, "android", BasePermission.TYPE_BUILTIN);
                    mSettings.mPermissions.put(perm.name, bp);
                }
                // 如果系统权限有所属的 gids，将其添加到 BasePermission 对象中！
                if (perm.gids != null) {
                    bp.setGids(perm.gids, perm.perUser);
                }
            }

            //【3】处理共享库，将共享库信息也保存一份在 PMS 中！
            ArrayMap<String, String> libConfig = systemConfig.getSharedLibraries();
            for (int i=0; i<libConfig.size(); i++) {
                mSharedLibraries.put(libConfig.keyAt(i),
                        new SharedLibraryEntry(libConfig.valueAt(i), null));
            }
```
创建了一个 BasePermission 对象，封装系统权限！！
```java
    BasePermission(String _name, String _sourcePackage, int _type) {
        name = _name; // 权限名；
        sourcePackage = _sourcePackage; // 定义权限的包名；
        type = _type; // 权限类型！
        // Default to most conservative protection level.
        protectionLevel = PermissionInfo.PROTECTION_SIGNATURE; // 权限界别！
    }
```
同时设置系统权限的 gid：
```java
    public void setGids(int[] gids, boolean perUser) {
        this.gids = gids; // 设置 gids
        this.perUser = perUser;
    }
```
可以看到，SystemConfig 保存的均是和系统相关的属性信息，比如系统权限，系统特性， Gid 等等，而 Settings 中则保存的系统和和应用所有的信息，所以，这里需要将 SystemConfig 中的权限信息保存过来！

这里就先分析到这里，后面会继续补充！

## 2.4 阶段总结

我们来总结一下，SystemConfig、Settings 以及 PackageManagerService 之间的数据关系：

![pic.png-81.1kB](https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/23/pic.png )


通过 SystemConfig 解析，我们可以系统定义的一些配置信息：

- 系统定义的一些 gid；
- 系统定义的权限和 uid 的映射关系；
- 系统定义的权限和 gid 的映射关系；
- 系统共享库和feature；

我们继续看！
# 6 PackageHandler
```java
        // 建立并启动一个名为 “PackageManager” 的 ServiceThread，用于处理一些耗时的操作！
        mHandlerThread = new ServiceThread(TAG,
                    Process.THREAD_PRIORITY_BACKGROUND, true /*allowIo*/);
        mHandlerThread.start();
        // 创建一个 PackageHandler 对象，绑定前面创建的 HandlerThread！
        mHandler = new PackageHandler(mHandlerThread.getLooper());
```
这里的 ServiceThread 继承了 HandlerThread，专门为系统服务定义的！

PackageHandler 处理的 msg 有如下：
```java
    static final int SEND_PENDING_BROADCAST = 1;
    static final int MCS_BOUND = 3;
    static final int END_COPY = 4;
    static final int INIT_COPY = 5;
    static final int MCS_UNBIND = 6;
    static final int START_CLEANING_PACKAGE = 7; // 清楚 package
    static final int FIND_INSTALL_LOC = 8;
    static final int POST_INSTALL = 9; // 安装 package
    static final int MCS_RECONNECT = 10;
    static final int MCS_GIVE_UP = 11;
    static final int UPDATED_MEDIA_STATUS = 12;
    static final int WRITE_SETTINGS = 13;
    static final int WRITE_PACKAGE_RESTRICTIONS = 14;
    static final int PACKAGE_VERIFIED = 15; // 校验 package
    static final int CHECK_PENDING_VERIFICATION = 16;
    static final int START_INTENT_FILTER_VERIFICATIONS = 17;
    static final int INTENT_FILTER_VERIFIED = 18;
    static final int WRITE_PACKAGE_LIST = 19;
```
一些耗时的操作，pms 都会交给 PackageHandler 去做，比如安装应用，卸载应用等等的操作，这个以后再分析！


# 7 SELinuxMMAC.readInstallPolicy

调用 SELinuxMMAC.readInstallPolicy 读取 /etc/security/mac_permissions.xml 文件中的 selinux 配置，设置应用程序的安全上下文！
```java
    public static boolean readInstallPolicy() {
        // Temp structure to hold the rules while we parse the xml file
        List<Policy> policies = new ArrayList<>();

        FileReader policyFile = null;
        XmlPullParser parser = Xml.newPullParser();
        try {
            //【1】获得 /etc/security/mac_permissions.xml 文件引用！
            policyFile = new FileReader(MAC_PERMISSIONS);
            Slog.d(TAG, "Using policy file " + MAC_PERMISSIONS);

            parser.setInput(policyFile);
            parser.nextTag();
            parser.require(XmlPullParser.START_TAG, null, "policy");

            while (parser.next() != XmlPullParser.END_TAG) {
                if (parser.getEventType() != XmlPullParser.START_TAG) {
                    continue;
                }

                switch (parser.getName()) {
                    case "signer": //【7.1】解析 "signer" 标签的内容，返回一个 Policy 对象！！
                        policies.add(readSignerOrThrow(parser));
                        break;
                    default:
                        skip(parser);
                }
            }
        } catch (IllegalStateException | IllegalArgumentException |
                XmlPullParserException ex) {
            ... ... ... ...
            return false;
        } catch (IOException ioe) {
            Slog.w(TAG, "Exception parsing " + MAC_PERMISSIONS, ioe);
            return false;
        } finally {
            IoUtils.closeQuietly(policyFile);
        }

        //【3】使用比较器对其进行排序！
        PolicyComparator policySort = new PolicyComparator();
        Collections.sort(policies, policySort);
        if (policySort.foundDuplicate()) {
            Slog.w(TAG, "ERROR! Duplicate entries found parsing " + MAC_PERMISSIONS);
            return false;
        }

        
        synchronized (sPolicies) {
            sPolicies = policies;
            if (DEBUG_POLICY_ORDER) {
                for (Policy policy : sPolicies) {
                    Slog.d(TAG, "Policy: " + policy.toString());
                }
            }
        }

        return true;
    }
```
这里用到了一个文件对象！
```java
    private static final File MAC_PERMISSIONS = new File(Environment.getRootDirectory(),
            "/etc/security/mac_permissions.xml");
```
我们来看下该文件的内容；
```xml
<policy>
    <!-- Platform dev key in AOSP -->
    <signer signature="@PLATFORM" >
      <seinfo value="platform" />
    </signer>

    <!-- Media key in AOSP -->
    <signer signature="@MEDIA" >
      <seinfo value="media" />
    </signer>
</policy>
```

在 Android 引入 SeLinux 以后，Android 会为每个文件打上 SE Label，对于 APK 而言，打 SE Label 的准则就是签名，即根据签名信息打上不同的 SE Label。

Android 将签名分类成为 platform，testkey，media等，签名与类别的映射关系就存在一个叫 mac_permission.xml 的文件中。

所以，需要读取该文件的内容。

根据 mac_permissions.xml 的定义，如果 App 是在 Android 源码编译环境下，其 Android.mk 中指定了 LOCAL_CERTIFICATE : = platform 的话，它的 seinfo 就是 platform。如果 Android.mk 中不进行对应的设置，setinfo 为默认值 default。

对于第三方 APK，其 seinfo 值通常为 default。

当 mac_permissions.xml 编译进 system/etc 目录时，@PLATFORM 将被实际的签名信息替换：

```java
<?xml version="1.0" encoding="iso-8859-1"?>
<!-- AUTOGENERATED FILE DO NOT MODIFY -->
<policy>
    <signer signature="308204......232e4fe9bd961394c6300e5138e3cfd285e6e4e483538cb8b1b357">
        <seinfo value="platform"/>
    </signer>
    <default>
        <seinfo value="default"/>
    </default>
</policy>
```
这里不多说了。

## 7.1 SELinuxMMAC.readSignerOrThrow

```java
    private static Policy readSignerOrThrow(XmlPullParser parser) throws IOException,
            XmlPullParserException {
        parser.require(XmlPullParser.START_TAG, null, "signer");
        Policy.PolicyBuilder pb = new Policy.PolicyBuilder();
        //【1】解析 signature 属性
        String cert = parser.getAttributeValue(null, "signature");
        if (cert != null) {
            pb.addSignature(cert);
        }

        while (parser.next() != XmlPullParser.END_TAG) {
            if (parser.getEventType() != XmlPullParser.START_TAG) {
                continue;
            }

            String tagName = parser.getName();
            if ("seinfo".equals(tagName)) { //【2】解析子标签 seinfo！
                String seinfo = parser.getAttributeValue(null, "value");
                pb.setGlobalSeinfoOrThrow(seinfo);
                readSeinfo(parser);
            } else if ("package".equals(tagName)) { //【3】解析子标签 package！
                readPackageOrThrow(parser, pb);
            } else if ("cert".equals(tagName)) { //【4】解析子标签 cert！
                String sig = parser.getAttributeValue(null, "signature");
                pb.addSignature(sig);
                readCert(parser);
            } else {
                skip(parser);
            }
        }

        return pb.build();
    }
```
可以看到，readSignerOrThrow 方法会解析 signer 标签和其子标签，然后将结果通过 buidler 模式，封装成一个 Policy 对象返回！

# 8 Settings.readLPw
继续看：
```java
    mFirstBoot = !mSettings.readLPw(sUserManager.getUsers(false));
```
这个方法用来恢复上一次安装的信息，参数传递：

- List<UserInfo> users：当前是设备用户 id！

```java
    boolean readLPw(@NonNull List<UserInfo> users) {
        FileInputStream str = null;
        
        //【1】如果 packages-backup.xml 存在，就从 packages-backup.xml 中读取
        if (mBackupSettingsFilename.exists()) {
            try {
                str = new FileInputStream(mBackupSettingsFilename);
                mReadMessages.append("Reading from backup settings file\n");
                PackageManagerService.reportSettingsProblem(Log.INFO,
                        "Need to read from backup settings file");
                
                // 如果 packages.xml 存在，删除！
                if (mSettingsFilename.exists()) {
                    Slog.w(PackageManagerService.TAG, "Cleaning up settings file "
                            + mSettingsFilename);
                    mSettingsFilename.delete();
                }
            } catch (java.io.IOException e) {
            }
        }

        // 清空一些集合，防止异常问题！
        mPendingPackages.clear();
        mPastSignatures.clear();
        mKeySetRefs.clear();
        mInstallerPackages.clear();

        try {
        
            //【2】如果 packages-backup.xml 不存在，就从 packages.xml 中读取！ 
            if (str == null) {
                if (!mSettingsFilename.exists()) {
                    mReadMessages.append("No settings file found\n");
                    PackageManagerService.reportSettingsProblem(Log.INFO,
                            "No settings file; creating initial state");
                    // It's enough to just touch version details to create them
                    // with default values
                    findOrCreateVersion(StorageManager.UUID_PRIVATE_INTERNAL);
                    findOrCreateVersion(StorageManager.UUID_PRIMARY_PHYSICAL);
                    return false;
                }
                str = new FileInputStream(mSettingsFilename);
            }

            XmlPullParser parser = Xml.newPullParser();
            parser.setInput(str, StandardCharsets.UTF_8.name());

            int type;
            while ((type = parser.next()) != XmlPullParser.START_TAG
                    && type != XmlPullParser.END_DOCUMENT) {
                ;
            }

            if (type != XmlPullParser.START_TAG) {
                mReadMessages.append("No start tag found in settings file\n");
                PackageManagerService.reportSettingsProblem(Log.WARN,
                        "No start tag found in package manager settings");
                Slog.wtf(PackageManagerService.TAG,
                        "No start tag found in package manager settings");
                return false;
            }

            int outerDepth = parser.getDepth();
            while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
                    && (type != XmlPullParser.END_TAG || parser.getDepth() > outerDepth)) {
                if (type == XmlPullParser.END_TAG || type == XmlPullParser.TEXT) {
                    continue;
                }

                String tagName = parser.getName();
                if (tagName.equals("package")) { //【2.1】解析 "package"
                    readPackageLPw(parser);
                    
                } else if (tagName.equals("permissions")) { //【2.2】解析 "permissions"
                    readPermissionsLPw(mPermissions, parser);
                    
                } else if (tagName.equals("permission-trees")) {  //【2.3】解析 "permission-trees"
                    readPermissionsLPw(mPermissionTrees, parser);
                    
                } else if (tagName.equals("shared-user")) { //【2.4】解析 "shared-user"
                    readSharedUserLPw(parser);
                    
                } else if (tagName.equals("preferred-packages")) { 
                    // no longer used.

                } else if (tagName.equals("preferred-activities")) { //【2.5】解析 "preferred-activities"
                    readPreferredActivitiesLPw(parser, 0);
                    
                } else if (tagName.equals(TAG_PERSISTENT_PREFERRED_ACTIVITIES)) { 
                    // 解析 "persistent-preferred-activities"
                    // Android 7.1.1 的默认应用已经不在 packages.xml 中定义了！
                    // 而是在 package-restrictions 文件中定义，这里是为了兼容以前的版本！
                    readPersistentPreferredActivitiesLPw(parser, 0); 
                    
                } else if (tagName.equals(TAG_CROSS_PROFILE_INTENT_FILTERS)) {
                    // 解析 "crossProfile-intent-filters"
                    readCrossProfileIntentFiltersLPw(parser, 0);
                    
                } else if (tagName.equals(TAG_DEFAULT_BROWSER)) { //【2.6】解析 "default-browser"
                    readDefaultAppsLPw(parser, 0);
                    
                } else if (tagName.equals("updated-package")) { //【2.7】解析 "updated-package"
                    readDisabledSysPackageLPw(parser);
                    
                } else if (tagName.equals("cleaning-package")) { //【2.8】解析 "cleaning-package"
                    String name = parser.getAttributeValue(null, ATTR_NAME);
                    String userStr = parser.getAttributeValue(null, ATTR_USER);
                    String codeStr = parser.getAttributeValue(null, ATTR_CODE);
                    if (name != null) {
                        int userId = UserHandle.USER_SYSTEM;
                        boolean andCode = true;
                        try {
                            if (userStr != null) {
                                userId = Integer.parseInt(userStr);
                            }
                        } catch (NumberFormatException e) {
                        }
                        if (codeStr != null) {
                            andCode = Boolean.parseBoolean(codeStr);
                        }
                        addPackageToCleanLPw(new PackageCleanItem(userId, name, andCode));
                    }
                    
                } else if (tagName.equals("renamed-package")) { //【2.9】解析 "renamed-package"
                    String nname = parser.getAttributeValue(null, "new");
                    String oname = parser.getAttributeValue(null, "old");
                    if (nname != null && oname != null) {
                        mRenamedPackages.put(nname, oname);
                    }
                    
                } else if (tagName.equals("restored-ivi")) { // 解析 "restored-ivi"
                    readRestoredIntentFilterVerifications(parser);
                    
                } else if (tagName.equals("last-platform-version")) { // 解析 "last-platform-version"
                    // Upgrade from older XML schema
                    final VersionInfo internal = findOrCreateVersion(
                            StorageManager.UUID_PRIVATE_INTERNAL);
                    final VersionInfo external = findOrCreateVersion(
                            StorageManager.UUID_PRIMARY_PHYSICAL);

                    internal.sdkVersion = XmlUtils.readIntAttribute(parser, "internal", 0);
                    external.sdkVersion = XmlUtils.readIntAttribute(parser, "external", 0);
                    internal.fingerprint = external.fingerprint =
                            XmlUtils.readStringAttribute(parser, "fingerprint");

                } else if (tagName.equals("database-version")) { // 解析 "database-version"
                    // Upgrade from older XML schema
                    final VersionInfo internal = findOrCreateVersion(
                            StorageManager.UUID_PRIVATE_INTERNAL);
                    final VersionInfo external = findOrCreateVersion(
                            StorageManager.UUID_PRIMARY_PHYSICAL);

                    internal.databaseVersion = XmlUtils.readIntAttribute(parser, "internal", 0);
                    external.databaseVersion = XmlUtils.readIntAttribute(parser, "external", 0);

                } else if (tagName.equals("verifier")) { // 解析 "verifier"
                    final String deviceIdentity = parser.getAttributeValue(null, "device");
                    try {
                        mVerifierDeviceIdentity = VerifierDeviceIdentity.parse(deviceIdentity);
                    } catch (IllegalArgumentException e) {
                        Slog.w(PackageManagerService.TAG, "Discard invalid verifier device id: "
                                + e.getMessage());
                    }
                    
                } else if (TAG_READ_EXTERNAL_STORAGE.equals(tagName)) { // 解析 "read-external-storage"
                    final String enforcement = parser.getAttributeValue(null, ATTR_ENFORCEMENT);
                    mReadExternalStorageEnforced = "1".equals(enforcement);
                    
                } else if (tagName.equals("keyset-settings")) { // 解析 "keyset-settings"
                    mKeySetManagerService.readKeySetsLPw(parser, mKeySetRefs);
                    
                } else if (TAG_VERSION.equals(tagName)) { // 解析 "version"
                    final String volumeUuid = XmlUtils.readStringAttribute(parser,
                            ATTR_VOLUME_UUID);
                    final VersionInfo ver = findOrCreateVersion(volumeUuid);
                    ver.sdkVersion = XmlUtils.readIntAttribute(parser, ATTR_SDK_VERSION);
                    ver.databaseVersion = XmlUtils.readIntAttribute(parser, ATTR_SDK_VERSION);
                    ver.fingerprint = XmlUtils.readStringAttribute(parser, ATTR_FINGERPRINT);
                    
                } else {
                    Slog.w(PackageManagerService.TAG, "Unknown element under <packages>: "
                            + parser.getName());
                    XmlUtils.skipCurrentTag(parser);
                    
                }
            }

            str.close();

        } catch (XmlPullParserException e) {
            ... ...

        } catch (java.io.IOException e) {
            ... ...
        }

        //【3】If the build is setup to drop runtime permissions
        // on update drop the files before loading them.
        if (PackageManagerService.CLEAR_RUNTIME_PERMISSIONS_ON_UPGRADE) {
            final VersionInfo internal = getInternalVersion();
            if (!Build.FINGERPRINT.equals(internal.fingerprint)) {
                for (UserInfo user : users) {
                    mRuntimePermissionsPersistence.deleteUserRuntimePermissionsFile(user.id);
                }
            }
        }

        //【4】接着，对 mPendingPackages 集合中的需要验证共享用户 id 有效性的 package，进行共享用户 id 有效性的验证!
        final int N = mPendingPackages.size();
        for (int i = 0; i < N; i++) {
            final PendingPackage pp = mPendingPackages.get(i);
            
            // 看 sharedId 是否能对应找到一个 ShardUserSetting 对象!
            Object idObj = getUserIdLPr(pp.sharedId);
            
            if (idObj != null && idObj instanceof SharedUserSetting) { // 能找到，说明共享用户 ID 是有效的!
                PackageSetting p = getPackageLPw(pp.name, null, pp.realName,
                        (SharedUserSetting) idObj, pp.codePath, pp.resourcePath,
                        pp.legacyNativeLibraryPathString, pp.primaryCpuAbiString,
                        pp.secondaryCpuAbiString, pp.versionCode, pp.pkgFlags, pp.pkgPrivateFlags,
                        null, true /* add */, false /* allowInstall */, pp.parentPackageName,
                        pp.childPackageNames);

                if (p == null) {
                    PackageManagerService.reportSettingsProblem(Log.WARN,
                            "Unable to create application package for " + pp.name);
                    continue;
                }
                p.copyFrom(pp); // 保存到 Settings.mPackages 对象中，表明 ID 有效，已经分配 ID 了！

            } else if (idObj != null) {
                ... ... ... 

            } else {
                ... ... ...

            }
        }

        mPendingPackages.clear(); // 清空 

        //【8.3】读取 packages-stopped-backup.xml 和 packages_stopped.xml 文件 ！
        if (mBackupStoppedPackagesFilename.exists()
                || mStoppedPackagesFilename.exists()) {
            //【8.3.1】如果 packages_stopped.xml 存在，我们先会读取文件数据，然后删除 packages_stopped.xml 相关文件！
            readStoppedLPw();
            mBackupStoppedPackagesFilename.delete();
            mStoppedPackagesFilename.delete();
            //【8.3.2】然后用最新的数据更新偏好设置！
            writePackageRestrictionsLPr(UserHandle.USER_SYSTEM);
        } else {
            //【8.3.3】否则，我们直接读取已有的偏好设置，更新上次安装的信息！
            for (UserInfo user : users) {
                readPackageRestrictionsLPr(user.id);
            }
        }

        //【8.4】读取每个 user 下的运行是权限信息！
        for (UserInfo user : users) {
            mRuntimePermissionsPersistence.readStateForUserSyncLPr(user.id);
        }

        // mDisabledSysPackages 里面保存着所有被替换的 Package 信息，如果这些 package 之前配置的是共享用户 uid
        // 那这里要建立 package 和 共享用户 id 的关联！
        final Iterator<PackageSetting> disabledIt = mDisabledSysPackages.values().iterator();
        while (disabledIt.hasNext()) {
            final PackageSetting disabledPs = disabledIt.next();
            final Object id = getUserIdLPr(disabledPs.appId);
            if (id != null && id instanceof SharedUserSetting) {
                disabledPs.sharedUser = (SharedUserSetting) id;
            }
        }

        mReadMessages.append("Read completed successfully: " + mPackages.size() + " packages, "
                + mSharedUsers.size() + " shared uids\n");

        writeKernelMappingLPr();

        return true;
    }

```
这个方法的主要流程是：

- 读取 packages-backup.xml 或者 packages.xml 文件中的数据并解析，获得上一次应用的安装和使用信息！
- 校验共享用户id 的应用程序包的 uid 的有效性！
- 读取 packages-stopped-backup.xml 和 packages_stopped.xml 文件中的数据并解析！
- 处理被替换的系统应用的共享用户 id 和自身的关系！

下面我们来看下重要的属性的解析：

## 8.1 读取 pacakges.xml 文件

这个流程主要会解析如下的标签：

|标签|标签解释|
|:--|:--|
|"package"| 系统中所有应用程序包的信息 |
|"permissions"| 系统中定义的所有的权限 |
|"permission-trees"|系统中定义的权限树|
|"shared-user"|系统中定义的共享用户|
|"preferred-activities"| 系统默认应用相关 |
|"persistent-preferred-activities"|系统默认应用相关|
|"crossProfile-intent-filters"||
|"default-browser"|默认的浏览器|
|"updated-package"|系统中被更新的应用程序包|
|"cleaning-package"|系统中被清除掉的应用程序包|
|"renamed-package"|系统中被重命名的应用程序包|
|"restored-ivi"||
|"last-platform-version"|系统升级之前的版本|
|"database-version"|数据库版本|
|"verifier"||
|"read-external-storage"||
|"keyset-settings"||
|"version"|当前系统的版本|

我们重点分析和 package、permission，shared-user 相关的内容，其他的内容我们后续接触到在分析！

```xml
<packages>
    <version sdkVersion="27" databaseVersion="3" fingerprint="../release-keys" />
    <version volumeUuid="primary_physical" sdkVersion="27" databaseVersion="27"
             fingerprint=".../release-keys" />
    <permission-trees />
    <permissions>
        <item name="android.permission.REAL_GET_TASKS" package="android" protection="18" />
        <!-- ... ... ...-->
    </permissions>
    <package name="com.android.providers.calendar" codePath="/system/priv-app/CalendarProvider" 
                    nativeLibraryPath="/system/priv-app/CalendarProvider/lib" 
					primaryCpuAbi="armeabi-v7a" publicFlags="940064325" privateFlags="8" 
					ft="1624a4dcee0" it="1624a4dcee0" ut="1624a4dcee0"
					version="0" sharedUserId="10006" isOrphaned="true">
        <sigs count="1">
            <cert index="0" />
        </sigs>
        <perms>
            <item name="android.permission.WRITE_SETTINGS" granted="true" flags="0" />
			<!-- ... ... ...-->
        </perms>
        <proper-signing-keyset identifier="1" />
    </package>
    <updated-package name="com.coolqi.papapa" codePath="/system/app/papapa" 
					 ft="1624a6f0ab0" it="1624a6f0ab0" ut="1624a6f0ab0" version="5023" 
					 nativeLibraryPath="/system/app/papapa/lib" primaryCpuAbi="arm64-v8a"
					 sharedUserId="1000" />
    <shared-user name="android.uid.bluetooth" userId="1002">
        <sigs count="1">
            <cert index="0" />
        </sigs>
        <perms>
            <item name="android.permission.REAL_GET_TASKS" granted="true" flags="0" />
			<!-- ... ... -->
        </perms>
    </shared-user>
    <keyset-settings version="1">
        <keys>
            <public-key identifier="1" value="MIIBIjjYLv....uhfiUDQIDAQAB" />
            <!-- ... ... -->
        </keys>
        <keysets>
            <keyset identifier="1">
                <key-id identifier="1" />
            </keyset>
            <!-- ... ... -->
        </keysets>
        <lastIssuedKeyId value="16" />
        <lastIssuedKeySetId value="16" />
    </keyset-settings>
</packages>
```
上面是 pacakges.xml 的主要内容！

### 8.1.1 解析 "package" 标签
和 "package" 相关内容如下：
```xml
    <package name="com.qualcomm.qti.haven.telemetry.service" codePath="/system/app/TelemetryService" 
            nativeLibraryPath="/system/app/TelemetryService/lib" 
            publicFlags="940097093" privateFlags="0" ft="11e8dc5d800" it="11e8dc5d800" ut="11e8dc5d800" 
            version="25" userId="10091" isOrphaned="true">
        <sigs count="1">
            <cert index="0" />
        </sigs>
        <perms>
            <item name="android.permission.REAL_GET_TASKS" granted="true" flags="0" />
            <item name="android.permission.RECEIVE_BOOT_COMPLETED" granted="true" flags="0" />
            <item name="android.permission.INTERNET" granted="true" flags="0" />
            <item name="android.permission.ACCESS_NETWORK_STATE" granted="true" flags="0" />
        </perms>
        <proper-signing-keyset identifier="1" />
    </package>
```
接下来，看看具体的解析过程：

```java
    if (tagName.equals("package")) {
        readPackageLPw(parser);     
    }
```
#### 8.1.1.1 Settings.readPackageLPw

调用 readPackageLPw 方法！
```java
    private void readPackageLPw(XmlPullParser parser) throws XmlPullParserException, IOException {
        // 需要解析的属性
        String name = null;
        String realName = null;
        String idStr = null;
        String sharedIdStr = null;
        String codePathStr = null;
        String resourcePathStr = null;
        String legacyCpuAbiString = null;
        String legacyNativeLibraryPathStr = null;
        String primaryCpuAbiString = null;
        String secondaryCpuAbiString = null;
        String cpuAbiOverrideString = null;
        String systemStr = null;
        String installerPackageName = null;
        String isOrphaned = null;
        String volumeUuid = null;
        String uidError = null;
        int pkgFlags = 0;
        int pkgPrivateFlags = 0;
        long timeStamp = 0;
        long firstInstallTime = 0;
        long lastUpdateTime = 0;
        PackageSettingBase packageSetting = null;
        String version = null;
        int versionCode = 0;
        String parentPackageName;
        try {
            
            //【1】获得应用的 name！
            name = parser.getAttributeValue(null, ATTR_NAME);
            realName = parser.getAttributeValue(null, "realName");
            
            //【2】获得 userId 和 sharedId 的名称, userId 和 sharedIdStr 不能同时存在！
            idStr = parser.getAttributeValue(null, "userId");
            uidError = parser.getAttributeValue(null, "uidError");
            sharedIdStr = parser.getAttributeValue(null, "sharedUserId");
            
            //【3】获得应用程序包的路径，例如：/system/priv-app/CalendarProvider！
            codePathStr = parser.getAttributeValue(null, "codePath");
            resourcePathStr = parser.getAttributeValue(null, "resourcePath");

            legacyCpuAbiString = parser.getAttributeValue(null, "requiredCpuAbi");

            parentPackageName = parser.getAttributeValue(null, "parentPackageName");

            legacyNativeLibraryPathStr = parser.getAttributeValue(null, "nativeLibraryPath");
            primaryCpuAbiString = parser.getAttributeValue(null, "primaryCpuAbi");
            secondaryCpuAbiString = parser.getAttributeValue(null, "secondaryCpuAbi");
            cpuAbiOverrideString = parser.getAttributeValue(null, "cpuAbiOverride");

            if (primaryCpuAbiString == null && legacyCpuAbiString != null) {
                primaryCpuAbiString = legacyCpuAbiString;
            }

            //【4】获得版本号！
            version = parser.getAttributeValue(null, "version");
            if (version != null) {
                try {
                    versionCode = Integer.parseInt(version);
                } catch (NumberFormatException e) {
                }
            }

            // 获得安装器的名称！
            installerPackageName = parser.getAttributeValue(null, "installer");
            isOrphaned = parser.getAttributeValue(null, "isOrphaned");
            volumeUuid = parser.getAttributeValue(null, "volumeUuid");
            
            //【5】获得flags相关信息，读取顺序为：publicFlags 和 privateFlags、flags、system;
            systemStr = parser.getAttributeValue(null, "publicFlags");
            if (systemStr != null) {
                try {
                    pkgFlags = Integer.parseInt(systemStr);
                } catch (NumberFormatException e) {
                }
                systemStr = parser.getAttributeValue(null, "privateFlags");
                if (systemStr != null) {
                    try {
                        pkgPrivateFlags = Integer.parseInt(systemStr);
                    } catch (NumberFormatException e) {
                    }
                }
            } else {
                // 在 Android M 之前，是没有 publicFlags 和 privateFlags 之分的，所有的标志位都存放在 flags 中！
                systemStr = parser.getAttributeValue(null, "flags");
                if (systemStr != null) {
                    try {
                        pkgFlags = Integer.parseInt(systemStr);
                    } catch (NumberFormatException e) {
                    }
                    if ((pkgFlags & PRE_M_APP_INFO_FLAG_HIDDEN) != 0) {
                        pkgPrivateFlags |= ApplicationInfo.PRIVATE_FLAG_HIDDEN;
                    }
                    if ((pkgFlags & PRE_M_APP_INFO_FLAG_CANT_SAVE_STATE) != 0) {
                        pkgPrivateFlags |= ApplicationInfo.PRIVATE_FLAG_CANT_SAVE_STATE;
                    }
                    if ((pkgFlags & PRE_M_APP_INFO_FLAG_FORWARD_LOCK) != 0) {
                        pkgPrivateFlags |= ApplicationInfo.PRIVATE_FLAG_FORWARD_LOCK;
                    }
                    if ((pkgFlags & PRE_M_APP_INFO_FLAG_PRIVILEGED) != 0) {
                        pkgPrivateFlags |= ApplicationInfo.PRIVATE_FLAG_PRIVILEGED;
                    }
                    pkgFlags &= ~(PRE_M_APP_INFO_FLAG_HIDDEN
                            | PRE_M_APP_INFO_FLAG_CANT_SAVE_STATE
                            | PRE_M_APP_INFO_FLAG_FORWARD_LOCK
                            | PRE_M_APP_INFO_FLAG_PRIVILEGED);
                } else {
                    // For backward compatibility
                    systemStr = parser.getAttributeValue(null, "system");
                    if (systemStr != null) {
                        pkgFlags |= ("true".equalsIgnoreCase(systemStr)) ? ApplicationInfo.FLAG_SYSTEM
                                : 0;
                    } else {
                        // Old settings that don't specify system... just treat
                        // them as system, good enough.
                        pkgFlags |= ApplicationInfo.FLAG_SYSTEM;
                    }
                }
            }
            
            // 获得时间戳！
            String timeStampStr = parser.getAttributeValue(null, "ft");
            if (timeStampStr != null) {
                try {
                    timeStamp = Long.parseLong(timeStampStr, 16);
                } catch (NumberFormatException e) {
                }
            } else {
                timeStampStr = parser.getAttributeValue(null, "ts");
                if (timeStampStr != null) {
                    try {
                        timeStamp = Long.parseLong(timeStampStr);
                    } catch (NumberFormatException e) {
                    }
                }
            }
            
            //【6】获得第一次安装时间
            timeStampStr = parser.getAttributeValue(null, "it");
            if (timeStampStr != null) {
                try {
                    firstInstallTime = Long.parseLong(timeStampStr, 16);
                } catch (NumberFormatException e) {
                }
            }
            
            //【7】获得最近更新时间！
            timeStampStr = parser.getAttributeValue(null, "ut");
            if (timeStampStr != null) {
                try {
                    lastUpdateTime = Long.parseLong(timeStampStr, 16);
                } catch (NumberFormatException e) {
                }
            }
            if (PackageManagerService.DEBUG_SETTINGS)
                Log.v(PackageManagerService.TAG, "Reading package: " + name + " userId=" + idStr
                        + " sharedUserId=" + sharedIdStr);
                        
            //【8】获得 userId 的 int 值，如果 AndroidManifest.xml 有设置 android:sharedUserId 属性，
            // 那么应用的 userId 就为 0！！
            int userId = idStr != null ? Integer.parseInt(idStr) : 0;
            if (resourcePathStr == null) {
                resourcePathStr = codePathStr;
            }
            if (realName != null) {
                realName = realName.intern();
            }
            if (name == null) {
                PackageManagerService.reportSettingsProblem(Log.WARN,
                        "Error in package manager settings: <package> has no name at "
                                + parser.getPositionDescription());
            } else if (codePathStr == null) {
                PackageManagerService.reportSettingsProblem(Log.WARN,
                        "Error in package manager settings: <package> has no codePath at "
                                + parser.getPositionDescription());
            } else if (userId > 0) {
                //【8.1】如果 userId 大于0，说明 package 是独立用户 id
                // 调用 addPackageLPw 方法保存这个有独立的 Linux 用户 ID 的 Package!
                packageSetting = addPackageLPw(name.intern(), realName, new File(codePathStr),
                        new File(resourcePathStr), legacyNativeLibraryPathStr, primaryCpuAbiString,
                        secondaryCpuAbiString, cpuAbiOverrideString, userId, versionCode, pkgFlags,
                        pkgPrivateFlags, parentPackageName, null);

                if (PackageManagerService.DEBUG_SETTINGS)
                    Log.i(PackageManagerService.TAG, "Reading package " + name + ": userId="
                            + userId + " pkg=" + packageSetting);
                if (packageSetting == null) {
                    PackageManagerService.reportSettingsProblem(Log.ERROR, "Failure adding uid "
                            + userId + " while parsing settings at "
                            + parser.getPositionDescription());
                } else {
                    // 设置时间戳，第一次安装时间，最近更新时间
                    packageSetting.setTimeStamp(timeStamp);
                    packageSetting.firstInstallTime = firstInstallTime;
                    packageSetting.lastUpdateTime = lastUpdateTime;
                }
            } else if (sharedIdStr != null) {
            
                //【8.2】sharedIdStr 不为 null，说明 package 设置了共享用户 id！
                userId = sharedIdStr != null ? Integer.parseInt(sharedIdStr) : 0;
                if (userId > 0) {
                    // 对于共享用户 ID 这种情况，还需要验证其有效性!
                    // 创建一个 PendingPackage 对象，来封装这个有共享用户 ID 的 package 的信息!
                    packageSetting = new PendingPackage(name.intern(), realName, new File(
                            codePathStr), new File(resourcePathStr), legacyNativeLibraryPathStr,
                            primaryCpuAbiString, secondaryCpuAbiString, cpuAbiOverrideString,
                            userId, versionCode, pkgFlags, pkgPrivateFlags, parentPackageName,
                            null);
                       
                    // 设置时间戳，第一次安装时间，最近更新时间！     
                    packageSetting.setTimeStamp(timeStamp);
                    packageSetting.firstInstallTime = firstInstallTime;
                    packageSetting.lastUpdateTime = lastUpdateTime;
                    
                    //【8.2.1】添加到 mPendingPackages 中，因为后续需要确定 shareUserId 的有效性！
                    mPendingPackages.add((PendingPackage) packageSetting);
                    
                    if (PackageManagerService.DEBUG_SETTINGS)
                        Log.i(PackageManagerService.TAG, "Reading package " + name
                                + ": sharedUserId=" + userId + " pkg=" + packageSetting);
                } else {
                    PackageManagerService.reportSettingsProblem(Log.WARN,
                            "Error in package manager settings: package " + name
                                    + " has bad sharedId " + sharedIdStr + " at "
                                    + parser.getPositionDescription());
                }
            } else {
                PackageManagerService.reportSettingsProblem(Log.WARN,
                        "Error in package manager settings: package " + name + " has bad userId "
                                + idStr + " at " + parser.getPositionDescription());
            }
        } catch (NumberFormatException e) {
            PackageManagerService.reportSettingsProblem(Log.WARN,
                    "Error in package manager settings: package " + name + " has bad userId "
                            + idStr + " at " + parser.getPositionDescription());
        }
        
        //【9】接下来设置其他的属性值。
        if (packageSetting != null) {
            packageSetting.uidError = "true".equals(uidError);
            packageSetting.installerPackageName = installerPackageName;
            packageSetting.isOrphaned = "true".equals(isOrphaned);
            packageSetting.volumeUuid = volumeUuid;
            packageSetting.legacyNativeLibraryPathString = legacyNativeLibraryPathStr;
            packageSetting.primaryCpuAbiString = primaryCpuAbiString;
            packageSetting.secondaryCpuAbiString = secondaryCpuAbiString;
            // Handle legacy string here for single-user mode
            final String enabledStr = parser.getAttributeValue(null, ATTR_ENABLED);

            if (enabledStr != null) {
                try {
                    packageSetting.setEnabled(Integer.parseInt(enabledStr), 0 /* userId */, null);
                } catch (NumberFormatException e) {
                    if (enabledStr.equalsIgnoreCase("true")) {
                        packageSetting.setEnabled(COMPONENT_ENABLED_STATE_ENABLED, 0, null);
                    } else if (enabledStr.equalsIgnoreCase("false")) {
                        packageSetting.setEnabled(COMPONENT_ENABLED_STATE_DISABLED, 0, null);
                    } else if (enabledStr.equalsIgnoreCase("default")) {
                        packageSetting.setEnabled(COMPONENT_ENABLED_STATE_DEFAULT, 0, null);
                    } else {
                        PackageManagerService.reportSettingsProblem(Log.WARN,
                                "Error in package manager settings: package " + name
                                        + " has bad enabled value: " + idStr + " at "
                                        + parser.getPositionDescription());
                    }
                }
            } else {
                packageSetting.setEnabled(COMPONENT_ENABLED_STATE_DEFAULT, 0, null);
            }

            if (installerPackageName != null) {
                mInstallerPackages.add(installerPackageName);
            }
            // package 安装状态！
            final String installStatusStr = parser.getAttributeValue(null, "installStatus");
            if (installStatusStr != null) {
                if (installStatusStr.equalsIgnoreCase("false")) {
                    packageSetting.installStatus = PackageSettingBase.PKG_INSTALL_INCOMPLETE;
                } else {
                    packageSetting.installStatus = PackageSettingBase.PKG_INSTALL_COMPLETE;
                }
            }

            int outerDepth = parser.getDepth();
            int type;
            while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
                    && (type != XmlPullParser.END_TAG || parser.getDepth() > outerDepth)) {
                if (type == XmlPullParser.END_TAG || type == XmlPullParser.TEXT) {
                    continue;
                }

                String tagName = parser.getName();
                // Legacy
                if (tagName.equals(TAG_DISABLED_COMPONENTS)) {
                    //【8.1.1.3】解析 "disabled-components"
                    readDisabledComponentsLPw(packageSetting, parser, 0);
                    
                } else if (tagName.equals(TAG_ENABLED_COMPONENTS)) {
                    //【8.1.1.3】解析 "enabled-components"
                    readEnabledComponentsLPw(packageSetting, parser, 0);
                    
                } else if (tagName.equals("sigs")) { // 解析 "sigs"
                    packageSetting.signatures.readXml(parser, mPastSignatures);
                    
                } else if (tagName.equals(TAG_PERMISSIONS)) { // 解析 perms
                    //【8.1.1.4】解析 "perms" 获得这个 package 的权限管理对象！
                    readInstallPermissionsLPr(parser, packageSetting.getPermissionsState());
                    packageSetting.installPermissionsFixed = true;
                    
                } else if (tagName.equals("proper-signing-keyset")) {
                    long id = Long.parseLong(parser.getAttributeValue(null, "identifier"));
                    Integer refCt = mKeySetRefs.get(id);
                    if (refCt != null) {
                        mKeySetRefs.put(id, refCt + 1);
                    } else {
                        mKeySetRefs.put(id, 1);
                    }
                    packageSetting.keySetData.setProperSigningKeySet(id);
                    
                } else if (tagName.equals("signing-keyset")) {
                    // from v1 of keysetmanagerservice - no longer used
                } else if (tagName.equals("upgrade-keyset")) {
                    long id = Long.parseLong(parser.getAttributeValue(null, "identifier"));
                    packageSetting.keySetData.addUpgradeKeySetById(id);
                    
                } else if (tagName.equals("defined-keyset")) {
                    long id = Long.parseLong(parser.getAttributeValue(null, "identifier"));
                    String alias = parser.getAttributeValue(null, "alias");
                    Integer refCt = mKeySetRefs.get(id);
                    if (refCt != null) {
                        mKeySetRefs.put(id, refCt + 1);
                    } else {
                        mKeySetRefs.put(id, 1);
                    }
                    packageSetting.keySetData.addDefinedKeySet(id, alias);
                    
                } else if (tagName.equals(TAG_DOMAIN_VERIFICATION)) { // 解析 "domain-verification"
                    readDomainVerificationLPw(parser, packageSetting);
                    
                } else if (tagName.equals(TAG_CHILD_PACKAGE)) { // 解析 "child-package"
                    String childPackageName = parser.getAttributeValue(null, ATTR_NAME);
                    if (packageSetting.childPackageNames == null) {
                        packageSetting.childPackageNames = new ArrayList<>();
                    }
                    packageSetting.childPackageNames.add(childPackageName);
                    
                } else {
                    PackageManagerService.reportSettingsProblem(Log.WARN,
                            "Unknown element under <package>: " + parser.getName());
                    XmlUtils.skipCurrentTag(parser);
                    
                }
            }
        } else {
            XmlUtils.skipCurrentTag(parser);
        }
    }
        
```
流程总结：

- 对于分配了独立的 uid 的 package，创建 PackageSetting 对象，添加到 mPackage 中，并且将自身和 uid 的引用关心保存到 mUsersId 或者 mOthersId 中；
- 对于分配了共享的 uid 的 package，创建 PendingPackage 对象，添加到 mPendingPackages 中，后续会对其共享 uid 的有效性进行校验！


#### 8.1.1.2 Settings.addPackageLPw
```java
    PackageSetting addPackageLPw(String name, String realName, File codePath, File resourcePath,
            String legacyNativeLibraryPathString, String primaryCpuAbiString,
            String secondaryCpuAbiString, String cpuAbiOverrideString, int uid, int vc, int
            pkgFlags, int pkgPrivateFlags, String parentPackageName,
            List<String> childPackageNames) {
        //【1】根据 name 值从 mPackage 集合中查询 name 对应的 PackageSettings 对象。
        PackageSetting p = mPackages.get(name);
        if (p != null) {
            // 如果能得到，且两个 uid 相等，说明已经创建过了，那就直接返回！
            if (p.appId == uid) {
                return p;
            }
            PackageManagerService.reportSettingsProblem(Log.ERROR,
                    "Adding duplicate package, keeping first: " + name);
            // 如果发现 uid 变化了，那就不会读取上一次的安装信息！
            return null;
        }
        
        //【2】否则，就创建一个 PackageSettings 对象来封装这个应用程序的信息。
        p = new PackageSetting(name, realName, codePath, resourcePath,
                legacyNativeLibraryPathString, primaryCpuAbiString, secondaryCpuAbiString,
                cpuAbiOverrideString, vc, pkgFlags, pkgPrivateFlags, parentPackageName,
                childPackageNames);
        
        //【3】他的 appid        
        p.appId = uid;
        
        //【8.1.1.2.1】在系统中保留值为 uid 的 Linux 用户 ID，添加到 mUserIds 或者 mOtherUserIds 中！
        if (addUserIdLPw(uid, p, name)) {
        
            //【4.1】将这个 PackageSettings 对象保存在 Settings.mPackages 中！！
            mPackages.put(name, p);
            return p;
        }
        return null;
    }
```
这里会调用 addUserIdLPw 方法，将 package 的 uid 进行封装，根据其范围，将其添加到 mUserIds 或者 mOtherUserIds 中！

##### 8.1.1.2.1 Settings.addUserIdLPw

```java
    private boolean addUserIdLPw(int uid, Object obj, Object name) {
        //【1】当 uid > Process.LAST_APPLICATION_UID 时，超过了系统给应用程序分配的 uid 的最大值，这是非法的 uid
        if (uid > Process.LAST_APPLICATION_UID) {
            return false;
        }

        //【2】当 uid 介于 Process.FIRST_APPLICATION_UID 和 Process.LAST_APPLICATION_UID 之间时，
        // 说明这是保留给应用程序使用的。
        if (uid >= Process.FIRST_APPLICATION_UID) {
            int N = mUserIds.size();
            
            //【2.1】根据 uid 来计算在 idex！
            final int index = uid - Process.FIRST_APPLICATION_UID;
            while (index >= N) {
                mUserIds.add(null);
                N++;
            }
            if (mUserIds.get(index) != null) {
                PackageManagerService.reportSettingsProblem(Log.ERROR,
                        "Adding duplicate user id: " + uid
                        + " name=" + name);
                return false;
            }
            
            //【2.2】把这个应用程序 PackageSettings 对象保存在这个列表中
            mUserIds.set(index, obj);
            
        } else {
            //【2.3】当 uid < Process.FIRST_APPLICATION_UID，说明是给系统使用的用户 id。
            if (mOtherUserIds.get(uid) != null) {
                PackageManagerService.reportSettingsProblem(Log.ERROR,
                        "Adding duplicate shared id: " + uid
                                + " name=" + name);
                return false;
            }
            
            //【2.4】同理
            mOtherUserIds.put(uid, obj);
        }
        return true;
    }
```

到这里，我们解析 /data/system/packages.xml 里 “package” 标签对应的 xml 元素，对于 "package" 标签的子标签 "perms"，我们最后在再分析！

#### 8.1.1.3 解析 "disabled-components" / "enabled-component 子标签

下面是解析可用和不可用组件的过程：

##### 8.1.1.3.1 Settings.readDisabledComponentsLPw
```java
    private void readDisabledComponentsLPw(PackageSettingBase packageSetting, XmlPullParser parser,
            int userId) throws IOException, XmlPullParserException {
        int outerDepth = parser.getDepth();
        int type;
        while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
                && (type != XmlPullParser.END_TAG || parser.getDepth() > outerDepth)) {
            if (type == XmlPullParser.END_TAG || type == XmlPullParser.TEXT) {
                continue;
            }

            String tagName = parser.getName();
            if (tagName.equals(TAG_ITEM)) { // item 标签
                String name = parser.getAttributeValue(null, ATTR_NAME); // 解析 name 属性
                if (name != null) {
                    packageSetting.addDisabledComponent(name.intern(), userId); 
                } else {
                    PackageManagerService.reportSettingsProblem(Log.WARN,
                            "Error in package manager settings: <disabled-components> has"
                                    + " no name at " + parser.getPositionDescription());
                }
            } else {
                PackageManagerService.reportSettingsProblem(Log.WARN,
                        "Unknown element under <disabled-components>: " + parser.getName());
            }
            XmlUtils.skipCurrentTag(parser);
        }
    }
```
这里调用了 PackageSettingBase 的 addDisabledComponent 方法！
```java
    void addDisabledComponent(String componentClassName, int userId) {
        modifyUserStateComponents(userId, true, false).disabledComponents.add(componentClassName);
    }
```
这里要看下 PackageSettingBase 的 modifyUserStateComponents 方法！

```java
    PackageUserState modifyUserStateComponents(int userId, boolean disabled, boolean enabled) {
        //【1】创建该 package 在当前 userId 下的 PackageUserState 对象！
        PackageUserState state = modifyUserState(userId);
        if (disabled && state.disabledComponents == null) {
            state.disabledComponents = new ArraySet<String>(1);
        }
        if (enabled && state.enabledComponents == null) {
            state.enabledComponents = new ArraySet<String>(1);
        }
        return state;
    }
```
PackageSettingBase 有一个 userState 成员变量，用于保存该 package 在不同 userId 下的使用情况！
```java
private final SparseArray<PackageUserState> userState = new SparseArray<PackageUserState>();
```

PackageSettingBase.modifyUserState 返回 userId 下该 package 的 PackageUserState 对象，若没还有就会创建新的！
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
PackageUserState 的属性如下，调用了默认的构造器，这里先不多说！
```java
public class PackageUserState {
    public long ceDataInode;
    public boolean installed;
    public boolean stopped;
    public boolean notLaunched;
    public boolean hidden; // Is the app restricted by owner / admin
    public boolean suspended;
    public boolean blockUninstall;
    public int enabled;
    public String lastDisableAppCaller;
    public int domainVerificationStatus;
    public int appLinkGeneration;

    public ArraySet<String> disabledComponents; // 保存不可用的组件
    public ArraySet<String> enabledComponents; // 保存可用组件！

    public PackageUserState() {
        installed = true;
        hidden = false;
        suspended = false;
        enabled = COMPONENT_ENABLED_STATE_DEFAULT;
        domainVerificationStatus =
                PackageManager.INTENT_FILTER_DOMAIN_VERIFICATION_STATUS_UNDEFINED;
    }
    ... ... ... ...
}
```
解析不可用组件的过程到这里就结束了！

##### 8.1.1.3.2 Settings.readEnabledComponentsLPw
```java
    private void readEnabledComponentsLPw(PackageSettingBase packageSetting, XmlPullParser parser,
            int userId) throws IOException, XmlPullParserException {
        int outerDepth = parser.getDepth();
        int type;
        while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
                && (type != XmlPullParser.END_TAG || parser.getDepth() > outerDepth)) {
            if (type == XmlPullParser.END_TAG || type == XmlPullParser.TEXT) {
                continue;
            }

            String tagName = parser.getName();
            if (tagName.equals(TAG_ITEM)) { // item 标签
                String name = parser.getAttributeValue(null, ATTR_NAME); // 解析 name 属性
                if (name != null) {
                    packageSetting.addEnabledComponent(name.intern(), userId);
                } else {
                    PackageManagerService.reportSettingsProblem(Log.WARN,
                            "Error in package manager settings: <enabled-components> has"
                                    + " no name at " + parser.getPositionDescription());
                }
            } else {
                PackageManagerService.reportSettingsProblem(Log.WARN,
                        "Unknown element under <enabled-components>: " + parser.getName());
            }
            XmlUtils.skipCurrentTag(parser);
        }
    }
```
同样的，这里调用了 PackageSettingBase 的 addEnabledComponent 方法：

```java
    void addEnabledComponent(String componentClassName, int userId) {
        modifyUserStateComponents(userId, false, true).enabledComponents.add(componentClassName);
    }
```
这里就不在分析了！

#### 8.1.1.4 解析 "perms" 子标签 - 解析安装时权限授予情况
```xml
   <perms>
        <item name="android.permission.REAL_GET_TASKS" granted="true" flags="0" />
        <item name="android.permission.RECEIVE_BOOT_COMPLETED" granted="true" flags="0" />
        <item name="android.permission.INTERNET" granted="true" flags="0" />
        <item name="android.permission.ACCESS_NETWORK_STATE" granted="true" flags="0" />
   </perms>
```
对于 PackageSetting 和 SharedUserSetting 都有自己的权限管理对象 PermissionsState，用于保存和管理 package 和 SharedUser 所有的权限，其权限通过解析子标签 "perms" 获得，具体的解析方法是 readInstallPermissionsLPr！

这里会涉及到一个对象，每一个 PackageSetting 对象都有一个 PermissionsState 对象，用于管理 package 的权限！
```java
public class PermissionsState {

    public static final int PERMISSION_OPERATION_FAILURE = -1; // 返回值：权限操作失败
    public static final int PERMISSION_OPERATION_SUCCESS = 0; // 返回值：权限操作成功，gid 没有改变！
    public static final int PERMISSION_OPERATION_SUCCESS_GIDS_CHANGED = 1; // 返回值：权限操作成功，gid 改变！
    
    private ArrayMap<String, PermissionData> mPermissions; // 用于封装这个应用程序的所有权限！
    private int[] mGlobalGids = NO_GIDS;  
    
    ... ... ...
}
```
我们继续分析：

##### 8.1.1.4.1 Setting.readInstallPermissionsLPr

参数传递：

- XmlPullParser parser：xml解析类对象，指向子标签 "perms" 
- PermissionsState permissionsState：PackageSetting.getPermissionsState 或者 SharedUserSetting.getPermissionsState!

</br>

接着，调用了 readInstallPermissionsLPr 来解析 package 所使用的权限和其授予情况！

```java
    void readInstallPermissionsLPr(XmlPullParser parser,
            PermissionsState permissionsState) throws IOException, XmlPullParserException {
        int outerDepth = parser.getDepth();
        int type;
        while ((type=parser.next()) != XmlPullParser.END_DOCUMENT
                && (type != XmlPullParser.END_TAG
                || parser.getDepth() > outerDepth)) {
            if (type == XmlPullParser.END_TAG
                    || type == XmlPullParser.TEXT) {
                continue;
            }
            String tagName = parser.getName();
            if (tagName.equals(TAG_ITEM)) { // 解析 "item"
                String name = parser.getAttributeValue(null, ATTR_NAME); // 接续 "name"，权限名称！
                //【1】从之前解析的 Settings.mPermissions 获得对应的权限；
                BasePermission bp = mPermissions.get(name);
                if (bp == null) {
                    Slog.w(PackageManagerService.TAG, "Unknown permission: " + name);
                    XmlUtils.skipCurrentTag(parser);
                    continue;
                }
                //【2】解析 "granted" 标签，获得授予情况，如果没有该属性或者设置为 true，那么表示授予！
                String grantedStr = parser.getAttributeValue(null, ATTR_GRANTED);
                final boolean granted = grantedStr == null
                        || Boolean.parseBoolean(grantedStr);

                //【3】解析 "flags"标签
                String flagsStr = parser.getAttributeValue(null, ATTR_FLAGS);
                final int flags = (flagsStr != null)
                        ? Integer.parseInt(flagsStr, 16) : 0;

                //【4】处理权限授予情况！
                if (granted) { 
                    //【9.1】默认授予的情况！
                    if (permissionsState.grantInstallPermission(bp) ==
                            PermissionsState.PERMISSION_OPERATION_FAILURE) {
                        Slog.w(PackageManagerService.TAG, "Permission already added: " + name);
                        XmlUtils.skipCurrentTag(parser);
                    } else {
                        permissionsState.updatePermissionFlags(bp, UserHandle.USER_ALL,
                                PackageManager.MASK_PERMISSION_FLAGS, flags);
                    }
                } else { 
                    //【9.2】默认不授予的情况！
                    if (permissionsState.revokeInstallPermission(bp) ==
                            PermissionsState.PERMISSION_OPERATION_FAILURE) {
                        Slog.w(PackageManagerService.TAG, "Permission already added: " + name);
                        XmlUtils.skipCurrentTag(parser);
                    } else {
                        //【9.3】更新权限标志位！
                        permissionsState.updatePermissionFlags(bp, UserHandle.USER_ALL,
                                PackageManager.MASK_PERMISSION_FLAGS, flags);
                    }
                }
            } else {
                Slog.w(PackageManagerService.TAG, "Unknown element under <permissions>: "
                        + parser.getName());
                XmlUtils.skipCurrentTag(parser);
            }
        }
    }
```
Settings.mPermissions 中保存的权限来自两部分，一个是 SystemConfig.mPermissions 中解析到了系统定义的权限，还有一部分，来自继续 packages.xml 中获得的权限信息！

这里要简答的说下 flags 属性，其可以取一下的几个或者多个值：

```java
    @SystemApi
    public static final int FLAG_PERMISSION_USER_SET = 1 << 0;

    @SystemApi
    public static final int FLAG_PERMISSION_USER_FIXED =  1 << 1;

    @SystemApi
    public static final int FLAG_PERMISSION_POLICY_FIXED =  1 << 2;

    @SystemApi
    public static final int FLAG_PERMISSION_REVOKE_ON_UPGRADE =  1 << 3;

    @SystemApi
    public static final int FLAG_PERMISSION_SYSTEM_FIXED =  1 << 4;

    @SystemApi
    public static final int FLAG_PERMISSION_GRANTED_BY_DEFAULT =  1 << 5;

    @SystemApi
    public static final int FLAG_PERMISSION_REVIEW_REQUIRED =  1 << 6;
```

这里简单的解释下这个这些 flags 的作用！

### 8.1.2 解析 "permissions" "permission-trees" 标签
我们先来看看 "permissions" 标签的内容，可以看到该标签的内容是：权限和定义权限的包名
```xml
    <permissions>
        <item name="android.permission.REAL_GET_TASKS" package="android" protection="18" />
        <item name="android.permission.SEND_RECEIVE_STK_INTENT" package="com.android.stk" protection="2" />
        <item name="android.permission.ACCESS_CACHE_FILESYSTEM" package="android" protection="18" />
        <item name="android.permission.REMOTE_AUDIO_PLAYBACK" package="android" protection="2" />
        <item name="android.permission.DOWNLOAD_WITHOUT_NOTIFICATION" package="com.android.providers.downloads" />
        <item name="android.permission.REGISTER_WINDOW_MANAGER_LISTENERS" package="android" protection="2" />

        <!---->
    </permissions>    
```
下面是具体的解析过程，**注意 "permissions" 的内容是最先解析的**！

对于 "permission-trees" 和 "permissions" 解析的方法都是 readPermissionsLPw，但是 permissions 的解析结果，会保存到 mPermissions 中，而 permission-trees 的解析结果，会保存到 mPermissionTrees 中；


#### 8.1.2.1 Settings.readPermissionsLPw

这里的 ArrayMap<String, BasePermission> out 分别是 Settings.permissions 和 Settings.mPermissionTrees：

```java
    final ArrayMap<String, BasePermission> mPermissions = new ArrayMap<String, BasePermission>();
    final ArrayMap<String, BasePermission> mPermissionTrees = new ArrayMap<String, BasePermission>();
```

下面我们来看看 readPermissionsLPw 方法的逻辑：
```java
    private void readPermissionsLPw(ArrayMap<String, BasePermission> out, XmlPullParser parser)
            throws IOException, XmlPullParserException {
        int outerDepth = parser.getDepth();
        int type;
        while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
                && (type != XmlPullParser.END_TAG || parser.getDepth() > outerDepth)) {
            if (type == XmlPullParser.END_TAG || type == XmlPullParser.TEXT) {
                continue;
            }

            final String tagName = parser.getName();
            if (tagName.equals(TAG_ITEM)) { // 解析 item 标签
                final String name = parser.getAttributeValue(null, ATTR_NAME); // 获得 permission 的名称！
                final String sourcePackage = parser.getAttributeValue(null, "package"); // 获得定义 permission 包名!
                final String ptype = parser.getAttributeValue(null, "type"); // 解析 type 类型！
                
                if (name != null && sourcePackage != null) {
                    final boolean dynamic = "dynamic".equals(ptype);
                    //【1】尝试从 Settings 对应集合中获得权限 bp！
                    BasePermission bp = out.get(name);

                    //【2】如果 bp 为 null 或者
                    // bp 不为 null 且其类型不是 BasePermission.TYPE_BUILTIN，就创建一个新的！
                    // BasePermission.TYPE_BUILTIN 类型的权限是系统权限，前面已经解析过！
                    // BasePermission.TYPE_DYNAMIC 类型是针对于权限树的类型的！
                    if (bp == null || bp.type != BasePermission.TYPE_BUILTIN) {
                        bp = new BasePermission(name.intern(), sourcePackage,
                                dynamic ? BasePermission.TYPE_DYNAMIC : BasePermission.TYPE_NORMAL);
                    }

                    //【3】获得定义 permission 的级别，默认为 PROTECTION_NORMAL，然后对保护级别修正! 
                    bp.protectionLevel = readInt(parser, null, "protection", PermissionInfo.PROTECTION_NORMAL);
                    bp.protectionLevel = PermissionInfo.fixProtectionLevel(bp.protectionLevel);

                    if (dynamic) { 
                        //【3.1】如果是动态权限，进一步处理，动态权限会封装为 PermissionInfo！
                        // 动态权限是可以动态添加和移除的权限！通过权限树指定！
                        PermissionInfo pi = new PermissionInfo();
                        pi.packageName = sourcePackage.intern();
                        pi.name = name.intern();
                        pi.icon = readInt(parser, null, "icon", 0);
                        pi.nonLocalizedLabel = parser.getAttributeValue(null, "label");
                        pi.protectionLevel = bp.protectionLevel;
                        bp.pendingInfo = pi;
                    }
                    
                    //【4】封装成 BasePermission 保存到 mPermissions 集合中!
                    out.put(bp.name, bp);
                } else {
                    ... ... ... ...
                }
            } else {
                ... ... ... ...
            }
            XmlUtils.skipCurrentTag(parser);
        }
    }

```
所有的非动态权限信息最终都会被解析保存到 Setting.mPermissions 中了！！

而动态权限信息会被保存在 Setting.mPermissionTrees 中了！！


### 8.1.3 解析 "shared-user" 标签

共享用户相关数据如下：
```xml
    <shared-user name="android.media" userId="10014">
        <sigs count="1">
            <cert index="3" />
        </sigs>
        <perms>
            <item name="android.permission.WRITE_SETTINGS" granted="true" flags="0" />
            <item name="android.permission.RECEIVE_BOOT_COMPLETED" granted="true" flags="0" />
            <item name="android.permission.WRITE_MEDIA_STORAGE" granted="true" flags="0" />
            <item name="android.permission.INTERNET" granted="true" flags="0" />
            <item name="android.permission.MANAGE_USB" granted="true" flags="0" />
            <item name="android.permission.MANAGE_USERS" granted="true" flags="0" />
            <item name="android.permission.ACCESS_NETWORK_STATE" granted="true" flags="0" />
            <item name="android.permission.ACCESS_MTP" granted="true" flags="0" />
            <item name="android.permission.READ_LOGS" granted="true" flags="0" />
            <item name="android.permission.INTERACT_ACROSS_USERS" granted="true" flags="0" />
            <item name="android.permission.ACCESS_WIFI_STATE" granted="true" flags="0" />
            <item name="oppo.permission.OPPO_COMPONENT_SAFE" granted="true" flags="0" />
            <item name="android.permission.WAKE_LOCK" granted="true" flags="0" />
        </perms>
    </shared-user>
```
#### 8.1.3.1 Settings.readSharedUserLPw 
解析系统定义的共享用户 id：
```java
    private void readSharedUserLPw(XmlPullParser parser) throws XmlPullParserException,IOException {
        String name = null;
        String idStr = null;
        int pkgFlags = 0;
        int pkgPrivateFlags = 0;
        SharedUserSetting su = null;

        try {
            name = parser.getAttributeValue(null, ATTR_NAME); // 获得共享用户的名称
            idStr = parser.getAttributeValue(null, "userId");  // 获得共享用户的id
            int userId = idStr != null ? Integer.parseInt(idStr) : 0;

            if ("true".equals(parser.getAttributeValue(null, "system"))) { // 是否是系统的用户 id！
                pkgFlags |= ApplicationInfo.FLAG_SYSTEM;
            }

            if (name == null) {
                PackageManagerService.reportSettingsProblem(Log.WARN,
                        "Error in package manager settings: <shared-user> has no name at "
                                + parser.getPositionDescription());
            } else if (userId == 0) {
                PackageManagerService.reportSettingsProblem(Log.WARN,
                        "Error in package manager settings: shared-user " + name
                                + " has bad userId " + idStr + " at "
                                + parser.getPositionDescription());
            } else {

                //【8.1.3.2】调用 addSharedUserLPw 方法，将这个共享用户和对应的 uid 保存下来！
                if ((su = addSharedUserLPw(name.intern(), userId, pkgFlags, pkgPrivateFlags))
                        == null) {
                    PackageManagerService
                            .reportSettingsProblem(Log.ERROR, "Occurred while parsing settings at "
                                    + parser.getPositionDescription());
                }
            }
        } catch (NumberFormatException e) {
            PackageManagerService.reportSettingsProblem(Log.WARN,
                    "Error in package manager settings: package " + name + " has bad userId "
                            + idStr + " at " + parser.getPositionDescription());
        }

        if (su != null) { // 解析子标签！
            int outerDepth = parser.getDepth();
            int type;

            while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
                    && (type != XmlPullParser.END_TAG || parser.getDepth() > outerDepth)) {
                if (type == XmlPullParser.END_TAG || type == XmlPullParser.TEXT) {
                    continue;
                }

                String tagName = parser.getName();
                if (tagName.equals("sigs")) {
                    su.signatures.readXml(parser, mPastSignatures);
                    
                } else if (tagName.equals("perms")) { // 解析 "perms" 标签
                    //【8.1.1.3.1】解析共享用户的权限信息！
                    readInstallPermissionsLPr(parser, su.getPermissionsState());
                    
                } else {
                    PackageManagerService.reportSettingsProblem(Log.WARN,
                            "Unknown element under <shared-user>: " + parser.getName());
                    XmlUtils.skipCurrentTag(parser);

                }
            }
        } else {
            XmlUtils.skipCurrentTag(parser);
        }
    }
```
#### 8.1.3.2 Settings.addSharedUserLPw
```java
    SharedUserSetting addSharedUserLPw(String name, int uid, int pkgFlags, int pkgPrivateFlags) {
        // 创建共享用户对应的 SharedUserSetting 对象！
        SharedUserSetting s = mSharedUsers.get(name);
        if (s != null) {
            if (s.userId == uid) {
                return s;
            }
            PackageManagerService.reportSettingsProblem(Log.ERROR,
                    "Adding duplicate shared user, keeping first: " + name);
            return null;
        }
        s = new SharedUserSetting(name, pkgFlags, pkgPrivateFlags);
        s.userId = uid;
        
        //【8.1.1.2.1】根据 uid 的范围，保存到 mUsersId 和 mOthersId 中！
        if (addUserIdLPw(uid, s, name)) {
            // 将其添加到 mSharedUsers 集合中！
            mSharedUsers.put(name, s);
            return s;
        }
        return null;
    }

```
流程总结：

- 解析 "shared-user" ，将其分装成 SharedUserSetting 对象，保存到 mSharedUsers，并根据 uid 的取值将其保存到 mUserIds 或者 mOtherIds 中！
- 解析 "shared-user" 子标签 "perm" 等等，解析共享用户的权限，每个权限对应一个 PermissionData 对象，保存进入 PermisssionState 中，用于管理共享用户的权限！

### 8.1.4 解析 "preferred-activities" 相关标签

#### 8.1.4.1 Settings.readPreferredActivitiesLPw
```java
    void readPreferredActivitiesLPw(XmlPullParser parser, int userId)
            throws XmlPullParserException, IOException {
        int outerDepth = parser.getDepth();
        int type;
        while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
                && (type != XmlPullParser.END_TAG || parser.getDepth() > outerDepth)) {
            if (type == XmlPullParser.END_TAG || type == XmlPullParser.TEXT) {
                continue;
            }

            String tagName = parser.getName();
            if (tagName.equals(TAG_ITEM)) {
            
                // 创建 PreferredActivity 对象！
                PreferredActivity pa = new PreferredActivity(parser);
                if (pa.mPref.getParseError() == null) {
                    editPreferredActivitiesLPw(userId).addFilter(pa);
                } else {
                    PackageManagerService.reportSettingsProblem(Log.WARN,
                            "Error in package manager settings: <preferred-activity> "
                                    + pa.mPref.getParseError() + " at "
                                    + parser.getPositionDescription());
                }
            } else {
                PackageManagerService.reportSettingsProblem(Log.WARN,
                        "Unknown element under <preferred-activities>: " + parser.getName());
                XmlUtils.skipCurrentTag(parser);
            }
        }
    }
```
#### 8.1.4.2 Settings.readPersistentPreferredActivitiesLPw
```java
       private void readPersistentPreferredActivitiesLPw(XmlPullParser parser, int userId)
            throws XmlPullParserException, IOException {
        int outerDepth = parser.getDepth();
        int type;
        while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
                && (type != XmlPullParser.END_TAG || parser.getDepth() > outerDepth)) {
            if (type == XmlPullParser.END_TAG || type == XmlPullParser.TEXT) {
                continue;
            }
            String tagName = parser.getName();
            if (tagName.equals(TAG_ITEM)) {
            
                //【1】创建 PersistentPreferredActivity 对象！
                PersistentPreferredActivity ppa = new PersistentPreferredActivity(parser);
                editPersistentPreferredActivitiesLPw(userId).addFilter(ppa);
            } else {
                PackageManagerService.reportSettingsProblem(Log.WARN,
                        "Unknown element under <" + TAG_PERSISTENT_PREFERRED_ACTIVITIES + ">: "
                        + parser.getName());
                XmlUtils.skipCurrentTag(parser);
            }
        }
    }
```
不多说了！！

### 8.1.5 解析 "updated-package" 标签
我们来看看这个标签的内容，进行对比：
```xml
<package name="com.android.xxxx" 
         codePath="/data/app/com.android.xxxx-1" 
         nativeLibraryPath="/data/app/com.android.xxxx-2/lib" 
         publicFlags="944258757" 
         privateFlags="0" 
         ft="15920cea638" it="15920cea638" ut="15920cea638" version="1000" userId="10053"  
         installer="com.android.packageinstaller">
         <perms>
            <!---->
         </perms>
</package>

<updated-package name="com.android.xxxx" 
         codePath="/data/app/com.android.xxxx-1" 
         ft="158ff005268" it="158ff005268" ut="158ff005268" version="845"         
         nativeLibraryPath="/data/app/com.android.xxxx-1/lib" userId="10053">
         <perms>
            <!---->
         </perms>
</updated-package>
```
接下里，我们去看看解析过程：

#### 8.1.5.1 Settings.readDisabledSysPackageLPw
```java
    private void readDisabledSysPackageLPw(XmlPullParser parser) throws XmlPullParserException,
            IOException {
        String name = parser.getAttributeValue(null, ATTR_NAME);
        String realName = parser.getAttributeValue(null, "realName");
        String codePathStr = parser.getAttributeValue(null, "codePath");
        String resourcePathStr = parser.getAttributeValue(null, "resourcePath");

        String legacyCpuAbiStr = parser.getAttributeValue(null, "requiredCpuAbi");
        String legacyNativeLibraryPathStr = parser.getAttributeValue(null, "nativeLibraryPath");

        String parentPackageName = parser.getAttributeValue(null, "parentPackageName");

        String primaryCpuAbiStr = parser.getAttributeValue(null, "primaryCpuAbi");
        String secondaryCpuAbiStr = parser.getAttributeValue(null, "secondaryCpuAbi");
        String cpuAbiOverrideStr = parser.getAttributeValue(null, "cpuAbiOverride");

        if (primaryCpuAbiStr == null && legacyCpuAbiStr != null) {
            primaryCpuAbiStr = legacyCpuAbiStr;
        }

        if (resourcePathStr == null) {
            resourcePathStr = codePathStr;
        }
        String version = parser.getAttributeValue(null, "version");
        int versionCode = 0;
        if (version != null) {
            try {
                versionCode = Integer.parseInt(version);
            } catch (NumberFormatException e) {
            }
        }

        int pkgFlags = 0;
        int pkgPrivateFlags = 0;
        pkgFlags |= ApplicationInfo.FLAG_SYSTEM;
        final File codePathFile = new File(codePathStr);
        if (PackageManagerService.locationIsPrivileged(codePathFile)) {
            pkgPrivateFlags |= ApplicationInfo.PRIVATE_FLAG_PRIVILEGED;
        }
        
        // 封装成 PackageSetting 对象
        PackageSetting ps = new PackageSetting(name, realName, codePathFile,
                new File(resourcePathStr), legacyNativeLibraryPathStr, primaryCpuAbiStr,
                secondaryCpuAbiStr, cpuAbiOverrideStr, versionCode, pkgFlags, pkgPrivateFlags,
                parentPackageName, null);
                
        // 获得时间戳，第一次安装时间，最后一次更新时间，省略！
        ... ... ... ...
 
        // 处理其独立 uid 或者共享 uid 信息！
        String idStr = parser.getAttributeValue(null, "userId");
        ps.appId = idStr != null ? Integer.parseInt(idStr) : 0;
        if (ps.appId <= 0) {
            String sharedIdStr = parser.getAttributeValue(null, "sharedUserId");
            ps.appId = sharedIdStr != null ? Integer.parseInt(sharedIdStr) : 0;
        }

        int outerDepth = parser.getDepth();
        int type;
        while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
                && (type != XmlPullParser.END_TAG || parser.getDepth() > outerDepth)) {
            if (type == XmlPullParser.END_TAG || type == XmlPullParser.TEXT) {
                continue;
            }

            if (parser.getName().equals(TAG_PERMISSIONS)) { // 解析 "perms" 标签！
                // ps.getPermissionsState() 是 package 的权限管理对象！
                readInstallPermissionsLPr(parser, ps.getPermissionsState());
                
            } else if (parser.getName().equals(TAG_CHILD_PACKAGE)) {
                String childPackageName = parser.getAttributeValue(null, ATTR_NAME);
                if (ps.childPackageNames == null) {
                    ps.childPackageNames = new ArrayList<>();
                }
                ps.childPackageNames.add(childPackageName);
            } else {
                PackageManagerService.reportSettingsProblem(Log.WARN,
                        "Unknown element under <updated-package>: " + parser.getName());
                XmlUtils.skipCurrentTag(parser);
            }
        }
        
        //【1】保存到 mDisabledSysPackages 中！
        mDisabledSysPackages.put(name, ps);
    }

```


### 8.1.6 解析 "cleaning-package" 标签

```java
                } else if (tagName.equals("cleaning-package")) {
                    String name = parser.getAttributeValue(null, ATTR_NAME);
                    String userStr = parser.getAttributeValue(null, ATTR_USER);
                    String codeStr = parser.getAttributeValue(null, ATTR_CODE);
                    if (name != null) {
                        int userId = UserHandle.USER_SYSTEM;
                        boolean andCode = true;
                        try {
                            if (userStr != null) {
                                userId = Integer.parseInt(userStr);
                            }
                        } catch (NumberFormatException e) {
                        }
                        if (codeStr != null) {
                            andCode = Boolean.parseBoolean(codeStr);
                        }
                        // 创建一个 PackageCleanItem 对象！
                        addPackageToCleanLPw(new PackageCleanItem(userId, name, andCode));
                    }
                } 
            
```
调用 addPackageToCleanLPw 方法；
```java
    void addPackageToCleanLPw(PackageCleanItem pkg) {
        if (!mPackagesToBeCleaned.contains(pkg)) {
            
            //【1】添加到 mPackagesToBeCleaned！
            mPackagesToBeCleaned.add(pkg);
        }
    }
```
这个方法很简单！

### 8.1.7 解析 "renamed-package" 标签
```java
                else if (tagName.equals("renamed-package")) {
                    String nname = parser.getAttributeValue(null, "new");
                    String oname = parser.getAttributeValue(null, "old");
                    if (nname != null && oname != null) {
                        mRenamedPackages.put(nname, oname);
                    }
                }
```
代码很简单，将重新命名的应用的数据，key 为 新名字。value 为旧名字，添加到 mRenamedPackages 中！

### 8.1.8 阶段总结

我们来用一张图总结一下，PermissionsState 和 PackageSetting，SharedUserSetting 的关系！
![未命名文件.png-59.5kB][3]

- 如果 pacakge 是共享用户 id，那么所有 uid 为共享用户 id 的 package，其权限是一样的，都是共享用户的权限;
- 如果 pacakge 是独立用户 id，那么这个 package 有自己独立的权限！

## 8.2 确定共享 uid 有效性

主要逻辑如下：
```java
        //【1】对 mPendingPackages 集合中的需要验证共享用户 id 有效性的 package，进行共享用户 id 有效性的验证!
        final int N = mPendingPackages.size();
        for (int i = 0; i < N; i++) {
            final PendingPackage pp = mPendingPackages.get(i);

            //【8.2.1】看 sharedId 是否能对应找到一个 ShardUserSetting 对象!
            Object idObj = getUserIdLPr(pp.sharedId);
            if (idObj != null && idObj instanceof SharedUserSetting) { // 能找到，说明共享用户 ID 是有效的!
            
                //【8.2.2】创建 PackageSetting！
                PackageSetting p = getPackageLPw(pp.name, null, pp.realName,
                        (SharedUserSetting) idObj, pp.codePath, pp.resourcePath,
                        pp.legacyNativeLibraryPathString, pp.primaryCpuAbiString,
                        pp.secondaryCpuAbiString, pp.versionCode, pp.pkgFlags, pp.pkgPrivateFlags,
                        null, true /* add */, false /* allowInstall */, pp.parentPackageName,
                        pp.childPackageNames);

                if (p == null) {
                    PackageManagerService.reportSettingsProblem(Log.WARN,
                            "Unable to create application package for " + pp.name);
                    continue;
                }
                
                // 将 PendingPackage 的其他数据拷贝到 PackageSetting 中！
                p.copyFrom(pp); 

            } else if (idObj != null) {
                String msg = "Bad package setting: package " + pp.name + " has shared uid "
                        + pp.sharedId + " that is not a shared uid\n";
                mReadMessages.append(msg);
                PackageManagerService.reportSettingsProblem(Log.ERROR, msg);

            } else {
                String msg = "Bad package setting: package " + pp.name + " has shared uid "
                        + pp.sharedId + " that is not defined\n";
                mReadMessages.append(msg);
                PackageManagerService.reportSettingsProblem(Log.ERROR, msg);

            }
        }
        mPendingPackages.clear(); // 清空 mPendingPackages
```
这里调用 getUserIdLPr 来从 mUserIds 或 mOtherUserIds 中获得一个共享用户对象！

### 8.2.1 Settings.getUserIdLPr
获得指定 uid 对应的 Object，可能是一个 PackageSetting 或者是 SharedUserSetting！
```java
    public Object getUserIdLPr(int uid) {
        if (uid >= Process.FIRST_APPLICATION_UID) {
            final int N = mUserIds.size();
            final int index = uid - Process.FIRST_APPLICATION_UID;
            return index < N ? mUserIds.get(index) : null;

        } else {
            return mOtherUserIds.get(uid);

        }
    }

```
之前在分析解析 "shared-user" 的时候，就知道 SharedUserSetting 会被保存到这两个几个当中的，这里就不多说了！

接着，调用 getPackageLPw 方法来创建这个 PendingPackage 对应的 PackageSetting 对象！


### 8.2.2 Settings.getPackageLPw


参数传递：

- PackageSetting origPackage：表示源包，传入 null；
- SharedUserSetting sharedUser：共享 uid 对象，这里是不为 null！
- UserHandle installUser：为 null；
- add：true；

```java
    private PackageSetting getPackageLPw(String name, PackageSetting origPackage,
            String realName, SharedUserSetting sharedUser, File codePath, File resourcePath,
            String legacyNativeLibraryPathString, String primaryCpuAbiString,
            String secondaryCpuAbiString, int vc, int pkgFlags, int pkgPrivateFlags,
            UserHandle installUser, boolean add, boolean allowInstall, String parentPackage,
            List<String> childPackageNames) {

        PackageSetting p = mPackages.get(name); //【1】看是否之前有添加过这个包！
        UserManagerService userManager = UserManagerService.getInstance();

        if (p != null) { 
            ... ... ...// 如果已经添加了，这显然是不可能的，这里不会走这个分支！
        }

        if (p == null) {  //【2】之前没有添加过，那就需要创建新的 PackageSetting 对象!
        
            if (origPackage != null) { 
            
               ... ... ... ... ...// 如果有原包的话，这里不进入这个发分支！
                
            } else {
            
                //【3】如果这个 package 没有原包的话，就以参数 name 为包名，创建 PackageSetting 对象。
                p = new PackageSetting(name, realName, codePath, resourcePath,
                        legacyNativeLibraryPathString, primaryCpuAbiString, secondaryCpuAbiString,
                        null /* cpuAbiOverrideString */, vc, pkgFlags, pkgPrivateFlags,
                        parentPackage, childPackageNames);

                p.setTimeStamp(codePath.lastModified());

                //【4】设置其 sharedUser 属性，因为它是共享 uid 的！
                p.sharedUser = sharedUser;

                //【5】如果是非系统的应用，设置他在每个设备用户下的安装状态！
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
                
                //【6】设置他的 appId，在默认设备用户下，appId 等于 uid！
                if (sharedUser != null) {
                    p.appId = sharedUser.userId;

                } else {

                    ... ... ... ...// 根据前面的分析，显然 sharedUser 不为 null，不进入这个分支！
                
                }
     
            }

            if (p.appId < 0) {
                PackageManagerService.reportSettingsProblem(Log.WARN,
                        "Package " + name + " could not be assigned a valid uid");
                return null;
            }

            if (add) { 

                //【7】add 为 true，进入这个分支：
                // 将确定了共享用户 id 的 PackageSetting 对象，添加到 mPackages 中！
                addPackageSettingLPw(p, name, sharedUser);
            }
        } else {
     
            ... ... ... // 这个分支也不走！
        }

        return p;
    }

```
继续看：
#### 8.2.2.1 Settings.addPackageSettingLPw

将确定了共享用户 id 的 PackageSetting 对象，添加到 mPackages 中！

```java
    private void addPackageSettingLPw(PackageSetting p, String name, SharedUserSetting sharedUser) {
        //【1】添加到 mPackages 中！
        mPackages.put(name, p);

        if (sharedUser != null) { // 进入这里！
            if (p.sharedUser != null && p.sharedUser != sharedUser) {
                PackageManagerService.reportSettingsProblem(Log.ERROR,
                        "Package " + p.name + " was user "
                        + p.sharedUser + " but is now " + sharedUser
                        + "; I am not changing its files so it will probably fail!");
                p.sharedUser.removePackage(p);

            } else if (p.appId != sharedUser.userId) {
                PackageManagerService.reportSettingsProblem(Log.ERROR,
                    "Package " + p.name + " was user id " + p.appId
                    + " but is now user " + sharedUser
                    + " with id " + sharedUser.userId
                    + "; I am not changing its files so it will probably fail!");
            }
            
            // 确保 sharedUser 和 PackageSetting 相互引用正确！
            sharedUser.addPackage(p);
            p.sharedUser = sharedUser;
            p.appId = sharedUser.userId;
        }

        //【2】获得这个 package 对应的 SharedUserSetting 对象！
        Object userIdPs = getUserIdLPr(p.appId);

        if (sharedUser == null) {
            //【2.1】如果 sharedUser 为 null，说明共享用户无效，那就将创建的 PackageSetting 添加进来！
            if (userIdPs != null && userIdPs != p) {
                replaceUserIdLPw(p.appId, p);
            }
        } else {
            //【2.2】进入这个分支：
            if (userIdPs != null && userIdPs != sharedUser) {
                // 更新 uid 和 SharedUserSetting 的关系！
                replaceUserIdLPw(p.appId, sharedUser);
            }
        }

        IntentFilterVerificationInfo ivi = mRestoredIntentFilterVerifications.get(name);
        if (ivi != null) {
            if (DEBUG_DOMAIN_VERIFICATION) {
                Slog.i(TAG, "Applying restored IVI for " + name + " : " + ivi.getStatusString());
            }
            mRestoredIntentFilterVerifications.remove(name);
            p.setIntentFilterVerificationInfo(ivi);
        }
    }
```
下面是 replaceUserIdLPw 方法：
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

## 8.3 读取 packages_stopped.xml 文件，偏好设置

代码段如下：

```java
        if (mBackupStoppedPackagesFilename.exists()
                || mStoppedPackagesFilename.exists()) {
            // Read old file
            readStoppedLPw();
            mBackupStoppedPackagesFilename.delete();
            mStoppedPackagesFilename.delete();
            // Migrate to new file format
            writePackageRestrictionsLPr(UserHandle.USER_SYSTEM);
        } else {
            for (UserInfo user : users) {
                readPackageRestrictionsLPr(user.id);
            }
        }
```

读取 packages_stopped.xml 文件调用的是 readStoppedLPw 方法：  
### 8.3.1 Settings.readStoppedLPw
```java
    void readStoppedLPw() {
        FileInputStream str = null;
        
        //【1】如果 packages-stopped-backup.xml 文件存在，就读取它！
        if (mBackupStoppedPackagesFilename.exists()) {
            try {
                str = new FileInputStream(mBackupStoppedPackagesFilename);
                mReadMessages.append("Reading from backup stopped packages file\n");
                PackageManagerService.reportSettingsProblem(Log.INFO,
                        "Need to read from backup stopped packages file");
                if (mSettingsFilename.exists()) {
                    Slog.w(PackageManagerService.TAG, "Cleaning up stopped packages file "
                            + mStoppedPackagesFilename);
                            
                    //【1.1】如果，两个文件都存在，使用 back_up 文件，删掉 packages-stopped.xml 文件！
                    mStoppedPackagesFilename.delete();
                }
            } catch (java.io.IOException e) {
                // We'll try for the normal settings file.
            }
        }

        //【2】如果 packages-stopped-backup.xml 文件不存在，就读取 packages-stopped.xml 文件！
        try {
            if (str == null) {
                if (!mStoppedPackagesFilename.exists()) {
                    ... ... ...// log
                    
                    //【2.1】如果  packages-stopped.xml 也不存在，说明没有 package 处于 stop 状态；
                    // 同时，第一次开机，所有的 package 也都不会处于 stop 状态！
                    for (PackageSetting pkg : mPackages.values()) {
                        pkg.setStopped(false, 0); // 没有被 stop！
                        pkg.setNotLaunched(false, 0); // 也没有被启动过！
                    }
                    return;
                }
                str = new FileInputStream(mStoppedPackagesFilename);
            }
            final XmlPullParser parser = Xml.newPullParser();
            parser.setInput(str, null);

            int type;
            while ((type=parser.next()) != XmlPullParser.START_TAG
                       && type != XmlPullParser.END_DOCUMENT) {
                ;
            }

            if (type != XmlPullParser.START_TAG) {
                mReadMessages.append("No start tag found in stopped packages file\n");
                PackageManagerService.reportSettingsProblem(Log.WARN,
                        "No start tag found in package manager stopped packages");
                return;
            }
            //【3】开始解析！
            int outerDepth = parser.getDepth();
            while ((type=parser.next()) != XmlPullParser.END_DOCUMENT
                   && (type != XmlPullParser.END_TAG
                           || parser.getDepth() > outerDepth)) {
                if (type == XmlPullParser.END_TAG
                        || type == XmlPullParser.TEXT) {
                    continue;
                }

                String tagName = parser.getName();

                //【3.1】解析 stop 的 package 数据！
                if (tagName.equals(TAG_PACKAGE)) {
                    String name = parser.getAttributeValue(null, ATTR_NAME); // 解析 "name" 标签
                    
                    // 从之前安装的 mPackages 中，获得包名为 name 的 package 数据！
                    PackageSetting ps = mPackages.get(name);
                    if (ps != null) {
                        //【3.1.1】设置 stopped 属性！
                        ps.setStopped(true, 0);
                        if ("1".equals(parser.getAttributeValue(null, ATTR_NOT_LAUNCHED))) { // 解析 "nl" 标签
                            //【3.1.2】设置 notLaunched 属性！
                            ps.setNotLaunched(true, 0);
                        }
                    } else {
                        Slog.w(PackageManagerService.TAG,
                                "No package known for stopped package " + name);
                    }
                    XmlUtils.skipCurrentTag(parser);
                } else {
                    Slog.w(PackageManagerService.TAG, "Unknown element under <stopped-packages>: "
                          + parser.getName());
                    XmlUtils.skipCurrentTag(parser);
                }
            }
            str.close();

        } catch (XmlPullParserException e) {
            ... ... ... ...

        } catch (java.io.IOException e) {
            ... ... ... ...

        }
    }

```
这个过程很简单，解析 packages-stopped-backup.xml 或 packages-stopped.xml 文件，设置 package 的 stopped 和 notLaunched 属性！

### 8.3.2 Settings.writePackageRestrictionsLPr

writePackageRestrictionsLPr 方法用户保存用户对于系统应用的一些偏好设置，通过获得 package 在指定用户下的状态信息，我们就知道那些应用被 stop，那些应用被 hidden 等等！

```java
    void writePackageRestrictionsLPr(int userId) {
        if (DEBUG_MU) {
            Log.i(TAG, "Writing package restrictions for user=" + userId);
        }

        // 获得 data/system/users/0/package-restrictions.xml 文件对象！
        File userPackagesStateFile = getUserPackagesStateFile(userId);
        // 获得 data/system/users/0/package-restrictions-backup.xml 文件对象！
        File backupFile = getUserPackagesStateBackupFile(userId);
        new File(userPackagesStateFile.getParent()).mkdirs();
        
        // 如果备份文件不存在的话，那么我们会现将 package-restrictions.xml 备份再操作！
        if (userPackagesStateFile.exists()) {
            if (!backupFile.exists()) {
                if (!userPackagesStateFile.renameTo(backupFile)) {
                    Slog.wtf(PackageManagerService.TAG,
                            "Unable to backup user packages state file, "
                            + "current changes will be lost at reboot");
                    return;
                }
            } else {
                userPackagesStateFile.delete();
                Slog.w(PackageManagerService.TAG, "Preserving older stopped packages backup");
            }
        }
        
        // 开始解析 xml 文件！
        try {
            final FileOutputStream fstr = new FileOutputStream(userPackagesStateFile);
            final BufferedOutputStream str = new BufferedOutputStream(fstr);

            final XmlSerializer serializer = new FastXmlSerializer();
            serializer.setOutput(str, StandardCharsets.UTF_8.name());
            serializer.startDocument(null, true);
            serializer.setFeature("http://xmlpull.org/v1/doc/features.html#indent-output", true);

            serializer.startTag(null, TAG_PACKAGE_RESTRICTIONS); // "package-restrictions" 标签

            for (final PackageSetting pkg : mPackages.values()) {
                final PackageUserState ustate = pkg.readUserState(userId);
                if (DEBUG_MU) Log.i(TAG, "  pkg=" + pkg.name + ", state=" + ustate.enabled);

                serializer.startTag(null, TAG_PACKAGE); // pkg 标签
                serializer.attribute(null, ATTR_NAME, pkg.name); // name 属性
                if (ustate.ceDataInode != 0) {
                    XmlUtils.writeLongAttribute(serializer, ATTR_CE_DATA_INODE, ustate.ceDataInode);
                }
                if (!ustate.installed) { 
                    serializer.attribute(null, ATTR_INSTALLED, "false"); // inst 属性
                }
                if (ustate.stopped) {
                    serializer.attribute(null, ATTR_STOPPED, "true"); // stopped 属性
                }
                if (ustate.notLaunched) {
                    serializer.attribute(null, ATTR_NOT_LAUNCHED, "true"); // nl 属性
                }
                if (ustate.hidden) {
                    serializer.attribute(null, ATTR_HIDDEN, "true"); // hidden 属性
                }
                if (ustate.suspended) {
                    serializer.attribute(null, ATTR_SUSPENDED, "true"); // suspended 属性
                }
                if (ustate.blockUninstall) {
                    serializer.attribute(null, ATTR_BLOCK_UNINSTALL, "true"); // blockUninstall 属性
                }
                if (ustate.enabled != COMPONENT_ENABLED_STATE_DEFAULT) { // 包默认是否可用
                    serializer.attribute(null, ATTR_ENABLED, // enabled 属性
                            Integer.toString(ustate.enabled));
                    if (ustate.lastDisableAppCaller != null) {
                        serializer.attribute(null, ATTR_ENABLED_CALLER, // enabledCaller 属性
                                ustate.lastDisableAppCaller);
                    }
                }
                if (ustate.domainVerificationStatus !=
                        PackageManager.INTENT_FILTER_DOMAIN_VERIFICATION_STATUS_UNDEFINED) {
                    XmlUtils.writeIntAttribute(serializer, ATTR_DOMAIN_VERIFICATON_STATE,
                            ustate.domainVerificationStatus);
                }
                if (ustate.appLinkGeneration != 0) {
                    XmlUtils.writeIntAttribute(serializer, ATTR_APP_LINK_GENERATION,
                            ustate.appLinkGeneration);
                }
                if (!ArrayUtils.isEmpty(ustate.enabledComponents)) { // 可用组件
                    serializer.startTag(null, TAG_ENABLED_COMPONENTS);
                    for (final String name : ustate.enabledComponents) {
                        serializer.startTag(null, TAG_ITEM);
                        serializer.attribute(null, ATTR_NAME, name);
                        serializer.endTag(null, TAG_ITEM);
                    }
                    serializer.endTag(null, TAG_ENABLED_COMPONENTS);
                }
                if (!ArrayUtils.isEmpty(ustate.disabledComponents)) { // 不可用组件
                    serializer.startTag(null, TAG_DISABLED_COMPONENTS);
                    for (final String name : ustate.disabledComponents) {
                        serializer.startTag(null, TAG_ITEM);
                        serializer.attribute(null, ATTR_NAME, name);
                        serializer.endTag(null, TAG_ITEM);
                    }
                    serializer.endTag(null, TAG_DISABLED_COMPONENTS);
                }

                serializer.endTag(null, TAG_PACKAGE);
            }
            
            // 接下来会处理默认应用的处理，这里先不看！
            writePreferredActivitiesLPr(serializer, userId, true);
            writePersistentPreferredActivitiesLPr(serializer, userId);
            writeCrossProfileIntentFiltersLPr(serializer, userId);
            writeDefaultAppsLPr(serializer, userId);

            serializer.endTag(null, TAG_PACKAGE_RESTRICTIONS);

            serializer.endDocument();

            str.flush();
            FileUtils.sync(fstr);
            str.close();

            // 删除备份文件，并设置主文件的权限！
            backupFile.delete();
            FileUtils.setPermissions(userPackagesStateFile.toString(),
                    FileUtils.S_IRUSR|FileUtils.S_IWUSR
                    |FileUtils.S_IRGRP|FileUtils.S_IWGRP,
                    -1, -1);

            // Done, all is good!
            return;
        } catch(java.io.IOException e) {
            Slog.wtf(PackageManagerService.TAG,
                    "Unable to write package manager user packages state, "
                    + " current changes will be lost at reboot", e);
        }

        // 异常情况，清除偏好设置文件！
        if (userPackagesStateFile.exists()) {
            if (!userPackagesStateFile.delete()) {
                Log.i(PackageManagerService.TAG, "Failed to clean up mangled file: "
                        + mStoppedPackagesFilename);
            }
        }
    }

```
我们来看下偏好设置的 xml 的内容！

```java
<?xml version='1.0' encoding='utf-8' standalone='yes' ?>
<package-restrictions>
    <pkg name="com.papapa.activation" ceDataInode="3940">
        <disabled-components>
            <item name="com.papapa.activation.BootReceiver" />
        </disabled-components>
    </pkg>

    <preferred-activities>
        <item name="com.android.mms/.ui.ComposeMessageActivity" match="200000" always="true" set="1">
            <set name="com.android.mms/.ui.ComposeMessageActivity" />
            <filter>
                <action name="android.intent.action.SENDTO" />
                <cat name="android.intent.category.DEFAULT" />
                <scheme name="mmsto" />
            </filter>
        </item>
    </preferred-activities>
    <persistent-preferred-activities />
    <crossProfile-intent-filters />
    <default-apps>
        <default-dialer packageName="com.android.contacts" />
    </default-apps>
</package-restrictions>
```
这里只列出了一部分的内容，其他的很类似，不再多说！


### 8.3.3 Settings.readPackageRestrictionsLPr

该方法用于读取已有的偏好设置，更新 package 安装的信息！
```java
    void readPackageRestrictionsLPr(int userId) {
        if (DEBUG_MU) {
            Log.i(TAG, "Reading package restrictions for user=" + userId);
        }
        FileInputStream str = null;
        // 同样的，获得 data/system/users/0/package-restrictions.xml 和 
        // data/system/users/0/package-restrictions-backup.xml 文件对象！
        File userPackagesStateFile = getUserPackagesStateFile(userId);
        File backupFile = getUserPackagesStateBackupFile(userId);
        // 优先从 backupFile 读取，如果两个文件都存在，那就会删除非备份文件！
        if (backupFile.exists()) {
            try {
                str = new FileInputStream(backupFile);
                mReadMessages.append("Reading from backup stopped packages file\n");
                PackageManagerService.reportSettingsProblem(Log.INFO,
                        "Need to read from backup stopped packages file");
                if (userPackagesStateFile.exists()) {
                    Slog.w(PackageManagerService.TAG, "Cleaning up stopped packages file "
                            + userPackagesStateFile);
                    userPackagesStateFile.delete();
                }
            } catch (java.io.IOException e) {
                // We'll try for the normal settings file.
            }
        }

        try {
            if (str == null) {
                if (!userPackagesStateFile.exists()) {
                    mReadMessages.append("No stopped packages file found\n");
                    PackageManagerService.reportSettingsProblem(Log.INFO,
                            "No stopped packages file; "
                            + "assuming all started");
                    // 对于第一次启动，这里会初始化操作，并返回！！
                    for (PackageSetting pkg : mPackages.values()) {
                        pkg.setUserState(userId, 0, COMPONENT_ENABLED_STATE_DEFAULT,
                                true,   // installed
                                false,  // stopped
                                false,  // notLaunched
                                false,  // hidden
                                false,  // suspended
                                null, null, null,
                                false, // blockUninstall
                                INTENT_FILTER_DOMAIN_VERIFICATION_STATUS_UNDEFINED, 0);
                    }
                    return;
                }
                str = new FileInputStream(userPackagesStateFile);
            }
            final XmlPullParser parser = Xml.newPullParser();
            parser.setInput(str, StandardCharsets.UTF_8.name());

            int type;
            while ((type=parser.next()) != XmlPullParser.START_TAG
                       && type != XmlPullParser.END_DOCUMENT) {
                ;
            }

            if (type != XmlPullParser.START_TAG) {
                mReadMessages.append("No start tag found in package restrictions file\n");
                PackageManagerService.reportSettingsProblem(Log.WARN,
                        "No start tag found in package manager stopped packages");
                return;
            }

            int maxAppLinkGeneration = 0;

            int outerDepth = parser.getDepth();
            PackageSetting ps = null;
            while ((type=parser.next()) != XmlPullParser.END_DOCUMENT
                   && (type != XmlPullParser.END_TAG
                           || parser.getDepth() > outerDepth)) {
                if (type == XmlPullParser.END_TAG
                        || type == XmlPullParser.TEXT) {
                    continue;
                }
                
                //【1】解析 pkg 标签的属性！
                String tagName = parser.getName();
                if (tagName.equals(TAG_PACKAGE)) {
                    String name = parser.getAttributeValue(null, ATTR_NAME);
                    ps = mPackages.get(name);
                    if (ps == null) {
                        Slog.w(PackageManagerService.TAG, "No package known for stopped package "
                                + name);
                        XmlUtils.skipCurrentTag(parser);
                        continue;
                    }

                    final long ceDataInode = XmlUtils.readLongAttribute(parser, ATTR_CE_DATA_INODE,
                            0);
                    final boolean installed = XmlUtils.readBooleanAttribute(parser, ATTR_INSTALLED,
                            true);
                    final boolean stopped = XmlUtils.readBooleanAttribute(parser, ATTR_STOPPED,
                            false);
                    final boolean notLaunched = XmlUtils.readBooleanAttribute(parser,
                            ATTR_NOT_LAUNCHED, false);

                    // For backwards compatibility with the previous name of "blocked", which
                    // now means hidden, read the old attribute as well.
                    final String blockedStr = parser.getAttributeValue(null, ATTR_BLOCKED);
                    boolean hidden = blockedStr == null
                            ? false : Boolean.parseBoolean(blockedStr);
                    final String hiddenStr = parser.getAttributeValue(null, ATTR_HIDDEN);
                    hidden = hiddenStr == null
                            ? hidden : Boolean.parseBoolean(hiddenStr);

                    final boolean suspended = XmlUtils.readBooleanAttribute(parser, ATTR_SUSPENDED,
                            false);
                    final boolean blockUninstall = XmlUtils.readBooleanAttribute(parser,
                            ATTR_BLOCK_UNINSTALL, false);
                    final int enabled = XmlUtils.readIntAttribute(parser, ATTR_ENABLED,
                            COMPONENT_ENABLED_STATE_DEFAULT);
                    final String enabledCaller = parser.getAttributeValue(null,
                            ATTR_ENABLED_CALLER);

                    final int verifState = XmlUtils.readIntAttribute(parser,
                            ATTR_DOMAIN_VERIFICATON_STATE,
                            PackageManager.INTENT_FILTER_DOMAIN_VERIFICATION_STATUS_UNDEFINED);
                    final int linkGeneration = XmlUtils.readIntAttribute(parser,
                            ATTR_APP_LINK_GENERATION, 0);
                    if (linkGeneration > maxAppLinkGeneration) {
                        maxAppLinkGeneration = linkGeneration;
                    }

                    ArraySet<String> enabledComponents = null;
                    ArraySet<String> disabledComponents = null;

                    int packageDepth = parser.getDepth();
                    while ((type=parser.next()) != XmlPullParser.END_DOCUMENT
                            && (type != XmlPullParser.END_TAG
                            || parser.getDepth() > packageDepth)) {
                        if (type == XmlPullParser.END_TAG
                                || type == XmlPullParser.TEXT) {
                            continue;
                        }
                        tagName = parser.getName();
                        if (tagName.equals(TAG_ENABLED_COMPONENTS)) {
                            enabledComponents = readComponentsLPr(parser);
                        } else if (tagName.equals(TAG_DISABLED_COMPONENTS)) {
                            disabledComponents = readComponentsLPr(parser);
                        }
                    }
                    //【2】通过解析到的偏好设置，来更新安装的 package 的信息！
                    ps.setUserState(userId, ceDataInode, enabled, installed, stopped, notLaunched,
                            hidden, suspended, enabledCaller, enabledComponents, disabledComponents,
                            blockUninstall, verifState, linkGeneration);
                } else if (tagName.equals("preferred-activities")) { // 解析默认应用的属性！
                    readPreferredActivitiesLPw(parser, userId);
                } else if (tagName.equals(TAG_PERSISTENT_PREFERRED_ACTIVITIES)) {
                    readPersistentPreferredActivitiesLPw(parser, userId);
                } else if (tagName.equals(TAG_CROSS_PROFILE_INTENT_FILTERS)) {
                    readCrossProfileIntentFiltersLPw(parser, userId);
                } else if (tagName.equals(TAG_DEFAULT_APPS)) {
                    readDefaultAppsLPw(parser, userId);
                } else {
                    Slog.w(PackageManagerService.TAG, "Unknown element under <stopped-packages>: "
                          + parser.getName());
                    XmlUtils.skipCurrentTag(parser);
                }
            }

            str.close();

            mNextAppLinkGeneration.put(userId, maxAppLinkGeneration + 1);

        } catch (XmlPullParserException e) {
            mReadMessages.append("Error reading: " + e.toString());
            PackageManagerService.reportSettingsProblem(Log.ERROR,
                    "Error reading stopped packages: " + e);
            Slog.wtf(PackageManagerService.TAG, "Error reading package manager stopped packages",
                    e);

        } catch (java.io.IOException e) {
            mReadMessages.append("Error reading: " + e.toString());
            PackageManagerService.reportSettingsProblem(Log.ERROR, "Error reading settings: " + e);
            Slog.wtf(PackageManagerService.TAG, "Error reading package manager stopped packages",
                    e);
        }
    }
```
逻辑很简单，我们就不多说了！

## 8.4 读取运行时权限信息，并处理授予情况！

```java
        //【8.4】读取每个 user 下的运行是权限信息！
        for (UserInfo user : users) {
            mRuntimePermissionsPersistence.readStateForUserSyncLPr(user.id);
        }
```
mRuntimePermissionsPersistence 对象我们前面介绍过，其专门用于处理运行时权限！

### 8.4.1 RuntimePermissionsPersistence.readStateForUserSyncLPr

该方法会读取指定的 userId 下的运行时权限信息！

```java
        public void readStateForUserSyncLPr(int userId) {
            //【1】获得运行时权限文件对象！
            File permissionsFile = getUserRuntimePermissionsFile(userId);
            if (!permissionsFile.exists()) {
                return;
            }

            FileInputStream in;
            try {
                in = new AtomicFile(permissionsFile).openRead();
            } catch (FileNotFoundException fnfe) {
                Slog.i(PackageManagerService.TAG, "No permissions state");
                return;
            }

            try {
                XmlPullParser parser = Xml.newPullParser();
                parser.setInput(in, null);
                // 解析运行时权限数据！
                parseRuntimePermissionsLPr(parser, userId);

            } catch (XmlPullParserException | IOException e) {
                throw new IllegalStateException("Failed parsing permissions file: "
                        + permissionsFile , e);
            } finally {
                IoUtils.closeQuietly(in);
            }
        }
```
首先，会获得保存了运行时权限的文件对象！
```java
    private File getUserRuntimePermissionsFile(int userId) {
        File userDir = new File(new File(mSystemDir, "users"), Integer.toString(userId));
        return new File(userDir, RUNTIME_PERMISSIONS_FILE_NAME);
    }
```
该文件位于 /data/system/users/0/runtime-permissions.xml，和偏好设置的文件位于同一个目录下，我们来简单的看下该文件的具体内容：

```xml
<?xml version='1.0' encoding='UTF-8' standalone='yes' ?>
<runtime-permissions fingerprint="OPPO/R11s/R11s:8.1.0/OPM1.171019.011/1519198279:user/release-keys">
  <pkg name="com.sohu.inputmethod.sogouoem">
    <item name="android.permission.ACCESS_FINE_LOCATION" granted="true" flags="20" />
    <item name="android.permission.READ_EXTERNAL_STORAGE" granted="true" flags="20" />
    <item name="android.permission.ACCESS_COARSE_LOCATION" granted="true" flags="20" />
    ... ... ...
  </pkg>
  <shared-user name="android.uid.calendar">
    <item name="android.permission.READ_CALENDAR" granted="true" flags="20" />
    <item name="android.permission.ACCESS_FINE_LOCATION" granted="true" flags="20" />
    <item name="android.permission.READ_EXTERNAL_STORAGE" granted="true" flags="20" />
    <item name="android.permission.ACCESS_COARSE_LOCATION" granted="true" flags="20" />
    ... ... ...
</runtime-permissions>
```

### 8.4.2 Settings.parseRuntimePermissionsLPr

用于解析运行时权限文件，获得运行时权限相关的信息！
```java
        private void parseRuntimePermissionsLPr(XmlPullParser parser, int userId)
                throws IOException, XmlPullParserException {
            final int outerDepth = parser.getDepth();
            int type;
            while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
                    && (type != XmlPullParser.END_TAG || parser.getDepth() > outerDepth)) {
                if (type == XmlPullParser.END_TAG || type == XmlPullParser.TEXT) {
                    continue;
                }

                switch (parser.getName()) {
                    case TAG_RUNTIME_PERMISSIONS: { // runtime-permissions 标签
                        String fingerprint = parser.getAttributeValue(null, ATTR_FINGERPRINT); // fingerprint 属性
                        mFingerprints.put(userId, fingerprint); // 保存 fingerprint 到 mFingerprints 中！
                        // 然后判断是否对该设备用户授予默认的权限，添加到 mDefaultPermissionsGranted 中！
                        final boolean defaultsGranted = Build.FINGERPRINT.equals(fingerprint);
                        mDefaultPermissionsGranted.put(userId, defaultsGranted);
                    } break;

                    case TAG_PACKAGE: {  //【1】pkg 标签
                        String name = parser.getAttributeValue(null, ATTR_NAME);
                        PackageSetting ps = mPackages.get(name);
                        if (ps == null) {
                            Slog.w(PackageManagerService.TAG, "Unknown package:" + name);
                            XmlUtils.skipCurrentTag(parser);
                            continue;
                        }
                        //【8.4.2.1】解析并处理 package 的运行时权限授予情况！
                        parsePermissionsLPr(parser, ps.getPermissionsState(), userId);
                    } break;

                    case TAG_SHARED_USER: {   //【2】shared-user 标签
                        String name = parser.getAttributeValue(null, ATTR_NAME);
                        SharedUserSetting sus = mSharedUsers.get(name);
                        if (sus == null) {
                            Slog.w(PackageManagerService.TAG, "Unknown shared user:" + name);
                            XmlUtils.skipCurrentTag(parser);
                            continue;
                        }
                        //【8.4.2.1】解析并处理 shared-user 的运行时权限授予情况！
                        parsePermissionsLPr(parser, sus.getPermissionsState(), userId);
                    } break;

                    case TAG_RESTORED_RUNTIME_PERMISSIONS: { //【3】restored-perms 标签
                        //【3.1】解析要被恢复的权限所属的应用包名！
                        final String pkgName = parser.getAttributeValue(null, ATTR_PACKAGE_NAME);
                        //【8.4.2.2】解析 restored-perms!
                        parseRestoredRuntimePermissionsLPr(parser, pkgName, userId);
                    } break;
                }
            }
        }
```
这个过程主要工作：

- 解析并处理 package 的运行时权限授予情况；
- 解析并处理 shared-user 的运行时权限授予情况；


#### 8.4.2.1 Settings.parsePermissionsLPr

我们来看下 parsePermissionsLPr 是如何解析和处理运行时权限的！
```java
        private void parsePermissionsLPr(XmlPullParser parser, PermissionsState permissionsState,
                int userId) throws IOException, XmlPullParserException {
            final int outerDepth = parser.getDepth();
            int type;
            while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
                    && (type != XmlPullParser.END_TAG || parser.getDepth() > outerDepth)) {
                if (type == XmlPullParser.END_TAG || type == XmlPullParser.TEXT) {
                    continue;
                }

                switch (parser.getName()) {
                    case TAG_ITEM: {
                        String name = parser.getAttributeValue(null, ATTR_NAME); // name 属性！
                        BasePermission bp = mPermissions.get(name);
                        if (bp == null) {
                            Slog.w(PackageManagerService.TAG, "Unknown permission:" + name);
                            XmlUtils.skipCurrentTag(parser);
                            continue;
                        }

                        String grantedStr = parser.getAttributeValue(null, ATTR_GRANTED);  // granted 属性！
                        final boolean granted = grantedStr == null
                                || Boolean.parseBoolean(grantedStr);

                        String flagsStr = parser.getAttributeValue(null, ATTR_FLAGS); // flags 属性！
                        final int flags = (flagsStr != null)
                                ? Integer.parseInt(flagsStr, 16) : 0;

                        //【8.4.2.1.1】处理运行时权限的授予情况，这里和处理安装时权限很类似！
                        if (granted) {
                            // 如果上次安装时，该运行时权限处于授予状态，接着更新 flags！
                            permissionsState.grantRuntimePermission(bp, userId);
                            permissionsState.updatePermissionFlags(bp, userId,
                                        PackageManager.MASK_PERMISSION_FLAGS, flags);
                        } else {
                            // 如果上次安装时，该运行时权限处于未授予状态，只更新 flags！
                            permissionsState.updatePermissionFlags(bp, userId,
                                    PackageManager.MASK_PERMISSION_FLAGS, flags);
                        }

                    } break;
                }
            }
        }
```

我们看到，对于运行时权限已授予的情况，我们会先进行一次运行时权限授予，然后更新权限的 flags；对于运行时权限未授予的情况，只是更新 flags 即可！

##### 8.4.2.1.1 Permissions.grantRuntimePermission

处理运行时权限的授予！

```java
    public int grantRuntimePermission(BasePermission permission, int userId) {
        enforceValidUserId(userId);
        //【1】可以看到，对于运行时权限，userId 只能为当前设备用户，不能为 USER_ALL
        if (userId == UserHandle.USER_ALL) {
            return PERMISSION_OPERATION_FAILURE;
        }
        //【2】接下来，就和处理安装时权限一样了！
        return grantPermission(permission, userId);
    }
```
这里不多说！

#### 8.4.2.2 Settings.parseRestoredRuntimePermissionsLPr

解析那些需要恢复的权限信息：

```java
        private void parseRestoredRuntimePermissionsLPr(XmlPullParser parser,
                final String pkgName, final int userId) throws IOException, XmlPullParserException {
            final int outerDepth = parser.getDepth();
            int type;
            while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
                    && (type != XmlPullParser.END_TAG || parser.getDepth() > outerDepth)) {
                if (type == XmlPullParser.END_TAG || type == XmlPullParser.TEXT) {
                    continue;
                }

                switch (parser.getName()) {
                    //【1】解析 perm 标签！
                    case TAG_PERMISSION_ENTRY: {
                        //【1.1】解析 name 属性，获得权限的名称；
                        final String permName = parser.getAttributeValue(null, ATTR_NAME);
                        //【1.2】解析 granted 属性，获得权限的授予情况；
                        final boolean isGranted = "true".equals(
                                parser.getAttributeValue(null, ATTR_GRANTED));
                        //【1.3】解析获得权限的 flags 设置！
                        int permBits = 0;
                        if ("true".equals(parser.getAttributeValue(null, ATTR_USER_SET))) {
                            permBits |= FLAG_PERMISSION_USER_SET;
                        }
                        if ("true".equals(parser.getAttributeValue(null, ATTR_USER_FIXED))) {
                            permBits |= FLAG_PERMISSION_USER_FIXED;
                        }
                        if ("true".equals(parser.getAttributeValue(null, ATTR_REVOKE_ON_UPGRADE))) {
                            permBits |= FLAG_PERMISSION_REVOKE_ON_UPGRADE;
                        }
                        if (isGranted || permBits != 0) {
                            //【8.4.2.2.1】如果权限是授予状态，或者其 flags 不为 null！
                            // 那么我们会将其记录到内充中！
                            rememberRestoredUserGrantLPr(pkgName, permName, isGranted, permBits, userId);
                        }
                    } break;
                }
            }
        }
```
如果权限是被授予的状态，或者其 flags 不为 null，那么会调用 rememberRestoredUserGrantLPr 方法，保存到 Settings.mRestoredUserGrants 中！

#####8.4.2.2.1 Settings.rememberRestoredUserGrantLPr
```java
        public void rememberRestoredUserGrantLPr(String pkgName, String permission,
                boolean isGranted, int restoredFlagSet, int userId) {
            //【1】获得该 userId 下的 restore perms 数据集合，是一个 Map！
            // 如果为 null，就初始化！
            ArrayMap<String, ArraySet<RestoredPermissionGrant>> grantsByPackage =
                    mRestoredUserGrants.get(userId);
            if (grantsByPackage == null) {
                grantsByPackage = new ArrayMap<String, ArraySet<RestoredPermissionGrant>>();
                mRestoredUserGrants.put(userId, grantsByPackage);
            }
            //【2】获得该 package 的 restore perms 数据集合，如果为 null，就初始化！
            ArraySet<RestoredPermissionGrant> grants = grantsByPackage.get(pkgName);
            if (grants == null) {
                grants = new ArraySet<RestoredPermissionGrant>();
                grantsByPackage.put(pkgName, grants);
            }
            //【3】将 retore 的权限封装成一个 RestoredPermissionGrant，添加到集合中！
            RestoredPermissionGrant grant = new RestoredPermissionGrant(permission,
                    isGranted, restoredFlagSet);
            grants.add(grant);
        }
```

Settings 内部有一个 SparseArray：
```java
   private final SparseArray<ArrayMap<String, ArraySet<RestoredPermissionGrant>>>
            mRestoredUserGrants =
                new SparseArray<ArrayMap<String, ArraySet<RestoredPermissionGrant>>>();
```
下标是 userId，值是一个 ArrayMap<String, ArraySet<RestoredPermissionGrant>>，封装了该 userId 下所有 package的 retore perms！

```java
    final class RestoredPermissionGrant {
        String permissionName;
        boolean granted;
        int grantBits;

        RestoredPermissionGrant(String name, boolean isGranted, int theGrantBits) {
            permissionName = name;
            granted = isGranted;
            grantBits = theGrantBits;
        }
    }
```
RestoredPermissionGrant 内部的结构很简单，不多说了！！

# 9 安装时权限处理过程分析 - 接 8.1.1.4

markdown 只支持 6 级标题，所以这里接 8.1.1.4 继续分析！

下面来较为深入的分析下解析 "perms" 标签，并通过解析结果授予安装时权限的过程：

```java
   <perms>
        <item name="android.permission.REAL_GET_TASKS" granted="true" flags="0" />
        <item name="android.permission.RECEIVE_BOOT_COMPLETED" granted="true" flags="0" />
        <item name="android.permission.INTERNET" granted="true" flags="0" />
        <item name="android.permission.ACCESS_NETWORK_STATE" granted="true" flags="0" />
   </perms>
```

回顾一下，Settings.readInstallPermissionsLPr 的方法：

```java
    void readInstallPermissionsLPr(XmlPullParser parser, PermissionsState permissionsState) 
                                                        throws IOException, XmlPullParserException {
        int outerDepth = parser.getDepth();
        int type;
        while ((type=parser.next()) != XmlPullParser.END_DOCUMENT
                && (type != XmlPullParser.END_TAG
                || parser.getDepth() > outerDepth)) {
            if (type == XmlPullParser.END_TAG
                    || type == XmlPullParser.TEXT) {
                continue;
            }
            String tagName = parser.getName();
            if (tagName.equals(TAG_ITEM)) { // 解析 "item"
                String name = parser.getAttributeValue(null, ATTR_NAME); // 接续 "name"，权限名称！
                //【1】从之前解析的 Settings.mPermissions 获得对应的权限；
                BasePermission bp = mPermissions.get(name);
                if (bp == null) {
                    Slog.w(PackageManagerService.TAG, "Unknown permission: " + name);
                    XmlUtils.skipCurrentTag(parser);
                    continue;
                }
                //【2】解析 "granted" 标签，获得授予情况！
                String grantedStr = parser.getAttributeValue(null, ATTR_GRANTED);
                final boolean granted = grantedStr == null
                        || Boolean.parseBoolean(grantedStr);
                // 解析 "flags"标签
                String flagsStr = parser.getAttributeValue(null, ATTR_FLAGS);
                final int flags = (flagsStr != null)
                        ? Integer.parseInt(flagsStr, 16) : 0;

                //【3】处理权限授予的操作！
                if (granted) { 
                    //【9.1】处理默认授予的情况！
                    if (permissionsState.grantInstallPermission(bp) ==
                            PermissionsState.PERMISSION_OPERATION_FAILURE) {
                        Slog.w(PackageManagerService.TAG, "Permission already added: " + name);
                        XmlUtils.skipCurrentTag(parser);
                    } else {
                        permissionsState.updatePermissionFlags(bp, UserHandle.USER_ALL,
                                PackageManager.MASK_PERMISSION_FLAGS, flags);
                    }
                } else { 
                    //【9.2】处理默认不授予的情况！
                    if (permissionsState.revokeInstallPermission(bp) ==
                            PermissionsState.PERMISSION_OPERATION_FAILURE) {
                        Slog.w(PackageManagerService.TAG, "Permission already added: " + name);
                        XmlUtils.skipCurrentTag(parser);
                    } else {
                        //【9.3】更新应用权限信息！
                        permissionsState.updatePermissionFlags(bp, UserHandle.USER_ALL,
                                PackageManager.MASK_PERMISSION_FLAGS, flags);
                    }
                }
            } else {
                Slog.w(PackageManagerService.TAG, "Unknown element under <permissions>: "
                        + parser.getName());
                XmlUtils.skipCurrentTag(parser);
            }
        }
    }
```

接下来，我们看一看，这个流程是如何处理权限的：

这里有涉及到 permissionsState 对象，每一个 PackageSetting 和 SharedUserSetting 都有一个 permissionsState 对象，用来保存其权限的状态信息！

在创建 PackageSetting 和 SharedUserSetting 的时候，都会默认创建一个 PermissionsState 对象！

PermissionsState 内部有一个 ArrayMap<String, PermissionData> mPermissions 用来保存权限名和其对应权限信息数据！

下面为了简化分析过程，PermissionsState 统一化为 PermissionsS

## 9.1 PermissionsS.grantInstallPermission - 处理授予情况

```java
    // 授予一个安装时权限，可以看到，安装时权限是对所有用户都默认授予的！
    public int grantInstallPermission(BasePermission permission) {
        return grantPermission(permission, UserHandle.USER_ALL);
    }

```
我们继续看，参数传入：

- **BasePermission permission**：权限封装对象
- **int userId**：设备用户，因为是安装时权限，所以对所有设备用户都是一样，这里传入 **UserHandle.USER_ALL**！
```java
    private int grantPermission(BasePermission permission, int userId) {
        //【9.1.1】如果之前有权限，本次授予失败！
        if (hasPermission(permission.name, userId)) {
            return PERMISSION_OPERATION_FAILURE;
        }
        
        //【9.1.2】判断权限 BasePermission 在 userId 下是否有映射 gid，如果有，那 hasGids 为 true！
        final boolean hasGids = !ArrayUtils.isEmpty(permission.computeGids(userId));

        //【9.1.3】如果权限 BasePermission 有映射的 gid，那就计算下该 package 的所属的 Gid 组！
        final int[] oldGids = hasGids ? computeGids(userId) : NO_GIDS;
        
        //【9.1.4】创建权限的 permissionData 对象，并添加到 PermissionState.mPermissions 中！
        PermissionData permissionData = ensurePermissionData(permission);

        //【9.1.5】授予权限！
        if (!permissionData.grant(userId)) { // 授权失败，就返回 PERMISSION_OPERATION_FAILURE！
            return PERMISSION_OPERATION_FAILURE;
        }

        // 如果权限有映射的 gid，在授予权限后，再次计算该 package 的所有映射的 gid！
        // 比较授权前后，gid 组是否发生了变化，如果有，返回 PERMISSION_OPERATION_SUCCESS_GIDS_CHANGED！
        if (hasGids) {
            final int[] newGids = computeGids(userId);
            if (oldGids.length != newGids.length) {
                return PERMISSION_OPERATION_SUCCESS_GIDS_CHANGED;
            }
        }

        return PERMISSION_OPERATION_SUCCESS; // 返回 PERMISSION_OPERATION_SUCCESS！
    }
```
方法逻辑如下：

- 如果已经授予权限，那就不处理，返回 PERMISSION_OPERATION_FAILURE；
- 如果授予权限失败，那就不处理，返回 PERMISSION_OPERATION_FAILURE；
- 如果授予权限成功，但是发现权限所属的 gid 发生了变化，返回 PERMISSION_OPERATION_SUCCESS_GIDS_CHANGED；
- 如果授予权限成功，但是发现权限所属的 gid 没有发生变化，返回 PERMISSION_OPERATION_SUCCESS；


### 9.1.1 PermissionsS.hasPermission

hasPermission 用来判断是否已经授予的该权限！
```java
    public boolean hasPermission(String name, int userId) {
        // 校验 uid 的有效性！
        enforceValidUserId(userId);
        if (mPermissions == null) {
            return false;
        }
        //【1】获得该权限对应的 permissionData 对象！
        PermissionData permissionData = mPermissions.get(name);
        //【9.1.1.1】判断是否已经授予了该权限！
        return permissionData != null && permissionData.isGranted(userId);
    }
```
授予该权限的条件是，其对应的 PermissionData 不为 null，同时该权限属于已经授予的状态；

#### 9.1.1.1 PermissionData.isGranted

PermissionData 的 isGranted 用于判断该权限在 userId 所在的设备用户下是否是授予状态！
```java
        public boolean isGranted(int userId) {
            if (isInstallPermission()) { // 如果是安装时权限，userId 即 key 为 USER_ALL！
                userId = UserHandle.USER_ALL;
            }

            PermissionState userState = mUserStates.get(userId);
            if (userState == null) {
                return false;
            }

            return userState.mGranted; // 返回是否授予！
        }
```
isInstallPermission 方法用来判断，该权限是否是一个安装时的权限：
```java
        private boolean isInstallPermission() {
            return mUserStates.size() == 1
                    && mUserStates.get(UserHandle.USER_ALL) != null;
        }
```
判断依据很简单，对于安装时权限，所有的设备用户都一样，所以 mUserStates 大小为 1，且 key 为 UserHandle.USER_ALL！

### 9.1.2 BasePermission.computeGids

BasePermission 的 computeGids 方法用于通过 userId 来调整该权限所映射的 gids！

```java
    public int[] computeGids(int userId) {
        if (perUser) {
            final int[] userGids = new int[gids.length];
            for (int i = 0; i < gids.length; i++) {
                userGids[i] = UserHandle.getUid(userId, gids[i]);
            }
            return userGids;
        } else {
            return gids;
        }
    }
```
如果 perUser 为 true，表示需要根据当前应用所在的 userId，调整映射的 gid，那么会通过 UserHandle.getUid 方法计算出当前 userId 下的该权限所映射的 gid！

如果 perUser 为 false，那么该权限在所有的设备用户下映射的 gid 是一样的！

### 9.1.3 PermissionsS.computeGids

返回指定设备用户 id 下，该 package 当前被授予的所有权限映射的 gid 数组！！
```java
    public int[] computeGids(int userId) {
        enforceValidUserId(userId);

        int[] gids = mGlobalGids;

        if (mPermissions != null) {
            final int permissionCount = mPermissions.size();
            for (int i = 0; i < permissionCount; i++) {
                //【1】遍历该 package 的已经被授予的所有权限！
                // 没有授予就跳过！
                String permission = mPermissions.keyAt(i);
                if (!hasPermission(permission, userId)) {
                    continue;
                }

                PermissionData permissionData = mPermissions.valueAt(i);
                //【9.1.3.1】计算该权限在当前设备用户下的 gid，将其添加到 gids！
                final int[] permGids = permissionData.computeGids(userId);
                if (permGids != NO_GIDS) {
                    gids = appendInts(gids, permGids);
                }
            }
        }
        // 并返回 gids！！
        return gids;
    }
```
mGlobalGids 是 PermissionsState 的成员变量，默认取值为：
```java
    private static final int[] NO_GIDS = {};
    private int[] mGlobalGids = NO_GIDS;
```

#### 9.1.3.1 PermissionData.computeGids

```java
    private static final class PermissionData {
        private final BasePermission mPerm;
        ... ... ...
        public int[] computeGids(int userId) {
            // 计算该权限所属的 gid！
            return mPerm.computeGids(userId);
        }
```
PermissionData.computeGids 方法最后调用的是 BasePermission.computeGids 方法，这里不多说了。

### 9.1.4 PermissionsS.ensurePermissionData

创建 PermissionData，并添加到 PermissionsState 内部的 mPermissions 集合中！
```java    
    private PermissionData ensurePermissionData(BasePermission permission) {
        if (mPermissions == null) {
            mPermissions = new ArrayMap<>();
        }
        //【9.1.2.1】创建权限对应的 PermissionData 对象！
        PermissionData permissionData = mPermissions.get(permission.name);
        if (permissionData == null) {
            permissionData = new PermissionData(permission);
            mPermissions.put(permission.name, permissionData);
        }
        return permissionData;
    }
```
可以看到，在 PermissionsState 中有一个 mPermissions 表，保存了 permission.name 和 PermissionData 的映射关系，如果是第一次创建某个权限的 PermissionData，会创建 PermissionData 实例！

#### 9.1.4.1 new PermissionData

创建 PermissionData 对象！

```java
    private static final class PermissionData {
        private final BasePermission mPerm;
        //【1】用于保存该权限在不同 userId 下的授予情况！
        private SparseArray<PermissionState> mUserStates = new SparseArray<>();

        public PermissionData(BasePermission perm) {
            mPerm = perm;
        }
        ... ... ... ...
    }
```
创建 PermissionData 对象！

mUserStates 是以 userId 为下标，存储了每个设备用户下该应用的这个权限的状态信息！

### 9.1.5 PermissionData.grant

下面我们来看看具体的权限授予过程！
```java
        public boolean grant(int userId) {
            if (!isCompatibleUserId(userId)) {
                return false;
            }

            if (isGranted(userId)) { // 如果已经授予，那就返回 false！
                return false;
            }

            // 获得该权限对应在 userId 下对应的 PermissionState 对象，没有的话就创建新的！
            PermissionState userState = mUserStates.get(userId);
            if (userState == null) {
                userState = new PermissionState(mPerm.name);
                mUserStates.put(userId, userState);
            }
            // 设置在该 userId 下的权限状态为 true！
            userState.mGranted = true;

            return true;
        }
```

首先，判断 userId 的兼容性！

```java
        private boolean isCompatibleUserId(int userId) {
            return isDefault() || !(isInstallPermission() ^ isInstallPermissionKey(userId));
        }
```
userId 要满足兼容性要求，需要至少满足其中之一条件：

- PermissionData.mUserStates 的大小 <= 0，意味着该权限还没有初始化在该 userId 下的状态！

```java
        public boolean isDefault() {
            return mUserStates.size() <= 0;
        }
```
- 要么，该权限是安装时权限，同时 userId 只能为 UserHandle.USER_ALL; 要么，该权限不是安装时权限，同时 userId 不能为 UserHandle.USER_ALL;

```java
        public static boolean isInstallPermissionKey(int userId) {
            return userId == UserHandle.USER_ALL;
        }
```
对于 isInstallPermission 方法，前面已经分析了，不多说了！

## 9.2 PermissionsS.revokeInstallPermission  - 处理不授予情况

```java
    public int revokeInstallPermission(BasePermission permission) {
        return revokePermission(permission, UserHandle.USER_ALL);
    }
```
我们继续看，参数传入;

- BasePermission permission：权限封装对象
- int userId：设备用户，因为是安装时权限，所以对所有设备用户都是一样，这里传入：UserHandle.USER_ALL

```java
    private int revokePermission(BasePermission permission, int userId) {
        //【1】如果没有授予该权限，那就返回 PERMISSION_OPERATION_FAILURE！
        if (!hasPermission(permission.name, userId)) {
            return PERMISSION_OPERATION_FAILURE;
        }

        //【2】同样，计算该权限是否有映射 gid，如果有 gid，计算当前 package 持有的权限映射的所有 gid！
        // 保存到 oldGids 中！
        final boolean hasGids = !ArrayUtils.isEmpty(permission.computeGids(userId));
        final int[] oldGids = hasGids ? computeGids(userId) : NO_GIDS;

        PermissionData permissionData = mPermissions.get(permission.name);

        //【9.2.1】取消权限的授予！
        if (!permissionData.revoke(userId)) {
            return PERMISSION_OPERATION_FAILURE;
        }

        //【9.2.2】如果该权限在所有设备用户下均没有授予该 package，即 mUserStates.size() <= 0；
        // 那么我们要移除该权限！
        if (permissionData.isDefault()) {
            ensureNoPermissionData(permission.name);
        }

        //【3】这里是比较，在撤销安装时权限后，该 package 所持有的 gid 是否发生变化！
        if (hasGids) {
            final int[] newGids = computeGids(userId);
            if (oldGids.length != newGids.length) {
                return PERMISSION_OPERATION_SUCCESS_GIDS_CHANGED;
            }
        }

        return PERMISSION_OPERATION_SUCCESS;
    }
```
撤销安装时权限的过程，和授予有很多类似的地方，我们重点看那些不同的地方！

### 9.2.1 PermissionData.revoke


接下来看看安装时权限撤销的过程！
```java
        public boolean revoke(int userId) {
            if (!isCompatibleUserId(userId)) {
                return false;
            }
            // 如果权限还没有被授予，无法撤销！
            if (!isGranted(userId)) {
                return false;
            }
            //【1】将权限在该 userId 下的状态设置为非授予的情况！
            PermissionState userState = mUserStates.get(userId);
            userState.mGranted = false;

            // 如果该权限恢复了默认状态，从 mUserStates 中移除掉它！
            if (userState.isDefault()) {
                mUserStates.remove(userId);
            }

            return true;
        }
```

### 9.2.2 PermissionsS.ensureNoPermissionData

```java
    private void ensureNoPermissionData(String name) {
        if (mPermissions == null) {
            return;
        }
        mPermissions.remove(name);
        if (mPermissions.isEmpty()) {
            mPermissions = null;
        }
    }
```
ensureNoPermissionData 用于从当前的 package 中移除该权限数据！

## 9.3 PermissionsS.updatePermissionFlags  - 更新权限标志位

当 revokeInstallPermission 或者 grantInstallPermission 方法返回的是 PERMISSION_OPERATION_SUCCESS_GIDS_CHANGED 或者 PERMISSION_OPERATION_SUCCESS 的时候，我们需要更新权限的标志位！

参数传递：

- userId 传入的是 UserHandle.USER_ALL；
- flagMask 传入的是 PackageManager.MASK_PERMISSION_FLAGS，取值为 0xFF，转为二进制就是 11111
- flagValues 则是解析上次安装信息时，解析该 package 的 perms 标签时，每一个权限的 flags！

```java
    public boolean updatePermissionFlags(BasePermission permission, int userId,
            int flagMask, int flagValues) {
        enforceValidUserId(userId);
  
        final boolean mayChangeFlags = flagValues != 0 || flagMask != 0;

        //【1】如果 mPermissions 为 null，且需要更新 flags，那么会初始化 mPermissions，将 permission 添加进入！
        if (mPermissions == null) {
            if (!mayChangeFlags) {
                return false;
            }
            ensurePermissionData(permission);
        }
        //【2】如果该权限 permission 没有对应的 PermissionData，且需要更新 flags，那么我们会创建 PermissionData 添加！
        PermissionData permissionData = mPermissions.get(permission.name);
        if (permissionData == null) {
            if (!mayChangeFlags) {
                return false;
            }
            permissionData = ensurePermissionData(permission);
        }

        //【3】获得该权限的旧的 flags，用于后续对比！
        final int oldFlags = permissionData.getFlags(userId);
 
        //【9.3.1】更新权限标志位！
        final boolean updated = permissionData.updateFlags(userId, flagMask, flagValues);

        //【4】如果权限的状态发生了变化，那就判断是否需要重新申请！
        if (updated) {
            final int newFlags = permissionData.getFlags(userId);
            if ((oldFlags & PackageManager.FLAG_PERMISSION_REVIEW_REQUIRED) == 0
                    && (newFlags & PackageManager.FLAG_PERMISSION_REVIEW_REQUIRED) != 0) {
                if (mPermissionReviewRequired == null) {
                    mPermissionReviewRequired = new SparseBooleanArray();
                }
                //【4.1】如果在该 userId 下需要重新申请，那么就将其添加到 mPermissionReviewRequired 中！
                mPermissionReviewRequired.put(userId, true);
            } else if ((oldFlags & PackageManager.FLAG_PERMISSION_REVIEW_REQUIRED) != 0
                    && (newFlags & PackageManager.FLAG_PERMISSION_REVIEW_REQUIRED) == 0) {
                //【4.2】如果在该 userId 下不需要重新申请，那么就将其从 mPermissionReviewRequired 中移除！
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
对于 mayChangeFlags，由于我们 flagMask 传入的是 PackageManager.MASK_PERMISSION_FLAGS，那就意味了需要更新 flags！

可以看到，如果 mayChangeFlags 为 true 的话，即使没有权限对应的 PermissionData 也会创建，因为这次会更新 flags！

PermissionData.getFlags 方法会获得权限旧的 flags！

重点的操作在 PermissionData.updateFlag 方法中：

### 9.3.1 PermissionData.updateFlags

更新指定权限的 flags，flagValues 是本次解析获得的 flags！！

```java
        public boolean updateFlags(int userId, int flagMask, int flagValues) {
            if (isInstallPermission()) { // 如果是安装时权限，那么 userId 为 UserHandle.USER_ALL;
                userId = UserHandle.USER_ALL;
            }
            if (!isCompatibleUserId(userId)) { // 如果 userId 不兼容，那么会直接返回！
                return false;
            }
            //【1】这里进行 & 操作，保留了 flagValues 和 flagMask 相同的位！
            final int newFlags = flagValues & flagMask;
    
            //【2】获得该 userId 下的权限状态值！
            PermissionState userState = mUserStates.get(userId);
            if (userState != null) {
                final int oldFlags = userState.mFlags;
                // 取消旧的 flags，设置新的 flags！
                userState.mFlags = (userState.mFlags & ~flagMask) | newFlags;
                if (userState.isDefault()) {
                    mUserStates.remove(userId);
                }
                // 判断 flags 是否发生变化，如果发生了变化，那就返回 true！
                return userState.mFlags != oldFlags;
            } else if (newFlags != 0) {
                // 如果该权限在该 userId 下没有状态信息，那就创建状态信息，直接设置新的 flags，返回 true！
                userState = new PermissionState(mPerm.name);
                userState.mFlags = newFlags;
                mUserStates.put(userId, userState);
                return true;
            }

            return false;
        }
```
这里有一个操作：

- 首先，进行 & 操作，保留了 flagValues 和 flagMask 相同的位，为 newFlags；
- 接着，userState.mFlags 表示旧的 flags，这里在 userState.mFlags 和 ~flagMask 中做了 & 操作，等价于取了 userState.mFlags 和 flagMask 不同的位，然后加上 newFlags，作为新的 flags！


# 10 阶段总结

在这个阶段，PMS 主要完成了如下了几个过程：


一、解析 SystemConfig 系统配置，主要会解析一下系统属性：

1、group：系统中定义的 gid，保存到了 SystemConfig.mGlobalGids 中！

2、permission：系统权限，每一个权限都创建一个 PermissionEntry，保存到 SystemConfig.mPermissions 中；
			   PermissionEntry 有如下属性：name 权限名：perUser：表示该权限的 gid 是否针对不同的 userId 做调整；gids：该权限所属 gid
			   
3、assign-permission：系统 uid 和其具有的权限，保存到了 SystemConfig.mSystemPermissions 中！每一个 uid 都对应一个或多个权限；

4、library：共享库信息，保存到了 SystemConfig.mSharedLibraries 中！

5、feature：特性信息，保存到了 SystemConfig.mAvailableFeatures 和 SystemConfig.mUnavailableFeatures 中！

这里我们重点看一些属性的解析，其他的属性我们先不关注！

然后将 SystemConfig 中的一些解析结果保存到 Settings 中：

Settings.mGlobalGids = systemConfig.getGlobalGids();
Settings.mSystemPermissions = systemConfig.getSystemPermissions();
Settings.mAvailableFeatures = systemConfig.getAvailableFeatures();

接着处理 SystemConfig.mPermissions 中解析到的系统权限！

对于每一个 PermissionEntry 创建一个 BasePermission 对象：
	在创建时会设置 BasePermission.name 为权限名；
	设置 BasePermission.sourcePackage 为定义权限的源包，因为是系统权限，所以为 android；
	设置 BasePermission.type 权限类型为 BasePermission.TYPE_BUILTIN；
	设置 BasePermission.protectionLevel 权限级别默认为 PermissionInfo.PROTECTION_SIGNATURE;

同时会设置 BasePermission.perUser 为 PermissionEntry.perUser；
同时会设置 BasePermission.gids 为 PermissionEntry.gids；

最后新创建的 BasePermission，会添加到 Settings.mPermissions 集合中！

二、读取上一次的安装信息！
