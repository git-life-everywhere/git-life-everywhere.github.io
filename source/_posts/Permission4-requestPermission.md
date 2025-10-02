# Permission第 4 篇 - requestPermission 权限申请
title: Permission第 4 篇 - requestPermission 权限申请
date: 2017/08/13 20:46:25
catalog: true
categories:
- AndroidFramework源码分析
- Permission权限管理
tags: Permission权限管理
---

[TOC]

# 0 综述

基于 Android 7.1.1 源码，分析系统的权限管理机制！

我们来分析下 dangerous 权限的申请，这类权限只能在 Activity 中申请，Activity 提供了如下接口，来帮助程序申请权限：

# 1 Activity.requestPermissions - 请求入口
```java
    public final void requestPermissions(@NonNull String[] permissions, int requestCode) {
        if (mHasCurrentPermissionsRequest) {
            Log.w(TAG, "Can reqeust only one set of permissions at a time");
            //【1】防止同一时间多次申请权限！
            onRequestPermissionsResult(requestCode, new String[0], new int[0]);
            return;
        }
        //【2】设置了发送给 packageInstaller 的 intent！
        Intent intent = getPackageManager().buildRequestPermissionsIntent(permissions);
        startActivityForResult(REQUEST_PERMISSIONS_WHO_PREFIX, intent, requestCode, null);
        mHasCurrentPermissionsRequest = true;
    }
```

## 1.1 PackageManager.buildRequestPermissionsIntent
```java
    public Intent buildRequestPermissionsIntent(@NonNull String[] permissions) {
        if (ArrayUtils.isEmpty(permissions)) {
           throw new IllegalArgumentException("permission cannot be null or empty");
        }
        Intent intent = new Intent(ACTION_REQUEST_PERMISSIONS);
        intent.putExtra(EXTRA_REQUEST_PERMISSIONS_NAMES, permissions);
        intent.setPackage(getPermissionControllerPackageName());
        return intent;
    }
```
这里创建了一个 intent，action 为 android.content.pm.action.REQUEST_PERMISSIONS，同时用 intent 封装了权限信息！
```java
    @SystemApi
    public static final String ACTION_REQUEST_PERMISSIONS =
            "android.content.pm.action.REQUEST_PERMISSIONS"; // hide
    @SystemApi
    public static final String EXTRA_REQUEST_PERMISSIONS_NAMES =
            "android.content.pm.extra.REQUEST_PERMISSIONS_NAMES";
```
接下来，我们要问，谁会接受这个 intent 呢？

下面还调用了 getPermissionControllerPackageName 方法，设置了 intent 的接收者
```java
    @Override
    public String getPermissionControllerPackageName() {
        synchronized (mPackages) {
            return mRequiredInstallerPackage; // 就是 PackageInstaller！
        }
    }
```
就是 PackageInstaller，我们需要进入 PackageInstaller，分析接下来的逻辑！

# 2 PackageInstaller.GrantPermissionsActivity

通过分析 PackageInstaller 的代码，我们知道了 GrantPermissionsActivity 会接受该 intent！

```xml
        <activity android:name=".permission.ui.GrantPermissionsActivity"
                android:configChanges="orientation|keyboardHidden|screenSize"
                android:excludeFromRecents="true"
                android:theme="@style/GrantPermissions">
            <intent-filter>
                <action android:name="android.content.pm.action.REQUEST_PERMISSIONS" />
                <category android:name="android.intent.category.DEFAULT" />
            </intent-filter>
        </activity>
```
GrantPermissionsActivity 虽然是一个 Activity，但实际的现实效果是一个 Dialog，下面我们会一个一个分析！

## 2.1 GrantPermissionsActivity.onCreate
我们去 onCreate 方法中去看看：

```java
public class GrantPermissionsActivity extends OverlayTouchActivity
        implements GrantPermissionsViewHandler.ResultListener {

    @Override
    public void onCreate(Bundle icicle) {
        super.onCreate(icicle);
        setFinishOnTouchOutside(false);

        setTitle(R.string.permission_request_title);
        // 根据不同的设备类型进行不同的操作！
        if (DeviceUtils.isTelevision(this)) {
            mViewHandler = new com.android.packageinstaller.permission.ui.television
                    .GrantPermissionsViewHandlerImpl(this).setResultListener(this);
        } else if (DeviceUtils.isWear(this)) {
            mViewHandler = new GrantPermissionsWatchViewHandler(this).setResultListener(this);
        } else {
            //【2.1.1】对于 Android 来说，进入这里，创建了一个 GrantPermissionsViewHandlerImpl，用于显示权限弹窗
            // 以及和处理和弹窗相关的逻辑！
            mViewHandler = new com.android.packageinstaller.permission.ui.handheld
                    .GrantPermissionsViewHandlerImpl(this).setResultListener(this);
        }
        //【1】获得应用本次要请求的运行时权限！
        mRequestedPermissions = getIntent().getStringArrayExtra(
                PackageManager.EXTRA_REQUEST_PERMISSIONS_NAMES);
        if (mRequestedPermissions == null) {
            mRequestedPermissions = new String[0];
        }
        //【2】创建每个权限对应的授结果况数组，默认值为！
        final int requestedPermCount = mRequestedPermissions.length;
        mGrantResults = new int[requestedPermCount];
        Arrays.fill(mGrantResults, PackageManager.PERMISSION_DENIED);

        if (requestedPermCount == 0) { // 如果请求权限的数量为 0 ，无法申请！
            //【2.1.2】调用 setResultAndFinish 结束申请！
            setResultAndFinish();
            return;
        }
        //【2.1.3】获得请求的应用程序信息
        PackageInfo callingPackageInfo = getCallingPackageInfo();
        // 如果应用程序不存在，或者应用程序没有申请权限，无法申请！
        if (callingPackageInfo == null || callingPackageInfo.requestedPermissions == null
                || callingPackageInfo.requestedPermissions.length <= 0) {
            setResultAndFinish();
            return;
        }

        // 如果应用程序的 targetSdkVersion 低于 Android M，不能申请运行时权限，直接取消！
        if (callingPackageInfo.applicationInfo.targetSdkVersion < Build.VERSION_CODES.M) {
            mRequestedPermissions = new String[0];
            mGrantResults = new int[0];
            setResultAndFinish();
            return;
        }

        //【2.1.4】通过 DevicePolicyManager 获得设备对于权限的配置，然后根据配置信息，更新运行时权限的默认状态！
        DevicePolicyManager devicePolicyManager = getSystemService(DevicePolicyManager.class);
        final int permissionPolicy = devicePolicyManager.getPermissionPolicy(null);
        updateDefaultResults(callingPackageInfo, permissionPolicy);

        //【2.1.5】创建了一个 AppPermissions 对象！
        mAppPermissions = new AppPermissions(this, callingPackageInfo, null, false,
                new Runnable() {
                    @Override
                    public void run() {
                        setResultAndFinish();
                    }
                });

        //【3】接着遍历本次请求的所有运行时权限！
        for (String requestedPermission : mRequestedPermissions) {
            AppPermissionGroup group = null;
            //【3.1】找到该运行时权限所在的 group，我们会授予该组内的所有运行时权限！
            for (AppPermissionGroup nextGroup : mAppPermissions.getPermissionGroups()) {
                if (nextGroup.hasPermission(requestedPermission)) {
                    group = nextGroup;
                    break;
                }
            }
            if (group == null) {
                continue;
            }

            //【2.1.6】处理那些没有被 fix 的需要提醒的权限！
            if (!group.isUserFixed() && !group.isPolicyFixed()) {
                // 根据设备配置，对权限做不同的处理！
                switch (permissionPolicy) {
                    case DevicePolicyManager.PERMISSION_POLICY_AUTO_GRANT: {
                        // 如果设备策略管理设定的是自动授予，
                        // 如果该 group 中有权限未授予，那就授予他们！
                        if (!group.areRuntimePermissionsGranted()) {
                            group.grantRuntimePermissions(false);
                        }
                        group.setPolicyFixed(); // 设置 policy fix 标志，不再提醒！
                    } break;

                    case DevicePolicyManager.PERMISSION_POLICY_AUTO_DENY: {
                        // 如果设备策略管理设定的是自动拒绝，
                        // 如果该 group 中有权限已授予，那就拒绝他们！
                        if (group.areRuntimePermissionsGranted()) {
                            group.revokeRuntimePermissions(false);
                        }
                        group.setPolicyFixed();
                    } break;

                    default: {
                        // 如果设备策略管理设定的是其他策略（也是正常情况），
                        // 如果 group 中有权限未授予，那就将其添加到 mRequestGrantPermissionGroups，接下来要申请！
                        // 如果 group 中有权限已授予，那就授予所有的权限，并更新授予结果；
                        if (!group.areRuntimePermissionsGranted()) {
                            mRequestGrantPermissionGroups.put(group.getName(),
                                    new GroupState(group));
                        } else {
                            group.grantRuntimePermissions(false);
                            updateGrantResults(group); // 更新申请结果
                        }
                    } break;
                }
            } else {
                // 如果权限已经被 fix，不用再提醒，那就直接更新权限的申请结果！
                updateGrantResults(group);
            }
        }

        //【2.1.7】显示权限申请弹窗界面！
        setContentView(mViewHandler.createView());

        Window window = getWindow();
        WindowManager.LayoutParams layoutParams = window.getAttributes();
        mViewHandler.updateWindowAttributes(layoutParams);
        window.setAttributes(layoutParams);

        //【2.1.8】显示权限请求 ui，处理权限的授予和拒绝！
        if (!showNextPermissionGroupGrantRequest()) {
            //【2.1.9】返回处理结果！
            setResultAndFinish();
        }
    }
}
```


### 2.1.1 new GrantPermissionsViewHandlerImpl(this)

有一个成员变量 private GrantPermissionsViewHandler mViewHandler，用于处理权限弹窗的显示和相关逻辑显示，在 onCreate 方法中，首先会创建一个 GrantPermissionsViewHandlerImpl 对象！
```java
public final class GrantPermissionsViewHandlerImpl
        implements GrantPermissionsViewHandler, OnClickListener {
 
    public GrantPermissionsViewHandlerImpl(Context context) {
        mContext = context;
    } 
        
}        
```
同时，又调用了 setResultListener，设置监听器！

```java
    @Override
    public GrantPermissionsViewHandlerImpl setResultListener(ResultListener listener) {
        mResultListener = listener;
        return this;
    }
```
这里的 ResultListener 就是 GrantPermissionsActivity，因为其实现了 ResultListener 接口！


### 2.1.2 GrantPermissionsActivity.setResultAndFinish

当遇到一些特殊情况，不能继续申请权限的时候，我们会调用 setResultAndFinish 结束申请！

```java
    private void setResultAndFinish() {
        setResultIfNeeded(RESULT_OK);
        finish();
    }
```
继续调用了 setResultIfNeeded 方法！
```java
    private void setResultIfNeeded(int resultCode) {
        if (!mResultSet) {
            mResultSet = true;
            logRequestedPermissionGroups();
            Intent result = new Intent(PackageManager.ACTION_REQUEST_PERMISSIONS);
            result.putExtra(PackageManager.EXTRA_REQUEST_PERMISSIONS_NAMES, mRequestedPermissions);
            result.putExtra(PackageManager.EXTRA_REQUEST_PERMISSIONS_RESULTS, mGrantResults);
            // 调用 Activity.setResult 方法！
            setResult(resultCode, result);
        }
    }
```

### 2.1.3 GrantPermissionsActivity.getCallingPackageInfo

该方法用于获得申请运行时权限的应用的 PackageInfo 对象！
```java
    private PackageInfo getCallingPackageInfo() {
        try {
            //【2.1.3.1】最终调用 PMS 的 getPackageInfo 方法！
            return getPackageManager().getPackageInfo(getCallingPackage(),
                    PackageManager.GET_PERMISSIONS);
        } catch (NameNotFoundException e) {
            Log.i(LOG_TAG, "No package: " + getCallingPackage(), e);
            return null;
        }
    }
```
PackageManager.GET_PERMISSIONS 标志是为了限制查询到的数据的量！

#### 2.1.3.1 PackageManagerS.getPackageInfo

flags 传入 PackageManager.GET_PERMISSIONS！
```java
    @Override
    public PackageInfo getPackageInfo(String packageName, int flags, int userId) {
        if (!sUserManager.exists(userId)) return null;
        flags = updateFlagsForPackage(flags, userId, packageName);
        enforceCrossUserPermission(Binder.getCallingUid(), userId,
                false /* requireFullPermission */, false /* checkShell */, "get package info");

        synchronized (mPackages) {
            // 如果 package 重命名过，返回旧名字！
            packageName = normalizePackageNameLPr(packageName);

            final boolean matchFactoryOnly = (flags & MATCH_FACTORY_ONLY) != 0;
            PackageParser.Package p = null;
            if (matchFactoryOnly) {
                final PackageSetting ps = mSettings.getDisabledSystemPkgLPr(packageName);
                if (ps != null) {
                    return generatePackageInfo(ps, flags, userId);
                }
            }
            // 从 PMS.mPackages 中获取扫描解析后的 PackageParser.Package 对象！
            if (p == null) {
                p = mPackages.get(packageName);
                if (matchFactoryOnly && p != null && !isSystemApp(p)) {
                    return null;
                }
            }
            if (DEBUG_PACKAGE_INFO)
                Log.v(TAG, "getPackageInfo " + packageName + ": " + p);
            if (p != null) { 
                //【2.1.3.2】如果有扫描结果，那就调用 generatePackageInfo 返回包信息！
                return generatePackageInfo((PackageSetting)p.mExtras, flags, userId);
            }
            // 如果没有扫描结果，那么我们从上一次的安装信息中获取包信息！
            if (!matchFactoryOnly && (flags & MATCH_UNINSTALLED_PACKAGES) != 0) {
                final PackageSetting ps = mSettings.mPackages.get(packageName);
                return generatePackageInfo(ps, flags, userId);
            }
        }
        return null;
    }
```
流程很简单，不多说了！

#### 2.1.3.2 PackageManagerS.generatePackageInfo
```java
    private PackageInfo generatePackageInfo(PackageSetting ps, int flags, int userId) {
        if (!sUserManager.exists(userId)) return null;
        if (ps == null) {
            return null;
        }
        // 获得扫描信息！
        final PackageParser.Package p = ps.pkg;
        if (p == null) {
            return null;
        }
        // 获得该 package 的权限状态信息！
        final PermissionsState permissionsState = ps.getPermissionsState();

        // 如果 flags 设置了 GET_GIDS，说明我们要获得 gids 信息，那就计算 gid！
        final int[] gids = (flags & PackageManager.GET_GIDS) == 0
                ? EMPTY_INT_ARRAY : permissionsState.computeGids(userId);
        // 获取应用已经被授予的所有权限！
        final Set<String> permissions = ArrayUtils.isEmpty(p.requestedPermissions)
                ? Collections.<String>emptySet() : permissionsState.getPermissions(userId);
        // 获取应用在当前 userId 下的状态信息！
        final PackageUserState state = ps.readUserState(userId);
        //【2.1.3.3】继续调用 generatePackageInfo！
        return PackageParser.generatePackageInfo(p, gids, flags,
                ps.firstInstallTime, ps.lastUpdateTime, permissions, state, userId);
    }
```
继续看：

#### 2.1.3.3 PackageManagerS.generatePackageInfo


```java
    public static PackageInfo generatePackageInfo(PackageParser.Package p,
            int gids[], int flags, long firstInstallTime, long lastUpdateTime,
            Set<String> grantedPermissions, PackageUserState state, int userId) {
        if (!checkUseInstalledOrHidden(flags, state) || !p.isMatch(flags)) {
            return null;
        }
        //【1】创建了一个 PackageInfo 对象，保存应用的包信息的拷贝！
        PackageInfo pi = new PackageInfo();
        pi.packageName = p.packageName;
        pi.splitNames = p.splitNames;
        pi.versionCode = p.mVersionCode;
        pi.baseRevisionCode = p.baseRevisionCode;
        pi.splitRevisionCodes = p.splitRevisionCodes;
        pi.versionName = p.mVersionName;
        pi.sharedUserId = p.mSharedUserId;
        pi.sharedUserLabel = p.mSharedUserLabel;
        pi.applicationInfo = generateApplicationInfo(p, flags, state, userId);
        pi.installLocation = p.installLocation;
        pi.coreApp = p.coreApp;
        if ((pi.applicationInfo.flags&ApplicationInfo.FLAG_SYSTEM) != 0
                || (pi.applicationInfo.flags&ApplicationInfo.FLAG_UPDATED_SYSTEM_APP) != 0) {
            pi.requiredForAllUsers = p.mRequiredForAllUsers;
        }
        pi.restrictedAccountType = p.mRestrictedAccountType;
        pi.requiredAccountType = p.mRequiredAccountType;
        pi.overlayTarget = p.mOverlayTarget;
        pi.firstInstallTime = firstInstallTime;
        pi.lastUpdateTime = lastUpdateTime;
        //【2】根据 flags 设置的标志位，获取标志为对应的信息！
        if ((flags&PackageManager.GET_GIDS) != 0) {
            pi.gids = gids;
        }
        if ((flags&PackageManager.GET_CONFIGURATIONS) != 0) {
            int N = p.configPreferences != null ? p.configPreferences.size() : 0;
            if (N > 0) {
                pi.configPreferences = new ConfigurationInfo[N];
                p.configPreferences.toArray(pi.configPreferences);
            }
            N = p.reqFeatures != null ? p.reqFeatures.size() : 0;
            if (N > 0) {
                pi.reqFeatures = new FeatureInfo[N];
                p.reqFeatures.toArray(pi.reqFeatures);
            }
            N = p.featureGroups != null ? p.featureGroups.size() : 0;
            if (N > 0) {
                pi.featureGroups = new FeatureGroupInfo[N];
                p.featureGroups.toArray(pi.featureGroups);
            }
        }
        if ((flags & PackageManager.GET_ACTIVITIES) != 0) {
            final int N = p.activities.size();
            if (N > 0) {
                int num = 0;
                final ActivityInfo[] res = new ActivityInfo[N];
                for (int i = 0; i < N; i++) {
                    final Activity a = p.activities.get(i);
                    if (state.isMatch(a.info, flags)) {
                        res[num++] = generateActivityInfo(a, flags, state, userId);
                    }
                }
                pi.activities = ArrayUtils.trimToSize(res, num);
            }
        }
        if ((flags & PackageManager.GET_RECEIVERS) != 0) {
            final int N = p.receivers.size();
            if (N > 0) {
                int num = 0;
                final ActivityInfo[] res = new ActivityInfo[N];
                for (int i = 0; i < N; i++) {
                    final Activity a = p.receivers.get(i);
                    if (state.isMatch(a.info, flags)) {
                        res[num++] = generateActivityInfo(a, flags, state, userId);
                    }
                }
                pi.receivers = ArrayUtils.trimToSize(res, num);
            }
        }
        if ((flags & PackageManager.GET_SERVICES) != 0) {
            final int N = p.services.size();
            if (N > 0) {
                int num = 0;
                final ServiceInfo[] res = new ServiceInfo[N];
                for (int i = 0; i < N; i++) {
                    final Service s = p.services.get(i);
                    if (state.isMatch(s.info, flags)) {
                        res[num++] = generateServiceInfo(s, flags, state, userId);
                    }
                }
                pi.services = ArrayUtils.trimToSize(res, num);
            }
        }
        if ((flags & PackageManager.GET_PROVIDERS) != 0) {
            final int N = p.providers.size();
            if (N > 0) {
                int num = 0;
                final ProviderInfo[] res = new ProviderInfo[N];
                for (int i = 0; i < N; i++) {
                    final Provider pr = p.providers.get(i);
                    if (state.isMatch(pr.info, flags)) {
                        res[num++] = generateProviderInfo(pr, flags, state, userId);
                    }
                }
                pi.providers = ArrayUtils.trimToSize(res, num);
            }
        }
        if ((flags&PackageManager.GET_INSTRUMENTATION) != 0) {
            int N = p.instrumentation.size();
            if (N > 0) {
                pi.instrumentation = new InstrumentationInfo[N];
                for (int i=0; i<N; i++) {
                    pi.instrumentation[i] = generateInstrumentationInfo(
                            p.instrumentation.get(i), flags);
                }
            }
        }
        // 因为我们只设置了 GET_PERMISSIONS，所以只会手机和权限相关的信息！
        if ((flags&PackageManager.GET_PERMISSIONS) != 0) {
            // 处理该应用定义的所有权限！
            int N = p.permissions.size();
            if (N > 0) {
                pi.permissions = new PermissionInfo[N];
                for (int i=0; i<N; i++) {
                    pi.permissions[i] = generatePermissionInfo(p.permissions.get(i), flags);
                }
            }
            // 处理该应用请求的所有权限！
            N = p.requestedPermissions.size();
            if (N > 0) {
                pi.requestedPermissions = new String[N];
                pi.requestedPermissionsFlags = new int[N];
                for (int i=0; i<N; i++) {
                    final String perm = p.requestedPermissions.get(i);
                    // requestedPermissions[i] 表示应用申请的权限名；
                    pi.requestedPermissions[i] = perm;
                    // requestedPermissionsFlags[i] 表示应用申请的权限的授予情况；
                    // 开始默认初始化为需要再次请求，然后比较该权限是否已经被授予，如果授予，改为授予状态！
                    pi.requestedPermissionsFlags[i] |= PackageInfo.REQUESTED_PERMISSION_REQUIRED;
                    if (grantedPermissions != null && grantedPermissions.contains(perm)) {
                        pi.requestedPermissionsFlags[i] |= PackageInfo.REQUESTED_PERMISSION_GRANTED;
                    }
                }
            }
        }
        if ((flags&PackageManager.GET_SIGNATURES) != 0) {
           int N = (p.mSignatures != null) ? p.mSignatures.length : 0;
           if (N > 0) {
                pi.signatures = new Signature[N];
                System.arraycopy(p.mSignatures, 0, pi.signatures, 0, N);
            }
        }
        return pi;
    }
```
最后，返回的是一个 PackageInfo 对象！


### 2.1.4 GrantPermissionsActivity.updateDefaultResults

更新下权限的默认授予情况！
```java
    private void updateDefaultResults(PackageInfo callingPackageInfo, int permissionPolicy) {
        final int requestedPermCount = mRequestedPermissions.length;
        for (int i = 0; i < requestedPermCount; i++) {
            String permission = mRequestedPermissions[i];
            //【2.1.3.1 】如果申请应用的信息存在，那么会调用 computePermissionGrantState 方法，获得权限的默认授予状态！
            // 否则，所有运行时权限默认为 deny！
            mGrantResults[i] = callingPackageInfo != null
                    ? computePermissionGrantState(callingPackageInfo, permission, permissionPolicy)
                    : PERMISSION_DENIED;
        }
    }
```
关键的实现是在 computePermissionGrantState 中！

#### 2.1.4.1 GrantPermissionsActivity.computePermissionGrantState

该方法用于根据参数，计算该运行时权限的授予情况，这里的 permission 是应用本次申请的运行时权限！
```java
    private int computePermissionGrantState(PackageInfo callingPackageInfo,
            String permission, int permissionPolicy) {
        boolean permissionRequested = false;
        //【1】如果运行时权限 permission 在 package 请求权限列表中
        // 并且已经授予，那么就返回 PERMISSION_GRANTED！
        for (int i = 0; i < callingPackageInfo.requestedPermissions.length; i++) {
            if (permission.equals(callingPackageInfo.requestedPermissions[i])) {
                permissionRequested = true;
                if ((callingPackageInfo.requestedPermissionsFlags[i]
                        & PackageInfo.REQUESTED_PERMISSION_GRANTED) != 0) {
                    return PERMISSION_GRANTED;
                }
                break;
            }
        }
        //【2】申请的运行时权限，应用没有申明，返回 PERMISSION_DENIED！
        if (!permissionRequested) {
            return PERMISSION_DENIED;
        }
        //【3】只有基本权限类型为 dangerous 的权限才需要动态申请，其他类型的权限，默认返回 PERMISSION_DENIED！
        try {
            PermissionInfo pInfo = getPackageManager().getPermissionInfo(permission, 0);
            if ((pInfo.protectionLevel & PermissionInfo.PROTECTION_MASK_BASE)
                    != PermissionInfo.PROTECTION_DANGEROUS) {
                return PERMISSION_DENIED;
            }
        } catch (NameNotFoundException e) {
            return PERMISSION_DENIED;
        }

        //【4】根据 permissionPolicy 的值来处理权限的授予情况，如果是 PERMISSION_POLICY_AUTO_GRANT，表示自动授予；
        // 那么返回 PERMISSION_GRANTED，否则返回 PERMISSION_DENIED！
        switch (permissionPolicy) {
            case DevicePolicyManager.PERMISSION_POLICY_AUTO_GRANT: {
                return PERMISSION_GRANTED;
            }
            default: {
                return PERMISSION_DENIED;
            }
        }
    }
```
最后 computePermissionGrantState 的结果会保存 mGrantResults 数组中！

不多说了！

### 2.1.5 new AppPermissions - 封装 App 的权限请求信息

这里的 onErrorCallback 是一个 Runnable，执行后会触发 setResultAndFinish 方法！

```java
    public AppPermissions(Context context, PackageInfo packageInfo, String[] permissions,
            boolean sortGroups, Runnable onErrorCallback) {
        mContext = context;
        mPackageInfo = packageInfo; // 请求权限的应用的 PackageInfo 实例！
        mFilterPermissions = permissions; // 传入 null；
        mAppLabel = BidiFormatter.getInstance().unicodeWrap(
                packageInfo.applicationInfo.loadSafeLabel(
                context.getPackageManager()).toString());
        mSortGroups = sortGroups; // 传入 false；
        mOnErrorCallback = onErrorCallback;

        //【2.1.4.1】加载权限组信息！
        loadPermissionGroups();
    }
```

mFilterPermissions 属性表示要过滤的权限，这里我们传入的是 null；

同时，AppPermissions 还有如下的属性：

```java
    // 保存所有运行时权限所属的组信息 AppPermissionGroup
    private final ArrayList<AppPermissionGroup> mGroups = new ArrayList<>();
    // 保存组名和 AppPermissionGroup 的映射关系
    private final LinkedHashMap<String, AppPermissionGroup> mNameToGroupMap = new LinkedHashMap<>();
```

#### 2.1.5.1 AppPermissions.loadPermissionGroups

加载权限组信息

```java
    private void loadPermissionGroups() {
        mGroups.clear();

        if (mPackageInfo.requestedPermissions == null) {
            return;
        }

        if (mFilterPermissions != null) { //【1】mFilterPermissions 为 null，不进入！
            for (String filterPermission : mFilterPermissions) {
                for (String requestedPerm : mPackageInfo.requestedPermissions) {
                    if (!filterPermission.equals(requestedPerm)) { // 过滤掉那些不是 filterPermission 的权限！
                        continue;
                    }

                    if (hasGroupForPermission(requestedPerm)) {
                        break;
                    }

                    AppPermissionGroup group = AppPermissionGroup.create(mContext,
                            mPackageInfo, requestedPerm);
                    if (group == null) {
                        break;
                    }

                    mGroups.add(group);
                    break;
                }
            }
        } else {
            //【2】遍历该 package 请求的所有权限（安装/运行），
            for (String requestedPerm : mPackageInfo.requestedPermissions) {
                //【2.1.4.1.1】判断是否已经有对应的 group 存在，存在的话，那就不处理！
                if (hasGroupForPermission(requestedPerm)) {
                    continue;
                }
                //【2.1.4.1.2】如果不存在，那就创建一个 AppPermissionGroup 对象，封装运行时权限所属的组信息！
                AppPermissionGroup group = AppPermissionGroup.create(mContext,
                        mPackageInfo, requestedPerm);
                if (group == null) {
                    continue;
                }

                mGroups.add(group); //【2.1】添加到 mGroups 中！
            }
        }

        if (mSortGroups) { //【3】如果 mSortGroups 为 true，表示对 group 进行排序！
            Collections.sort(mGroups);
        }
        
        //【4】mNameToGroupMap 用于保存组名和 AppPermissionGroup 的映射关系！
        mNameToGroupMap.clear();
        for (AppPermissionGroup group : mGroups) {
            mNameToGroupMap.put(group.getName(), group);
        }
    }
```
对于 mPackageInfo.requestedPermissions，不仅包含运行时权限，还包含安装时权限！

##### 2.1.5.1.1 AppPermissions.hasGroupForPermission

加载权限组信息！
```java
    private boolean hasGroupForPermission(String permission) {
        //【1】遍历所有的 AppPermissionGroup，如果 permission 在 group 的权限集合中，返回 true！
        for (AppPermissionGroup group : mGroups) {
            if (group.hasPermission(permission)) {
                return true;
            }
        }
        return false;
    }
```
AppPermissions 内部有一个 mGroups，保存所有的权限组信息 AppPermissionGroup 实例！

```java
public final class AppPermissionGroup implements Comparable<AppPermissionGroup> {
    private final ArrayMap<String, Permission> mPermissions = new ArrayMap<>();
    //【2】在该 group 中是否有该权限！
    public boolean hasPermission(String permission) {
        return mPermissions.get(permission) != null;
    }
}
```
AppPermissionGroup 内部有一个 mPermissions 哈希表，用于保存该 group 中所有权限的权限名和其 Permission 的映射关系！

##### 2.1.5.1.2 AppPermissionGroup.create[3] - 创建运行时权限组信息

接下来我们看下，AppPermissionGroup 的创建，String permissionName 表示的是该 package 的申请的权限：

```java
    public static AppPermissionGroup create(Context context, PackageInfo packageInfo,
            String permissionName) {
        //【1】获得运行时权限的信息，找不到，返回 null；
        PermissionInfo permissionInfo;
        try {
            permissionInfo = context.getPackageManager().getPermissionInfo(permissionName, 0);
        } catch (PackageManager.NameNotFoundException e) {
            return null;
        }
        //【2】校验权限的类型，如果该权限的基本类型不是 dangerous 或者说权限的 flags 没有设置 FLAG_INSTALLED 属性
        // 或者权限的 flags 设置了 FLAG_REMOVED 属性，那么，返回 null!
        if (permissionInfo.protectionLevel != PermissionInfo.PROTECTION_DANGEROUS
                || (permissionInfo.flags & PermissionInfo.FLAG_INSTALLED) == 0
                || (permissionInfo.flags & PermissionInfo.FLAG_REMOVED) != 0) {
            return null;
        }
        
        //【3】查询该权限所属 group 的信息，如果权限的 group 属性不为 null，说明其有所属组；
        PackageItemInfo groupInfo = permissionInfo;
        if (permissionInfo.group != null) {
            try {
                groupInfo = context.getPackageManager().getPermissionGroupInfo(
                        permissionInfo.group, 0);
            } catch (PackageManager.NameNotFoundException e) {
                /* ignore */
            }
        }

        //【4】查询所有属于该 group 的权限信息，保存到 permissionInfos 列表中！！
        // 注意这里的 permissionInfos 会包含应用没有申请的权限！
        List<PermissionInfo> permissionInfos = null;
        if (groupInfo instanceof PermissionGroupInfo) {
            try {
                permissionInfos = context.getPackageManager().queryPermissionsByGroup(
                        groupInfo.name, 0);
            } catch (PackageManager.NameNotFoundException e) {
                /* ignore */
            }
        }

        //【2.1.4.1.3】创建 AppPermissionGroup 对象！！
        return create(context, packageInfo, groupInfo, permissionInfos,
                Process.myUserHandle());
    }
```
这里可以看到，流程【2】中，会排除掉所有的非 dangerous 类型的权限！

在创建一个 group 的时候，会将属于该 group 的所有权限都查出来！！


这里继续调用了五参数的 create 方法！

##### 2.1.5.1.3 AppPermissionGroup.create[5]

继续调用 create 方法来创建 AppPermissionGroup 对象！

参数分析：

- **PackageInfo packageInfo**：本次请求权限的 package 的 PackageInfo 对象；
- **PackageItemInfo groupInfo**：该 package 的请求的权限所属的组信息；
- **List<PermissionInfo> permissionInfos**：属于该组的所有权限信息；

```java
    public static AppPermissionGroup create(Context context, PackageInfo packageInfo,
            PackageItemInfo groupInfo, List<PermissionInfo> permissionInfos,
            UserHandle userHandle) {
        //【2.1.5.1.3.1】创建一个 AppPermissionGroup 对象，封装了 group name，label 相关信息！
        AppPermissionGroup group = new AppPermissionGroup(context, packageInfo, groupInfo.name,
                groupInfo.packageName, groupInfo.loadLabel(context.getPackageManager()),
                loadGroupDescription(context, groupInfo), groupInfo.packageName, groupInfo.icon,
                userHandle);

        // 如果 groupInfo 不是权限组对象，而是一个运行时权限，那就添加到 permissionInfos 中！
        if (groupInfo instanceof PermissionInfo) {
            permissionInfos = new ArrayList<>();
            permissionInfos.add((PermissionInfo) groupInfo);
        }

        //【1】如果权限组没有权限，返回 null，该 group 无效！
        if (permissionInfos == null || permissionInfos.isEmpty()) {
            return null;
        }

        //【2】遍历该 package 请求的所有权限（安装/运行）！
        final int permissionCount = packageInfo.requestedPermissions.length;
        for (int i = 0; i < permissionCount; i++) {
            String requestedPermission = packageInfo.requestedPermissions[i];

            PermissionInfo requestedPermissionInfo = null;
            //【2.1】如果应用请求的权限（安装/运行）中有权限属于该权限组，我们会处理该权限！
            // permissionInfos 是该权限组中的所有权限，这里我们要过滤掉非应用申请的权限！
            for (PermissionInfo permissionInfo : permissionInfos) {
                if (requestedPermission.equals(permissionInfo.name)) {
                    requestedPermissionInfo = permissionInfo;
                    break;
                }
            }

            if (requestedPermissionInfo == null) {
                continue;
            }

            //【2.2】我们只处理运行时权限，对于非 dangerous 权限，直接跳过！！
            if (requestedPermissionInfo.protectionLevel != PermissionInfo.PROTECTION_DANGEROUS) {
                continue;
            }

            // Don't allow toggling non-platform permission groups for legacy apps via app ops.
            // 如果应用程序的 targetSdkVersion 不高于 Android 5.1，并且定义权限组的 package 不是 android！
            // 不处理该运行时权限！
            if (packageInfo.applicationInfo.targetSdkVersion <= Build.VERSION_CODES.LOLLIPOP_MR1
                    && !PLATFORM_PACKAGE_NAME.equals(groupInfo.packageName)) {
                continue;
            }

            //【2.3】判断该运行时权限是否已经授予！
            final boolean granted = (packageInfo.requestedPermissionsFlags[i]
                    & PackageInfo.REQUESTED_PERMISSION_GRANTED) != 0;

            //【2.4】当该运行时权限的定义者不为系统时，appOp 为 null，如果为系统权限的话，匹配其对应的 AppOps，
            // 也有可能为 null！
            final String appOp = PLATFORM_PACKAGE_NAME.equals(requestedPermissionInfo.packageName)
                    ? AppOpsManager.permissionToOp(requestedPermissionInfo.name) : null;
            //【2.5】如果该系统权限有对应的 AppOps，获得其模式的值是否为 ALLOWED!
            final boolean appOpAllowed = appOp != null
                    && context.getSystemService(AppOpsManager.class).checkOpNoThrow(appOp,
                    packageInfo.applicationInfo.uid, packageInfo.packageName)
                    == AppOpsManager.MODE_ALLOWED;
            
            //【2.6】获得权限的 flags，可以取值为 user set,user fixed 等等！
            final int flags = context.getPackageManager().getPermissionFlags(
                    requestedPermission, packageInfo.packageName, userHandle);

            //【2.1.5.1.3.2】创建一个 Permission 对象，保存该运行时权限的信息！
            Permission permission = new Permission(requestedPermission, granted,
                    appOp, appOpAllowed, flags);
            //【2.1.5.1.3.3】添加到 AppPermissionGroup 中去！
            group.addPermission(permission);
        }

        return group;
    }
```

到这里，AppPermissionGroup 就正式创建完了！

###### 2.1.5.1.3.1 new AppPermissionGroup

创建了一个 AppPermissionGroup，封装运行时权限所在权限组信息！
```java
public final class AppPermissionGroup implements Comparable<AppPermissionGroup> {
    ... ... ...
    private AppPermissionGroup(Context context, PackageInfo packageInfo, String name,
            String declaringPackage, CharSequence label, CharSequence description,
            String iconPkg, int iconResId, UserHandle userHandle) {
        mContext = context;
        mUserHandle = userHandle;
        mPackageManager = mContext.getPackageManager();
        mPackageInfo = packageInfo;
        mAppSupportsRuntimePermissions = packageInfo.applicationInfo
                .targetSdkVersion > Build.VERSION_CODES.LOLLIPOP_MR1; // 该应用是否支持运行时权限！
        mAppOps = context.getSystemService(AppOpsManager.class);
        mActivityManager = context.getSystemService(ActivityManager.class);
        mDeclaringPackage = declaringPackage;
        mName = name;
        mLabel = label;
        mDescription = description;
        if (iconResId != 0) {
            mIconPkg = iconPkg;
            mIconResId = iconResId;
        } else {
            mIconPkg = context.getPackageName();
            mIconResId = R.drawable.ic_perm_device_info;
        }
    }
    ... ... ...
}
```
AppPermissionGroup 内部有一个 mPermissions 集合，负责管理该 group 中该 package 申请的权限！

###### 2.1.5.1.3.2 new Permission

该 Permission 是 PackageInstaller 定义的一个类，和系统中的不一样！

Permission 用来封装一个运行时权限信息！
```java
public final class Permission {
    private final String mName;
    private final String mAppOp;

    private boolean mGranted; // 是否已经授予该权限！
    private boolean mAppOpAllowed;
    private int mFlags;

    public Permission(String name, boolean granted,
            String appOp, boolean appOpAllowed, int flags) {
        mName = name; //
        mGranted = granted;
        mAppOp = appOp;
        mAppOpAllowed = appOpAllowed;
        mFlags = flags;
    }
```

###### 2.1.5.1.3.3 AppPermissionGroup.addPermission

将该运行时权限添加到 AppPermissionGroup.mPermissions 中去！
```java
    private void addPermission(Permission permission) {
        mPermissions.put(permission.getName(), permission);
    }
```


### 2.1.6 处理运行时权限组

```java
        //【3】接着遍历本次请求的所有运行时权限！
        for (String requestedPermission : mRequestedPermissions) {
            AppPermissionGroup group = null;
            //【3.1】找到该运行时权限所在的 group!！
            for (AppPermissionGroup nextGroup : mAppPermissions.getPermissionGroups()) {
                if (nextGroup.hasPermission(requestedPermission)) {
                    group = nextGroup;
                    break;
                }
            }
            if (group == null) {
                continue;
            }

            //【2.1.6.1】处理那些 no fix 的需要提醒的权限！
            if (!group.isUserFixed() && !group.isPolicyFixed()) {
                // 根据设备配置，对权限做不同的处理！
                switch (permissionPolicy) {
                    case DevicePolicyManager.PERMISSION_POLICY_AUTO_GRANT: {
                        // 如果设备策略管理设定的是自动授予，
                        //【2.1.6.2】且该 group 中有权限未授予，那就授予他们！
                        if (!group.areRuntimePermissionsGranted()) {
                            //【3.1】授予权限
                            group.grantRuntimePermissions(false);
                        }
                        group.setPolicyFixed(); // 设置 policy fix 标志位
                    } break;

                    case DevicePolicyManager.PERMISSION_POLICY_AUTO_DENY: {
                        // 如果设备策略管理设定的是自动拒绝，
                        //【2.1.6.2】且该 group 中有权限已授予，那就拒绝他们！
                        if (group.areRuntimePermissionsGranted()) {
                            //【3.2】撤掉权限
                            group.revokeRuntimePermissions(false);
                        }
                        group.setPolicyFixed();  // 设置 policy fix 标志位
                    } break;

                    default: {
                        // 如果设备策略管理设定的是其他策略（正常情况），
                        // 如果 group 中有权限未授予，那就将其添加到 mRequestGrantPermissionGroups，接下来要申请！
                        // 如果 group 中有权限已授予，那就授予所有的权限，并更新授予结果；
                        if (!group.areRuntimePermissionsGranted()) { // 同【2.1.6.2】
                            mRequestGrantPermissionGroups.put(group.getName(),
                                    new GroupState(group));
                        } else {
                            //【3.1】直接授予运行时权限！
                            group.grantRuntimePermissions(false);
                            updateGrantResults(group); // 更新申请结果！
                        }
                    } break;
                }
            } else {
                // 如果权限已经被 fix，不用再提醒，那就直接更新权限的申请结果！
                updateGrantResults(group);
            }
            ... ... ... ...
        }
```

当然，一般情况下，我们会进入 default 分支！

这里的 mRequestGrantPermissionGroups 用于保存所有的需要申请的 group 信息：

```java
    private static final class GroupState {
        static final int STATE_UNKNOWN = 0;
        static final int STATE_ALLOWED = 1;
        static final int STATE_DENIED = 2;

        final AppPermissionGroup mGroup;
        int mState = STATE_UNKNOWN;

        GroupState(AppPermissionGroup group) {
            mGroup = group;
        }
    }
```
AppPermissionGroup 会被封装为一个 GroupState 对象，GroupState.mState 保存了该组的授予情况！

#### 2.1.6.1 AppPermissionGroup.isUserFixed/isPolicyFixed

判断该权限组中是否包含没被 user fix 的权限，如果包含，返回 false！

```java
    public boolean isUserFixed() {
        final int permissionCount = mPermissions.size();
        for (int i = 0; i < permissionCount; i++) {
            Permission permission = mPermissions.valueAt(i);
            //【1】该权限组只要有一个 no user fix 的权限，那个这个组就是 no user fix 的！
            if (!permission.isUserFixed()) {
                return false;
            }
        }
        return true;
    }
```
调用了 Permission.isUserFixed 方法！
```java
    public boolean isUserFixed() {
        return (mFlags & PackageManager.FLAG_PERMISSION_USER_FIXED) != 0;
    }
```

同样的，isPolicyFixed 也是类似的作用，判断该权限组中是否包含没有被 policy fix 的权限！
```java
    public boolean isPolicyFixed() {
        final int permissionCount = mPermissions.size();
        for (int i = 0; i < permissionCount; i++) {
            Permission permission = mPermissions.valueAt(i);
            //【1】该权限组中没有任何一个 policy fix 的权限，那个这个组才是 no policy fix 的！
            if (permission.isPolicyFixed()) {
                return true;
            }
        }
        return false;
    }
```
这是因为 policy fix 属于系统的默认授予机制，针对整个组的权限！

同样的调用了 Permission.isPolicyFixed 方法！
```java
    public boolean isPolicyFixed() {
        return (mFlags & PackageManager.FLAG_PERMISSION_POLICY_FIXED) != 0;
    }
```


#### 2.1.6.2 AppPermissionGroup.areRuntimePermissionsGranted

该方法用户判断该 group 中是否有运行时权限被授予！
```java
    public boolean areRuntimePermissionsGranted() {
        return areRuntimePermissionsGranted(null);
    }
```
接着会调用一参数的 areRuntimePermissionsGranted 方法，参数的意思不多说了！
```java
    public boolean areRuntimePermissionsGranted(String[] filterPermissions) {
        if (LocationUtils.isLocationGroupAndProvider(mName, mPackageInfo.packageName)) {
            return LocationUtils.isLocationEnabled(mContext);
        }
        //【1】遍历该组中的所有运行时权限，根据前面知道，该组中的权限都是该 package 申请的！
        final int permissionCount = mPermissions.size();
        for (int i = 0; i < permissionCount; i++) {
            Permission permission = mPermissions.valueAt(i);
            if (filterPermissions != null // 如果 filterPermissions 不为 null，那就只处理其内部的权限！
                    && !ArrayUtils.contains(filterPermissions, permission.getName())) {
                continue;
            }
            //【1.1】如果应用支持运行时权限机制，并且该权限已经被授予，那就返回 true！
            if (mAppSupportsRuntimePermissions) {
                if (permission.isGranted()) {
                    return true;
                }
            } else if (permission.isGranted() && (permission.getAppOp() == null
                    || permission.isAppOpAllowed())) {
                return true;
            }
        }
        return false; // 如果该 group 中没有运行时权限被授予，返回 false！
    }
```
如果有一个运行时权限被授予，那么就会返回 true！

如果没有一个运行时权限被授予，那么就会返回 false！

### 2.1.7 GPViewHandlerImpl.createView

接着，通过 setContentView，创建布局界面！
```java
    @Override
    public View createView() {
        mRootView = (ManualLayoutFrame) LayoutInflater.from(mContext)
                .inflate(R.layout.grant_permissions, null);
        mButtonBar = (ButtonBarLayout) mRootView.findViewById(R.id.button_group);
        mButtonBar.setAllowStacking(true);
        mMessageView = (TextView) mRootView.findViewById(R.id.permission_message);
        mIconView = (ImageView) mRootView.findViewById(R.id.permission_icon);
        mCurrentGroupView = (TextView) mRootView.findViewById(R.id.current_page_text);
        mDoNotAskCheckbox = (CheckBox) mRootView.findViewById(R.id.do_not_ask_checkbox);
        mAllowButton = (Button) mRootView.findViewById(R.id.permission_allow_button);

        mDialogContainer = (ViewGroup) mRootView.findViewById(R.id.dialog_container);
        mDescContainer = (ViewGroup) mRootView.findViewById(R.id.desc_container);
        mCurrentDesc = (ViewGroup) mRootView.findViewById(R.id.perm_desc_root);

        mAllowButton.setOnClickListener(this); // 授予按钮！
        mRootView.findViewById(R.id.permission_deny_button).setOnClickListener(this); // 拒绝按钮！
        mDoNotAskCheckbox.setOnClickListener(this); // 不在提醒按钮！

        if (mGroupName != null) { // mGroupName 用于恢复界面显示！
            updateDescription();
            updateGroup();
            updateDoNotAskCheckBox();
        }

        return mRootView;
    }
```


我们看到，这里给三个按钮绑定的点击事件！

```java
    @Override
    public void onClick(View view) {
        switch (view.getId()) {
            case R.id.permission_allow_button: // 授予权限
                if (mResultListener != null) {
                    view.clearAccessibilityFocus();
                    mResultListener.onPermissionGrantResult(mGroupName, true, false); 
                }
                break;
            case R.id.permission_deny_button: // 拒绝权限
                mAllowButton.setEnabled(true);
                if (mResultListener != null) {
                    view.clearAccessibilityFocus();
                    mResultListener.onPermissionGrantResult(mGroupName, false,
                            mShowDonNotAsk && mDoNotAskCheckbox.isChecked());
                }
                break;
            case R.id.do_not_ask_checkbox: // 不再提醒
                mAllowButton.setEnabled(!mDoNotAskCheckbox.isChecked());
                break;
        }
    }
```


当我们点击 allow 或者 deny 后，会回调 mResultListener.onPermissionGrantResult 方法！

当我们点击授予后，执行：
```java
mResultListener.onPermissionGrantResult(mGroupName, true, false); 
```
当我们点击拒绝后，执行：
```java
mResultListener.onPermissionGrantResult(mGroupName, false, mShowDonNotAsk && mDoNotAskCheckbox.isChecked());
```
- **onPermissionGrantResult 参数解析**

onPermissionGrantResult 的第二个参数 granted 表示授予或者拒绝，为 true 表示授予，为 false 表示拒绝；

onPermissionGrantResult 的第三个参数 doNotAskAgain 表示是否不再询问，为 true 表示不再询问，为 false 需再次询问；


- **onPermissionGrantResult 参数处理**
 
  - 如果用户本次授予了权限，那么 doNotAskAgain 为 false，意味着下次会继续询问；
  - 如果用户本次拒绝了权限，但是没有选择不再提醒，那么 doNotAskAgain 为 false，意味着下次会继续询问；
  - 如果用户本次拒绝了权限，但是同时选择不再提醒，那么 doNotAskAgain 为 true，那么下次就不会再提醒询问了；

当我们点击不再提醒 CheckBox 后，可以看到，授予按钮是无法点击的！

#### 2.1.7.1 GrantPermissionsActivity.onPermissionGrantResult

mResultListener 实际上就是 GrantPermissionsActivity，我们去看看 onPermissionGrantResult 方法！

```java
    @Override
    public void onPermissionGrantResult(String name, boolean granted, boolean doNotAskAgain) {
        //【1】根据 name 获得对应的运行时权限组！
        GroupState groupState = mRequestGrantPermissionGroups.get(name);
        if (groupState.mGroup != null) {
            if (granted) { 
                //【2.1.6.3】处理授予情况！
                groupState.mGroup.grantRuntimePermissions(doNotAskAgain); 
                groupState.mState = GroupState.STATE_ALLOWED; // 更新 group 状态！
            } else { 
                //【2.1.6.4】处理拒绝情况！
                groupState.mGroup.revokeRuntimePermissions(doNotAskAgain);
                groupState.mState = GroupState.STATE_DENIED;
            }
            updateGrantResults(groupState.mGroup); // 更新结果！
        }

        if (!showNextPermissionGroupGrantRequest()) { //【2.1.8】继续下一个组的处理！
            setResultAndFinish(); //【2.1.9】处理请求结果，结束！
        }
    }
```
可以看到，对于每个组的运行时权限，最后调用的是 setResultAndFinish 会将权限处理的结果返回给请求的 activity！

### 2.1.8 GPViewHandlerImpl.showNextPermissionGroupGrantRequest

接下来，就是请求运行时权限了！mRequestGrantPermissionGroups 中保存了所有需要请求的运行时权限组！

```java
    private boolean showNextPermissionGroupGrantRequest() {
        final int groupCount = mRequestGrantPermissionGroups.size();

        int currentIndex = 0;
        for (GroupState groupState : mRequestGrantPermissionGroups.values()) {
            if (groupState.mState == GroupState.STATE_UNKNOWN) {
                CharSequence appLabel = mAppPermissions.getAppLabel();
                Spanned message = Html.fromHtml(getString(R.string.permission_warning_template,
                        appLabel, groupState.mGroup.getDescription()), 0);

                //【1】将权限信息作为 titile！
                setTitle(message);

                Resources resources;
                try {
                    resources = getPackageManager().getResourcesForApplication(
                            groupState.mGroup.getIconPkg());
                } catch (NameNotFoundException e) {
                    // Fallback to system.
                    resources = Resources.getSystem();
                }
                int icon = groupState.mGroup.getIconResId();

                //【2.1.8.1】更新 ui 显示！
                mViewHandler.updateUi(groupState.mGroup.getName(), groupCount, currentIndex,
                        Icon.createWithResource(resources, icon), message,
                        groupState.mGroup.isUserSet());
                return true;
            }

            currentIndex++;
        }

        return false;
    }
```
最后会调用了 updateUi 方法，更新 ui 显示！

updateUi 的最后一个参数：boolean showDonNotAsk 表示是否显示　“不再提醒”　的 CheckBox，为 true，表示不显示，其值取决于：
```java
groupState.mGroup.isUserSet()
```
我们去看看 group 的 isUserSet 方法！
```java
public boolean isUserSet() {
    final int permissionCount = mPermissions.size();
    for (int i = 0; i < permissionCount; i++) {
        Permission permission = mPermissions.valueAt(i);
        if (!permission.isUserSet()) {
            return false;
        }
    }
    return true;
}
```
也就是说，取决于该 group 中该应用请求的运行时权限是否都是 user set 的，即：**之前用户是否拒绝授予过权限**，只有之前拒绝过并且没有选择不在提醒，下一次提醒用户时，才会显示 checkbox！

#### 2.1.8.1 GPViewHandlerImpl.updateUi

showDonNotAsk 的取值为 groupState.mGroup.isUserSet()，就似乎说，如果用户之前拒绝了该权限并且没有选择不再提醒，那么再次提示时 showDonNotAsk 为 true！

```java
    @Override
    public void updateUi(String groupName, int groupCount, int groupIndex, Icon icon,
            CharSequence message, boolean showDonNotAsk) {
        //【1】保存当前的显示内容和进度！
        mGroupName = groupName; // group 名！
        mGroupCount = groupCount; // group 的总数！
        mGroupIndex = groupIndex; // 当前处理的 group！ 
        mGroupIcon = icon;
        mGroupMessage = message;
        mShowDonNotAsk = showDonNotAsk; // 是否显示 checkBox
        mDoNotAskChecked = false; // 是否不再提醒！

        // If this is a second (or later) permission and the views exist, then animate.
        if (mIconView != null) {
            if (mGroupIndex > 0) {
                // The first message will be announced as the title of the activity, all others
                // we need to announce ourselves.
                mDescContainer.announceForAccessibility(message);
                animateToPermission();
            } else {
                updateDescription();
                updateGroup();
                updateDoNotAskCheckBox();
            }
        }
    }
```
接下来，就会显示出权限弹窗了！

用户会通过点击弹窗中的按钮，授予或者拒绝权限！

##### 2.1.8.1.1 GPViewHandlerImpl.updateDescription
```java
    private void updateDescription() {
        mIconView.setImageDrawable(mGroupIcon.loadDrawable(mContext));
        mMessageView.setText(mGroupMessage);
    }

```
##### 2.1.8.1.2 GPViewHandlerImpl.updateGroup

```java
    private void updateGroup() {
        if (mGroupCount > 1) {
            mCurrentGroupView.setVisibility(View.VISIBLE);
            mCurrentGroupView.setText(mContext.getString(R.string.current_permission_template,
                    mGroupIndex + 1, mGroupCount));
        } else {
            mCurrentGroupView.setVisibility(View.INVISIBLE);
        }
    }
```

##### 2.1.8.1.3 GPViewHandlerImpl.updateDoNotAskCheckBox

是否显示 CheckBox 取决于 mShowDonNotAsk，如果 mShowDonNotAsk 为 true，那就显示 checkBox！

CheckBox 的默认状态则是由 mDoNotAskChecked 来决定，默认 mDoNotAskChecked 为 false，此时 CheckBox 没有被选中！

同时 CheckBox 的点击也会修改 mDoNotAskChecked 的值！
```java
    private void updateDoNotAskCheckBox() {
        if (mShowDonNotAsk) {
            // 显示 CheckBox
            mDoNotAskCheckbox.setVisibility(View.VISIBLE);
            mDoNotAskCheckbox.setOnClickListener(this);
            mDoNotAskCheckbox.setChecked(mDoNotAskChecked); // 根据 mDoNotAskChecked 的值设置 CheckBox 的状态！
        } else {
            // 不显示 CheckBox
            mDoNotAskCheckbox.setVisibility(View.GONE);
            mDoNotAskCheckbox.setOnClickListener(null);
        }
    }
```

### 2.1.9 GrantPermissionsActivity.setResultAndFinish

处理结果：
```java
    private void setResultAndFinish() {
        setResultIfNeeded(RESULT_OK);
        finish(); // 调用 finish，结束当前 activity！
    }
```
setResultAndFinish 会调用 setResultIfNeeded 方法！
```java
    private void setResultIfNeeded(int resultCode) {
        if (!mResultSet) {
            mResultSet = true;
            logRequestedPermissionGroups();
            // 同样的，创建一个 intent，action 为 ACTION_REQUEST_PERMISSIONS！
            Intent result = new Intent(PackageManager.ACTION_REQUEST_PERMISSIONS);
            // 保存了请求的权限和请求结果！
            result.putExtra(PackageManager.EXTRA_REQUEST_PERMISSIONS_NAMES, mRequestedPermissions);
            result.putExtra(PackageManager.EXTRA_REQUEST_PERMISSIONS_RESULTS, mGrantResults);
            // 将结果返回给请求权限的 activity
            setResult(resultCode, result);
        }
    }
```
在调用了 finish 后，请求结果会返回到请求的 acitivty！


#### 2.1.9.1 RequestActivity.onRequestPermissionsResult

```java
    @Override
    public void onRequestPermissionsResult(int requestCode, String[] permissions, int[] grantResults) {
        super.onRequestPermissionsResult(requestCode, permissions, grantResults);
    }
```

最后，我们会在请求的 activity 中收到权限请求的结果！

# 3 AppPermissionGroup: Grant or Revoke - 授予/拒绝

下面我们来看看 AppPermissionGroup 是如何授予和拒绝权限的！

## 3.1 AppPermissionGroup.grantRuntimePermissions

fixedByTheUser 表示是否是被用户 fix 的:

对于 permissionPolicy 如果为 DevicePolicyManager.PERMISSION_POLICY_AUTO_GRANT 的情况，其值为 false，我们会自动授予权限！

正常情况下，是需要根据用户的选择来设置这个值的，如果用户本次授予了权限，那么 fixedByTheUser 依然为 false； 

```java
    public boolean grantRuntimePermissions(boolean fixedByTheUser) {
        return grantRuntimePermissions(fixedByTheUser, null);
    }
```
调用的二参数的同名方法！
```java
    public boolean grantRuntimePermissions(boolean fixedByTheUser, String[] filterPermissions) {
        final int uid = mPackageInfo.applicationInfo.uid;
        //【1】我们仅将权限切换到支持运行时权限的应用程序，否则，如果权限授予应用程序，则会切换与权限对应的应用程序op。
        for (Permission permission : mPermissions.values()) {
            if (filterPermissions != null // 处理权限过滤，这里不关注！
                    && !ArrayUtils.contains(filterPermissions, permission.getName())) {
                continue;
            }

            if (mAppSupportsRuntimePermissions) { //【1.1】对于运行时权限进入这里！
                if (permission.isSystemFixed()) { // 如果运行时权限被 system fix 了，结束处理！
                    return false;
                }

                //【1.1.1】确保运行时权限在授予之前，AppOp 为 allow 状态！
                // 判断该权限是否有对应的 AppOps，如果有且不处于 Allow 状态，设置其为 Allow！
                if (permission.hasAppOp() && !permission.isAppOpAllowed()) {
                    permission.setAppOpAllowed(true);
                    mAppOps.setUidMode(permission.getAppOp(), uid, AppOpsManager.MODE_ALLOWED);
                }

                if (!permission.isGranted()) {
                    permission.setGranted(true);
                    //【4.1】调用了 pms 的接口授予用户权限
                    mPackageManager.grantRuntimePermission(mPackageInfo.packageName,
                            permission.getName(), mUserHandle);
                }

                //【1.1.2】如果 fixedByTheUser 为 false，更新权限的 flags。
                if (!fixedByTheUser) {
                    // 如果该运行时权限处于 user fix 或者 user set 状态，那么我们会强制取消 user fix 状态！
                    // 这样用户就可以再次请求请求权限，因为其不处于 user fix 状态！
                    if (permission.isUserFixed() || permission.isUserSet()) {
                        permission.setUserFixed(false);
                        permission.setUserSet(true);
                        // 更新权限的 flags，
                        // 去掉 flags 中 FLAG_PERMISSION_USER_FIXED 和 FLAG_PERMISSION_USER_SET 位！
                        mPackageManager.updatePermissionFlags(permission.getName(),
                                mPackageInfo.packageName,
                                PackageManager.FLAG_PERMISSION_USER_FIXED
                                        | PackageManager.FLAG_PERMISSION_USER_SET,
                                0, mUserHandle);
                    }
                }
            } else {
                //【1.2】如果应用不支持运行时权限，这里不处理未授予情况，因为在 API 23 之前运行时权限是自动授予的！
                if (!permission.isGranted()) {
                    continue;
                }

                int killUid = -1;
                int mask = 0;

                // 如果权限没有对应的 app op，那么说明这是一个第三方应用，那么我们不对其做调整；
                // 否则，我们会调整其 app op！
                if (permission.hasAppOp()) {
                    // 如果权限不允许 app op，那就开启 app ops！
                    if (!permission.isAppOpAllowed()) {
                        permission.setAppOpAllowed(true);
                        // 开启 app op
                        mAppOps.setUidMode(permission.getAppOp(), uid, AppOpsManager.MODE_ALLOWED);

                        // 对于旧版本的不支持运行时权限的应用，当其运行时权限改变后，其不会尝试重试资源的访问，
                        // 所以我们需要杀掉其进程，使其重新加载！
                        killUid = uid;
                    }

                    // 如果应用从不支持运行时权限升级到了支持运行时权限的版本，升级后不能自动授予权限！
                    // 权限设置了 FLAG_PERMISSION_REVOKE_ON_UPGRADE 标志位！
                    if (permission.shouldRevokeOnUpgrade()) {
                        permission.setRevokeOnUpgrade(false);
                        mask |= PackageManager.FLAG_PERMISSION_REVOKE_ON_UPGRADE;
                    }
                }

                if (mask != 0) {
                    // 更新该权限的 flags，取消 FLAG_PERMISSION_REVOKE_ON_UPGRADE 标志位！
                    mPackageManager.updatePermissionFlags(permission.getName(),
                            mPackageInfo.packageName, mask, 0, mUserHandle);
                }

                if (killUid != -1) { // 杀掉进程！
                    mActivityManager.killUid(uid, KILL_REASON_APP_OP_CHANGE);
                }
            }
        }

        return true;
    }
```
对于允许权限的情况：

- 那么 fixedByTheUser 只能是 false，我们会取消 user set 和 user fix 标志位！

## 3.2 AppPermissionGroup.revokeRuntimePermissions

对于 permissionPolicy 为 DevicePolicyManager.PERMISSION_POLICY_AUTO_DENY 的情况，fixedByTheUser 为 false，同时会自动撤销权限！

正常情况下，是需要根据用户的选择来设置这个值的，如果用户本次拒绝了权限，但是 fixedByTheUser 可能有两种情况：

- 如果用户只是拒绝了权限，但是没有点击不再提醒，那么 fixedByTheUser 为 false；
- 如果用户不仅拒绝了权限，同时选择了不再提醒，那么 fixedByTheUser 为 true；

```java
    public boolean revokeRuntimePermissions(boolean fixedByTheUser) {
        return revokeRuntimePermissions(fixedByTheUser, null);
    }
```
调用了 revokeRuntimePermissions 的另一个方法！
```java
    public boolean revokeRuntimePermissions(boolean fixedByTheUser, String[] filterPermissions) {
        final int uid = mPackageInfo.applicationInfo.uid;
        //【1】我们仅将权限切换到支持运行时权限的应用程序，否则，如果权限授予应用程序，则会切换与权限对应的应用程序 op。
        for (Permission permission : mPermissions.values()) {
            if (filterPermissions != null
                    && !ArrayUtils.contains(filterPermissions, permission.getName())) {
                continue;
            }

            if (mAppSupportsRuntimePermissions) { //【1.1】对于运行时权限进入这里！
                if (permission.isSystemFixed()) { // 如果运行时权限被 system fix 了，结束处理！
                    return false;
                }

                if (permission.isGranted()) {
                    permission.setGranted(false);
                    //【4.2】调用了 PMS 的接口撤销权限
                    mPackageManager.revokeRuntimePermission(mPackageInfo.packageName,
                            permission.getName(), mUserHandle);
                }

                //【1.1.1】更新权限的标志位！
                if (fixedByTheUser) {
                    //【1.1.1.1】如果 fixedByTheUser 是 true，且权限是 user set 或者不是 user fix，
                    // 我们取消 set 设置其为 user fix！
                    if (permission.isUserSet() || !permission.isUserFixed()) {
                        permission.setUserSet(false);
                        permission.setUserFixed(true);
                        // 取消 user set 标志，设置 user fix 标志！
                        mPackageManager.updatePermissionFlags(permission.getName(),
                                mPackageInfo.packageName,
                                PackageManager.FLAG_PERMISSION_USER_SET
                                        | PackageManager.FLAG_PERMISSION_USER_FIXED,
                                PackageManager.FLAG_PERMISSION_USER_FIXED,
                                mUserHandle);
                    }
                } else {
                    //【1.1.1.2】如果 fixedByTheUser 是 false，且权限不是 user set，我们设置其为 user set！
                    if (!permission.isUserSet()) {
                        permission.setUserSet(true);
                        // 设置 user set 标志，表示用户已经做过选择！！
                        mPackageManager.updatePermissionFlags(permission.getName(),
                                mPackageInfo.packageName,
                                PackageManager.FLAG_PERMISSION_USER_SET,
                                PackageManager.FLAG_PERMISSION_USER_SET,
                                mUserHandle);
                    }
                }
            } else {
                //【1.2】如果应用不支持运行时权限，这里不处理未授予情况，因为在 API 23 之前运行时权限是自动授予的！
                if (!permission.isGranted()) {
                    continue;
                }

                int mask = 0;
                int flags = 0;
                int killUid = -1;

                // 如果权限没有对应的 app op，那么说明这是一个第三方应用，那么我们不对其做调整；
                // 否则，我们会调整其 app op！
                if (permission.hasAppOp()) {
                    if (permission.isAppOpAllowed()) { // 如果权限允许 app op，那就关闭 app ops！
                        permission.setAppOpAllowed(false);
                        mAppOps.setUidMode(permission.getAppOp(), uid, AppOpsManager.MODE_IGNORED);

                        // 对于旧版本的不支持运行时权限的引用，当其运行时权限改变后，其不会尝试重试资源的访问，
                        // 所以我们需要杀掉其进程，使其重新加载！
                        killUid = uid;
                    }

                    // 如果应用从不支持运行时权限升级到了支持运行时权限的版本，升级后不能自动授予权限！
                    // 权限未设置 FLAG_PERMISSION_REVOKE_ON_UPGRADE 标志位！
                    if (!permission.shouldRevokeOnUpgrade()) {
                        permission.setRevokeOnUpgrade(true);
                        mask |= PackageManager.FLAG_PERMISSION_REVOKE_ON_UPGRADE;
                        flags |= PackageManager.FLAG_PERMISSION_REVOKE_ON_UPGRADE;
                    }
                }

                if (mask != 0) {
                    // 更新该权限的 flags，增加 FLAG_PERMISSION_REVOKE_ON_UPGRADE 标志位！
                    mPackageManager.updatePermissionFlags(permission.getName(),
                            mPackageInfo.packageName, mask, flags, mUserHandle);
                }

                if (killUid != -1) { // 杀掉进程！
                    mActivityManager.killUid(uid, KILL_REASON_APP_OP_CHANGE);
                }
            }
        }

        return true;
    }
```
对于拒绝权限的情况：

- 如果用户没有选择不再提醒，那么 fixedByTheUser 为 false，我们会设置 user set 标志位！

- 如果用户同时还选择了不再提醒，那么 fixedByTheUser 为 true，那么会取消 user set 标志位，同时设置 user fix 标志位！


# 4 PackageManagerService: Grant or Revoke - 核心接口

当然，核心方法是调用 pms 的接口完成的，我们去看看这些方法！

## 4.1 PackageManagerS.grantRuntimePermission

授予运行时权限：

```java
    @Override
    public void grantRuntimePermission(String packageName, String name, final int userId) {
        if (!sUserManager.exists(userId)) { //【1】校验用户是否存在！
            Log.e(TAG, "No such user:" + userId);
            return;
        }
    
        mContext.enforceCallingOrSelfPermission( //【2】校验调用者是否有相应权限；
                android.Manifest.permission.GRANT_RUNTIME_PERMISSIONS,
                "grantRuntimePermission");

        enforceCrossUserPermission(Binder.getCallingUid(), userId,
                true /* requireFullPermission */, true /* checkShell */,
                "grantRuntimePermission");

        final int uid;
        final SettingBase sb;

        synchronized (mPackages) {
            final PackageParser.Package pkg = mPackages.get(packageName);
            if (pkg == null) { //【3】package 不存在，异常！
                throw new IllegalArgumentException("Unknown package: " + packageName);
            }

            final BasePermission bp = mSettings.mPermissions.get(name);
            if (bp == null) { //【4】权限不存在，异常！
                throw new IllegalArgumentException("Unknown permission: " + name);
            }

            enforceDeclaredAsUsedAndRuntimeOrDevelopmentPermission(pkg, bp);

            //【5】如果 targetSdkVersion 小于 M，那么是不支持运行时权限的，我们在安装的时候会自动授予运行时权限！
            if (Build.PERMISSIONS_REVIEW_REQUIRED
                    && pkg.applicationInfo.targetSdkVersion < Build.VERSION_CODES.M
                    && bp.isRuntime()) {
                return;
            }

            uid = UserHandle.getUid(userId, pkg.applicationInfo.uid);
            sb = (SettingBase) pkg.mExtras;
            if (sb == null) {//【6】升级包未安装过，异常！
                throw new IllegalArgumentException("Unknown package: " + packageName);
            }

            final PermissionsState permissionsState = sb.getPermissionsState();
            //【7】获得权限的 flags 标志位，如果该权限是被 system fix 的，我们不能操作这样的权限！
            final int flags = permissionsState.getPermissionFlags(name, userId);
            if ((flags & PackageManager.FLAG_PERMISSION_SYSTEM_FIXED) != 0) {
                throw new SecurityException("Cannot grant system fixed permission "
                        + name + " for package " + packageName);
            }

            //【8】如果该权限是开发者权限，特殊处理，作为安装时权限，立刻授予！
            // 授予成功，会触发 Settings.writeLPr 方法，该方法会更新多个文件
            // 包括 pacakges.xml，pacakges.list，runtime-permissions.xml 等文件！
            if (bp.isDevelopment()) {
                if (permissionsState.grantInstallPermission(bp) !=
                        PermissionsState.PERMISSION_OPERATION_FAILURE) {
                    scheduleWriteSettingsLocked();
                }
                return;
            }

            if (pkg.applicationInfo.targetSdkVersion < Build.VERSION_CODES.M) {
                Slog.w(TAG, "Cannot grant runtime permission to a legacy app");
                return;
            }

            //【9】授予权限，并处理授予结果！！
            final int result = permissionsState.grantRuntimePermission(bp, userId);
            switch (result) {
                case PermissionsState.PERMISSION_OPERATION_FAILURE: {
                    //【9.1】授予失败，可能应用已经被授予权限了！
                    return;
                }

                case PermissionsState.PERMISSION_OPERATION_SUCCESS_GIDS_CHANGED: {
                    //【9.2】授予成功，但是应用映射的 gids 变化了！
                    final int appId = UserHandle.getAppId(pkg.applicationInfo.uid);
                    mHandler.post(new Runnable() {
                        @Override
                        public void run() {
                            //【9.2.1】杀掉应用的进程，应用后续会重启！
                            killUid(appId, userId, KILL_APP_REASON_GIDS_CHANGED);
                        }
                    });
                }
                break;
            }

            mOnPermissionChangeListeners.onPermissionsChanged(uid);

            //【10】更新 runtime-permissions.xml 文件！
            mSettings.writeRuntimePermissionsForUserLPr(userId, false);
        }

        // Only need to do this if user is initialized. Otherwise it's a new user
        // and there are no processes running as the user yet and there's no need
        // to make an expensive call to remount processes for the changed permissions.
        if (READ_EXTERNAL_STORAGE.equals(name)
                || WRITE_EXTERNAL_STORAGE.equals(name)) {
            final long token = Binder.clearCallingIdentity();
            try {
                if (sUserManager.isInitialized(userId)) {
                    MountServiceInternal mountServiceInternal = LocalServices.getService(
                            MountServiceInternal.class);
                    mountServiceInternal.onExternalStoragePolicyChanged(uid, packageName);
                }
            } finally {
                Binder.restoreCallingIdentity(token);
            }
        }
    }
```
## 4.2 PackageManagerS.revokeRuntimePermission

拒绝运行时权限！

```java
    @Override
    public void revokeRuntimePermission(String packageName, String name, int userId) {
        if (!sUserManager.exists(userId)) { //【1】校验用户是否存在！
            Log.e(TAG, "No such user:" + userId);
            return;
        }

        mContext.enforceCallingOrSelfPermission( //【2】校验调用者是否有相应权限；
                android.Manifest.permission.REVOKE_RUNTIME_PERMISSIONS,
                "revokeRuntimePermission");

        enforceCrossUserPermission(Binder.getCallingUid(), userId,
                true /* requireFullPermission */, true /* checkShell */,
                "revokeRuntimePermission");

        final int appId;

        synchronized (mPackages) {
            final PackageParser.Package pkg = mPackages.get(packageName);
            if (pkg == null) { //【3】package 不存在，异常！
                throw new IllegalArgumentException("Unknown package: " + packageName);
            }

            final BasePermission bp = mSettings.mPermissions.get(name);
            if (bp == null) { //【4】权限不存在，异常！
                throw new IllegalArgumentException("Unknown permission: " + name);
            }

            enforceDeclaredAsUsedAndRuntimeOrDevelopmentPermission(pkg, bp);

            //【5】如果 targetSdkVersion 小于 M，那么是不支持运行时权限的，我们在安装的时候会自动授予运行时权限！
            if (Build.PERMISSIONS_REVIEW_REQUIRED
                    && pkg.applicationInfo.targetSdkVersion < Build.VERSION_CODES.M
                    && bp.isRuntime()) {
                return;
            }

            SettingBase sb = (SettingBase) pkg.mExtras;
            if (sb == null) { //【6】升级包未安装过，异常！
                throw new IllegalArgumentException("Unknown package: " + packageName);
            }

            final PermissionsState permissionsState = sb.getPermissionsState();
            //【7】获得权限的 flags 标志位，如果该权限是被 system fix 的，我们不能撤销这样的权限！
            final int flags = permissionsState.getPermissionFlags(name, userId);
            if ((flags & PackageManager.FLAG_PERMISSION_SYSTEM_FIXED) != 0) {
                throw new SecurityException("Cannot revoke system fixed permission "
                        + name + " for package " + packageName);
            }
            //【8】如果该权限是开发者权限，特殊处理，立刻授予！
            // 授予成功，会触发 Settings.writeLPr 方法，该方法会更新多个文件
            // 包括 pacakges.xml，pacakges.list，runtime-permissions.xml 等文件！
            if (bp.isDevelopment()) {
                if (permissionsState.revokeInstallPermission(bp) !=
                        PermissionsState.PERMISSION_OPERATION_FAILURE) {
                    scheduleWriteSettingsLocked();
                }
                return;
            }
            //【9】撤销运行时权限！
            if (permissionsState.revokeRuntimePermission(bp, userId) ==
                    PermissionsState.PERMISSION_OPERATION_FAILURE) {
                return;
            }

            mOnPermissionChangeListeners.onPermissionsChanged(pkg.applicationInfo.uid);

            //【10】更新 runtime-permissions.xml 文件！
            mSettings.writeRuntimePermissionsForUserLPr(userId, true);

            appId = UserHandle.getAppId(pkg.applicationInfo.uid);
        }

        killUid(appId, userId, KILL_APP_REASON_PERMISSIONS_REVOKED); // 杀掉应用的进程！
    }
```
最后的又进入了 PermissionsState 对象，这个我在 PMS 中有分析过，这里就不在多说了！！


# 5 Activity.shouldShowRequestPermissionRationale - 解释权限含义

为了防止用户因为不理解而拒绝了权限，我们可以在恰当的时间给用户解释为什么申请权限，这里要用到 shouldShowRequestPermissionRationale 方法！

```java
    public boolean shouldShowRequestPermissionRationale(@NonNull String permission) {
        return getPackageManager().shouldShowRequestPermissionRationale(permission);
    }
```

最终会调用 PackageManagerService 中的方法！


## 5.1 PackageManagerS.shouldShowRequestPermissionRationale

该方法的返回值决定了是否应该给用户解释权限！

```java
    @Override
    public boolean shouldShowRequestPermissionRationale(String permissionName,
            String packageName, int userId) {
        if (UserHandle.getCallingUserId() != userId) {
            mContext.enforceCallingPermission(
                    android.Manifest.permission.INTERACT_ACROSS_USERS_FULL,
                    "canShowRequestPermissionRationale for user " + userId);
        }

        final int uid = getPackageUid(packageName, MATCH_DEBUG_TRIAGED_MISSING, userId);
        if (UserHandle.getAppId(getCallingUid()) != UserHandle.getAppId(uid)) {
            return false;
        }
        //【1】如果已经拒绝权限，那么就返回 false！
        if (checkPermission(permissionName, packageName, userId)
                == PackageManager.PERMISSION_GRANTED) {
            return false;
        }

        final int flags;

        final long identity = Binder.clearCallingIdentity();
        try {
            // 获得权限的标志位！
            flags = getPermissionFlags(permissionName,
                    packageName, userId);
        } finally {
            Binder.restoreCallingIdentity(identity);
        }
        // fixedFlags 包含了所有的被 fix 的情况！
        final int fixedFlags  = PackageManager.FLAG_PERMISSION_SYSTEM_FIXED
                | PackageManager.FLAG_PERMISSION_POLICY_FIXED
                | PackageManager.FLAG_PERMISSION_USER_FIXED;

        //【2】如果该权限被 fix 了，那就返回 false；
        if ((flags & fixedFlags) != 0) {
            return false;
        }
        //【3】判断权限是否被 user set，如果被用户 set 了，返回 true！
        return (flags & PackageManager.FLAG_PERMISSION_USER_SET) != 0;
    }
```
policy fix 和 system fix 这些状态都是系统机制的处理！

我们可以看到：

- 如果用户授予了权限，那么该方法返回的是 false；
- 如果该权限被 system/policy/user fix 了，即用户拒绝了授权，又点击了不再提醒，那返回的是 false；
- 如果该权限被 user set 了，即用户拒绝了授权，但是没有点击不再提醒，那么会返回 true；



