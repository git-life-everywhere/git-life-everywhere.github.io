# PMS 第 9 篇 - 通过 adb 指令分析 uninstall 过程
title: PMS 第 9 篇 - 通过 adb 指令分析 uninstall 过程
date: 2018/09/01
categories:
- AndroidFramework源码分析
- PackageManager包管理
tags: PackageManager包管理
---

[toc]

基于 Android 7.1.1 源码分析 PackageManagerService 的架构和逻辑实现！

# 0 综述

本篇文章总结下 uninstall package 的过程，一般来说，卸载一个应用有如下的方式：

- adb uninstall（最终调用的还是 pm uninstall/cmd package uninstall）；
- adb cmd package uninstall;
- adb shell pm uninstall;
- 进入应用管理器中，手动触发卸载（进入 packageInstaller）；

这里我们先来看通过 adb 指令 uninstall 的过程：


# 1 adb uninstall - commandline::adb_commandline

同样的 adb uninstall 的执行也是从 system/core/adb/commandline.cpp 开始：

adb_commandline 中会设置到大量的 adb 指令的处理，这里我们只关注 adb install 的处理：

```java
int adb_commandline(int argc, const char **argv) {
    ... ... ...
    else if (!strcmp(argv[0], "uninstall")) { // adb uninstall 命令!
        if (argc < 2) return usage();
        if (_use_legacy_install()) {
            //【*1.2.2】不支持 cmd 使用 pm 安装！
            return uninstall_app_legacy(transport_type, serial, argc, argv);
        }
        //【*1.2.1】支持 cmd 使用 cmd 安装！
        return uninstall_app(transport_type, serial, argc, argv);
    }
    ... ... ...
    usage();
    return 1;
}
```

这里看到，如果支持 cmd 命令的情况下，_use_legacy_install() 方法返回 false，会调用 uninstall_app；不支持的话，执行 uninstall_app_legacy 方法！

Android cmd 命令功能非常强大，包含了我们之前使用 am pm 等等的命令，这里我们不深入分析 cmd 的命令的实现，我们关注和 uninstall 相关的逻辑！


下面继续分析，进一步的安装过程：

## 1.1 cmd uninstall - 支持 cmd 指令
### 1.1.1 commandline::uninstall_app

```java
static int uninstall_app(TransportType transport, const char* serial, int argc, const char** argv) {
    //【1】adb uninstall 会转为 cmd package uninstall，参数相同；
    std::string cmd = "cmd package";
    while (argc-- > 0) {
        if (strcmp(*argv, "-k") == 0) {
            printf(
                "The -k option uninstalls the application while retaining the data/cache.\n"
                "At the moment, there is no way to remove the remaining data.\n"
                "You will have to reinstall the application with the same signature, and fully uninstall it.\n"
                "If you truly wish to continue, execute 'adb shell cmd package uninstall -k'.\n");
            return EXIT_FAILURE;
        }
        cmd += " " + escape_arg(*argv++);
    }
    //【*1.2.3】调用 send_shell_command 指令；
    return send_shell_command(transport, serial, cmd, false);
}
```

## 1.2 pm uninstall - 不支持 cmd 指令

### 1.2.1 commandline::uninstall_app_legacy

```java
static int uninstall_app_legacy(TransportType transport, const char* serial, int argc, const char** argv) {
    int i;
    for (i = 1; i < argc; i++) {
        if (!strcmp(argv[i], "-k")) {
            printf(
                "The -k option uninstalls the application while retaining the data/cache.\n"
                "At the moment, there is no way to remove the remaining data.\n"
                "You will have to reinstall the application with the same signature, and fully uninstall it.\n"
                "If you truly wish to continue, execute 'adb shell pm uninstall -k'\n.");
            return EXIT_FAILURE;
        }
    }
    //【*1.2.2】调用 pm_command 指令；
    return pm_command(transport, serial, argc, argv);
}
```


### 1.2.2 commandline::pm_command

```java
static int pm_command(TransportType transport, const char* serial, int argc, const char** argv) {
   //【1】adb uninstall 会转为 pm uninstall，参数相同；
    std::string cmd = "pm";

    while (argc-- > 0) {
        cmd += " " + escape_arg(*argv++);
    }
    //【*1.2.3】调用 send_shell_command 指令；
    return send_shell_command(transport, serial, cmd, false);
}
```

### 1.2.3 commandline::send_shell_command

```java
int send_shell_command(TransportType transport_type, const char* serial, const std::string& command,
                       bool disable_shell_protocol, StandardStreamsCallbackInterface* callback) {
    int fd;
    bool use_shell_protocol = false;
    while (true) {
        bool attempt_connection = true;
        // 使用 shell protocol
        if (!disable_shell_protocol) {
            FeatureSet features;
            std::string error;
            if (adb_get_feature_set(&features, &error)) {
                // 如果系统支持 shell_v2 的 feature，则使用 shell！！
                use_shell_protocol = CanUseFeature(features, kFeatureShell2);
            } else {
                attempt_connection = false;
            }
        }
        if (attempt_connection) {
            std::string error;
            // 如果是 pm uninstall 此时 command 中携带的就是以 pm 开头的命令；
            // 如果是 cmd package uninstall 此时 command 中携带的就是以 cmd 开头的命令；
            std::string service_string = ShellServiceString(use_shell_protocol, "", command);
            // 向 shell protocol 发送命令
            fd = adb_connect(service_string, &error);
            if (fd >= 0) {
                break;
            }
        }
        fprintf(stderr, "- waiting for device -\n");
        if (!wait_for_device("wait-for-device", transport_type, serial)) {
            return 1;
        }
    }
    // 处理命令执行结果！
    int exit_code = read_and_dump(fd, use_shell_protocol, callback);
    if (adb_close(fd) < 0) {
        PLOG(ERROR) << "failure closing FD " << fd;
    }
    return exit_code;
}
```
可以看到，最后都是调用 shell 执行相关指令！

前面我们有分析过：

- cmd package uninstall 最后调用的是 PackageManagerShellCommand 相关方法；
- pm uninstall 最后调用的是 pm 相关方法；

# 2 Pm install

对于 pm 命令的执行过程，我们不在过多分析，直接进入重点：

## 2.1 runUninstall

```java
    private int runUninstall() {
        //【*2.2】继续处理！
        return runShellCommand("package", mArgs);
    }
```

## 2.2 runShellCommand

我们看到，我们传入的 Service name 是 "package"，其实看到这里，我们已经能猜到了，其和 cmd install 一样，最后会调用了 PackageManagerShellCommand 的 onCommand 方法！

```java
    private int runShellCommand(String serviceName, String[] args) {
        final HandlerThread handlerThread = new HandlerThread("results");
        handlerThread.start();
        try {
            //【*3.1】通过 pms 触发 PackageManagerShellCommand 的 onCommand 方法，最后会根据参数
            // 最后会进入 runUninstall 方法中！
            ServiceManager.getService(serviceName).shellCommand(
                    FileDescriptor.in, FileDescriptor.out, FileDescriptor.err,
                    args, new ResultReceiver(new Handler(handlerThread.getLooper())));
            return 0;
        } catch (RemoteException e) {
            e.printStackTrace();
        } finally {
            handlerThread.quitSafely();
        }
        return -1;
    }
```
我们可以去 pms 的代码中看到，pms 有如下的逻辑：

```java
    @Override
    public void onShellCommand(FileDescriptor in, FileDescriptor out,
            FileDescriptor err, String[] args, ResultReceiver resultReceiver) {
        //【*3.1】调用了 PackageManagerShellCommand 的接口！！
        (new PackageManagerShellCommand(this)).exec(
                this, in, out, err, args, resultReceiver);
    }
```
PackageManagerShellCommand 继承了 ShellCommand， exec 内部会触发 onCommand 方法，有兴趣大家可以去学习，这里不关注！！


# 3 PackageManagerShellCommand

## 3.1 runUninstall

我们来分析下 runUninstall 的逻辑：
```java
    private int runUninstall() throws RemoteException {
        final PrintWriter pw = getOutPrintWriter();
        int flags = 0;
        int userId = UserHandle.USER_ALL; // 默认所有用户！

        String opt;
        //【1】读取额外参数
        while ((opt = getNextOption()) != null) {
            switch (opt) {
                case "-k":
                    //【1.1】是否再卸载后保留数据；
                    flags |= PackageManager.DELETE_KEEP_DATA;
                    break;
                case "--user":
                    //【1.1】是否指定 user！
                    userId = UserHandle.parseUserArg(getNextArgRequired());
                    break;
                default:
                    pw.println("Error: Unknown option: " + opt);
                    return 1;
            }
        }
        //【3】获得要卸载的应用包名；
        final String packageName = getNextArg();
        if (packageName == null) {
            pw.println("Error: package name not specified");
            return 1;
        }
        
        //【4】获得要卸载的应用的 split apk 包名，如果指定的 split name，那就只卸载对应的 split apk！
        final String splitName = getNextArg();
        if (splitName != null) {
            //【*3.2】移除 split apk！
            return runRemoveSplit(packageName, splitName);
        }
        //【*3.1.1】对 userId 做一个转换处理;
        userId = translateUserId(userId, "runUninstall");
        if (userId == UserHandle.USER_ALL) {
            //【5】如果是从所有用户下删除，那么 userId 变为 USER_SYSTEM；
            // flags 设置 DELETE_ALL_USERS 标志位；
            userId = UserHandle.USER_SYSTEM;
            flags |= PackageManager.DELETE_ALL_USERS;
        } else {
            //【6】如果是从指定用户下删除，那么要判断在该 userId 下是否有安装信息；
            final PackageInfo info = mInterface.getPackageInfo(packageName, 0, userId);
            if (info == null) {
                pw.println("Failure [not installed for " + userId + "]");
                return 1;
            }
            //【7】判断是否是 sys app，如果是的话 flags 增加 DELETE_SYSTEM_APP 标志位！
            final boolean isSystem =
                    (info.applicationInfo.flags & ApplicationInfo.FLAG_SYSTEM) != 0;
            if (isSystem) {
                flags |= PackageManager.DELETE_SYSTEM_APP;
            }
        }

        //【*3.1.2.1】这里是注册可以了本地监听器 LocalIntentReceiver，监听卸载结果，在 install 我们
        // 有分析过，这里就不多说了！
        final LocalIntentReceiver receiver = new LocalIntentReceiver();

        //【*4.1】触发卸载；
        mInterface.getPackageInstaller().uninstall(packageName, null /*callerPackageName*/, flags,
                receiver.getIntentSender(), userId);
        
        //【*3.1.2.2】接收卸载结果，就是前面分析时，创建的 intent fillIn！！
        final Intent result = receiver.getResult();
        final int status = result.getIntExtra(PackageInstaller.EXTRA_STATUS,
                PackageInstaller.STATUS_FAILURE);
        if (status == PackageInstaller.STATUS_SUCCESS) {
            pw.println("Success");
            return 0;
        } else {
            pw.println("Failure ["
                    + result.getStringExtra(PackageInstaller.EXTRA_STATUS_MESSAGE) + "]");
            return 1;
        }
    }
```
这里的 mInterface.getPackageInstaller() 返回的是 PackageInstallerService，下面我们去 PackageInstallerService 中看看：

### 3.1.1 translateUserId

如果 uninstall 指定的 user，那么这里会对 userId 做一个转换处理：
```java
    private int translateUserId(int userId, String logContext) {
        return ActivityManager.handleIncomingUser(Binder.getCallingPid(), Binder.getCallingUid(),
                userId, true, true, logContext, "pm command");
    }
```

### 3.1.2 LocalIntentReceiver

LocalIntentReceiver 主要用于接收最终的返回结果，以及和其他模块通信：

#### 3.1.2.1 new LocalIntentReceiver

```java
    private static class LocalIntentReceiver {
        //【1】内有一个阻塞队列，用于保存 intent！
        private final SynchronousQueue<Intent> mResult = new SynchronousQueue<>();

        //【2】发送 intent 给其他模块：
        private IIntentSender.Stub mLocalSender = new IIntentSender.Stub() {
            @Override
            public void send(int code, Intent intent, String resolvedType,
                    IIntentReceiver finishedReceiver, String requiredPermission, Bundle options) {
                try {
                    //【2.1】将 Proxy 传来的 intent 加入的阻塞队列中！
                    mResult.offer(intent, 5, TimeUnit.SECONDS);
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }
            }
        };
```
不多数说了！！


#### 3.1.2.2 getIntentSender

返回代理对象，用于跨进程通信：
```java
        public IntentSender getIntentSender() {
            return new IntentSender((IIntentSender) mLocalSender);
        }
```

#### 3.1.2.2 getResult

从内部的阻塞队列中返回结果！

```java
        public Intent getResult() {
            try {
                return mResult.take();
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
        }
```

## 3.2 runRemoveSplit

我们来看下删除 split apk 的逻辑！

参数 String packageName 是主 pkg，String splitName 则是 split apk 的包名！

```java
    private int runRemoveSplit(String packageName, String splitName) throws RemoteException {
        final PrintWriter pw = getOutPrintWriter();
        //【1-review】这边创建了一个 SessionParams 实例，封装卸载的事务参数！
        // 这里就不在分析了，前面看过！
        final SessionParams sessionParams = new SessionParams(SessionParams.MODE_INHERIT_EXISTING);
        sessionParams.installFlags |= PackageManager.INSTALL_REPLACE_EXISTING;
        sessionParams.appPackageName = packageName;

        //【*3.2.1】创建一个卸载事务！
        final int sessionId =
                doCreateSession(sessionParams, null /*installerPackageName*/, UserHandle.USER_ALL);
        boolean abandonSession = true;
        try {
            //【*3.2.2】移除 split apk！
            if (doRemoveSplit(sessionId, splitName, false /*logSuccess*/)
                    != PackageInstaller.STATUS_SUCCESS) {
                return 1;
            }
            //【*3.2.3】提交事务
            if (doCommitSession(sessionId, false /*logSuccess*/)
                    != PackageInstaller.STATUS_SUCCESS) {
                return 1;
            }
            abandonSession = false;
            pw.println("Success");
            return 0;

        } finally {
            if (abandonSession) {
                try {
                    doAbandonSession(sessionId, false /*logSuccess*/);
                } catch (Exception ignore) {
                }
            }
        }
    }
```
这里先创建了一个 SessionParams，参数均是 @hide 的，这里我省略了！

```java
    public static class SessionParams implements Parcelable {
        public int mode = MODE_INVALID;
        public int installFlags;
        public int installLocation = PackageInfo.INSTALL_LOCATION_INTERNAL_ONLY; // 默认为仅内置
        public long sizeBytes = -1;
        public String appPackageName; // 这里其为主 pkg 的包名！
        public Bitmap appIcon;
        public String appLabel;
        public long appIconLastModified = -1;
        public Uri originatingUri;
        public int originatingUid = UID_UNKNOWN;
        public Uri referrerUri;
        public String abiOverride;
        public String volumeUuid;
        public String[] grantedRuntimePermissions;
        
        public SessionParams(int mode) {
            this.mode = mode;
        }
        ... ... ...
    }
```


可以看到，对于 remove split apk，其流程和 install 很类似！

### 3.2.1 doCreateSession

创建事务：

```java
    private int doCreateSession(SessionParams params, String installerPackageName, int userId)
            throws RemoteException {
        //【*3.1.1】对 user 进行一个转换；
        userId = translateUserId(userId, "runInstallCreate");
        if (userId == UserHandle.USER_ALL) {
            userId = UserHandle.USER_SYSTEM;
            params.installFlags |= PackageManager.INSTALL_ALL_USERS;
        }

        //【*7.1.1】调用 PackageInstallerService 的 createSession 创建一个新的事务！
        final int sessionId = mInterface.getPackageInstaller()
                .createSession(params, installerPackageName, userId);
        return sessionId;
    }
```
这里和 install 很类似！

### 3.2.2 doRemoveSplit

移除 split apk，这里是真正的对 split apk 做处理，前面只是创建了一个主 apk 的 install 食物：

```java
    private int doRemoveSplit(int sessionId, String splitName, boolean logSuccess)
            throws RemoteException {
        final PrintWriter pw = getOutPrintWriter();
        PackageInstaller.Session session = null;
        try {
            //【1-review】返回之前创建的 PackageInstallerSession，将其封装为 PackageInstaller.Session 实例！
            //【*7.1.1.2】获得事务；
            session = new PackageInstaller.Session(
                    mInterface.getPackageInstaller().openSession(sessionId));
                    
            //【*7.2.2】这个地方我们知道，最后调用了 PackageInstallerSession 的 removeSplit 方法!
            session.removeSplit(splitName);

            if (logSuccess) {
                pw.println("Success");
            }
            return 0;
        } catch (IOException e) {
            pw.println("Error: failed to remove split; " + e.getMessage());
            return 1;
        } finally {
            IoUtils.closeQuietly(session);
        }
    }
```
Session 其实很简单：
```java
    public static class Session implements Closeable {
        private IPackageInstallerSession mSession;
        ... ... ...
    }
```
不多数了！

继续分析，我们看看 removeSplit 发生了什么：

### 3.2.3 doCommitSession

提交事务：

```java
    private int doCommitSession(int sessionId, boolean logSuccess) throws RemoteException {
        final PrintWriter pw = getOutPrintWriter();
        PackageInstaller.Session session = null;
        try {
            session = new PackageInstaller.Session(
                    mInterface.getPackageInstaller().openSession(sessionId));
    
            //【*3.1.2.1】这里是注册可以了本地监听器 LocalIntentReceiver，监听卸载结果，在 install 我们
            // 有分析过，这里就不多说了！
            final LocalIntentReceiver receiver = new LocalIntentReceiver();
            
            //【*7.2.4】提交事务
            session.commit(receiver.getIntentSender());

            final Intent result = receiver.getResult();
            final int status = result.getIntExtra(PackageInstaller.EXTRA_STATUS,
                    PackageInstaller.STATUS_FAILURE);
            if (status == PackageInstaller.STATUS_SUCCESS) {
                if (logSuccess) {
                    pw.println("Success");
                }
            } else {
                pw.println("Failure ["
                        + result.getStringExtra(PackageInstaller.EXTRA_STATUS_MESSAGE) + "]");
            }
            return status;
        } finally {
            IoUtils.closeQuietly(session);
        }
    }

```


# 4 PackageInstallerService 

## 4.1 uninstall

这里我们说一下参数 IntentSender statusReceiver，其是前面 LocalIntentReceiver.getIntentSender() 返回的 IntentSender 实例：

```java
    @Override
    public void uninstall(String packageName, String callerPackageName, int flags,
                IntentSender statusReceiver, int userId) {
        final int callingUid = Binder.getCallingUid();
        //【1】会校验调用者是否具有 across user 的权限，同时也会调用 appOps 去检查 callingUid 和 callerPackageName
        // 是否匹配！
        mPm.enforceCrossUserPermission(callingUid, userId, true, true, "uninstall");
        if ((callingUid != Process.SHELL_UID) && (callingUid != Process.ROOT_UID)) {
            mAppOps.checkPackage(callingUid, callerPackageName);
        }

        //【2】检查 caller 是否是设备用户自身！
        DevicePolicyManager dpm = (DevicePolicyManager) mContext.getSystemService(
                Context.DEVICE_POLICY_SERVICE);
        boolean isDeviceOwner = (dpm != null) && dpm.isDeviceOwnerAppOnCallingUser(
                callerPackageName);
        
        //【*4.1.1.1】创建一个 PackageDeleteObserverAdapter 监听卸载删除结果！
        final PackageDeleteObserverAdapter adapter = new PackageDeleteObserverAdapter(mContext,
                statusReceiver, packageName, isDeviceOwner, userId);
                
        //【3】检查调用者是否有 DELETE_PACKAGES 的权限，如果有直接通过 pms 直接 deletePackage！
        if (mContext.checkCallingOrSelfPermission(android.Manifest.permission.DELETE_PACKAGES)
                    == PackageManager.PERMISSION_GRANTED) {
            //【*5.1】继续卸载；
            mPm.deletePackage(packageName, adapter.getBinder(), userId, flags);

        } else if (isDeviceOwner) {
            //【4】检查调用者是否是 DeviceOwner，如果有直接通过 pms 直接 deletePackage！
            // 这里会将调用者转为系统进程，继续处理！
            long ident = Binder.clearCallingIdentity();
            try {
                //【*5.1】继续卸载；
                mPm.deletePackage(packageName, adapter.getBinder(), userId, flags);
            } finally {
                Binder.restoreCallingIdentity(ident);
            }

        } else {
            //【5】这种情况需要通知用户，让用户主动卸载，会发送 Intent.ACTION_UNINSTALL_PACKAGE 给 PackageInstaller！
            final Intent intent = new Intent(Intent.ACTION_UNINSTALL_PACKAGE);
            intent.setData(Uri.fromParts("package", packageName, null));
            intent.putExtra(PackageInstaller.EXTRA_CALLBACK, adapter.getBinder().asBinder());

            //【*4.1.1.2】通知用户！
            adapter.onUserActionRequired(intent);
        }
    }
```

代码很简单，不多说了！

### 4.1.1 PackageDeleteObserverAdapter

#### 4.1.1.1 new PackageInstallerService

创建了一个 package delete 监听器！

boolean showNotification 表示是否需要显示通知，上面传入的是 isDeviceOwner，就是说如果是设备拥有者，那么一定会显示通知！

```java
    static class PackageDeleteObserverAdapter extends PackageDeleteObserver {
        private final Context mContext;
        private final IntentSender mTarget; // 对应的 LocalIntentReceiver.getIntentSender(！
        private final String mPackageName; // 要卸载的 apk
        private final Notification mNotification; // 显示的通知！

        public PackageDeleteObserverAdapter(Context context, IntentSender target,
                String packageName, boolean showNotification, int userId) {
            mContext = context;
            mTarget = target;
            mPackageName = packageName;
            if (showNotification) {
                mNotification = buildSuccessNotification(mContext,
                        mContext.getResources().getString(R.string.package_deleted_device_owner),
                        packageName,
                        userId);
            } else {
                mNotification = null;
            }
        }
        ... ... ...
    }
```
不多说了！

#### 4.1.1.2 onUserActionRequired

当没有权限直接卸载时，需要用户参与卸载时，会触发该方法，可以看到，该方法最后会发送 Intent.ACTION_UNINSTALL_PACKAGE 到 PackageInstaller 中去！

```java
        @Override
        public void onUserActionRequired(Intent intent) {
            //【1】创建了一个新的 Intent fillIn；
            final Intent fillIn = new Intent();
            //【2】卸载的 apk 包名；
            fillIn.putExtra(PackageInstaller.EXTRA_PACKAGE_NAME, mPackageName);
            //【3】卸载的结果 code！！
            fillIn.putExtra(PackageInstaller.EXTRA_STATUS,
                    PackageInstaller.STATUS_PENDING_USER_ACTION);
            //【4】卸载后用于额外通信的的 Intent！！
            fillIn.putExtra(Intent.EXTRA_INTENT, intent);
            try {
                //【5】发送结果 fillIn 到 3.1.2.1！
                mTarget.sendIntent(mContext, 0, fillIn, null, null);
            } catch (SendIntentException ignored) {
            }
        }
```
对于传入的参数 intent：

```java
        // action 是 Intent.ACTION_UNINSTALL_PACKAGE！
        final Intent intent = new Intent(Intent.ACTION_UNINSTALL_PACKAGE);
        // 设置 data；
        intent.setData(Uri.fromParts("package", packageName, null));
        // 设置额外的回调；
        intent.putExtra(PackageInstaller.EXTRA_CALLBACK, adapter.getBinder().asBinder());
```
接着，在 onUserActionRequired 有对参数 intent 进行了进一步的安装！

关于 PackageInstaller.STATUS_PENDING_USER_ACTION 的广播，我们后面再分析，这里先不关注！


#### 4.1.1.3 onPackageDeleted

卸载成功后回调该接口：

```java
        @Override
        public void onPackageDeleted(String basePackageName, int returnCode, String msg) {
            //【1】如果安装成功的话，会有通知！！
            if (PackageManager.DELETE_SUCCEEDED == returnCode && mNotification != null) {
                NotificationManager notificationManager = (NotificationManager)
                        mContext.getSystemService(Context.NOTIFICATION_SERVICE);
                notificationManager.notify(basePackageName, 0, mNotification);
            }
            //【2】创建封装结果的 intent！
            final Intent fillIn = new Intent();
            //【3】卸载的 apk！
            fillIn.putExtra(PackageInstaller.EXTRA_PACKAGE_NAME, mPackageName);
            //【4】卸载的结果 code！！
            fillIn.putExtra(PackageInstaller.EXTRA_STATUS,
                    PackageManager.deleteStatusToPublicStatus(returnCode));
            //【5】卸载的额外信息！！
            fillIn.putExtra(PackageInstaller.EXTRA_STATUS_MESSAGE,
                    PackageManager.deleteStatusToString(returnCode, msg));
            //【4】卸载的原始结果 code！！
            fillIn.putExtra(PackageInstaller.EXTRA_LEGACY_STATUS, returnCode);
            try {
                //【5】发送结果 fillIn 到 3.1.2.1！
                mTarget.sendIntent(mContext, 0, fillIn, null, null);
            } catch (SendIntentException ignored) {
            }
        }
```
不多说了！！

# 5 PackageManagerService

卸载的核心接口在 PackageManagerService 中：

## 5.1 deletePackage

开始卸载删除应用：

我们知道，在前面的时候，如果是从所有用户下卸载，那么 userId 和 flags 会有如下的变化：

userId = UserHandle.USER_SYSTEM;
flags |= PackageManager.DELETE_ALL_USERS;

如果是从指定用户下卸载，那么 userId 就是我们指定的用户！

```java
    @Override
    public void deletePackage(final String packageName,
            final IPackageDeleteObserver2 observer, final int userId, final int deleteFlags) {
        mContext.enforceCallingOrSelfPermission(
                android.Manifest.permission.DELETE_PACKAGES, null);
        Preconditions.checkNotNull(packageName);
        Preconditions.checkNotNull(observer);
        final int uid = Binder.getCallingUid();
        //【*5.1.1】首先判断该 apk 是否是孤立的；
        //【*5.1.2】同时判断是否允许静默卸载！
        // 如果这个 apk 不是孤立的，并且不允许静默卸载，那么需要用户参与卸载，也就是通过 PackageInstaller
        // 这里会结束流程，同时调用 observer.onUserActionRequired 返回结果！
        if (!isOrphaned(packageName)
                && !isCallerAllowedToSilentlyUninstall(uid, packageName)) {
            try {
                final Intent intent = new Intent(Intent.ACTION_UNINSTALL_PACKAGE);
                intent.setData(Uri.fromParts(PACKAGE_SCHEME, packageName, null));
                intent.putExtra(PackageInstaller.EXTRA_CALLBACK, observer.asBinder());

                //【*4.1.1.2】通知用户参与安装！
                observer.onUserActionRequired(intent);
            } catch (RemoteException re) {
            }
            return;
        }
        //【2】判断是否是从所有 user 下卸载该应用，如果需要同时校验是否有 INTERACT_ACROSS_USERS_FULL 的权限！！
        final boolean deleteAllUsers = (deleteFlags & PackageManager.DELETE_ALL_USERS) != 0;
        final int[] users = deleteAllUsers ? sUserManager.getUserIds() : new int[]{ userId };

        if (UserHandle.getUserId(uid) != userId || (deleteAllUsers && users.length > 1)) {
            mContext.enforceCallingOrSelfPermission(
                    android.Manifest.permission.INTERACT_ACROSS_USERS_FULL,
                    "deletePackage for user " + userId);
        }
        //【*5.1.3】如果该用户下不允许卸载应用（用户限制）
        if (isUserRestricted(userId, UserManager.DISALLOW_UNINSTALL_APPS)) {
            try {
                //【*4.1.1.3】结束卸载，返回！
                observer.onPackageDeleted(packageName,
                        PackageManager.DELETE_FAILED_USER_RESTRICTED, null);
            } catch (RemoteException re) {
            }
            return;
        }
        //【3】如果不是从所有的用户下卸载，并且该 apk 处于 BlockUninstall 状态
        // 那么不能卸载！
        //【*5.1.4】通过 getBlockUninstallForUser 来判断应用是否处于 Blockinstall 状态！
        if (!deleteAllUsers && getBlockUninstallForUser(packageName, userId)) {
            try {
                //【*4.1.1.3】结束卸载，返回！
                observer.onPackageDeleted(packageName,
                        PackageManager.DELETE_FAILED_OWNER_BLOCKED, null);
            } catch (RemoteException re) {
            }
            return;
        }

        if (DEBUG_REMOVE) {
            Slog.d(TAG, "deletePackageAsUser: pkg=" + packageName + " user=" + userId
                    + " deleteAllUsers: " + deleteAllUsers );
        }

        //【4】这里的 mHandler 我们在 pms 的启动的时候有分析过，其持有子线程的 looper！
        mHandler.post(new Runnable() {
            public void run() {
                mHandler.removeCallbacks(this);
                int returnCode;
                
                //【4.1】如果不是从所有的 user 下卸载该 apk，进入 if 分支，否则，进入 else 分支！
                if (!deleteAllUsers) {
                    //【*5.2】调用 deletePackageX 继续卸载！
                    returnCode = deletePackageX(packageName, userId, deleteFlags);

                } else {
                    //【*5.1.4】先获得 pkg 在所有 user 下的 block unistall 状态！
                    int[] blockUninstallUserIds = getBlockUninstallForUsers(packageName, users);
                    
                    //【4.2】如果都不处于 block unistall，那么就直接卸载！
                    if (ArrayUtils.isEmpty(blockUninstallUserIds)) {
                        //【*5.2】调用 deletePackageX 继续卸载！
                        returnCode = deletePackageX(packageName, userId, deleteFlags);

                    } else {
                        //【4.3】flags 取消 DELETE_ALL_USERS 标志位！
                        final int userFlags = deleteFlags & ~PackageManager.DELETE_ALL_USERS;
                        for (int userId : users) {
                            //【4.3.1】只会卸载那些不处于 block unistall 状态下的 user 中的 apk！
                            if (!ArrayUtils.contains(blockUninstallUserIds, userId)) {
                                //【*5.2】调用 deletePackageX 继续卸载！
                                returnCode = deletePackageX(packageName, userId, userFlags);
                                if (returnCode != PackageManager.DELETE_SUCCEEDED) {
                                    Slog.w(TAG, "Package delete failed for user " + userId
                                            + ", returnCode " + returnCode);
                                }
                            }
                        }
                        //【4.4】由于在某些 user 下 pkg 处于 block uninstall 状态导致无法安装
                        // 所以需要返回给用户！
                        returnCode = PackageManager.DELETE_FAILED_OWNER_BLOCKED;
                    }
                }
                try {
                    //【*4.1.1.3】返回结果！
                    observer.onPackageDeleted(packageName, returnCode, null);
                } catch (RemoteException e) {
                    Log.i(TAG, "Observer no longer exists.");
                } //end catch
            } //end run
        });
    }
```
过程分析的很详细！



### 5.1.1 isOrphaned

判断该应用是否是孤立的：

```java
    public boolean isOrphaned(String packageName) {
        // reader
        synchronized (mPackages) {
            //【*5.1.1.1】进一步判断！
            return mSettings.isOrphaned(packageName);
        }
    }
```

#### 5.1.1.1 Settings.isOrphaned

```java
    boolean isOrphaned(String packageName) {
        final PackageSetting pkg = mPackages.get(packageName);
        if (pkg == null) {
            throw new IllegalArgumentException("Unknown package: " + packageName);
        }
        //【1】通过 PackageSetting 的 isOrphaned 属性判断！
        return pkg.isOrphaned;
    }
```
不多数了！！

### 5.1.2 isCallerAllowedToSilentlyUninstall

判断是否允许静默卸载：

```java
    private boolean isCallerAllowedToSilentlyUninstall(int callingUid, String pkgName) {
        //【1】如果 callingUid 是 shell，root 或者 system，那么是可以静默卸载的！
        if (callingUid == Process.SHELL_UID || callingUid == Process.ROOT_UID
              || callingUid == Process.SYSTEM_UID) {
            return true;
        }
        final int callingUserId = UserHandle.getUserId(callingUid);

        //【2】如果 caller 就是安装该 apk 的 installer，那么支持静默卸载！
        //【*5.1.2.1】通过 getPackageUid 获得 package uid；
        //【*5.1.2.2】通过 getInstallerPackageName 获得 package 的安装者；
        if (callingUid == getPackageUid(getInstallerPackageName(pkgName), 0, callingUserId)) {
            return true;
        }

        //【3】如果 caller 是 package verifier，那么允许静默卸载！
        if (mRequiredVerifierPackage != null &&
                callingUid == getPackageUid(mRequiredVerifierPackage, 0, callingUserId)) {
            return true;
        }

        //【4】如果 caller 是 package unstaller，那么允许静默卸载！
        if (mRequiredUninstallerPackage != null &&
                callingUid == getPackageUid(mRequiredUninstallerPackage, 0, callingUserId)) {
            return true;
        }

        //【5】如果 caller 是 storage manager，那么允许静默卸载！
        if (mStorageManagerPackage != null &&
                callingUid == getPackageUid(mStorageManagerPackage, 0, callingUserId)) {
            return true;
        }
        return false;
    }
```

#### 5.1.2.1 getPackageUid

获得 pacakge 的 uid：

```java
    @Override
    public int getPackageUid(String packageName, int flags, int userId) {
        if (!sUserManager.exists(userId)) return -1;
        flags = updateFlagsForPackage(flags, userId, packageName);
        enforceCrossUserPermission(Binder.getCallingUid(), userId,
                false /* requireFullPermission */, false /* checkShell */, "get package uid");

        // reader
        synchronized (mPackages) {
            final PackageParser.Package p = mPackages.get(packageName);
            if (p != null && p.isMatch(flags)) {
                return UserHandle.getUid(userId, p.applicationInfo.uid);
            }
            if ((flags & MATCH_UNINSTALLED_PACKAGES) != 0) {
                final PackageSetting ps = mSettings.mPackages.get(packageName);
                if (ps != null && ps.isMatch(flags)) {
                    return UserHandle.getUid(userId, ps.appId);
                }
            }
        }

        return -1;
    }
```

#### 5.1.2.2 getInstallerPackageName

获得 package 的安装者：

```java
    @Override
    public String getInstallerPackageName(String packageName) {
        synchronized (mPackages) {
            //【*5.1.2.2.1】通过 Settings 获得该 pkg 的 installer！
            return mSettings.getInstallerPackageNameLPr(packageName);
        }
    }
```

##### 5.1.2.2.1 Settings.getInstallerPackageNameLPr

```java
    String getInstallerPackageNameLPr(String packageName) {
        final PackageSetting pkg = mPackages.get(packageName);
        if (pkg == null) {
            throw new IllegalArgumentException("Unknown package: " + packageName);
        }
        //【1】返回 PackageSetting 的属性 installerPackageName
        return pkg.installerPackageName;
    }
```
不多说了！

### 5.1.3 isUserRestricted

判断在 userId 是否有用户限制，限制由 restrictionKey 指定：

```java
    boolean isUserRestricted(int userId, String restrictionKey) {
        Bundle restrictions = sUserManager.getUserRestrictions(userId);
        if (restrictions.getBoolean(restrictionKey, false)) {
            Log.w(TAG, "User is restricted: " + restrictionKey);
            return true;
        }
        return false;
    }
```

### 5.1.4 getBlockUninstallForUser

判断该 package 是否处于 block uninstall 的状态：

```java
    @Override
    public boolean getBlockUninstallForUser(String packageName, int userId) {
        synchronized (mPackages) {
            PackageSetting ps = mSettings.mPackages.get(packageName);
            if (ps == null) {
                Log.i(TAG, "Package doesn't exist in get block uninstall " + packageName);
                return false;
            }
            //【*5.1.4.1】通过 PackageSetting 返回其 block uninstall 的状态！
            return ps.getBlockUninstall(userId);
        }
    }
```

#### 5.1.4.1 PackageSettingBase.getBlockUninstall

PackageSetting 继承了 PackageSettingBase，getBlockUninstall 方法是在父类中：

```java
    boolean getBlockUninstall(int userId) {
        //【*5.1.4.2】返回该 pkg 的使用状态对象：PackageUserState！
        return readUserState(userId).blockUninstall;
    }
```
PackageUserState 实例，表示每个 pkg 在对应的 user 下的使用状态，其在我们分析 pms 的启动时有分析过，这里不多说了！


#### 5.1.4.2 PackageSettingBase.readUserState

```java
    public PackageUserState readUserState(int userId) {
        PackageUserState state = userState.get(userId);
        if (state != null) {
            //【1】如果有的话，就返回这个 user 下的 PackageUserState 实例！
            return state;
        }
        //【2】否则，返回默认的！
        return DEFAULT_USER_STATE;
    }
```

这里的 DEFAULT_USER_STATE 是一个 PackageUserState 对象！！

```java
 private static final PackageUserState DEFAULT_USER_STATE = new PackageUserState();
```

### 5.1.5 getBlockUninstallForUsers

该方法用于获得该 package 在多个 user 下的 block uninstall 状态：

```java
    private int[] getBlockUninstallForUsers(String packageName, int[] userIds) {
        int[] result = EMPTY_INT_ARRAY;
        for (int userId : userIds) {
            //【*5.1.4】获得单个 user 下的 pkg 的 block uninstall 状态，并将状态为 true 的 user！
            // 保存到数组中，返回！
            if (getBlockUninstallForUser(packageName, userId)) {
                result = ArrayUtils.appendInt(result, userId);
            }
        }
        return result;
    }
```

不多说了！！

## 5.2 deletePackageX

接着是进入第二阶段的卸载：

- 如果可以卸载所有 user 下的安装，那么 deleteFlags 会被设置为 PackageManager.DELETE_ALL_USERS; 而 user 则是每一个用户；
- 如果是卸载指定的 user 下的安装，那么 deleteFlags 不会设置为 PackageManager.DELETE_ALL_USERS；

```java
    private int deletePackageX(String packageName, int userId, int deleteFlags) {
        //【*5.2.1.1】创建一个 PackageRemovedInfo 对象！
        final PackageRemovedInfo info = new PackageRemovedInfo();
        final boolean res;
        
        //【1】这里对 user 又做了一次处理。如果是卸载所有用户下的安装，那么 removeUser 为 UserHandle.USER_ALL！
        final int removeUser = (deleteFlags & PackageManager.DELETE_ALL_USERS) != 0
                ? UserHandle.USER_ALL : userId;

        //【*5.2.2】如果要卸载的 pkg 是用于设备管理的，那么禁止卸载，返回！
        if (isPackageDeviceAdmin(packageName, removeUser)) {
            Slog.w(TAG, "Not removing package " + packageName + ": has active device admin");
            return PackageManager.DELETE_FAILED_DEVICE_POLICY_MANAGER;
        }

        PackageSetting uninstalledPs = null; // 要卸载的 apk 的安装信息！

        int[] allUsers;
        synchronized (mPackages) {
            //【2】获得上一次的安装信息，如果为 null，直接返回！！
            uninstalledPs = mSettings.mPackages.get(packageName);
            if (uninstalledPs == null) {
                Slog.w(TAG, "Not removing non-existent package " + packageName);
                return PackageManager.DELETE_FAILED_INTERNAL_ERROR;
            }
            //【3】获得所用的 user，并判断 pkg 是安装在哪些 user 下的，返回这些 user 的数组！
            allUsers = sUserManager.getUserIds();
            info.origUsers = uninstalledPs.queryInstalledUsers(allUsers, true);
        }

        final int freezeUser; // 用于保存卸载前，应用在那个 user 下处于冻结状态！
        
        //【4】如果这是要卸载的应用是一个被覆盖安装更新过的 sys apk，同时 deleteFlags 没有设置 DELETE_SYSTEM_APP 位
        // 那么，我们在所有用户下冻结，否则我们只在 removeUser 下冻结！
        //【*5.2.3】isUpdatedSystemApp 判断是否是被覆盖安装更新过的 sys apk！
        if (isUpdatedSystemApp(uninstalledPs)
                && ((deleteFlags & PackageManager.DELETE_SYSTEM_APP) == 0)) {
            freezeUser = UserHandle.USER_ALL;
        } else {
            freezeUser = removeUser;
        }

        synchronized (mInstallLock) {
            if (DEBUG_REMOVE) Slog.d(TAG, "deletePackageX: pkg=" + packageName + " user=" + userId);
            //【*5.2.4】在卸载前进入冻结状态！
            try (PackageFreezer freezer = freezePackageForDelete(packageName, freezeUser,
                    deleteFlags, "deletePackageX")) {

                //【*5.3】继续卸载；
                res = deletePackageLIF(packageName, UserHandle.of(removeUser), true, allUsers,
                        deleteFlags | REMOVE_CHATTY, info, true, null);
            }
            synchronized (mPackages) {
                if (res) {
                    mEphemeralApplicationRegistry.onPackageUninstalledLPw(uninstalledPs.pkg);
                }
            }
        }
        
        //【5】res 为 true，表示卸载成功了，那么发送相关的广播！
        if (res) {
            final boolean killApp = (deleteFlags & PackageManager.DELETE_DONT_KILL_APP) == 0;
            //【*5.2.1.2】发送 removed 广播；
            info.sendPackageRemovedBroadcasts(killApp);
            //【*5.2.1.3】发送 updated 广播；
            info.sendSystemPackageUpdatedBroadcasts();
            //【*5.2.1.4】发送 appeared广播；
            info.sendSystemPackageAppearedBroadcasts();
        }

        Runtime.getRuntime().gc(); // gc 回收资源！！
    
        // Delete the resources here after sending the broadcast to let
        // other processes clean up before deleting resources.
        if (info.args != null) {
            synchronized (mInstallLock) {
                //【*6.2.2.1.2】执行删除 apk 的操作！！
                info.args.doPostDeleteLI(true);
            }
        }

        return res ? PackageManager.DELETE_SUCCEEDED : PackageManager.DELETE_FAILED_INTERNAL_ERROR;
    }
```
流程已经分析的很详细了！！


### 5.2.1 PackageRemovedInfo

#### 5.2.1.1 new PackageRemovedInfo

创建一个 PackageRemovedInfo 实例，分装要卸载的 apk 的相关信息！

```java
    class PackageRemovedInfo {
        String removedPackage;
        int uid = -1;
        int removedAppId = -1;
        int[] origUsers;
        int[] removedUsers = null;
        boolean isRemovedPackageSystemUpdate = false;
        boolean isUpdate;
        boolean dataRemoved;
        boolean removedForAllUsers;
        InstallArgs args = null; // 参数实例，用于执行卸载，清理的操作，后面会分析到！
        ArrayMap<String, PackageRemovedInfo> removedChildPackages;
        ArrayMap<String, PackageInstalledInfo> appearedChildPackages;

        ... ... ...
    }
```

同时，PackageRemovedInfo 内部也有很多的方法，用于发送广播，这里我们先分析当前流程能用的到的!!

#### 5.2.1.2 sendPackageRemovedBroadcasts

发送升级包被移除的广播：

```java
       void sendPackageRemovedBroadcasts(boolean killApp) {
            //【*5.2.1.2.1】发送 removed 的广播；
            sendPackageRemovedBroadcastInternal(killApp);
            final int childCount = removedChildPackages != null ? removedChildPackages.size() : 0;
            for (int i = 0; i < childCount; i++) {
                PackageRemovedInfo childInfo = removedChildPackages.valueAt(i);
                //【*5.2.1.2.1】对 child pkg 一样的处理；
                childInfo.sendPackageRemovedBroadcastInternal(killApp);
            }
        }
```
逻辑简单，不多说了！

##### 5.2.1.2.1 sendPackageRemovedBroadcastInternal

发送 removed 的广播：

```java
        private void sendPackageRemovedBroadcastInternal(boolean killApp) {
            Bundle extras = new Bundle(2);
            extras.putInt(Intent.EXTRA_UID, removedAppId >= 0  ? removedAppId : uid);
            extras.putBoolean(Intent.EXTRA_DATA_REMOVED, dataRemoved);
            extras.putBoolean(Intent.EXTRA_DONT_KILL_APP, !killApp);
            if (isUpdate || isRemovedPackageSystemUpdate) {
                extras.putBoolean(Intent.EXTRA_REPLACING, true);
            }
            extras.putBoolean(Intent.EXTRA_REMOVED_FOR_ALL_USERS, removedForAllUsers);
            if (removedPackage != null) {
                //【1】首先会发送 Intent.ACTION_PACKAGE_REMOVED 的广播；
                sendPackageBroadcast(Intent.ACTION_PACKAGE_REMOVED , removedPackage,
                        extras, 0, null, null, removedUsers);
                //【2】然后会发送 Intent.ACTION_PACKAGE_FULLY_REMOVED 的广播；
                // 但前提的是清楚了 data，并且本次是卸载的三方应用；
                if (dataRemoved && !isRemovedPackageSystemUpdate) {
                    sendPackageBroadcast(Intent.ACTION_PACKAGE_FULLY_REMOVED,
                            removedPackage, extras, 0, null, null, removedUsers);
                }
            }
            if (removedAppId >= 0) {
                //【3】最后，发送 Intent.ACTION_UID_REMOVED 广播！
                sendPackageBroadcast(Intent.ACTION_UID_REMOVED, null, extras, 0, null, null,
                        removedUsers);
            }
        }
```
这里就不不多说了！


#### 5.2.1.3 sendSystemPackageUpdatedBroadcasts

发送 system app updated 广播：

```java
        void sendSystemPackageUpdatedBroadcasts() {
            if (isRemovedPackageSystemUpdate) {
                //【*5.2.1.3.1】发送 system app updated 广播；
                sendSystemPackageUpdatedBroadcastsInternal();
                final int childCount = (removedChildPackages != null)
                        ? removedChildPackages.size() : 0;
                for (int i = 0; i < childCount; i++) {
                    PackageRemovedInfo childInfo = removedChildPackages.valueAt(i);
                    if (childInfo.isRemovedPackageSystemUpdate) {
                        //【*5.2.1.3.1】对 child pkg 一样的处理；
                        childInfo.sendSystemPackageUpdatedBroadcastsInternal();
                    }
                }
            }
        }
```
逻辑简单，不多说了！

##### 5.2.1.3.1 sendSystemPackageUpdatedBroadcastsInternal

核心方法：

```java
        private void sendSystemPackageUpdatedBroadcastsInternal() {
            Bundle extras = new Bundle(2);
            extras.putInt(Intent.EXTRA_UID, removedAppId >= 0 ? removedAppId : uid);
            extras.putBoolean(Intent.EXTRA_REPLACING, true);
            //【1】依次发送如下的三个广播；
            sendPackageBroadcast(Intent.ACTION_PACKAGE_ADDED, removedPackage,
                    extras, 0, null, null, null);
            sendPackageBroadcast(Intent.ACTION_PACKAGE_REPLACED, removedPackage,
                    extras, 0, null, null, null);
            sendPackageBroadcast(Intent.ACTION_MY_PACKAGE_REPLACED, null,
                    null, 0, removedPackage, null, null);
        }
```
不多说了！


#### 5.2.1.4 sendSystemPackageAppearedBroadcasts

发送 system app appeared 广播，针对于 child apk：

```java
        void sendSystemPackageAppearedBroadcasts() {
            final int packageCount = (appearedChildPackages != null)
                    ? appearedChildPackages.size() : 0;
            for (int i = 0; i < packageCount; i++) {
                PackageInstalledInfo installedInfo = appearedChildPackages.valueAt(i);
                for (int userId : installedInfo.newUsers) {
                    //【*5.2.1.4.1】发送 child apk appeared 的广播！
                    sendPackageAddedForUser(installedInfo.name, true,
                            UserHandle.getAppId(installedInfo.uid), userId);
                }
            }
        }
```

##### 5.2.1.4.1 sendPackageAddedForUser(of pms)

```java
    private void sendPackageAddedForUser(String packageName, PackageSetting pkgSetting,
            int userId) {
        //【1】判断是否是 sys app！
        final boolean isSystem = isSystemApp(pkgSetting) || isUpdatedSystemApp(pkgSetting);
        //【*5.2.1.4.2】继续发送：
        sendPackageAddedForUser(packageName, isSystem, pkgSetting.appId, userId);
    }
```
##### 5.2.1.4.2 sendPackageAddedForUser(of pms)

核心方法，这里的 boolean isSystem 传入的是 true：

```java
    private void sendPackageAddedForUser(String packageName, boolean isSystem,
            int appId, int userId) {
        Bundle extras = new Bundle(1);
        extras.putInt(Intent.EXTRA_UID, UserHandle.getUid(userId, appId));
        //【1】发送 Intent.ACTION_PACKAGE_ADDED 广播！！
        sendPackageBroadcast(Intent.ACTION_PACKAGE_ADDED,
                packageName, extras, 0, null, null, new int[] {userId});
        try {
            IActivityManager am = ActivityManagerNative.getDefault();
            if (isSystem && am.isUserRunning(userId, 0)) {
                // The just-installed/enabled app is bundled on the system, so presumed
                // to be able to run automatically without needing an explicit launch.
                // Send it a BOOT_COMPLETED if it would ordinarily have gotten one.
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


### 5.2.2 isPackageDeviceAdmin
```java
    private boolean isPackageDeviceAdmin(String packageName, int userId) {
        IDevicePolicyManager dpm = IDevicePolicyManager.Stub.asInterface(
                ServiceManager.getService(Context.DEVICE_POLICY_SERVICE));
        try {
            if (dpm != null) {
                final ComponentName deviceOwnerComponentName = dpm.getDeviceOwnerComponent(
                        /* callingUserOnly =*/ false);
                final String deviceOwnerPackageName = deviceOwnerComponentName == null ? null
                        : deviceOwnerComponentName.getPackageName();
                // Does the package contains the device owner?
                // TODO Do we have to do it even if userId != UserHandle.USER_ALL?  Otherwise,
                // this check is probably not needed, since DO should be registered as a device
                // admin on some user too. (Original bug for this: b/17657954)
                if (packageName.equals(deviceOwnerPackageName)) {
                    return true;
                }
                // Does it contain a device admin for any user?
                int[] users;
                if (userId == UserHandle.USER_ALL) {
                    users = sUserManager.getUserIds();
                } else {
                    users = new int[]{userId};
                }
                for (int i = 0; i < users.length; ++i) {
                    if (dpm.packageHasActiveAdmins(packageName, users[i])) {
                        return true;
                    }
                }
            }
        } catch (RemoteException e) {
        }
        return false;
    }
```

### 5.2.3 isUpdatedSystemApp
```java
    private static boolean isUpdatedSystemApp(PackageSetting ps) {
        return (ps.pkgFlags & ApplicationInfo.FLAG_UPDATED_SYSTEM_APP) != 0;
    }
```

### 5.2.4 freezePackageForInstall

进入冻结状态！
```JAVA
    public PackageFreezer freezePackageForInstall(String packageName, int userId, int installFlags,
            String killReason) {
        if ((installFlags & PackageManager.INSTALL_DONT_KILL_APP) != 0) {
            return new PackageFreezer();
        } else {
            //【*5.2.4.1】对于卸载的情况，是进入这里的！
            return freezePackage(packageName, userId, killReason);
        }
    }
```

#### 5.2.4.1 freezePackage

看代码是创建了一个 PackageFreezer 实例：

```java
    public PackageFreezer freezePackage(String packageName, int userId, String killReason) {
        //【*5.2.4.2】创建一个 PackageFreezer 实例！
        return new PackageFreezer(packageName, userId, killReason);
    }
```

#### 5.2.4.2 new PackageFreezer


PackageFreezer 是一个冻结对象，在创建它的时候就会执行冻结操作！

```java
    private class PackageFreezer implements AutoCloseable {
        private final String mPackageName; // 要被冻结的 pkg
        private final PackageFreezer[] mChildren; // 要被冻结的 child pkg

        private final boolean mWeFroze;

        private final AtomicBoolean mClosed = new AtomicBoolean();
        private final CloseGuard mCloseGuard = CloseGuard.get();

        public PackageFreezer() {
            mPackageName = null;
            mChildren = null;
            mWeFroze = false;
            mCloseGuard.open("close");
        }

        public PackageFreezer(String packageName, int userId, String killReason) {
            synchronized (mPackages) {
                mPackageName = packageName;
                //【1】将该 pkg 添加到 pms 的内部 mFrozenPackages 集合中！
                mWeFroze = mFrozenPackages.add(mPackageName);

                //【2】返回 pkg 的安装信息，如果不会 null，那就 kill 掉该进程！
                final PackageSetting ps = mSettings.mPackages.get(mPackageName);
                if (ps != null) {
                    //【2.1】kill app 进程，这里不过多关注！
                    killApplication(ps.name, ps.appId, userId, killReason);
                }
                
                //【3】如果该 package 有 child package，做同样的处理！
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
        ... ... ...
    }
```
在 pms 的内部，有一个集合：

```java
    @GuardedBy("mPackages")
    final ArraySet<String> mFrozenPackages = new ArraySet<>();
```

用于保存所有的处于冻结状态的 package！


## 5.3 deletePackageLIF

接着进入卸载的第三个阶段，我们来回归下参数：

- boolean deleteCodeAndResources：表示是否删除 apk 和资源，这里传入的是 true；
- int flags：卸载的 flags，传入 deleteFlags | REMOVE_CHATTY；
- boolean writeSettings：是否持久化处理的数据，这里传入的是 true；
- PackageParser.Package replacingPackage：用于取代的 pkg，这里传入的是 null；

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
            //【1】获得上一次的安装信息！
            ps = mSettings.mPackages.get(packageName);
            if (ps == null) {
                Slog.w(TAG, "Package named '" + packageName + "' doesn't exist.");
                return false;
            }
            //【2】如果卸载的是 child package，并且
            // 其不是 sys app（无 FLAG_SYSTEM 标志位），或者是 sys app，且卸载 flags 设置了 DELETE_SYSTEM_APP 位！
            // 那么这里立刻执行卸载！
            if (ps.parentPackageName != null && (!isSystemApp(ps)
                    || (flags & PackageManager.DELETE_SYSTEM_APP) != 0)) {
                if (DEBUG_REMOVE) {
                    Slog.d(TAG, "Uninstalled child package:" + packageName + " for user:"
                            + ((user == null) ? UserHandle.USER_ALL : user));
                }
                final int removedUserId = (user != null) ? user.getIdentifier()
                        : UserHandle.USER_ALL;
                        
                //【*5.3.1】清理该 pkg 在 removedUserId 下的数据！
                if (!clearPackageStateForUserLIF(ps, removedUserId, outInfo)) {
                    return false;
                }

                //【*5.3.2】设置该 pkg 的使用状态信息！
                markPackageUninstalledForUserLPw(ps, user);

                // 更新应用的偏好设置，这里我们就先不分析，有时间了加进去！
                scheduleWritePackageRestrictionsLocked(user);
                return true;
            }
        }

        //【2】如果卸载的不是 sys app（无 FLAG_SYSTEM 标志位），或者卸载 flags 设置了 DELETE_SYSTEM_APP 位！
        // 同时，只是在某个用户下卸载该 apk，进入下面的逻辑
        // 可以看到：如果 apk 是 sys，那么还必须要设置 DELETE_SYSTEM_APP 标志位才行！
        if (((!isSystemApp(ps) || (flags&PackageManager.DELETE_SYSTEM_APP) != 0) && user != null
                && user.getIdentifier() != UserHandle.USER_ALL)) {

            //【*5.3.2】设置该 pkg 的使用状态信息！
            markPackageUninstalledForUserLPw(ps, user);

            //【2.1】如果是 data app，进入 if 分支，而 sys app 进入 else 分支！
            if (!isSystemApp(ps)) {
                //【*5.3.3】判断下该应用是否需要被缓存下来！！
                boolean keepUninstalledPackage = shouldKeepUninstalledPackageLPr(packageName);
                
                //【2.1.1】该 apk 在一些用户下处于 install 状态，或者该 pkg 需要被 keep！
                if (ps.isAnyInstalled(sUserManager.getUserIds()) || keepUninstalledPackage) {
                    if (DEBUG_REMOVE) Slog.d(TAG, "Still installed by other users");
                    
                    //【*5.3.1】清理该 pkg 在 user 下的数据，清楚shibai！
                    if (!clearPackageStateForUserLIF(ps, user.getIdentifier(), outInfo)) {
                        return false;
                    }
                    
                    // 更新应用的偏好设置，这里我们就先不分析，有时间了加进去！
                    scheduleWritePackageRestrictionsLocked(user);
                    return true;
                    
                } else {
                    if (DEBUG_REMOVE) Slog.d(TAG, "Not installed by other users, full delete");
                    //【2.1.1】该 apk 没有在任何 user 下安装，同时也不需要 keep，那么这里会将其在该 user 下的
                    // 的安装状态设置为 true，这样卸载广播就能正确的发出了（感觉像是解决一个 bug）
                    ps.setInstalled(true, user.getIdentifier());
                }
                
            } else {
                if (DEBUG_REMOVE) Slog.d(TAG, "Deleting system app");
                //【2.2】对于 sys app，所有用户都会有该 app，所以这里我们会清楚在该 user 下的数据！
                //【*5.3.1】清理该 pkg 在 user 下的数据，清楚shibai！
                if (!clearPackageStateForUserLIF(ps, user.getIdentifier(), outInfo)) {
                    return false;
                }
                
                // 更新应用的偏好设置，这里我们就先不分析，有时间了加进去！
                scheduleWritePackageRestrictionsLocked(user);
                return true;
            }
        }

        //【3】如果要卸载的 apk 是一个复合 apk，有 split apk，那么这里会对其 child pkg 做同样的处理！！
        if (ps.childPackageNames != null && outInfo != null) {
            synchronized (mPackages) {
                final int childCount = ps.childPackageNames.size();
                //【*5.2.1】将每一个 child pkg 都封装成一个 PackageRemovedInfo 实例，并计算其 origUsers
                // 加到 parent pkg 的 outInfo.removedChildPackages 中！
                outInfo.removedChildPackages = new ArrayMap<>(childCount);
                for (int i = 0; i < childCount; i++) {
                    String childPackageName = ps.childPackageNames.get(i);
                    PackageRemovedInfo childInfo = new PackageRemovedInfo();
                    childInfo.removedPackage = childPackageName;
                    outInfo.removedChildPackages.put(childPackageName, childInfo);
                    PackageSetting childPs = mSettings.peekPackageLPr(childPackageName);
                    if (childPs != null) {
                        childInfo.origUsers = childPs.queryInstalledUsers(allUserHandles, true);
                    }
                }
            }
        }
        
        boolean ret = false;
        //【4】进入核心的卸载阶段，这个阶段在返回后，会创建一个 InstallArgs 对象！！
        if (isSystemApp(ps)) {
            if (DEBUG_REMOVE) Slog.d(TAG, "Removing system package: " + ps.name);
            //【*6.1】卸载 sys app，如果 sys app 被覆盖安装过，那么会 fall back 回 sys app！
            ret = deleteSystemPackageLIF(ps.pkg, ps, allUserHandles, flags, outInfo, writeSettings);
            
        } else {
            if (DEBUG_REMOVE) Slog.d(TAG, "Removing non-system package: " + ps.name);
            //【*6.1】卸载 data app!
            ret = deleteInstalledPackageLIF(ps, deleteCodeAndResources, flags, allUserHandles,
                    outInfo, writeSettings, replacingPackage);

        }

        //【5】这里是记录下我们是否是在所有用户下移除 pkg，对 child apk 也做同样的处理！
        if (outInfo != null) {
            outInfo.removedForAllUsers = mPackages.get(ps.name) == null;
            if (outInfo.removedChildPackages != null) {
                synchronized (mPackages) {
                    final int childCount = outInfo.removedChildPackages.size();
                    for (int i = 0; i < childCount; i++) {
                        PackageRemovedInfo childInfo = outInfo.removedChildPackages.valueAt(i);
                        if (childInfo != null) {
                            childInfo.removedForAllUsers = mPackages.get(
                                    childInfo.removedPackage) == null;
                        }
                    }
                }
            }

            //【5.1】这里对被覆盖安装过的 sys app 又做了特殊处理！！我们知道当我们删除了位于 data 的 app 数据后
            // 我们会恢复 sys app 的安装数据！这里主要是处理如下情况：
            // 如果 sys app 有 child pkg，但是可能有一些 child pkg 只申明在了 sys app 中，没有在 updated app 中
            // 此时我们会重新创建 child pkg 的 PackageInstalledInfo，保存到 outInfo 中！
            if (isSystemApp(ps)) {
                synchronized (mPackages) {
                    PackageSetting updatedPs = mSettings.peekPackageLPr(ps.name);
                    final int childCount = (updatedPs.childPackageNames != null)
                            ? updatedPs.childPackageNames.size() : 0;
                    for (int i = 0; i < childCount; i++) {
                        String childPackageName = updatedPs.childPackageNames.get(i);
                        //【5.1.1】如果 outInfo 没有保存该 child pkg，进行以下逻辑：
                        if (outInfo.removedChildPackages == null
                                || outInfo.removedChildPackages.indexOfKey(childPackageName) < 0) {
                            PackageSetting childPs = mSettings.peekPackageLPr(childPackageName);
                            if (childPs == null) {
                                continue;
                            }
                            //【5.1.1】为该 child 创建一个 PackageInstalledInfo 实例，记录相关属性
                            // 保存到 outInfo.appearedChildPackages 集合中！
                            PackageInstalledInfo installRes = new PackageInstalledInfo();
                            installRes.name = childPackageName;
                            installRes.newUsers = childPs.queryInstalledUsers(allUserHandles, true);
                            installRes.pkg = mPackages.get(childPackageName);
                            installRes.uid = childPs.pkg.applicationInfo.uid;
                            if (outInfo.appearedChildPackages == null) {
                                outInfo.appearedChildPackages = new ArrayMap<>();
                            }
                            outInfo.appearedChildPackages.put(childPackageName, installRes);
                        }
                    }
                }
            }
        }

        return ret;
    }
```
到这里，这一阶段就分析结束了！


### 5.3.1 clearPackageStateForUserLIF

清理数据，可以看到这个方法里面执行的操作有很多：清楚数据等等：

```java
    private boolean clearPackageStateForUserLIF(PackageSetting ps, int userId,
            PackageRemovedInfo outInfo) {
        final PackageParser.Package pkg;
        synchronized (mPackages) {
            pkg = mPackages.get(ps.name);
        }
        //【1】如果是 UserHandle.USER_ALL，那么这里会返回当前的所有 user id！！
        final int[] userIds = (userId == UserHandle.USER_ALL) ? sUserManager.getUserIds()
                : new int[] {userId};

        //【2】遍历执行删除操作：
        for (int nextUserId : userIds) {
            if (DEBUG_REMOVE) {
                Slog.d(TAG, "Updating package:" + ps.name + " install state for user:"
                        + nextUserId);
            }

            destroyAppDataLIF(pkg, userId,
                    StorageManager.FLAG_STORAGE_DE | StorageManager.FLAG_STORAGE_CE);
            destroyAppProfilesLIF(pkg, userId);
            removeKeystoreDataIfNeeded(nextUserId, ps.appId);
            schedulePackageCleaning(ps.name, nextUserId, false);
            synchronized (mPackages) {
                if (clearPackagePreferredActivitiesLPw(ps.name, nextUserId)) {
                    scheduleWritePackageRestrictionsLocked(nextUserId);
                }
                resetUserChangesToRuntimePermissionsAndFlagsLPw(ps, nextUserId);
            }
        }

        if (outInfo != null) {
            outInfo.removedPackage = ps.name;
            outInfo.removedAppId = ps.appId;
            outInfo.removedUsers = userIds;
        }

        return true;
    }

```


### 5.3.2 markPackageUninstalledForUserLPw

修改在该 user 下的使用状态为 no install 的状态：

```java
    private void markPackageUninstalledForUserLPw(PackageSetting ps, UserHandle user) {
        final int[] userIds = (user == null || user.getIdentifier() == UserHandle.USER_ALL)
                ? sUserManager.getUserIds() : new int[] {user.getIdentifier()};
        for (int nextUserId : userIds) {
            if (DEBUG_REMOVE) {
                Slog.d(TAG, "Marking package:" + ps.name + " uninstalled for user:" + nextUserId);
            }
            //【1】设置对应的 PackageUserState 中的状态！
            ps.setUserState(nextUserId, 0, COMPONENT_ENABLED_STATE_DEFAULT,
                    false /*installed*/, true /*stopped*/, true /*notLaunched*/,
                    false /*hidden*/, false /*suspended*/, null, null, null,
                    false /*blockUninstall*/,
                    ps.readUserState(nextUserId).domainVerificationStatus, 0);
        }
    }
```
方法很简单，就不多说了！

### 5.3.3 shouldKeepUninstalledPackageLPr

```java
    private boolean shouldKeepUninstalledPackageLPr(String packageName) {
        return mKeepUninstalledPackages != null && mKeepUninstalledPackages.contains(packageName);
    }
```

# 6 PackageManagerService

接下来，我们分别分析下 sys app 和 data app 的卸载过程：

## 6.1 deleteSystemPackageLIF

```java
    private boolean deleteSystemPackageLIF(PackageParser.Package deletedPkg,
            PackageSetting deletedPs, int[] allUserHandles, int flags, PackageRemovedInfo outInfo,
            boolean writeSettings) {
        if (deletedPs.parentPackageName != null) {
            Slog.w(TAG, "Attempt to delete child system package " + deletedPkg.packageName);
            return false;
        }
        //【1】判断是否考虑用户限制！
        final boolean applyUserRestrictions
                = (allUserHandles != null) && (outInfo.origUsers != null);
        final PackageSetting disabledPs;

        //【2】判断该应用是否是一个被覆盖安装更新过的 sys app!!
        synchronized (mPackages) {
            disabledPs = mSettings.getDisabledSystemPkgLPr(deletedPs.name);
        }

        if (DEBUG_REMOVE) Slog.d(TAG, "deleteSystemPackageLI: newPs=" + deletedPkg.packageName
                + " disabledPs=" + disabledPs);
        
        //【3】只有覆盖安装过的 sys app 才能被卸载，实际上卸载的是处于 data 的那个 apk！
        if (disabledPs == null) {
            Slog.w(TAG, "Attempt to delete unknown system package "+ deletedPkg.packageName);
            return false;
        } else if (DEBUG_REMOVE) {
            Slog.d(TAG, "Deleting system pkg from data partition");
        }

        if (DEBUG_REMOVE) {
            if (applyUserRestrictions) {
                Slog.d(TAG, "Remembering install states:");
                for (int userId : allUserHandles) {
                    final boolean finstalled = ArrayUtils.contains(outInfo.origUsers, userId);
                    Slog.d(TAG, "   u=" + userId + " inst=" + finstalled);
                }
            }
        }

        //【4】设置 pkg 对应的 outInfo 的 isRemovedPackageSystemUpdate 为 true，表示移除的是更新！
        // 如果 pkg 有 child pkg，也要设置其对应的属性！
        outInfo.isRemovedPackageSystemUpdate = true;
        if (outInfo.removedChildPackages != null) {
            final int childCount = (deletedPs.childPackageNames != null)
                    ? deletedPs.childPackageNames.size() : 0;
            for (int i = 0; i < childCount; i++) {
                String childPackageName = deletedPs.childPackageNames.get(i);
                if (disabledPs.childPackageNames != null && disabledPs.childPackageNames
                        .contains(childPackageName)) {
                    PackageRemovedInfo childInfo = outInfo.removedChildPackages.get(
                            childPackageName);
                    if (childInfo != null) {
                        childInfo.isRemovedPackageSystemUpdate = true;
                    }
                }
            }
        }

        //【5】判断下覆盖安装前后的 versioncode，如果覆盖前的小，那么本次卸载后，数据也会被清除；
        // 如果相等，那么就保留数据！
        if (disabledPs.versionCode < deletedPs.versionCode) {
            flags &= ~PackageManager.DELETE_KEEP_DATA;
        } else {
            flags |= PackageManager.DELETE_KEEP_DATA;
        }

        //【*6.2】继续处理卸载，可以看到，此时和卸载 data app 的一样的了！
        boolean ret = deleteInstalledPackageLIF(deletedPs, true, flags, allUserHandles,
                outInfo, writeSettings, disabledPs.pkg);
        if (!ret) {
            return false;
        }

        //【6】接着，需要恢复 sys app！
        synchronized (mPackages) {
            // Reinstate the old system package
            //【*6.1.1】恢复 sys app 的安装数据！
            enableSystemPackageLPw(disabledPs.pkg);
            //【*6.1.2】移除所有的本地库！
            removeNativeBinariesLI(deletedPs);
        }

        //【7】准备重新扫描 sys app，首先会设置基本的扫描参数！
        // 如果是 pri app，还要设置 PARSE_IS_PRIVILEGED 标志位！
        if (DEBUG_REMOVE) Slog.d(TAG, "Re-installing system package: " + disabledPs);
        int parseFlags = mDefParseFlags
                | PackageParser.PARSE_MUST_BE_APK
                | PackageParser.PARSE_IS_SYSTEM
                | PackageParser.PARSE_IS_SYSTEM_DIR;
        //【*6.1.3】判断是否是 pri app！
        if (locationIsPrivileged(disabledPs.codePath)) {
            parseFlags |= PackageParser.PARSE_IS_PRIVILEGED;
        }

        final PackageParser.Package newPkg;
        try {
            //【8】重新扫描 sys app，这里我们在开机扫描的时候有分析过，不多说了！！
            newPkg = scanPackageTracedLI(disabledPs.codePath, parseFlags, SCAN_NO_PATHS, 0, null);
        } catch (PackageManagerException e) {
            Slog.w(TAG, "Failed to restore system package:" + deletedPkg.packageName + ": "
                    + e.getMessage());
            return false;
        }
        try {
            //【*6.1.4】更新共享库！
            updateSharedLibrariesLPw(newPkg, null);
        } catch (PackageManagerException e) {
            Slog.e(TAG, "updateAllSharedLibrariesLPw failed: " + e.getMessage());
        }
        
        //【*6.1.5】准备应用的数据目录！
        prepareAppDataAfterInstallLIF(newPkg);

        //【9】最后就是要想最新的信息持久化到本地文件：包括安装信息，偏好设置，权限等等！
        synchronized (mPackages) {
            //【9.1】读取最新的安装信息！
            PackageSetting ps = mSettings.mPackages.get(newPkg.packageName);

            //【9.2】将卸载前的权限授予信息拷贝到本次新安装的信息中！！
            ps.getPermissionsState().copyFrom(deletedPs.getPermissionsState());
            
            //【9.3-review】更新权限信息，这里我们在 pms 的启动时分析过，这里就不再细说了！
            // 这里会更新所有应用的权限信息，移除过时的运行时权限，自动授予安装时权限等！
            updatePermissionsLPw(newPkg.packageName, newPkg,
                    UPDATE_PERMISSIONS_ALL | UPDATE_PERMISSIONS_REPLACE_PKG);
            
            //【9.4】如果需要应用用户限制，会进入这个分支！
            if (applyUserRestrictions) {
                if (DEBUG_REMOVE) {
                    Slog.d(TAG, "Propagating install state across reinstall");
                }
                for (int userId : allUserHandles) {
                    final boolean installed = ArrayUtils.contains(outInfo.origUsers, userId);
                    if (DEBUG_REMOVE) {
                        Slog.d(TAG, "    user " + userId + " => " + installed);
                    }
                    //【9.4.1】重新设置在每个用户下的 install 状态！
                    ps.setInstalled(installed, userId);
                    
                    //【9.4.2】持久化所有用户下的运行时权限信息！
                    mSettings.writeRuntimePermissionsForUserLPr(userId, false);
                }
                //【9.4.3】持久化所有用户下的应用偏好设置！
                mSettings.writeAllUsersPackageRestrictionsLPr();
            }
            //【9.5】持久化 Settings 中的数据，包括 packages.xml，packages.list 等等！
            if (writeSettings) {
                mSettings.writeLPr();
            }
        }
        return true;
    }
```
整个逻辑很详细了，不多说了！！

### 6.1.1 enableSystemPackageLPw 

恢复 sys app 的安装信息！

```java
    private void enableSystemPackageLPw(PackageParser.Package pkg) {
        //【*6.1.1.1】恢复 pkg 的安装信息！
        mSettings.enableSystemPackageLPw(pkg.packageName);
        //【*6.1.1.1】恢复 pkg 的 child pkg 的安装信息！
        final int childCount = (pkg.childPackages != null) ? pkg.childPackages.size() : 0;
        for (int i = 0; i < childCount; i++) {
            PackageParser.Package childPkg = pkg.childPackages.get(i);
            mSettings.enableSystemPackageLPw(childPkg.packageName);
        }
    }
```
不多说了！

#### 6.1.1.1 Settings.enableSystemPackageLPw 

核心方法是在 Settings 中：

```java
    PackageSetting enableSystemPackageLPw(String name) {
        //【1】判断下是否被覆盖安装过！
        PackageSetting p = mDisabledSysPackages.get(name);
        if(p == null) {
            Log.w(PackageManagerService.TAG, "Package " + name + " is not disabled");
            return null;
        }
        //【2】取消掉 FLAG_UPDATED_SYSTEM_APP 标志位！
        if((p.pkg != null) && (p.pkg.applicationInfo != null)) {
            p.pkg.applicationInfo.flags &= ~ApplicationInfo.FLAG_UPDATED_SYSTEM_APP;
        }
        //【3-review】创建一个新的 PackageSetting 实例，同时将其添加到 Settings 内部的集合中！
        // 这个我们在 pms 开机中分析过！
        PackageSetting ret = addPackageLPw(name, p.realName, p.codePath, p.resourcePath,
                p.legacyNativeLibraryPathString, p.primaryCpuAbiString,
                p.secondaryCpuAbiString, p.cpuAbiOverrideString,
                p.appId, p.versionCode, p.pkgFlags, p.pkgPrivateFlags,
                p.parentPackageName, p.childPackageNames);

        //【4】从 mDisabledSysPackages 删除信息！
        mDisabledSysPackages.remove(name);
        return ret;
    }
```
继续分析！

### 6.1.2 removeNativeBinariesLI 

移除本地库：

```java
    private void removeNativeBinariesLI(PackageSetting ps) {
        //【1】移除 pkg 的本地库！
        if (ps != null) {
            NativeLibraryHelper.removeNativeBinariesLI(ps.legacyNativeLibraryPathString);
            //【1.1】移除 pkg 的 child pkg 的本地库！
            final int childCount = (ps.childPackageNames != null) ? ps.childPackageNames.size() : 0;
            for (int i = 0; i < childCount; i++) {
                PackageSetting childPs = null;
                synchronized (mPackages) {
                    childPs = mSettings.peekPackageLPr(ps.childPackageNames.get(i));
                }
                if (childPs != null) {
                    NativeLibraryHelper.removeNativeBinariesLI(childPs
                            .legacyNativeLibraryPathString);
                }
            }
        }
    }
```
核心是通过 NativeLibraryHelper 的相关接口来移除的，这里就不过多分析了！



### 6.1.3 locationIsPrivileged 

判断是否是 pri app：
```java
    static boolean locationIsPrivileged(File path) {
        try {
            final String privilegedAppDir = new File(Environment.getRootDirectory(), "priv-app")
                    .getCanonicalPath();
            //【1】核心逻辑，是否是以 /system/priv-app 开头的！
            return path.getCanonicalPath().startsWith(privilegedAppDir);
        } catch (IOException e) {
            Slog.e(TAG, "Unable to access code path " + path);
        }
        return false;
    }
```
不多说了！！


### 6.1.4 updateSharedLibrariesLPw 

更新共享库文件：

```java
    private void updateSharedLibrariesLPw(PackageParser.Package pkg,
            PackageParser.Package changingLib) throws PackageManagerException {
        if (pkg.usesLibraries != null || pkg.usesOptionalLibraries != null) {
            //【1】用于手机该 pkg 需要的所有共享库！
            final ArraySet<String> usesLibraryFiles = new ArraySet<>();
            //【2】处理 pkg.usesLibraries 指定依赖的库
            int N = pkg.usesLibraries != null ? pkg.usesLibraries.size() : 0;
            for (int i=0; i<N; i++) {
                //【2.1】要尝试在系统已有的共享库中找到对应的库！
                final SharedLibraryEntry file = mSharedLibraries.get(pkg.usesLibraries.get(i));
                if (file == null) {
                    throw new PackageManagerException(INSTALL_FAILED_MISSING_SHARED_LIBRARY,
                            "Package " + pkg.packageName + " requires unavailable shared library "
                            + pkg.usesLibraries.get(i) + "; failing!");
                }
                //【*6.1.4.1】将依赖的库加入到 usesLibraryFiles 中！
                addSharedLibraryLPw(usesLibraryFiles, file, changingLib);
            }
            //【3】处理 pkg.usesOptionalLibraries 指定依赖的库
            N = pkg.usesOptionalLibraries != null ? pkg.usesOptionalLibraries.size() : 0;
            for (int i=0; i<N; i++) {
                final SharedLibraryEntry file = mSharedLibraries.get(pkg.usesOptionalLibraries.get(i));
                if (file == null) {
                    Slog.w(TAG, "Package " + pkg.packageName
                            + " desires unavailable shared library "
                            + pkg.usesOptionalLibraries.get(i) + "; ignoring!");
                } else {
                    //【*6.1.4.1】将依赖的库加入到 usesLibraryFiles 中！
                    addSharedLibraryLPw(usesLibraryFiles, file, changingLib);
                }
            }
            //【4】最后将收集到的共享库文件路径保存到 pkg.usesLibraryFiles 中！
            N = usesLibraryFiles.size();
            if (N > 0) {
                pkg.usesLibraryFiles = usesLibraryFiles.toArray(new String[N]);
            } else {
                pkg.usesLibraryFiles = null;
            }
        }
    }
```
PackageParser.Package 内有如下的集合，表示该 pkg 依赖的共享库的名称：
```java
        public ArrayList<String> usesLibraries = null;
        public ArrayList<String> usesOptionalLibraries = null;
```
同时也有下面的集合，保存了依赖的所有的共享库的路径：
```java
    public String[] usesLibraryFiles = null;
```
不多说了！

#### 6.1.4.1 addSharedLibraryLPw 

参数 PackageParser.Package changingLib 表示我们改变了共享库的定义 apk，那么我们要将新的 apk 传进来，作为新的依赖！

```java
    private void addSharedLibraryLPw(ArraySet<String> usesLibraryFiles, SharedLibraryEntry file,
            PackageParser.Package changingLib) {
        //【1】如果共享库的 path 不为 null，那就直接加入到 usesLibraryFiles 中！
        if (file.path != null) {
            usesLibraryFiles.add(file.path);
            return;
        }
        //【2】否则就找到定义共享库的 apk！
        PackageParser.Package p = mPackages.get(file.apk);
        if (changingLib != null && changingLib.packageName.equals(file.apk)) {
            //【2.1】如果此时 changingLib 不为 null，同时匹配，那么我们就依赖这个 changingLib！
            if (p == null || p.packageName.equals(changingLib.packageName)) {
                p = changingLib;
            }
        }
        //【3】依赖定义 lib 的 apk！
        if (p != null) {
            usesLibraryFiles.addAll(p.getAllCodePaths());
        }
    }
```
不多说了！

### 6.1.5 prepareAppDataAfterInstallLIF
准备数据目录！
```java
    private void prepareAppDataAfterInstallLIF(PackageParser.Package pkg) {
        final PackageSetting ps;
        synchronized (mPackages) {
            //【1】保存 Kernel Map 数据！
            ps = mSettings.mPackages.get(pkg.packageName);
            mSettings.writeKernelMappingLPr(ps);
        }
        final UserManager um = mContext.getSystemService(UserManager.class);
        UserManagerInternal umInternal = getUserManagerInternal();
        for (UserInfo user : um.getUsers()) {
            final int flags;
            if (umInternal.isUserUnlockingOrUnlocked(user.id)) {
                flags = StorageManager.FLAG_STORAGE_DE | StorageManager.FLAG_STORAGE_CE;
            } else if (umInternal.isUserRunning(user.id)) {
                flags = StorageManager.FLAG_STORAGE_DE;
            } else {
                continue;
            }
            //【2】如果在该 user 下是安装状态，那就在该设备用户下准备数据目录！
            if (ps.getInstalled(user.id)) {
                //【*6.1.5.1】准备数据目录！
                prepareAppDataLIF(pkg, user.id, flags);
            }
        }
    }
```
继续分析：

#### 6.1.5.1 prepareAppDataLIF

这个方法会对 pkg 以及其 child pkg 准备目录：

```java
    private void prepareAppDataLIF(PackageParser.Package pkg, int userId, int flags) {
        if (pkg == null) {
            Slog.wtf(TAG, "Package was null!", new Throwable());
            return;
        }
        //【*6.1.5.2】准备父包的数据目录
        prepareAppDataLeafLIF(pkg, userId, flags);
        final int childCount = (pkg.childPackages != null) ? pkg.childPackages.size() : 0;
        for (int i = 0; i < childCount; i++) {
            //【*6.1.5.2】准备子包的数据目录
            prepareAppDataLeafLIF(pkg.childPackages.get(i), userId, flags);
        }
    }
```
继续分析：

#### 6.1.5.2 prepareAppDataLeafLIF

该方法是核心的方法：

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
            //【1】通过 installd 来准备数据目录！
            mInstaller.createAppData(volumeUuid, packageName, userId, flags,
                    appId, app.seinfo, app.targetSdkVersion);
        } catch (InstallerException e) {
            //【2】如果是系统应用，第一次准备失败后，还会在尝试一次！
            if (app.isSystemApp()) {
                logCriticalInfo(Log.ERROR, "Failed to create app data for " + packageName
                        + ", but trying to recover: " + e);
                //【2.1】先删除之前创建的脏目录！
                destroyAppDataLeafLIF(pkg, userId, flags);
                try {
                    //【2.2】再次创建数据目录；
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
        //【*6.1.5.2】为 native libs 创建链接！
        prepareAppDataContentsLeafLIF(pkg, userId, flags);
    }
```
继续分析：

#### 6.1.5.3 prepareAppDataContentsLeafLIF

为 native libs 创建链接：

```java
    private void prepareAppDataContentsLeafLIF(PackageParser.Package pkg, int userId, int flags) {
        final String volumeUuid = pkg.volumeUuid;
        final String packageName = pkg.packageName;
        final ApplicationInfo app = pkg.applicationInfo;

        if ((flags & StorageManager.FLAG_STORAGE_CE) != 0) {
            //【1】只为 32 位的 native libs 创建 link！
            if (app.primaryCpuAbi != null && !VMRuntime.is64BitAbi(app.primaryCpuAbi)) {
                final String nativeLibPath = app.nativeLibraryDir;
                try {
                    mInstaller.linkNativeLibraryDirectory(volumeUuid, packageName,
                            nativeLibPath, userId);
                } catch (InstallerException e) {
                    Slog.e(TAG, "Failed to link native for " + packageName + ": " + e);
                }
            }
        }
    }
```
先分析到这里！

## 6.2 deleteInstalledPackageLIF

卸载三方应用，或者覆盖更新的 sys app，那么我们调用的是该方法：

```java
    private boolean deleteInstalledPackageLIF(PackageSetting ps,
            boolean deleteCodeAndResources, int flags, int[] allUserHandles,
            PackageRemovedInfo outInfo, boolean writeSettings,
            PackageParser.Package replacingPackage) {
        synchronized (mPackages) {
        
            //【1】将要卸载的 apk 的 appid 保存到 PackageRemovedInfo 的 uid 属性中
            // 如果有 child pkg，对其也这样处理！
            if (outInfo != null) {
                outInfo.uid = ps.appId;
            }
            if (outInfo != null && outInfo.removedChildPackages != null) {
                final int childCount = (ps.childPackageNames != null)
                        ? ps.childPackageNames.size() : 0;
                for (int i = 0; i < childCount; i++) {
                    String childPackageName = ps.childPackageNames.get(i);
                    PackageSetting childPs = mSettings.mPackages.get(childPackageName);
                    if (childPs == null) {
                        return false;
                    }
                    PackageRemovedInfo childInfo = outInfo.removedChildPackages.get(
                            childPackageName);
                    if (childInfo != null) {
                        childInfo.uid = childPs.appId;
                    }
                }
            }
        }

        //【*6.2.1】删除 apk 的使用数据，如果有 child pkg，对其也这样处理！！
        removePackageDataLIF(ps, allUserHandles, outInfo, flags, writeSettings);
        final int childCount = (ps.childPackageNames != null) ? ps.childPackageNames.size() : 0;
        for (int i = 0; i < childCount; i++) {
            PackageSetting childPs;
            synchronized (mPackages) {
                childPs = mSettings.peekPackageLPr(ps.childPackageNames.get(i));
            }
            if (childPs != null) {
                PackageRemovedInfo childOutInfo = (outInfo != null
                        && outInfo.removedChildPackages != null)
                        ? outInfo.removedChildPackages.get(childPs.name) : null;
                //【2.1】对于 child pkg，这里的卸载 flags 比较特殊，如果 flags 设置了 DELETE_KEEP_DATA
                // 同时指定了 replacingPackage，而 replacingPackage 并不是其 parent pkg，这种情况，
                // 不需要保留数据，去掉 DELETE_KEEP_DATA 标志位；
                final int deleteFlags = (flags & DELETE_KEEP_DATA) != 0
                        && (replacingPackage != null
                        && !replacingPackage.hasChildPackage(childPs.name))
                        ? flags & ~DELETE_KEEP_DATA : flags;
                 //【*6.2.1】删除 child apk 的使用数据！
                removePackageDataLIF(childPs, allUserHandles, childOutInfo,
                        deleteFlags, writeSettings);
            }
        }

        //【2】只删除 pkg 的 apk 文件（child pkg 并不会被删除）
        if (ps.parentPackageName == null) {
            if (deleteCodeAndResources && (outInfo != null)) {
                //【*6.2.2】创建一个 intallArgs，用于卸载！
                outInfo.args = createInstallArgsForExisting(packageFlagsToInstallFlags(ps),
                        ps.codePathString, ps.resourcePathString, getAppDexInstructionSets(ps));
                if (DEBUG_SD_INSTALL) Slog.i(TAG, "args=" + outInfo.args);
            }
        }

        return true;
    }
```


### 6.2.1 removePackageDataLIF

删除 package 的数据！

```java
    private void removePackageDataLIF(PackageSetting ps, int[] allUserHandles,
            PackageRemovedInfo outInfo, int flags, boolean writeSettings) {
        String packageName = ps.name;
        if (DEBUG_REMOVE) Slog.d(TAG, "removePackageDataLI: " + ps);
        final PackageParser.Package deletedPkg;
        final PackageSetting deletedPs;
        synchronized (mPackages) {
            //【1】获得要被删除的 apk 的 PackageSetting 和 PackageParser.Package 对象！
            deletedPkg = mPackages.get(packageName);
            deletedPs = mSettings.mPackages.get(packageName);
            if (outInfo != null) {
                outInfo.removedPackage = packageName;
                outInfo.removedUsers = deletedPs != null
                        ? deletedPs.queryInstalledUsers(sUserManager.getUserIds(), true)
                        : null;
            }
        }
        //【*6.2.1.1】第一部移除，扫描的四大组件信息！
        removePackageLI(ps, (flags & REMOVE_CHATTY) != 0);
        
        //【2】如果 flags 没有设置 DELETE_KEEP_DATA，那么会清楚 apk 的数据！！
        if ((flags & PackageManager.DELETE_KEEP_DATA) == 0) {
            final PackageParser.Package resolvedPkg;
            if (deletedPkg != null) {
                resolvedPkg = deletedPkg;
            } else {
                resolvedPkg = new PackageParser.Package(ps.name);
                resolvedPkg.setVolumeUuid(ps.volumeUuid);
            }
            //【*6.2.1.2】删除 apk 的 data 数据！！
            destroyAppDataLIF(resolvedPkg, UserHandle.USER_ALL,
                    StorageManager.FLAG_STORAGE_DE | StorageManager.FLAG_STORAGE_CE);
                    
            //【*6.2.1.3】删除 apk 的 profiles 数据！！
            destroyAppProfilesLIF(resolvedPkg, UserHandle.USER_ALL);
            if (outInfo != null) {
                outInfo.dataRemoved = true; // 表示数据移除了；
            }
            //【*6.2.1.4】执行 package 清除！！
            schedulePackageCleaning(packageName, UserHandle.USER_ALL, true);
        }
        //【3】进一步处理！
        synchronized (mPackages) {
            if (deletedPs != null) {
                //【3.1】如果 flags 没有设置 DELETE_KEEP_DATA 标志位，那么执行其他的清楚操作！
                if ((flags & PackageManager.DELETE_KEEP_DATA) == 0) {
                
                    //【3.1.1】清楚 intentfilter verify 和 默认浏览器的设置数据！
                    clearIntentFilterVerificationsLPw(deletedPs.name, UserHandle.USER_ALL);
                    clearDefaultBrowserIfNeeded(packageName);
                    if (outInfo != null) {
                        //【3.1.1.1】移除 key set 信息！
                        mSettings.mKeySetManagerService.removeAppKeySetDataLPw(packageName);
    
                        //【*6.2.1.5】删除 pkg 的 PackageSetting 数据！
                        outInfo.removedAppId = mSettings.removePackageLPw(packageName);
                    }
                    
                    //【3.1.2-review】更新权限信息，这里我们在 pms 的启动时分析过，这里就不再细说了！
                    // 这里会更新所有应用的权限信息，移除过时的运行时权限，自动授予安装时权限等！
                    updatePermissionsLPw(deletedPs.name, null, 0);
                    
                    //【3.1.3】如果该应用是共享 shared user 的，进入这里！
                    if (deletedPs.sharedUser != null) {
                        for (int userId : UserManagerService.getInstance().getUserIds()) {
                        
                            //【3.1.3.1】更新该共享 shared uid 的权限，该 package 被移除掉，会导致和该
                            // 应用相关连的权限的变化，从而导致共享 shared uid 的 gids 发生变化！
                            final int userIdToKill = mSettings.updateSharedUserPermsLPw(deletedPs,
                                    userId);
                                    
                            if (userIdToKill == UserHandle.USER_ALL
                                    || userIdToKill >= UserHandle.USER_SYSTEM) {
                                //【3.1.3.1】如果共享 shared uid 的 gids 发生变化，杀掉该 uid 下的
                                // 所有的 app 进程！
                                mHandler.post(new Runnable() {
                                    @Override
                                    public void run() {
                                        killApplication(deletedPs.name, deletedPs.appId,
                                                KILL_APP_REASON_GIDS_CHANGED);
                                    }
                                });
                                break;
                            }
                        }
                    }
                    //【3.1.4】清除默认应用的数据，先不关注；
                    clearPackagePreferredActivitiesLPw(deletedPs.name, UserHandle.USER_ALL);
                }
                //【3.4】更新下在每个 user 下的安装状态！
                if (allUserHandles != null && outInfo != null && outInfo.origUsers != null) {
                    if (DEBUG_REMOVE) {
                        Slog.d(TAG, "Propagating install state across downgrade");
                    }
                    for (int userId : allUserHandles) {
                        final boolean installed = ArrayUtils.contains(outInfo.origUsers, userId);
                        if (DEBUG_REMOVE) {
                            Slog.d(TAG, "    user " + userId + " => " + installed);
                        }
                        ps.setInstalled(installed, userId);
                    }
                }
            }
            //【3.5】持久化 Settings 中的数据！！
            if (writeSettings) {
                mSettings.writeLPr();
            }
        }
        if (outInfo != null) {
            //【4】移除 key-store！
            removeKeystoreDataIfNeeded(UserHandle.USER_ALL, outInfo.removedAppId);
        }
    }
```
这里我们详细的分析了整个流程！！

#### 6.2.1.1 removePackageLI
移除 PackageSetting 对应的扫描数据：
```java
    void removePackageLI(PackageSetting ps, boolean chatty) {
        if (DEBUG_INSTALL) {
            if (chatty)
                Log.d(TAG, "Removing package " + ps.name);
        }
        synchronized (mPackages) {
            //【1】移除扫描信息！
            mPackages.remove(ps.name);
            final PackageParser.Package pkg = ps.pkg;
            if (pkg != null) {
                //【*6.2.1.1.1】移除四大组件，共享库解析对象！
                cleanPackageDataStructuresLILPw(pkg, chatty);
            }
        }
    }
```

这个就不多说了！！

##### 6.2.1.1.1 cleanPackageDataStructuresLILPw
用于删除 apk 的四大组件和共享库数据：
```java
    void cleanPackageDataStructuresLILPw(PackageParser.Package pkg, boolean chatty) {
        //【1】移除 provider！
        int N = pkg.providers.size();
        StringBuilder r = null;
        int i;
        for (i=0; i<N; i++) {
            PackageParser.Provider p = pkg.providers.get(i);
            mProviders.removeProvider(p);
            if (p.info.authority == null) {
                //【1.1】表示系统之前已经有相同 authority 的 provider，那么这个应用的 provider 是不会注册的！
                // 对于没有注册的 provider 不处理！
                continue;
            }
            String names[] = p.info.authority.split(";");
            for (int j = 0; j < names.length; j++) {
                if (mProvidersByAuthority.get(names[j]) == p) {
                    mProvidersByAuthority.remove(names[j]);
                    if (DEBUG_REMOVE) {
                        if (chatty)
                            Log.d(TAG, "Unregistered content provider: " + names[j]
                                    + ", className = " + p.info.name + ", isSyncable = "
                                    + p.info.isSyncable);
                    }
                }
            }
            if (DEBUG_REMOVE && chatty) {
                if (r == null) {
                    r = new StringBuilder(256);
                } else {
                    r.append(' ');
                }
                r.append(p.info.name);
            }
        }
        if (r != null) {
            if (DEBUG_REMOVE) Log.d(TAG, "  Providers: " + r);
        }
        //【2】移除 service！
        N = pkg.services.size();
        r = null;
        for (i=0; i<N; i++) {
            PackageParser.Service s = pkg.services.get(i);
            mServices.removeService(s);
            if (chatty) {
                if (r == null) {
                    r = new StringBuilder(256);
                } else {
                    r.append(' ');
                }
                r.append(s.info.name);
            }
        }
        if (r != null) {
            if (DEBUG_REMOVE) Log.d(TAG, "  Services: " + r);
        }
        //【3】移除 receiver！
        N = pkg.receivers.size();
        r = null;
        for (i=0; i<N; i++) {
            PackageParser.Activity a = pkg.receivers.get(i);
            mReceivers.removeActivity(a, "receiver");
            if (DEBUG_REMOVE && chatty) {
                if (r == null) {
                    r = new StringBuilder(256);
                } else {
                    r.append(' ');
                }
                r.append(a.info.name);
            }
        }
        if (r != null) {
            if (DEBUG_REMOVE) Log.d(TAG, "  Receivers: " + r);
        }
        //【4】移除 activity！
        N = pkg.activities.size();
        r = null;
        for (i=0; i<N; i++) {
            PackageParser.Activity a = pkg.activities.get(i);
            mActivities.removeActivity(a, "activity");
            if (DEBUG_REMOVE && chatty) {
                if (r == null) {
                    r = new StringBuilder(256);
                } else {
                    r.append(' ');
                }
                r.append(a.info.name);
            }
        }
        if (r != null) {
            if (DEBUG_REMOVE) Log.d(TAG, "  Activities: " + r);
        }
        //【5】移除定义的 permission，设置了 appop 标志为的权限，从 mAppOpPermissionPackages 也要移除！
        N = pkg.permissions.size();
        r = null;
        for (i=0; i<N; i++) {
            PackageParser.Permission p = pkg.permissions.get(i);
            BasePermission bp = mSettings.mPermissions.get(p.info.name);
            if (bp == null) {
                bp = mSettings.mPermissionTrees.get(p.info.name);
            }
            if (bp != null && bp.perm == p) {
                bp.perm = null;
                if (DEBUG_REMOVE && chatty) {
                    if (r == null) {
                        r = new StringBuilder(256);
                    } else {
                        r.append(' ');
                    }
                    r.append(p.info.name);
                }
            }
            if ((p.info.protectionLevel&PermissionInfo.PROTECTION_FLAG_APPOP) != 0) {
                ArraySet<String> appOpPkgs = mAppOpPermissionPackages.get(p.info.name);
                if (appOpPkgs != null) {
                    appOpPkgs.remove(pkg.packageName);
                }
            }
        }
        if (r != null) {
            if (DEBUG_REMOVE) Log.d(TAG, "  Permissions: " + r);
        }
        //【6】移除请求的 permission，设置了 appop 标志为的权限，从 mAppOpPermissionPackages 也要移除！！
        N = pkg.requestedPermissions.size();
        r = null;
        for (i=0; i<N; i++) {
            String perm = pkg.requestedPermissions.get(i);
            BasePermission bp = mSettings.mPermissions.get(perm);
            if (bp != null && (bp.protectionLevel&PermissionInfo.PROTECTION_FLAG_APPOP) != 0) {
                ArraySet<String> appOpPkgs = mAppOpPermissionPackages.get(perm);
                if (appOpPkgs != null) {
                    appOpPkgs.remove(pkg.packageName);
                    if (appOpPkgs.isEmpty()) {
                        mAppOpPermissionPackages.remove(perm);
                    }
                }
            }
        }
        if (r != null) {
            if (DEBUG_REMOVE) Log.d(TAG, "  Permissions: " + r);
        }
        //【7】移除请求的 instrumentation！
        N = pkg.instrumentation.size();
        r = null;
        for (i=0; i<N; i++) {
            PackageParser.Instrumentation a = pkg.instrumentation.get(i);
            mInstrumentation.remove(a.getComponentName());
            if (DEBUG_REMOVE && chatty) {
                if (r == null) {
                    r = new StringBuilder(256);
                } else {
                    r.append(' ');
                }
                r.append(a.info.name);
            }
        }
        if (r != null) {
            if (DEBUG_REMOVE) Log.d(TAG, "  Instrumentation: " + r);
        }
        //【8】移除 SharedLibraries！
        r = null;
        if ((pkg.applicationInfo.flags&ApplicationInfo.FLAG_SYSTEM) != 0) {
            // Only system apps can hold shared libraries.
            if (pkg.libraryNames != null) {
                for (i=0; i<pkg.libraryNames.size(); i++) {
                    String name = pkg.libraryNames.get(i);
                    SharedLibraryEntry cur = mSharedLibraries.get(name);
                    if (cur != null && cur.apk != null && cur.apk.equals(pkg.packageName)) {
                        mSharedLibraries.remove(name);
                        if (DEBUG_REMOVE && chatty) {
                            if (r == null) {
                                r = new StringBuilder(256);
                            } else {
                                r.append(' ');
                            }
                            r.append(name);
                        }
                    }
                }
            }
        }
        if (r != null) {
            if (DEBUG_REMOVE) Log.d(TAG, "  Libraries: " + r);
        }
    }
```
该阶段的逻辑比较简单，不多说了！


#### 6.2.1.2 destroyAppDataLIF ->[Leaf]

```java
    private void destroyAppDataLIF(PackageParser.Package pkg, int userId, int flags) {
        if (pkg == null) {
            Slog.wtf(TAG, "Package was null!", new Throwable());
            return;
        }
        //【1】删除父包的数据！！
        destroyAppDataLeafLIF(pkg, userId, flags);
        final int childCount = (pkg.childPackages != null) ? pkg.childPackages.size() : 0;
        for (int i = 0; i < childCount; i++) {
            //【2】删除子包的数据！
            destroyAppDataLeafLIF(pkg.childPackages.get(i), userId, flags);
        }
    }
```
继续看：
```java
    private void destroyAppDataLeafLIF(PackageParser.Package pkg, int userId, int flags) {
        final PackageSetting ps;
        synchronized (mPackages) {
            //【1】获得该应用的安装信息 PackageSetting ！
            ps = mSettings.mPackages.get(pkg.packageName);
        }
        for (int realUserId : resolveUserIds(userId)) {
            //【2】获得要删除的状态信息目录：
            final long ceDataInode = (ps != null) ? ps.getCeDataInode(realUserId) : 0;
            try {
                //【3】调用了 Installd 删除指定目录！
                mInstaller.destroyAppData(pkg.volumeUuid, pkg.packageName, realUserId, flags,
                        ceDataInode);
            } catch (InstallerException e) {
                Slog.w(TAG, String.valueOf(e));
            }
        }
    }
```
这里使用了 PackageSetting.getCeDataInode 方法：

```java
    long getCeDataInode(int userId) {
        return readUserState(userId).ceDataInode;
    }
```

该方法返回的是 PackageUserState.ceDataInode 的值！

#### 6.2.1.4 schedulePackageCleaning
```java
    void schedulePackageCleaning(String packageName, int userId, boolean andCode) {
        //【*6.2.1.1.3.1】这里会发送一个 START_CLEANING_PACKAGE 的消息给 PackageHandler ！
        final Message msg = mHandler.obtainMessage(START_CLEANING_PACKAGE,
                userId, andCode ? 1 : 0, packageName);
        if (mSystemReady) {
            msg.sendToTarget();
        } else {
            if (mPostSystemReadyMessages == null) {
                mPostSystemReadyMessages = new ArrayList<>();
            }
            mPostSystemReadyMessages.add(msg);
        }
    }
```
##### 6.2.1.4.1 Packagehandler.doHandleMessage[START_CLEANING_PACKAGE]
```java
    case START_CLEANING_PACKAGE: {
        Process.setThreadPriority(Process.THREAD_PRIORITY_DEFAULT);
        final String packageName = (String)msg.obj;
        final int userId = msg.arg1;
        final boolean andCode = msg.arg2 != 0;
        synchronized (mPackages) {
            if (userId == UserHandle.USER_ALL) {
                int[] users = sUserManager.getUserIds();
                for (int user : users) {
                    //【1】将 package 加入到 Settings 内部的 mPackagesToBeCleaned 集合中！
                    mSettings.addPackageToCleanLPw(
                            new PackageCleanItem(user, packageName, andCode));
                }
            } else {
                mSettings.addPackageToCleanLPw(
                        new PackageCleanItem(userId, packageName, andCode));
            }
        }
        Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
        //【*6.2.1.4.2】开始清理操作！
        startCleaningPackages();
    } break;
```
##### 6.2.1.4.2 startCleaningPackages
执行扩展存储清理操作：
```java
    void startCleaningPackages() {
        // reader
        if (!isExternalMediaAvailable()) {
            return;
        }
        synchronized (mPackages) {
            if (mSettings.mPackagesToBeCleaned.isEmpty()) {
                return;
            }
        }
        //【1】发送 action PackageManager.ACTION_CLEAN_EXTERNAL_STORAGE！
        Intent intent = new Intent(PackageManager.ACTION_CLEAN_EXTERNAL_STORAGE);
        //【2】目标组件服务：DefaultContainerService
        intent.setComponent(DEFAULT_CONTAINER_COMPONENT);
        IActivityManager am = ActivityManagerNative.getDefault();
        if (am != null) {
            try {
                //【2.1】启动服务！
                am.startService(null, intent, null, mContext.getOpPackageName(),
                        UserHandle.USER_SYSTEM);
            } catch (RemoteException e) {
            }
        }
    }
```
进入 DefaultContainerService.onHandleIntent 方法：
```java
    @Override
    protected void onHandleIntent(Intent intent) {
        if (PackageManager.ACTION_CLEAN_EXTERNAL_STORAGE.equals(intent.getAction())) {
            final IPackageManager pm = IPackageManager.Stub.asInterface(
                    ServiceManager.getService("package"));
            PackageCleanItem item = null;
            try {
                while ((item = pm.nextPackageToClean(item)) != null) {
                    final UserEnvironment userEnv = new UserEnvironment(item.userId);
                    eraseFiles(userEnv.buildExternalStorageAppDataDirs(item.packageName));
                    eraseFiles(userEnv.buildExternalStorageAppMediaDirs(item.packageName));
                    if (item.andCode) {
                        eraseFiles(userEnv.buildExternalStorageAppObbDirs(item.packageName));
                    }
                }
            } catch (RemoteException e) {
            }
        }
    }
```
这里的逻辑就不多说了！

#### 6.2.1.5 Settings.removePackageLPw

移除该 pkg 的安装信息：PackageSetting！
```java
    int removePackageLPw(String name) {
        final PackageSetting p = mPackages.get(name);
        if (p != null) {
            //【1】从 mPackages 中移除该 PackageSetting！！
            mPackages .remove(name);
            //【*6.2.1.5.1】如果其实 installer，还要修改和其相关的其他 pkg 的属性！
            removeInstallerPackageStatus(name);
            //【3】如果 pkg 是共享 uid，要解除相互引用！
            if (p.sharedUser != null) {
                p.sharedUser.removePackage(p);
                if (p.sharedUser.packages.size() == 0) {
                    mSharedUsers.remove(p.sharedUser.name);
                    //【3.1-review】从相关集合中删除该 ps！
                    removeUserIdLPw(p.sharedUser.userId);
                    return p.sharedUser.userId;
                }
            } else {
                //【3.1-review】从相关集合中删除该 ps！
                removeUserIdLPw(p.appId);
                return p.appId;
            }
        }
        return -1;
    }
```
该方法最后会返回 pkg 的 appID!

##### 6.2.1.5.1 Settings.removeInstallerPackageStatus

如果该 pkg 是 installer，那么
```java
    private void removeInstallerPackageStatus(String packageName) {
        //【1】如果岂不是 installer，返回！
        if (!mInstallerPackages.contains(packageName)) {
            return;
        }
        //【2】如果其是 installer，那么需要找到所有由其安装的 pkg，设置其属性！
        for (int i = 0; i < mPackages.size(); i++) {
            final PackageSetting ps = mPackages.valueAt(i);
            final String installerPackageName = ps.getInstallerPackageName();
            if (installerPackageName != null
                    && installerPackageName.equals(packageName)) {
                //【2.1】置空其 InstallerPackageName 属性，同时设置 ps.isOrphaned 为 ture！ 
                ps.setInstallerPackageName(null);
                ps.isOrphaned = true;
            }
        }
        //【3】从 mInstallerPackages 中移除该 pkg！
        mInstallerPackages.remove(packageName);
    }
```
不多说了！！

### 6.2.2 createInstallArgsForExisting - 用于卸载 apk

这里是针对已存在的应用创建一个 InstallArgs

```java
    private InstallArgs createInstallArgsForExisting(int installFlags, String codePath,
            String resourcePath, String[] instructionSets) {
        final boolean isInAsec;
        if (installOnExternalAsec(installFlags)) {
            //【1】如果是安装到外置的，那就创建 AsecInstallArgs！
            isInAsec = true;
        } else if (installForwardLocked(installFlags)
                && !codePath.startsWith(mDrmAppPrivateInstallDir.getAbsolutePath())) {
            //【2】对于 forward lock 安装，如果目录是 drm app pri
            // 那就创建 AsecInstallArgs！
            isInAsec = true;
        } else {
            isInAsec = false;
        }
        if (isInAsec) {
            //【3】创建 AsecInstallArgs 安装参数
            return new AsecInstallArgs(codePath, instructionSets,
                    installOnExternalAsec(installFlags), installForwardLocked(installFlags));
        } else {
            //【*6.2.2.1.1】一般情况下，会创建 FileInstallArgs，这里通过 FileInstallArgs 的另一构造器
            // 创建了实例，描述一个已经存在的 app！
            return new FileInstallArgs(codePath, resourcePath, instructionSets);
        }
    }
```
这里又回到了 5.5.3.1 的 FileInstallArgs 的相关创建！

#### 6.2.2.1 FileInstallArgs



##### 6.2.2.1.1 new FileInstallArgs

```java
    class FileInstallArgs extends InstallArgs {
        private File codeFile;
        private File resourceFile;
        // Example topology:
        // /data/app/com.example/base.apk
        // /data/app/com.example/split_foo.apk
        // /data/app/com.example/lib/arm/libfoo.so
        // /data/app/com.example/lib/arm64/libfoo.so
        // /data/app/com.example/dalvik/arm/base.apk@classes.dex
        //【1】安装一个新的 apk！
        FileInstallArgs(InstallParams params) {
            super(params.origin, params.move, params.observer, params.installFlags,
                    params.installerPackageName, params.volumeUuid,
                    params.getUser(), null /*instructionSets*/, params.packageAbiOverride,
                    params.grantedRuntimePermissions,
                    params.traceMethod, params.traceCookie, params.certificates);
            //【1.1】这里校验了下是否是  Forward Locked 的！
            if (isFwdLocked()) {
                throw new IllegalArgumentException("Forward locking only supported in ASEC");
            }
        }
        //【2】用于描述已存在的一个安装，显然，这里调用的是这个构造器！
        FileInstallArgs(String codePath, String resourcePath, String[] instructionSets) {
            super(OriginInfo.fromNothing(), null, null, 0, null, null, null, instructionSets,
                    null, null, null, 0, null /*certificates*/);
            this.codeFile = (codePath != null) ? new File(codePath) : null;
            this.resourceFile = (resourcePath != null) ? new File(resourcePath) : null;
        }
        ... ... ...
    }
```
我们看到 FileInstallArgs 有两个构造器！

一参数构造器用于创建安装过程中的 InstallArgs！

三参数构造器，用于描述一个已存在的安装，主要用于清除旧的安装，或者作为移动应用的时候的源数据，我们在 pms 开机初始化的过程中就已经看到过了！


##### 6.2.2.1.2 doPostDeleteLI

继续来看：
```java
    boolean doPostDeleteLI(boolean delete) {
        //【*6.2.2.1.3】清楚 apk 文件 和 dex 文件！
        cleanUpResourcesLI();
        return true;
    }
```
继续分析：

##### 6.2.2.1.3 cleanUpResourcesLI

继续分析：
```java
    void cleanUpResourcesLI() {
        List<String> allCodePaths = Collections.EMPTY_LIST;
        if (codeFile != null && codeFile.exists()) {
            try {
                //【1】收集 apk path！
                final PackageLite pkg = PackageParser.parsePackageLite(codeFile, 0);
                allCodePaths = pkg.getAllCodePaths();
            } catch (PackageParserException e) {
                // Ignored; we tried our best
            }
        }
        //【2】清除 apk 文件，调用 mInstaller.rmPackageDir 删除！
        cleanUp();
        //【3】清除 dex files，调用 mInstaller.rmdex 删除！
        removeDexFiles(allCodePaths, instructionSets);
    }
```

# 7 remove split apk - 移除 split（模块）apk

在最上面的分析中，我们知道，如果要删除的是 split apk，那么会进入另外一个接口，这个接口和 install 的流程很类似，我们在这里做一下分析：

因为和 install 有很多相似之处，所以我省略掉一些无关紧要的代码段！！

## 7.1 PackageInstallerService


### 7.1.1 createSession(Internal) - 创建事务

创建事务：

```java
    @Override
    public int createSession(SessionParams params, String installerPackageName, int userId) {
        try {
            //【1】继续来看！
            return createSessionInternal(params, installerPackageName, userId);
        } catch (IOException e) {
            throw ExceptionUtils.wrap(e);
        }
    }
```
createSession 方法调用了 createSessionInternal 方法：

```java
    private int createSessionInternal(SessionParams params, String installerPackageName, int userId)
            throws IOException {
        final int callingUid = Binder.getCallingUid();
        //【1】权限检查！
        mPm.enforceCrossUserPermission(callingUid, userId, true, true, "createSession");

        //【2】用户操作检查！
        if (mPm.isUserRestricted(userId, UserManager.DISALLOW_INSTALL_APPS)) {
            throw new SecurityException("User restriction prevents installing");
        }

        //【3】如果调用进程的 uid 是 SHELL_UID 或者 ROOT_UID，那么 installFlags 增加爱你 INSTALL_FROM_ADB
        // 表示通过 adb 进行安装！
        if ((callingUid == Process.SHELL_UID) || (callingUid == Process.ROOT_UID)) {
            params.installFlags |= PackageManager.INSTALL_FROM_ADB;

        } else {
            // 如果不是 shell or root，校验下 package 是否属于 uid，
            mAppOps.checkPackage(callingUid, installerPackageName);
            
            // 取消 INSTALL_FROM_ADB 和 INSTALL_ALL_USERS 标志位，设置 INSTALL_REPLACE_EXISTING 标志位！
            params.installFlags &= ~PackageManager.INSTALL_FROM_ADB;
            params.installFlags &= ~PackageManager.INSTALL_ALL_USERS;
            params.installFlags |= PackageManager.INSTALL_REPLACE_EXISTING;
        }

        //【4】如果 installFlags 设置了 INSTALL_GRANT_RUNTIME_PERMISSIONS 标志位，那需要判断调用者是否有 
        // INSTALL_GRANT_RUNTIME_PERMISSIONS 权限！
        if ((params.installFlags & PackageManager.INSTALL_GRANT_RUNTIME_PERMISSIONS) != 0
                && mContext.checkCallingOrSelfPermission(Manifest.permission
                .INSTALL_GRANT_RUNTIME_PERMISSIONS) == PackageManager.PERMISSION_DENIED) {
            throw new SecurityException("You need the "
                    + "android.permission.INSTALL_GRANT_RUNTIME_PERMISSIONS permission "
                    + "to use the PackageManager.INSTALL_GRANT_RUNTIME_PERMISSIONS flag");
        }

        //【5】调整应用的 icon 图标！
        if (params.appIcon != null) {
            final ActivityManager am = (ActivityManager) mContext.getSystemService(
                    Context.ACTIVITY_SERVICE);
            final int iconSize = am.getLauncherLargeIconSize();
            if ((params.appIcon.getWidth() > iconSize * 2)
                    || (params.appIcon.getHeight() > iconSize * 2)) {
                params.appIcon = Bitmap.createScaledBitmap(params.appIcon, iconSize, iconSize,
                        true);
            }
        }
        //【6】检查 mode 取值是否正确！
        switch (params.mode) {
            case SessionParams.MODE_FULL_INSTALL:
            case SessionParams.MODE_INHERIT_EXISTING:
                break;
            default:
                throw new IllegalArgumentException("Invalid install mode: " + params.mode);
        }

        //【7】根据 installFlags 设置，调整安装位置，如果用户显示设置了位置，系统会对其进行检查，否则
        // 系统会选择合适的位置！
        if ((params.installFlags & PackageManager.INSTALL_INTERNAL) != 0) {
            //【7.1】如果显式指定内置，判断是否合适安装！
            if (!PackageHelper.fitsOnInternal(mContext, params.sizeBytes)) {
                throw new IOException("No suitable internal storage available");
            }
            
        } else if ((params.installFlags & PackageManager.INSTALL_EXTERNAL) != 0) {
            //【7.2】如果显式指定外置，判断是否合适安装！
            if (!PackageHelper.fitsOnExternal(mContext, params.sizeBytes)) {
                throw new IOException("No suitable external storage available");
            }
            
        } else if ((params.installFlags & PackageManager.INSTALL_FORCE_VOLUME_UUID) != 0) {
            params.setInstallFlagsInternal();
            
        } else {
            //【7.4】默认情况下，进入这里，setInstallFlagsInternal 方法会设置 INSTALL_INTERNAL 标志位
            // 取消 INSTALL_EXTERNAL 标志位！
            params.setInstallFlagsInternal();
            // 选择最好的位置来安装！
            final long ident = Binder.clearCallingIdentity();
            try {
                params.volumeUuid = PackageHelper.resolveInstallVolume(mContext,
                        params.appPackageName, params.installLocation, params.sizeBytes);
            } finally {
                Binder.restoreCallingIdentity(ident);
            }
        }

        final int sessionId;
        final PackageInstallerSession session;
        synchronized (mSessions) {
            //【7.5-review】判断，同一个 uid 是否有过多的正在处理的 Session，如果超过了 1024 个，那就不能安装！
            final int activeCount = getSessionCount(mSessions, callingUid);
            if (activeCount >= MAX_ACTIVE_SESSIONS) {
                throw new IllegalStateException(
                        "Too many active sessions for UID " + callingUid);
            }
            // 同样，判断同一个 uid，是否已经提交了过多的 Session，如果超过了 1048576 个，那当前就不能执行安装！
            final int historicalCount = getSessionCount(mHistoricalSessions, callingUid);
            if (historicalCount >= MAX_HISTORICAL_SESSIONS) {
                throw new IllegalStateException(
                        "Too many historical sessions for UID " + callingUid);
            }
    
            //【7.6-review】给本次安装分配一个事务 id！
            sessionId = allocateSessionIdLocked();
        }
        final long createdMillis = System.currentTimeMillis();

        //【8】决定安装目录，因为默认是内置空间，这里会直接进入 buildStageDir 方法！
        File stageDir = null;
        String stageCid = null;
        if ((params.installFlags & PackageManager.INSTALL_INTERNAL) != 0) {
            final boolean isEphemeral =
                    (params.installFlags & PackageManager.INSTALL_EPHEMERAL) != 0;
            //【8.1-review】创建文件临时目录；/data/app/vmdl[sessionId].tmp！
            stageDir = buildStageDir(params.volumeUuid, sessionId, isEphemeral);

        } else {
            // 如果是外置，会直接返回 "smdl" + sessionId + ".tmp"
            stageCid = buildExternalStageCid(sessionId);
        }

        //【*7.2.1-review】创建 PackageInstallerSession 对象！
        session = new PackageInstallerSession(mInternalCallback, mContext, mPm,
                mInstallThread.getLooper(), sessionId, userId, installerPackageName, callingUid,
                params, createdMillis, stageDir, stageCid, false, false);
    
        synchronized (mSessions) {
            //【8】将新创建的 PackageInstallerSession 添加到 mSessions 集合中！
            mSessions.put(sessionId, session);
        }

        //【*9-review】通知有新的事务创建了，这里是直接回调 Callback 的接口！！
        mCallbacks.notifySessionCreated(session.sessionId, session.userId);
    
        //【*10-review】持久化事务 Session！
        writeSessionsAsync();

        return sessionId;
    }
```
整个流程和 install 很像，我们可以大胆推测，其在移除了 split apk 后，还会把主 apk 再 install 一次！！


### 7.1.2 openSession - 获得事务

openSession 方法可以获得 id 对应的 PackageInstallerSession！
```java
    @Override
    public IPackageInstallerSession openSession(int sessionId) {
        try {
            //【×7.1.2.1】调用另外一个方法！
            return openSessionInternal(sessionId);
        } catch (IOException e) {
            throw ExceptionUtils.wrap(e);
        }
    }
```

#### 7.1.2.1 openSessionInternal

```java
    private IPackageInstallerSession openSessionInternal(int sessionId) throws IOException {
        synchronized (mSessions) {
            final PackageInstallerSession session = mSessions.get(sessionId);
            //【1-review】判断 uid 是否被允许获得该事务！
            if (session == null || !isCallingUidOwner(session)) {
                throw new SecurityException("Caller has no access to session " + sessionId);
            }
            session.open();
            return session;
        }
    }
```

## 7.2 PackageInstallerSession

### 7.2.1 new PackageInstallerSession - 事务实例

创建 PackageInstallerSession，对前面的 SessionParams 再次封装！

```java
public class PackageInstallerSession extends IPackageInstallerSession.Stub {
    public PackageInstallerSession(PackageInstallerService.InternalCallback callback,
            Context context, PackageManagerService pm, Looper looper, int sessionId, int userId,
            String installerPackageName, int installerUid, SessionParams params, long createdMillis,
            File stageDir, String stageCid, boolean prepared, boolean sealed) {
        //【1-review】InternalCallback 回调！
        mCallback = callback;
        mContext = context;
        mPm = pm;
        //【2】创建 Handler 绑定到子线程 mInstallThread，该子线程是在 PackageInstallerService 构造器中创建的！
        //【2.1-review】这里通过 mHandlerCallback 指定了一个回调函数！
        mHandler = new Handler(looper, mHandlerCallback);
        //【3】基本属性保存
        this.sessionId = sessionId;
        this.userId = userId;
        this.installerPackageName = installerPackageName;
        this.installerUid = installerUid;
        this.params = params;
        this.createdMillis = createdMillis;
        this.stageDir = stageDir; // 内置临时目录：/data/app/vmdl[sessionId].tmp；
        this.stageCid = stageCid; // 默认为 null；
        if ((stageDir == null) == (stageCid == null)) {
            throw new IllegalArgumentException(
                    "Exactly one of stageDir or stageCid stage must be set");
        }
        mPrepared = prepared; // 传入 false；
        mSealed = sealed; // 传入 false；
        //【4】获得 DevicePolicyManager 对象，用于静默安装相关的判断，如果是安装者是设备拥有者，
        // 可以不检查权限，直接静默安装！
        DevicePolicyManager dpm = (DevicePolicyManager) mContext.getSystemService(
                Context.DEVICE_POLICY_SERVICE);
        //【5】校验安装者 uid 是否有 INSTALL_PACKAGES 权限！
        final boolean isPermissionGranted =
                (mPm.checkUidPermission(android.Manifest.permission.INSTALL_PACKAGES, installerUid)
                        == PackageManager.PERMISSION_GRANTED);
        //【6】安装者是否是 root 用户！
        final boolean isInstallerRoot = (installerUid == Process.ROOT_UID);
        //【7】是否强制提醒！
        final boolean forcePermissionPrompt =
                (params.installFlags & PackageManager.INSTALL_FORCE_PERMISSION_PROMPT) != 0;
        //【8】安装者是否是设备拥有者自身！
        mIsInstallerDeviceOwner = (dpm != null) && dpm.isDeviceOwnerAppOnCallingUser(
                installerPackageName);
        //【8】如果 mPermissionsAccepted 为 true，那么我们就可以静默安装！
        if ((isPermissionGranted
                        || isInstallerRoot
                        || mIsInstallerDeviceOwner)
                && !forcePermissionPrompt) {
            mPermissionsAccepted = true;
        } else {
            mPermissionsAccepted = false;
        }
        final long identity = Binder.clearCallingIdentity();
        try {
            final int uid = mPm.getPackageUid(PackageManagerService.DEFAULT_CONTAINER_PACKAGE,
                    PackageManager.MATCH_SYSTEM_ONLY, UserHandle.USER_SYSTEM);
            defaultContainerGid = UserHandle.getSharedAppGid(uid);
        } finally {
            Binder.restoreCallingIdentity(identity);
        }
    }
```
可以看到 PackageInstallerSession 除了用来表示一个 Session 之外，由于继承了 IPackageInstallerSession.Stub，因此其还可以作为服务端的桩对象，进行跨进程的通信！

### 7.2.2 removeSplit

```java
    @Override
    public void removeSplit(String splitName) {
        //【1】首先如果要卸载 split apk，必须指定 parent pkg！
        if (TextUtils.isEmpty(params.appPackageName)) {
            throw new IllegalStateException("Must specify package name to remove a split");
        }
        try {
            //【*7.2.3】创建一个 mark 标记，来记录那些需要被移除的 split apk！！
            createRemoveSplitMarker(splitName);
        } catch (IOException e) {
            throw ExceptionUtils.wrap(e);
        }
    }
```

### 7.2.3 createRemoveSplitMarker

```java
    private void createRemoveSplitMarker(String splitName) throws IOException {
        try {
            //【1】创建了一个标记名称：
            final String markerName = splitName + REMOVE_SPLIT_MARKER_EXTENSION;
            if (!FileUtils.isValidExtFilename(markerName)) {
                throw new IllegalArgumentException("Invalid marker: " + markerName);
            }
            //【2】在临时目录下创建创建了一个 "splitName".removed 的文件！
            final File target = new File(resolveStageDir(), markerName);
            target.createNewFile();
            Os.chmod(target.getAbsolutePath(), 0 /*mode*/);
        } catch (ErrnoException e) {
            throw e.rethrowAsIOException();
        }
    }
```

这里的 REMOVE_SPLIT_MARKER_EXTENSION 是一个字符串后缀：

```java
private static final String REMOVE_SPLIT_MARKER_EXTENSION = ".removed";
```

#### 7.2.3.1 resolveStageDir
```java
    private File resolveStageDir() throws IOException {
        synchronized (mLock) {
            if (mResolvedStageDir == null) {
                if (stageDir != null) {
                    //【1】返回的就是前面的 stageDir！
                    mResolvedStageDir = stageDir;
                } else {
                    final String path = PackageHelper.getSdDir(stageCid);
                    if (path != null) {
                        mResolvedStageDir = new File(path);
                    } else {
                        throw new IOException("Failed to resolve path to container " + stageCid);
                    }
                }
            }
            return mResolvedStageDir;
        }
    }
```


### 7.2.4 commitLocked

按照流程，我们进入了 commitLocked 中：

参数 PackageInfo pkgInfo 和 ApplicationInfo appInfo 分别表示已经安装的主 apk 的信息对象！

```java
    private void commitLocked(PackageInfo pkgInfo, ApplicationInfo appInfo)
            throws PackageManagerException {
        if (mDestroyed) {
            throw new PackageManagerException(INSTALL_FAILED_INTERNAL_ERROR, "Session destroyed");
        }
        if (!mSealed) {
            throw new PackageManagerException(INSTALL_FAILED_INTERNAL_ERROR, "Session not sealed");
        }

        try {
            //【*7.2.3.1】获得 tmp 目录，也就是前面我们的 .removed 文件所在的目录；
            resolveStageDir();
        } catch (IOException e) {
            throw new PackageManagerException(INSTALL_FAILED_CONTAINER_ERROR,
                    "Failed to resolve stage location", e);
        }

        //【*7.2.4.1】校验安装有效性！
        validateInstallLocked(pkgInfo, appInfo);

        Preconditions.checkNotNull(mPackageName);
        Preconditions.checkNotNull(mSignatures);
        Preconditions.checkNotNull(mResolvedBaseFile);

        if (!mPermissionsAccepted) { // 这里我们就跳过，不分析，install 的时候分析过！
            final Intent intent = new Intent(PackageInstaller.ACTION_CONFIRM_PERMISSIONS);
            intent.setPackage(mContext.getPackageManager().getPermissionControllerPackageName());
            intent.putExtra(PackageInstaller.EXTRA_SESSION_ID, sessionId);
            try {
                mRemoteObserver.onUserActionRequired(intent);
            } catch (RemoteException ignored) {
            }
            close();
            return;
        }
        if (stageCid != null) {
            final long finalSize = calculateInstalledSize();
            resizeContainer(stageCid, finalSize);
        }

        //【1】如果安装方式是继承已存在的 apk，那我们要尝试继承！！
        // 显然，对于卸载 split apk 肯定是会走这一步的！！
        if (params.mode == SessionParams.MODE_INHERIT_EXISTING) {
            try {
                //【1.1】mResolvedInheritedFiles 中都是需要从之前安装的目录下继承过来的 apk 和 odex 文件；
                final List<File> fromFiles = mResolvedInheritedFiles;
                //【1.2】这是我们本次安装的目录；
                final File toDir = resolveStageDir();
                if (LOGD) Slog.d(TAG, "Inherited files: " + mResolvedInheritedFiles);
                if (!mResolvedInheritedFiles.isEmpty() && mInheritedFilesBase == null) {
                    throw new IllegalStateException("mInheritedFilesBase == null");
                }
                //【1.3】如果可以直接建立 link 的话，不行的话，就 copy！
                if (isLinkPossible(fromFiles, toDir)) {
                    if (!mResolvedInstructionSets.isEmpty()) {
                        final File oatDir = new File(toDir, "oat");
                        createOatDirs(mResolvedInstructionSets, oatDir);
                    }
                    linkFiles(fromFiles, toDir, mInheritedFilesBase);
                } else {
                    //【1.4】拷贝已经安装的 apk 到新目录下！
                    copyFiles(fromFiles, toDir);
                }
            } catch (IOException e) {
                throw new PackageManagerException(INSTALL_FAILED_INSUFFICIENT_STORAGE,
                        "Failed to inherit existing install", e);
            }
        }

        mInternalProgress = 0.5f;
        computeProgressLocked(true);
        extractNativeLibraries(mResolvedStageDir, params.abiOverride);
        if (stageCid != null) {
            finalizeAndFixContainer(stageCid);
        }
        final IPackageInstallObserver2 localObserver = new IPackageInstallObserver2.Stub() {
            @Override
            public void onUserActionRequired(Intent intent) {
                throw new IllegalStateException();
            }
            @Override
            public void onPackageInstalled(String basePackageName, int returnCode, String msg,
                    Bundle extras) {；
                destroyInternal();
                dispatchSessionFinished(returnCode, msg, extras);
            }
        };
        final UserHandle user;
        if ((params.installFlags & PackageManager.INSTALL_ALL_USERS) != 0) {
            user = UserHandle.ALL;
        } else {
            user = new UserHandle(userId);
        }
        mRelinquished = true;

        //【2】开始安装！
        mPm.installStage(mPackageName, stageDir, stageCid, localObserver, params,
                installerPackageName, installerUid, user, mCertificates);
    }
```

对于卸载 split apk 的情况：

可以看到和 install 是一样的，唯独不一样的是，会将之前已经安装的 /data/app/package-Name/ 中除了要卸载的 split apk 以外的其他 apk 拷贝到新创建的目录下，重新安装；


#### 7.2.4.1 validateInstallLocked

校验安装有效性，这里的 mResolvedStageDir 就是前面的 /data/app/vmdl[sessionId].tmp 目录！

removeSplitList 用于表示要删除的 split apk 列表：

```java
    private void validateInstallLocked(PackageInfo pkgInfo, ApplicationInfo appInfo)
            throws PackageManagerException {
        mPackageName = null;
        mVersionCode = -1;
        mSignatures = null;
        mResolvedBaseFile = null;
        mResolvedStagedFiles.clear();
        mResolvedInheritedFiles.clear();

        //【1】返回 /data/app/vmdl[sessionId].tmp 目录下所有的 .removed 文件！
        // 去除后缀，将前缀名保存到 removeSplitList！
        final File[] removedFiles = mResolvedStageDir.listFiles(sRemovedFilter);
        final List<String> removeSplitList = new ArrayList<>();
        if (!ArrayUtils.isEmpty(removedFiles)) {
            for (File removedFile : removedFiles) {
                final String fileName = removedFile.getName();
                final String splitName = fileName.substring(
                        0, fileName.length() - REMOVE_SPLIT_MARKER_EXTENSION.length());
                removeSplitList.add(splitName);
            }
        }
        //【2】返回 /data/app/vmdl[sessionId].tmp 目录下所有的非 .removed 文件！
        // 并判断是否正常，如果该目录下没有任何 apk 和 .removed 文件，那么抛出异常！
        final File[] addedFiles = mResolvedStageDir.listFiles(sAddedFilter);
        if (ArrayUtils.isEmpty(addedFiles) && removeSplitList.size() == 0) {
            throw new PackageManagerException(INSTALL_FAILED_INVALID_APK, "No packages staged");
        }
        //【3】遍历该目录下的非 .removed 文件，解析其中的 apk 文件，也就是我们之前 copy 到这里的目标文件！
        // 对于卸载 split apk 的情况，显然这里不会进入，因为我们的目录里面只有 .removed 文件！
        final ArraySet<String> stagedSplits = new ArraySet<>();
        for (File addedFile : addedFiles) {
            final ApkLite apk;
            try {
                //【3.1】解析要安装的 apk，具体的流程这里就不分析了！
                apk = PackageParser.parseApkLite(
                        addedFile, PackageParser.PARSE_COLLECT_CERTIFICATES);
            } catch (PackageParserException e) {
                throw PackageManagerException.from(e);
            }
            //【3.2】将其添加到 stagedSplits 中，注意 base.apk 的 apk.splitName 为 null！
            if (!stagedSplits.add(apk.splitName)) {
                throw new PackageManagerException(INSTALL_FAILED_INVALID_APK,
                        "Split " + apk.splitName + " was defined multiple times");
            }
            //【3.3】将第一个被解析 apk 的包名，版本号，签名，证书保存下载，这个目录下的其他 apk 
            // 的这几项要和其保持一致！
            if (mPackageName == null) {
                mPackageName = apk.packageName;
                mVersionCode = apk.versionCode;
            }
            if (mSignatures == null) {
                mSignatures = apk.signatures;
                mCertificates = apk.certificates;
            }
            //【3.4】校验 apk 关联性，校验包名。版本号，签名；
            assertApkConsistent(String.valueOf(addedFile), apk);
            //【3.5】设置 apk 文件的目标名称！
            final String targetName;
            if (apk.splitName == null) {
                targetName = "base.apk"; // 一般情况下！
            } else {
                targetName = "split_" + apk.splitName + ".apk";
            }
            if (!FileUtils.isValidExtFilename(targetName)) {
                throw new PackageManagerException(INSTALL_FAILED_INVALID_APK,
                        "Invalid filename: " + targetName);
            }
            //【3.6】当 addedFile 命名不标准的话，会改名;
            final File targetFile = new File(mResolvedStageDir, targetName);
            if (!addedFile.equals(targetFile)) {
                addedFile.renameTo(targetFile);
            }
            //【3.7】找到了 base apk，将其保存到 mResolvedBaseFile！
            // 同时将其添加到 mResolvedStagedFiles 中！
            if (apk.splitName == null) {
                mResolvedBaseFile = targetFile;
            }
            mResolvedStagedFiles.add(targetFile);
        }
        //【4】处理 .removed 文件，此时我们正在卸载 split apk，所以会进入这里！！
        if (removeSplitList.size() > 0) {
            //【4.1】如果找不到该 split apk 文件的话，抛出异常！
            for (String splitName : removeSplitList) {
                if (!ArrayUtils.contains(pkgInfo.splitNames, splitName)) {
                    throw new PackageManagerException(INSTALL_FAILED_INVALID_APK,
                            "Split not found: " + splitName);
                }
            }
            //【4.2】再次获得要安装的应用的包名，版本号，签名！
            if (mPackageName == null) {
                mPackageName = pkgInfo.packageName;
                mVersionCode = pkgInfo.versionCode;
            }
            if (mSignatures == null) {
                mSignatures = pkgInfo.signatures;
            }
        }
        //【5】处理安装模式！
        if (params.mode == SessionParams.MODE_FULL_INSTALL) {
            //【5.1】全量安装必须要有 base.apk
            if (!stagedSplits.contains(null)) {
                throw new PackageManagerException(INSTALL_FAILED_INVALID_APK,
                        "Full install must include a base package");
            }
        } else {
            //【5.2】部分安装必须基于现有的安装！
            if (appInfo == null) {
                throw new PackageManagerException(INSTALL_FAILED_INVALID_APK,
                        "Missing existing base package for " + mPackageName);
            }
            //【5.3】获得已存在的 apk 安装信息！
            final PackageLite existing;
            final ApkLite existingBase;
            try {
                //【5.3.1】对于安装 split apk，我们会解析下已存在的 split apk！！
                existing = PackageParser.parsePackageLite(new File(appInfo.getCodePath()), 0);
                existingBase = PackageParser.parseApkLite(new File(appInfo.getBaseCodePath()),
                        PackageParser.PARSE_COLLECT_CERTIFICATES);
            } catch (PackageParserException e) {
                throw PackageManagerException.from(e);
            }
            //【*4.3.2.1】再次校验要本次要安装的 apk 和已存在的 apk 是有关联，包括包名，签名，版本号！
            assertApkConsistent("Existing base", existingBase);
            //【5.4】继承已有的 base apk，如果没有指定安装的 apk！！
            if (mResolvedBaseFile == null) {
                mResolvedBaseFile = new File(appInfo.getBaseCodePath());
                mResolvedInheritedFiles.add(mResolvedBaseFile);
            }
            //【5.5】继承已有的 split apk！！
            if (!ArrayUtils.isEmpty(existing.splitNames)) {
                for (int i = 0; i < existing.splitNames.length; i++) {
                    final String splitName = existing.splitNames[i];
                    final File splitFile = new File(existing.splitCodePaths[i]);
                    final boolean splitRemoved = removeSplitList.contains(splitName);
                    if (!stagedSplits.contains(splitName) && !splitRemoved) {
                        mResolvedInheritedFiles.add(splitFile);
                    }
                }
            }
            //【5.6】继承已有的 oat 相关文件！！
            final File packageInstallDir = (new File(appInfo.getBaseCodePath())).getParentFile();
            mInheritedFilesBase = packageInstallDir;
            final File oatDir = new File(packageInstallDir, "oat");
            if (oatDir.exists()) {
                final File[] archSubdirs = oatDir.listFiles();
                if (archSubdirs != null && archSubdirs.length > 0) {
                    final String[] instructionSets = InstructionSets.getAllDexCodeInstructionSets();
                    for (File archSubDir : archSubdirs) {
                        // Skip any directory that isn't an ISA subdir.
                        if (!ArrayUtils.contains(instructionSets, archSubDir.getName())) {
                            continue;
                        }
                        // 将要继承的 oat 目录文件名添加到 mResolvedInstructionSets！
                        mResolvedInstructionSets.add(archSubDir.getName());
                        List<File> oatFiles = Arrays.asList(archSubDir.listFiles());
                        if (!oatFiles.isEmpty()) {
                            // 将要继承的 odex 相关文件添加到 mResolvedInheritedFiles！
                            mResolvedInheritedFiles.addAll(oatFiles);
                        }
                    }
                }
            }
        }
    }
```
其实这里，我们可以看到。对于卸载 split apk 的情况，我们会收集之前安装的目录下的所有不再 remove 列表中的 apk 文件到 mResolvedInheritedFiles 集合中！
