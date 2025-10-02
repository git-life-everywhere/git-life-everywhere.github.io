# PMS 第 12 篇 - 通过 adb 指令分析 enable/disable 过程
title: PMS 第 12 篇 - 通过 adb 指令分析 enable/disable 过程
date: 2018/09/17
categories:
- AndroidFramework源码分析
- PackageManager包管理
tags: PackageManager包管理
---

[toc]

基于 Android7.1.1 分析 PackageManagerService 的架构设计！
  
# 0 综述  

本文来分析下 pms enable 相关的操作：

- adb shell pm enable
- adb shell pm disable

同样的，我们从 Pm 中看起！


# 1 Pm


## 1.1 run

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

        if ("enable".equals(op)) {
            //【*1.2】调用 runSetEnabledSetting 方法；
            return runSetEnabledSetting(PackageManager.COMPONENT_ENABLED_STATE_ENABLED);
        }

        if ("disable".equals(op)) {
            //【*1.2】调用 runSetEnabledSetting 方法；
            return runSetEnabledSetting(PackageManager.COMPONENT_ENABLED_STATE_DISABLED);
        }

        if ("disable-user".equals(op)) {
            //【*1.2】调用 runSetEnabledSetting 方法；**
            return runSetEnabledSetting(PackageManager.COMPONENT_ENABLED_STATE_DISABLED_USER);
        }

        if ("disable-until-used".equals(op)) {
            //【*1.2】调用 runSetEnabledSetting 方法；
            return runSetEnabledSetting(PackageManager.COMPONENT_ENABLED_STATE_DISABLED_UNTIL_USED);
        }

        if ("default-state".equals(op)) {
            //【*1.2】调用 runSetEnabledSetting 方法；
            return runSetEnabledSetting(PackageManager.COMPONENT_ENABLED_STATE_DEFAULT);
        }

        ... ... ...
    }
```
我们可以看到，和 enable/disable 相关的 pm 指令有很多，但无疑最后调用的都是：runSetEnabledSetting，唯一的区别是参数 int state 不一样！

我们知道，在 AndroidManifest.xml 中，我们可以给 application，activity 等四大组件设置如下的属性，来设置其是否可用：

```java
android:enabled="true|false"
```

下面，我们来看下 state 的值：

- **PackageManager.COMPONENT_ENABLED_STATE_ENABLED**：组件或应用程序已被明确启用，无论其清单中指定了什么，适用于 setApplicationEnabledSetting 和 setComponentEnabledSetting；

- **PackageManager.COMPONENT_ENABLED_STATE_DISABLED**：组件或应用程序已被明确禁用，无论其清单中指定了什么，适用于 setApplicationEnabledSetting 和 setComponentEnabledSetting；

- **PackageManager.COMPONENT_ENABLED_STATE_DISABLED_USER**：应用程序已被明确禁用，无论其清单中指定了什么。因为这是由于用户的请求，所以如果需要，他们可以通过适当的系统 UI 重新启用它，只适用于 setApplicationEnabledSetting；

- **PackageManager.COMPONENT_ENABLED_STATE_DISABLED_UNTIL_USED**：

- **PackageManager.COMPONENT_ENABLED_STATE_DEFAULT**：组件或应用程序处于默认的 enable 状态，也就是我们在 AndroidManifest.xml 中设置的值！

接下来，我们继续分析下流程：

## 1.2 runSetEnabledSetting

```java
    private int runSetEnabledSetting(int state) {
        int userId = UserHandle.USER_SYSTEM;
        String option = nextOption();
        //【1】如果有指定 user，那么将 user id 保存到 userId 中；
        if (option != null && option.equals("--user")) {
            String optionData = nextOptionData();
            if (optionData == null || !isNumber(optionData)) {
                System.err.println("Error: no USER_ID specified");
                return showUsage();
            } else {
                userId = Integer.parseInt(optionData);
            }
        }
        //【2】获得传入的包名参数；
        String pkg = nextArg();
        if (pkg == null) {
            System.err.println("Error: no package or component specified");
            return showUsage();
        }
        //【2】尝试将其转为组件名；
        ComponentName cn = ComponentName.unflattenFromString(pkg);
        if (cn == null) {
            try {
                //【*2.1】如果传入的参数指定的是包名，调用 setApplicationEnabledSetting 方法！
                mPm.setApplicationEnabledSetting(pkg, state, 0, userId,
                        "shell:" + android.os.Process.myUid());
                System.out.println("Package " + pkg + " new state: "
                        + enabledSettingToString(
                        mPm.getApplicationEnabledSetting(pkg, userId)));
                return 0;
            } catch (RemoteException e) {
                System.err.println(e.toString());
                System.err.println(PM_NOT_RUNNING_ERR);
                return 1;
            }
        } else {
            try {
                //【*2.2】如果传入的参数指定的是组件名，调用 setComponentEnabledSetting 方法；
                mPm.setComponentEnabledSetting(cn, state, 0, userId);
                System.out.println("Component " + cn.toShortString() + " new state: "
                        + enabledSettingToString(
                        mPm.getComponentEnabledSetting(cn, userId)));
                return 0;
            } catch (RemoteException e) {
                System.err.println(e.toString());
                System.err.println(PM_NOT_RUNNING_ERR);
                return 1;
            }
        }
    }
```
下面会进入 PackageManagerService 中去！


# 2 PackageManagerService



## 2.1 setApplicationEnabledSetting

```java
    @Override
    public void setApplicationEnabledSetting(String appPackageName,
            int newState, int flags, int userId, String callingPackage) {
        if (!sUserManager.exists(userId)) return;
        if (callingPackage == null) {
            callingPackage = Integer.toString(Binder.getCallingUid());
        }
        //【*2.3】设置 enable 状态
        setEnabledSetting(appPackageName, null, newState, flags, userId, callingPackage);
    }
```


## 2.2 setComponentEnabledSetting

```java
    @Override
    public void setComponentEnabledSetting(ComponentName componentName,
            int newState, int flags, int userId) {
        if (!sUserManager.exists(userId)) return;
        //【*2.3】设置 enable 状态
        setEnabledSetting(componentName.getPackageName(),
                componentName.getClassName(), newState, flags, userId, null);
    }
```

## 2.3 setEnabledSetting - 核心入口

可以看到，无论 application 环视 component，最后调用的都是该方法：

```java
    private void setEnabledSetting(final String packageName, String className, int newState,
            final int flags, int userId, String callingPackage) {
        //【1】首先是对 newState 的取值做检查！
        if (!(newState == COMPONENT_ENABLED_STATE_DEFAULT
              || newState == COMPONENT_ENABLED_STATE_ENABLED
              || newState == COMPONENT_ENABLED_STATE_DISABLED
              || newState == COMPONENT_ENABLED_STATE_DISABLED_USER
              || newState == COMPONENT_ENABLED_STATE_DISABLED_UNTIL_USED)) {
            throw new IllegalArgumentException("Invalid new component state: "
                    + newState);
        }
        PackageSetting pkgSetting;
        final int uid = Binder.getCallingUid();
        final int permission;
        //【2】校验权限，如果是 system uid 默认是授予；如果是其他 uid 检查下是否有 CHANGE_COMPONENT_ENABLED_STATE
        // 的权限；
        if (uid == Process.SYSTEM_UID) {
            permission = PackageManager.PERMISSION_GRANTED;
        } else {
            permission = mContext.checkCallingOrSelfPermission(
                    android.Manifest.permission.CHANGE_COMPONENT_ENABLED_STATE);
        }
        //【3】检查是否有跨 user 的权限；
        enforceCrossUserPermission(uid, userId,
                false /* requireFullPermission */, true /* checkShell */, "set enabled");

        final boolean allowedByPermission = (permission == PackageManager.PERMISSION_GRANTED);
        boolean sendNow = false;
        boolean isApp = (className == null);
        String componentName = isApp ? packageName : className;
        int packageUid = -1;
        ArrayList<String> components;

        // writer
        synchronized (mPackages) {
            //【4】获得上一次的安装信息；
            pkgSetting = mSettings.mPackages.get(packageName);
            if (pkgSetting == null) {
                if (className == null) {
                    throw new IllegalArgumentException("Unknown package: " + packageName);
                }
                throw new IllegalArgumentException(
                        "Unknown component: " + packageName + "/" + className);
            }
        }

        //【5】对于可以执行 enable/disable 的应用做限制，如果 calling uid 和要被 enable/disable 的应用不是同一个
        // ，那么，如果前面没有权限，不允许 enable/disable；如果该应用是受保护的，那么也不允许；
        if (!UserHandle.isSameApp(uid, pkgSetting.appId)) {
            if (!allowedByPermission) {
                throw new SecurityException(
                        "Permission Denial: attempt to change component state from pid="
                        + Binder.getCallingPid()
                        + ", uid=" + uid + ", package uid=" + pkgSetting.appId);
            }
            if (mProtectedPackages.isPackageStateProtected(userId, packageName)) {
                throw new SecurityException("Cannot disable a protected package: " + packageName);
            }
        }

        //【6】进一步处理！
        synchronized (mPackages) {
            if (uid == Process.SHELL_UID) {
                //【6.1】Shell 只能改变 application 的状态，在 ENABLED 和 DISABLED_USER 之间；
                // Shell 不能该 compnent 的状态；
                int oldState = pkgSetting.getEnabled(userId);
                if (className == null
                    &&
                    (oldState == COMPONENT_ENABLED_STATE_DISABLED_USER
                     || oldState == COMPONENT_ENABLED_STATE_DEFAULT
                     || oldState == COMPONENT_ENABLED_STATE_ENABLED)
                    &&
                    (newState == COMPONENT_ENABLED_STATE_DISABLED_USER
                     || newState == COMPONENT_ENABLED_STATE_DEFAULT
                     || newState == COMPONENT_ENABLED_STATE_ENABLED)) {
                    // ok
                } else {
                    throw new SecurityException(
                            "Shell cannot change component state for " + packageName + "/"
                            + className + " to " + newState);
                }
            }
            //【6.2】开始设置 state 状态；
            if (className == null) {
                //【6.2.1】设置的是 application/package 级别的 state；
                if (pkgSetting.getEnabled(userId) == newState) {
                    // 如果本次设置的状态和上一次的一样，不做任何处理
                    return;
                }
                //【6.2.2】如果新的状态是 default 或者 enabled 那么我们不关注 callingPackage！
                if (newState == PackageManager.COMPONENT_ENABLED_STATE_DEFAULT
                    || newState == PackageManager.COMPONENT_ENABLED_STATE_ENABLED) {
                    callingPackage = null;
                }
                //【*3.2】设置 state 状态；
                pkgSetting.setEnabled(newState, userId, callingPackage);
                // pkgSetting.pkg.mSetEnabled = newState;

            } else {
                //【6.2.3】设置的是 component 级别的 state，前提是 component 的 className 是有效的；
                PackageParser.Package pkg = pkgSetting.pkg;
                if (pkg == null || !pkg.hasComponentClassName(className)) {
                    if (pkg != null &&
                            pkg.applicationInfo.targetSdkVersion >=
                                    Build.VERSION_CODES.JELLY_BEAN) {
                        throw new IllegalArgumentException("Component class " + className
                                + " does not exist in " + packageName);
                    } else {
                        Slog.w(TAG, "Failed setComponentEnabledSetting: component class "
                                + className + " does not exist in " + packageName);
                    }
                }

                //【6.2.4】处理新的 state 状态；
                switch (newState) {
                case COMPONENT_ENABLED_STATE_ENABLED:
                    //【*3.3】enable 组件；
                    if (!pkgSetting.enableComponentLPw(className, userId)) {
                        return;
                    }
                    break;

                case COMPONENT_ENABLED_STATE_DISABLED:
                    //【*3.4】disable 组件；
                    if (!pkgSetting.disableComponentLPw(className, userId)) {
                        return;
                    }
                    break;

                case COMPONENT_ENABLED_STATE_DEFAULT:
                    //【*3.5】restore 组件；
                    if (!pkgSetting.restoreComponentLPw(className, userId)) {
                        return;
                    }
                    break;

                default:
                    Slog.e(TAG, "Invalid new component state: " + newState);
                    return;
                }
            }
            
            //【7】保存偏好设置文件；
            scheduleWritePackageRestrictionsLocked(userId);
            
            //【8】准备发送 Package Changed 广播，mPendingBroadcasts 用于保存需要延迟发送的b包广播;
            components = mPendingBroadcasts.get(userId, packageName);

            //【9】判断该 userId 下是否是第一次添加 package；
            final boolean newPackage = components == null;
            if (newPackage) {
                components = new ArrayList<String>();
            }
            if (!components.contains(componentName)) {
                components.add(componentName);
            }

            //【10】如果没有设置 DONT_KILL_APP 标志，那么就会立即发送，sendNow 为 true；
            if ((flags&PackageManager.DONT_KILL_APP) == 0) {
                sendNow = true;
                //【10.1】因为是立即发送，所以会将这次处理的 package 从中移除；
                mPendingBroadcasts.remove(userId, packageName);

            } else {
                //【10.2】如果该 userId 下是第一次添加 package，那么我们会把对应关系保存到 mPendingBroadcasts 中！
                if (newPackage) {
                    mPendingBroadcasts.put(userId, packageName, components);
                }
                if (!mHandler.hasMessages(SEND_PENDING_BROADCAST)) {
                    // 延迟 10s 发送广播；
                    mHandler.sendEmptyMessageDelayed(SEND_PENDING_BROADCAST, BROADCAST_DELAY);
                }
            }
        }

        long callingId = Binder.clearCallingIdentity();
        try {
            //【11】这里是立即发送广播 Intent.ACTION_PACKAGE_CHANGED !
            //（注意这里使用了 Binder 相关的方法，将发送广播的 uid 和 pid 变成了 system process）
            if (sendNow) {
                packageUid = UserHandle.getUid(userId, pkgSetting.appId);
                //【*2.3.1】发送 PACKAGE CHANGED 的广播；
                sendPackageChangedBroadcast(packageName,
                        (flags&PackageManager.DONT_KILL_APP) != 0, components, packageUid);
            }
        } finally {
            Binder.restoreCallingIdentity(callingId);
        }
    }
```
Pms 内部有一个 PendingPackageBroadcasts 对象 mPendingBroadcasts，用于保存需要延迟返送的广播：

```java
    final PendingPackageBroadcasts mPendingBroadcasts = new PendingPackageBroadcasts();
```

我们来看下 PendingPackageBroadcasts 的定义：

```java
    // Set of pending broadcasts for aggregating enable/disable of components.
    static class PendingPackageBroadcasts {
        // for each user id, a map of <package name -> components within that package>
        final SparseArray<ArrayMap<String, ArrayList<String>>> mUidMap;

        public PendingPackageBroadcasts() {
            mUidMap = new SparseArray<ArrayMap<String, ArrayList<String>>>(2);
        }
        ... ... ...
   }
```
可以看到，其内部有一个 mUidMap 的 SparseArray 数组，保存的数据结构满足如下关系：

```java
userId -->  ArrayMap< packageName --> ArrayList<componentName> >
```

### 2.3.1 sendPackageChangedBroadcast

发送 ACTION_PACKAGE_CHANGED 的广播：

```java
    private void sendPackageChangedBroadcast(String packageName,
            boolean killFlag, ArrayList<String> componentNames, int packageUid) {
        if (DEBUG_INSTALL)
            Log.v(TAG, "Sending package changed: package=" + packageName + " components="
                    + componentNames);

        //【1】保存要发送的一些数据；
        Bundle extras = new Bundle(4);
        extras.putString(Intent.EXTRA_CHANGED_COMPONENT_NAME, componentNames.get(0));
        String nameList[] = new String[componentNames.size()];
        componentNames.toArray(nameList);
        extras.putStringArray(Intent.EXTRA_CHANGED_COMPONENT_NAME_LIST, nameList);
        extras.putBoolean(Intent.EXTRA_DONT_KILL_APP, killFlag);
        extras.putInt(Intent.EXTRA_UID, packageUid);

        // If this is not reporting a change of the overall package, then only send it
        // to registered receivers.  We don't want to launch a swath of apps for every
        // little component state change.
        final int flags = !componentNames.contains(packageName)
                ? Intent.FLAG_RECEIVER_REGISTERED_ONLY : 0;

        //【2】发送广播，这个方法前面看过，这里就不多说了。
        sendPackageBroadcast(Intent.ACTION_PACKAGE_CHANGED,  packageName, extras, flags, null, null,
                new int[] {UserHandle.getUserId(packageUid)});
    }

```
不多说了！

# 3 PackageSetting

下面的方法确切的讲是 PackageSetting 的父类 PackageSettingBase 父类的方法：

## 3.1 getEnabled

获得指定 user 下的 app 的 enabled 状态：

```java
    int getEnabled(int userId) {
        //【*3.1.1】读取指定 userId 下的 PackageUserState 实例：
        return readUserState(userId).enabled;
    }
```

### 3.1.1 readUserState

PackageSetting 内部有一个 SparseArray 数组：userState

```java
    public PackageUserState readUserState(int userId) {
        PackageUserState state = userState.get(userId);
        if (state != null) {
            return state;
        }
        return DEFAULT_USER_STATE;
    }
```

## 3.2 setEnabled

设置指定 user 下的 app 的 enabled 状态：

```java
    void setEnabled(int state, int userId, String callingPackage) {
        //【*3.2.1】modify 和 read 的区别是，在没有对应的使用信息的情况下，modify 会创建新的使用信息对象；
        PackageUserState st = modifyUserState(userId);
        //【1】设置状态的值，和修改该值的 pkg；
        st.enabled = state;
        st.lastDisableAppCaller = callingPackage;
    }
```

### 3.2.1 modifyUserState

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


## 3.3 enableComponentLPw

enable 指定组件：

```java
    boolean enableComponentLPw(String componentClassName, int userId) {
        //【*3.3.1】返回对应的 PackageUserState 实例；
        PackageUserState state = modifyUserStateComponents(userId, false, true);
        //【1】changed 用来记录是否发生了变化；
        boolean changed = state.disabledComponents != null
                ? state.disabledComponents.remove(componentClassName) : false;
        //【2】将要 enable 的组件从 disabledComponents 移动到 enabledComponents 中；
        changed |= state.enabledComponents.add(componentClassName);
        return changed;
    }
```

### 3.3.1 modifyUserStateComponents

modify 方法是对 PackageUserState 做一个初始化，然后返回：

```java
    PackageUserState modifyUserStateComponents(int userId, boolean disabled, boolean enabled) {
        //【*3.2.1】返回对应的 PackageUserState 对象；
        PackageUserState state = modifyUserState(userId);
        //【1】如果是 disable 的话，disabledComponents 为 null，那么会初始化该 set；
        if (disabled && state.disabledComponents == null) {
            state.disabledComponents = new ArraySet<String>(1);
        }
        //【2】如果是 enable 的话，enabledComponents 为 null，那么会初始化该 set；
        if (enabled && state.enabledComponents == null) {
            state.enabledComponents = new ArraySet<String>(1);
        }
        return state;
    }
```

PackageUserState 内部有两个 ArraySet 集合:

```java
    public ArraySet<String> disabledComponents; // 禁用的组件；
    public ArraySet<String> enabledComponents; // 可用的组件；
```

... ...


## 3.3 disableComponentLPw

disable 指定组件：

```java
    boolean disableComponentLPw(String componentClassName, int userId) {
        //【*3.3.1】返回对应的 PackageUserState 实例；
        PackageUserState state = modifyUserStateComponents(userId, true, false);
        //【1】changed 用来记录是否发生了变化；
        boolean changed = state.enabledComponents != null
                ? state.enabledComponents.remove(componentClassName) : false;
        //【2】将要 enable 的组件从 enabledComponents 移动到 disabledComponents 中；
        changed |= state.disabledComponents.add(componentClassName);
        return changed;
    }
```

## 3.4 restoreComponentLPw

restore 指定组件：

```java
    boolean restoreComponentLPw(String componentClassName, int userId) {
        //【*3.3.1】返回对应的 PackageUserState 实例；
        PackageUserState state = modifyUserStateComponents(userId, true, true);
        //【1】changed 用来记录是否发生了变化；
        boolean changed = state.disabledComponents != null
                ? state.disabledComponents.remove(componentClassName) : false;
        //【2】将要 restore 的组件从 enabledComponents 和 disabledComponents 中都移除；
        changed |= state.enabledComponents != null
                ? state.enabledComponents.remove(componentClassName) : false;
        return changed;
    }
```
不多说了！


# 4 PackageHandler

然后我们来看下 PackageHandler 对于消息的处理：

## 4.1 doHandleMessage[SEND_PENDING_BROADCAST]

然后我们来看下延迟发送广播的处理：

```java
    case SEND_PENDING_BROADCAST: {
        String packages[];
        ArrayList<String> components[];
        int size = 0;
        int uids[];
        Process.setThreadPriority(Process.THREAD_PRIORITY_DEFAULT);
        synchronized (mPackages) {
            if (mPendingBroadcasts == null) {
                return;
            }
            size = mPendingBroadcasts.size();
            if (size <= 0) {
                // Nothing to be done. Just return
                return;
            }
            //【1】保存所有 change 的 package；
            packages = new String[size];
            //【2】保存所有 change 的 component；
            components = new ArrayList[size];
            //【3】保存所有 change 的 package 的 uid；
            uids = new int[size];
            int i = 0;  // filling out the above arrays

            //【4】遍历 mPendingBroadcasts 数组，处理每一项；
            for (int n = 0; n < mPendingBroadcasts.userIdCount(); n++) {
                int packageUserId = mPendingBroadcasts.userIdAt(n);
                Iterator<Map.Entry<String, ArrayList<String>>> it
                        = mPendingBroadcasts.packagesForUserId(packageUserId)
                                .entrySet().iterator();

                //【4.1】将 mPendingBroadcasts 中的每一项保存到 packages[i]，components[i] 和 uids[i] 中！
                while (it.hasNext() && i < size) {
                    Map.Entry<String, ArrayList<String>> ent = it.next();
                    packages[i] = ent.getKey();
                    components[i] = ent.getValue();
                    PackageSetting ps = mSettings.mPackages.get(ent.getKey());
                    uids[i] = (ps != null)
                            ? UserHandle.getUid(packageUserId, ps.appId)
                            : -1;
                    i++;
                }
            }
            size = i;
            mPendingBroadcasts.clear();
        }
        //【*2.3.1】发送 Package Changed 广播！
        for (int i = 0; i < size; i++) {
            sendPackageChangedBroadcast(packages[i], true, components[i], uids[i]);
        }
        Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
        break;
    }
```


# 5 不同的 enable 状态的区别