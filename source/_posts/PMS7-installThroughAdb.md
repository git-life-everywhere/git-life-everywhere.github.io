# PMS 第 7 篇 - 通过 adb 指令分析 Install 过程   
title: PMS 第 7 篇 - 通过 pm 指令分析 Install 过程
date: 2018/05/03
categories:
- AndroidFramework源码分析
- PackageManager包管理
tags: PackageManager包管理
---

[toc]

基于 Android 7.1.1 源码分析 PackageManagerService 的架构和逻辑实现！

# 0 综述

本篇文章总结下 install package 的过程，在 Abdroid 7.1.1 上安装一个应用有如下的方式：

- adb install（根据情况转为 cmd package install 或者 pm install）
- adb shell cmd package install（目前都是优先使用）
- adb shell pm install;
- 拷贝 apk 到文件管理器中触发安装；
- 通过应用商店安装；

这里我们先从 adb install 入手，对于其他的安装方式，我们后面再分析！

# 1 adb install - commandline::adb_commandline

adb install 的执行从 system/core/adb/commandline.cpp 开始：

adb_commandline 中会涉及到大量的 adb 指令的处理，这里我们只关注 adb install 的处理：

```c++
int adb_commandline(int argc, const char **argv) {
    ... ... ...
    
    else if (!strcmp(argv[0], "push")) { // adb push 命令！
        bool copy_attrs = false;
        std::vector<const char*> srcs;
        const char* dst = nullptr;

        parse_push_pull_args(&argv[1], argc - 1, &srcs, &dst, &copy_attrs);
        if (srcs.empty() || !dst) return usage();
        return do_sync_push(srcs, dst) ? 0 : 1;
    }
    
    ... ... ...
    
    else if (!strcmp(argv[0], "install")) { //【1.2】adb install 命令！
        if (argc < 2) return usage();
        if (_use_legacy_install()) {
            //【1.2】正常情况下调用 install_app_legacy！
            return install_app_legacy(transport_type, serial, argc, argv);
        }
        // 如果支持 FeatureCmd，那就调用 install_app
        return install_app(transport_type, serial, argc, argv);
    }
    else if (!strcmp(argv[0], "install-multiple")) { // adb install-multiple 命令
        if (argc < 2) return usage();
        return install_multiple_app(transport_type, serial, argc, argv);
    }
    else if (!strcmp(argv[0], "uninstall")) { // adb uninstall 命令
        if (argc < 2) return usage();
        if (_use_legacy_install()) {
            return uninstall_app_legacy(transport_type, serial, argc, argv);
        }
        return uninstall_app(transport_type, serial, argc, argv);
    }

    ... ... ...

    usage();
    return 1;
}
```

这里看到：

- 如果支持 cmd 命令的情况下，会调用 install_app；
- 如果不支持，执行 install_app_legacy 方法！

## 1.1 cmd install - 支持 cmd 指令

### 1.1.1 commandline::install_app

```java
static int install_app(TransportType transport, const char* serial, int argc, const char** argv) {
    // The last argument must be the APK file
    const char* file = argv[argc - 1];
    const char* dot = strrchr(file, '.');
    bool found_apk = false;
    struct stat sb;
    if (dot && !strcasecmp(dot, ".apk")) {
        if (stat(file, &sb) == -1 || !S_ISREG(sb.st_mode)) {
            fprintf(stderr, "Invalid APK file: %s\n", file);
            return EXIT_FAILURE;
        }
        found_apk = true;
    }

    if (!found_apk) {
        fprintf(stderr, "Missing APK file\n");
        return EXIT_FAILURE;
    }

    int localFd = adb_open(file, O_RDONLY);
    if (localFd < 0) {
        fprintf(stderr, "Failed to open %s: %s\n", file, strerror(errno));
        return 1;
    }

    std::string error;

    //【1】使用 cmd package 命令！
    std::string cmd = "exec:cmd package";

    // don't copy the APK name, but, copy the rest of the arguments as-is
    while (argc-- > 1) {
        cmd += " " + escape_arg(std::string(*argv++));
    }

    // add size parameter [required for streaming installs]
    // do last to override any user specified value
    cmd += " " + android::base::StringPrintf("-S %" PRIu64, static_cast<uint64_t>(sb.st_size));

    int remoteFd = adb_connect(cmd, &error);
    if (remoteFd < 0) {
        fprintf(stderr, "Connect error for write: %s\n", error.c_str());
        adb_close(localFd);
        return 1;
    }

    char buf[BUFSIZ];
    copy_to_file(localFd, remoteFd);
    read_status_line(remoteFd, buf, sizeof(buf));

    adb_close(localFd);
    adb_close(remoteFd);

    if (strncmp("Success", buf, 7)) {
        fprintf(stderr, "Failed to install %s: %s", file, buf);
        return 1;
    }
    fputs(buf, stderr);
    return 0;
}
```

### 1.1.5 new PackageManagerShellCommand

#### 1.1.5.1 onCommand

最终，会调用 PackageManagerShellCommand 的 onCommand 方法：

```java
class PackageManagerShellCommand extends ShellCommand {
    ... ... ... ...
    @Override
    public int onCommand(String cmd) {
        if (cmd == null) {
            return handleDefaultCommands(cmd);
        }

        final PrintWriter pw = getOutPrintWriter();
        try {
            switch(cmd) {
                case "install": 
                    //【*1.1.5.2】对于 install 调用，调用 runInstall 方法：
                    return runInstall();
                case "install-abandon":
                case "install-destroy":
                    return runInstallAbandon();
                case "install-commit":
                    return runInstallCommit();
                case "install-create":
                    return runInstallCreate();
                case "install-remove":
                    return runInstallRemove();
                case "install-write":
                    return runInstallWrite();
                ... ... ...
                default:
                    return handleDefaultCommands(cmd);
            }
        } catch (RemoteException e) {
            pw.println("Remote exception: " + e);
        }
        return -1;
    }
}
```
这里我们先看 runInstall，其他指令相对于 runInstall 简单了不少！


#### 1.1.5.2 runInstall
```java
    private int runInstall() throws RemoteException {
        final PrintWriter pw = getOutPrintWriter();
        //【*1.1.5.2.1】这个和 pm install 的流程是一样的，虽然这个同名方法是在 PackageManagerShellCommand 中，但实际上
        // 两个方法的代码一模一样，大家可以参考【*2.9.1】
        final InstallParams params = makeInstallParams();
        final String inPath = getNextArg(); // 获得安装包的 path！
        boolean installExternal =
                (params.sessionParams.installFlags & PackageManager.INSTALL_EXTERNAL) != 0;
        if (params.sessionParams.sizeBytes < 0 && inPath != null) {
            File file = new File(inPath);
            if (file.isFile()) {
                if (installExternal) {
                    try {
                        //【1】解析 apk，用于计算 apk 大小！！
                        ApkLite baseApk = PackageParser.parseApkLite(file, 0);
                        PackageLite pkgLite = new PackageLite(null, baseApk, null, null, null);
                        //【*4.2.3.4.1】调用 PackageHelper 计算空间大小；
                        params.sessionParams.setSize(
                                PackageHelper.calculateInstalledSize(pkgLite, false,
                                        params.sessionParams.abiOverride));
                    } catch (PackageParserException | IOException e) {
                        pw.println("Error: Failed to parse APK file : " + e);
                        return 1;
                    }
                } else {
                    params.sessionParams.setSize(file.length());
                }
            }
        }
        //【*1.1.5.2.2】创建安装事务，返回事务 id！
        final int sessionId = doCreateSession(params.sessionParams,
                params.installerPackageName, params.userId);
        boolean abandonSession = true;
        try {
            if (inPath == null && params.sessionParams.sizeBytes == 0) {
                pw.println("Error: must either specify a package size or an APK file");
                return 1;
            }
            //【*1.1.5.2.3】进一步处理事务，我们可以看到，这种方法只能装 base apk！！
            if (doWriteSplit(sessionId, inPath, params.sessionParams.sizeBytes, "base.apk",
                    false /*logSuccess*/) != PackageInstaller.STATUS_SUCCESS) {
                return 1;
            }
            //【*1.1.5.2.4】提交事务！！
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


##### 1.1.5.2.1 makeInstallParams

这个和 pm install 的流程是一样的，虽然这个同名方法是在 PackageManagerShellCommand 中，但实际上两个方法的代码一模一样，大家可以参考【*2.9.1】!

```java
    private InstallParams makeInstallParams() {
        //【1】这个 InstallParams 也是内部类，但是和 pm install 中的代码是一样的！
        final SessionParams sessionParams = new SessionParams(SessionParams.MODE_FULL_INSTALL);
        final InstallParams params = new InstallParams();
        params.sessionParams = sessionParams;
        String opt;
        while ((opt = getNextOption()) != null) {
            switch (opt) {
                case "-l":
                    sessionParams.installFlags |= PackageManager.INSTALL_FORWARD_LOCK;
                    break;
                case "-r":
                    sessionParams.installFlags |= PackageManager.INSTALL_REPLACE_EXISTING;
                    break;
                ... ... ...
                default:
                    throw new IllegalArgumentException("Unknown option " + opt);
            }
        }
        return params;
    }
```
不多说了！

##### 1.1.5.2.2 doCreateSession

这个和 pm install 的流程是一样的，虽然这个同名方法是在 PackageManagerShellCommand 中，但实际上两个方法的代码几乎一模一样，大家可以参考【*2.9.2】!

```java
    private int doCreateSession(SessionParams params, String installerPackageName, int userId)
            throws RemoteException {
        userId = translateUserId(userId, "runInstallCreate");
        if (userId == UserHandle.USER_ALL) {
            userId = UserHandle.USER_SYSTEM;
            params.installFlags |= PackageManager.INSTALL_ALL_USERS;
        }
        //【*3.1】调用 PackageInstallerService 的 createSession 方法，创建实例！
        final int sessionId = mInterface.getPackageInstaller()
                .createSession(params, installerPackageName, userId);
        return sessionId;
    }
```
这里的 mInterface 是 PMS 的代理对象，通过 getPackageInstaller 获得了 PackageInstallerService 实例！

##### 1.1.5.2.3 doWriteSplit

doWriteSplit 方法和 2.9.3 Pm.doWriteSession 是一样的：

```java
    private int doWriteSplit(int sessionId, String inPath, long sizeBytes, String splitName,
            boolean logSuccess) throws RemoteException {
        final PrintWriter pw = getOutPrintWriter();
        if (sizeBytes <= 0) {
            pw.println("Error: must specify a APK size");
            return 1;
        }
        if (inPath != null && !"-".equals(inPath)) {
            pw.println("Error: APK content must be streamed");
            return 1;
        }
        inPath = null;

        //【*3.3.2】根据前面创建的 Session id，获得本次安装事务对象！
        final SessionInfo info = mInterface.getPackageInstaller().getSessionInfo(sessionId);

        PackageInstaller.Session session = null;
        InputStream in = null;
        OutputStream out = null;
        try {
            //【*3.2.1】通过 PackageInstallerService.openSession 获得对应的 PackageInstallerSession 对象！
            //【*2.9.3.1】封装成 Session 实例！
            session = new PackageInstaller.Session(
                    mInterface.getPackageInstaller().openSession(sessionId));

            if (inPath != null) {
                in = new FileInputStream(inPath);
            } else {
                in = new SizedInputStream(getRawInputStream(), sizeBytes);
            }
            
            //【*2.9.3.1.1】定义输出流对象，指向待安装 APK 对应文件的源地址！
            out = session.openWrite(splitName, 0, sizeBytes);

            int total = 0;
            byte[] buffer = new byte[65536];
            int c;
            while ((c = in.read(buffer)) != -1) {
                total += c;
                //【1】拷贝文件到 /data/app/ 目标目录！
                out.write(buffer, 0, c);

                if (info.sizeBytes > 0) {
                    final float fraction = ((float) c / (float) info.sizeBytes);
                    //【2】同时更新进度！
                    session.addProgress(fraction);
                }
            }
            session.fsync(out);

            if (logSuccess) {
                pw.println("Success: streamed " + total + " bytes");
            }
            return 0;
        } catch (IOException e) {
            pw.println("Error: failed to write; " + e.getMessage());
            return 1;
        } finally {
            IoUtils.closeQuietly(out);
            IoUtils.closeQuietly(in);
            IoUtils.closeQuietly(session);
        }
    }
```
不多说了！

##### 1.1.5.2.4 doCommitSession

doWriteSplit 方法和 2.9.4 Pm.doCommitSession 是一样的：

```java
    private int doCommitSession(int sessionId, boolean logSuccess) throws RemoteException {
        final PrintWriter pw = getOutPrintWriter();
        PackageInstaller.Session session = null;
        try {
            //【×2.9.3.1】和前面一样，重新获得一个 Session 对象！
            session = new PackageInstaller.Session(
                    mInterface.getPackageInstaller().openSession(sessionId));
            //【×2.9.4.1】创建了一个 LocalIntentReceiver！
            final LocalIntentReceiver receiver = new LocalIntentReceiver();
            //【×2.9.4.2】将 receiver.getIntentSender() 传递给 pms，用户客户端进程获得安装结果
            //【×2.9.4.3】调用 PackageInstallerSession 的 commit 方法，提交事务！
            session.commit(receiver.getIntentSender());
            //【×2.9.4.4】处理返回结果！
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

## 1.2 pm install - 不支持 cmd 指令

### 1.2.1 commandline::install_app_legacy

```c++
static int install_app_legacy(TransportType transport, const char* serial, int argc, const char** argv) {
    // 此时我们需要先将 apk 将拷贝到手机的指定目录下！
    //【1】如果是要安装到内部存储，目标路径是 DATA_DEST，如果是存储则是 SD_DEST！
    static const char *const DATA_DEST = "/data/local/tmp/%s";
    static const char *const SD_DEST = "/sdcard/tmp/%s";
    const char* where = DATA_DEST;
    int i;
    struct stat sb;
    //【2】如果 adb 指令设置了 -s 参数，那就安装到外置存储，默认是内置存储！
    for (i = 1; i < argc; i++) {
        if (!strcmp(argv[i], "-s")) {
            where = SD_DEST;
        }
    }

    //【3】判断是否有 apk 文件参数！
    int last_apk = -1;
    for (i = argc - 1; i >= 0; i--) {
        const char* file = argv[i];
        const char* dot = strrchr(file, '.');
        if (dot && !strcasecmp(dot, ".apk")) {
            if (stat(file, &sb) == -1 || !S_ISREG(sb.st_mode)) {
                fprintf(stderr, "Invalid APK file: %s\n", file);
                return EXIT_FAILURE;
            }

            last_apk = i;
            break;
        }
    }

    if (last_apk == -1) {
        fprintf(stderr, "Missing APK file\n");
        return EXIT_FAILURE;
    }

    int result = -1;
    std::vector<const char*> apk_file = {argv[last_apk]};
    //【4】计算目标位置，将要安装的 apk 拷贝到目标缓存位置
    // 同时将 adb 命令的最后一个参数改为 apk 拷贝后的目标缓存位置！
    // 目标位置：/data/local/tmp/name.apk!
    std::string apk_dest = android::base::StringPrintf(
        where, adb_basename(argv[last_apk]).c_str());、

    // do_sync_push 将此 apk 文件传输到目标路径，失败的话将跳转到 clenaup_apk
    if (!do_sync_push(apk_file, apk_dest.c_str())) goto cleanup_apk;
    // 设置新位置！
    argv[last_apk] = apk_dest.c_str();

    //【*1.2.2】执行安装
    result = pm_command(transport, serial, argc, argv);

cleanup_apk:
    // 删除目标缓存 apk 文件，PMS 会把该 apk 拷贝到 /data/app 目录下，所以这个缓存中的 apk 没用了！
    delete_file(transport, serial, apk_dest);
    return result;
}
```

其实整个过程很简单，不多说了！

### 1.2.2 commandline::pm_command

我们继续来分析：
```java
static int pm_command(TransportType transport, const char* serial, int argc, const char** argv) {
    // 我们看到，这里会将 adb install 命令和参数，转为 pm 指令！
    std::string cmd = "pm";

    while (argc-- > 0) {
        cmd += " " + escape_arg(*argv++);
    }
    //【*1.2.3】继续调用 send_shell_command 方法！
    return send_shell_command(transport, serial, cmd, false);
}
```

### 1.2.3 commandline::send_shell_command

```c++
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
                // 如果定义了 feature，则替换 shell protocol!
                use_shell_protocol = CanUseFeature(features, kFeatureShell2);
            } else {
                attempt_connection = false;
            }
        }

        if (attempt_connection) {
            std::string error;
            // 此时 command 中携带的就是以 pm 开头的命令
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
到这里，我们知道，最后其实是向 shell 服务发送 pm 命令，触发 apk 的安装！

# 2 pm install

cmd 的调用上面已经分析了，其实和 pm install 没有区别！

这里我们来看下 pm 命令的主要用法，通过 pm install 继续分析安装：

```java
usage: pm path [--user USER_ID] PACKAGE
       pm dump PACKAGE
       pm install [-lrtsfd] [-i PACKAGE] [--user USER_ID] [PATH]
       pm install-create [-lrtsfdp] [-i PACKAGE] [-S BYTES]
               [--install-location 0/1/2]
               [--force-uuid internal|UUID]
       pm install-write [-S BYTES] SESSION_ID SPLIT_NAME [PATH]
       pm install-commit SESSION_ID
       pm install-abandon SESSION_ID
       pm uninstall [-k] [--user USER_ID] [--versionCode VERSION_CODE] PACKAGE
       pm set-installer PACKAGE INSTALLER
       pm move-package PACKAGE [internal|UUID]
       pm move-primary-storage [internal|UUID]
       pm clear [--user USER_ID] PACKAGE
       pm enable [--user USER_ID] PACKAGE_OR_COMPONENT
       pm disable [--user USER_ID] PACKAGE_OR_COMPONENT
       pm disable-user [--user USER_ID] PACKAGE_OR_COMPONENT
       pm disable-until-used [--user USER_ID] PACKAGE_OR_COMPONENT
       pm default-state [--user USER_ID] PACKAGE_OR_COMPONENT
       pm set-user-restriction [--user USER_ID] RESTRICTION VALUE
       pm hide [--user USER_ID] PACKAGE_OR_COMPONENT
       pm unhide [--user USER_ID] PACKAGE_OR_COMPONENT
       pm grant [--user USER_ID] PACKAGE PERMISSION
       pm revoke [--user USER_ID] PACKAGE PERMISSION
       pm reset-permissions
       pm set-app-link [--user USER_ID] PACKAGE {always|ask|never|undefined}
       pm get-app-link [--user USER_ID] PACKAGE
       pm set-install-location [0/auto] [1/internal] [2/external]
       pm get-install-location
       pm set-permission-enforced PERMISSION [true|false]
       pm trim-caches DESIRED_FREE_SPACE [internal|UUID]
       pm create-user [--profileOf USER_ID] [--managed] [--restricted] [--ephemeral] [--guest] USER_NAME
       pm remove-user USER_ID
       pm get-max-users
```
pm 命令的源码位于 frameworks/base/cmds/pm 中，我们直接去看下目录结构：

```java
.
├── Android.mk
├── MODULE_LICENSE_APACHE2
├── NOTICE
├── pm
└── src
    └── com
        └── android
            └── commands
                └── pm
                    └── Pm.java
```
我们先来看看 Android.mk 中的内容：

```java
# Copyright 2007 The Android Open Source Project
#
LOCAL_PATH:= $(call my-dir)
include $(CLEAR_VARS)

LOCAL_SRC_FILES := $(call all-subdir-java-files)
LOCAL_MODULE := pm
include $(BUILD_JAVA_LIBRARY)


include $(CLEAR_VARS)
ALL_PREBUILT += $(TARGET_OUT)/bin/pm
$(TARGET_OUT)/bin/pm : $(LOCAL_PATH)/pm | $(ACP)
	$(transform-prebuilt-to-target)
```
可以看到，编译期间，编译系统会将 pm 脚本放置到 /system/bin/ 目录下。

接着，来看看 pm 脚本的内容：
```xml
# Script to start "pm" on the device, which has a very rudimentary
# shell.
#
base=/system
export CLASSPATH=$base/framework/pm.jar
exec app_process $base/bin com.android.commands.pm.Pm "$@"
```
该脚本的作用是通过 app_process 启动 pm java 进程！


## 2.1 app_process::main

app_process 其实就是 Zygote 的源码，在开机篇我们已经分析过了 Zygote 的启动，这里略过和 Zygote 相关的代码：

```c++
int main(int argc, char* const argv[]) {
    ... ... ...

    ++i;  // Skip unused "parent dir" argument.
    while (i < argc) {
        const char* arg = argv[i++];
        if (strcmp(arg, "--zygote") == 0) {
            zygote = true;
            niceName = ZYGOTE_NICE_NAME;
        } else if (strcmp(arg, "--start-system-server") == 0) {
            startSystemServer = true;
        } else if (strcmp(arg, "--application") == 0) {
            application = true;
        } else if (strncmp(arg, "--nice-name=", 12) == 0) {
            niceName.setTo(arg + 12);
        } else if (strncmp(arg, "--", 2) != 0) {
            //【1】获得要启动的 java 进程的 className
            className.setTo(arg);
            break;
        } else {
            --i;
            break;
        }
    }

    Vector<String8> args;
    if (!className.isEmpty()) {
        //【2】根据 application 的取值，判断我们启动的是应用类型进程，还是工具类型进程！
        args.add(application ? String8("application") : String8("tool"));
        runtime.setClassNameAndArgs(className, argc - i, argv + i);
    } else {
        ... ... ...// zygote 启动才会进入！
    }

    if (!niceName.isEmpty()) {
        runtime.setArgv0(niceName.string());
        set_process_name(niceName.string());
    }

    if (zygote) {
        runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
    } else if (className) {
        //【2.2】非 Zygote 进程的启动方式如下
        runtime.start("com.android.internal.os.RuntimeInit", args, zygote);
    } else {
        fprintf(stderr, "Error: no class name or --zygote supplied.\n");
        app_usage();
        LOG_ALWAYS_FATAL("app_process: no class name or --zygote supplied.");
        return 10;
    }
}
```
接下来，会进入 AndroidRuntime 中去！！

## 2.2 AndroidRuntime::start

这里的 className 传入的值为 com.android.internal.os.RuntimeInit！

```java
void AndroidRuntime::start(const char* className, const Vector<String8>& options, bool zygote)
{
    ALOGD(">>>>>> START %s uid %d <<<<<<\n",
            className != NULL ? className : "(unknown)", getuid());

    static const String8 startSystemServer("start-system-server");

    for (size_t i = 0; i < options.size(); ++i) {
        if (options[i] == startSystemServer) {
           /* track our progress through the boot sequence */
           const int LOG_BOOT_PROGRESS_START = 3000;
           LOG_EVENT_LONG(LOG_BOOT_PROGRESS_START,  ns2ms(systemTime(SYSTEM_TIME_MONOTONIC)));
        }
    }

    const char* rootDir = getenv("ANDROID_ROOT");
    if (rootDir == NULL) {
        rootDir = "/system";
        if (!hasDir("/system")) {
            LOG_FATAL("No root directory specified, and /android does not exist.");
            return;
        }
        setenv("ANDROID_ROOT", rootDir, 1);
    }

    //const char* kernelHack = getenv("LD_ASSUME_KERNEL");
    //ALOGD("Found LD_ASSUME_KERNEL='%s'\n", kernelHack);

    //【1】启动虚拟机！
    JniInvocation jni_invocation;
    jni_invocation.Init(NULL);
    JNIEnv* env;
    if (startVm(&mJavaVM, &env, zygote) != 0) {
        return;
    }
    onVmCreated(env);

    //【2】注册 Android 函数！
    if (startReg(env) < 0) {
        ALOGE("Unable to register all android natives\n");
        return;
    }

    //【3】创建一个 String 数组来保存传递进来的参数！
    jclass stringClass;
    jobjectArray strArray;
    jstring classNameStr;

    stringClass = env->FindClass("java/lang/String");
    assert(stringClass != NULL);
    strArray = env->NewObjectArray(options.size() + 1, stringClass, NULL);
    assert(strArray != NULL);
    classNameStr = env->NewStringUTF(className);
    assert(classNameStr != NULL);
    env->SetObjectArrayElement(strArray, 0, classNameStr);

    for (size_t i = 0; i < options.size(); ++i) {
        jstring optionsStr = env->NewStringUTF(options.itemAt(i).string());
        assert(optionsStr != NULL);
        env->SetObjectArrayElement(strArray, i + 1, optionsStr);
    }

    //【4】启动虚拟机的主线程，该主线程不会退出，直到虚拟机退出！
    char* slashClassName = toSlashClassName(className);
    jclass startClass = env->FindClass(slashClassName);
    if (startClass == NULL) {
        ALOGE("JavaVM unable to locate class '%s'\n", slashClassName);
        /* keep going */
    } else {
        //【2.3】反射调用 RuntimeInit.main 函数，从 native 层进入 java 世界！
        jmethodID startMeth = env->GetStaticMethodID(startClass, "main",
            "([Ljava/lang/String;)V");
        if (startMeth == NULL) {
            ALOGE("JavaVM unable to find main() in '%s'\n", className);
            /* keep going */
        } else {
            env->CallStaticVoidMethod(startClass, startMeth, strArray);
#if 0
            if (env->ExceptionCheck())
                threadExitUncaughtException(env);
#endif
        }
    }
    free(slashClassName);

    ALOGD("Shutting down VM\n");
    if (mJavaVM->DetachCurrentThread() != JNI_OK)
        ALOGW("Warning: unable to detach main thread\n");
    if (mJavaVM->DestroyJavaVM() != 0)
        ALOGW("Warning: VM did not shut down cleanly\n");
}
```
通过反射，进入 RuntimeInit 中！

## 2.3 RuntimeInit.main

```java
    public static final void main(String[] argv) {
        enableDdms();
        if (argv.length == 2 && argv[1].equals("application")) {
            if (DEBUG) Slog.d(TAG, "RuntimeInit: Starting application");
            redirectLogStreams();
        } else {
            if (DEBUG) Slog.d(TAG, "RuntimeInit: Starting tool");
        }
        //【1】进程一些初始化的操作，比如设置线程的默认异常处理 Handler，设置 Log 系统等等，在进程的启动时，我们分析过！！
        commonInit();

        //【2.4】这里的关键点在这里！
        nativeFinishInit();

        if (DEBUG) Slog.d(TAG, "Leaving RuntimeInit!");
    }
```
nativeFinishInit 调用的是 native 方法，位于 AndroidRuntime.cpp 文件中！

## 2.4 AR::com_android_internal_os_RuntimeInit_nativeFinishInit

```java
static void com_android_internal_os_RuntimeInit_nativeFinishInit(JNIEnv* env, jobject clazz)
{
    //【2.5】gCurRuntime 是 AndroidRuntime 子类 AppRuntime 的实例！
    gCurRuntime->onStarted();
}
```
实际上 AndroidRuntime 并没有实现 onStarted 方法，真正是实现是在 App_main.cpp 中的 AppRuntime 类中，他是 AndroidRuntime 的子类！


## 2.5 AppRuntime::onStarted

```java
    virtual void onStarted()
    {
        sp<ProcessState> proc = ProcessState::self();
        ALOGV("App process: starting thread pool.\n");
        proc->startThreadPool();
        //【2.6】执行 AndroidRuntime 的 callMain 方法！
        AndroidRuntime* ar = AndroidRuntime::getRuntime();
        ar->callMain(mClassName, mClass, mArgs);

        IPCThreadState::self()->stopProcess();
    }
```

## 2.6 AndroidRuntime::callMain

```java
status_t AndroidRuntime::callMain(const String8& className, jclass clazz,
    const Vector<String8>& args)
{
    JNIEnv* env;
    jmethodID methodId;

    ALOGD("Calling main entry %s", className.string());

    env = getJNIEnv();
    if (clazz == NULL || env == NULL) {
        return UNKNOWN_ERROR;
    }

    methodId = env->GetStaticMethodID(clazz, "main", "([Ljava/lang/String;)V");
    if (methodId == NULL) {
        ALOGE("ERROR: could not find method %s.main(String[])\n", className.string());
        return UNKNOWN_ERROR;
    }
    //【1】创建一个 String 数组来封装参数！
    jclass stringClass;
    jobjectArray strArray;

    const size_t numArgs = args.size();
    stringClass = env->FindClass("java/lang/String");
    strArray = env->NewObjectArray(numArgs, stringClass, NULL);

    for (size_t i = 0; i < numArgs; i++) {
        jstring argStr = env->NewStringUTF(args[i].string());
        env->SetObjectArrayElement(strArray, i, argStr);
    }
    //【*2.7】最终反射调用了 Pm.main 方法！
    env->CallStaticVoidMethod(clazz, methodId, strArray);
    return NO_ERROR;
}
```

## 2.7 Pm.main

```java
    public static void main(String[] args) {
        int exitCode = 1;
        try {
            //【*2.8】执行 Pm.run f方法！
            exitCode = new Pm().run(args);
        } catch (Exception e) {
            Log.e(TAG, "Error", e);
            System.err.println("Error: " + e);
            if (e instanceof RemoteException) {
                System.err.println(PM_NOT_RUNNING_ERR);
            }
        }
        System.exit(exitCode);
    }
```

## 2.8 Pm.run

run 方法中涉及到的命令有很多，这里我们重点关注和 install 相关的！

```java
    public int run(String[] args) throws RemoteException {
        boolean validCommand = false;
        if (args.length < 1) {
            return showUsage();
        }
        mAm = IAccountManager.Stub.asInterface(ServiceManager.getService(Context.ACCOUNT_SERVICE));
        mUm = IUserManager.Stub.asInterface(ServiceManager.getService(Context.USER_SERVICE));
        //【1】获得 PacakgeManager 对象！
        mPm = IPackageManager.Stub.asInterface(ServiceManager.getService("package"));

        if (mPm == null) {
            System.err.println(PM_NOT_RUNNING_ERR);
            return 1;
        }
        //【2】获得 PackageInstallerService 代理对象！
        mInstaller = mPm.getPackageInstaller();

        mArgs = args;
        String op = args[0];
        mNextArg = 1;

        ... ... ...

        //【*2.9】runInstall 方法，执行执行安装！
        if ("install".equals(op)) {
            return runInstall();
        }

        ... ... ... ...

        try {
            if (args.length == 1) {
                if (args[0].equalsIgnoreCase("-l")) {
                    validCommand = true;
                    return runShellCommand("package", new String[] { "list", "package" });
                } else if (args[0].equalsIgnoreCase("-lf")) {
                    validCommand = true;
                    return runShellCommand("package", new String[] { "list", "package", "-f" });
                }
            } else if (args.length == 2) {
                if (args[0].equalsIgnoreCase("-p")) {
                    validCommand = true;
                    return displayPackageFilePath(args[1], UserHandle.USER_SYSTEM);
                }
            }
            return 1;
        } finally {
            if (validCommand == false) {
                if (op != null) {
                    System.err.println("Error: unknown command '" + op + "'");
                }
                showUsage();
            }
        }
    }
```
继续看！

## 2.9 Pm.runInstall

我们继续分析：

```java
    private int runInstall() throws RemoteException {
        //【*2.9.1】创建安装参数
        final InstallParams params = makeInstallParams();
        final String inPath = nextArg(); // 获得安装 apk 的路径！
        boolean installExternal =
                (params.sessionParams.installFlags & PackageManager.INSTALL_EXTERNAL) != 0;
        if (params.sessionParams.sizeBytes < 0 && inPath != null) {
            File file = new File(inPath);
            if (file.isFile()) {
                if (installExternal) {
                    try {
                        //【1】解析 apk，用于计算 apk 大小！
                        ApkLite baseApk = PackageParser.parseApkLite(file, 0);
                        PackageLite pkgLite = new PackageLite(null, baseApk, null, null, null);
                        params.sessionParams.setSize(
                                PackageHelper.calculateInstalledSize(pkgLite, false,
                                        params.sessionParams.abiOverride));
                    } catch (PackageParserException | IOException e) {
                        System.err.println("Error: Failed to parse APK file : " + e);
                        return 1;
                    }
                } else {
                    params.sessionParams.setSize(file.length());
                }
            }
        }
        //【*2.9.2】创建安装 Session！
        final int sessionId = doCreateSession(params.sessionParams,
                params.installerPackageName, params.userId);

        try {
            if (inPath == null && params.sessionParams.sizeBytes == 0) {
                System.err.println("Error: must either specify a package size or an APK file");
                return 1;
            }
            //【*2.9.3】该过程其实是将 inPath 指向的 apk 写入到 .tmp 目录下，指定了文件名是 "base.apk"
            if (doWriteSession(sessionId, inPath, params.sessionParams.sizeBytes, "base.apk",
                    false /*logSuccess*/) != PackageInstaller.STATUS_SUCCESS) {
                return 1;
            }
            //【*2.9.4】提交 Session 触发安装！
            if (doCommitSession(sessionId, false /*logSuccess*/)
                    != PackageInstaller.STATUS_SUCCESS) {
                return 1;
            }
            System.out.println("Success");
            return 0;
        } finally {
            try {
                //【*2.9.5】如果安装异常，忽视本次事务！
                mInstaller.abandonSession(sessionId);
            } catch (Exception ignore) {
            }
        }
    }

```

### 2.9.1 Pm.makeInstallParams - 创建事务参数

Pm 中会根据传入的安装参数，创建 InstallParams，封装安装参数！

```java
    private InstallParams makeInstallParams() {
        //【*2.9.1.1】创建 SessionParams 实例，封装安装参数！
        final SessionParams sessionParams = new SessionParams(SessionParams.MODE_FULL_INSTALL);
        //【*2.9.1.2】创建 InstallParams，解耦合！
        final InstallParams params = new InstallParams();
        params.sessionParams = sessionParams; // 设置引用关系！
        String opt;
        
        //【1】根据额外的参数，设置安装参数！
        while ((opt = nextOption()) != null) {
            switch (opt) {
                case "-l": 
                    //【1.1】该应用以 forward locked 方式安装，在该模式下，只有应用自己能够访问自己的代码
                    // 和非资源文件！ 
                    sessionParams.installFlags |= PackageManager.INSTALL_FORWARD_LOCK;
                    break;
                case "-r": //【1.2】替换已存在的 apk
                    sessionParams.installFlags |= PackageManager.INSTALL_REPLACE_EXISTING;
                    break;
                case "-i": // 显式指定安装器
                    params.installerPackageName = nextArg();
                    if (params.installerPackageName == null) {
                        throw new IllegalArgumentException("Missing installer package");
                    }
                    break;
                case "-t": // 允许测试应用安装（设置了 android:testOnly）
                    sessionParams.installFlags |= PackageManager.INSTALL_ALLOW_TEST;
                    break;
                case "-s": // 安装到外置存储
                    sessionParams.installFlags |= PackageManager.INSTALL_EXTERNAL;
                    break;
                case "-f": // 安装到内置存储
                    sessionParams.installFlags |= PackageManager.INSTALL_INTERNAL;
                    break;
                case "-d": //【1.3】允许降级安装！
                    sessionParams.installFlags |= PackageManager.INSTALL_ALLOW_DOWNGRADE;
                    break;
                case "-g": //【1.4】授予运行时权限
                    sessionParams.installFlags |= PackageManager.INSTALL_GRANT_RUNTIME_PERMISSIONS;
                    break;
                case "--dont-kill": // 安装不会杀 app 进程
                    sessionParams.installFlags |= PackageManager.INSTALL_DONT_KILL_APP;
                    break;
                case "--originating-uri":
                    sessionParams.originatingUri = Uri.parse(nextOptionData());
                    break;
                case "--referrer":
                    sessionParams.referrerUri = Uri.parse(nextOptionData());
                    break;
                case "-p": //【1.5】继承已存在的 apk，appPackageName 用以保存要继承的包名，也就是主 apk 的报名！
                    sessionParams.mode = SessionParams.MODE_INHERIT_EXISTING;
                    sessionParams.appPackageName = nextOptionData();
                    if (sessionParams.appPackageName == null) {
                        throw new IllegalArgumentException("Missing inherit package name");
                    }
                    break;
                case "-S": // 显式指定应用的大小
                    sessionParams.setSize(Long.parseLong(nextOptionData()));
                    break;
                case "--abi": // 显式指定 abi
                    sessionParams.abiOverride = checkAbiArgument(nextOptionData());
                    break;
                case "--ephemeral": // 作为一个 lightweight "ephemeral" 应用！
                    sessionParams.installFlags |= PackageManager.INSTALL_EPHEMERAL;
                    break;
                case "--user": // 指定安装的 userId
                    params.userId = UserHandle.parseUserArg(nextOptionData());
                    break;
                case "--install-location": // 指定安装位置
                    sessionParams.installLocation = Integer.parseInt(nextOptionData());
                    break;
                case "--force-uuid":
                    sessionParams.installFlags |= PackageManager.INSTALL_FORCE_VOLUME_UUID;
                    sessionParams.volumeUuid = nextOptionData();
                    if ("internal".equals(sessionParams.volumeUuid)) {
                        sessionParams.volumeUuid = null;
                    }
                    break;
                case "--force-sdk":
                    sessionParams.installFlags |= PackageManager.INSTALL_FORCE_SDK;
                    break;
                default:
                    throw new IllegalArgumentException("Unknown option " + opt);
            }
        }
        return params;
    }
```
额外的参数很多，根据额外的参数，对 SessionParams 进行了属性设置！

sessionParams.installFlags 会根据不同的额外参数，设置不同的二进制位，具体的含义我上面也简单的注释了下。大家可以自己去看源码！

这里我们要注意下：

- 如果我们安装的是 split apk 的话，那么我们可以指定 -p 参数，后面指定主 apk 的包名，后面我们会分析其作用！

#### 2.9.1.1 new SessionParams

SessionParams 定义在 PackageInstaller.java 文件中：

```java
    public static class SessionParams implements Parcelable {
        /** {@hide} */
        public int mode = MODE_INVALID;
        /** {@hide} */
        public int installFlags;
        /** {@hide} */
        public int installLocation = PackageInfo.INSTALL_LOCATION_INTERNAL_ONLY; // 默认为仅内置
        /** {@hide} */
        public long sizeBytes = -1;
        /** {@hide} */
        public String appPackageName; // 要继承的主 apk 的包名；
        /** {@hide} */
        public Bitmap appIcon;
        /** {@hide} */
        public String appLabel;
        /** {@hide} */
        public long appIconLastModified = -1;
        /** {@hide} */
        public Uri originatingUri;
        /** {@hide} */
        public int originatingUid = UID_UNKNOWN;
        /** {@hide} */
        public Uri referrerUri;
        /** {@hide} */
        public String abiOverride;
        /** {@hide} */
        public String volumeUuid;
        /** {@hide} */
        public String[] grantedRuntimePermissions;

        public SessionParams(int mode) {
            this.mode = mode;
        }

        /** {@hide} */
        public SessionParams(Parcel source) {
            mode = source.readInt();
            installFlags = source.readInt();
            installLocation = source.readInt();
            sizeBytes = source.readLong();
            appPackageName = source.readString();
            appIcon = source.readParcelable(null);
            appLabel = source.readString();
            originatingUri = source.readParcelable(null);
            originatingUid = source.readInt();
            referrerUri = source.readParcelable(null);
            abiOverride = source.readString();
            volumeUuid = source.readString();
            grantedRuntimePermissions = source.readStringArray();
        }

        ... ... ... ...
    }
```
这里我们来说下 mode 属性，它可以取三个值：

```java
        public static final int MODE_INVALID = -1;

        // 表示：完全替换已存在的 apk
        public static final int MODE_FULL_INSTALL = 1;

        // 表示：会继承已存在的 apk，可以用于给 apk 添加 split APKS，如果没有已存在的 apk，
        // 效果和 MODE_FULL_INSTALL 是一样的！
        public static final int MODE_INHERIT_EXISTING = 2;
```


我们省略掉暂时用不到的一些接口，后面有机会再分析！

#### 2.9.1.2 new Pm.InstallParams

InstallParams 类很简单，内部有一个 SessionParams 实例变量，封装安装参数！
```java
    private static class InstallParams {
        SessionParams sessionParams;
        String installerPackageName; // 安装器的名字！
        int userId = UserHandle.USER_ALL;
    }
```

### 2.9.2 Pm.doCreateSession - 创建事务

创建 install Session 对象！

```java
    private int doCreateSession(SessionParams params, String installerPackageName, int userId)
            throws RemoteException {
        userId = translateUserId(userId, "runInstallCreate");
        if (userId == UserHandle.USER_ALL) {
            userId = UserHandle.USER_SYSTEM;
            params.installFlags |= PackageManager.INSTALL_ALL_USERS;
        }
        //【*3.1】进入 PackageInstallerService！
        final int sessionId = mInstaller.createSession(params, installerPackageName, userId);
        return sessionId;
    }
```
进入了 PackageInstallerService 服务，创建 Session 对象！

### 2.9.3 Pm.doWriteSession - 写入事务（实际上是拷贝文件）

回顾下参数：

```java
    //【*2.9.3】将安装 Session 持久化到本地文件！
    if (doWriteSession(sessionId, inPath, params.sessionParams.sizeBytes, "base.apk",...))
```

这里的 splitName 值为 "base.apk"；

```java
    private int doWriteSession(int sessionId, String inPath, long sizeBytes, String splitName,
            boolean logSuccess) throws RemoteException {
        //【1】这里的 inPath 就是前面创建的 /data/local/tmp/ 目录！
        if ("-".equals(inPath)) {
            inPath = null;
        } else if (inPath != null) {
            final File file = new File(inPath);
            if (file.isFile()) {
                sizeBytes = file.length();
            }
        }
        //【*3.3.2】根据前面创建的 Session id，获得本次安装事务对象！
        final SessionInfo info = mInstaller.getSessionInfo(sessionId);

        PackageInstaller.Session session = null;
        InputStream in = null;
        OutputStream out = null;
        try {
            //【*3.2.1】通过 PackageInstallerService.openSession 获得对应的 PackageInstallerSession 对象！
            //【*2.9.3.1】封装成 Session 实例！
            session = new PackageInstaller.Session(
                    mInstaller.openSession(sessionId));

            //【3】 定义输入流对象，指向待安装 APK 对应文件的源地址！
            if (inPath != null) {
                in = new FileInputStream(inPath);
            } else {
                in = new SizedInputStream(System.in, sizeBytes);
            }

            //【*2.9.3.1.1】 定义输出流对象，指向待安装 APK 对应文件的源地址！
            out = session.openWrite(splitName, 0, sizeBytes);

            int total = 0;
            byte[] buffer = new byte[65536];
            int c;
            while ((c = in.read(buffer)) != -1) {
                total += c;
                //【5】拷贝文件到 /data/app/vmdl[sessionId].tmp 目标目录！
                out.write(buffer, 0, c);

                if (info.sizeBytes > 0) {
                    final float fraction = ((float) c / (float) info.sizeBytes);
                    //【6】同时更新拷贝进度
                    session.addProgress(fraction);
                }
            }
            session.fsync(out);

            if (logSuccess) {
                System.out.println("Success: streamed " + total + " bytes");
            }
            //【3】拷贝完成，返回 STATUS_SUCCESS！
            return PackageInstaller.STATUS_SUCCESS;
        } catch (IOException e) {
            System.err.println("Error: failed to write; " + e.getMessage());
            return PackageInstaller.STATUS_FAILURE;
        } finally {
            IoUtils.closeQuietly(out);
            IoUtils.closeQuietly(in);
            IoUtils.closeQuietly(session);
        }
    }
```
doWriteSession 整个流程其实就是进行 apk 的拷贝操作！ 


#### 2.9.3.1 new PackageInstaller.Session

Session 是对 PackageInstallerSession 的简单封装！

```java
    public static class Session implements Closeable {
        private IPackageInstallerSession mSession;

        /** {@hide} */
        public Session(IPackageInstallerSession session) {
            mSession = session;
        }
```
mSession 指向了对应的 PackageInstallerSession 实例！

同时其还提供了很多借口，来操作 PackageInstallerSession，这里我们用一张图先简单的看下：

```java
Session.setProgress  ->  PackageInstallerSession.setClientProgress
Session.setStagingProgress  ->  PackageInstallerSession.setClientProgress
Session.addProgress  ->  PackageInstallerSession.addClientProgress

Session.openWrite  ->  PackageInstallerSession.openWrite
Session.openRead  ->  PackageInstallerSession.openRead

Session.commit  ->  PackageInstallerSession.commit
Session.abandon  ->  PackageInstallerSession.abandon
... ... ...
```
这里我们先来看目前已经涉及到的一些重要方法：

##### 2.9.3.1.1 Session.openWrite

这里的 name 传入的是 base.apk：

```java
        public @NonNull OutputStream openWrite(@NonNull String name, long offsetBytes,
                long lengthBytes) throws IOException {
            try {
                //【*4.1】这里会调用 PackageInstallerSession 的 openWrite 方法
                // 将前面创建 Session 的文件目录 /data/app/vmdl[sessionId].tmp 封装成一个文件描述符对象！  
                final ParcelFileDescriptor clientSocket = mSession.openWrite(name,
                        offsetBytes, lengthBytes);
                //【2】获得该文件描述符输出流！
                return new FileBridge.FileBridgeOutputStream(clientSocket);
            } catch (RuntimeException e) {
                ExceptionUtils.maybeUnwrapIOException(e);
                throw e;
            } catch (RemoteException e) {
                throw e.rethrowFromSystemServer();
            }
        }
```

mSession.openWrite 方法涉及到了文件相关处理，这里就不过多关注！

##### 2.9.3.1.2 Session.addProgress

```java
    public void addProgress(float progress) {
        try {
            //【*2.9.3.1.2.1】这里会调用 PackageInstallerSession 的 addClientProgress 方法
            // 更新文件读写进度！
            mSession.addClientProgress(progress);
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
```
继续来看：

###### 2.9.3.1.2.1 PackageInstallerSession.addClientProgress

```java
    @Override
    public void addClientProgress(float progress) {
        synchronized (mLock) {
            //【*2.9.3.1.2.2】调用了 setClientProgress 方法！
            setClientProgress(mClientProgress + progress);
        }
    }
```
###### 2.9.3.1.2.2 PackageInstallerSession.setClientProgress
```java
    @Override
    public void setClientProgress(float progress) {
        synchronized (mLock) {
            //【1】用于第一次更新进度！
            final boolean forcePublish = (mClientProgress == 0);
            mClientProgress = progress;
            //【*2.9.3.1.2.3】最终，调用 computeProgressLocked 方法！
            computeProgressLocked(forcePublish);
        }
    }
```

###### 2.9.3.1.2.3 PackageInstallerSession.computeProgressLocked
```java
    private void computeProgressLocked(boolean forcePublish) {
        //【1】计算进度！
        mProgress = MathUtils.constrain(mClientProgress * 0.8f, 0f, 0.8f)
                + MathUtils.constrain(mInternalProgress * 0.2f, 0f, 0.2f);
        //【2】更新进度
        if (forcePublish || Math.abs(mProgress - mReportedProgress) >= 0.01) {
            mReportedProgress = mProgress;
            //【*3.1.4.2】同时通知事务观察者，进度的变化！
            mCallback.onSessionProgressChanged(this, mProgress);
        }
    }
```
逻辑很简单，这里就不多说了！


### 2.9.4 Pm.doCommitSession - 提交事务

```java
    private int doCommitSession(int sessionId, boolean logSuccess) throws RemoteException {
        PackageInstaller.Session session = null;
        try {
            //【*2.9.3.1】获得前面创建的 Session 对象！
            session = new PackageInstaller.Session(
                    mInstaller.openSession(sessionId));
            //【*2.9.4.1】创建了一个 LocalIntentReceiver！
            final LocalIntentReceiver receiver = new LocalIntentReceiver();

            //【*2.9.4.2】将 receiver.getIntentSender() 传递给 pms，用户客户端进程获得安装结果
            //【*2.9.4.3】调用 PackageInstallerSession 的 commit 方法，提交事务！
            session.commit(receiver.getIntentSender());

            //【*2.9.4.4】处理返回结果！
            final Intent result = receiver.getResult();
            final int status = result.getIntExtra(PackageInstaller.EXTRA_STATUS,
                    PackageInstaller.STATUS_FAILURE);

            if (status == PackageInstaller.STATUS_SUCCESS) {
                if (logSuccess) {
                    System.out.println("Success");
                }
            } else {
                System.err.println("Failure ["
                        + result.getStringExtra(PackageInstaller.EXTRA_STATUS_MESSAGE) + "]");
            }
            return status;
        } finally {
            IoUtils.closeQuietly(session);
        }
    }
```

到这里，提交过程就结束了，最终的逻辑是在 PackageInstallerService 中实现！

#### 2.9.4.1 new LocalIntentReceiver - 跨进程返回结果

LocalIntentReceiver 用来接收安装结果！

```java
    private static class LocalIntentReceiver {
        //【1】以 Intent 的方式保存安装结果！
        private final SynchronousQueue<Intent> mResult = new SynchronousQueue<>();

        private IIntentSender.Stub mLocalSender = new IIntentSender.Stub() {
            @Override
            public void send(int code, Intent intent, String resolvedType,
                    IIntentReceiver finishedReceiver, String requiredPermission, Bundle options) {
                try {
                    //【1.1】将结果保存到 mResult 中！
                    mResult.offer(intent, 5, TimeUnit.SECONDS);
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }
            }
        };
        ... ... ...
```
LocalIntentReceiver 有一个内部成员 mLocalSender，实现了 IIntentSender.Stub，用于跨进程通信！

系统进程会持有 mLocalSender 对应的代理对象，通过 send 方法，将安装的状态或者结果返回保存到 mResult 中！



#### 2.9.4.2 LocalIntentReceiver.getIntentSender

getIntentSender 将 getIntentSender 封装到 IntentSender 中，由于 IntentSender 实现了 Parcelable，所以可以跨进程传递：

```java
        public IntentSender getIntentSender() {
            return new IntentSender((IIntentSender) mLocalSender);
        }
```

#### 2.9.4.3 Session.commit

```java
        public void commit(@NonNull IntentSender statusReceiver) {
            try {
                //【*4.2】最终，调用了 PackageInstallerSession 的 commit 方法，进入系统进程！
                mSession.commit(statusReceiver);
            } catch (RemoteException e) {
                throw e.rethrowFromSystemServer();
            }
        }
```
继续来看：

#### 2.9.4.4 LocalIntentReceiver.getResult

getResult 用于获得安装结果！

```java
        public Intent getResult() {
            try {
                //【1】返回安装结果！
                return mResult.take();
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
        }
```
就先分析到这里：

#3 PackageInstallerService

下面来分析下 PackageInstallerService 中的逻辑，我们先来看看 PackageInstallerService 的创建，当然，这部分的逻辑是在开机的时候，这里我们再回顾下：

```java
    public PackageInstallerService(Context context, PackageManagerService pm) {
        mContext = context;
        mPm = pm;
        //【1】启动了一个 HandlerThread 线程！
        mInstallThread = new HandlerThread(TAG);
        mInstallThread.start();
        //【2】创建了 mInstallThread 对应的 Handler；
        mInstallHandler = new Handler(mInstallThread.getLooper());
        //【*3.1.4.1】创建了 Callbacks 实例，用于处理安装回调，传入了子线程的 Loopder！
        mCallbacks = new Callbacks(mInstallThread.getLooper());
        //【3】创建了 sessions 本地持久化文件和目录对象！
        mSessionsFile = new AtomicFile(
                new File(Environment.getDataSystemDirectory(), "install_sessions.xml"));
        mSessionsDir = new File(Environment.getDataSystemDirectory(), "install_sessions");
        mSessionsDir.mkdirs();

        synchronized (mSessions) {
            //【4】读取本次持久化文件中的事务，创建对应的 PackageInstallerSession 实例！
            readSessionsLocked();
            //【5】解决临时拷贝目录和持久化事务的冲突，过程很简单，移除那些没有持久化事务的临时拷贝文件！
            reconcileStagesLocked(StorageManager.UUID_PRIVATE_INTERNAL, false /*isEphemeral*/);
            reconcileStagesLocked(StorageManager.UUID_PRIVATE_INTERNAL, true /*isEphemeral*/);

            final ArraySet<File> unclaimedIcons = newArraySet(
                    mSessionsDir.listFiles());
            for (int i = 0; i < mSessions.size(); i++) {
                final PackageInstallerSession session = mSessions.valueAt(i);
                unclaimedIcons.remove(buildAppIconFile(session.sessionId));
            }
            for (File icon : unclaimedIcons) {
                Slog.w(TAG, "Deleting orphan icon " + icon);
                icon.delete();
            }
        }
    }
```
关于 PackageInstallerService 这里就不在过多分析，我们继续看！！

## 3.1 PackageInstallerS.createSession(Internal) - 创建事务

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
createSession 方法调用了 createSessionInternal 方法！

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
            // 如果是 shell or root，校验下 package 是否属于 uid，
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
            //【*3.1.1】判断，同一个 uid 是否有过多的正在处理的 Session，如果超过了 1024 个，那当前就不能执行安装！
            final int activeCount = getSessionCount(mSessions, callingUid);
            if (activeCount >= MAX_ACTIVE_SESSIONS) {
                throw new IllegalStateException(
                        "Too many active sessions for UID " + callingUid);
            }
            // 同样。判断同一个 uid，是否已经提交了过多的 Session，如果超过了 1048576 个，那当前就不能执行安装！
            final int historicalCount = getSessionCount(mHistoricalSessions, callingUid);
            if (historicalCount >= MAX_HISTORICAL_SESSIONS) {
                throw new IllegalStateException(
                        "Too many historical sessions for UID " + callingUid);
            }
            //【*3.1.2】给本次安装分配一个事务 id！
            sessionId = allocateSessionIdLocked();
        }

        final long createdMillis = System.currentTimeMillis();

        //【8】决定安装目录，因为默认是内置空间，这里会直接进入 buildStageDir 方法！
        File stageDir = null;
        String stageCid = null;
        if ((params.installFlags & PackageManager.INSTALL_INTERNAL) != 0) {
            final boolean isEphemeral =
                    (params.installFlags & PackageManager.INSTALL_EPHEMERAL) != 0;
            //【*3.1.3】创建文件临时目录；/data/app/vmdl[sessionId].tmp！
            stageDir = buildStageDir(params.volumeUuid, sessionId, isEphemeral);

        } else {
            // 如果是外置，会直接返回 "smdl" + sessionId + ".tmp"
            stageCid = buildExternalStageCid(sessionId);
        }

        //【*3.1.4】创建 PackageInstallerSession 对象！
        session = new PackageInstallerSession(mInternalCallback, mContext, mPm,
                mInstallThread.getLooper(), sessionId, userId, installerPackageName, callingUid,
                params, createdMillis, stageDir, stageCid, false, false);

        synchronized (mSessions) {
            //【9】将新创建的 PackageInstallerSession 添加到 mSessions 集合中！
            mSessions.put(sessionId, session);
        }
        
        //【*3.1.4.2】通知有新的事务创建了，这里是直接回调 Callback 的接口！！
        mCallbacks.notifySessionCreated(session.sessionId, session.userId);

        //【*3.1.6】持久化事务 Session！
        writeSessionsAsync();
        return sessionId;
    }
```


### 3.1.1 PackageInstallerS.getSessionCount

获得 installerUid 创建的 Session 总数：

```java
    private static int getSessionCount(SparseArray<PackageInstallerSession> sessions,
            int installerUid) {
        int count = 0;
        final int size = sessions.size();
        for (int i = 0; i < size; i++) {
            final PackageInstallerSession session = sessions.valueAt(i);
            //【1】匹配创建者的 uid！
            if (session.installerUid == installerUid) {
                count++;
            }
        }
        return count;
    }
```
PackageInstallerService 有两个 SparseBooleanArray 成员变量：

```java
    @GuardedBy("mSessions")
    private final SparseArray<PackageInstallerSession> mSessions = new SparseArray<>();
```
mSessions 保存了所有正在处理的 Session 实例，下标为创建事务的 uid，值为 PackageInstallerSession 是对 Session 的封装！

```java
    /** Historical sessions kept around for debugging purposes */
    @GuardedBy("mSessions")
    private final SparseArray<PackageInstallerSession> mHistoricalSessions = new SparseArray<>();
```
mHistoricalSessions 保存了所有已经处理的 Session 实例，下标为创建事务的 uid，值为 PackageInstallerSession 是对 Session 的封装！

###3.1.2 PackageInstallerS.allocateSessionIdLocked

allocateSessionIdLocked 方法用于给新的 Session 分配 id！

```java
    private int allocateSessionIdLocked() {
        int n = 0;
        int sessionId;
        do {
            sessionId = mRandom.nextInt(Integer.MAX_VALUE - 1) + 1;
            if (!mAllocatedSessions.get(sessionId, false)) {
                mAllocatedSessions.put(sessionId, true);
                return sessionId;
            }
        } while (n++ < 32);

        throw new IllegalStateException("Failed to allocate session ID");
    }
```
PackageInstallerService 还有一个 SparseBooleanArray 成员变量：

```java
    /** All sessions allocated */
    @GuardedBy("mSessions")
    private final SparseBooleanArray mAllocatedSessions = new SparseBooleanArray();
```
mAllocatedSessions 保存了所有的 Session 实例，包括正在处理和已经处理的（mSessions 和 mHistoricalSessions），下标为创建事务的 uid，值为 PackageInstallerSession 是对 Session 的封装！

### 3.1.3 PackageInstallerS.buildStageDir

```java
    private File buildStageDir(String volumeUuid, int sessionId, boolean isEphemeral) {
        final File stagingDir = buildStagingDir(volumeUuid, isEphemeral);
        return new File(stagingDir, "vmdl" + sessionId + ".tmp");
    }
```

我们可以看到，在安装时，创建的临时文件目录是 /data/app/vmdl[sessionId].tmp！！

#### 3.1.3.1 PackageInstallerS.buildStagingDir

buildStagingDir 用于返回文件根目录！

```java
    private File buildStagingDir(String volumeUuid, boolean isEphemeral) {
        if (isEphemeral) {
            // 如果显示指定了 ephemeral 参数的话，返回的是 /data/app-ephemeral 目录！
            return Environment.getDataAppEphemeralDirectory(volumeUuid);
        }
        //【1】默认情况，返回的是 /data/app 目录！
        return Environment.getDataAppDirectory(volumeUuid);
    }
```


### 3.1.4 new PackageInstallerSession - 事务实例

创建 PackageInstallerSession，对前面的 SessionParams 再次封装！

```java
public class PackageInstallerSession extends IPackageInstallerSession.Stub {

    public PackageInstallerSession(PackageInstallerService.InternalCallback callback,
            Context context, PackageManagerService pm, Looper looper, int sessionId, int userId,
            String installerPackageName, int installerUid, SessionParams params, long createdMillis,
            File stageDir, String stageCid, boolean prepared, boolean sealed) {

        //【×3.1.4.1】InternalCallback 回调！
        mCallback = callback;
        mContext = context;
        mPm = pm;
        //【2】创建 Handler 绑定到子线程 mInstallThread，该子线程是在 PackageInstallerService 构造器中创建的！
        //【*4.2】这里通过 mHandlerCallback 指定了一个回调函数！
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
            //【1】获得 container 的 uid 和 gid！
            final int uid = mPm.getPackageUid(PackageManagerService.DEFAULT_CONTAINER_PACKAGE,
                    PackageManager.MATCH_SYSTEM_ONLY, UserHandle.USER_SYSTEM);
            defaultContainerGid = UserHandle.getSharedAppGid(uid);
        } finally {
            Binder.restoreCallingIdentity(identity);
        }
    }
```
可以看到 PackageInstallerSession 除了用来表示一个 Session 之外，由于继承了 IPackageInstallerSession.Stub，因此其还可以作为服务端的桩对象，进行跨进程的通信！

这里的 DEFAULT_CONTAINER_PACKAGE 是一个字符串常量：
```java
static final String DEFAULT_CONTAINER_PACKAGE = "com.android.defcontainer";
```

#### 3.1.4.1 new InternalCallback

在创建 PackageInstallerSession 时，我们传入了一个回调对象 InternalCallback：

```java
private final InternalCallback mInternalCallback = new InternalCallback();
```

InternalCallback 类定义在 PackageInstallerService 中：

```java
    class InternalCallback {
        public void onSessionBadgingChanged(PackageInstallerSession session) {
            mCallbacks.notifySessionBadgingChanged(session.sessionId, session.userId);
            //【*3.1.6】更新持久化文件
            writeSessionsAsync();
        }

        //【1】当 Session 的活跃状态发生变化时，回调触发！
        public void onSessionActiveChanged(PackageInstallerSession session, boolean active) {
            mCallbacks.notifySessionActiveChanged(session.sessionId, session.userId, active);
        }
        //【2】当 Session 的进度发生了变化，会触发该方法！
        public void onSessionProgressChanged(PackageInstallerSession session, float progress) {
            mCallbacks.notifySessionProgressChanged(session.sessionId, session.userId, progress);
        }
        //【3】当 Session 完成后，会触发该方法！
        public void onSessionFinished(final PackageInstallerSession session, boolean success) {
            mCallbacks.notifySessionFinished(session.sessionId, session.userId, success);

            mInstallHandler.post(new Runnable() {
                @Override
                public void run() {
                    synchronized (mSessions) {
                        mSessions.remove(session.sessionId);
                        mHistoricalSessions.put(session.sessionId, session);

                        final File appIconFile = buildAppIconFile(session.sessionId);
                        if (appIconFile.exists()) {
                            appIconFile.delete();
                        }
                        //【*3.1.6.1】更新持久化文件
                        writeSessionsLocked();
                    }
                }
            });
        }

        public void onSessionPrepared(PackageInstallerSession session) {
            //【*3.1.6】更新持久化文件
            writeSessionsAsync();
        }

        public void onSessionSealedBlocking(PackageInstallerSession session) {
            synchronized (mSessions) {
                //【*3.1.6.1】更新持久化文件
                writeSessionsLocked();
            }
        }
    }
```
可以看到，当 Session 的状态发生变化后，InternalCallback 回调会触发！

同时会回调 mCallbacks 的一些接口，而 mCallbacks 是在 PackageInstallerService 中创建的：

```java
    public PackageInstallerService(Context context, PackageManagerService pm) {
        ... ... ...
        //【*3.1.4.1.1】初始化 Callbacks！
        mCallbacks = new Callbacks(mInstallThread.getLooper());
        ... ... ...
    }
```
我们来看下 Callbacks 的逻辑！

##### 3.1.4.1.1 new Callbacks

Callbacks 是 Handler 的子类，持有子线程 mInstallThread 的 looper，Callbacks 是在

```java
   private static class Callbacks extends Handler {
    private static class Callbacks extends Handler {
        //【1】内部会处理的消息！
        private static final int MSG_SESSION_CREATED = 1;
        private static final int MSG_SESSION_BADGING_CHANGED = 2;
        private static final int MSG_SESSION_ACTIVE_CHANGED = 3;
        private static final int MSG_SESSION_PROGRESS_CHANGED = 4;
        private static final int MSG_SESSION_FINISHED = 5;

        //【2】监听安装状态的观察者 list
        private final RemoteCallbackList<IPackageInstallerCallback>
                mCallbacks = new RemoteCallbackList<>();

        public Callbacks(Looper looper) {
            super(looper);
        }
```
Callbacks 内部有一个 list，保存了所有监听 Session 状态变化的观察者，同时提供了 register 接口，动态注册观察者！
```java
        public void register(IPackageInstallerCallback callback, int userId) {
            mCallbacks.register(callback, new UserHandle(userId));
        }

        public void unregister(IPackageInstallerCallback callback) {
            mCallbacks.unregister(callback);
        }

```
Callbacks 内部有如下的 notify 接口，来通知 Session 的状态变化！

##### 3.1.4.1.2 Callbacks.notifySessionXXXX
```java
        private void notifySessionCreated(int sessionId, int userId) {
            obtainMessage(MSG_SESSION_CREATED, sessionId, userId).sendToTarget();
        }

        private void notifySessionBadgingChanged(int sessionId, int userId) {
            obtainMessage(MSG_SESSION_BADGING_CHANGED, sessionId, userId).sendToTarget();
        }

        private void notifySessionActiveChanged(int sessionId, int userId, boolean active) {
            obtainMessage(MSG_SESSION_ACTIVE_CHANGED, sessionId, userId, active).sendToTarget();
        }

        private void notifySessionProgressChanged(int sessionId, int userId, float progress) {
            obtainMessage(MSG_SESSION_PROGRESS_CHANGED, sessionId, userId, progress).sendToTarget();
        }

        public void notifySessionFinished(int sessionId, int userId, boolean success) {
            obtainMessage(MSG_SESSION_FINISHED, sessionId, userId, success).sendToTarget();
        }
    }
```

本质上是发送不同的 msg，下面来看看：

##### 3.1.4.1.3 Callbacks.handleMessage

```java
        @Override
        public void handleMessage(Message msg) {
            final int userId = msg.arg2;
            final int n = mCallbacks.beginBroadcast();
            for (int i = 0; i < n; i++) {
                //【1】遍历所有的观察者
                final IPackageInstallerCallback callback = mCallbacks.getBroadcastItem(i);
                final UserHandle user = (UserHandle) mCallbacks.getBroadcastCookie(i);
                // TODO: dispatch notifications for slave profiles
                if (userId == user.getIdentifier()) {
                    try {
                        //【*3.1.4.1.3】分发 Session 消息给观察者！
                        invokeCallback(callback, msg);
                    } catch (RemoteException ignored) {
                    }
                }
            }
            mCallbacks.finishBroadcast();
        }
```
最后调用了 invokeCallback，其实可以看到 IPackageInstallerCallback 针对于不同的消息也有不同的接口：


##### 3.1.4.1.4 Callbacks.invokeCallback

继续分析：
```java
        private void invokeCallback(IPackageInstallerCallback callback, Message msg)
                throws RemoteException {
            final int sessionId = msg.arg1;
            switch (msg.what) {
                case MSG_SESSION_CREATED:
                    callback.onSessionCreated(sessionId);
                    break;
                case MSG_SESSION_BADGING_CHANGED:
                    callback.onSessionBadgingChanged(sessionId);
                    break;
                case MSG_SESSION_ACTIVE_CHANGED:
                    callback.onSessionActiveChanged(sessionId, (boolean) msg.obj);
                    break;
                case MSG_SESSION_PROGRESS_CHANGED:
                    callback.onSessionProgressChanged(sessionId, (float) msg.obj);
                    break;
                case MSG_SESSION_FINISHED:
                    callback.onSessionFinished(sessionId, (boolean) msg.obj);
                    break;
            }
        }
```

这里我们先分析到这里！

### 3.1.6 PackageInstallerS.writeSessionsAsync - 持久化事务

```java
    private void writeSessionsAsync() {
        IoThread.getHandler().post(new Runnable() {
            @Override
            public void run() {
                synchronized (mSessions) {
                    //【*3.1.6.1】将事务记录到 mSessionsFile 文件中！
                    writeSessionsLocked();
                }
            }
        });
    }
```

#### 3.1.6.1 PackageInstallerS.writeSessionsLocked

mSessionsFile 在 PackageInstallerService 构造器中有见过：/data/system/install_sessions.xml！！
```java
    private void writeSessionsLocked() {
        if (LOGD) Slog.v(TAG, "writeSessionsLocked()");

        FileOutputStream fos = null;
        try {
            fos = mSessionsFile.startWrite();

            XmlSerializer out = new FastXmlSerializer();
            out.setOutput(fos, StandardCharsets.UTF_8.name());
            out.startDocument(null, true);
            out.startTag(null, TAG_SESSIONS); //【1】写入 sessions 标签！
            final int size = mSessions.size();
            for (int i = 0; i < size; i++) {
                final PackageInstallerSession session = mSessions.valueAt(i);
                //【*3.1.6.2】写入所有的 Sessions！！
                writeSessionLocked(out, session);
            }
            out.endTag(null, TAG_SESSIONS);
            out.endDocument();

            mSessionsFile.finishWrite(fos);
        } catch (IOException e) {
            if (fos != null) {
                mSessionsFile.failWrite(fos);
            }
        }
    }
```

#### 3.1.6.2 PackageInstallerS.writeSessionLocked

```java
    private void writeSessionLocked(XmlSerializer out, PackageInstallerSession session)
            throws IOException {
        final SessionParams params = session.params;

        out.startTag(null, TAG_SESSION); // 写入 session 标签

        writeIntAttribute(out, ATTR_SESSION_ID, session.sessionId); // 写入 sessionId 属性
        writeIntAttribute(out, ATTR_USER_ID, session.userId); // 写入 userId 属性
        writeStringAttribute(out, ATTR_INSTALLER_PACKAGE_NAME, // 写入 installerPackageName 属性
                session.installerPackageName);
        writeIntAttribute(out, ATTR_INSTALLER_UID, session.installerUid); // 写入 installerUid 属性
        writeLongAttribute(out, ATTR_CREATED_MILLIS, session.createdMillis);  // 写入 createdMillis 属性
        if (session.stageDir != null) {
            writeStringAttribute(out, ATTR_SESSION_STAGE_DIR, // 写入 sessionStageDir 属性
                    session.stageDir.getAbsolutePath());
        }
        if (session.stageCid != null) {
            writeStringAttribute(out, ATTR_SESSION_STAGE_CID, session.stageCid); // 写入 sessionStageCid 属性
        }
        writeBooleanAttribute(out, ATTR_PREPARED, session.isPrepared()); // 写入 prepared 属性
        writeBooleanAttribute(out, ATTR_SEALED, session.isSealed()); // 写入 sealed 属性

        writeIntAttribute(out, ATTR_MODE, params.mode); // 写入 mode 子标签
        writeIntAttribute(out, ATTR_INSTALL_FLAGS, params.installFlags); // 写入 installFlags 属性
        writeIntAttribute(out, ATTR_INSTALL_LOCATION, params.installLocation); // 写入 installLocation 属性
        writeLongAttribute(out, ATTR_SIZE_BYTES, params.sizeBytes); // 写入 sizeBytes 属性
        writeStringAttribute(out, ATTR_APP_PACKAGE_NAME, params.appPackageName); // 写入 appPackageName 属性
        writeStringAttribute(out, ATTR_APP_LABEL, params.appLabel); // 写入 appLabel 属性
        writeUriAttribute(out, ATTR_ORIGINATING_URI, params.originatingUri); // 写入 originatingUri 属性
        writeIntAttribute(out, ATTR_ORIGINATING_UID, params.originatingUid); // 写入 originatingUid 属性
        writeUriAttribute(out, ATTR_REFERRER_URI, params.referrerUri); // 写入 referrerUri 属性
        writeStringAttribute(out, ATTR_ABI_OVERRIDE, params.abiOverride); // 写入 abiOverride 属性
        writeStringAttribute(out, ATTR_VOLUME_UUID, params.volumeUuid); // 写入 volumeUuid 属性

        // Persist app icon if changed since last written
        final File appIconFile = buildAppIconFile(session.sessionId);
        if (params.appIcon == null && appIconFile.exists()) {
            appIconFile.delete();
        } else if (params.appIcon != null
                && appIconFile.lastModified() != params.appIconLastModified) {
            if (LOGD) Slog.w(TAG, "Writing changed icon " + appIconFile);
            FileOutputStream os = null;
            try {
                os = new FileOutputStream(appIconFile);
                params.appIcon.compress(CompressFormat.PNG, 90, os);
            } catch (IOException e) {
                Slog.w(TAG, "Failed to write icon " + appIconFile + ": " + e.getMessage());
            } finally {
                IoUtils.closeQuietly(os);
            }

            params.appIconLastModified = appIconFile.lastModified();
        }
        //【*3.1.6.3】将要在安装时授予的运行时权限集合写入到持久化文件中！
        writeGrantedRuntimePermissions(out, params.grantedRuntimePermissions);

        out.endTag(null, TAG_SESSION);
    }
```

#### 3.1.6.3 PackageInstallerS.writeGrantedRuntimePermissions
```java
    private static void writeGrantedRuntimePermissions(XmlSerializer out,
            String[] grantedRuntimePermissions) throws IOException {
        if (grantedRuntimePermissions != null) {
            for (String permission : grantedRuntimePermissions) {
                out.startTag(null, TAG_GRANTED_RUNTIME_PERMISSION); // 写入 granted-runtime-permission 子标签
                writeStringAttribute(out, ATTR_NAME, permission); // 写入 name 属性！
                out.endTag(null, TAG_GRANTED_RUNTIME_PERMISSION);
            }
        }
    }
```

## 3.2 PackageInstallerS.openSession - 获得事务

openSession 方法可以获得 id 对应的 PackageInstallerSession！

```java
    @Override
    public IPackageInstallerSession openSession(int sessionId) {
        try {
            //【*3.2.1】调用另外一个方法！
            return openSessionInternal(sessionId);
        } catch (IOException e) {
            throw ExceptionUtils.wrap(e);
        }
    }
```
### 3.2.1 PackageInstallerS.openSessionInternal
```java
    private IPackageInstallerSession openSessionInternal(int sessionId) throws IOException {
        synchronized (mSessions) {
            final PackageInstallerSession session = mSessions.get(sessionId);
            //【*3.2.2】判断 uid 是否被允许获得该事务！
            if (session == null || !isCallingUidOwner(session)) {
                throw new SecurityException("Caller has no access to session " + sessionId);
            }
            session.open();
            return session;
        }
    }
```
### 3.2.2 PackageInstallerS.isCallingUidOwner
```java
    private boolean isCallingUidOwner(PackageInstallerSession session) {
        final int callingUid = Binder.getCallingUid();
        if (callingUid == Process.ROOT_UID) {
            //【1】要么是 root 用户！
            return true;
        } else {
            //【2】要么调用者的 uid 必须是事务的创建者 uid！
            return (session != null) && (callingUid == session.installerUid);
        }
    }
```

## 3.3 PackageInstallerS.getSessionInfo - 获得事务

getSessionInfo 方法也是获得 id 对应的 Session 实例，但是封装的结果不同！

```java
    @Override
    public SessionInfo getSessionInfo(int sessionId) {
        synchronized (mSessions) {
            final PackageInstallerSession session = mSessions.get(sessionId);
            //【*3.3.1】调用 PackageInstallerSession 的 generateInfo 方法！
            return session != null ? session.generateInfo() : null;
        }
    }
```

### 3.3.1 PackageInstallerSession.generateInfo

PackageInstallerSession 会返回一个 SessionInfo 对象！用于保存 Session 的一些细节信息！

```java
    public SessionInfo generateInfo() {
        final SessionInfo info = new SessionInfo();
        synchronized (mLock) {
            info.sessionId = sessionId;
            info.installerPackageName = installerPackageName;
            info.resolvedBaseCodePath = (mResolvedBaseFile != null) ?
                    mResolvedBaseFile.getAbsolutePath() : null;
            info.progress = mProgress;
            info.sealed = mSealed;
            info.active = mActiveCount.get() > 0;

            info.mode = params.mode;
            info.sizeBytes = params.sizeBytes;
            info.appPackageName = params.appPackageName;
            info.appIcon = params.appIcon;
            info.appLabel = params.appLabel;
        }
        return info;
    }
```

### 3.3.2 new PackageInstaller.SessionInfo

SessionInfo 定义在 PackageInstaller.java 中，用于保存 Session 的信息，这里我们简单的分析下！

```java
    public static class SessionInfo implements Parcelable {

        /** {@hide} */
        public int sessionId;
        /** {@hide} */
        public String installerPackageName;
        /** {@hide} */
        public String resolvedBaseCodePath;
        /** {@hide} */
        public float progress;
        /** {@hide} */
        public boolean sealed;
        /** {@hide} */
        public boolean active;

        /** {@hide} */
        public int mode;
        /** {@hide} */
        public long sizeBytes;
        /** {@hide} */
        public String appPackageName;
        /** {@hide} */
        public Bitmap appIcon;
        /** {@hide} */
        public CharSequence appLabel;

        /** {@hide} */
        public SessionInfo() {
        }

        /** {@hide} */
        public SessionInfo(Parcel source) {
            sessionId = source.readInt();
            installerPackageName = source.readString();
            resolvedBaseCodePath = source.readString();
            progress = source.readFloat();
            sealed = source.readInt() != 0;
            active = source.readInt() != 0;

            mode = source.readInt();
            sizeBytes = source.readLong();
            appPackageName = source.readString();
            appIcon = source.readParcelable(null);
            appLabel = source.readString();
        }
        ... ... ...
    }
```

# 4 PackageInstallerSession

## 4.1 openWrite

这里的 name 传入的是 base.apk

```java
    @Override
    public ParcelFileDescriptor openWrite(String name, long offsetBytes, long lengthBytes) {
        try {
            //【*4.1.1】继续调用 openWriteInternal 方法处理；
            return openWriteInternal(name, offsetBytes, lengthBytes);
        } catch (IOException e) {
            throw ExceptionUtils.wrap(e);
        }
    }
```
### 4.1.1 openWriteInternal

继续来看，这里的参数 name 传入的是 "base.apk"

```java
    private ParcelFileDescriptor openWriteInternal(String name, long offsetBytes, long lengthBytes)
            throws IOException {
        final FileBridge bridge;
        synchronized (mLock) {
            assertPreparedAndNotSealed("openWrite");

            bridge = new FileBridge();
            mBridges.add(bridge);
        }

        try {
            if (!FileUtils.isValidExtFilename(name)) {
                throw new IllegalArgumentException("Invalid name: " + name);
            }
            final File target;
            final long identity = Binder.clearCallingIdentity();
            try {
                //【1】创建 /data/app/vmdl[sessionId].tmp/base.apk 对应的文件对象！
                target = new File(resolveStageDir(), name);
            } finally {
                Binder.restoreCallingIdentity(identity);
            }

            //【2】返回其文件描述符；
            final FileDescriptor targetFd = Libcore.os.open(target.getAbsolutePath(),
                    O_CREAT | O_WRONLY, 0644);
            Os.chmod(target.getAbsolutePath(), 0644);

            //【3】如果指定了大小，那么这里会做一次空间回收！
            if (lengthBytes > 0) {
                final StructStat stat = Libcore.os.fstat(targetFd);
                final long deltaBytes = lengthBytes - stat.st_size;
                //【3.1】回收空间；
                if (stageDir != null && deltaBytes > 0) {
                    mPm.freeStorage(params.volumeUuid, deltaBytes);
                }
                Libcore.os.posix_fallocate(targetFd, 0, lengthBytes);
            }

            if (offsetBytes > 0) {
                Libcore.os.lseek(targetFd, offsetBytes, OsConstants.SEEK_SET);
            }
            //【4】使用 FileBridge 封装文件描述符！
            bridge.setTargetFile(targetFd);
            bridge.start();
            return new ParcelFileDescriptor(bridge.getClientSocket());

        } catch (ErrnoException e) {
            throw e.rethrowAsIOException();
        }
    }
```
整个方法很简单，不多说了！

## 4.2 commit - 提交事务核心入口

在上面我们分析到，会进入 PackageInstallerSession.commit 提交事务。下面我们继续来分析！

```java
    @Override
    public void commit(IntentSender statusReceiver) {
        Preconditions.checkNotNull(statusReceiver);

        final boolean wasSealed;
        synchronized (mLock) {
            wasSealed = mSealed;
            if (!mSealed) {
                // 校验所有的写操作都已经完成了，正常情况下，是肯定完成了的！
                for (FileBridge bridge : mBridges) {
                    if (!bridge.isClosed()) {
                        throw new SecurityException("Files still open");
                    }
                }
                mSealed = true;
            }

            //【1】由于此时文件已经拷贝完成，这里更新进度为完成！
            mClientProgress = 1f;
            
            //【*4.2.3.5】通知结果！
            computeProgressLocked(true);
        }

        if (!wasSealed) {
            // Persist the fact that we've sealed ourselves to prevent
            // mutations of any hard links we create. We do this without holding
            // the session lock, since otherwise it's a lock inversion.
            mCallback.onSessionSealedBlocking(this);
        }

        //【2】活跃计数 +1，表示该 Session 处于活跃状态；
        mActiveCount.incrementAndGet();

        //【*4.2.1】创建了一个 PackageInstallObserverAdapter 实例！
        // 会将前面创建的 IntentSender 实例，作为参数传入！
        final PackageInstallObserverAdapter adapter = new PackageInstallObserverAdapter(mContext,
                statusReceiver, sessionId, mIsInstallerDeviceOwner, userId);

        //【*4.2.2】发送 MSG_COMMIT 消息
        mHandler.obtainMessage(MSG_COMMIT, adapter.getBinder()).sendToTarget();
    }
```
mActiveCount 加 1，表示该事务处于活跃状态，直到安装完成！

这里的 PackageInstallObserverAdapter.getBinder() 返回是一个服务端 Stub 桩对象！


整个流程很清晰，我们继续看：

### 4.2.1 new PackageInstallObserverAdapter

PackageInstallObserverAdapter 定义在 PackageInstallService 中：

```java
    static class PackageInstallObserverAdapter extends PackageInstallObserver {
        private final Context mContext; // 系统进程的上下文
        private final IntentSender mTarget; // 前面创建的 IntentSender 实例
        private final int mSessionId; // 事务的 id！
        private final boolean mShowNotification; // 是否显示通知，取决于安装者是否是设备用户！
        private final int mUserId;

        public PackageInstallObserverAdapter(Context context, IntentSender target, int sessionId,
                boolean showNotification, int userId) {
            mContext = context;
            mTarget = target;
            mSessionId = sessionId;
            mShowNotification = showNotification; 
            mUserId = userId;
        }

```
PackageInstallObserverAdapter 继承了 PackageInstallObserver：

```java
public class PackageInstallObserver {
    //【1】服务端桩对象！
    private final IPackageInstallObserver2.Stub mBinder = new IPackageInstallObserver2.Stub() {
        @Override
        public void onUserActionRequired(Intent intent) {
            //【1.1】调用子类的 onUserActionRequired 方法！
            PackageInstallObserver.this.onUserActionRequired(intent);
        }

        @Override
        public void onPackageInstalled(String basePackageName, int returnCode,
                String msg, Bundle extras) {
            //【1.2】调用子类的 onPackageInstalled 方法！
            PackageInstallObserver.this.onPackageInstalled(basePackageName, returnCode, msg,
                    extras);
        }
    };

    /** {@hide} */
    public IPackageInstallObserver2 getBinder() {
        //【2】用于返回服务端桩对象！
        return mBinder;
    }
    ... ... ...
}
```

PackageInstallObserverAdapter 继承了 PackageInstallObserver，并覆写了以下两个方法：

```java
onUserActionRequired
onPackageInstalled
```

#### 4.2.1.1 PackageInstallObserverAdapter.onUserActionRequired

覆写 onUserActionRequired，当安装过程需要用户参与授权时，会回调该接口，此时会中断安装，事务从 active 变为 idle 状态！

```java
        @Override
        public void onUserActionRequired(Intent intent) {
            final Intent fillIn = new Intent();
            //【1】事务 id！
            fillIn.putExtra(PackageInstaller.EXTRA_SESSION_ID, mSessionId);
            //【2】当前的状态：PackageInstaller.STATUS_PENDING_USER_ACTION！
            fillIn.putExtra(PackageInstaller.EXTRA_STATUS,
                    PackageInstaller.STATUS_PENDING_USER_ACTION);
            fillIn.putExtra(Intent.EXTRA_INTENT, intent); // 额外的 intent
            try {
                //【*2.9.4.1】发送 intent，其实，这里我们知道，该 intent 会发送到前面的 LocalIntentReceiver!
                mTarget.sendIntent(mContext, 0, fillIn, null, null);
            } catch (SendIntentException ignored) {
            }
        }
```

关于这个额外的 Intent，我们后面会看到！

#### 4.1.1.2 PackageInstallObserverAdapter.onPackageInstalled

覆写 onPackageInstalled，当安装完成后，会回调该接口！

```java
        @Override
        public void onPackageInstalled(String basePackageName, int returnCode, String msg,
                Bundle extras) {
            //【1】当安装成功，并且需要弹出通知时，会在这里显示通知！
            if (PackageManager.INSTALL_SUCCEEDED == returnCode && mShowNotification) {
                boolean update = (extras != null) && extras.getBoolean(Intent.EXTRA_REPLACING);
                Notification notification = buildSuccessNotification(mContext,
                        mContext.getResources()
                                .getString(update ? R.string.package_updated_device_owner :
                                        R.string.package_installed_device_owner),
                        basePackageName,
                        mUserId);
                if (notification != null) {
                    NotificationManager notificationManager = (NotificationManager)
                            mContext.getSystemService(Context.NOTIFICATION_SERVICE);
                    notificationManager.notify(basePackageName, 0, notification);
                }
            }
            
            //【2】创建一个 intent，保存了安装结果等信息！
            final Intent fillIn = new Intent();
            fillIn.putExtra(PackageInstaller.EXTRA_PACKAGE_NAME, basePackageName);
            fillIn.putExtra(PackageInstaller.EXTRA_SESSION_ID, mSessionId);
            fillIn.putExtra(PackageInstaller.EXTRA_STATUS,
                    PackageManager.installStatusToPublicStatus(returnCode));
            fillIn.putExtra(PackageInstaller.EXTRA_STATUS_MESSAGE,
                    PackageManager.installStatusToString(returnCode, msg));
            fillIn.putExtra(PackageInstaller.EXTRA_LEGACY_STATUS, returnCode);
            if (extras != null) {
                final String existing = extras.getString(
                        PackageManager.EXTRA_FAILURE_EXISTING_PACKAGE);
                if (!TextUtils.isEmpty(existing)) {
                    fillIn.putExtra(PackageInstaller.EXTRA_OTHER_PACKAGE_NAME, existing);
                }
            }
            try {
                //【*2.9.4.1】发送 intent，其实，这里我们知道，该 intent 会发送到前面的 LocalIntentReceiver
                mTarget.sendIntent(mContext, 0, fillIn, null, null);
            } catch (SendIntentException ignored) {
            }
        }
```
我们继续分析！

### 4.2.2 send MSG_COMMIT

代码片段如下：

```java
    //【*4.2.2.1】发送 MSG_COMMIT 消息
    mHandler.obtainMessage(MSG_COMMIT, adapter.getBinder()).sendToTarget();
```
mHandler 的初始化是在 PackageInstallerSession 的构造器中：

```java
    public PackageInstallerSession(PackageInstallerService.InternalCallback callback,
            Context context, PackageManagerService pm, Looper looper, int sessionId, int userId,
            String installerPackageName, int installerUid, SessionParams params, long createdMillis,
            File stageDir, String stageCid, boolean prepared, boolean sealed) {
        mCallback = callback;
        mContext = context;
        mPm = pm;
        //【*4.2.1】handler 会处理该消息，我们看到，其传入了一个 mHandlerCallback 回调！
        mHandler = new Handler(looper, mHandlerCallback);
        ... ... ...
    }
```
如果我们直接发送 MSG_COMMIT 消息，回调 mHandlerCallback 会立刻触发：

#### 4.2.2.1 Handler.Callback.handleMessage[MSG_COMMIT]

```java
    private final Handler.Callback mHandlerCallback = new Handler.Callback() {
        @Override
        public boolean handleMessage(Message msg) {
            //【1】获得要安装的应用的信息，如果是第一次安装的话，那么二者返回的均为 null！
            final PackageInfo pkgInfo = mPm.getPackageInfo(
                    params.appPackageName, PackageManager.GET_SIGNATURES /*flags*/, userId);
            final ApplicationInfo appInfo = mPm.getApplicationInfo(
                    params.appPackageName, 0, userId);

            synchronized (mLock) {
                if (msg.obj != null) {
                    //【2】获得前面 PackageInstallObserverAdapter 内部的 PackageInstallObserver2 桩对象！
                    // 保存到了 mRemoteObserver 中！
                    mRemoteObserver = (IPackageInstallObserver2) msg.obj;
                }

                try {
                    //【*4.2.3】调用 commitLocked 继续处理！
                    commitLocked(pkgInfo, appInfo);

                } catch (PackageManagerException e) {
                    final String completeMsg = ExceptionUtils.getCompleteMessage(e);
                    Slog.e(TAG, "Commit of session " + sessionId + " failed: " + completeMsg);
                    destroyInternal();
                    dispatchSessionFinished(e.error, completeMsg, null);
                }

                return true;
            }
        }
    };
```
PackageInstallerSession 内部有一个 mRemoteObserver 成员变量，后面我们会见到：

```java
    @GuardedBy("mLock")
    private IPackageInstallObserver2 mRemoteObserver;
```

接下来，进入了 commitLocked 方法！

### 4.2.3 commitLocked 

继续来看：

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
            //【*4.2.3.1】解析安装地址，即前文APK文件copy后的目的地址
            resolveStageDir();
        } catch (IOException e) {
            throw new PackageManagerException(INSTALL_FAILED_CONTAINER_ERROR,
                    "Failed to resolve stage location", e);
        }

        //【*4.2.3.2】校验安装有效性！
        validateInstallLocked(pkgInfo, appInfo);

        Preconditions.checkNotNull(mPackageName); // 非 null 校验！
        Preconditions.checkNotNull(mSignatures);
        Preconditions.checkNotNull(mResolvedBaseFile);

        //【1】mPermissionsAccepted 为 true，那么用户就可以静默安装了，如果为 false，那么就需要用户确认权限！！
        if (!mPermissionsAccepted) {

            //【1.1】这里会创建一个 intent， action 为 PackageInstaller.ACTION_CONFIRM_PERMISSIONS
            // 目标应用是 PackageInstaller，这里会先进入 packageinstaller 中确认权限信息！
            final Intent intent = new Intent(PackageInstaller.ACTION_CONFIRM_PERMISSIONS);
            intent.setPackage(mContext.getPackageManager().getPermissionControllerPackageName());
            intent.putExtra(PackageInstaller.EXTRA_SESSION_ID, sessionId); // 事务 id 也要从传递过去！
            try {
                //【*4.2.1.1】回调了 PackageInstallObserverAdapter 的 onUserActionRequired 接口
                // 将 intent 传递过去！
                mRemoteObserver.onUserActionRequired(intent);
            } catch (RemoteException ignored) {
            }

            //【*4.2.3.3】关闭该事务，使其从 active 变为 idle 状态！！
            close();
            // 停止安装，等待用户确认权限，用户在 PackageInstaller 点击安装，安装会继续！！
            return;
        }

        //【2】安装到外置时，计算最终安装大小！
        if (stageCid != null) {
            //【*4.2.3.4】计算大小
            final long finalSize = calculateInstalledSize();
            resizeContainer(stageCid, finalSize);
        }

        //【3】如果安装方式是继承已存在的 apk，那我们就要尝试基于已有的安装进行安装，这个一般用于安装和卸载 split apk！
        if (params.mode == SessionParams.MODE_INHERIT_EXISTING) {
            try {
                final List<File> fromFiles = mResolvedInheritedFiles;
                final File toDir = resolveStageDir(); // 这是我们本次安装的目录

                if (LOGD) Slog.d(TAG, "Inherited files: " + mResolvedInheritedFiles);
                if (!mResolvedInheritedFiles.isEmpty() && mInheritedFilesBase == null) {
                    throw new IllegalStateException("mInheritedFilesBase == null");
                }
                //【3.1】如果可以直接建立 link 的话，不行的话，就 copy！
                if (isLinkPossible(fromFiles, toDir)) {
                    if (!mResolvedInstructionSets.isEmpty()) {
                        final File oatDir = new File(toDir, "oat");
                        createOatDirs(mResolvedInstructionSets, oatDir);
                    }
                    linkFiles(fromFiles, toDir, mInheritedFilesBase);
                } else {
                    //【3.2】执行拷贝，其实是将已经存在的目录的 apk，oat 文件拷贝到了这个目录！
                    // 对于要 remove 的文件，则会跳过拷贝！
                    copyFiles(fromFiles, toDir);
                }
            } catch (IOException e) {
                throw new PackageManagerException(INSTALL_FAILED_INSUFFICIENT_STORAGE,
                        "Failed to inherit existing install", e);
            }
        }

        //【4】更新进度！
        mInternalProgress = 0.5f;
        
        //【*4.2.3.5】通知结果！
        computeProgressLocked(true);

        //【*4.2.3.6】提取 lib 库文件到目录中！
        extractNativeLibraries(mResolvedStageDir, params.abiOverride);

        // 如果安装到外置，释放 Container
        if (stageCid != null) {
            finalizeAndFixContainer(stageCid);
        }

        //【5】创建一个本地的安装观察者，监听安装结果！
        final IPackageInstallObserver2 localObserver = new IPackageInstallObserver2.Stub() {
            @Override
            public void onUserActionRequired(Intent intent) {
                throw new IllegalStateException();
            }

            @Override
            public void onPackageInstalled(String basePackageName, int returnCode, String msg,
                    Bundle extras) {
                //【*5.8.2.1】删除目录文件；
                destroyInternal();
                //【*5.8.2.2】处理回调，通知监听者！
                dispatchSessionFinished(returnCode, msg, extras);
            }
        };
        
        //【6】处理要安装的目标设别用户
        final UserHandle user;
        if ((params.installFlags & PackageManager.INSTALL_ALL_USERS) != 0) {
            user = UserHandle.ALL;
        } else {
            user = new UserHandle(userId);
        }
        
        mRelinquished = true;

        //【×5.1】然后继续安装！
        mPm.installStage(mPackageName, stageDir, stageCid, localObserver, params,
                installerPackageName, installerUid, user, mCertificates);
    }

```
继续分析：

#### 4.2.3.1 resolveStageDir
```java
    private File resolveStageDir() throws IOException {
        synchronized (mLock) {
            //【1】将之前保存的目录 stageDir 值赋给 mResolvedStageDir 并返回！
            if (mResolvedStageDir == null) {
                if (stageDir != null) {
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
逻辑很简单，不多说了！

#### 4.2.3.2 validateInstallLocked

校验安装有效性，这里的 mResolvedStageDir 就是前面的 /data/app/vmdl[sessionId].tmp 目录！

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
        // 并判断是否正常，如果该目录下没有任何 apk 和 .removed 文件，那么抛出异常！！
        final File[] addedFiles = mResolvedStageDir.listFiles(sAddedFilter);
        if (ArrayUtils.isEmpty(addedFiles) && removeSplitList.size() == 0) {
            throw new PackageManagerException(INSTALL_FAILED_INVALID_APK, "No packages staged");
        }

        //【3】遍历该目录下的非 .removed 文件，解析其中的 apk 文件，也就是我们之前 copy 到这里的目标文件！
        // 卸载和安装 split 都会进入这里！
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

            //【*4.3.2.1】校验 apk 关联性！
            assertApkConsistent(String.valueOf(addedFile), apk);

            //【3.4】设置 apk 文件的目标名称！
            final String targetName;
            if (apk.splitName == null) {
                targetName = "base.apk"; // 一般情况下，我们只装 base apk！
            } else {
                targetName = "split_" + apk.splitName + ".apk"; // 对于 split apk 名称是这样的！！
            }
            if (!FileUtils.isValidExtFilename(targetName)) {
                throw new PackageManagerException(INSTALL_FAILED_INVALID_APK,
                        "Invalid filename: " + targetName);
            }
            
            //【3.5】当 addedFile 命名不标准的话，会改名;
            final File targetFile = new File(mResolvedStageDir, targetName);
            if (!addedFile.equals(targetFile)) {
                addedFile.renameTo(targetFile);
            }

            //【3.6】找到了 base apk，将其保存到 mResolvedBaseFile，同时将其添加到 mResolvedStagedFiles 中！
            if (apk.splitName == null) {
                mResolvedBaseFile = targetFile;
            }
            mResolvedStagedFiles.add(targetFile);
        }

        //【4】处理 .removed 文件（卸载 split apk，才会进入这里）
        if (removeSplitList.size() > 0) {
            //【4.1】找不到 split apk，抛出异常！
            for (String splitName : removeSplitList) {
                if (!ArrayUtils.contains(pkgInfo.splitNames, splitName)) {
                    throw new PackageManagerException(INSTALL_FAILED_INVALID_APK,
                            "Split not found: " + splitName);
                }
            }

            // 再次获得要安装的应用的包名，版本号，签名；
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
            //【5.1】全量安装必须要有 base.apk；
            if (!stagedSplits.contains(null)) {
                throw new PackageManagerException(INSTALL_FAILED_INVALID_APK,
                        "Full install must include a base package");
            }

        } else {
            //【5.2】部分安装必须基于现有的安装（卸载和安装 split apk 也会进入这里）！
            if (appInfo == null) {
                throw new PackageManagerException(INSTALL_FAILED_INVALID_APK,
                        "Missing existing base package for " + mPackageName);
            }

            //【5.3】获得已存在的 apk 安装信息，这里会解析主 apk 的安装信息！
            final PackageLite existing;
            final ApkLite existingBase;
            try {
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

            //【5.5】继承已有的 split apk，要继承的 split apk 不能是在 removeSplitList 列表中！！
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
总结下：


1、返回 /data/app/vmdl[sessionId].tmp 目录下所有的 .removed 文件，去除后缀，将前缀名保存到 removeSplitList；
2、/data/app/vmdl[sessionId].tmp 目录下必须要有 apk 文件或者 .removed 文件；
3、遍历该目录下的非 .removed 文件，对其 packagename versionCode 和签名做关联校验；
4、stagedSplits 用于保存该目录下的所有 apk 的 splitName，base apk 的 splitName 为 null；
5、mResolvedBaseFile 用于保存 base apk；
6、mResolvedStagedFiles 用于保存目录下所有的 apk；

7、removeSplitList 大于 0，说明有要移除的 split apk，前提是主 apk 要有 split apk！

8、对于 MODE_FULL_INSTALL，全量安装，必须要有 base apk！
9、对于 MODE_INHERIT_EXISTING，继承安装，会再次解析主 apk，收集那些不在 removeSplitList 列表中的 splitApk 路径到 mResolvedInheritedFiles 中；


##### 4.2.3.2.1 assertApkConsistent

校验 apk 的一致性，ApkLite apk 为扫描到的文件！

```java
    private void assertApkConsistent(String tag, ApkLite apk)
            throws PackageManagerException {
        //【1】扫描到的 apk 的包名必须和该目录下第一个被扫描到的 apk 包名保持一致！
        if (!mPackageName.equals(apk.packageName)) {
            throw new PackageManagerException(INSTALL_FAILED_INVALID_APK, tag + " package "
                    + apk.packageName + " inconsistent with " + mPackageName);
        }
        //【2】如果是要继承已安装的 apk，那么包名要一样！
        if (params.appPackageName != null && !params.appPackageName.equals(apk.packageName)) {
            throw new PackageManagerException(INSTALL_FAILED_INVALID_APK, tag
                    + " specified package " + params.appPackageName
                    + " inconsistent with " + apk.packageName);
        }
        //【3】扫描到的 apk 的版本号必须和该目录下第一个被扫描到的 apk 版本号保持一致！
        if (mVersionCode != apk.versionCode) {
            throw new PackageManagerException(INSTALL_FAILED_INVALID_APK, tag
                    + " version code " + apk.versionCode + " inconsistent with "
                    + mVersionCode);
        }
        //【4】扫描到的 apk 的签名必须和该目录下第一个被扫描到的 apk 签名保持一致！
        if (!Signature.areExactMatch(mSignatures, apk.signatures)) {
            throw new PackageManagerException(INSTALL_FAILED_INVALID_APK,
                    tag + " signatures are inconsistent");
        }
    }
```

流程很简单，不多说了！

#### 4.2.3.3 close

close 方法很简单，将事务的 mActiveCount 引用计数自减 1，同时回调 

```java
    @Override
    public void close() {
        //【1】引用计数减去 1；
        if (mActiveCount.decrementAndGet() == 0) {
            //【*3.1.4.2】反馈事务状态！
            mCallback.onSessionActiveChanged(this, false);
        }
    }
```


#### 4.2.3.4 calculateInstalledSize

用于计算最终的安装大小！

```java
    private long calculateInstalledSize() throws PackageManagerException {
        Preconditions.checkNotNull(mResolvedBaseFile);

        final ApkLite baseApk;
        try {
            //【1】解析要安装的 apk 的数据信息！
            baseApk = PackageParser.parseApkLite(mResolvedBaseFile, 0);
        } catch (PackageParserException e) {
            throw PackageManagerException.from(e);
        }
        //【2】获得其 split apk 的文件路径！
        final List<String> splitPaths = new ArrayList<>();
        for (File file : mResolvedStagedFiles) {
            if (mResolvedBaseFile.equals(file)) continue;
            splitPaths.add(file.getAbsolutePath());
        }
        //【3】获得其要继承的 apk 的文件路径！
        for (File file : mResolvedInheritedFiles) {
            if (mResolvedBaseFile.equals(file)) continue;
            splitPaths.add(file.getAbsolutePath());
        }

        //【4】创建一个 PackageLite 对象！
        final PackageLite pkg = new PackageLite(null, baseApk, null,
                splitPaths.toArray(new String[splitPaths.size()]), null);
        final boolean isForwardLocked =
                (params.installFlags & PackageManager.INSTALL_FORWARD_LOCK) != 0;

        try {
            //【*4.2.3.4.1】调用 PackageHelper 计算空间大小！
            return PackageHelper.calculateInstalledSize(pkg, isForwardLocked, params.abiOverride);
        } catch (IOException e) {
            throw new PackageManagerException(INSTALL_FAILED_INVALID_APK,
                    "Failed to calculate install size", e);
        }
    }
```
我们去 PackageHelper.calculateInstalledSize 方法中去看看：

##### 4.2.3.4.1 calculateInstalledSize
```java
    public static long calculateInstalledSize(PackageLite pkg, boolean isForwardLocked,
            String abiOverride) throws IOException {
        NativeLibraryHelper.Handle handle = null;
        try {
            handle = NativeLibraryHelper.Handle.create(pkg);
            return calculateInstalledSize(pkg, handle, isForwardLocked, abiOverride);
        } finally {
            IoUtils.closeQuietly(handle);
        }
    }

    public static long calculateInstalledSize(PackageLite pkg, NativeLibraryHelper.Handle handle,
            boolean isForwardLocked, String abiOverride) throws IOException {
        long sizeBytes = 0;

        //【1】包括 apk 集合其他的一些资源！
        for (String codePath : pkg.getAllCodePaths()) {
            final File codeFile = new File(codePath);
            sizeBytes += codeFile.length();
            //【2】如果是 forward lock 模式，还要加入一些公共资源！
            if (isForwardLocked) {
                sizeBytes += PackageHelper.extractPublicFiles(codeFile, null);
            }
        }

        //【3】加入 native code 的大小！
        sizeBytes += NativeLibraryHelper.sumNativeBinariesWithOverride(handle, abiOverride);
        //【4】返回最终大小！
        return sizeBytes;
    }
```
这里就分析到这里！


#### 4.2.3.5 computeProgressLocked

```java
    private void computeProgressLocked(boolean forcePublish) {
        mProgress = MathUtils.constrain(mClientProgress * 0.8f, 0f, 0.8f)
                + MathUtils.constrain(mInternalProgress * 0.2f, 0f, 0.2f);

        // Only publish when meaningful change
        if (forcePublish || Math.abs(mProgress - mReportedProgress) >= 0.01) {
            mReportedProgress = mProgress;
            //【3.1.4.1】调用 InternalCallback 回调！
            mCallback.onSessionProgressChanged(this, mProgress);
        }
    }
```

#### 4.2.3.6 extractNativeLibraries

提取本地的 lib 库文件！

```java
    private static void extractNativeLibraries(File packageDir, String abiOverride)
            throws PackageManagerException {
        //【1】libDir 指向 /data/app/vmdl[id].tmp/lib 目录，这里是删除目录下存在的 lib 库文件！
        final File libDir = new File(packageDir, NativeLibraryHelper.LIB_DIR_NAME);
        NativeLibraryHelper.removeNativeBinariesFromDirLI(libDir, true);

        NativeLibraryHelper.Handle handle = null;
        try {
            //【2】创建 lib 子目录，并将应用程序中的 lib 库拷贝到该目录下！
            handle = NativeLibraryHelper.Handle.create(packageDir);
            final int res = NativeLibraryHelper.copyNativeBinariesWithOverride(handle, libDir,
                    abiOverride);
            if (res != PackageManager.INSTALL_SUCCEEDED) {
                throw new PackageManagerException(res,
                        "Failed to extract native libraries, res=" + res);
            }
        } catch (IOException e) {
            throw new PackageManagerException(INSTALL_FAILED_INTERNAL_ERROR,
                    "Failed to extract native libraries", e);
        } finally {
            IoUtils.closeQuietly(handle);
        }
    }
```
对于 lib 的提取，这里涉及到了 NativeLibraryHelper，我们先不过多关注其实现！

# 5 PackageManagerService

## 5.1 PackageManagerService.installStage

接下来，进入 PackageManagerService 阶段。

这里的 IPackageInstallObserver2 observer 是前面创建的本次 localObserver：

```java
   void installStage(String packageName, File stagedDir, String stagedCid,
            IPackageInstallObserver2 observer, PackageInstaller.SessionParams sessionParams,
            String installerPackageName, int installerUid, UserHandle user,
            Certificate[][] certificates) {
        if (DEBUG_EPHEMERAL) {
            if ((sessionParams.installFlags & PackageManager.INSTALL_EPHEMERAL) != 0) {
                Slog.d(TAG, "Ephemeral install of " + packageName);
            }
        }
        //【5.1.1】创建一个 VerificationInfo 对象，用于校验
        final VerificationInfo verificationInfo = new VerificationInfo(
                sessionParams.originatingUri, sessionParams.referrerUri,
                sessionParams.originatingUid, installerUid);
        //【5.1.2】创建了一个 OriginInfo 对象！
        final OriginInfo origin;
        if (stagedDir != null) {
            origin = OriginInfo.fromStagedFile(stagedDir);
        } else {
            origin = OriginInfo.fromStagedContainer(stagedCid);
        }
        //【1】创建了一个 INIT_COPY 消息！
        final Message msg = mHandler.obtainMessage(INIT_COPY);
        //【5.1.3】创建一个 InstallParams 安装参数对象！
        final InstallParams params = new InstallParams(origin, null, observer,
                sessionParams.installFlags, installerPackageName, sessionParams.volumeUuid,
                verificationInfo, user, sessionParams.abiOverride,
                sessionParams.grantedRuntimePermissions, certificates);
        params.setTraceMethod("installStage").setTraceCookie(System.identityHashCode(params));
        msg.obj = params;

        Trace.asyncTraceBegin(TRACE_TAG_PACKAGE_MANAGER, "installStage",
                System.identityHashCode(msg.obj));
        Trace.asyncTraceBegin(TRACE_TAG_PACKAGE_MANAGER, "queueInstall",
                System.identityHashCode(msg.obj));
                
        //【5.2】发送 INIT_COPY 消息！
        mHandler.sendMessage(msg);
    }
```
这里的 mHandler 是在 PackageManagerService 的构造器中创建的：

```java
        mHandlerThread = new ServiceThread(TAG,
                Process.THREAD_PRIORITY_BACKGROUND, true /*allowIo*/);
        mHandlerThread.start();
        mHandler = new PackageHandler(mHandlerThread.getLooper());
```
是一个 PackageHandler 实例，其绑定了一个子线程 ServiceThread！

### 5.1.1 new VerificationInfo

VerificationInfo 类定义在 PackageManagerService 中，用于信息校验：

```java
    static class VerificationInfo {
        /** A constant used to indicate that a uid value is not present. */
        public static final int NO_UID = -1;

        /** URI referencing where the package was downloaded from. */
        final Uri originatingUri;

        /** HTTP referrer URI associated with the originatingURI. */
        final Uri referrer;

        /** UID of the application that the install request originated from. */
        final int originatingUid;

        final int installerUid; // 安装者应用的 uid！

        VerificationInfo(Uri originatingUri, Uri referrer, int originatingUid, int installerUid) {
            this.originatingUri = originatingUri;
            this.referrer = referrer;
            this.originatingUid = originatingUid;
            this.installerUid = installerUid;
        }
    }
```

### 5.1.2 new OriginInfo

这里创建了一个 OriginInfo 实例，封装安装目录相关信息！

```java
    static class OriginInfo {
        final File file; // 安装到内置存储，不为 null
        final String cid; // 安装到内置存储，为 null
        final boolean staged;
        final boolean existing;

        final String resolvedPath;
        final File resolvedFile;

        static OriginInfo fromNothing() {
            return new OriginInfo(null, null, false, false);
        }

        static OriginInfo fromUntrustedFile(File file) {
            return new OriginInfo(file, null, false, false);
        }

        static OriginInfo fromExistingFile(File file) {
            return new OriginInfo(file, null, false, true);
        }

        static OriginInfo fromStagedFile(File file) { //【1】安装到内置存储会调用
            return new OriginInfo(file, null, true, false);
        }

        static OriginInfo fromStagedContainer(String cid) { //【2】安装到外置存储会调用
            return new OriginInfo(null, cid, true, false);
        }

        private OriginInfo(File file, String cid, boolean staged, boolean existing) {
            this.file = file; // 内置时为：/data/app/vmdl[sessionId].tmp
            this.cid = cid;
            this.staged = staged; // 安装到内置存储时为 true。
            this.existing = existing; // 安装到内置存储时为 false。

            if (cid != null) {
                resolvedPath = PackageHelper.getSdDir(cid);
                resolvedFile = new File(resolvedPath);
            } else if (file != null) {
                resolvedPath = file.getAbsolutePath();
                resolvedFile = file;
            } else {
                resolvedPath = null;
                resolvedFile = null;
            }
        }
    }
```
这里我们先关注安装到内置存储中的逻辑！

### 5.1.3 new PMS.InstallParams 

InstallParams  类定义在 PackageManagerService 中，封装了安装参数：

```java
    class InstallParams extends HandlerParams {
        final OriginInfo origin;
        final MoveInfo move; // move package 才会传入，安装时为 null！
        final IPackageInstallObserver2 observer; // 本地传观察者；
        int installFlags;
        final String installerPackageName;
        final String volumeUuid;
        private InstallArgs mArgs;
        private int mRet;
        final String packageAbiOverride;
        final String[] grantedRuntimePermissions; // 安装时授予的运行时权限列表；
        final VerificationInfo verificationInfo;
        final Certificate[][] certificates;

        InstallParams(OriginInfo origin, MoveInfo move, IPackageInstallObserver2 observer,
                int installFlags, String installerPackageName, String volumeUuid,
                VerificationInfo verificationInfo, UserHandle user, String packageAbiOverride,
                String[] grantedPermissions, Certificate[][] certificates) {
            super(user);
            this.origin = origin;
            this.move = move;
            this.observer = observer;
            this.installFlags = installFlags;
            this.installerPackageName = installerPackageName;
            this.volumeUuid = volumeUuid;
            this.verificationInfo = verificationInfo;
            this.packageAbiOverride = packageAbiOverride;
            this.grantedRuntimePermissions = grantedPermissions;
            this.certificates = certificates;
        }
        ... ... ...   
    }
```

InstallParams 继承了 HandlerParams！

这里涉及到一个 MoveInfo move，在 movePackageInternal 也就是移动 package 时才会调用，这里是安装，我们先不关注！

## 5.2 PackageHandler.doHandleMessage[INIT_COPY]

PackageHandler 会在子线程中处理 INIT_COPY 消息：

```java
        case INIT_COPY: {
            //【1】取出 InstallParams！
            HandlerParams params = (HandlerParams) msg.obj;
            // mPendingInstalls 中会保存所有正在等待的安装！
            int idx = mPendingInstalls.size();
            if (DEBUG_INSTALL) Slog.i(TAG, "init_copy idx=" + idx + ": " + params);

            //【2】mBound 用来判断是否已经绑定到了 DefaultContainerService，该服务用于安装！
            if (!mBound) {
                Trace.asyncTraceBegin(TRACE_TAG_PACKAGE_MANAGER, "bindingMCS",
                        System.identityHashCode(mHandler));

                //【5.2.1】尝试去 bind 服务，bind 成功后，mBound 置为 true！
                if (!connectToService()) {
                    Slog.e(TAG, "Failed to bind to media container service");
                    //【5.2.2】如果不能 bind 成功，那就触发 HandlerParams 的 serviceError 方法！
                    params.serviceError();
                    Trace.asyncTraceEnd(TRACE_TAG_PACKAGE_MANAGER, "bindingMCS",
                            System.identityHashCode(mHandler));
                    if (params.traceMethod != null) {
                        Trace.asyncTraceEnd(TRACE_TAG_PACKAGE_MANAGER, params.traceMethod,
                                params.traceCookie);
                    }
                    return;
                } else {
                    //【3】将本次的安装参数添加到等待集合中！！
                    mPendingInstalls.add(idx, params);
                }
            } else {
                //【4】如果之前已经 bind 了，那就直接将安装参数添加到等待集合中！！
                mPendingInstalls.add(idx, params);
                //【5.3】如果是第一次添加，发送 MCS_BOUND 消息！
                if (idx == 0) {
                    mHandler.sendEmptyMessage(MCS_BOUND);
                }
            }
            break;
        }
```
这里逻辑很清晰了，我们继续看！

### 5.2.1 PackageHandler.connectToService

```java
        private boolean connectToService() {
            if (DEBUG_SD_INSTALL) Log.i(TAG, "Trying to bind to" +
                    " DefaultContainerService");
            Intent service = new Intent().setComponent(DEFAULT_CONTAINER_COMPONENT);
            Process.setThreadPriority(Process.THREAD_PRIORITY_DEFAULT);
            //【1】开始 bind 服务，传入了 mDefContainerConn 对象！
            if (mContext.bindServiceAsUser(service, mDefContainerConn,
                    Context.BIND_AUTO_CREATE, UserHandle.SYSTEM)) {
                Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                mBound = true;
                return true;
            }
            Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
            return false;
        }
```
这里 bind 的服务是 DefaultContainerService！
```java
    static final ComponentName DEFAULT_CONTAINER_COMPONENT = new ComponentName(
            DEFAULT_CONTAINER_PACKAGE,
            "com.android.defcontainer.DefaultContainerService");
```

PackageManagerService 内部持有一个 DefaultContainerConnection 实例！

```java
    final private DefaultContainerConnection mDefContainerConn =
            new DefaultContainerConnection();

    class DefaultContainerConnection implements ServiceConnection {
        public void onServiceConnected(ComponentName name, IBinder service) {
            if (DEBUG_SD_INSTALL) Log.i(TAG, "onServiceConnected");
            //【1】获得 DefaultContainerConnection 代理对象！
            IMediaContainerService imcs =
                IMediaContainerService.Stub.asInterface(service);
    
            //【5.3】发送 MCS_BOUND 消息！
            mHandler.sendMessage(mHandler.obtainMessage(MCS_BOUND, imcs));
        }

        public void onServiceDisconnected(ComponentName name) {
            if (DEBUG_SD_INSTALL) Log.i(TAG, "onServiceDisconnected");
        }
    }
```

最后会发送 MCS_BOUND 消息给 PackageHandler 对象！


### 5.2.2 InstallParams.serviceError

serviceError 方法是从 HandlerParams 中继承到的：

```java
    final void serviceError() {
        if (DEBUG_INSTALL) Slog.i(TAG, "serviceError");
        //【5.4.2】安装异常，保存结果
        handleServiceError();
        //【5.4.4】处理结果！
        handleReturnCode();
    }
```
handleServiceError 方法和 handleReturnCode 方法的最终实现也是在 InstallParams 中！


## 5.3 PackageHandler.doHandleMessage[MCS_BOUND] - 绑定服务

继续来看下 PackageHandler 对 MCS_BOUND 的处理：

```java
        case MCS_BOUND: {
            if (DEBUG_INSTALL) Slog.i(TAG, "mcs_bound");
            //【1】这里的 msg.obj 是前面 bind 获得的代理对象，将其保存到 mContainerService 中！ 
            if (msg.obj != null) {
                mContainerService = (IMediaContainerService) msg.obj;
                Trace.asyncTraceEnd(TRACE_TAG_PACKAGE_MANAGER, "bindingMCS",
                        System.identityHashCode(mHandler));
            }
            //【2】异常处理，如果 mContainerService 为 null，且 mBound 为 ture，那么这种情况是异常！
            if (mContainerService == null) {
                if (!mBound) {
                    Slog.e(TAG, "Cannot bind to media container service");
                    
                    //【2.1】遍历所有的 HandlerParams 安装参数，回调 serviceError！
                    for (HandlerParams params : mPendingInstalls) {
                        params.serviceError();
                        Trace.asyncTraceEnd(TRACE_TAG_PACKAGE_MANAGER, "queueInstall",
                                System.identityHashCode(params));
                        if (params.traceMethod != null) {
                            Trace.asyncTraceEnd(TRACE_TAG_PACKAGE_MANAGER,
                                    params.traceMethod, params.traceCookie);
                        }
                        return;
                    }
                    //【2.2】清空 mPendingInstalls 集合！
                    mPendingInstalls.clear();
                } else {
                    Slog.w(TAG, "Waiting to connect to media container service");
                }
            } else if (mPendingInstalls.size() > 0) {
                //【3】正常情况下，bind 是有效的，那么会进入这里！
                HandlerParams params = mPendingInstalls.get(0);
                if (params != null) {
                    Trace.asyncTraceEnd(TRACE_TAG_PACKAGE_MANAGER, "queueInstall",
                            System.identityHashCode(params));
                    Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "startCopy");
                    //【5.4】调用了 HandlerParams 的 startCopy 方法！
                    if (params.startCopy()) {
                        // We are done...  look for more work or to
                        // go idle.
                        if (DEBUG_SD_INSTALL) Log.i(TAG,
                                "Checking for more work or unbind...");
                        //【3.1】本次安装完成，删除本次处理的安装参数！
                        if (mPendingInstalls.size() > 0) {
                            mPendingInstalls.remove(0);
                        }
                        if (mPendingInstalls.size() == 0) {
                            //【3.2】如果所有的安装参数都处理玩了，unbind 服务！
                            // 这里的 unbind 延迟了 10s
                            if (mBound) {
                                if (DEBUG_SD_INSTALL) Log.i(TAG,
                                        "Posting delayed MCS_UNBIND");
                                removeMessages(MCS_UNBIND);
                                Message ubmsg = obtainMessage(MCS_UNBIND);
                                sendMessageDelayed(ubmsg, 10000);
                            }
                        } else {
                            //【3.3】在安装队列中有其他的安装项，我们发送 MCS_BOUND 消息继续处理！
                            if (DEBUG_SD_INSTALL) Log.i(TAG,
                                    "Posting MCS_BOUND for next work");
                            mHandler.sendEmptyMessage(MCS_BOUND);
                        }
                    }
                    Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
                }
            } else {
                // Should never happen ideally.
                Slog.w(TAG, "Empty queue");
            }
            break;
        }
```

## 5.4 InstallParams.startCopy

InstallParams 继承了 HandlerParams：

```java
        final boolean startCopy() {
            boolean res;
            try {
                if (DEBUG_INSTALL) Slog.i(TAG, "startCopy " + mUser + ": " + this);
                //【1】如果重试次数大于 4 ，那么就放弃本次安装！
                if (++mRetries > MAX_RETRIES) {
                    Slog.w(TAG, "Failed to invoke remote methods on default container service. Giving up");
                    //【5.4.1】发送 MCS_GIVE_UP 消息！
                    mHandler.sendEmptyMessage(MCS_GIVE_UP);
                    //【5.4.2】处理失败结果
                    handleServiceError();
                    return false;

                } else {
                    //【5.5】继续安装！
                    handleStartCopy();
                    res = true;
                }
            } catch (RemoteException e) {
                if (DEBUG_INSTALL) Slog.i(TAG, "Posting install MCS_RECONNECT");
                //【5.4.3】发送 MCS_RECONNECT 消息！
                mHandler.sendEmptyMessage(MCS_RECONNECT);
                res = false;
            }

            //【5.4.4】处理返回码！
            handleReturnCode();
            return res;
        }
```
这里定义了安装尝试次数：
```java
private static final int MAX_RETRIES = 4;
```

### 5.4.1 PackageHandler.doHandleMessage[MCS_GIVE_UP] - 放弃安装

重试次数大于 4 ，那么就放弃本次安装！

```java
    case MCS_GIVE_UP: {
        if (DEBUG_INSTALL) Slog.i(TAG, "mcs_giveup too many retries");
        //【1】移除这个安装参数。
        HandlerParams params = mPendingInstalls.remove(0);
        Trace.asyncTraceEnd(TRACE_TAG_PACKAGE_MANAGER, "queueInstall",
                System.identityHashCode(params));
        break;
    }
```


### 5.4.2 InstallParams.handleServiceError

处理本次安装失败的结果：

```java
        @Override
        void handleServiceError() {
            //【5.2.3】创建安装参数 InstallArgs！
            mArgs = createInstallArgs(this);
            mRet = PackageManager.INSTALL_FAILED_INTERNAL_ERROR;
        }
```
这里的 createInstallArgs 方法我们后面再分析！

### 5.4.3 PackageHandler.doHandleMessage[MCS_RECONNECT] - 重连服务

MCS_RECONNECT 消息会尝试重连服务！

```java
    case MCS_RECONNECT: {
        if (DEBUG_INSTALL) Slog.i(TAG, "mcs_reconnect");
        if (mPendingInstalls.size() > 0) {
            //【1】如果 mBound 为 true，先尝试断开连接！
            if (mBound) {
                disconnectService();
            }
            //【2】开始重新连接服务！
            if (!connectToService()) {
                Slog.e(TAG, "Failed to bind to media container service");
                for (HandlerParams params : mPendingInstalls) {
                    //【5.2.2】返回安装异常！
                    params.serviceError();
                    Trace.asyncTraceEnd(TRACE_TAG_PACKAGE_MANAGER, "queueInstall",
                            System.identityHashCode(params));
                }
                //【3】清空 mPendingInstalls 集合！
                mPendingInstalls.clear();
            }
        }
        break;
    }
```
流程很简单，不多说了！

### 5.4.4 InstallParams.handleReturnCode

当 mArgs 为 null 的时候，此时该 package 正在被校验，所以需要等到校验成功后才能安装，所以不会进入这里！

```java
        @Override
        void handleReturnCode() {
            //【1】如果 mArgs 这里为 null，可能是不需要校验！
            if (mArgs != null) {
                //【5.7】继续安装！
                processPendingInstall(mArgs, mRet);
            }
        }
```
我们继续看！

## 5.5 InstallParams.handleStartCopy

```java
    public void handleStartCopy() throws RemoteException {
        //【1】保存安装结果，默认为成功！
        int ret = PackageManager.INSTALL_SUCCEEDED;
    
        //【2】我们知道 origin 中存储了安装目录相关的信息！
        if (origin.staged) {
            if (origin.file != null) {
                //【2.1】安装到内置存储中！
                installFlags |= PackageManager.INSTALL_INTERNAL;
                installFlags &= ~PackageManager.INSTALL_EXTERNAL;
            } else if (origin.cid != null) {
                //【2.2】这里是安装到外置存储中的逻辑！
                installFlags |= PackageManager.INSTALL_EXTERNAL;
                installFlags &= ~PackageManager.INSTALL_INTERNAL;
            } else {
                throw new IllegalStateException("Invalid stage location");
            }
        }
        //【3】判断安装位置！
        final boolean onSd = (installFlags & PackageManager.INSTALL_EXTERNAL) != 0;
        final boolean onInt = (installFlags & PackageManager.INSTALL_INTERNAL) != 0;
        final boolean ephemeral = (installFlags & PackageManager.INSTALL_EPHEMERAL) != 0;
        PackageInfoLite pkgLite = null;

        //【4】判断安装位置是否满足条件！
        if (onInt && onSd) {
            //【4.1】不能同时设置安装在 sd 和内置中！
            Slog.w(TAG, "Conflicting flags specified for installing on both internal and external");
            ret = PackageManager.INSTALL_FAILED_INVALID_INSTALL_LOCATION;
        } else if (onSd && ephemeral) {
            Slog.w(TAG,  "Conflicting flags specified for installing ephemeral on external");
            //【4.2】不能设置短暂安装在 sd 中！
            ret = PackageManager.INSTALL_FAILED_INVALID_INSTALL_LOCATION;
        } else {
            //【5.5.1】利用 ContainerService 获取 PackageInfoLite，同时判断空间是否合适！
            pkgLite = mContainerService.getMinimalPackageInfo(origin.resolvedPath, installFlags,
                    packageAbiOverride);
    
            if (DEBUG_EPHEMERAL && ephemeral) {
                Slog.v(TAG, "pkgLite for install: " + pkgLite);
            }
    
            //【4.3】空间不足，尝试释放空尽！
            if (!origin.staged && pkgLite.recommendedInstallLocation
                    == PackageHelper.RECOMMEND_FAILED_INSUFFICIENT_STORAGE) {
                // TODO: focus freeing disk space on the target device
                final StorageManager storage = StorageManager.from(mContext);
                final long lowThreshold = storage.getStorageLowBytes(
                        Environment.getDataDirectory());
    
                final long sizeBytes = mContainerService.calculateInstalledSize(
                        origin.resolvedPath, isForwardLocked(), packageAbiOverride);
    
                try {
                    //【4.4】通过 installd 释放存储！
                    mInstaller.freeCache(null, sizeBytes + lowThreshold);
                    //【5.5.1】再次利用 ContainerService 获取 PackageInfoLite，同时判断空间是否合适！
                    pkgLite = mContainerService.getMinimalPackageInfo(origin.resolvedPath,
                            installFlags, packageAbiOverride);
                } catch (InstallerException e) {
                    Slog.w(TAG, "Failed to free cache", e);
                }
    
                //【4.5】依然无法安装，设置结果为 RECOMMEND_FAILED_INSUFFICIENT_STORAGE！
                if (pkgLite.recommendedInstallLocation
                        == PackageHelper.RECOMMEND_FAILED_INVALID_URI) {
                    pkgLite.recommendedInstallLocation
                        = PackageHelper.RECOMMEND_FAILED_INSUFFICIENT_STORAGE;
                }
            }
        }
        
        //【5】处理前面的空间判断后的结果！
        if (ret == PackageManager.INSTALL_SUCCEEDED) {
            int loc = pkgLite.recommendedInstallLocation;
            if (loc == PackageHelper.RECOMMEND_FAILED_INVALID_LOCATION) {
                ret = PackageManager.INSTALL_FAILED_INVALID_INSTALL_LOCATION;

            } else if (loc == PackageHelper.RECOMMEND_FAILED_ALREADY_EXISTS) {
                ret = PackageManager.INSTALL_FAILED_ALREADY_EXISTS;
    
            } else if (loc == PackageHelper.RECOMMEND_FAILED_INSUFFICIENT_STORAGE) {
                ret = PackageManager.INSTALL_FAILED_INSUFFICIENT_STORAGE;
    
            } else if (loc == PackageHelper.RECOMMEND_FAILED_INVALID_APK) {
                ret = PackageManager.INSTALL_FAILED_INVALID_APK;

            } else if (loc == PackageHelper.RECOMMEND_FAILED_INVALID_URI) {
                ret = PackageManager.INSTALL_FAILED_INVALID_URI;
    
            } else if (loc == PackageHelper.RECOMMEND_MEDIA_UNAVAILABLE) {
                ret = PackageManager.INSTALL_FAILED_MEDIA_UNAVAILABLE;

            } else {
                //【5.5.2】调用 installLocationPolicy 方法，针对于降级安装和替换安装做处理！
                loc = installLocationPolicy(pkgLite);
                if (loc == PackageHelper.RECOMMEND_FAILED_VERSION_DOWNGRADE) {
                    // 处理无法降级的结果！
                    ret = PackageManager.INSTALL_FAILED_VERSION_DOWNGRADE;

                } else if (!onSd && !onInt) {
                    // 如果 flags 没有指定内置还是外置，那么由 installLocationPolicy 的返回值指定！
                    if (loc == PackageHelper.RECOMMEND_INSTALL_EXTERNAL) {
                        installFlags |= PackageManager.INSTALL_EXTERNAL; // 外置
                        installFlags &= ~PackageManager.INSTALL_INTERNAL;

                    } else if (loc == PackageHelper.RECOMMEND_INSTALL_EPHEMERAL) {
                        if (DEBUG_EPHEMERAL) {
                            Slog.v(TAG, "...setting INSTALL_EPHEMERAL install flag");
                        }
                        installFlags |= PackageManager.INSTALL_EPHEMERAL; // 内置并且是短暂安装！
                        installFlags &= ~(PackageManager.INSTALL_EXTERNAL
                                |PackageManager.INSTALL_INTERNAL);
                    } else {
                        installFlags |= PackageManager.INSTALL_INTERNAL; // 内置
                        installFlags &= ~PackageManager.INSTALL_EXTERNAL;
                    }
                }
            }
        }
        //【5.5.3】创建一个 InstallArgs 对象，和 InstallParams 相互引用！
        final InstallArgs args = createInstallArgs(this);
        mArgs = args;
    
        if (ret == PackageManager.INSTALL_SUCCEEDED) {
            // TODO: http://b/22976637
            //【6】如果该应用是安装给 all users 的，那么需要校验应用！
            UserHandle verifierUser = getUser();
            if (verifierUser == UserHandle.ALL) {
                verifierUser = UserHandle.SYSTEM;
            }
    
            //【7】尝试找到系统中安装的 pacakge 校验器，如果可以找到，那就要做校验啦！
            //【7.1】获得校验器的 uid！
            final int requiredUid = mRequiredVerifierPackage == null ? -1
                    : getPackageUid(mRequiredVerifierPackage, MATCH_DEBUG_TRIAGED_MISSING,
                            verifierUser.getIdentifier());
            //【7.1】如果是安装到内置存储（existing 为 true），并且系统中有校验器，并且系统打开了校验功能，那么就尝试校验！
            //【5.5.4】通过 isVerificationEnabled 判断是否打开校验功能！
            if (!origin.existing && requiredUid != -1
                    && isVerificationEnabled(verifierUser.getIdentifier(), installFlags)) {

                //【7.1.1】准备发送校验广播
                final Intent verification = new Intent(
                        Intent.ACTION_PACKAGE_NEEDS_VERIFICATION);
                verification.addFlags(Intent.FLAG_RECEIVER_FOREGROUND);
                verification.setDataAndType(Uri.fromFile(new File(origin.resolvedPath)),
                        PACKAGE_MIME_TYPE); // 设置 DataAndType 属性，包含了 apk 路径对应的 uri
                verification.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION); // 增加了临时的权限授予！
    
                //【7.1.2】查询校验器！
                final List<ResolveInfo> receivers = queryIntentReceiversInternal(verification,
                        PACKAGE_MIME_TYPE, 0, verifierUser.getIdentifier());
    
                if (DEBUG_VERIFY) {
                    Slog.d(TAG, "Found " + receivers.size() + " verifiers for intent "
                            + verification.toString() + " with " + pkgLite.verifiers.length
                            + " optional verifiers");
                }
    
                final int verificationId = mPendingVerificationToken++; // 计算本次校验的唯一标识 token
                verification.putExtra(PackageManager.EXTRA_VERIFICATION_ID, verificationId);
                verification.putExtra(PackageManager.EXTRA_VERIFICATION_INSTALLER_PACKAGE,
                        installerPackageName); // 安装器（packageInstaller）
                verification.putExtra(PackageManager.EXTRA_VERIFICATION_INSTALL_FLAGS,
                        installFlags); // 本次安装的 installFlags
                verification.putExtra(PackageManager.EXTRA_VERIFICATION_PACKAGE_NAME,
                        pkgLite.packageName); // 要校验的应用
                verification.putExtra(PackageManager.EXTRA_VERIFICATION_VERSION_CODE,
                        pkgLite.versionCode); // 版本号
              
                if (verificationInfo != null) { // 如果指定了校验信息，将其写入 intent！
                    if (verificationInfo.originatingUri != null) {
                        verification.putExtra(Intent.EXTRA_ORIGINATING_URI,
                                verificationInfo.originatingUri);
                    }
                    if (verificationInfo.referrer != null) {
                        verification.putExtra(Intent.EXTRA_REFERRER,
                                verificationInfo.referrer);
                    }
                    if (verificationInfo.originatingUid >= 0) {
                        verification.putExtra(Intent.EXTRA_ORIGINATING_UID,
                                verificationInfo.originatingUid);
                    }
                    if (verificationInfo.installerUid >= 0) {
                        verification.putExtra(PackageManager.EXTRA_VERIFICATION_INSTALLER_UID,
                                verificationInfo.installerUid);
                    }
                }

                //【5.5.5】创建一个 PackageVerificationState 对象，用于保存应用校验状态，同时将创建的
                // installArgs 作为参数传入，并加入 mPendingVerification 集合中！！
                final PackageVerificationState verificationState = new PackageVerificationState(
                        requiredUid, args);
                mPendingVerification.append(verificationId, verificationState);
    
                //【7.1.3】如果应用指定了校验者，那么我们要尝试先用指定的校验器校验！
                final List<ComponentName> sufficientVerifiers = matchVerifiers(pkgLite,
                        receivers, verificationState);
                if (sufficientVerifiers != null) {
                    final int N = sufficientVerifiers.size();
                    if (N == 0) {
                        Slog.i(TAG, "Additional verifiers required, but none installed.");
                        ret = PackageManager.INSTALL_FAILED_VERIFICATION_FAILURE;
                    } else {
                        for (int i = 0; i < N; i++) {
                            final ComponentName verifierComponent = sufficientVerifiers.get(i);
                            // 使用前面创建的 verification 意图，再创建一个 intent！
                            // 指定组件为应用指定的校验器，并发送广播！
                            final Intent sufficientIntent = new Intent(verification);
                            sufficientIntent.setComponent(verifierComponent);
                            mContext.sendBroadcastAsUser(sufficientIntent, verifierUser);
                        }
                    }
                }
                //【7.1.4】找到和 mRequiredVerifierPackage（系统指定的默认校验器）匹配的接收者！
                final ComponentName requiredVerifierComponent = matchComponentForVerifier(
                        mRequiredVerifierPackage, receivers);

                if (ret == PackageManager.INSTALL_SUCCEEDED
                        && mRequiredVerifierPackage != null) {
                    Trace.asyncTraceBegin(
                            TRACE_TAG_PACKAGE_MANAGER, "verification", verificationId);
                    //【7.1.5】指定目标应用为 mRequiredVerifierPackage 的内部组件！
                    // 发送 verification 意图！
                    verification.setComponent(requiredVerifierComponent);
                    mContext.sendOrderedBroadcastAsUser(verification, verifierUser,
                            android.Manifest.permission.PACKAGE_VERIFICATION_AGENT,
                            new BroadcastReceiver() {
                                @Override
                                public void onReceive(Context context, Intent intent) {
                                    //【7.1.6】当 mRequiredVerifierPackage 接收到广播后，会回调
                                    // 该 BroadcastReceiver 的 onReceive 方法！
                                    //【5.6】此时校验完成，这里会发送 CHECK_PENDING_VERIFICATION 给 PackageHandler
                                    // 携带校验标识符！
                                    final Message msg = mHandler
                                            .obtainMessage(CHECK_PENDING_VERIFICATION);
                                    msg.arg1 = verificationId;
                                    mHandler.sendMessageDelayed(msg, getVerificationTimeout());
                                }
                            }, null, 0, null, null);
    
                    /*
                     * We don't want the copy to proceed until verification
                     * succeeds, so null out this field.
                     */
                    mArgs = null;
                }
            } else {
                //【5.7】没有合适的校验器，那么会调用 InstallArgs 的 copyApk 方法！
                ret = args.copyApk(mContainerService, true);
            }
        }
    
        mRet = ret;
    }
```
直接可以看到，最终


### 5.5.1 DefaultContainerService.getMinimalPackageInfo

getMinimalPackageInfo 方法会解析 apk，返回 apk 的解析信息，同时判断空间状态！

```java
        @Override
        public PackageInfoLite getMinimalPackageInfo(String packagePath, int flags,
                String abiOverride) {
            final Context context = DefaultContainerService.this;
            final boolean isForwardLocked = (flags & PackageManager.INSTALL_FORWARD_LOCK) != 0;
            //【5.5.1.1】创建了一个 PackageInfoLite 实例！
            PackageInfoLite ret = new PackageInfoLite();
            if (packagePath == null) {
                Slog.i(TAG, "Invalid package file " + packagePath);
                ret.recommendedInstallLocation = PackageHelper.RECOMMEND_FAILED_INVALID_APK; // 该 apk 无效
                return ret;
            }

            final File packageFile = new File(packagePath);
            final PackageParser.PackageLite pkg;
            final long sizeBytes;
            try {
                //【2】解析 apk，获得其 PackageLite 对象，并计算安装所需空间！
                pkg = PackageParser.parsePackageLite(packageFile, 0);
                sizeBytes = PackageHelper.calculateInstalledSize(pkg, isForwardLocked, abiOverride);
            } catch (PackageParserException | IOException e) {
                Slog.w(TAG, "Failed to parse package at " + packagePath + ": " + e);

                if (!packageFile.exists()) {
                    ret.recommendedInstallLocation = PackageHelper.RECOMMEND_FAILED_INVALID_URI; // 该 apk 无效
                } else {
                    ret.recommendedInstallLocation = PackageHelper.RECOMMEND_FAILED_INVALID_APK; // 该 apk 无效
                }

                return ret;
            }
            //【3】将解析到的参数保存到 PackageInfoLite 中！
            ret.packageName = pkg.packageName;
            ret.splitNames = pkg.splitNames;
            ret.versionCode = pkg.versionCode;
            ret.baseRevisionCode = pkg.baseRevisionCode;
            ret.splitRevisionCodes = pkg.splitRevisionCodes;
            ret.installLocation = pkg.installLocation;
            ret.verifiers = pkg.verifiers;
            
            //【5.5.1.2】解析安装位置状态信息！
            ret.recommendedInstallLocation = PackageHelper.resolveInstallLocation(context,
                    pkg.packageName, pkg.installLocation, sizeBytes, flags);
            ret.multiArch = pkg.multiArch;
            //【4】返回该 PackageInfoLite 实例！
            return ret;
        }
```

#### 5.5.1.1 new PackageInfoLite

PackageInfoLite 用来保存解析到的 apk 的一些信息！

```java
public class PackageInfoLite implements Parcelable {
    public String packageName; // 应用包名；
    public String[] splitNames; // split apk 的名字
    public int versionCode; // 版本号
    public int baseRevisionCode;
    public int[] splitRevisionCodes;

    /**
     * The android:multiArch flag from the package manifest. If set,
     * we will extract all native libraries for the given app, not just those
     * from the preferred ABI.
     */
    public boolean multiArch; 
    public int recommendedInstallLocation;
    public int installLocation;

    public VerifierInfo[] verifiers;

    public PackageInfoLite() {
    }
    ... ... ...
}
```
这里的 recommendedInstallLocation 可以取下面四个值：

```java
PackageHelper.RECOMMEND_INSTALL_INTERNAL // 内置
PackageHelper.RECOMMEND_INSTALL_EXTERNAL // 外置
PackageHelper.RECOMMEND_FAILED_INSUFFICIENT_STORAGE // 存储异常
PackageHelper.RECOMMEND_FAILED_INVALID_APK // apk解析异常
```


#### 5.5.1.2 PackageHelper.resolveInstallLocation

resolveInstallLocation 方法用于计算一个合适的安装位置给 apk！
```java
    public static int resolveInstallLocation(Context context, String packageName,
            int installLocation, long sizeBytes, int installFlags) {
        //【1】如果该应用已经安装了，那么我们获得上一次安装后的信息！！
        ApplicationInfo existingInfo = null;
        try {
            existingInfo = context.getPackageManager().getApplicationInfo(packageName,
                    PackageManager.GET_UNINSTALLED_PACKAGES);
        } catch (NameNotFoundException ignored) {
        }

        final int prefer;
        final boolean checkBoth;
        boolean ephemeral = false;

        //【2】根据安装参数 installFlags，来选择合适的安装位置，按照优先级依次解析！
        if ((installFlags & PackageManager.INSTALL_EPHEMERAL) != 0) {
            //【2.1】如果指定了 PackageManager.INSTALL_EPHEMERAL，优先内置！
            prefer = RECOMMEND_INSTALL_INTERNAL;
            ephemeral = true; // 表示短暂安装！
            checkBoth = false;
            
        } else if ((installFlags & PackageManager.INSTALL_INTERNAL) != 0) {
            //【2.2】如果指定了 PackageManager.INSTALL_INTERNAL，优先内置！
            prefer = RECOMMEND_INSTALL_INTERNAL;
            checkBoth = false;
            
        } else if ((installFlags & PackageManager.INSTALL_EXTERNAL) != 0) {
            //【2.3】如果指定了 PackageManager.INSTALL_EXTERNAL，优先外置！
            prefer = RECOMMEND_INSTALL_EXTERNAL;
            checkBoth = false;
            
        } else if (installLocation == PackageInfo.INSTALL_LOCATION_INTERNAL_ONLY) {
            //【2.4】如果指定了 PackageManager.INSTALL_LOCATION_INTERNAL_ONLY，优先内置！
            prefer = RECOMMEND_INSTALL_INTERNAL;
            checkBoth = false;
            
        } else if (installLocation == PackageInfo.INSTALL_LOCATION_PREFER_EXTERNAL) {
            //【2.5】如果指定了 PackageManager.INSTALL_LOCATION_PREFER_EXTERNAL，优先外置！
            prefer = RECOMMEND_INSTALL_EXTERNAL;
            checkBoth = true;
            
        } else if (installLocation == PackageInfo.INSTALL_LOCATION_AUTO) {
            //【2.6】如果指定了 PackageManager.INSTALL_LOCATION_AUTO，那么我们自动调整！
            if (existingInfo != null) {
                //【2.6.1】如果之前已经安装过该应用，那么就和之前安装的位置保持一致！
                if ((existingInfo.flags & ApplicationInfo.FLAG_EXTERNAL_STORAGE) != 0) {
                    prefer = RECOMMEND_INSTALL_EXTERNAL;
                } else {
                    prefer = RECOMMEND_INSTALL_INTERNAL;
                }
            } else {
                //【2.6.2】否则，自动安装到内置！
                prefer = RECOMMEND_INSTALL_INTERNAL;
            }
            checkBoth = true;
            
        } else {
            //【2.7】其他情况，默认是内置！
            prefer = RECOMMEND_INSTALL_INTERNAL;
            checkBoth = false;
        }

        //【3】再次校验内置和外置是否合适！
        boolean fitsOnInternal = false;
        if (checkBoth || prefer == RECOMMEND_INSTALL_INTERNAL) {
            fitsOnInternal = fitsOnInternal(context, sizeBytes);
        }

        boolean fitsOnExternal = false;
        if (checkBoth || prefer == RECOMMEND_INSTALL_EXTERNAL) {
            fitsOnExternal = fitsOnExternal(context, sizeBytes);
        }
        //【4】最后，返回合适的安装位置！
        if (prefer == RECOMMEND_INSTALL_INTERNAL) {
            //【4.1】如果优先安装到内置，且内置存储是合适的，根据是否是 ephemeral 返回不同的值！
            if (fitsOnInternal) {
                return (ephemeral)
                        ? PackageHelper.RECOMMEND_INSTALL_EPHEMERAL
                        : PackageHelper.RECOMMEND_INSTALL_INTERNAL;
            }
        } else if (prefer == RECOMMEND_INSTALL_EXTERNAL) {
            //【4.2】如果优先安装到外置，且外置存储是合适的，返回结果！
            if (fitsOnExternal) {
                return PackageHelper.RECOMMEND_INSTALL_EXTERNAL;
            }
        }
        //【4.3】其他情况！
        if (checkBoth) {
            if (fitsOnInternal) {
                return PackageHelper.RECOMMEND_INSTALL_INTERNAL;
            } else if (fitsOnExternal) {
                return PackageHelper.RECOMMEND_INSTALL_EXTERNAL;
            }
        }
        //【5】异常情况，返回 PackageHelper.RECOMMEND_FAILED_INSUFFICIENT_STORAGE！
        return PackageHelper.RECOMMEND_FAILED_INSUFFICIENT_STORAGE;
    }
```
整个过程很简单，不多说了！

### 5.5.2 PackageManagerS.installLocationPolicy

PackageInfoLite pkgLite 保存了本次要安装的应用的信息！

installLocationPolicy 方法会对降级安装和替换安装做一个校验和判断！

```java
        private int installLocationPolicy(PackageInfoLite pkgLite) {
            String packageName = pkgLite.packageName;
            int installLocation = pkgLite.installLocation;
            boolean onSd = (installFlags & PackageManager.INSTALL_EXTERNAL) != 0;
            synchronized (mPackages) {
                //【1】如果该应用已经安装过的话，那就获得上次安装后的解析信息！
                PackageParser.Package installedPkg = mPackages.get(packageName);
                // 下面这段代码主要是处理卸载但是保留了数据的情况，比如 adb uninstall -k！
                PackageParser.Package dataOwnerPkg = installedPkg;
                if (dataOwnerPkg  == null) {
                    PackageSetting ps = mSettings.mPackages.get(packageName);
                    if (ps != null) {
                        dataOwnerPkg = ps.pkg;
                    }
                }
                //【2】如果 dataOwnerPkg 不为 nulkl，说明之前已经安装了！
                if (dataOwnerPkg != null) {
                    //【2.1】如果安装标志位设置了 INSTALL_ALLOW_DOWNGRADE，表示允许降级安装！
                    final boolean downgradeRequested =
                            (installFlags & PackageManager.INSTALL_ALLOW_DOWNGRADE) != 0;
                    //【2.2】判断应用是否允许 debug！
                    final boolean packageDebuggable =
                                (dataOwnerPkg.applicationInfo.flags
                                        & ApplicationInfo.FLAG_DEBUGGABLE) != 0;
                    //【2.3】如果要能够降级安装，必须满足 2 个条件：1、安装标志位设置了 allow downgrade
                    // 2、系统允许 debug 或者该应用可以 debug！
                    final boolean downgradePermitted =
                            (downgradeRequested) && ((Build.IS_DEBUGGABLE) || (packageDebuggable));
                    if (!downgradePermitted) {
                        try {
                            //【5.5.2.1】如果，安装不允许降级，那就需要做检查！
                            checkDowngrade(dataOwnerPkg, pkgLite);
                        } catch (PackageManagerException e) {
                            Slog.w(TAG, "Downgrade detected: " + e.getMessage());
                            return PackageHelper.RECOMMEND_FAILED_VERSION_DOWNGRADE;
                        }
                    }
                }
                //【3】接着处理覆盖安装的情况，即相同包名的应用已经存在！
                if (installedPkg != null) {
                    //【3.1】覆盖安装的情况，必须携带 PackageManager.INSTALL_REPLACE_EXISTING 安装标志位！
                    if ((installFlags & PackageManager.INSTALL_REPLACE_EXISTING) != 0) {
                        if ((installedPkg.applicationInfo.flags & ApplicationInfo.FLAG_SYSTEM) != 0) {
                            //【3.1.1】对于系统应用，不能覆盖安装到 sd 卡上！
                            if (onSd) {
                                Slog.w(TAG, "Cannot install update to system app on sdcard");
                                return PackageHelper.RECOMMEND_FAILED_INVALID_LOCATION;
                            }
                            return PackageHelper.RECOMMEND_INSTALL_INTERNAL;
                        } else {
                            //【3.1.2】对于非系统应用，如果是安装到 sdcard，直接返回对应值！
                            if (onSd) {
                                return PackageHelper.RECOMMEND_INSTALL_EXTERNAL;
                            }
                            //【3.1.2】对于非系统应用，如果只安装到内置，直接返回对应值！
                            if (installLocation == PackageInfo.INSTALL_LOCATION_INTERNAL_ONLY) {
                                return PackageHelper.RECOMMEND_INSTALL_INTERNAL;
                                
                            } else if (installLocation == PackageInfo.INSTALL_LOCATION_PREFER_EXTERNAL) {
                                //【3.1.3】对于非系统应用，如果优先安装到外置，那么安装位置由 
                                // pkgLite.recommendedInstallLocation 决定！
                            } else {
                                //【3.1.4】对于非系统应用，其他情况进入这里！
                                if (isExternal(installedPkg)) {
                                    return PackageHelper.RECOMMEND_INSTALL_EXTERNAL;
                                }
                                return PackageHelper.RECOMMEND_INSTALL_INTERNAL;
                            }
                        }
                    } else {
                        //【3.2】异常情况
                        return PackageHelper.RECOMMEND_FAILED_ALREADY_EXISTS;
                    }
                }
            }
            //【4】如果上面的条件都不满足，并且指定了在 sdcard，那么就返回 RECOMMEND_INSTALL_EXTERNAL；
            if (onSd) {
                return PackageHelper.RECOMMEND_INSTALL_EXTERNAL;
            }
            //【5】其他情况，均由 pkgLite.recommendedInstallLocation 决定！
            return pkgLite.recommendedInstallLocation;
        }
```


### 5.5.3 PackageManagerS.createInstallArgs

其实就是针对不同的安装方式，创建不同的 InstallArgs！

```java
    private InstallArgs createInstallArgs(InstallParams params) {
        if (params.move != null) {
            //【1】如果是 move package，那么会创建 MoveInstallArgs 实例！
            return new MoveInstallArgs(params);
        } else if (installOnExternalAsec(params.installFlags) || params.isForwardLocked()) {
            //【2】对于安装到外置存储，或者 forward locked 安装，会创建 AsecInstallArgs 实例！
            return new AsecInstallArgs(params);
        } else {
            //【5.5.3.1】对于一般安装，创建 FileInstallArgs 实例！
            return new FileInstallArgs(params);
        }
    }
```
这里我们先关注一般情况，即创建 FileInstallArgs 实例的情况！

#### 5.5.3.1 new FileInstallArgs - 要安装的 apk

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

        //【2】用于描述已存在的一个安装！
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

- 一参数构造器用于创建安装过程中的 InstallArgs！

- 三参数构造器，用于描述一个已存在的安装，主要用于清除旧的安装，或者作为移动应用的时候的源数据，我们在 pms 开机初始化的过程中就已经看到过了！

当然，这里我们重点关注安装过程！


### 5.5.4 PackageManagerService.isVerificationEnabled

isVerificationEnabled 用于校验系统是否打开了校验功能！

```java
    private boolean isVerificationEnabled(int userId, int installFlags) {
        if (!DEFAULT_VERIFY_ENABLE) {//【1】如果默认没有打开校验，false！
            return false;
        }
        //【2】如果是 INSTALL_EPHEMERAL 方式的安装，不校验，false！
        if ((installFlags & PackageManager.INSTALL_EPHEMERAL) != 0) {
            if (DEBUG_EPHEMERAL) {
                Slog.d(TAG, "INSTALL_EPHEMERAL so skipping verification");
            }
            return false;
        }
        
        //【3】判断该用户下是否允许校验应用！
        boolean ensureVerifyAppsEnabled = isUserRestricted(userId, UserManager.ENSURE_VERIFY_APPS);

        //【4】如果安装指定了 INSTALL_FROM_ADB，进入这里！
        if ((installFlags & PackageManager.INSTALL_FROM_ADB) != 0) {
            //【4.1】test harness environment 不校验！
            if (ActivityManager.isRunningInTestHarness()) {
                return false;
            }
            //【4.2】如果同时该用户下能校验应用，返回 true！
            if (ensureVerifyAppsEnabled) {
                return true;
            }
            //【4.3】对于 adb install，如果系统属性 PACKAGE_VERIFIER_INCLUDE_ADB 为 0 ，那就不用校验！
            if (android.provider.Settings.Global.getInt(mContext.getContentResolver(),
                    android.provider.Settings.Global.PACKAGE_VERIFIER_INCLUDE_ADB, 1) == 0) {
                return false;
            }
        }
        
        //【5】如果该用户下能校验应用，返回 true！
        if (ensureVerifyAppsEnabled) {
            return true;
        }
        //【6】判断系统属性 PACKAGE_VERIFIER_ENABLE 如果为 0，那就不用校验！
        return android.provider.Settings.Global.getInt(mContext.getContentResolver(),
                android.provider.Settings.Global.PACKAGE_VERIFIER_ENABLE, 1) == 1;
    }
```
就不多说了！


### 5.5.5 new PackageVerificationState

PackageVerificationState 用于保存被安装的应用的校验状态信息：

```java
class PackageVerificationState {
    private final InstallArgs mArgs; // InstallArgs 实例
    private final SparseBooleanArray mSufficientVerifierUids; 
    private final int mRequiredVerifierUid; // 校验者的 uid！
    private boolean mSufficientVerificationComplete;
    private boolean mSufficientVerificationPassed;
    private boolean mRequiredVerificationComplete;
    private boolean mRequiredVerificationPassed;
    private boolean mExtendedTimeout;

    public PackageVerificationState(int requiredVerifierUid, InstallArgs args) {
        mRequiredVerifierUid = requiredVerifierUid;
        mArgs = args;
        mSufficientVerifierUids = new SparseBooleanArray();
        mExtendedTimeout = false; // 表示是否超时！
    }
```
不多说了！

## 5.6 PackageHandler.doHandleMessage[CHECK_PENDING_VERIFICATION] - 校验完成

```java
    case CHECK_PENDING_VERIFICATION: {
        //【1】获得校验唯一标识！
        final int verificationId = msg.arg1;
        //【2】获得本次校验的 PackageVerificationState 实例！
        final PackageVerificationState state = mPendingVerification.get(verificationId);
        if ((state != null) && !state.timeoutExtended()) {
            //【3】获得 InstallArgs 实例！
            final InstallArgs args = state.getInstallArgs();
            final Uri originUri = Uri.fromFile(args.origin.resolvedFile);

            Slog.i(TAG, "Verification timed out for " + originUri);
            //【4】从 mPendingVerification 中移除校验状态对象！
            mPendingVerification.remove(verificationId);

            int ret = PackageManager.INSTALL_FAILED_VERIFICATION_FAILURE;
            
            //【5】处理校验结果！
            if (getDefaultVerificationResponse() == PackageManager.VERIFICATION_ALLOW) {
                //【5.1】校验成功
                Slog.i(TAG, "Continuing with installation of " + originUri);
                state.setVerifierResponse(Binder.getCallingUid(),
                        PackageManager.VERIFICATION_ALLOW_WITHOUT_SUFFICIENT);
                //【5.6.1】发送 Intent.ACTION_PACKAGE_VERIFIED 广播
                broadcastPackageVerified(verificationId, originUri,
                        PackageManager.VERIFICATION_ALLOW,
                        state.getInstallArgs().getUser());
                try {
                    //【5.6.2】校验成功后，调用 installArgs.copyApk 继续处理！
                    ret = args.copyApk(mContainerService, true);
                } catch (RemoteException e) {
                    Slog.e(TAG, "Could not contact the ContainerService");
                }
            } else {
                //【5.6.1】校验失败，发送 Intent.ACTION_PACKAGE_VERIFIED 广播
                broadcastPackageVerified(verificationId, originUri,
                        PackageManager.VERIFICATION_REJECT,
                        state.getInstallArgs().getUser());
            }

            Trace.asyncTraceEnd(
                    TRACE_TAG_PACKAGE_MANAGER, "verification", verificationId);
            
            //【5.7】继续安装！
            processPendingInstall(args, ret);
            
            // 安装过程结束，发送 MCS_UNBIND 消息！
            mHandler.sendEmptyMessage(MCS_UNBIND);
        }
        break;
    }
```
这里我们通过 getDefaultVerificationResponse 方法返回校验结果：
```java
    private int getDefaultVerificationResponse() {
        return android.provider.Settings.Global.getInt(mContext.getContentResolver(),
                android.provider.Settings.Global.PACKAGE_VERIFIER_DEFAULT_RESPONSE,
                DEFAULT_VERIFICATION_RESPONSE);
    }
```
默认返回的是 PackageManager.VERIFICATION_ALLOW，比如在校验超时的情况下！

### 5.6.1 PackageManagerS.broadcastPackageVerified
```java
    private void broadcastPackageVerified(int verificationId, Uri packageUri,
            int verificationCode, UserHandle user) {
        final Intent intent = new Intent(Intent.ACTION_PACKAGE_VERIFIED);
        intent.setDataAndType(packageUri, PACKAGE_MIME_TYPE);
        intent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
        intent.putExtra(PackageManager.EXTRA_VERIFICATION_ID, verificationId);
        intent.putExtra(PackageManager.EXTRA_VERIFICATION_RESULT, verificationCode);

        mContext.sendBroadcastAsUser(intent, user,
                android.Manifest.permission.PACKAGE_VERIFICATION_AGENT);
    }
```

### 5.6.2 FileInstallArgs.(do)copyApk

```java
    int copyApk(IMediaContainerService imcs, boolean temp) throws RemoteException {
        Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "copyApk");
        try {
            return doCopyApk(imcs, temp);
        } finally {
            Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
        }
    }
```
copyApk 调用了 doCopyApk 方法：

```java
        private int doCopyApk(IMediaContainerService imcs, boolean temp) throws RemoteException {
            //【1】如果 origin.staged 为 true，那么说明应用已经 copy 到目标目录了，
            // 那就直接返回 PackageManager.INSTALL_SUCCEEDED！
            if (origin.staged) {
                if (DEBUG_INSTALL) Slog.d(TAG, origin.file + " already staged; skipping copy");
                //【1.1】设置 FileInstallArgs 的 codeFile 属性！
                codeFile = origin.file;
                resourceFile = origin.file;
                return PackageManager.INSTALL_SUCCEEDED;
            }
            
            //【2】如果 origin.staged 为 false，说明应用没有拷贝到目标目录！
            try {
                final boolean isEphemeral = (installFlags & PackageManager.INSTALL_EPHEMERAL) != 0;
                //【5.6.2.1】获得要拷贝的目标目录！
                final File tempDir =
                        mInstallerService.allocateStageDirLegacy(volumeUuid, isEphemeral);
                codeFile = tempDir;
                resourceFile = tempDir;

            } catch (IOException e) {
                Slog.w(TAG, "Failed to create copy file: " + e);
                return PackageManager.INSTALL_FAILED_INSUFFICIENT_STORAGE;
            }
            
            //【3】回调接口！
            final IParcelFileDescriptorFactory target = new IParcelFileDescriptorFactory.Stub() {
                @Override
                public ParcelFileDescriptor open(String name, int mode) throws RemoteException {
                    if (!FileUtils.isValidExtFilename(name)) {
                        throw new IllegalArgumentException("Invalid filename: " + name);
                    }
                    try {
                        //【3.1】访问 codeFile 目录下的 name 文件，设置权限，并返回其文件描述符！
                        final File file = new File(codeFile, name);
                        final FileDescriptor fd = Os.open(file.getAbsolutePath(),
                                O_RDWR | O_CREAT, 0644);
                        Os.chmod(file.getAbsolutePath(), 0644);
                        return new ParcelFileDescriptor(fd);
                    } catch (ErrnoException e) {
                        throw new RemoteException("Failed to open: " + e.getMessage());
                    }
                }
            };

            int ret = PackageManager.INSTALL_SUCCEEDED;
            //【5.6.2.2】将应用程序拷贝到目标目录！
            ret = imcs.copyPackage(origin.file.getAbsolutePath(), target);
            if (ret != PackageManager.INSTALL_SUCCEEDED) {
                Slog.e(TAG, "Failed to copy package");
                return ret;
            }
            
            //【4】解压本地 lib 库文件！
            final File libraryRoot = new File(codeFile, LIB_DIR_NAME);
            NativeLibraryHelper.Handle handle = null;
            try {
                handle = NativeLibraryHelper.Handle.create(codeFile);
                ret = NativeLibraryHelper.copyNativeBinariesWithOverride(handle, libraryRoot,
                        abiOverride);
            } catch (IOException e) {
                Slog.e(TAG, "Copying native libraries failed", e);
                ret = PackageManager.INSTALL_FAILED_INTERNAL_ERROR;
            } finally {
                IoUtils.closeQuietly(handle);
            }

            return ret;
        }
```
根据前面创建 OriginInfo 时知道：

```java
        if (stagedDir != null) {
            origin = OriginInfo.fromStagedFile(stagedDir);
        } else {
            origin = OriginInfo.fromStagedContainer(stagedCid);
        }
```
通过 adb 安装时，安装到内置存储，会调用 OriginInfo.fromStagedFile 方法，此时 OriginInfo.staged 为 true，安装到外置存储时， OriginInfo.staged 也会为 true。

这是因为在进行 adb install 过程中时，我们通过 Session，已经拷贝到目标目录了：/data/app/vml[sessionId].tmp/，所以这里 doCopyApk 无需在继续进行下去！！

对于其他的安装方式，OriginInfo.staged 为 false 的，那么会进入 doCopyApk 的下一步！


#### 5.6.2.1 PackageInstallerService.allocateStageDirLegacy
```java
    @Deprecated
    public File allocateStageDirLegacy(String volumeUuid, boolean isEphemeral) throws IOException {
        synchronized (mSessions) {
            try {
                //【3.1.2】分配一个 sessionId，保存到 mLegacySessions 中！
                final int sessionId = allocateSessionIdLocked();
                mLegacySessions.put(sessionId, true);
                //【3.1.2】创建目标目录！
                final File stageDir = buildStageDir(volumeUuid, sessionId, isEphemeral);
                //【2】创建目录，设置权限！
                prepareStageDir(stageDir);
                return stageDir;
            } catch (IllegalStateException e) {
                throw new IOException(e);
            }
        }
    }
```
这里，前面分析过，这里就不多说了！


#### 5.6.2.2 DefaultContainerService.copyPackage(Inner)

```java
        @Override
        public int copyPackage(String packagePath, IParcelFileDescriptorFactory target) {
            if (packagePath == null || target == null) {
                return PackageManager.INSTALL_FAILED_INVALID_URI;
            }

            PackageLite pkg = null;
            try {
                final File packageFile = new File(packagePath);
                //【1】解析应用！
                pkg = PackageParser.parsePackageLite(packageFile, 0);
                //【2】继续处理 copyPackageInner
                return copyPackageInner(pkg, target);
            } catch (PackageParserException | IOException | RemoteException e) {
                Slog.w(TAG, "Failed to copy package at " + packagePath + ": " + e);
                return PackageManager.INSTALL_FAILED_INSUFFICIENT_STORAGE;
            }
        }
```
这里继续调用了 copyPackageInner 方法
```java
    private int copyPackageInner(PackageLite pkg, IParcelFileDescriptorFactory target)
            throws IOException, RemoteException {
        //【1】对 base apk 和 split apk 分别执行 copy！
        copyFile(pkg.baseCodePath, target, "base.apk");
        if (!ArrayUtils.isEmpty(pkg.splitNames)) {
            for (int i = 0; i < pkg.splitNames.length; i++) {
                copyFile(pkg.splitCodePaths[i], target, "split_" + pkg.splitNames[i] + ".apk");
            }
        }

        return PackageManager.INSTALL_SUCCEEDED;
    }
```
最终，调用了 copyFile 执行 copy！

```java
    private void copyFile(String sourcePath, IParcelFileDescriptorFactory target, String targetName)
            throws IOException, RemoteException {
        Slog.d(TAG, "Copying " + sourcePath + " to " + targetName);
        InputStream in = null;
        OutputStream out = null;
        try {
            in = new FileInputStream(sourcePath);
            out = new ParcelFileDescriptor.AutoCloseOutputStream(
                    target.open(targetName, ParcelFileDescriptor.MODE_READ_WRITE));
            Streams.copy(in, out);
        } finally {
            IoUtils.closeQuietly(out);
            IoUtils.closeQuietly(in);
        }
    }
```
到这里就结束了！

## 5.7 PackageManagerS.processPendingInstall

processPendingInstall 用于继续安装：

```java
    private void processPendingInstall(final InstallArgs args, final int currentStatus) {
        // Queue up an async operation since the package installation may take a little while.
        mHandler.post(new Runnable() {
            public void run() {
                mHandler.removeCallbacks(this);

                //【5.7.1】创建 PackageInstalledInfo 实例，封装安装结果！
                PackageInstalledInfo res = new PackageInstalledInfo();
                res.setReturnCode(currentStatus); // 保存当前的返回码
                res.uid = -1;
                res.pkg = null;
                res.removedInfo = null;

                //【1】如果返回码为 PackageManager.INSTALL_SUCCEEDED！
                if (res.returnCode == PackageManager.INSTALL_SUCCEEDED) {
                    //【×5.7.2】安装前处理；
                    args.doPreInstall(res.returnCode);
                    synchronized (mInstallLock) {
                        //【×5.7.3】执行安装！
                        installPackageTracedLI(args, res);
                    }
                    //【×5.7.4】安装后处理；
                    args.doPostInstall(res.returnCode, res.uid);
                }

                //【2】判断是否执行备份，要备份必须满足 3 个条件！
                // 1、安装正常；2、本次安装并不是更新操作！3、应用允许备份
                final boolean update = res.removedInfo != null
                        && res.removedInfo.removedPackage != null;
                final int flags = (res.pkg == null) ? 0 : res.pkg.applicationInfo.flags;
                boolean doRestore = !update
                        && ((flags & ApplicationInfo.FLAG_ALLOW_BACKUP) != 0);

                //【3】计算本次安装的 token
                int token;
                if (mNextInstallToken < 0) mNextInstallToken = 1;
                token = mNextInstallToken++;
                
                //【*5.7.5】创建一个 PostInstallData 对象，并将其加入 mRunningInstalls 中！
                PostInstallData data = new PostInstallData(args, res);
                mRunningInstalls.put(token, data);

                if (DEBUG_INSTALL) Log.v(TAG, "+ starting restore round-trip " + token);

                if (res.returnCode == PackageManager.INSTALL_SUCCEEDED && doRestore) {
                    //【4】如果安装成功，并且需要备份，那就获得 BackupManager 执行备份！
                    IBackupManager bm = IBackupManager.Stub.asInterface(
                            ServiceManager.getService(Context.BACKUP_SERVICE));
                    if (bm != null) {
                        if (DEBUG_INSTALL) Log.v(TAG, "token " + token
                                + " to BM for possible restore");
                        Trace.asyncTraceBegin(TRACE_TAG_PACKAGE_MANAGER, "restore", token);
                        try {
                            // TODO: http://b/22388012
                            if (bm.isBackupServiceActive(UserHandle.USER_SYSTEM)) {
                                bm.restoreAtInstall(res.pkg.applicationInfo.packageName, token);
                            } else {
                                doRestore = false;
                            }
                        } catch (RemoteException e) {
                            // can't happen; the backup manager is local
                        } catch (Exception e) {
                            Slog.e(TAG, "Exception trying to enqueue restore", e);
                            doRestore = false;
                        }
                    } else {
                        Slog.e(TAG, "Backup Manager not found!");
                        doRestore = false;
                    }
                }
                //【5】如果备份失败或者不需要备份，那就进入这个阶段！
                if (!doRestore) {
                    // No restore possible, or the Backup Manager was mysteriously not
                    // available -- just fire the post-install work request directly.
                    if (DEBUG_INSTALL) Log.v(TAG, "No restore - queue post-install for " + token);

                    Trace.asyncTraceBegin(TRACE_TAG_PACKAGE_MANAGER, "postInstall", token);

                    Message msg = mHandler.obtainMessage(POST_INSTALL, token, 0);
                    mHandler.sendMessage(msg);
                }
            }
        });
    }
```

### 5.7.1 new PackageInstalledInfo

PackageInstalledInfo 用于保存该应用的安装结果信息！

```java
    static class PackageInstalledInfo {
        String name;
        int uid;
        int[] origUsers; // 该 pacakge 之前安装后所属的 user
        int[] newUsers; // 该 pacakge 现在安装后所属的 user
        PackageParser.Package pkg;
        int returnCode; // 返回码
        String returnMsg;
        //【7.7.1.1】用于封装要移除的 apk 的信息！
        PackageRemovedInfo removedInfo;
        ArrayMap<String, PackageInstalledInfo> addedChildPackages; // split apk 的安装结果信息！

        ... ... ...

        // In some error cases we want to convey more info back to the observer
        String origPackage;
        String origPermission;
    }
```
我们先看到这里！

#### 5.7.1.1 new PackageRemovedInfo

封装要移除的 apk 的信息：

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
        //【1】InstallArgs 用于清除 apk 的相关数据，后面会看到！
        InstallArgs args = null;
        ArrayMap<String, PackageRemovedInfo> removedChildPackages;
        ArrayMap<String, PackageInstalledInfo> appearedChildPackages;
        ... ... ...
    }
```

### 5.7.2 FileInstallArgs.doPreInstall

安装前执行清理操作，正常情况不会触发！
```java
    int doPreInstall(int status) {
        //【1】如果安装前的状态不是 PackageManager.INSTALL_SUCCEEDED
        if (status != PackageManager.INSTALL_SUCCEEDED) {
            //【5.7.2.1】执行清理操作！
            cleanUp();
        }
        return status;
    }
```
#### 5.7.2.1 FileInstallArgs.cleanUp(ResourcesLI)

清理操作！
```java
        private boolean cleanUp() {
            if (codeFile == null || !codeFile.exists()) {
                return false;
            }
            //【1】删除目标目录下的所有文件！
            removeCodePathLI(codeFile);

            if (resourceFile != null && !FileUtils.contains(codeFile, resourceFile)) {
                resourceFile.delete();
            }

            return true;
        }
```
可以看到，最后调用了 removeCodePathLI 方法！
```java
    void removeCodePathLI(File codePath) {
        if (codePath.isDirectory()) {
            try {
                mInstaller.rmPackageDir(codePath.getAbsolutePath());
            } catch (InstallerException e) {
                Slog.w(TAG, "Failed to remove code path", e);
            }
        } else {
            codePath.delete();
        }
    }
```

### 5.7.3 PackageManagerS.installPackageTracedLI - 核心安装入口

```java
    private void installPackageTracedLI(InstallArgs args, PackageInstalledInfo res) {
        try {
            Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "installPackage");
            //【1】调用了 installPackageLI 方法！
            installPackageLI(args, res);
        } finally {
            Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
        }
    }
```
这里调用了 installPackageLI 方法：

```java
    private void installPackageLI(InstallArgs args, PackageInstalledInfo res) {
        //【1】获得安装时传入的参数和标志位！！
        final int installFlags = args.installFlags;
        final String installerPackageName = args.installerPackageName;
        final String volumeUuid = args.volumeUuid;
        final File tmpPackageFile = new File(args.getCodePath());
        final boolean forwardLocked = ((installFlags & PackageManager.INSTALL_FORWARD_LOCK) != 0);
        final boolean onExternal = (((installFlags & PackageManager.INSTALL_EXTERNAL) != 0)
                || (args.volumeUuid != null));
        final boolean ephemeral = ((installFlags & PackageManager.INSTALL_EPHEMERAL) != 0);
        final boolean forceSdk = ((installFlags & PackageManager.INSTALL_FORCE_SDK) != 0);
        boolean replace = false;

        //【2】重新设置扫描参数
        int scanFlags = SCAN_NEW_INSTALL | SCAN_UPDATE_SIGNATURE;
        if (args.move != null) {
            //【2.1】如果 args.move 不为 null，表示正在移动一个 app，我们会对其进行一个初始化的扫描
            // 增加 SCAN_INITIAL 位！
            scanFlags |= SCAN_INITIAL;
        }
        if ((installFlags & PackageManager.INSTALL_DONT_KILL_APP) != 0) { 
            //【2.2】如果安装参数指定了 INSTALL_DONT_KILL_APP，那么增加 SCAN_DONT_KILL_APP 位！
            scanFlags |= SCAN_DONT_KILL_APP;
        }

        //【3】更新结果码！
        res.setReturnCode(PackageManager.INSTALL_SUCCEEDED);

        if (DEBUG_INSTALL) Slog.d(TAG, "installPackageLI: path=" + tmpPackageFile);

        //【4】检查 ephemeral 是否和 forwardLocked/onExternal 共存，共存则报错！
        if (ephemeral && (forwardLocked || onExternal)) {
            Slog.i(TAG, "Incompatible ephemeral install; fwdLocked=" + forwardLocked
                    + " external=" + onExternal);
            res.setReturnCode(PackageManager.INSTALL_FAILED_EPHEMERAL_INVALID);
            return;
        }

        //【5】设置解析参数 parseFlags
        final int parseFlags = mDefParseFlags | PackageParser.PARSE_CHATTY
                | PackageParser.PARSE_ENFORCE_CODE
                | (forwardLocked ? PackageParser.PARSE_FORWARD_LOCK : 0)
                | (onExternal ? PackageParser.PARSE_EXTERNAL_STORAGE : 0)
                | (ephemeral ? PackageParser.PARSE_IS_EPHEMERAL : 0)
                | (forceSdk ? PackageParser.PARSE_FORCE_SDK : 0);

        //【6】创建解析对象！
        PackageParser pp = new PackageParser();
        pp.setSeparateProcesses(mSeparateProcesses);
        pp.setDisplayMetrics(mMetrics);

        Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "parsePackage");
        final PackageParser.Package pkg;
        try {
            //【*5.7.3.1】解析 apk，获得其对应的 Package 对象！
            pkg = pp.parsePackage(tmpPackageFile, parseFlags);
        } catch (PackageParserException e) {
            res.setError("Failed parse during installPackageLI", e);
            return;
        } finally {
            Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
        }

        //【7】如果该应用有子包的话，那么对于每个子包，也会创建安装结果对象！！
        if (pkg.childPackages != null) {
            synchronized (mPackages) {
                final int childCount = pkg.childPackages.size();
                for (int i = 0; i < childCount; i++) {
                    PackageParser.Package childPkg = pkg.childPackages.get(i);
                    //【*5.7.1】针对子包，创建 PackageInstalledInfo 对象!
                    // 设置返回码，子包包名！
                    PackageInstalledInfo childRes = new PackageInstalledInfo();
                    childRes.setReturnCode(PackageManager.INSTALL_SUCCEEDED);
                    childRes.pkg = childPkg;
                    childRes.name = childPkg.packageName;
                    //【7.1】获得子包的源 user!
                    PackageSetting childPs = mSettings.peekPackageLPr(childPkg.packageName);
                    if (childPs != null) {
                        childRes.origUsers = childPs.queryInstalledUsers(
                                sUserManager.getUserIds(), true);
                    }
                    //【7.2】如果子包之前被扫描到了（安装过），创建 PackageRemovedInfo 对象！
                    if ((mPackages.containsKey(childPkg.packageName))) {
                        childRes.removedInfo = new PackageRemovedInfo();
                        childRes.removedInfo.removedPackage = childPkg.packageName;
                    }
                    if (res.addedChildPackages == null) {
                        res.addedChildPackages = new ArrayMap<>();
                    }
                    //【7.3】将子包的安装结果对象保存到 base apk 的信息中！
                    res.addedChildPackages.put(childPkg.packageName, childRes);
                }
            }
        }
        // 如果应用没有指定 abi，我们通过安装参数指定！
        if (TextUtils.isEmpty(pkg.cpuAbiOverride)) {
            pkg.cpuAbiOverride = args.abiOverride;
        }
        // 如果应用指定了 ApplicationInfo.FLAG_TEST_ONLY，那么安装参数也需要指定这个参数！
        String pkgName = res.name = pkg.packageName;
        if ((pkg.applicationInfo.flags&ApplicationInfo.FLAG_TEST_ONLY) != 0) {
            if ((installFlags & PackageManager.INSTALL_ALLOW_TEST) == 0) {
                res.setError(INSTALL_FAILED_TEST_ONLY, "installPackageLI");
                return;
            }
        }

        try {
            //【8】收集证书信息！
            if (args.certificates != null) {
                try {
                    PackageParser.populateCertificates(pkg, args.certificates);
                } catch (PackageParserException e) {
                    PackageParser.collectCertificates(pkg, parseFlags);
                }
            } else {
                PackageParser.collectCertificates(pkg, parseFlags);
            }
        } catch (PackageParserException e) {
            res.setError("Failed collect during installPackageLI", e);
            return;
        }

        // Get rid of all references to package scan path via parser.
        pp = null;
        String oldCodePath = null;
        boolean systemApp = false;
        synchronized (mPackages) {
            //【9】如果安装参数指定了 INSTALL_REPLACE_EXISTING，那么我们要尝试判断是否存在已安装的应用！
            // 如果存在，那就要 replace！
            if ((installFlags & PackageManager.INSTALL_REPLACE_EXISTING) != 0) {
                String oldName = mSettings.mRenamedPackages.get(pkgName); // 判断该应用是否重命名过！
                if (pkg.mOriginalPackages != null
                        && pkg.mOriginalPackages.contains(oldName)
                        && mPackages.containsKey(oldName)) {

                    //【9.1】如果有源包（系统应用才会有），要命名为源包，replace 为 true！
                    pkg.setPackageName(oldName);
                    pkgName = pkg.packageName;
                    replace = true;
                    if (DEBUG_INSTALL) Slog.d(TAG, "Replacing existing renamed package: oldName="
                            + oldName + " pkgName=" + pkgName);

                } else if (mPackages.containsKey(pkgName)) {
                    //【9.2】如果没有源包（系统应用才会有），但是已经有相同包名的应用存在，replace 为 true！
                    replace = true;
                    if (DEBUG_INSTALL) Slog.d(TAG, "Replace existing pacakge: " + pkgName);
                }

                //【9.2】对于子包，只能通过父包更新，这里不处理子包的 replace！
                if (pkg.parentPackage != null) {
                    res.setError(PackageManager.INSTALL_PARSE_FAILED_BAD_PACKAGE_NAME,
                            "Package " + pkg.packageName + " is child of package "
                                    + pkg.parentPackage.parentPackage + ". Child packages "
                                    + "can be updated only through the parent package.");
                    return;
                }

                //【9.3】如果需要替换已存在的 apk，那么需要做 sdk 的校验！
                // 如果旧应用支持运行时，不允许新的应用不支持运行时！
                if (replace) {
                    PackageParser.Package oldPackage = mPackages.get(pkgName);
                    final int oldTargetSdk = oldPackage.applicationInfo.targetSdkVersion;
                    final int newTargetSdk = pkg.applicationInfo.targetSdkVersion;
                    if (oldTargetSdk > Build.VERSION_CODES.LOLLIPOP_MR1
                            && newTargetSdk <= Build.VERSION_CODES.LOLLIPOP_MR1) {
                        res.setError(PackageManager.INSTALL_FAILED_PERMISSION_MODEL_DOWNGRADE,
                                "Package " + pkg.packageName + " new target SDK " + newTargetSdk
                                        + " doesn't support runtime permissions but the old"
                                        + " target SDK " + oldTargetSdk + " does.");
                        return;
                    }

                    //【9.4】如果旧包也是子包，也不安装！
                    if (oldPackage.parentPackage != null) {
                        res.setError(PackageManager.INSTALL_PARSE_FAILED_BAD_PACKAGE_NAME,
                                "Package " + pkg.packageName + " is child of package "
                                        + oldPackage.parentPackage + ". Child packages "
                                        + "can be updated only through the parent package.");
                        return;
                    }
                }
            }
            
            //【10】获得上一次的安装数据！
            PackageSetting ps = mSettings.mPackages.get(pkgName);
            if (ps != null) {
                if (DEBUG_INSTALL) Slog.d(TAG, "Existing package: " + ps);
                //【10.1】如果安装过旧版本，那就要校验签名！
                //【×5.3.7.2】shouldCheckUpgradeKeySetLP 用于判断是否检查签名更新；
                if (shouldCheckUpgradeKeySetLP(ps, scanFlags)) {
                    //【×5.3.7.3】checkUpgradeKeySetLP 用于检查签名更新；
                    if (!checkUpgradeKeySetLP(ps, pkg)) {
                        res.setError(INSTALL_FAILED_UPDATE_INCOMPATIBLE, "Package "
                                + pkg.packageName + " upgrade keys do not match the "
                                + "previously installed version");
                        return;
                    }
                } else {
                    try {
                        //【×5.3.7.4】校验签名，如果签名不相等，那就禁止安装！
                        verifySignaturesLP(ps, pkg);
                    } catch (PackageManagerException e) {
                        res.setError(e.error, e.getMessage());
                        return;
                    }
                }
                //【10.2】获得旧的 apk 的路径，判断旧应用是否是系统应用！
                oldCodePath = mSettings.mPackages.get(pkgName).codePathString;
                if (ps.pkg != null && ps.pkg.applicationInfo != null) {
                    systemApp = (ps.pkg.applicationInfo.flags &
                            ApplicationInfo.FLAG_SYSTEM) != 0;
                }
                res.origUsers = ps.queryInstalledUsers(sUserManager.getUserIds(), true);
            }

            //【11】检查权限是否被重新定义，如果重新定义，那就要校验权限！
            int N = pkg.permissions.size();
            for (int i = N-1; i >= 0; i--) {
                //【11.1】获得该应用定义的权限
                PackageParser.Permission perm = pkg.permissions.get(i);
                //【11.2】尝试从系统中获得该权限已有的信息！
                BasePermission bp = mSettings.mPermissions.get(perm.info.name);
                if (bp != null) {
                    //【11.3】如果该权限已经被定义过了，那就要校验下签名！
                    final boolean sigsOk;
                    if (bp.sourcePackage.equals(pkg.packageName)
                            && (bp.packageSetting instanceof PackageSetting)
                            && (shouldCheckUpgradeKeySetLP((PackageSetting) bp.packageSetting,
                                    scanFlags))) {
                        //【11.3.1】检查签名更新！
                        sigsOk = checkUpgradeKeySetLP((PackageSetting) bp.packageSetting, pkg);
                    } else {
                        //【11.3.2】如果不检查签名更新，那就直接比较签名！
                        sigsOk = compareSignatures(bp.packageSetting.signatures.mSignatures,
                                pkg.mSignatures) == PackageManager.SIGNATURE_MATCH;
                    }
                    //【11.4】如果签名发生了变化！
                    if (!sigsOk) {
                        if (!bp.sourcePackage.equals("android")) {
                            //【11.4.1】如果权限的定义者不是系统，那么不允许重新定义，同时不允许继续安装！
                            res.setError(INSTALL_FAILED_DUPLICATE_PERMISSION, "Package "
                                    + pkg.packageName + " attempting to redeclare permission "
                                    + perm.info.name + " already owned by " + bp.sourcePackage);
                            res.origPermission = perm.info.name;
                            res.origPackage = bp.sourcePackage;
                            return;
                        } else {
                            //【11.4.2】如果权限的定义不是系统，那么允许安装，但是会忽视掉新的定义！
                            Slog.w(TAG, "Package " + pkg.packageName
                                    + " attempting to redeclare system permission "
                                    + perm.info.name + "; ignoring new declaration");
                            pkg.permissions.remove(i);
                        }
                    }
                }
            }
        }
        
        //【12】如果覆盖更新的是系统应用，要针对安装位置做判断！
        // 新的 apk 不能是 onExternal / ephemeral！
        if (systemApp) {
            if (onExternal) {
                res.setError(INSTALL_FAILED_INVALID_INSTALL_LOCATION,
                        "Cannot install updates to system apps on sdcard");
                return;
            } else if (ephemeral) {
                res.setError(INSTALL_FAILED_EPHEMERAL_INVALID,
                        "Cannot update a system app with an ephemeral app");
                return;
            }
        }

        //【13】根据安装参数做不同的处理！
        if (args.move != null) {
            //【13.1】如果是 move package，进入这里！
            scanFlags |= SCAN_NO_DEX; // 设置以下标签，无需做 odex，我们需要已有的移动过去即可！
            scanFlags |= SCAN_MOVE;

            synchronized (mPackages) {
                final PackageSetting ps = mSettings.mPackages.get(pkgName);
                if (ps == null) {
                    res.setError(INSTALL_FAILED_INTERNAL_ERROR,
                            "Missing settings for moved package " + pkgName);
                }

                //【13.1.1】对于 abi，和移动前的保持一致！
                pkg.applicationInfo.primaryCpuAbi = ps.primaryCpuAbiString;
                pkg.applicationInfo.secondaryCpuAbi = ps.secondaryCpuAbiString;
            }

        } else if (!forwardLocked && !pkg.applicationInfo.isExternalAsec()) {
            //【13.2】如果不是 forward lock 模式安装且没有安装到外置存储上，进入这里！
            scanFlags |= SCAN_NO_DEX; // 扫描参数设置 SCAN_NO_DEX，意味着后面不做 odex，因为这里会做！

            try {
                String abiOverride = (TextUtils.isEmpty(pkg.cpuAbiOverride) ?
                    args.abiOverride : pkg.cpuAbiOverride);
                derivePackageAbi(pkg, new File(pkg.codePath), abiOverride,
                        true /* extract libs */);
            } catch (PackageManagerException pme) {
                Slog.e(TAG, "Error deriving application ABI", pme);
                res.setError(INSTALL_FAILED_INTERNAL_ERROR, "Error deriving application ABI");
                return;
            }

            //【13.2.1】更新共享库文件！！
            synchronized (mPackages) {
                try {
                    updateSharedLibrariesLPw(pkg, null);
                } catch (PackageManagerException e) {
                    Slog.e(TAG, "updateSharedLibrariesLPw failed: " + e.getMessage());
                }
            }
            Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "dexopt");
            
            //【13.2.2】执行 odex 优化
            mPackageDexOptimizer.performDexOpt(pkg, pkg.usesLibraryFiles,
                    null /* instructionSets */, false /* checkProfiles */,
                    getCompilerFilterForReason(REASON_INSTALL),
                    getOrCreateCompilerPackageStats(pkg));
            Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);

            //【13.2.3】将该应用从 odex fail list 中删除！
            BackgroundDexOptService.notifyPackageChanged(pkg.packageName);
        }

        //【×5.7.3.5】重命名文件目录，此时文件目录由 /data/app/tmpl[SessionId].tmp 变为 /data/app/packageName-X！
        // 如果 X 存在，那么会 X + 1;
        if (!args.doRename(res.returnCode, pkg, oldCodePath)) {
            res.setError(INSTALL_FAILED_INSUFFICIENT_STORAGE, "Failed rename");
            return;
        }

        //【*5.7.3.6】开始 intentFilter 校验！
        startIntentFilterVerifications(args.user.getIdentifier(), replace, pkg);

        //【*5.7.3.7】冻结应用！
        try (PackageFreezer freezer = freezePackageForInstall(pkgName, installFlags,
                "installPackageLI")) {
            if (replace) {
                //【*6】如果是覆盖安装，进入这里！
                replacePackageLIF(pkg, parseFlags, scanFlags | SCAN_REPLACING, args.user,
                        installerPackageName, res);
            } else {
                //【*7】如果是全新安装，进入这里！
                installNewPackageLIF(pkg, parseFlags, scanFlags | SCAN_DELETE_DATA_ON_FAILURES,
                        args.user, installerPackageName, volumeUuid, res);
            }
        }
        //【14】为本次安装的 apk 更新目标用户信息！
        synchronized (mPackages) {
            final PackageSetting ps = mSettings.mPackages.get(pkgName);
            if (ps != null) {
                res.newUsers = ps.queryInstalledUsers(sUserManager.getUserIds(), true);
            }

            final int childCount = (pkg.childPackages != null) ? pkg.childPackages.size() : 0;
            for (int i = 0; i < childCount; i++) {
                PackageParser.Package childPkg = pkg.childPackages.get(i);
                PackageInstalledInfo childRes = res.addedChildPackages.get(childPkg.packageName);
                PackageSetting childPs = mSettings.peekPackageLPr(childPkg.packageName);
                if (childPs != null) {
                    childRes.newUsers = childPs.queryInstalledUsers(
                            sUserManager.getUserIds(), true);
                }
            }
        }
    }

```

#### 5.7.3.1 PackageParser.parsePackage

PackageParser.parsePackage 其实我们在 PMS 启动扫描的过程中已经分析过了，这里我们重点关注其对 installFlags 的处理！




#### 5.7.3.2 shouldCheckUpgradeKeySetLP

判断是否应该检查签名更新：

```java
    private boolean shouldCheckUpgradeKeySetLP(PackageSetting oldPs, int scanFlags) {
        //【1】以下情况无需检查签名更新：
        // 1、本次是全新安装；2、扫描设置了 SCAN_INITIAL(move pkg)；3、本次是覆盖安装，但是应用是 sharedUser 的！
        // 4、应用不使用 upgradeKeySets！
        if (oldPs == null || (scanFlags & SCAN_INITIAL) != 0 || oldPs.sharedUser != null
                || !oldPs.keySetData.isUsingUpgradeKeySets()) {
            return false;
        }
        //【2】如果应用是覆盖安装，且不是 move pkg，且不是共享 uid 的，且使用 upgradeKeySets，那么就要检查
        // upgradeKeySets 有效性，只有有效的 upgradeKeySets 才能检查更新！
        KeySetManagerService ksms = mSettings.mKeySetManagerService;
        long[] upgradeKeySets = oldPs.keySetData.getUpgradeKeySets();
        for (int i = 0; i < upgradeKeySets.length; i++) {
            //【2.1】判断 upgradeKeySets 是否有效！
            if (!ksms.isIdValidKeySetId(upgradeKeySets[i])) {
                Slog.wtf(TAG, "Package "
                         + (oldPs.name != null ? oldPs.name : "<null>")
                         + " contains upgrade-key-set reference to unknown key-set: "
                         + upgradeKeySets[i]
                         + " reverting to signatures check.");
                return false;
            }
        }
        return true;
    }
```
先分析到这里！

#### 5.7.3.3 checkUpgradeKeySetLP

进行签名更新检查：
```java
    private boolean checkUpgradeKeySetLP(PackageSetting oldPS, PackageParser.Package newPkg) {
        //【1】检查 key set 更新，更新有效的前提是新的 apk 持有的 keyset 至少包含旧应用的 keyset！
        long[] upgradeKeySets = oldPS.keySetData.getUpgradeKeySets();
        KeySetManagerService ksms = mSettings.mKeySetManagerService;
        for (int i = 0; i < upgradeKeySets.length; i++) {
            Set<PublicKey> upgradeSet = ksms.getPublicKeysFromKeySetLPr(upgradeKeySets[i]);
            if (upgradeSet != null && newPkg.mSigningKeys.containsAll(upgradeSet)) {
                return true;
            }
        }
        return false;
    }
```

#### 5.7.3.4 verifySignaturesLP

校验签名：

```java
    private void verifySignaturesLP(PackageSetting pkgSetting, PackageParser.Package pkg)
            throws PackageManagerException {
        if (pkgSetting.signatures.mSignatures != null) {
            // Already existing package. Make sure signatures match
            boolean match = compareSignatures(pkgSetting.signatures.mSignatures, pkg.mSignatures)
                    == PackageManager.SIGNATURE_MATCH;
            if (!match) {
                match = compareSignaturesCompat(pkgSetting.signatures, pkg)
                        == PackageManager.SIGNATURE_MATCH;
            }
            if (!match) {
                match = compareSignaturesRecover(pkgSetting.signatures, pkg)
                        == PackageManager.SIGNATURE_MATCH;
            }
            if (!match) {
                throw new PackageManagerException(INSTALL_FAILED_UPDATE_INCOMPATIBLE, "Package "
                        + pkg.packageName + " signatures do not match the "
                        + "previously installed version; ignoring!");
            }
        }

        // Check for shared user signatures
        if (pkgSetting.sharedUser != null && pkgSetting.sharedUser.signatures.mSignatures != null) {
            // Already existing package. Make sure signatures match
            boolean match = compareSignatures(pkgSetting.sharedUser.signatures.mSignatures,
                    pkg.mSignatures) == PackageManager.SIGNATURE_MATCH;
            if (!match) {
                match = compareSignaturesCompat(pkgSetting.sharedUser.signatures, pkg)
                        == PackageManager.SIGNATURE_MATCH;
            }
            if (!match) {
                match = compareSignaturesRecover(pkgSetting.sharedUser.signatures, pkg)
                        == PackageManager.SIGNATURE_MATCH;
            }
            if (!match) {
                throw new PackageManagerException(INSTALL_FAILED_SHARED_USER_INCOMPATIBLE,
                        "Package " + pkg.packageName
                        + " has no signatures that match those in shared user "
                        + pkgSetting.sharedUser.name + "; ignoring!");
            }
        }
    }
```


#### 5.7.3.5 FileInstallArgs.doRename - 重命名临时目录

doRename 重命名文件！之前我们的临时目录为 /data/app/tmpl[SessionId].tmp，这里会将其重命名为 /data/app/packageName-X！！

```java
        boolean doRename(int status, PackageParser.Package pkg, String oldCodePath) {
            if (status != PackageManager.INSTALL_SUCCEEDED) {
                cleanUp();
                return false;
            }
            //【1】获得改名前的临时目录 targetDir 就是 /data/app/tmpl[SessionId].tmp！
            final File targetDir = codeFile.getParentFile();
            final File beforeCodeFile = codeFile;

            //【5.7.3.5.1】获得改名后的目录：/data/app/packageName-X，策略取决为 getNextCodePath！
            final File afterCodeFile = getNextCodePath(targetDir, pkg.packageName);

            if (DEBUG_INSTALL) Slog.d(TAG, "Renaming " + beforeCodeFile + " to " + afterCodeFile);
            try {
                Os.rename(beforeCodeFile.getAbsolutePath(), afterCodeFile.getAbsolutePath());
            } catch (ErrnoException e) {
                Slog.w(TAG, "Failed to rename", e);
                return false;
            }
            
            //【2】恢复默认的安全上下文！
            if (!SELinux.restoreconRecursive(afterCodeFile)) {
                Slog.w(TAG, "Failed to restorecon");
                return false;
            }

            //【3】更新 FileInstallArgs 的目录为最终目录！
            codeFile = afterCodeFile;
            resourceFile = afterCodeFile;

            //【4】更新扫描到的应用的 Package 中的目录数据！
            // 以及 Application 中的数据！
            pkg.setCodePath(afterCodeFile.getAbsolutePath());
            pkg.setBaseCodePath(FileUtils.rewriteAfterRename(beforeCodeFile,
                    afterCodeFile, pkg.baseCodePath));
            pkg.setSplitCodePaths(FileUtils.rewriteAfterRename(beforeCodeFile,
                    afterCodeFile, pkg.splitCodePaths));

            pkg.setApplicationVolumeUuid(pkg.volumeUuid);
            pkg.setApplicationInfoCodePath(pkg.codePath);
            pkg.setApplicationInfoBaseCodePath(pkg.baseCodePath);
            pkg.setApplicationInfoSplitCodePaths(pkg.splitCodePaths);
            pkg.setApplicationInfoResourcePath(pkg.codePath);
            pkg.setApplicationInfoBaseResourcePath(pkg.baseCodePath);
            pkg.setApplicationInfoSplitResourcePaths(pkg.splitCodePaths);

            return true;
        }
```

流程很简单，不多说了！

##### 5.7.3.5.1 getNextCodePath

getNextCodePath 方法用于获得最终的目录！

```java
    private File getNextCodePath(File targetDir, String packageName) {
        int suffix = 1;
        File result;
        do {
            //【1】最终目录为 /data/app/packageName-X(X=1,2,3...)
            // suffix 从 1 开始，如果 suffix 已存在，那就 +1;
            result = new File(targetDir, packageName + "-" + suffix);
            suffix++;
        } while (result.exists());
        return result;
    }
```
逻辑很简单，不多说了！


#### 5.7.3.7 freezePackageForInstall - 冻结应用

该方法用于冻结应用：

```java
    public PackageFreezer freezePackageForInstall(String packageName, int installFlags,
            String killReason) {
        return freezePackageForInstall(packageName, UserHandle.USER_ALL, installFlags, killReason);
    }
```
继续来看：
```java
    public PackageFreezer freezePackageForInstall(String packageName, int userId, int installFlags,
            String killReason) {
        if ((installFlags & PackageManager.INSTALL_DONT_KILL_APP) != 0) {
            //【1】如果指定了卸载时不 kill app，就用默认的参数（卸载时才会进入）
            return new PackageFreezer();
        } else {
            //【2】默认是需要 kill app 的，调用 freezePackage！
            return freezePackage(packageName, userId, killReason);
        }
    }
```
继续调用 freezePackage 方法：
```java
    public PackageFreezer freezePackage(String packageName, int userId, String killReason) {
        //【×5.7.3.7.1】创建了一个 PackageFreezer 实例！
        return new PackageFreezer(packageName, userId, killReason);
    }
```
##### 5.7.3.7.1 new PackageFreezer

如果不杀进程，那就只创建一个 PackageFreezer 实例，并返回，不做任何事情！

```java
        public PackageFreezer() {
            mPackageName = null;
            mChildren = null;
            mWeFroze = false;
            mCloseGuard.open("close");
        }
```

如果要 kill app，那么会创建 PackageFreezer（父包和子包），并且会杀掉进程！

```java
        public PackageFreezer(String packageName, int userId, String killReason) {
            synchronized (mPackages) {
                mPackageName = packageName;
                mWeFroze = mFrozenPackages.add(mPackageName);

                final PackageSetting ps = mSettings.mPackages.get(mPackageName);
                if (ps != null) {
                    //【1】杀掉父包的进程!
                    killApplication(ps.name, ps.appId, userId, killReason);
                }

                final PackageParser.Package p = mPackages.get(packageName);
                if (p != null && p.childPackages != null) {
                    final int N = p.childPackages.size();
                    mChildren = new PackageFreezer[N];
                    for (int i = 0; i < N; i++) {
                        //【2】为子包创建 PackageFreezer，并杀掉子包的进程！
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
这里就不多说了！！

### 5.7.4 FileInstallArgs.doPostInstall

清理操作，正常情况不会触发！

```java
    int doPostInstall(int status, int uid) {
        //【1】如果安装前的状态不是 PackageManager.INSTALL_SUCCEEDED
        if (status != PackageManager.INSTALL_SUCCEEDED) {
            //【5.7.2.1】执行清理操作！
            cleanUp();
        }
        return status;
    }
```
### 5.7.5 new PostInstallData

又创建了一个 PostInstallData 对象，对 PackageInstalledInfo 做了再次封装：

```java
    static class PostInstallData {
        public InstallArgs args;
        public PackageInstalledInfo res;

        PostInstallData(InstallArgs _a, PackageInstalledInfo _r) {
            args = _a;
            res = _r;
        }
    }
```
继续来看！!

## 5.8 PackageHandler.doHandleMessage[POST_INSTALL] - 安装收尾

PackageHandler 会处理 POST_INSTALL 消息，此时已经处于安装收尾阶段：

```java
    case POST_INSTALL: {
        if (DEBUG_INSTALL) Log.v(TAG, "Handling post-install for " + msg.arg1);

        //【1】我们在安装开始前，会用 PostInstallData 封装安装结果，并保存到 mRunningInstalls 中！
        // 在安装结束后，会处理本次的安装结果！
        PostInstallData data = mRunningInstalls.get(msg.arg1);
        final boolean didRestore = (msg.arg2 != 0);
        mRunningInstalls.delete(msg.arg1);  // 删除掉！

        if (data != null) {
            InstallArgs args = data.args;
            PackageInstalledInfo parentRes = data.res;

            //【2】本次安装是否授予运行时权限！
            final boolean grantPermissions = (args.installFlags
                    & PackageManager.INSTALL_GRANT_RUNTIME_PERMISSIONS) != 0;

            //【2】本次安装是否 kill app！
            final boolean killApp = (args.installFlags
                    & PackageManager.INSTALL_DONT_KILL_APP) == 0;
            final String[] grantedPermissions = args.installGrantPermissions;

            //【×5.8.1】调用 handlePackagePostInstall 继续处理收尾工作！
            handlePackagePostInstall(parentRes, grantPermissions, killApp,
                    grantedPermissions, didRestore, args.installerPackageName,
                    args.observer);

            //【3】处理子包！
            final int childCount = (parentRes.addedChildPackages != null)
                    ? parentRes.addedChildPackages.size() : 0;
            for (int i = 0; i < childCount; i++) {
                PackageInstalledInfo childRes = parentRes.addedChildPackages.valueAt(i);
                
                //【×5.8.1】调用 handlePackagePostInstall 继续处理收尾工作！
                handlePackagePostInstall(childRes, grantPermissions, killApp,
                        grantedPermissions, false, args.installerPackageName,
                        args.observer);
            }

            if (args.traceMethod != null) {
                Trace.asyncTraceEnd(TRACE_TAG_PACKAGE_MANAGER, args.traceMethod,
                        args.traceCookie);
            }
        } else {
            Slog.e(TAG, "Bogus post-install token " + msg.arg1);
        }

        Trace.asyncTraceEnd(TRACE_TAG_PACKAGE_MANAGER, "postInstall", msg.arg1);
    } break;
```

继续处理！

### 5.8.1 PackageManagerS.handlePackagePostInstall

处理安装收尾工作：

```java
    private void handlePackagePostInstall(PackageInstalledInfo res, boolean grantPermissions,
            boolean killApp, String[] grantedPermissions,
            boolean launchedForRestore, String installerPackage,
            IPackageInstallObserver2 installObserver) {

        //【1】当安装结果为 success 后，会进入后续的处理！
        if (res.returnCode == PackageManager.INSTALL_SUCCEEDED) {
            //【1.1】如果是 move pacakge，那么发送 removed 广播！
            if (res.removedInfo != null) {
                res.removedInfo.sendPackageRemovedBroadcasts(killApp);
            }

            //【1.2】如果安装时指定了授予运行时权限，并且应用的目标 sdk 支持运行时权限，那就授予运行时权限！
            if (grantPermissions && res.pkg.applicationInfo.targetSdkVersion
                    >= Build.VERSION_CODES.M) {
                grantRequestedRuntimePermissions(res.pkg, res.newUsers, grantedPermissions);
            }

            //【1.3】判断安装方式，是更新安装，还是全新安装！
            // 我们知道，如果 res.removedInfo 不为 null 的话，一定是覆盖更新！
            final boolean update = res.removedInfo != null
                    && res.removedInfo.removedPackage != null;

            //【1.4】如果被 disable 的特权应用之前没有子包，这是第一次有子包，那么我们会授予新的子包
            // 运行时权限，如果旧的特权应用之前已经授予！
            if (res.pkg.parentPackage != null) {
                synchronized (mPackages) {
                    grantRuntimePermissionsGrantedToDisabledPrivSysPackageParentLPw(res.pkg);
                }
            }

            synchronized (mPackages) {
                mEphemeralApplicationRegistry.onPackageInstalledLPw(res.pkg);
            }

            final String packageName = res.pkg.applicationInfo.packageName;
            Bundle extras = new Bundle(1);
            extras.putInt(Intent.EXTRA_UID, res.uid);

            //【1.5】决定在那些 user 下是第一次安装，那些用户下是覆盖更新！
            int[] firstUsers = EMPTY_INT_ARRAY;
            int[] updateUsers = EMPTY_INT_ARRAY;
            if (res.origUsers == null || res.origUsers.length == 0) {
                firstUsers = res.newUsers;
            } else {
                // res.newUsers 表示本次安装新增加的目标 user！
                // res.origUsers 标志之前安装的目标 user！
                for (int newUser : res.newUsers) {
                    boolean isNew = true;
                    for (int origUser : res.origUsers) {
                        if (origUser == newUser) {
                            isNew = false;
                            break;
                        }
                    }
                    if (isNew) {
                        firstUsers = ArrayUtils.appendInt(firstUsers, newUser);
                    } else {
                        updateUsers = ArrayUtils.appendInt(updateUsers, newUser);
                    }
                }
            }

            //【1.5】发送 ADD 和 REPLACE 广播，如果不是 Ephemeral app！
            if (!isEphemeral(res.pkg)) {
                mProcessLoggingHandler.invalidateProcessLoggingBaseApkHash(res.pkg.baseCodePath);

                //【1.5.1】给第一次安装的用户发送 ACTION_PACKAGE_ADDED 广播，不带 EXTRA_REPLACING 属性！
                sendPackageBroadcast(Intent.ACTION_PACKAGE_ADDED, packageName,
                        extras, 0 /*flags*/, null /*targetPackage*/,
                        null /*finishedReceiver*/, firstUsers);

                //【1.5.2】给覆盖更新的用户发送 ACTION_PACKAGE_ADDED 广播，带 EXTRA_REPLACING 属性！
                if (update) {
                    extras.putBoolean(Intent.EXTRA_REPLACING, true);
                }
                sendPackageBroadcast(Intent.ACTION_PACKAGE_ADDED, packageName,
                        extras, 0 /*flags*/, null /*targetPackage*/,
                        null /*finishedReceiver*/, updateUsers);

                //【1.5.3】给覆盖更新的用户发送 ACTION_PACKAGE_REPLACED / ACTION_MY_PACKAGE_REPLACED 广播！
                if (update) {
                    sendPackageBroadcast(Intent.ACTION_PACKAGE_REPLACED,
                            packageName, extras, 0 /*flags*/,
                            null /*targetPackage*/, null /*finishedReceiver*/,
                            updateUsers);
                            
                    sendPackageBroadcast(Intent.ACTION_MY_PACKAGE_REPLACED,
                            null /*package*/, null /*extras*/, 0 /*flags*/,
                            packageName /*targetPackage*/,
                            null /*finishedReceiver*/, updateUsers);
                            
                } else if (launchedForRestore && !isSystemApp(res.pkg)) {
                    //【1.5.4】如果是第一次安装，同时我们要做一个恢复操作，并且 apk 不是系统应用
                    // 那么我们会发送 ACTION_PACKAGE_FIRST_LAUNCH 广播！
                    if (DEBUG_BACKUP) {
                        Slog.i(TAG, "Post-restore of " + packageName
                                + " sending FIRST_LAUNCH in " + Arrays.toString(firstUsers));
                    }
                    sendFirstLaunchBroadcast(packageName, installerPackage, firstUsers);
                }

                //【1.5.5】如果该 apk 处于 forward locked 或者处于外置存储中，那么会给所有的用户发送
                // 资源变化的广播！
                if (res.pkg.isForwardLocked() || isExternal(res.pkg)) {
                    if (DEBUG_INSTALL) {
                        Slog.i(TAG, "upgrading pkg " + res.pkg
                                + " is ASEC-hosted -> AVAILABLE");
                    }
                    final int[] uidArray = new int[]{res.pkg.applicationInfo.uid};
                    ArrayList<String> pkgList = new ArrayList<>(1);
                    pkgList.add(packageName);
                    sendResourcesChangedBroadcast(true, true, pkgList, uidArray, null);
                }
            }

            //【1.6】针对 firstUsers 做一些权限恢复和默认浏览器设置！
            if (firstUsers != null && firstUsers.length > 0) {
                synchronized (mPackages) {
                    for (int userId : firstUsers) {
                        //【1.6.1】默认浏览器设置！！
                        if (packageIsBrowser(packageName, userId)) {
                            mSettings.setDefaultBrowserPackageNameLPw(null, userId);
                        }
                        //【1.6.2】处理那些正在等待或者需要恢复的运行时权限授予！
                        mSettings.applyPendingPermissionGrantsLPw(packageName, userId);
                    }
                }
            }

            EventLog.writeEvent(EventLogTags.UNKNOWN_SOURCES_ENABLED,
                    getUnknownSourcesSettings());

            // 触发 GC 回收资源！
            Runtime.getRuntime().gc();

            //【5.8.1.1】移除掉更新后的旧 apk！
            if (res.removedInfo != null && res.removedInfo.args != null) {
                synchronized (mInstallLock) {
                    res.removedInfo.args.doPostDeleteLI(true);
                }
            }
        }

        //【*5.8.2】通知观察者安装的结果，这里的 installObserver 是我们之前创建的 localObsever！！
        if (installObserver != null) {
            try {
                Bundle extras = extrasForInstallResult(res);
                installObserver.onPackageInstalled(res.name, res.returnCode,
                        res.returnMsg, extras);
            } catch (RemoteException e) {
                Slog.i(TAG, "Observer no longer exists.");
            }
        }
    }
```

我们看到，这里涉及到了几个重要的广播：

- **Intent.ACTION_PACKAGE_ADDED**：当有应用程序第一次安装时，会发送该广播给对应的 firstUsers！
    - 携带数据：Intent.EXTRA_UID，值为 apk uid；

- **Intent.ACTION_PACKAGE_REPLACED**：当有应用程序被覆盖安装时，会发送该广播给对应的 updateUsers！
    - 携带数据：Intent.EXTRA_UID，
    - 携带数据：Intent.EXTRA_REPLACING，置为 true；

- **Intent.ACTION_MY_PACKAGE_REPLACED**：当有应用程序被覆盖安装时，会发送该广播给对应的 updateUsers 下被更新的 pkg！
    - 携带数据：packageName，被更新的应用包； 

#### 5.8.1.1 FileInstallArgs.doPostDeleteLI - 删除被更新的旧 apk

继续来看：

```java
    boolean doPostDeleteLI(boolean delete) {
        //【*5.8.1.2】清楚 apk 文件 和 dex 文件！
        cleanUpResourcesLI();
        return true;
    }
```
继续分析：

#### 5.8.1.2 FileInstallArgs.cleanUpResourcesLI

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
        //【2】清除 apk！
        cleanUp();
        //【3】清除 dex files！
        removeDexFiles(allCodePaths, instructionSets);
    }
```
这里就不再过多分析！


### 5.8.2 IPackageInstallObserver2.onPackageInstalled

这里的 installObserver 是我们在 4.3 PackageInstallerSession.commitLocked 中创建的另一个 Observer：

```java
    //【1】创建一个本地的安装观察者，监听安装结果！
    final IPackageInstallObserver2 localObserver = new IPackageInstallObserver2.Stub() {
        @Override
        public void onUserActionRequired(Intent intent) {
            throw new IllegalStateException();
        }
        @Override
        public void onPackageInstalled(String basePackageName, int returnCode, String msg,
                Bundle extras) {
            //【×5.8.2.1】关闭文件桥，删除文件拷贝！
            destroyInternal();
            //【×5.8.2.2】处理回调，通知监听者！
            dispatchSessionFinished(returnCode, msg, extras);
        }
    };
```
#### 5.8.2.1 PackageInstallerSession.destroyInternal

关闭文件桥，删除文件拷贝：

```java
    private void destroyInternal() {
        synchronized (mLock) {
            mSealed = true;
            mDestroyed = true;

            //【1】关闭之前打开的文件桥对象！
            for (FileBridge bridge : mBridges) {
                bridge.forceClose();
            }
        }
        if (stageDir != null) {
            try {
                //【2】删除之前的文件拷贝目录！
                mPm.mInstaller.rmPackageDir(stageDir.getAbsolutePath());
            } catch (InstallerException ignored) {
            }
        }
        if (stageCid != null) {
            PackageHelper.destroySdDir(stageCid);
        }
    }
```

#### 5.8.2.2 PackageInstallerSession.dispatchSessionFinished

处理回调，通知监听者！

```java
    private void dispatchSessionFinished(int returnCode, String msg, Bundle extras) {
        mFinalStatus = returnCode;
        mFinalMessage = msg;
        //【*4.1.2】触发 mRemoteObserver 回调！
        if (mRemoteObserver != null) {
            try {
                mRemoteObserver.onPackageInstalled(mPackageName, returnCode, msg, extras);
            } catch (RemoteException ignored) {
            }
        }

        final boolean success = (returnCode == PackageManager.INSTALL_SUCCEEDED);
        //【*5.8.2.2.2】回调触发！！
        mCallback.onSessionFinished(this, success);
    }
```
在前面 4.1 commit 事务的时候，我们创建了一个 PackageInstallObserverAdapter，并将其保存到了 PackageInstallerSession.mRemoteObserver 中，这里首先会触发 mRemoteObserver 的回调！


在 3.1.4.1 new InternalCallback 的时候，我们在创建 PackageInstallerSession 时，传入了一个回调对象 InternalCallback：

```java
private final InternalCallback mInternalCallback = new InternalCallback();
```
InternalCallback 类定义在 PackageInstallerService 中，该对象的引用会被保存到 PackageInstallerSession.mCallback 变量中！

##### 5.8.2.2.2 InternalCallback.onSessionFinished

这里我们重点关于 onSessionFinished 方法：

```java
    public void onSessionFinished(final PackageInstallerSession session, boolean success) {
        //【×5.8.2.2.2.1】注册者回调！
        mCallbacks.notifySessionFinished(session.sessionId, session.userId, success);

        mInstallHandler.post(new Runnable() {
            @Override
            public void run() {
                synchronized (mSessions) {
                    //【1】从 PackageInstallerService 中的 mSessions 移除了该 Session；
                    mSessions.remove(session.sessionId);
                    //【2】将该 Sesssion 添加到历史中；
                    mHistoricalSessions.put(session.sessionId, session);

                    final File appIconFile = buildAppIconFile(session.sessionId);
                    if (appIconFile.exists()) {
                        appIconFile.delete();
                    }
                    //【×3.1.6.1】持久化事务记录文件！
                    writeSessionsLocked();
                }
            }
        });
    }
```
我们看到，在 InternalCallback 中又回调了另外一个 Callback mCallbacks，它也是 PackageInstallerService 的内部类：

###### 5.8.2.2.2.1 Callback.notifySessionFinished

前面我们分析过，Callback 本质上就是一个 Handler，这里就是向其所在的线程发送消息：
```java
    public void notifySessionFinished(int sessionId, int userId, boolean success) {
        obtainMessage(MSG_SESSION_FINISHED, sessionId, userId, success).sendToTarget();
    }
```
在 3.1.4.2 Callbacks.notifySessionXXXX 中，我们分析过，最终其实是很将安装的结果分发给了注册在 Callback 中的所有远程回调：

```java
        private final RemoteCallbackList<IPackageInstallerCallback>
                mCallbacks = new RemoteCallbackList<>();
```
这里就不多说了！

# 6 PackageManagerS.replacePackageLIF - 覆盖安装

## 6.1 参数分析

这里我们来回顾下传入的参数：final int policyFlags 就是我们之前的解析参数 parseFlags

```java
    // 默认情况下： mDefParseFlags = 0 
    final int parseFlags = mDefParseFlags | PackageParser.PARSE_CHATTY
            | PackageParser.PARSE_ENFORCE_CODE
            | (forwardLocked ? PackageParser.PARSE_FORWARD_LOCK : 0)
            | (onExternal ? PackageParser.PARSE_EXTERNAL_STORAGE : 0)
            | (ephemeral ? PackageParser.PARSE_IS_EPHEMERAL : 0)
            | (forceSdk ? PackageParser.PARSE_FORCE_SDK : 0);
```
同时，对于扫描标志位 scanFlags，会做如下处理：
```java
    //【2】设置扫描参数
    int scanFlags = SCAN_NEW_INSTALL | SCAN_UPDATE_SIGNATURE;
    if (args.move != null) {
    	//【2.1】如果 args.move 不为 null，表示正在移动一个 app，我们会对其进行一个初始化的扫描
    	// 增加 SCAN_INITIAL 位！
    	scanFlags |= SCAN_INITIAL;
    }
    if ((installFlags & PackageManager.INSTALL_DONT_KILL_APP) != 0) { 
    	//【2.2】如果安装参数指定了 INSTALL_DONT_KILL_APP，那么增加 SCAN_DONT_KILL_APP 位！
    	scanFlags |= SCAN_DONT_KILL_APP;
    }
    ... ...
    //【13】根据安装参数做不同的处理！
    if (args.move != null) {
    	//【13.1】如果是 move package，进入这里！
    	scanFlags |= SCAN_NO_DEX; // 设置以下标签，无需做 odex，我们需要已有的移动过去即可！
    	scanFlags |= SCAN_MOVE;
    	... ... ...
    } else if (!forwardLocked && !pkg.applicationInfo.isExternalAsec()) {
    	//【13.2】如果不是 forward lock 模式安装且没有安装到外置存储上，进入这里！
    	scanFlags |= SCAN_NO_DEX; // 扫描参数设置 SCAN_NO_DEX，意味着后面不做 odex，因为这里会做！
    	... ... ...
    }
```

上面，我们省略掉了不重要的代码段！

## 6.2 方法分析

下面继续分析核心方法：

```java
    private void replacePackageLIF(PackageParser.Package pkg, final int policyFlags, int scanFlags,
            UserHandle user, String installerPackageName, PackageInstalledInfo res) {
        final boolean isEphemeral = (policyFlags & PackageParser.PARSE_IS_EPHEMERAL) != 0; // 是否是 ephemeral 

        final PackageParser.Package oldPackage;
        final String pkgName = pkg.packageName;
        final int[] allUsers;
        final int[] installedUsers;

        synchronized(mPackages) {
            //【1】获得旧应用的 Package 对象！
            oldPackage = mPackages.get(pkgName);
            if (DEBUG_INSTALL) Slog.d(TAG, "replacePackageLI: new=" + pkg + ", old=" + oldPackage);

            // don't allow upgrade to target a release SDK from a pre-release SDK
            final boolean oldTargetsPreRelease = oldPackage.applicationInfo.targetSdkVersion
                    == android.os.Build.VERSION_CODES.CUR_DEVELOPMENT;
            final boolean newTargetsPreRelease = pkg.applicationInfo.targetSdkVersion
                    == android.os.Build.VERSION_CODES.CUR_DEVELOPMENT;
            //【2】如果安装设置了 PackageParser.PARSE_FORCE_SDK，那就要校验下 sdk！
            if (oldTargetsPreRelease
                    && !newTargetsPreRelease
                    && ((policyFlags & PackageParser.PARSE_FORCE_SDK) == 0)) {
                Slog.w(TAG, "Can't install package targeting released sdk");
                res.setReturnCode(PackageManager.INSTALL_FAILED_UPDATE_INCOMPATIBLE);
                return;
            }

            //【3】如果之前安装是 no ephemeral，现在是 ephemeral，那么不能安装！
            final boolean oldIsEphemeral = oldPackage.applicationInfo.isEphemeralApp();
            if (isEphemeral && !oldIsEphemeral) {
                Slog.w(TAG, "Can't replace app with ephemeral: " + pkgName);
                res.setReturnCode(PackageManager.INSTALL_FAILED_EPHEMERAL_INVALID);
                return;
            }

            //【4】校验签名，不匹配，不能安装！
            final PackageSetting ps = mSettings.mPackages.get(pkgName);
            if (shouldCheckUpgradeKeySetLP(ps, scanFlags)) {
                if (!checkUpgradeKeySetLP(ps, pkg)) {
                    res.setError(INSTALL_FAILED_UPDATE_INCOMPATIBLE,
                            "New package not signed by keys specified by upgrade-keysets: "
                                    + pkgName);
                    return;
                }
            } else {
                if (compareSignatures(oldPackage.mSignatures, pkg.mSignatures)
                        != PackageManager.SIGNATURE_MATCH) {
                    res.setError(INSTALL_FAILED_UPDATE_INCOMPATIBLE,
                            "New package has a different signature: " + pkgName);
                    return;
                }
            }

            //【5】如果旧的应用是 sys app，并且需要强制 hash 校验，那就会在这里校验 hash！
            if (oldPackage.restrictUpdateHash != null && oldPackage.isSystemApp()) {
                byte[] digestBytes = null;
                try {
                    final MessageDigest digest = MessageDigest.getInstance("SHA-512");
                    updateDigest(digest, new File(pkg.baseCodePath));
                    if (!ArrayUtils.isEmpty(pkg.splitCodePaths)) {
                        for (String path : pkg.splitCodePaths) {
                            updateDigest(digest, new File(path));
                        }
                    }
                    digestBytes = digest.digest();
                } catch (NoSuchAlgorithmException | IOException e) {
                    res.setError(INSTALL_FAILED_INVALID_APK,
                            "Could not compute hash: " + pkgName);
                    return;
                }
                if (!Arrays.equals(oldPackage.restrictUpdateHash, digestBytes)) {
                    res.setError(INSTALL_FAILED_INVALID_APK,
                            "New package fails restrict-update check: " + pkgName);
                    return;
                }
                // retain upgrade restriction
                pkg.restrictUpdateHash = oldPackage.restrictUpdateHash;
            }

            //【6】检查 shareUserId 是否变化！
            String invalidPackageName =
                    getParentOrChildPackageChangedSharedUser(oldPackage, pkg);
            if (invalidPackageName != null) {
                res.setError(INSTALL_FAILED_SHARED_USER_INCOMPATIBLE,
                        "Package " + invalidPackageName + " tried to change user "
                                + oldPackage.mSharedUserId);
                return;
            }

            //【7】获得系统中所有的 user，已经上一次安装的目标 user！
            allUsers = sUserManager.getUserIds();
            installedUsers = ps.queryInstalledUsers(allUsers, true);
        }

        //【×5.7.1.1】针对与 replace 的情况，会创建一个 PackageRemovedInfo，封装被 replace 的 app 的信息！
        // 如果有子包的话，也会做相同的事情！
        res.removedInfo = new PackageRemovedInfo();
        res.removedInfo.uid = oldPackage.applicationInfo.uid;
        res.removedInfo.removedPackage = oldPackage.packageName;
        res.removedInfo.isUpdate = true; // 表示要更新 package！！
        res.removedInfo.origUsers = installedUsers;

        final int childCount = (oldPackage.childPackages != null)
                ? oldPackage.childPackages.size() : 0;
        for (int i = 0; i < childCount; i++) {
            boolean childPackageUpdated = false;
            PackageParser.Package childPkg = oldPackage.childPackages.get(i);
            if (res.addedChildPackages != null) {
                PackageInstalledInfo childRes = res.addedChildPackages.get(childPkg.packageName);
                if (childRes != null) {
                    childRes.removedInfo.uid = childPkg.applicationInfo.uid;
                    childRes.removedInfo.removedPackage = childPkg.packageName;
                    childRes.removedInfo.isUpdate = true;
                    childPackageUpdated = true;
                }
            }
            if (!childPackageUpdated) {
                PackageRemovedInfo childRemovedRes = new PackageRemovedInfo();
                childRemovedRes.removedPackage = childPkg.packageName;
                childRemovedRes.isUpdate = false;
                childRemovedRes.dataRemoved = true;
                synchronized (mPackages) {
                    PackageSetting childPs = mSettings.peekPackageLPr(childPkg.packageName);
                    if (childPs != null) {
                        childRemovedRes.origUsers = childPs.queryInstalledUsers(allUsers, true);
                    }
                }
                if (res.removedInfo.removedChildPackages == null) {
                    res.removedInfo.removedChildPackages = new ArrayMap<>();
                }
                res.removedInfo.removedChildPackages.put(childPkg.packageName, childRemovedRes);
            }
        }
        
        //【9】判断下旧应用是系统应用，还是非系统应用，然后做不同的处理！
        boolean sysPkg = (isSystemApp(oldPackage));
        if (sysPkg) {
            //【9.1】如果是系统应用，再判断是是否是系统特权应用！
            final boolean privileged =
                    (oldPackage.applicationInfo.privateFlags
                            & ApplicationInfo.PRIVATE_FLAG_PRIVILEGED) != 0;
            //【9.2】对于系统应用会增加如下的解析 flags!
            final int systemPolicyFlags = policyFlags
                    | PackageParser.PARSE_IS_SYSTEM
                    | (privileged ? PackageParser.PARSE_IS_PRIVILEGED : 0);

            //【×6.2.1】处理系统应用的 replace！
            replaceSystemPackageLIF(oldPackage, pkg, systemPolicyFlags, scanFlags,
                    user, allUsers, installerPackageName, res);
        } else {
            //【×6.2.2】处理下非系统应用的 replace！
            replaceNonSystemPackageLIF(oldPackage, pkg, policyFlags, scanFlags,
                    user, allUsers, installerPackageName, res);
        }
    }
```
对于判断是否是系统应用，调用了如下的接口：

```java
    private static boolean isSystemApp(PackageParser.Package pkg) {
        return (pkg.applicationInfo.flags & ApplicationInfo.FLAG_SYSTEM) != 0;
    }
```

### 6.2.1 replaceSystemPackageLIF - 覆盖安装系统应用

这里我们来回顾下传入的参数：final int policyFlags 就是我们之前的解析参数 parseFlags

```java
    //【1】默认情况下： mDefParseFlags = 0 
    final int parseFlags = mDefParseFlags | PackageParser.PARSE_CHATTY
            | PackageParser.PARSE_ENFORCE_CODE
            | (forwardLocked ? PackageParser.PARSE_FORWARD_LOCK : 0)
            | (onExternal ? PackageParser.PARSE_EXTERNAL_STORAGE : 0)
            | (ephemeral ? PackageParser.PARSE_IS_EPHEMERAL : 0)
            | (forceSdk ? PackageParser.PARSE_FORCE_SDK : 0);
 
    //【2】针对系统应用额外增加如下的标志       
    final int systemPolicyFlags = policyFlags
        | PackageParser.PARSE_IS_SYSTEM
        | (privileged ? PackageParser.PARSE_IS_PRIVILEGED : 0);        
```
同时，对于扫描标志位 scanFlags，和上面保持一致，下面我们来分析下更新 sys app 的流程！

```java
    private void replaceSystemPackageLIF(PackageParser.Package deletedPackage,
            PackageParser.Package pkg, final int policyFlags, int scanFlags, UserHandle user,
            int[] allUsers, String installerPackageName, PackageInstalledInfo res) {
        if (DEBUG_INSTALL) Slog.d(TAG, "replaceSystemPackageLI: new=" + pkg
                + ", old=" + deletedPackage);
        final boolean disabledSystem;

        //【*6.2.1.1】移除掉已安装的 sys app 的数据，包括解析到的四大组件；
        removePackageLI(deletedPackage, true);

        synchronized (mPackages) {
            //【*6.2.1.2】disable 掉系统应用；
            disabledSystem = disableSystemPackageLPw(deletedPackage, pkg);
        }
        
        //【1】根据 disabledSystem 的值做不同处理！
        if (!disabledSystem) {
            //【1.1】如果 disabledSystem 为 false，说明我们之前已经更新过 sys app 了，其已经被 disable 过了
            // 我们现在更新的是位于 data 分区的那个 app，所以要删除掉那个旧的 data app！
            //【*6.2.1.3】这里会根据要删除 data 分区的 apk 的数据，创建一个 InstallArgs，用于执行删除操作！
            res.removedInfo.args = createInstallArgsForExisting(0,
                    deletedPackage.applicationInfo.getCodePath(),
                    deletedPackage.applicationInfo.getResourcePath(),
                    getAppDexInstructionSets(deletedPackage.applicationInfo));

        } else {
            //【1.2】此时无需删除 app，所以设置为 null
            res.removedInfo.args = null;
        }

        //【*6.2.1.4】清楚掉新安装的应用 code_cache 目录下的文件数据；
        clearAppDataLIF(pkg, UserHandle.USER_ALL, StorageManager.FLAG_STORAGE_DE
                | StorageManager.FLAG_STORAGE_CE | Installer.FLAG_CLEAR_CODE_CACHE_ONLY);

        //【*6.2.1.5】清楚掉旧的 app 的 profile 相关数据；
        clearAppProfilesLIF(deletedPackage, UserHandle.USER_ALL);

        res.setReturnCode(PackageManager.INSTALL_SUCCEEDED);

        //【2】给本次扫描的新 app 设置 ApplicationInfo.FLAG_UPDATED_SYSTEM_APP 标志位！
        pkg.setApplicationInfoFlags(ApplicationInfo.FLAG_UPDATED_SYSTEM_APP,
                ApplicationInfo.FLAG_UPDATED_SYSTEM_APP);

        PackageParser.Package newPackage = null;
        try {
            //【3】再次扫描 apk 文件！！
            newPackage = scanPackageTracedLI(pkg, policyFlags, scanFlags, 0, user);

            //【4】更新新 apk 的安装时间和最新更新时间（包括子包的）；
            PackageSetting deletedPkgSetting = (PackageSetting) deletedPackage.mExtras;
            setInstallAndUpdateTime(newPackage, deletedPkgSetting.firstInstallTime,
                    System.currentTimeMillis());

            //【5】处理安装成功的情况！
            if (res.returnCode == PackageManager.INSTALL_SUCCEEDED) {
                final int deletedChildCount = (deletedPackage.childPackages != null)
                        ? deletedPackage.childPackages.size() : 0;
                final int newChildCount = (newPackage.childPackages != null)
                        ? newPackage.childPackages.size() : 0;
    
                //【5.1】比较旧的应用和新安装的应用的子包！
                for (int i = 0; i < deletedChildCount; i++) {
                    PackageParser.Package deletedChildPkg = deletedPackage.childPackages.get(i);
                    boolean childPackageDeleted = true;
                    for (int j = 0; j < newChildCount; j++) {
                        PackageParser.Package newChildPkg = newPackage.childPackages.get(j);
                        if (deletedChildPkg.packageName.equals(newChildPkg.packageName)) {
                            childPackageDeleted = false;
                            break;
                        }
                    }
                    //【5.1.1】如果旧的应用的子包被删除了，那会移除旧的应用子包遗留下的数据！
                    if (childPackageDeleted) {
                        PackageSetting ps = mSettings.getDisabledSystemPkgLPr(
                                deletedChildPkg.packageName);
                        if (ps != null && res.removedInfo.removedChildPackages != null) {
                            PackageRemovedInfo removedChildRes = res.removedInfo
                                    .removedChildPackages.get(deletedChildPkg.packageName);
                            // 移除数据！
                            removePackageDataLIF(ps, allUsers, removedChildRes, 0, false);
                            removedChildRes.removedForAllUsers = mPackages.get(ps.name) == null;
                        }
                    }
                }
                //【*7.2.1】更新数据；
                updateSettingsLI(newPackage, installerPackageName, allUsers, res, user);
                //【*7.2.2】准备数据目录，即 data/data/packageName 的目录；
                prepareAppDataAfterInstallLIF(newPackage);
            }
        } catch (PackageManagerException e) {
            res.setReturnCode(INSTALL_FAILED_INTERNAL_ERROR);
            res.setError("Package couldn't be installed in " + pkg.codePath, e);
        }

        //【3】处理安装失败的情况！
        if (res.returnCode != PackageManager.INSTALL_SUCCEEDED) {
            if (newPackage != null) {
                //【*6.2.1.6】移除新解析的 apk 的数据
                removeInstalledPackageLI(newPackage, true);
            }

            try {
                //【3.1】重新扫描旧的 apk（注意如果之前 system apk 已经被更新过了，那么这里
                // 恢复的是处于 data 的那个新的 apk）！
                scanPackageTracedLI(deletedPackage, policyFlags, SCAN_UPDATE_SIGNATURE, 0, user);
            } catch (PackageManagerException e) {
                Slog.e(TAG, "Failed to restore original package: " + e.getMessage());
            }

            synchronized (mPackages) {
                if (disabledSystem) {
                    //【*6.2.1.7】如果本次是第一次更新（即 disable 了 sys app），那么我们要 enable sys app！
                    enableSystemPackageLPw(deletedPackage);
                }

                //【3.2】更新升级包的安装信息，就是 PackageInstaller！
                setInstallerPackageNameLPw(deletedPackage, installerPackageName);

                //【3.3】更新权限信息，关于权限的相关逻辑，这里不再分析，但是能推测到更新权限的原因！
                updatePermissionsLPw(deletedPackage, UPDATE_PERMISSIONS_ALL);
                
                //【3.4】持久化数据！
                mSettings.writeLPr();
            }

            Slog.i(TAG, "Successfully restored package : " + deletedPackage.packageName
                    + " after failed upgrade");
        }
    }
```
这里就不多说了！！


#### 6.2.1.1 removePackageLI

移除旧的 apk 的数据信息：

```java
    private void removePackageLI (PackageParser.Package pkg, boolean chatty) {
        //【1】处理父包！
        PackageSetting ps = (PackageSetting) pkg.mExtras;
        if (ps != null) {
            //【1.1】移除父包的 Settings 对象的 Settings 数据！
            removePackageLI(ps, chatty);
        }
        //【2】处理子包！
        final int childCount = (pkg.childPackages != null) ? pkg.childPackages.size() : 0;
        for (int i = 0; i < childCount; i++) {
            PackageParser.Package childPkg = pkg.childPackages.get(i);
            ps = (PackageSetting) childPkg.mExtras;
            if (ps != null) {
                //【2.1】移除子包的 Settings 对象的 Settings 数据！
                removePackageLI(ps, chatty);
            }
        }
    }
```
调用了另外一个 removePackageLI 方法！
```java
    void removePackageLI(PackageSetting ps, boolean chatty) {
        if (DEBUG_INSTALL) {
            if (chatty)
                Log.d(TAG, "Removing package " + ps.name);
        }

        synchronized (mPackages) {
            //【1】将应用对应的 PackageSettings 对象删除掉！
            mPackages.remove(ps.name);
            final PackageParser.Package pkg = ps.pkg;
            if (pkg != null) {
                //【×6.2.1.1.1】移除四大组件数据！
                cleanPackageDataStructuresLILPw(pkg, chatty);
            }
        }
    }
```

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


#### 6.2.1.2 disableSystemPackageLPw

disable 掉系统应用

```java
    private boolean disableSystemPackageLPw(PackageParser.Package oldPkg,
            PackageParser.Package newPkg) {
        //【6.2.1.2.1】disable 掉父包，父包是要通过 replace 方式！
        boolean disabled = mSettings.disableSystemPackageLPw(oldPkg.packageName, true);
        // Disable the child packages
        final int childCount = (oldPkg.childPackages != null) ? oldPkg.childPackages.size() : 0;
        for (int i = 0; i < childCount; i++) {
            PackageParser.Package childPkg = oldPkg.childPackages.get(i);
            final boolean replace = newPkg.hasChildPackage(childPkg.packageName);
            //【6.2.1.2.1】disable 掉子包，如果新包也有子包，那就通过 replace 方式！
            disabled |= mSettings.disableSystemPackageLPw(childPkg.packageName, replace);
        }
        return disabled;
    }
```
这里调用了 Settings 的 disableSystemPackageLPw 方法！

##### 6.2.1.2.1 Settings.disableSystemPackageLPw

我们继续分析：

```java
    boolean disableSystemPackageLPw(String name, boolean replaced) {
        final PackageSetting p = mPackages.get(name);
        //【1】异常判断，如果没安装，disable 个屁！
        if(p == null) {
            Log.w(PackageManagerService.TAG, "Package " + name + " is not an installed package");
            return false;
        }
        //【2】尝试从 mDisabledSysPackages 列表中获得 PackageSetting！
        final PackageSetting dp = mDisabledSysPackages.get(name);
        
        //【3】判断 disable 的条件是否满足：
        // 1、之前没有 disable；2、是系统应用；3、没有更新过！
        if (dp == null && p.pkg != null && p.pkg.isSystemApp() && !p.pkg.isUpdatedSystemApp()) {
            //【3.1】设置 ApplicationInfo.FLAG_UPDATED_SYSTEM_APP 标志位，表示更新过的 sys app！
            if((p.pkg != null) && (p.pkg.applicationInfo != null)) {
                p.pkg.applicationInfo.flags |= ApplicationInfo.FLAG_UPDATED_SYSTEM_APP;
            }
            //【3.2】将其旧应用的 PackageSetting 添加到 mDisabledSysPackages 中！
            mDisabledSysPackages.put(name, p);

            //【3.4】判断是否是 replace，显然，对于 base apk 是需要 replace 的，对于 child apk，
            // 需要看新安装的应用是否有相应的子包！
            if (replaced) {
                PackageSetting newp = new PackageSetting(p); // copy 一份旧数据
                //【3.5】用 copy 后的数据替换之前的数据！
                replacePackageLPw(name, newp);
            }
            return true;
        }
        return false;
    }
```

可以看到，对于系统应用来说，只有第一次覆盖更新时，会 disable 掉 sys 下的那个 app；如果多次覆盖安装，后续的不会再 disable！

下面是替换的具体操作！

```java
    private void replacePackageLPw(String name, PackageSetting newp) {
        //【1】获得之前数据，然后根据 uid 的不同做不同的处理！
        final PackageSetting p = mPackages.get(name);
        if (p != null) {
            if (p.sharedUser != null) {
                p.sharedUser.removePackage(p);
                p.sharedUser.addPackage(newp);
            } else {
                replaceUserIdLPw(p.appId, newp);
            }
        }
        //【2】更新 mPackages 中的数据！
        mPackages.put(name, newp);
    }

```
旧的数据，此时是在 mDisabledSysPackages 中！

#### 6.2.1.3 createInstallArgsForExisting - 要删除的被更新 apk

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
            //【5.5.3.1】一般情况下，会创建 FileInstallArgs，这里通过 FileInstallArgs 的另一构造器
            // 创建了实例，描述一个已经存在的 app！
            return new FileInstallArgs(codePath, resourcePath, instructionSets);
        }
    }
```
这里又回到了 5.5.3.1 的 FileInstallArgs 的相关创建！

#### 6.2.1.4 clearAppDataLIF

清楚 app 的数据：

```java
    private void clearAppDataLIF(PackageParser.Package pkg, int userId, int flags) {
        if (pkg == null) {
            Slog.wtf(TAG, "Package was null!", new Throwable());
            return;
        }
        //【1】处理父包！
        clearAppDataLeafLIF(pkg, userId, flags);
        final int childCount = (pkg.childPackages != null) ? pkg.childPackages.size() : 0;
        for (int i = 0; i < childCount; i++) {
            //【2】调用另外一个方法处理子包！
            clearAppDataLeafLIF(pkg.childPackages.get(i), userId, flags);
        }
    }
```
继续看：
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

#### 6.2.1.5 clearAppProfilesLIF

清楚 app 的 profiles 数据：

```java
    private void clearAppProfilesLIF(PackageParser.Package pkg, int userId) {
        if (pkg == null) {
            Slog.wtf(TAG, "Package was null!", new Throwable());
            return;
        }
        //【×6.2.1.5.1】清除父包的 profile 数据！
        clearAppProfilesLeafLIF(pkg);

        // We don't remove the base foreign use marker when clearing profiles because
        // we will rename it when the app is updated. Unlike the actual profile contents,
        // the foreign use marker is good across installs.
        destroyAppReferenceProfileLeafLIF(pkg, userId, false /* removeBaseMarker */);
        final int childCount = (pkg.childPackages != null) ? pkg.childPackages.size() : 0;
        for (int i = 0; i < childCount; i++) {
            //【×6.2.1.5.1】清除子包的！
            clearAppProfilesLeafLIF(pkg.childPackages.get(i));
        }
    }
```

##### 6.2.1.5.1 clearAppProfilesLeafLIF

最终调用了这个方法，清楚 profile 数据：
```java
    private void clearAppProfilesLeafLIF(PackageParser.Package pkg) {
        try {
            mInstaller.clearAppProfiles(pkg.packageName);
        } catch (InstallerException e) {
            Slog.w(TAG, String.valueOf(e));
        }
    }
```

#### 6.2.1.6 removeInstalledPackageLI

移除被扫描到的新的 apk 的数据：

```java
    void removeInstalledPackageLI(PackageParser.Package pkg, boolean chatty) {
        if (DEBUG_INSTALL) {
            if (chatty)
                Log.d(TAG, "Removing package " + pkg.applicationInfo.packageName);
        }

        // writer
        synchronized (mPackages) {
            //【1】移除父包!
            mPackages.remove(pkg.applicationInfo.packageName);
            //【*6.2.1.1.1】移除父包的数据结构！
            cleanPackageDataStructuresLILPw(pkg, chatty);

            // Remove the child packages
            final int childCount = (pkg.childPackages != null) ? pkg.childPackages.size() : 0;
            for (int i = 0; i < childCount; i++) {
                //【2】移除子包!
                PackageParser.Package childPkg = pkg.childPackages.get(i);
                mPackages.remove(childPkg.applicationInfo.packageName);
                //【*6.2.1.1.1】移除子包的数据结构！
                cleanPackageDataStructuresLILPw(childPkg, chatty);
            }
        }
    }
```

#### 6.2.1.7 enableSystemPackageLPw

恢复系统应用

```java
    private void enableSystemPackageLPw(PackageParser.Package pkg) {
        //【1】恢复父包！
        mSettings.enableSystemPackageLPw(pkg.packageName);

        final int childCount = (pkg.childPackages != null) ? pkg.childPackages.size() : 0;
        for (int i = 0; i < childCount; i++) {
            PackageParser.Package childPkg = pkg.childPackages.get(i);
            //【*6.2.1.7.1】恢复子包！
            mSettings.enableSystemPackageLPw(childPkg.packageName);
        }
    }
```
##### 6.2.1.7.1 Settings.enableSystemPackageLPw

最终会调用 Settings 的 enableSystemPackageLPw 方法 enable package：

```java
    PackageSetting enableSystemPackageLPw(String name) {
        //【1】判断该 sys package 是否被 disable 了；
        PackageSetting p = mDisabledSysPackages.get(name);
        if(p == null) {
            Log.w(PackageManagerService.TAG, "Package " + name + " is not disabled");
            return null;
        }
        //【2】取消 ApplicationInfo.FLAG_UPDATED_SYSTEM_APP 标志位！
        if((p.pkg != null) && (p.pkg.applicationInfo != null)) {
            p.pkg.applicationInfo.flags &= ~ApplicationInfo.FLAG_UPDATED_SYSTEM_APP;
        }
        //【3】重新为该 system 创建一个 PackageSetting 对象，并添加到相应的集合中！
        PackageSetting ret = addPackageLPw(name, p.realName, p.codePath, p.resourcePath,
                p.legacyNativeLibraryPathString, p.primaryCpuAbiString,
                p.secondaryCpuAbiString, p.cpuAbiOverrideString,
                p.appId, p.versionCode, p.pkgFlags, p.pkgPrivateFlags,
                p.parentPackageName, p.childPackageNames);
        //【4】从 mDisabledSysPackages 中删除！
        mDisabledSysPackages.remove(name);
        return ret;
    }
```
关于 addPackageLPw 的逻辑这里就不在分析了！


### 6.2.2 replaceNonSystemPackageLIF - 覆盖安装三方应用 -> 接入 8 

关于 replaceNonSystemPackageLIF 的逻辑，由于 markdown 不支持 6 级以上的标题，所以移动到第 8 节，单独分析!!


# 7 PackageManagerS.installNewPackageLIF - 全新安装

## 7.1 参数分析

这里我们来回顾下传入的参数：final int policyFlags 就是我们之前的解析参数 parseFlags

```java
        //【1】默认情况下： mDefParseFlags = 0 
        final int parseFlags = mDefParseFlags | PackageParser.PARSE_CHATTY
                | PackageParser.PARSE_ENFORCE_CODE
                | (forwardLocked ? PackageParser.PARSE_FORWARD_LOCK : 0)
                | (onExternal ? PackageParser.PARSE_EXTERNAL_STORAGE : 0)
                | (ephemeral ? PackageParser.PARSE_IS_EPHEMERAL : 0)
                | (forceSdk ? PackageParser.PARSE_FORCE_SDK : 0);
```
同时，对于扫描标志位 scanFlags，会做如下处理：
```java
    //【2】设置扫描参数
    int scanFlags = SCAN_NEW_INSTALL | SCAN_UPDATE_SIGNATURE;
    if (args.move != null) {
    	//【2.1】如果 args.move 不为 null，表示正在移动一个 app，我们会对其进行一个初始化的扫描
    	// 增加 SCAN_INITIAL 位！
    	scanFlags |= SCAN_INITIAL;
    }
    if ((installFlags & PackageManager.INSTALL_DONT_KILL_APP) != 0) { 
    	//【2.2】如果安装参数指定了 INSTALL_DONT_KILL_APP，那么增加 SCAN_DONT_KILL_APP 位！
    	scanFlags |= SCAN_DONT_KILL_APP;
    }
    ... ... ...
    //【13】根据安装参数做不同的处理！
    if (args.move != null) {
    	//【13.1】如果是 move package，进入这里！
    	scanFlags |= SCAN_NO_DEX; // 设置以下标签，无需做 odex，我们需要已有的移动过去即可！
    	scanFlags |= SCAN_MOVE;
    	... ... ...
    } else if (!forwardLocked && !pkg.applicationInfo.isExternalAsec()) {
    	//【13.2】如果不是 forward lock 模式安装且没有安装到外置存储上，进入这里！
    	scanFlags |= SCAN_NO_DEX; // 扫描参数设置 SCAN_NO_DEX，意味着后面不做 odex，因为这里会做！
    	... ... ...
    }
```
上面我们省略掉了不重要的代码段！

## 7.2 方法解析

下面继续分析核心方法：

```java
    private void installNewPackageLIF(PackageParser.Package pkg, final int policyFlags,
            int scanFlags, UserHandle user, String installerPackageName, String volumeUuid,
            PackageInstalledInfo res) {
        Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "installNewPackage");

        // Remember this for later, in case we need to rollback this install
        String pkgName = pkg.packageName;

        if (DEBUG_INSTALL) Slog.d(TAG, "installNewPackageLI: " + pkg);

        synchronized(mPackages) {
            //【1】如果已经有一个同名的应用安装了，并且其被改为的旧的名字，那么这种情况不能用该接口安装，
            // 需要使用 replace 接口来更新！
            if (mSettings.mRenamedPackages.containsKey(pkgName)) {
                res.setError(INSTALL_FAILED_ALREADY_EXISTS, "Attempt to re-install " + pkgName
                        + " without first uninstalling package running as "
                        + mSettings.mRenamedPackages.get(pkgName));
                return;
            }
            //【2】如果 PMS.mPackages 中已经扫描到同名的应用，这种情况不能用该接口安装，需要通过 replace 接口！
            if (mPackages.containsKey(pkgName)) {
                // Don't allow installation over an existing package with the same name.
                res.setError(INSTALL_FAILED_ALREADY_EXISTS, "Attempt to re-install " + pkgName
                        + " without first uninstalling.");
                return;
            }
        }

        try {
            //【3】核心方法，扫描 apk，这里和 PMS 开机的时候一样！
            PackageParser.Package newPackage = scanPackageTracedLI(pkg, policyFlags, scanFlags,
                    System.currentTimeMillis(), user);
            //【*7.2.1】更新 Settings 中对应的 PackageSettings 集合！
            updateSettingsLI(newPackage, installerPackageName, null, res, user);

            if (res.returnCode == PackageManager.INSTALL_SUCCEEDED) {
                //【*7.2.2】安装成功，准备数据目录！
                prepareAppDataAfterInstallLIF(newPackage);

            } else {
                //【*8.1】安装失败，删除 apk，但是保存之前就存在的数据！
                deletePackageLIF(pkgName, UserHandle.ALL, false, null,
                        PackageManager.DELETE_KEEP_DATA, res.removedInfo, true, null);
            }
        } catch (PackageManagerException e) {
            res.setError("Package couldn't be installed in " + pkg.codePath, e);
        }

        Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
    }
```
继续分析：

### 7.2.1 updateSettingsLI
```java
    private void updateSettingsLI(PackageParser.Package newPackage, String installerPackageName,
            int[] allUsers, PackageInstalledInfo res, UserHandle user) {
        //【×7.2.1.1】更新父包！
        updateSettingsInternalLI(newPackage, installerPackageName, allUsers, res.origUsers,
                res, user);
        //【1】对子包也做相同的处理；
        final int childCount = (newPackage.childPackages != null)
                ? newPackage.childPackages.size() : 0;
        for (int i = 0; i < childCount; i++) {
            PackageParser.Package childPackage = newPackage.childPackages.get(i);
            PackageInstalledInfo childRes = res.addedChildPackages.get(childPackage.packageName);
            //【×7.2.1.1】更新子包！
            updateSettingsInternalLI(childPackage, installerPackageName, allUsers,
                    childRes.origUsers, childRes, user);
        }
    }
```

继续来看：

#### 7.2.1.1 updateSettingsInternalLI

此时，apk 已经扫描了，在扫描的最后阶段，也会创建对应的 PackageSettings 对象，这里会根据安装结果，更新数据！

```java
    private void updateSettingsInternalLI(PackageParser.Package newPackage,
            String installerPackageName, int[] allUsers, int[] installedForUsers,
            PackageInstalledInfo res, UserHandle user) {
        Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "updateSettings");

        String pkgName = newPackage.packageName;
        synchronized (mPackages) {
            //【1】如果应用已经安装了，更新安装状态 INCOMPLETE，并调用 writeLPr 方法更新本地文件！
            // writeLPr 方法在 PMS 开机的时候后有分析过，这里不多说了！
            mSettings.setInstallStatus(pkgName, PackageSettingBase.PKG_INSTALL_INCOMPLETE);
            Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "writeSettings");
            mSettings.writeLPr();
            Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
        }

        if (DEBUG_INSTALL) Slog.d(TAG, "New package installed in " + newPackage.codePath);
        synchronized (mPackages) {
            //【2】更新权限信息，移除无效的权限和权限树，更新的 flags 设置了 UPDATE_PERMISSIONS_REPLACE_PKG，
            // 表示只更新自身；如果其有定义权限，还会增加 UPDATE_PERMISSIONS_ALL！
            // 关于 updatePermissionsLPw 我们在 PMS 启动时分析过！
            updatePermissionsLPw(newPackage.packageName, newPackage,
                    UPDATE_PERMISSIONS_REPLACE_PKG | (newPackage.permissions.size() > 0
                            ? UPDATE_PERMISSIONS_ALL : 0));

            //【3】对于系统应用，我们默认其为可用状态！
            PackageSetting ps = mSettings.mPackages.get(pkgName);
            final int userId = user.getIdentifier();
            if (ps != null) {
                if (isSystemApp(newPackage)) {
                    if (DEBUG_INSTALL) {
                        Slog.d(TAG, "Implicitly enabling system package on upgrade: " + pkgName);
                    }
                    //【3.1】设置在请求的 userId 下，该应用可用！
                    if (res.origUsers != null) {
                        for (int origUserId : res.origUsers) {
                            if (userId == UserHandle.USER_ALL || userId == origUserId) {
                                ps.setEnabled(COMPONENT_ENABLED_STATE_DEFAULT,
                                        origUserId, installerPackageName);
                            }
                        }
                    }
                    //【3.2】设置只有在指定要安装的目标 user 下，安装状态为 true！
                    if (allUsers != null && installedForUsers != null) {
                        for (int currentUserId : allUsers) {
                            final boolean installed = ArrayUtils.contains(
                                    installedForUsers, currentUserId);
                            if (DEBUG_INSTALL) {
                                Slog.d(TAG, "    user " + currentUserId + " => " + installed);
                            }
                            ps.setInstalled(installed, currentUserId);
                        }
                    }
                }
                //【3.3】如果安装目标用户不是 UserHandle.USER_ALL，设置其在该 userId 下为已安装状态并且可用！
                if (userId != UserHandle.USER_ALL) {
                    ps.setInstalled(true, userId);
                    ps.setEnabled(COMPONENT_ENABLED_STATE_DEFAULT, userId, installerPackageName);
                }
            }
            //【4】修改安装结果对象中的属性！
            res.name = pkgName;
            res.uid = newPackage.applicationInfo.uid;
            res.pkg = newPackage;

            //【5】更新 Settings 中该 PackageSetting 的安装状态！
            mSettings.setInstallStatus(pkgName, PackageSettingBase.PKG_INSTALL_COMPLETE);
            mSettings.setInstallerPackageName(pkgName, installerPackageName);
            
            res.setReturnCode(PackageManager.INSTALL_SUCCEEDED);
            
            //【6】并调用 writeLPr 方法更新本地文件！
            Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "writeSettings");
            mSettings.writeLPr();
            Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
        }

        Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
    }
```

### 7.2.2 prepareAppDataAfterInstallLIF

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
                //【7.2.2.1】准备数据目录！
                prepareAppDataLIF(pkg, user.id, flags);
            }
        }
    }
```

暂时分析到这里！

#### 7.2.2.1 prepareAppDataLIF

```java
    private void prepareAppDataLIF(PackageParser.Package pkg, int userId, int flags) {
        if (pkg == null) {
            Slog.wtf(TAG, "Package was null!", new Throwable());
            return;
        }
        //【×7.2.2】准备父包的数据目录
        prepareAppDataLeafLIF(pkg, userId, flags);
        final int childCount = (pkg.childPackages != null) ? pkg.childPackages.size() : 0;
        for (int i = 0; i < childCount; i++) {
            //【×7.2.2.2】准备子包的数据目录
            prepareAppDataLeafLIF(pkg.childPackages.get(i), userId, flags);
        }
    }
```

#### 7.2.2.2 prepareAppDataLeafLIF
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

        prepareAppDataContentsLeafLIF(pkg, userId, flags);
    }
```

# 8 PackageManagerS.replaceNonSystemPackageLIF - 接 6.2.2

这里我们分析下覆盖安装三方应用的流程，对于扫描标志位 scanFlags，和上面保持一致：

```java
    private void replaceNonSystemPackageLIF(PackageParser.Package deletedPackage,
            PackageParser.Package pkg, final int policyFlags, int scanFlags, UserHandle user,
            int[] allUsers, String installerPackageName, PackageInstalledInfo res) {
        if (DEBUG_INSTALL) Slog.d(TAG, "replaceNonSystemPackageLI: new=" + pkg + ", old="
                + deletedPackage);

        String pkgName = deletedPackage.packageName; // 获得包名！
        boolean deletedPkg = true;
        boolean addedPkg = false;
        boolean updatedSettings = false;
        final boolean killApp = (scanFlags & SCAN_DONT_KILL_APP) == 0;

        //【1】设置删除 flags，会保留用户数据！
        final int deleteFlags = PackageManager.DELETE_KEEP_DATA
                | (killApp ? 0 : PackageManager.DELETE_DONT_KILL_APP);

        final long origUpdateTime = (pkg.mExtras != null)
                ? ((PackageSetting)pkg.mExtras).lastUpdateTime : 0;

        //【*8.1】删除在 data 分区的旧 apk 数据，res.removedInfo 用于表示要移除的 apk！！
        // 删除失败会进入 if 分支！
        if (!deletePackageLIF(pkgName, null, true, allUsers, deleteFlags,
                res.removedInfo, true, pkg)) {

            res.setError(INSTALL_FAILED_REPLACE_COULDNT_DELETE, "replaceNonSystemPackageLI");
            deletedPkg = false;
        } else {
            //【2】删除成功后，要判断下被删除的 apk 是否安装在 External 位置或者其是否是 forward lock 的
            // 如果是，那么我们要发送资源变化的广播通知其他进程！！
            if (deletedPackage.isForwardLocked() || isExternal(deletedPackage)) {
                if (DEBUG_INSTALL) {
                    Slog.i(TAG, "upgrading pkg " + deletedPackage + " is ASEC-hosted -> UNAVAILABLE");
                }
                final int[] uidArray = new int[] { deletedPackage.applicationInfo.uid };
                final ArrayList<String> pkgList = new ArrayList<String>(1);
                pkgList.add(deletedPackage.applicationInfo.packageName);
                sendResourcesChangedBroadcast(false, true, pkgList, uidArray, null);
            }

            //【*6.2.1.4】清楚 code cache 数据！
            clearAppDataLIF(pkg, UserHandle.USER_ALL, StorageManager.FLAG_STORAGE_DE
                    | StorageManager.FLAG_STORAGE_CE | Installer.FLAG_CLEAR_CODE_CACHE_ONLY);

            //【*6.2.1.5】清楚要被删除的应用的 profile 数据，这个和 odex 优化相关！
            clearAppProfilesLIF(deletedPackage, UserHandle.USER_ALL);

            try {
                //【3】扫描新的 apk 应用！
                final PackageParser.Package newPackage = scanPackageTracedLI(pkg, policyFlags,
                        scanFlags | SCAN_UPDATE_TIME, System.currentTimeMillis(), user);
                        
                //【4】更新系统中的数据结构；
                updateSettingsLI(newPackage, installerPackageName, allUsers, res, user);

                //【5】记录保存被删除的旧 apk 的 code path，包括父包和子包！
                PackageSetting ps = mSettings.mPackages.get(pkgName);
                if (!killApp) {
                    if (ps.oldCodePaths == null) {
                        ps.oldCodePaths = new ArraySet<>();
                    }
                    Collections.addAll(ps.oldCodePaths, deletedPackage.baseCodePath);
                    if (deletedPackage.splitCodePaths != null) {
                        Collections.addAll(ps.oldCodePaths, deletedPackage.splitCodePaths);
                    }
                } else {
                    ps.oldCodePaths = null;
                }
                if (ps.childPackageNames != null) {
                    for (int i = ps.childPackageNames.size() - 1; i >= 0; --i) {
                        final String childPkgName = ps.childPackageNames.get(i);
                        final PackageSetting childPs = mSettings.mPackages.get(childPkgName);
                        childPs.oldCodePaths = ps.oldCodePaths;
                    }
                }
                
                //【*7.2.2】准备 app 数据目录；
                prepareAppDataAfterInstallLIF(newPackage);
                
                // 表示安装应用成功！
                addedPkg = true;
            } catch (PackageManagerException e) {
                res.setError("Package couldn't be installed in " + pkg.codePath, e);
            }
        }
        //【6】如果前面的处理没有问题，进入 if 分支！！
        if (res.returnCode != PackageManager.INSTALL_SUCCEEDED) {
            if (DEBUG_INSTALL) Slog.d(TAG, "Install failed, rolling pack: " + pkgName);

            //【*8.1】如果安装不成功，但是我们已经扫描了新的 apk 那么这里会删除其遗留的数据！
            if (addedPkg) {
                deletePackageLIF(pkgName, null, true, allUsers, deleteFlags,
                        res.removedInfo, true, null);
            }

            //【6.1】如果新 apk 安装失败了，并且执行了清理旧的 apk 的数据，那么这里会 reinstall 旧 apk！
            if (deletedPkg) {
                if (DEBUG_INSTALL) Slog.d(TAG, "Install failed, reinstalling: " + deletedPackage);
                File restoreFile = new File(deletedPackage.codePath);

                //【6.1.1】设置解析参数 flags！
                boolean oldExternal = isExternal(deletedPackage);
                int oldParseFlags  = mDefParseFlags | PackageParser.PARSE_CHATTY |
                        (deletedPackage.isForwardLocked() ? PackageParser.PARSE_FORWARD_LOCK : 0) |
                        (oldExternal ? PackageParser.PARSE_EXTERNAL_STORAGE : 0);
                int oldScanFlags = SCAN_UPDATE_SIGNATURE | SCAN_UPDATE_TIME;
                try {
                    //【6.1.2】重新解析和扫描旧的 apk！
                    scanPackageTracedLI(restoreFile, oldParseFlags, oldScanFlags, origUpdateTime,
                            null);
                } catch (PackageManagerException e) {
                    Slog.e(TAG, "Failed to restore package : " + pkgName + " after failed upgrade: "
                            + e.getMessage());
                    return;
                }

                synchronized (mPackages) {
                    //【6.1.3】设置 installer name！
                    setInstallerPackageNameLPw(deletedPackage, installerPackageName);
                    
                    //【6.1.4】更新被恢复的旧 apk 的权限信息！
                    updatePermissionsLPw(deletedPackage, UPDATE_PERMISSIONS_ALL);
                    
                    //【6.1.5】持久化内存中的应用数据！
                    mSettings.writeLPr();
                }

                Slog.i(TAG, "Successfully restored package : " + pkgName + " after failed upgrade");
            }
        } else {
            //【7】新 apk 覆盖安装成功了，进入 else！
            synchronized (mPackages) {
                PackageSetting ps = mSettings.peekPackageLPr(pkg.packageName);
                if (ps != null) {
                    //【7.1】更新要从那些 user 下移除！
                    res.removedInfo.removedForAllUsers = mPackages.get(ps.name) == null;
                    if (res.removedInfo.removedChildPackages != null) {
                        final int childCount = res.removedInfo.removedChildPackages.size();
                        for (int i = childCount - 1; i >= 0; i--) {
                            String childPackageName = res.removedInfo.removedChildPackages.keyAt(i);
                            if (res.addedChildPackages.containsKey(childPackageName)) {
                                res.removedInfo.removedChildPackages.removeAt(i);
                            } else {
                                PackageRemovedInfo childInfo = res.removedInfo
                                        .removedChildPackages.valueAt(i);
                                childInfo.removedForAllUsers = mPackages.get(
                                        childInfo.removedPackage) == null;
                            }
                        }
                    }
                }
            }
        }
    }
```
不多说了！

## 8.1 deletePackageLIF

删除 apk，参数 PackageParser.Package replacingPackage 表示用于 replace 的 package：

deletePackageLIF 中的逻辑很多涉及到了 delete package 的逻辑，和 install package 并没有太大关系，我们在后面的 delete package 中会继续分析！

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
            //【1】获得要删除的 apk 的 PackageSetting 数据！
            ps = mSettings.mPackages.get(packageName);
            if (ps == null) {
                Slog.w(TAG, "Package named '" + packageName + "' doesn't exist.");
                return false;
            }
            //【2】处理子包的数据：
            // 如果是子包，并且（其不是系统应用，或者 deleteFlags 设置了 DELETE_SYSTEM_APP 标志）！
            // 那么我们会清除该子包的使用状态，同时设置在设备用户下处于未安装状态！
            if (ps.parentPackageName != null && (!isSystemApp(ps)
                    || (flags & PackageManager.DELETE_SYSTEM_APP) != 0)) {
                if (DEBUG_REMOVE) {
                    Slog.d(TAG, "Uninstalled child package:" + packageName + " for user:"
                            + ((user == null) ? UserHandle.USER_ALL : user));
                }
                //【2.1】判断下要在那些 user 下删除该子包 apk 的状态信息！
                final int removedUserId = (user != null) ? user.getIdentifier()
                        : UserHandle.USER_ALL;
    
                //【*8.1.1】清除掉 userId 下的子包 apk 的使用信息！！
                if (!clearPackageStateForUserLIF(ps, removedUserId, outInfo)) {
                    return false;
                }
                
                //【*8.1.2】更新子包 apk 在该 user 下为未安装状态！
                markPackageUninstalledForUserLPw(ps, user);
                
                //【2.2】更新应用偏好设置！
                scheduleWritePackageRestrictionsLocked(user);
                return true;
            }
        }
        
        //【3】如果只是删除在指定 user 下的安装信息，那么我们会设置其在该 user 下为未安装的状态，同时删除其数据！
        // 如果 deleteFlags 设置了 DELETE_SYSTEM_APP 标志位，表示删除的是系统应用；
        if (((!isSystemApp(ps) || (flags & PackageManager.DELETE_SYSTEM_APP) != 0) && user != null
                && user.getIdentifier() != UserHandle.USER_ALL)) {

            //【*8.1.2】更新子包 apk 在该 user 下为未安装状态！
            markPackageUninstalledForUserLPw(ps, user);

            if (!isSystemApp(ps)) {
                //【3.1】对于非系统应用的情况，先判断下是否需要保留不卸载！
                boolean keepUninstalledPackage = shouldKeepUninstalledPackageLPr(packageName);

                //【3.2】如果该 package 在其他用户下有安装；或者其需要保留不卸载！
                // 那么我们只需要清楚在当前指定的用户 user 下的数据并设置其为未安装状态；
                if (ps.isAnyInstalled(sUserManager.getUserIds()) || keepUninstalledPackage) {
                    if (DEBUG_REMOVE) Slog.d(TAG, "Still installed by other users");

                    //【*8.1.1】清除掉当前指定的用户 user 下的 apk 的使用信息！！
                    if (!clearPackageStateForUserLIF(ps, user.getIdentifier(), outInfo)) {
                        return false;
                    }
                    //【3.3】更新应用偏好设置，然后返回！
                    scheduleWritePackageRestrictionsLocked(user);
                    return true;

                } else {
                    //【3.4】这里说明其没有在其他 user 下安装，同时也不需要保留，那么后面会执行删除操作
                    // 这里会将在当前指定的用户 user 下的安装状态设置为 installed，这样卸载广播能正确发出！ 
                    if (DEBUG_REMOVE) Slog.d(TAG, "Not installed by other users, full delete");
                    ps.setInstalled(true, user.getIdentifier());
                }
                
            } else {
                //【3.5】对于系统应用的情况，这里只会清楚当前指定用户下的数据，然后返回！
                if (DEBUG_REMOVE) Slog.d(TAG, "Deleting system app");
                
                //【*8.1.1】清除掉当前指定的用户 user 下的 apk 的使用信息！！
                if (!clearPackageStateForUserLIF(ps, user.getIdentifier(), outInfo)) {
                    return false;
                }
                
                //【3.6】更新应用偏好设置，然后返回！
                scheduleWritePackageRestrictionsLocked(user);
                return true;
            }
        }

        //【4】这里按照逻辑，只有系统 apk 才能进入，因为只有系统 apk 才有子包！
        // 这里是处理每个子包的移除信息，包括子包名，子包所在 user！
        if (ps.childPackageNames != null && outInfo != null) {
            synchronized (mPackages) {
                final int childCount = ps.childPackageNames.size();
                outInfo.removedChildPackages = new ArrayMap<>(childCount);
                for (int i = 0; i < childCount; i++) {
                    String childPackageName = ps.childPackageNames.get(i);
                    PackageRemovedInfo childInfo = new PackageRemovedInfo();
                    childInfo.removedPackage = childPackageName;
                    
                    //【4.1】添加到父包的 removedChildPackages 集合中！
                    outInfo.removedChildPackages.put(childPackageName, childInfo);
                    PackageSetting childPs = mSettings.peekPackageLPr(childPackageName);
                    if (childPs != null) {
                        childInfo.origUsers = childPs.queryInstalledUsers(allUserHandles, true);
                    }
                }
            }
        }

        boolean ret = false;
        if (isSystemApp(ps)) {
            if (DEBUG_REMOVE) Slog.d(TAG, "Removing system package: " + ps.name);
            //【×8.1.3】删除系统 apk，如果系统 apk 之前被覆盖更新了，那么我们会删除新的 apk
            // 回滚到 system 分区的旧 apk！！
            ret = deleteSystemPackageLIF(ps.pkg, ps, allUserHandles, flags, outInfo, writeSettings);
        } else {
            if (DEBUG_REMOVE) Slog.d(TAG, "Removing non-system package: " + ps.name);
            //【×8.1.4】删除非系统的 apk！
            ret = deleteInstalledPackageLIF(ps, deleteCodeAndResources, flags, allUserHandles,
                    outInfo, writeSettings, replacingPackage);
        }

        //【5】记录下，我们是否从所有的用户 user 下删除了该 package!
        if (outInfo != null) {
            //【5.1】更新父包和子包的 removedForAllUsers 属性！
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

            //【5.2】我们卸载的是一个系统 apk 的更新包，那么可能有些子包在 system 分区的旧 apk 中声明了
            // 但是在新的 data 更新 apk 中没有申明，这里会处理这种情况！！
            if (isSystemApp(ps)) {
                synchronized (mPackages) {
                    PackageSetting updatedPs = mSettings.peekPackageLPr(ps.name);
                    final int childCount = (updatedPs.childPackageNames != null)
                            ? updatedPs.childPackageNames.size() : 0;
                    for (int i = 0; i < childCount; i++) {
                        String childPackageName = updatedPs.childPackageNames.get(i);
                        if (outInfo.removedChildPackages == null
                                || outInfo.removedChildPackages.indexOfKey(childPackageName) < 0) {
                            PackageSetting childPs = mSettings.peekPackageLPr(childPackageName);
                            if (childPs == null) {
                                continue;
                            }
                            PackageInstalledInfo installRes = new PackageInstalledInfo();
                            installRes.name = childPackageName;
                            installRes.newUsers = childPs.queryInstalledUsers(allUserHandles, true);
                            installRes.pkg = mPackages.get(childPackageName);
                            installRes.uid = childPs.pkg.applicationInfo.uid;
                            if (outInfo.appearedChildPackages == null) {
                                outInfo.appearedChildPackages = new ArrayMap<>();
                            }
                            //【5.3】将这些子包保存到 outInfo.appearedChildPackages 中！
                            outInfo.appearedChildPackages.put(childPackageName, installRes);
                        }
                    }
                }
            }
        }

        return ret;
    }
```
先看到这里！

### 8.1.1 clearPackageStateForUserLIF

清楚指定 user 下的应用数据，这里涉及到的清理操作很多，由于篇幅，这里先不细讲，等到卸载应用时在深入分析！
```java
    private boolean clearPackageStateForUserLIF(PackageSetting ps, int userId,
            PackageRemovedInfo outInfo) {
        final PackageParser.Package pkg;
        synchronized (mPackages) {
            //【1】获得旧 apk 的扫描信息！
            pkg = mPackages.get(ps.name);
        }
        //【2】判断要删除那些 user 下的状态数据！
        final int[] userIds = (userId == UserHandle.USER_ALL) ? sUserManager.getUserIds()
                : new int[] {userId};
        for (int nextUserId : userIds) {
            if (DEBUG_REMOVE) {
                Slog.d(TAG, "Updating package:" + ps.name + " install state for user:"
                        + nextUserId);
            }
            //【×8.1.1.1】删除掉旧 apk 的使用状态本地记录文件！
            destroyAppDataLIF(pkg, userId,
                    StorageManager.FLAG_STORAGE_DE | StorageManager.FLAG_STORAGE_CE);
                    
            //【×8.1.1.2】删除 profile 相关信息！！
            destroyAppProfilesLIF(pkg, userId);
            
            //【4】移除 key store！！
            removeKeystoreDataIfNeeded(nextUserId, ps.appId);
            
            //【×8.1.1.3】执行 package clean 操作！！
            schedulePackageCleaning(ps.name, nextUserId, false);
            
            synchronized (mPackages) {
                //【5】清楚默认应用设置!
                if (clearPackagePreferredActivitiesLPw(ps.name, nextUserId)) {
                    //【5.1】更新偏好设置！
                    scheduleWritePackageRestrictionsLocked(nextUserId);
                }
                //【6】更新运行时权限信息！
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
关于以下的内容，会在卸载 app 和权限相关文章中分析，这里由于篇幅原因（markdown 只支持 6 级标题），就先不深入分析了！

```java
    //【3】删除 profile 相关信息！！
    destroyAppProfilesLIF(pkg, userId);
    //【4】移除 key store！！
    removeKeystoreDataIfNeeded(nextUserId, ps.appId);
    //【5】清楚默认应用设置
    clearPackagePreferredActivitiesLPw();
    //【6】更新运行时权限信息！
    resetUserChangesToRuntimePermissionsAndFlagsLPw(ps, nextUserId);
```

这里就不再多说了！！

#### 8.1.1.1 destroyAppDataLIF ->[Leaf]
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

##### 8.1.1.1.1 PackageSetting.getCeDataInode

```java
    long getCeDataInode(int userId) {
        return readUserState(userId).ceDataInode;
    }
```
该方法返回的是 PackageUserState.ceDataInode 的值！

#### 8.1.1.3 schedulePackageCleaning

```java
    void schedulePackageCleaning(String packageName, int userId, boolean andCode) {
        //【*8.1.1.3.1】这里会发送一个 START_CLEANING_PACKAGE 的消息给 PackageHandler ！
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

##### 8.1.1.3.1 Packagehandler.doHandleMessage[START_CLEANING_PACKAGE]

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
        //【*8.1.1.3.2】
        startCleaningPackages();
    } break;
```

##### 8.1.1.3.2 startCleaningPackages

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

### 8.1.2 markPackageUninstalledForUserLPw

更新每个用户下的 package 的用户状态：

```java
    private void markPackageUninstalledForUserLPw(PackageSetting ps, UserHandle user) {
        final int[] userIds = (user == null || user.getIdentifier() == UserHandle.USER_ALL)
                ? sUserManager.getUserIds() : new int[] {user.getIdentifier()};
        for (int nextUserId : userIds) {
            if (DEBUG_REMOVE) {
                Slog.d(TAG, "Marking package:" + ps.name + " uninstalled for user:" + nextUserId);
            }
            //【1】调用了 PackageSettings 的 setUserState 接口！
            ps.setUserState(nextUserId, 0, COMPONENT_ENABLED_STATE_DEFAULT,
                    false /*installed*/, true /*stopped*/, true /*notLaunched*/,
                    false /*hidden*/, false /*suspended*/, null, null, null,
                    false /*blockUninstall*/,
                    ps.readUserState(nextUserId).domainVerificationStatus, 0);
        }
    }
```

### 8.1.3 deleteSystemPackageLIF

关于 flags 的设置：

```java
    //【1】设置删除 flags，会保留用户数据！
    final int deleteFlags = PackageManager.DELETE_KEEP_DATA
            | (killApp ? 0 : PackageManager.DELETE_DONT_KILL_APP);
```

删除 system 更新 apk，PackageRemovedInfo outInfo 用于封装移除的相关信息！

```java
    private boolean deleteSystemPackageLIF(PackageParser.Package deletedPkg,
            PackageSetting deletedPs, int[] allUserHandles, int flags, PackageRemovedInfo outInfo,
            boolean writeSettings) {
        if (deletedPs.parentPackageName != null) {
            Slog.w(TAG, "Attempt to delete child system package " + deletedPkg.packageName);
            return false;
        }

        final boolean applyUserRestrictions
                = (allUserHandles != null) && (outInfo.origUsers != null);
        final PackageSetting disabledPs;

        synchronized (mPackages) {
            //【1】尝试获得被更新的 system apk 数据！
            disabledPs = mSettings.getDisabledSystemPkgLPr(deletedPs.name);
        }

        if (DEBUG_REMOVE) Slog.d(TAG, "deleteSystemPackageLI: newPs=" + deletedPkg.packageName
                + " disabledPs=" + disabledPs);
        //【2】如果没有更新，那就不能删除，对于系统 apk，我们只能删除覆盖更新在 data 分区的！
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

        //【3】设置父包和子包的 isRemovedPackageSystemUpdate 为 true！
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

        //【4】判读 version code，如果出现降级，那么就不能保留数据！
        if (disabledPs.versionCode < deletedPs.versionCode) {
            flags &= ~PackageManager.DELETE_KEEP_DATA;
        } else {
            flags |= PackageManager.DELETE_KEEP_DATA;
        }
        
        //【×8.1.4】执行删除 data 分区新 apk，删除失败就结束操作！
        boolean ret = deleteInstalledPackageLIF(deletedPs, true, flags, allUserHandles,
                outInfo, writeSettings, disabledPs.pkg);
        if (!ret) {
            return false;
        }

        //【5】恢复 system 分区的旧 apk 的数据，同时删除更新的 apk 的 native libs
        synchronized (mPackages) {
            enableSystemPackageLPw(disabledPs.pkg);
            removeNativeBinariesLI(deletedPs);
        }

        //【6】设置扫描 flags！
        if (DEBUG_REMOVE) Slog.d(TAG, "Re-installing system package: " + disabledPs);
        int parseFlags = mDefParseFlags
                | PackageParser.PARSE_MUST_BE_APK
                | PackageParser.PARSE_IS_SYSTEM
                | PackageParser.PARSE_IS_SYSTEM_DIR;
        if (locationIsPrivileged(disabledPs.codePath)) {
            parseFlags |= PackageParser.PARSE_IS_PRIVILEGED;
        }

        final PackageParser.Package newPkg;
        try {
            //【7】扫描旧的 system apk！
            newPkg = scanPackageTracedLI(disabledPs.codePath, parseFlags, SCAN_NO_PATHS, 0, null);
        } catch (PackageManagerException e) {
            Slog.w(TAG, "Failed to restore system package:" + deletedPkg.packageName + ": "
                    + e.getMessage());
            return false;
        }
        try {
            //【8】更新 shared libs 给重新安装的 system apk！
            updateSharedLibrariesLPw(newPkg, null);
        } catch (PackageManagerException e) {
            Slog.e(TAG, "updateAllSharedLibrariesLPw failed: " + e.getMessage());
        } 
        
        //【*7.2.2】准备 app 数据目录；
        prepareAppDataAfterInstallLIF(newPkg);

        //【9】更新 pms 数据！
        synchronized (mPackages) {
            PackageSetting ps = mSettings.mPackages.get(newPkg.packageName);

            //【9.1】从被删除的 data 分区 apk 那里继承最新的权限信息！
            ps.getPermissionsState().copyFrom(deletedPs.getPermissionsState());
            //【9.2】更新系统中所有应用的权限信息！
            updatePermissionsLPw(newPkg.packageName, newPkg,
                    UPDATE_PERMISSIONS_ALL | UPDATE_PERMISSIONS_REPLACE_PKG);

            if (applyUserRestrictions) {
                if (DEBUG_REMOVE) {
                    Slog.d(TAG, "Propagating install state across reinstall");
                }
                for (int userId : allUserHandles) {
                    final boolean installed = ArrayUtils.contains(outInfo.origUsers, userId);
                    if (DEBUG_REMOVE) {
                        Slog.d(TAG, "    user " + userId + " => " + installed);
                    }
                    ps.setInstalled(installed, userId);
                    //【9.3】持久化运行时权限；
                    mSettings.writeRuntimePermissionsForUserLPr(userId, false);
                }
                //【9.4】持久化应用偏好设置！
                mSettings.writeAllUsersPackageRestrictionsLPr();
            }
            //【9.5】持久化最新的所有应用数据！
            if (writeSettings) {
                mSettings.writeLPr();
            }
        }
        return true;
    }
```
这里就不在多说了！


### 8.1.4 deleteInstalledPackageLIF

关于 flags 的设置：

```java
    //【1】设置删除 flags，会保留用户数据！
    final int deleteFlags = PackageManager.DELETE_KEEP_DATA
            | (killApp ? 0 : PackageManager.DELETE_DONT_KILL_APP);
```

删除 data 分区的 apk：

```java
    private boolean deleteInstalledPackageLIF(PackageSetting ps,
            boolean deleteCodeAndResources, int flags, int[] allUserHandles,
            PackageRemovedInfo outInfo, boolean writeSettings,
            PackageParser.Package replacingPackage) {
        synchronized (mPackages) {
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

        //【×8.1.4.1】移除 package 的数据！
        removePackageDataLIF(ps, allUserHandles, outInfo, flags, writeSettings);

        //【1】如果有子包，也会移除子包的数据！
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
                final int deleteFlags = (flags & DELETE_KEEP_DATA) != 0
                        && (replacingPackage != null
                        && !replacingPackage.hasChildPackage(childPs.name))
                        ? flags & ~DELETE_KEEP_DATA : flags;
                        
                //【×8.1.4.1】移除 package 的数据！
                removePackageDataLIF(childPs, allUserHandles, childOutInfo,
                        deleteFlags, writeSettings);
            }
        }

        //【2】对于被删除的父包，会创建一个 InstallArgs，用于删除 apk 和资源！
        if (ps.parentPackageName == null) {
            if (deleteCodeAndResources && (outInfo != null)) {
            
                //【×6.2.1.3】根据一个存在的 package 创建一个 InstallArgs 中！
                outInfo.args = createInstallArgsForExisting(packageFlagsToInstallFlags(ps),
                        ps.codePathString, ps.resourcePathString, getAppDexInstructionSets(ps));

                if (DEBUG_SD_INSTALL) Slog.i(TAG, "args=" + outInfo.args);
            }
        }

        return true;
    }
```
不多说了！

#### 8.1.4.1 removePackageDataLIF

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
        //【×8.1.4.1】第一部移除，扫描和四大组件信息！
        removePackageLI(ps, (flags & REMOVE_CHATTY) != 0);

        //【2】如果 flags 没有设置 DELETE_KEEP_DATA，那么会清楚 apk 的数据，显然这里由于设置了，那么就不会清楚！
        if ((flags & PackageManager.DELETE_KEEP_DATA) == 0) {
            final PackageParser.Package resolvedPkg;
            if (deletedPkg != null) {
                resolvedPkg = deletedPkg;
            } else {
                resolvedPkg = new PackageParser.Package(ps.name);
                resolvedPkg.setVolumeUuid(ps.volumeUuid);
            }

            //【×8.1.1.1】删除 apk 的 data 数据！！
            destroyAppDataLIF(resolvedPkg, UserHandle.USER_ALL,
                    StorageManager.FLAG_STORAGE_DE | StorageManager.FLAG_STORAGE_CE);
            //【×8.1.1.2】删除 apk 的 profiles 数据！！
            destroyAppProfilesLIF(resolvedPkg, UserHandle.USER_ALL);
            
            if (outInfo != null) {
                outInfo.dataRemoved = true;
            }
            
            //【×8.1.1.3】执行 package 清除！！
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
                        mSettings.mKeySetManagerService.removeAppKeySetDataLPw(packageName);
                        outInfo.removedAppId = mSettings.removePackageLPw(packageName);
                    }

                    //【3.1.2】更新权限信息！
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
                    //【3.1.4】清除默认应用的数据；
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
            //【3.5】持久化 Settings！！
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
不多说了！！

##### 8.1.4.1.1 removePackageLI

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
                //【×6.2.1.6.1】移除四大组件，共享库解析对象！
                cleanPackageDataStructuresLILPw(pkg, chatty);
            }
        }
    }
```
这个就不多说了！！
