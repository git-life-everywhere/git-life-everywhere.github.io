# PMS 第 8 篇 - 通过 PackageInstaller 分析 Install 过程
title: PMS 第 8 篇 - 通过 PackageInstaller 分析 Install 过程
date: 2018/07/23
categories:
- AndroidFramework源码分析
- PackageManager包管理
tags: PackageManager包管理
---

[toc]

基于 Android7.1.1 分析 PackageManagerService 的逻辑，Android 版本虽然会不断更替，但是代码结构和思想史

# 0 综述

前面总结了通过 pm install 的方式来安装一个 apk，但是这种方式用户是不经常使用的，用户使用的安装途径主要如下：

- 通过应用商店下载 apk，进行安装；
- 将 apk 文件移动到关键管理器，点击触发安装；

这里我们来看看第二种情况！

对于第二种中情况，当我们触发安装的时候，实际上是发送了一个 intent，这个 intent 中携带了 apk 的文件路径等参数，在 sample 给出了如下的启动安装的方式：

```java
    private OnClickListener mUnknownSourceListener = new OnClickListener() {
        public void onClick(View v) {
            Intent intent = new Intent(Intent.ACTION_INSTALL_PACKAGE);
            intent.setData(Uri.fromFile(prepareApk("HelloActivity.apk")));
            startActivity(intent);
        }
    };

    private OnClickListener mMySourceListener = new OnClickListener() {
        public void onClick(View v) {
            Intent intent = new Intent(Intent.ACTION_INSTALL_PACKAGE);
            intent.setData(Uri.fromFile(prepareApk("HelloActivity.apk")));
            intent.putExtra(Intent.EXTRA_NOT_UNKNOWN_SOURCE, true);
            intent.putExtra(Intent.EXTRA_RETURN_RESULT, true);
            intent.putExtra(Intent.EXTRA_INSTALLER_PACKAGE_NAME,
                    getApplicationInfo().packageName);
            startActivityForResult(intent, REQUEST_INSTALL);
        }
    };

    private OnClickListener mReplaceListener = new OnClickListener() {
        public void onClick(View v) {
            Intent intent = new Intent(Intent.ACTION_INSTALL_PACKAGE);
            intent.setData(Uri.fromFile(prepareApk("HelloActivity.apk")));
            intent.putExtra(Intent.EXTRA_NOT_UNKNOWN_SOURCE, true);
            intent.putExtra(Intent.EXTRA_RETURN_RESULT, true);
            intent.putExtra(Intent.EXTRA_ALLOW_REPLACE, true);
            intent.putExtra(Intent.EXTRA_INSTALLER_PACKAGE_NAME,
                    getApplicationInfo().packageName);
            startActivityForResult(intent, REQUEST_INSTALL);
        }
    };
```

那谁来接收这个 intent 呢？显而易见，PacakgeInstaller，我们去其说明书中看看：

```xml
        <activity android:name=".PackageInstallerActivity"
                android:configChanges="orientation|keyboardHidden|screenSize"
                android:excludeFromRecents="true">
            <intent-filter>
                <action android:name="android.intent.action.VIEW" />
                <action android:name="android.intent.action.INSTALL_PACKAGE" />
                <category android:name="android.intent.category.DEFAULT" />
                <data android:scheme="file" />
                <data android:scheme="content" />
                <data android:mimeType="application/vnd.android.package-archive" />
            </intent-filter>
            <intent-filter>
                <action android:name="android.intent.action.INSTALL_PACKAGE" />
                <category android:name="android.intent.category.DEFAULT" />
                <data android:scheme="file" />
                <data android:scheme="package" />
                <data android:scheme="content" />
            </intent-filter>
            <intent-filter>
                <action android:name="android.content.pm.action.CONFIRM_PERMISSIONS" />
                <category android:name="android.intent.category.DEFAULT" />
            </intent-filter>
        </activity>
```
我们看到 PackageInstallerActivity 接收了 android.intent.action.INSTALL_PACKAGE action！

下面我们来分析下整个流程：

# 1 PackageInstallerActivity.onCreate

这里我们进入 PackageInstallerActivity，看看其做了什么处理：

```java
public class PackageInstallerActivity extends Activity implements OnCancelListener, OnClickListener {
    ... ... ...

    @Override
    protected void onCreate(Bundle icicle) {
        super.onCreate(icicle);

        mPm = getPackageManager();
        mInstaller = mPm.getPackageInstaller();
        mUserManager = (UserManager) getSystemService(Context.USER_SERVICE);

        //【1】获得安装传入的  Intent！
        final Intent intent = getIntent();
        mOriginatingUid = getOriginatingUid(intent);

        final Uri packageUri;
        
        if (PackageInstaller.ACTION_CONFIRM_PERMISSIONS.equals(intent.getAction())) {
            //【2】这个广播我们在 pm install 中有见过，当 install 需要确认权限信息的时候，会发送 intent
            // 其实是最终发送了这个 action 给 PackageInstaller 中了，同时会把 sessionId 发过来！
            final int sessionId = intent.getIntExtra(PackageInstaller.EXTRA_SESSION_ID, -1);
            final PackageInstaller.SessionInfo info = mInstaller.getSessionInfo(sessionId);
            if (info == null || !info.sealed || info.resolvedBaseCodePath == null) {
                Log.w(TAG, "Session " + mSessionId + " in funky state; ignoring");
                finish();
                return;
            }

            mSessionId = sessionId;
            packageUri = Uri.fromFile(new File(info.resolvedBaseCodePath));
            mOriginatingURI = null;
            mReferrerURI = null;
        } else {
            //【3】对于 install package 进入这里，会通过 getData 获得 packageUri！
            mSessionId = -1;
            packageUri = intent.getData();
            mOriginatingURI = intent.getParcelableExtra(Intent.EXTRA_ORIGINATING_URI);
            mReferrerURI = intent.getParcelableExtra(Intent.EXTRA_REFERRER);
        }

        //【4】如果 packageUri 为 null，那么这里会返回结果给启动安装的界面：INVALID_URI！
        if (packageUri == null) {
            Log.w(TAG, "Unspecified source");
            setPmResult(PackageManager.INSTALL_FAILED_INVALID_URI);
            finish();
            return;
        }

        if (DeviceUtils.isWear(this)) { // 可穿戴设备的逻辑，不关注！
            showDialogInner(DLG_NOT_SUPPORTED_ON_WEAR);
            return;
        }

        //【5】初始化界面！
        setContentView(R.layout.install_start);
        mInstallConfirm = findViewById(R.id.install_confirm_panel);
        mInstallConfirm.setVisibility(View.INVISIBLE);
        mOk = (Button)findViewById(R.id.ok_button);
        mCancel = (Button)findViewById(R.id.cancel_button);
        mOk.setOnClickListener(this);
        mCancel.setOnClickListener(this);

        //【*1.1】解析传入的 packageUri！
        boolean wasSetUp = processPackageUri(packageUri);
        if (!wasSetUp) {
            return;
        }
        //【*1.2】解析传入的 packageUri！
        checkIfAllowedAndInitiateInstall(false);
    }
    ... ... ...
}
```
我们看到 PackageInstallerActivity 方法实现了 OnCancelListener, OnClickListener，用于响应界面事件。。。

## 1.1 processPackageUri

解析 Uri 被为要安装的 apk 设置合适的 installer，如果返回 true，表示设置成功了！

```java
    private boolean processPackageUri(final Uri packageUri) {
        mPackageURI = packageUri;
        //【1】获得 scheme 属性，对应资源使用的协议！
        final String scheme = packageUri.getScheme();
        final PackageUtil.AppSnippet as;

        switch (scheme) {
            case SCHEME_PACKAGE: { //【1.1】package 类型!！
                try {
                    mPkgInfo = mPm.getPackageInfo(packageUri.getSchemeSpecificPart(),
                            PackageManager.GET_PERMISSIONS
                                    | PackageManager.GET_UNINSTALLED_PACKAGES);
                } catch (NameNotFoundException e) {
                }
                if (mPkgInfo == null) {
                    Log.w(TAG, "Requested package " + packageUri.getScheme()
                            + " not available. Discontinuing installation");
                    //【*6/3】应用包异常，提示用户！
                    showDialogInner(DLG_PACKAGE_ERROR);
                    setPmResult(PackageManager.INSTALL_FAILED_INVALID_APK);
                    return false;
                }
                //【*1.1.2.1】创建 AppSnippet 实例！
                as = new PackageUtil.AppSnippet(mPm.getApplicationLabel(mPkgInfo.applicationInfo),
                        mPm.getApplicationIcon(mPkgInfo.applicationInfo));
            } break;

            case SCHEME_FILE: { //【1.2】file 类型
                File sourceFile = new File(packageUri.getPath());
                //【*1.1.1】对 path 指定的 apk 进行解析，其实是创建了一个 PackageParser 对象，然后扫描 apk
                // 将结果封装为 PackageParser.Package 并返回！
                PackageParser.Package parsed = PackageUtil.getPackageInfo(sourceFile);

                if (parsed == null) { // 发生了 error！
                    Log.w(TAG, "Parse error when parsing manifest. Discontinuing installation");
                    //【*6/3】应用包异常，提示用户！
                    showDialogInner(DLG_PACKAGE_ERROR);
                    setPmResult(PackageManager.INSTALL_FAILED_INVALID_APK);
                    return false;
                }
                //【*1.1.2】将解析到的 Package 转为 PackageInfo 对象！
                // 除了获取到了基本信息，这里还额外指定了获取该应用的定义的权限和申请的权限信息！
                mPkgInfo = PackageParser.generatePackageInfo(parsed, null,
                        PackageManager.GET_PERMISSIONS, 0, 0, null,
                        new PackageUserState());
                //【*1.1.3】获得 apk 的 AppSnippet 实例！
                as = PackageUtil.getAppSnippet(this, mPkgInfo.applicationInfo, sourceFile);
            } break;

            case SCHEME_CONTENT: { //【1.3】content 类型
                mStagingAsynTask = new StagingAsyncTask();
                mStagingAsynTask.execute(packageUri);
                return false;
            }

            default: {
                //【1.4】不支持的资源协议，默认会进入这里！
                Log.w(TAG, "Unsupported scheme " + scheme);
                setPmResult(PackageManager.INSTALL_FAILED_INVALID_URI);
                clearCachedApkIfNeededAndFinish();
                return false;
            }
        }
        //【*1.1.4】继续处理 AppSnippet 实例！
        PackageUtil.initSnippetForNewApp(this, as, R.id.app_snippet);

        return true;
    }
```
对于安装文件管理器的 apk 的方式，其 scheme 值为 file，所以这里

对于 package 的解析流程，这里就不再多说了；

而 generatePackageInfo 方法的逻辑也很简单，flags 可以通过设置指定的位来获取特定的数据！


### 1.1.1 PackageUtil.getPackageInfo

```java
    public static PackageParser.Package getPackageInfo(File sourceFile) {
        final PackageParser parser = new PackageParser();
        try {
            return parser.parsePackage(sourceFile, 0);
        } catch (PackageParserException e) {
            return null;
        }
    }
```

### 1.1.2 PackageParser.generatePackageInfo

将解析到的 Package 封装为指定的 PackageInfo 对象，除了基本的信息之外，还有通过 flags 指定的额外信息！

```java
    public static PackageInfo generatePackageInfo(PackageParser.Package p,
            int gids[], int flags, long firstInstallTime, long lastUpdateTime,
            Set<String> grantedPermissions, PackageUserState state, int userId) {
        if (!checkUseInstalledOrHidden(flags, state) || !p.isMatch(flags)) {
            return null;
        }
        //【1】创建一个 PackageInfo 实例！
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

        ... ... ... ... // 这里省略了其他的 flags 处理！
        
        //【2】获得应用的权限信息 GET_PERMISSIONS！
        if ((flags&PackageManager.GET_PERMISSIONS) != 0) {
            //【2.1】获得其定义的权限；
            int N = p.permissions.size();
            if (N > 0) {
                pi.permissions = new PermissionInfo[N];
                for (int i=0; i<N; i++) {
                    pi.permissions[i] = generatePermissionInfo(p.permissions.get(i), flags);
                }
            }
            //【2.2】获得其申请的权限；
            N = p.requestedPermissions.size();
            if (N > 0) {
                pi.requestedPermissions = new String[N];
                pi.requestedPermissionsFlags = new int[N];
                for (int i=0; i<N; i++) {
                    final String perm = p.requestedPermissions.get(i);
                    pi.requestedPermissions[i] = perm;
                    //【2.2.1】更新申请的权限的 flags，设置 REQUESTED_PERMISSION_REQUIRED 标志位！
                    // 如果该权限已经被授予，那么还会增加 REQUESTED_PERMISSION_GRANTED 标志位！
                    pi.requestedPermissionsFlags[i] |= PackageInfo.REQUESTED_PERMISSION_REQUIRED;
                    if (grantedPermissions != null && grantedPermissions.contains(perm)) {
                        pi.requestedPermissionsFlags[i] |= PackageInfo.REQUESTED_PERMISSION_GRANTED;
                    }
                }
            }
        }
        ... ... ...
        return pi;
    }
```
这里涉及到了几个标志位：

```java
    // 表示该权限是应用程序运行必备的，用户不能 disable！
    public static final int REQUESTED_PERMISSION_REQUIRED = 1<<0;
    // 权限是否已经授予！
    public static final int REQUESTED_PERMISSION_GRANTED = 1<<1;    
```

这两个标签都是在 generatePackageInfo 方法中，配合 GET_PERMISSIONS 方法使用！

### 1.1.3 PackageUtil.getAppSnippet

这个方法用于加载应用的 label 和 icon：

```java
    public static AppSnippet getAppSnippet(
            Activity pContext, ApplicationInfo appInfo, File sourceFile) {
        final String archiveFilePath = sourceFile.getAbsolutePath();
        Resources pRes = pContext.getResources();
        AssetManager assmgr = new AssetManager();
        //【1】设置资源加载路径！
        assmgr.addAssetPath(archiveFilePath);
        Resources res = new Resources(assmgr, pRes.getDisplayMetrics(), pRes.getConfiguration());
        CharSequence label = null;
        
        //【2】尝试先从资源文件中加载 label！
        if (appInfo.labelRes != 0) {
            try {
                label = res.getText(appInfo.labelRes);
            } catch (Resources.NotFoundException e) {
            }
        }
        //【3】如果没有显式定义的话，那么就会使用包名；
        if (label == null) {
            label = (appInfo.nonLocalizedLabel != null) ?
                    appInfo.nonLocalizedLabel : appInfo.packageName;
        }
        Drawable icon = null;
        //【4】尝试从资源文件获取图标，如果没有定义，那么我们会从系统中获取默认图标！
        if (appInfo.icon != 0) {
            try {
                icon = res.getDrawable(appInfo.icon);
            } catch (Resources.NotFoundException e) {
            }
        }
        if (icon == null) {
            icon = pContext.getPackageManager().getDefaultActivityIcon();
        }

        //【*1.1.2.1】创建了一个 AppSnippet 对象！
        return new PackageUtil.AppSnippet(label, icon);
    }
```
#### 1.1.2.1 new AppSnippet

AppSnippet 用于保存 apk 的 label 和 icon：

```java
    static public class AppSnippet {
        CharSequence label;
        Drawable icon;
        public AppSnippet(CharSequence label, Drawable icon) {
            this.label = label;
            this.icon = icon;
        }
    }
```

### 1.1.4 PackageUtil.initSnippetForNewApp

根据返回的 AppSnippet，初始化界面，显示应用的名称和 label：
```java
    public static View initSnippetForNewApp(Activity pContext, AppSnippet as,
            int snippetId) {
        View appSnippet = pContext.findViewById(snippetId);
        ((ImageView)appSnippet.findViewById(R.id.app_icon)).setImageDrawable(as.icon);
        ((TextView)appSnippet.findViewById(R.id.app_name)).setText(as.label);
        return appSnippet;
    }
```


## 1.2 checkIfAllowedAndInitiateInstall

用于判断是否允许安装，如果允许安装，那就进行初始化；否则会弹出提示框，参数 ignoreUnknownSourcesSettings 表示是否忽视未知来源安装，这里我们传入的是 false；

```java
    private void checkIfAllowedAndInitiateInstall(boolean ignoreUnknownSourcesSettings) {
        //【*1.2.1】判断本次安装是否是来自未知来源！
        final boolean requestFromUnknownSource = isInstallRequestFromUnknownSource(getIntent());
        if (!requestFromUnknownSource) {
            //【*1.2.6】如果是特权应用发送的安装，那么就初始化安装！
            initiateInstall();
        }
        //【1】判断当前所在 user 是否是在另外一个 user 的 profile 中！
        final boolean isManagedProfile = mUserManager.isManagedProfile();
        //【*1.2.2】判断当前用户是否设置了不允许位置来源的用户限制！
        if (isUnknownSourcesDisallowed()) {
            if ((mUserManager.getUserRestrictionSource(UserManager.DISALLOW_INSTALL_UNKNOWN_SOURCES,
                    Process.myUserHandle()) & UserManager.RESTRICTION_SOURCE_SYSTEM) != 0) {
                if (ignoreUnknownSourcesSettings) {
                    //【*1.2.6】如果忽视未知来源，初始化安装！
                    initiateInstall();
                } else {
                    //【*6/1】显示 DLG_UNKNOWN_SOURCES 弹窗！
                    showDialogInner(DLG_UNKNOWN_SOURCES);
                }
            } else {
                startActivity(new Intent(Settings.ACTION_SHOW_ADMIN_SUPPORT_DETAILS));
                clearCachedApkIfNeededAndFinish();
            }
            
        } else if (!isUnknownSourcesEnabled() && isManagedProfile) {
            //【*1.2.3】如果没有设置用户限制，但是用户关闭了设置中的可未知来源安装
            // 同时当前用户是在另外用户的 profile 中！
            // 显示 DLG_ADMIN_RESTRICTS_UNKNOWN_SOURCES 弹窗！
            //【*6/2】提示用户！
            showDialogInner(DLG_ADMIN_RESTRICTS_UNKNOWN_SOURCES);

        } else if (!isUnknownSourcesEnabled()) {
            //【*1.2.3】如果用户关闭了设置中的可未知来源安装，并且我们忽视位置来源
            // 那么就立刻初始化！
            if (ignoreUnknownSourcesSettings) {
                //【*1.2.6】初始化安装！
                initiateInstall();
            } else {
                //【*6/1】显示 DLG_UNKNOWN_SOURCES 弹窗！
                showDialogInner(DLG_UNKNOWN_SOURCES);
            }
        } else {
            //【*1.2.6】如果不是来自未知来源的安装，那么就初始化安装！
            initiateInstall();
        }
    }
```

### 1.2.1 isInstallRequestFromUnknownSource

判断本次安装是否来自未知来源：

```java
    private boolean isInstallRequestFromUnknownSource(Intent intent) {
        String callerPackage = getCallingPackage();
        //【1】判断 intent 是否设置 EXTRA_NOT_UNKNOWN_SOURCE 为 true！
        if (callerPackage != null && intent.getBooleanExtra(
                Intent.EXTRA_NOT_UNKNOWN_SOURCE, false)) {
            try {
                mSourceInfo = mPm.getApplicationInfo(callerPackage, 0);
                if (mSourceInfo != null) {
                    if ((mSourceInfo.privateFlags & ApplicationInfo.PRIVATE_FLAG_PRIVILEGED)
                            != 0) {
                        //【1.1】如果设置了为 true，只有特权应用发起安装时才不会视为未知来源；
                        return false;
                    }
                }
            } catch (NameNotFoundException e) {
            }
        }
        //【2】其他情况，无论你是否设置，都会视为未知来源！
        return true;
    }
```
### 1.2.2 isUnknownSourcesDisallowed

判断下当前设备用户是否限制来自未知来源的安装，也就是是否设置了 DISALLOW_INSTALL_UNKNOWN_SOURCES 用户限制：

```java
    private boolean isUnknownSourcesDisallowed() {
        return mUserManager.hasUserRestriction(UserManager.DISALLOW_INSTALL_UNKNOWN_SOURCES);
    }
```

### 1.2.3 isUnknownSourcesEnabled

判断用户是否在设置中打开了位置来源限制：

```java
    private boolean isUnknownSourcesEnabled() {
        return Settings.Secure.getInt(getContentResolver(),
                Settings.Secure.INSTALL_NON_MARKET_APPS, 0) > 0;
    }
```


### 1.2.6 initiateInstall

初始化安装：

```java
    private void initiateInstall() {
        //【1】获得要安装的应用包名！
        String pkgName = mPkgInfo.packageName;
        
        //【2】判断下系统中是否存在该 pkg，但是被改为了其他名称，如果有，这里要调整下！
        String[] oldName = mPm.canonicalToCurrentPackageNames(new String[] { pkgName });
        if (oldName != null && oldName.length > 0 && oldName[0] != null) {
            pkgName = oldName[0];
            mPkgInfo.packageName = pkgName;
            mPkgInfo.applicationInfo.packageName = pkgName;
        }

        //【3】判断该 pkg 是否已经安装过，如果是，弹出 replace 的弹窗；
        try {
            //【3.1】这里通过 GET_UNINSTALLED_PACKAGES 标志为获得该应用的安装信息
            // 如果该应用在系统中只有 data 数据，我们也会将其视为已经安装！
            mAppInfo = mPm.getApplicationInfo(pkgName,
                    PackageManager.GET_UNINSTALLED_PACKAGES);
            //【3.2】判断是否有 FLAG_INSTALLED 标志为，如果没有，那就是没有安装过！
            if ((mAppInfo.flags&ApplicationInfo.FLAG_INSTALLED) == 0) {
                mAppInfo = null;
            }
        } catch (NameNotFoundException e) {
            mAppInfo = null;
        }
        //【*2】继续处理：
        startInstallConfirm();
    }
```


# 2 PackageInstallerActivity.startInstallConfirm

```java
    private void startInstallConfirm() {
        //【1】初始化界面；
        ((TextView) findViewById(R.id.install_confirm_question))
                .setText(R.string.install_confirm_question);
        findViewById(R.id.spacer).setVisibility(View.GONE);
        TabHost tabHost = (TabHost)findViewById(android.R.id.tabhost);
        tabHost.setup();
        tabHost.setVisibility(View.VISIBLE);
        ViewPager viewPager = (ViewPager)findViewById(R.id.pager);
        TabsAdapter adapter = new TabsAdapter(this, tabHost, viewPager);
        
        //【2】判断应用是否支持运行时权限！
        boolean supportsRuntimePermissions = mPkgInfo.applicationInfo.targetSdkVersion
                >= Build.VERSION_CODES.M;
        boolean permVisible = false;
        mScrollView = null;
        mOkCanInstall = false;
        int msg = 0;

        //【*2.1】创建了一个 AppSecurityPermissions 实例，封装 apk 的权限
        AppSecurityPermissions perms = new AppSecurityPermissions(this, mPkgInfo);
        //【*2.2】获得应用的申请的所有权限数量！
        final int N = perms.getPermissionCount(AppSecurityPermissions.WHICH_ALL);

        //【3】初始化界面，这里会用 mScrollView 来显示所有的权限信息，mAppInfo 不为 null，说明应用
        // 之前安装过！
        if (mAppInfo != null) {
            msg = (mAppInfo.flags & ApplicationInfo.FLAG_SYSTEM) != 0
                    ? R.string.install_confirm_question_update_system
                    : R.string.install_confirm_question_update;
            mScrollView = new CaffeinatedScrollView(this);
            mScrollView.setFillViewport(true);
            boolean newPermissionsFound = false;

            //【3.1】不支持运行时权限，这里会尝试先获取该应用的一些新的权限；
            if (!supportsRuntimePermissions) {
                 //【*2.2】获得应用的申请的所有被视为新的权限的数量！
                newPermissionsFound =
                        (perms.getPermissionCount(AppSecurityPermissions.WHICH_NEW) > 0);
                if (newPermissionsFound) {
                    permVisible = true;
                    mScrollView.addView(perms.getPermissionsView(
                            AppSecurityPermissions.WHICH_NEW));
                }
            }
            //【3.2】不支持运行时权限，同时没有新的申请的权限；
            if (!supportsRuntimePermissions && !newPermissionsFound) {
                LayoutInflater inflater = (LayoutInflater)getSystemService(
                        Context.LAYOUT_INFLATER_SERVICE);
                TextView label = (TextView)inflater.inflate(R.layout.label, null);
                label.setText(R.string.no_new_perms);
                mScrollView.addView(label);
            }
            //【3.3】这里会通过 tabHost 新增一个页面，用于显示申请的新增权限！
            adapter.addTab(tabHost.newTabSpec(TAB_ID_NEW).setIndicator(
                    getText(R.string.newPerms)), mScrollView);
        } else  {
            findViewById(R.id.tabscontainer).setVisibility(View.GONE);
            findViewById(R.id.spacer).setVisibility(View.VISIBLE);
        }
        
        //【3】如果不支持运行时权限，那么这里会处理申请的所有的权限！
        if (!supportsRuntimePermissions && N > 0) {
            permVisible = true;
            LayoutInflater inflater = (LayoutInflater)getSystemService(
                    Context.LAYOUT_INFLATER_SERVICE);
            View root = inflater.inflate(R.layout.permissions_list, null);
            if (mScrollView == null) {
                mScrollView = (CaffeinatedScrollView)root.findViewById(R.id.scrollview);
            }
            ((ViewGroup)root.findViewById(R.id.permission_list)).addView(
                        perms.getPermissionsView(AppSecurityPermissions.WHICH_ALL));
            //【4】这里会通过 tabHost 新增一个页面，用于显示申请的所有的权限！
            adapter.addTab(tabHost.newTabSpec(TAB_ID_ALL).setIndicator(
                    getText(R.string.allPerms)), root);
        }
        
        //【4】如果 permVisible 说明该应用没有要显示给用户的权限；
        if (!permVisible) {
            if (mAppInfo != null) {
                //【4.1】处理应用覆盖安装的情况，设置 msg 为没有 update_system_no_perms 或 update_no_perms！
                // 这是资源 id，会在界面显示！
                msg = (mAppInfo.flags & ApplicationInfo.FLAG_SYSTEM) != 0
                        ? R.string.install_confirm_question_update_system_no_perms
                        : R.string.install_confirm_question_update_no_perms;

                findViewById(R.id.spacer).setVisibility(View.VISIBLE);
            } else {
                //【4.1】处理应用全新安装的情况，设置 msg 为 no_perms，同上！
                msg = R.string.install_confirm_question_no_perms;
            }
            tabHost.setVisibility(View.INVISIBLE);
            mScrollView = null;
        }
        if (msg != 0) {
            ((TextView)findViewById(R.id.install_confirm_question)).setText(msg);
        }
        mInstallConfirm.setVisibility(View.VISIBLE);
        
        //【6】设置安装 button 可以点击！
        mOk.setEnabled(true);
        if (mScrollView == null) {
            //【6.1】如果不需要用 mScrollView 显示内容，那就直接显示安装
            mOk.setText(R.string.install);
            mOkCanInstall = true;
        } else {
            //【6.2】如果要显示权限信息，那么需要用户滚动界面，看完所有的权限信息
            // 安装 button 才可用！
            mScrollView.setFullScrollAction(new Runnable() {
                @Override
                public void run() {
                    mOk.setText(R.string.install);
                    mOkCanInstall = true;
                }
            });
        }
    }
```
不多说了，继续看！

## 2.1 new AppSecurityPermissions

AppSecurityPermissions 有多个构造器，这里调用了 AppSecurityPermissions 的双参构造函数，同时 AppSecurityPermissions 内部也有很多的成员变量，我们来分析下：

这里的 PackageInfo info 表示的是本次要安装的应用程序的 PackageInfo 实例！

```java
public class AppSecurityPermissions {
    ... ... ...
    public AppSecurityPermissions(Context context, PackageInfo info) {
        this(context);
        //【1】创建了一个 Set 集合用于收集权限
        Set<MyPermissionInfo> permSet = new HashSet<MyPermissionInfo>();
        if(info == null) {
            return;
        }
        mPackageName = info.packageName;

        //【2】用于保存已经安装的相同包名应用的信息！
        PackageInfo installedPkgInfo = null;

        //【3】如果应用有申请权限，那么就查询已经安装的同名应用（如果有）
        // 对应的权限信息，收集到 installedPkgInfo 中！
        if (info.requestedPermissions != null) {
            try {
                installedPkgInfo = mPm.getPackageInfo(info.packageName,
                        PackageManager.GET_PERMISSIONS);
            } catch (NameNotFoundException e) {
            }
            //【*2.1.1】这里会比较本次安装的应用和已经安装的旧应用（如果有）的权限，做相应处理
            // 最后将权限保存到 permSet 集合中！
            extractPerms(info, permSet, installedPkgInfo);
        }

        //【4】如果其是共享 uid 的，那么就要获得该 uid 的所有权限！
        if (info.sharedUserId != null) {
            int sharedUid;
            try {
                sharedUid = mPm.getUidForSharedUser(info.sharedUserId);
                //【*2.1.2】获得该 shared uid 的权限信息！
                getAllUsedPermissions(sharedUid, permSet);
            } catch (NameNotFoundException e) {
                Log.w(TAG, "Couldn't retrieve shared user id for: " + info.packageName);
            }
        }

        //【5】将搜集到的权限 set 集合加到 mPermsList 中！
        mPermsList.addAll(permSet);
        //【*2.1.3】进一步设置权限！
        setPermissions(mPermsList);
    }
    ... ... ...
}
```

AppSecurityPermissions 中有如下重要变量：

```java
    //【1】所属的所有的权限组，组名和 MyPermissionGroupInfo 的映射关系；
    private final Map<String, MyPermissionGroupInfo> mPermGroups
            = new HashMap<String, MyPermissionGroupInfo>();
    //【2】所属的所有的权限组；
    private final List<MyPermissionGroupInfo> mPermGroupsList
            = new ArrayList<MyPermissionGroupInfo>();
    //【3】用于给权限组排序；
    private final PermissionGroupInfoComparator mPermGroupComparator =
            new PermissionGroupInfoComparator();
    //【4】用于给权限排序；
    private final PermissionInfoComparator mPermComparator = new PermissionInfoComparator();
    //【5】应用请求的权限信息！
    private final List<MyPermissionInfo> mPermsList = new ArrayList<MyPermissionInfo>();
    private final CharSequence mNewPermPrefix;
    private String mPackageName;
```
我们后面分析的时候会遇到！


### 2.1.1 AppSecurityPermissions.extractPerms

这里会比较本次安装的应用和已经安装的旧应用（如果有）的权限，做相应处理：

```java
    private void extractPerms(PackageInfo info, Set<MyPermissionInfo> permSet,
            PackageInfo installedPkgInfo) {
        String[] strList = info.requestedPermissions;
        int[] flagsList = info.requestedPermissionsFlags;
        if ((strList == null) || (strList.length == 0)) {
            return;
        }
        //【1】遍历处理权限！
        for (int i=0; i<strList.length; i++) {
            String permName = strList[i];
            try {
                //【1.1】获得权限对应的 PermissionInfo 对象 tmpPermInfo，如果为 null 跳过！
                PermissionInfo tmpPermInfo = mPm.getPermissionInfo(permName, 0);
                if (tmpPermInfo == null) {
                    continue;
                }
                //【1.2】判断新安装的应用的权限在已经安装的应用（如果有）中是否存在，如果有 existingIndex 不为 -1；
                // existingIndex 为其在已经安装的应用的权限列表中的下标！
                int existingIndex = -1;
                if (installedPkgInfo != null
                        && installedPkgInfo.requestedPermissions != null) {
                    for (int j=0; j<installedPkgInfo.requestedPermissions.length; j++) {
                        if (permName.equals(installedPkgInfo.requestedPermissions[j])) {
                            existingIndex = j;
                            break;
                        }
                    }
                }
                //【1.3】判断下该权限在已经安装的应用中的权限标志位；
                final int existingFlags = existingIndex >= 0 ?
                        installedPkgInfo.requestedPermissionsFlags[existingIndex] : 0;

                //【*2.1.1.1】判断该权限是否需要在界面给用户显示出来，不需要的话那就跳过该权限；
                if (!isDisplayablePermission(tmpPermInfo, flagsList[i], existingFlags)) {
                    continue;
                }
                
                //【1.4】处理权限组名，默认为包名！
                final String origGroupName = tmpPermInfo.group;
                String groupName = origGroupName;
                if (groupName == null) {
                    groupName = tmpPermInfo.packageName;
                    tmpPermInfo.group = groupName;
                }
                
                //【1.5】获得/创建权限所属的权限组！
                MyPermissionGroupInfo group = mPermGroups.get(groupName);
                if (group == null) {
                    PermissionGroupInfo grp = null;
                    if (origGroupName != null) {
                        grp = mPm.getPermissionGroupInfo(origGroupName, 0);
                    }
                    if (grp != null) {
                        //【1.5.1】如果权限指定了所属的组，那么我们就查询组信息
                        // 然后创建 MyPermissionGroupInfo 实例，祖名为定义权限的包名！！
                        group = new MyPermissionGroupInfo(grp);
                    } else {
                        //【1.5.2】如果权限没有指定所属的组，或者组无法查到，那么我们就
                        // 新建 MyPermissionGroupInfo 实例.，默认权限组名为权限定义的包名！
                        tmpPermInfo.group = tmpPermInfo.packageName;
                        group = mPermGroups.get(tmpPermInfo.group);
                        if (group == null) {
                            group = new MyPermissionGroupInfo(tmpPermInfo);
                        }
                        group = new MyPermissionGroupInfo(tmpPermInfo);
                    }
                    //【1.5.3】将该组加入到内部集合 mPermGroups 中！
                    mPermGroups.put(tmpPermInfo.group, group);
                }
                
                //【1.6】判断该权限是否是一个新的权限，如果应用已经被安装了，同时并没有被授予该权限！
                final boolean newPerm = installedPkgInfo != null
                        && (existingFlags&PackageInfo.REQUESTED_PERMISSION_GRANTED) == 0;
                
                //【×2.1.1.3】创建一个 MyPermissionInfo 对象，封装该权限信息；
                MyPermissionInfo myPerm = new MyPermissionInfo(tmpPermInfo);
                myPerm.mNewReqFlags = flagsList[i];
                myPerm.mExistingReqFlags = existingFlags;

                // This is a new permission if the app is already installed and
                // doesn't currently hold this permission.
                myPerm.mNew = newPerm;
                
                //【1.7】将该权限加入到内部集合 permSet 中；
                permSet.add(myPerm);
            } catch (NameNotFoundException e) {
                Log.i(TAG, "Ignoring unknown permission:"+permName);
            }
        }
    }
```
不多说了。。

#### 2.1.1.1 AppSecurityPermissions.isDisplayablePermission

isDisplayablePermission 用于判断哪些权限会在界面中显示，那些不显示：

```java
    private boolean isDisplayablePermission(PermissionInfo pInfo, int newReqFlags,
            int existingReqFlags) {
        //【1】获得基本权限类型；
        final int base = pInfo.protectionLevel & PermissionInfo.PROTECTION_MASK_BASE;
        final boolean isNormal = (base == PermissionInfo.PROTECTION_NORMAL);

        //【2】如果是 normal 权限类型，那么我们不会在界面显示，返回 false；
        if (isNormal) {
            return false;
        }
        //【3】新应用的该权限是否是 dangerous 类型的权限；
        final boolean isDangerous = (base == PermissionInfo.PROTECTION_DANGEROUS)
                || ((pInfo.protectionLevel&PermissionInfo.PROTECTION_FLAG_PRE23) != 0);
        //【3】新应用的该权限是否是运行必须的；
        final boolean isRequired =
                ((newReqFlags&PackageInfo.REQUESTED_PERMISSION_REQUIRED) != 0);
        //【4】新应用的该权限是否是 development 的签名权限！
        final boolean isDevelopment =
                ((pInfo.protectionLevel&PermissionInfo.PROTECTION_FLAG_DEVELOPMENT) != 0);
        //【5】已存在的应用的该权限是否是被授予的！
        final boolean wasGranted =
                ((existingReqFlags&PackageInfo.REQUESTED_PERMISSION_GRANTED) != 0);
        //【6】新应用的该权限是否是授予的！
        final boolean isGranted =
                ((newReqFlags&PackageInfo.REQUESTED_PERMISSION_GRANTED) != 0);

        //【7】如果该权限是 dangerous 的，那必须给用户显示出来！！
        if (isDangerous && (isRequired || wasGranted || isGranted)) {
            return true;
        }

        //【8】如果 development 的权限，并且已存在的应用的该权限是被授予的，
        // 那么我们会给用户显示出来！
        if (isDevelopment && wasGranted) {
            if (localLOGV) Log.i(TAG, "Special perm " + pInfo.name
                    + ": protlevel=0x" + Integer.toHexString(pInfo.protectionLevel));
            return true;
        }
        return false;
    }
```
不多说了！

#### 2.1.1.2 new MyPermissionGroupInfo

创建一个权限组对象：

```java
    /** @hide */
    static class MyPermissionGroupInfo extends PermissionGroupInfo {
        CharSequence mLabel;
        // 所有属于该权限组的该应用申请的被视为新的权限（很绕口我知道。。）
        final ArrayList<MyPermissionInfo> mNewPermissions = new ArrayList<MyPermissionInfo>();
        // 所有属于该权限组的该应用申请的权限
        final ArrayList<MyPermissionInfo> mAllPermissions = new ArrayList<MyPermissionInfo>();

        MyPermissionGroupInfo(PermissionInfo perm) {
            name = perm.packageName; // 权限组名；
            packageName = perm.packageName; // 所属包名；
        }

        MyPermissionGroupInfo(PermissionGroupInfo info) {
            super(info);
        }

        public Drawable loadGroupIcon(Context context, PackageManager pm) {
            if (icon != 0) {
                return loadUnbadgedIcon(pm);
            } else {
                return context.getDrawable(R.drawable.ic_perm_device_info);
            }
        }
    }
```

#### 2.1.1.3 new MyPermissionInfo

```java
    /** @hide */
    private static class MyPermissionInfo extends PermissionInfo {
        CharSequence mLabel;

        // 新安装的应用的该权限标志位，值来自 requestedPermissionsFlags
        int mNewReqFlags;

        // 已经被安装的应用（如果有）的该权限标志位，值来自 requestedPermissionsFlags
        int mExistingReqFlags;

        // 判断该应用是否被视为一个新的权限！
        boolean mNew;

        MyPermissionInfo(PermissionInfo info) {
            super(info);
        }
    }

```


### 2.1.2 AppSecurityPermissions.getAllUsedPermissions

获得共享 uid 的权限！

```java
    private void getAllUsedPermissions(int sharedUid, Set<MyPermissionInfo> permSet) {
        //【1】获得所有共享了该 uid 的 pkg！
        String sharedPkgList[] = mPm.getPackagesForUid(sharedUid);
        if(sharedPkgList == null || (sharedPkgList.length == 0)) {
            return;
        }
        //【*2.1.2.1】获得其 request 的权限！
        for(String sharedPkg : sharedPkgList) {
            getPermissionsForPackage(sharedPkg, permSet);
        }
    }
```
#### 2.1.2.1 AppSecurityPermissions.getPermissionsForPackage

继续处理：

```java
    private void getPermissionsForPackage(String packageName, Set<MyPermissionInfo> permSet) {
        try {
            //【1】获得该 packageName 对应的应用程序！
            PackageInfo pkgInfo = mPm.getPackageInfo(packageName, PackageManager.GET_PERMISSIONS);
            //【*2.1.1】又调用了 extractPerms 方法！
            extractPerms(pkgInfo, permSet, pkgInfo);
        } catch (NameNotFoundException e) {
            Log.w(TAG, "Couldn't retrieve permissions for package: " + packageName);
        }
    }

```
不多说了！


### 2.1.3 AppSecurityPermissions.setPermissions

设置权限，将权限设置到对应的权限组中！

```java
    private void setPermissions(List<MyPermissionInfo> permList) {
        //【1】将申请的权限加入对应的 group 中！
        if (permList != null) {
            for (MyPermissionInfo pInfo : permList) {
                if(localLOGV) Log.i(TAG, "Processing permission:"+pInfo.name);
                //【*2.1.1.1】调用 isDisplayablePermission 判断权限是否不用显示给用户！
                if(!isDisplayablePermission(pInfo, pInfo.mNewReqFlags, pInfo.mExistingReqFlags)) {
                    if(localLOGV) Log.i(TAG, "Permission:"+pInfo.name+" is not displayable");
                    continue;
                }
                //【1.2】将要申请的权限加入到对应的权限中！
                MyPermissionGroupInfo group = mPermGroups.get(pInfo.group);
                if (group != null) {
                    pInfo.mLabel = pInfo.loadLabel(mPm);
                    //【*2.1.3.1】添加到 group.mAllPermissions 内部集合中；
                    addPermToList(group.mAllPermissions, pInfo);
                    if (pInfo.mNew) {
                        //【*2.1.3.1】添加到 group.mNewPermissions 内部集合中；
                        addPermToList(group.mNewPermissions, pInfo);
                    }
                }
            }
        }
        //【2】设置每个权限组的 label！
        for (MyPermissionGroupInfo pgrp : mPermGroups.values()) {
            if (pgrp.labelRes != 0 || pgrp.nonLocalizedLabel != null) {
                pgrp.mLabel = pgrp.loadLabel(mPm);
            } else {
                ApplicationInfo app;
                try {
                    app = mPm.getApplicationInfo(pgrp.packageName, 0);
                    pgrp.mLabel = app.loadLabel(mPm);
                } catch (NameNotFoundException e) {
                    pgrp.mLabel = pgrp.loadLabel(mPm);
                }
            }
            //【2.1】将权限添加到 mPermGroupsList 中！
            mPermGroupsList.add(pgrp);
        }
        //【3】对权限组进行排序！
        Collections.sort(mPermGroupsList, mPermGroupComparator);
    }
```
#### 2.1.3.1 AppSecurityPermissions.setPermissions
```java
    private void addPermToList(List<MyPermissionInfo> permList,
            MyPermissionInfo pInfo) {
        if (pInfo.mLabel == null) {
            pInfo.mLabel = pInfo.loadLabel(mPm);
        }
        //【1】这里会使用排序方式找到合适的位置，使用比较器是 mPermComparator！
        int idx = Collections.binarySearch(permList, pInfo, mPermComparator);
        if(localLOGV) Log.i(TAG, "idx="+idx+", list.size="+permList.size());
        if (idx < 0) {
            idx = -idx-1;
            //【1.1】插入到 permList 列表中；
            permList.add(idx, pInfo);
        }
    }
```

## 2.2 AppSecurityPermissions.getPermissionCount

获得权限的数量：

```java
    public int getPermissionCount(int which) {
        int N = 0;
        for (int i=0; i<mPermGroupsList.size(); i++) {
            //【*2.2.1】返回每个 group 的权限个数！
            N += getPermissionList(mPermGroupsList.get(i), which).size();
        }
        return N;
    }
```
不多说了！

### 2.2.1 AppSecurityPermissions.getPermissionList

根据 which 指定的标志，返回 grp.mNewPermissions 或者 grp.mAllPermissions
```java
    private List<MyPermissionInfo> getPermissionList(MyPermissionGroupInfo grp, int which) {
        if (which == WHICH_NEW) {
            return grp.mNewPermissions;
        } else {
            return grp.mAllPermissions;
        }
    }
```
具体的意思很简单了，就不多说了！


# 3 onClick -> trigger to install

```java
    public void onClick(View v) {
        if (v == mOk) {
            if (mOkCanInstall || mScrollView == null) {
                if (mSessionId != -1) {
                    //【1】这里对应的是 onCreate 中的 confirm permission 的广播
                    // 这里会进入系统进程，通知权限确认完成，继续下一阶段安装！
                    //【*3.1】通知 PackageInstallerService 权限确认完成！
                    mInstaller.setPermissionsResult(mSessionId, true);
                    clearCachedApkIfNeededAndFinish();
                } else {
                    //【*4】正常情况会进入 startInstall 阶段！
                    startInstall();
                }
            } else {
                //【2】强制滚动，让用户看完权限信息；
                mScrollView.pageScroll(View.FOCUS_DOWN);
            }
        } else if (v == mCancel) {
            //【3】当然，如果用户点击了取消，如果 mSessionId 不为 -1，
            // 那么也需要进入系统进程，通知系统无需继续安装！
            setResult(RESULT_CANCELED);
            if (mSessionId != -1) {
                //【*3.1】通知 PackageInstallerService 权限确认完成！
                mInstaller.setPermissionsResult(mSessionId, false);
            }
            clearCachedApkIfNeededAndFinish();
        }
    }
```

## 3.1 PackageInstallerService.setPermissionsResult

```java
    @Override
    public void setPermissionsResult(int sessionId, boolean accepted) {
        mContext.enforceCallingOrSelfPermission(android.Manifest.permission.INSTALL_PACKAGES, TAG);

        synchronized (mSessions) {
            PackageInstallerSession session = mSessions.get(sessionId);
            if (session != null) {
                //【*3.2】根据指定的 sessionId，找到对应的 PackageInstallerSession
                session.setPermissionsResult(accepted);
            }
        }
    }
```
## 3.2 PackageInstallerSession.setPermissionsResult
```java
    void setPermissionsResult(boolean accepted) {
        if (!mSealed) {
            throw new SecurityException("Must be sealed to accept permissions");
        }

        if (accepted) {
            //【1】这里设置了 mPermissionsAccepted 为 true，这样在下面的继续安装中，就不会重复确权了！
            synchronized (mLock) {
                mPermissionsAccepted = true;
            }
            //【2】发送 MSG_COMMIT 消息，继续安装！
            mHandler.obtainMessage(MSG_COMMIT).sendToTarget();
        } else {
            destroyInternal();
            dispatchSessionFinished(INSTALL_FAILED_ABORTED, "User rejected permissions", null);
        }
    }
```


下面，我们继续看 startInstall 方法：


# 4 PackageInstallerActivity.startInstall

继续分析：

```java
    private void startInstall() {
        //【1】创建了一个 intent，封装要安装的应用程序！
        Intent newIntent = new Intent();
        //【2】传递要安装的应用程序的 ApplicationInfo 实例；
        newIntent.putExtra(PackageUtil.INTENT_ATTR_APPLICATION_INFO,
                mPkgInfo.applicationInfo);
        newIntent.setData(mPackageURI);
        //【2】目标组件 InstallAppProgress！
        newIntent.setClass(this, InstallAppProgress.class);
        String installerPackageName = getIntent().getStringExtra(
                Intent.EXTRA_INSTALLER_PACKAGE_NAME);
        if (mOriginatingURI != null) {
            newIntent.putExtra(Intent.EXTRA_ORIGINATING_URI, mOriginatingURI);
        }
        if (mReferrerURI != null) {
            newIntent.putExtra(Intent.EXTRA_REFERRER, mReferrerURI);
        }
        if (mOriginatingUid != VerificationParams.NO_UID) {
            newIntent.putExtra(Intent.EXTRA_ORIGINATING_UID, mOriginatingUid);
        }
        if (installerPackageName != null) {
            newIntent.putExtra(Intent.EXTRA_INSTALLER_PACKAGE_NAME,
                    installerPackageName);
        }
        //【3】如果安装的启动者 start 的时候，指定了接收返回结果，此时我们启动了一个新的 activity
        // InstallAppProgress，所以这里指定 InstallAppProgress 会将安装结果返回给启动者！
        // 而不是启动 InstallAppProgress 的界面！
        if (getIntent().getBooleanExtra(Intent.EXTRA_RETURN_RESULT, false)) {
            newIntent.putExtra(Intent.EXTRA_RETURN_RESULT, true);
            //【3.1】这种效果是通过 FLAG_ACTIVITY_FORWARD_RESULT 标志位实现的！
            newIntent.addFlags(Intent.FLAG_ACTIVITY_FORWARD_RESULT);
        }
        if(localLOGV) Log.i(TAG, "downloaded app uri="+mPackageURI);
        startActivity(newIntent);
        finish();
    }
```
关于 start activity，我们有在 ams 的相关内容中看到！


# 5 InstallAppProgress

## 5.1 onCreate

```java
    @Override
    public void onCreate(Bundle icicle) {
        super.onCreate(icicle);
        Intent intent = getIntent();
        mAppInfo = intent.getParcelableExtra(PackageUtil.INTENT_ATTR_APPLICATION_INFO);
        mPackageURI = intent.getData();

        final String scheme = mPackageURI.getScheme();
        if (scheme != null && !"file".equals(scheme) && !"package".equals(scheme)) {
            throw new IllegalArgumentException("unexpected scheme " + scheme);
        }
        //【1】启动了一个 HandlerThread 用于安装；
        mInstallThread = new HandlerThread("InstallThread");
        mInstallThread.start();
        mInstallHandler = new Handler(mInstallThread.getLooper());

        IntentFilter intentFilter = new IntentFilter();
        intentFilter.addAction(BROADCAST_ACTION);
        //【*5.1.1】动态注册了一个广播接收者，监听 ACTION_INSTALL_COMMIT 广播！
        registerReceiver(
                mBroadcastReceiver, intentFilter, BROADCAST_SENDER_PERMISSION, null /*scheduler*/);

        //【*5.2】初始化界面！
        initView();
    }
```
InstallAppProgress 显示后，会注册一个接收者，监听下面的广播：
```java
    private static final String BROADCAST_ACTION =
            "com.android.packageinstaller.ACTION_INSTALL_COMMIT";
```
并且发送者要具有这样的权限：
```java
    private static final String BROADCAST_SENDER_PERMISSION =
            "android.permission.INSTALL_PACKAGES";
```

### 5.1.1 new BroadcastReceiver

用于接收 com.android.packageinstaller.ACTION_INSTALL_COMMIT 广播！！

```java
    private final BroadcastReceiver mBroadcastReceiver = new BroadcastReceiver() {
        @Override
        public void onReceive(Context context, Intent intent) {
            final int statusCode = intent.getIntExtra(
                    PackageInstaller.EXTRA_STATUS, PackageInstaller.STATUS_FAILURE);
            if (statusCode == PackageInstaller.STATUS_PENDING_USER_ACTION) {
                context.startActivity((Intent)intent.getParcelableExtra(Intent.EXTRA_INTENT));
            } else {
                onPackageInstalled(statusCode);
            }
        }
    };
```
我们在分析 pm install 的时候：

```java
        //【1】mPermissionsAccepted 为 true，那么用户就可以静默安装了，如果为 false，那么就需要用户确认权限！！
        if (!mPermissionsAccepted) {
            //【1.1】这里会创建一个 intent， action 为 PackageInstaller.ACTION_CONFIRM_PERMISSIONS
            // 目标应用是 PackageInstaller，这里会先进入 packageinstaller 中确认权限信息！
            final Intent intent = new Intent(PackageInstaller.ACTION_CONFIRM_PERMISSIONS);
            intent.setPackage(mContext.getPackageManager().getPermissionControllerPackageName());
            intent.putExtra(PackageInstaller.EXTRA_SESSION_ID, sessionId); // 事务 id 也要从传递过去！
            try {
                //【*4.1.1】回调了 PackageInstallObserverAdapter 的 onUserActionRequired 接口
                // 将 intent 传递过去！
                mRemoteObserver.onUserActionRequired(intent);
            } catch (RemoteException ignored) {
            }
            //【*4.3.3】关闭该事务，使其从 active 变为 idle 状态！！
            close();
            // 停止安装，等待用户确认权限，用户在 PackageInstaller 点击安装，安装会继续！！
            return;
        }
```
当我们需要用户确认权限的时候，会创建一个 intent，action 为 ACTION_CONFIRM_PERMISSIONS，这里的逻辑，然后会掉

RemoteObserver.onUserActionRequired 方法！

## 5.2 initView

继续来看：

```java
    void initView() {
        setContentView(R.layout.op_progress);
        //【1】这里有要创建一个 AppSnippet 实例！
        final PackageUtil.AppSnippet as;
        final PackageManager pm = getPackageManager();
        //【*5.2.1】解析安装标志位， 主要作用是指定安装方式，是取代已经存在的，还是全新安装！
        final int installFlags = getInstallFlags(mAppInfo.packageName);

        if((installFlags & PackageManager.INSTALL_REPLACE_EXISTING )!= 0) {
            Log.w(TAG, "Replacing package:" + mAppInfo.packageName);
        }
        if ("package".equals(mPackageURI.getScheme())) {
            //【*1.1.2.1】又创建了一个 AppSnippet 实例！
            as = new PackageUtil.AppSnippet(pm.getApplicationLabel(mAppInfo),
                    pm.getApplicationIcon(mAppInfo));
        } else {
            //【*1.1.3】又创建了一个 AppSnippet 实例！
            final File sourceFile = new File(mPackageURI.getPath());
            as = PackageUtil.getAppSnippet(this, mAppInfo, sourceFile);
        }
        mLabel = as.label;

        //【*1.1.4】初始化 AppSnippet 的相关属性，icon 等等！
        PackageUtil.initSnippetForNewApp(this, as, R.id.app_snippet);
        
        //【2】初始化界面！
        mStatusTextView = (TextView)findViewById(R.id.center_text);
        mExplanationTextView = (TextView) findViewById(R.id.explanation);
        mProgressBar = (ProgressBar) findViewById(R.id.progress_bar);
        mProgressBar.setIndeterminate(true);
        // Hide button till progress is being displayed
        mOkPanel = findViewById(R.id.buttons_panel);
        mDoneButton = (Button)findViewById(R.id.done_button);
        mLaunchButton = (Button)findViewById(R.id.launch_button);
        mOkPanel.setVisibility(View.INVISIBLE);

        //【3】如果 scheme 指定的是 package，这种情况是安装一个已经存在的 pkg，会进入 if 分支！
        // 如果是 file 类型，进入 else！
        if ("package".equals(mPackageURI.getScheme())) {
            try {
                //【×5.2.2】调用 pms 的 installExistingPackage 方法执行安装！
                pm.installExistingPackage(mAppInfo.packageName);
                //【×5.2.3】通知安装结果！
                onPackageInstalled(PackageInstaller.STATUS_SUCCESS);
            } catch (PackageManager.NameNotFoundException e) {
                onPackageInstalled(PackageInstaller.STATUS_FAILURE_INVALID);
            }
        } else {
            //【4】这里会创建一个 SessionParams 封装事务参数，这里在 pm install 中有讲过！
            final PackageInstaller.SessionParams params = new PackageInstaller.SessionParams(
                    PackageInstaller.SessionParams.MODE_FULL_INSTALL);
            params.referrerUri = getIntent().getParcelableExtra(Intent.EXTRA_REFERRER);
            params.originatingUri = getIntent().getParcelableExtra(Intent.EXTRA_ORIGINATING_URI);
            params.originatingUid = getIntent().getIntExtra(Intent.EXTRA_ORIGINATING_UID,
                    UID_UNKNOWN);
            //【5】创建一个 File 对象，指向 apk 的路径！
            File file = new File(mPackageURI.getPath());
            try {
                //【6】解析 package，获得 PackageLite 对象，parsePackageLite 我们在开机启动中有讲过，这里就不多说了！
                PackageLite pkg = PackageParser.parsePackageLite(file, 0);
                //【7】设置包名，安装路径，大小到 SessionParams 中！
                params.setAppPackageName(pkg.packageName);
                params.setInstallLocation(pkg.installLocation);
                params.setSize(
                    PackageHelper.calculateInstalledSize(pkg, false, params.abiOverride));
            } catch (PackageParser.PackageParserException e) {
                Log.e(TAG, "Cannot parse package " + file + ". Assuming defaults.");
                Log.e(TAG, "Cannot calculate installed size " + file + ". Try only apk size.");
                params.setSize(file.length());
            } catch (IOException e) {
                Log.e(TAG, "Cannot calculate installed size " + file + ". Try only apk size.");
                params.setSize(file.length());
            }
            
            //【8】在子线程中，执行安装！
            mInstallHandler.post(new Runnable() {
                @Override
                public void run() {
                    //【×5.3】触发安装！
                    doPackageStage(pm, params);
                }
            });
        }
    }
```

### 5.2.1 getInstallFlags


```java
    int getInstallFlags(String packageName) {
        PackageManager pm = getPackageManager();
        try {
            //【1】尝试获得包名为本次要安装的应用的安装信息!
            PackageInfo pi =
                    pm.getPackageInfo(packageName, PackageManager.GET_UNINSTALLED_PACKAGES);
            if (pi != null) {
                //【2】如果已经安装了，那么会返回 INSTALL_REPLACE_EXISTING！
                return PackageManager.INSTALL_REPLACE_EXISTING;
            }
        } catch (NameNotFoundException e) {
        }
        return 0;
    }
```

### 5.2.2 PackageManagerService.installExistingPackage

安装一个系统中已存在的包，这是一个 hide 方法：

```java
    @Override
    public int installExistingPackageAsUser(String packageName, int userId) {
        mContext.enforceCallingOrSelfPermission(android.Manifest.permission.INSTALL_PACKAGES,
                null);
        PackageSetting pkgSetting;
        final int uid = Binder.getCallingUid();
        //【1】校验权限！
        enforceCrossUserPermission(uid, userId,
                true /* requireFullPermission */, true /* checkShell */,
                "installExistingPackage for user " + userId);
        //【2】如果设置了不能安装应用的用户限制；
        if (isUserRestricted(userId, UserManager.DISALLOW_INSTALL_APPS)) {
            return PackageManager.INSTALL_FAILED_USER_RESTRICTED;
        }

        long callingId = Binder.clearCallingIdentity();
        try {
            boolean installed = false;

            synchronized (mPackages) {
                //【3】判断该应用是否存在，不存在的话，返回 INSTALL_FAILED_INVALID_URI！
                pkgSetting = mSettings.mPackages.get(packageName);
                if (pkgSetting == null) {
                    return PackageManager.INSTALL_FAILED_INVALID_URI;
                }
                //【4】更新 pkg 的 install 状态为 true，hide 状态为 false，并更新偏好设置！
                if (!pkgSetting.getInstalled(userId)) {
                    pkgSetting.setInstalled(true, userId);
                    pkgSetting.setHidden(false, userId);
                    mSettings.writePackageRestrictionsLPr(userId);
                    installed = true;
                }
            }

            if (installed) {
                if (pkgSetting.pkg != null) {
                    synchronized (mInstallLock) {
                        //【4】准备数据目录！
                        prepareAppDataAfterInstallLIF(pkgSetting.pkg);
                    }
                }
                //【5】发送 Intent.ACTION_PACKAGE_ADDED 给所有的设备用户！！
                sendPackageAddedForUser(packageName, pkgSetting, userId);
            }
        } finally {
            Binder.restoreCallingIdentity(callingId);
        }

        return PackageManager.INSTALL_SUCCEEDED;
    }
```
不多说了！

### 5.2.3 onPackageInstalled

发送 INSTALL_COMPLETE 消息给主线程的 Handler：

```java
    void onPackageInstalled(int statusCode) {
        //【*5.2.3.1】进入主线程的 handler！
        Message msg = mHandler.obtainMessage(INSTALL_COMPLETE);
        msg.arg1 = statusCode;
        mHandler.sendMessage(msg);
    }
```

#### 5.2.3.1 Handler.handleMessage[INSTALL_COMPLETE]

主要根据安装的结果！

```java
    private Handler mHandler = new Handler() {
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case INSTALL_COMPLETE:
                    //【1】判断下本次安装是否需要返回结果给启动安装的程序，如果需要，那就返回结果！
                    // 返回！
                    if (getIntent().getBooleanExtra(Intent.EXTRA_RETURN_RESULT, false)) {
                        Intent result = new Intent();
                        result.putExtra(Intent.EXTRA_INSTALL_RESULT, msg.arg1);
                        setResult(msg.arg1 == PackageInstaller.STATUS_SUCCESS
                                ? Activity.RESULT_OK : Activity.RESULT_FIRST_USER,
                                        result);
                        clearCachedApkIfNeededAndFinish();
                        return;
                    }
                    //【2】更新界面显示！
                    mProgressBar.setVisibility(View.GONE);
                    // Show the ok button
                    int centerTextLabel;
                    int centerExplanationLabel = -1;
                    
                    //【3】处理返回结果码！
                    if (msg.arg1 == PackageInstaller.STATUS_SUCCESS) {
                        //【3.1】安装成功的情况！
                        mLaunchButton.setVisibility(View.VISIBLE);
                        ((ImageView)findViewById(R.id.center_icon))
                                .setImageDrawable(getDrawable(R.drawable.ic_done_92));
                        centerTextLabel = R.string.install_done;
                        // Enable or disable launch button
                        mLaunchIntent = getPackageManager().getLaunchIntentForPackage(
                                mAppInfo.packageName);
                        boolean enabled = false;
                        if(mLaunchIntent != null) {
                            List<ResolveInfo> list = getPackageManager().
                                    queryIntentActivities(mLaunchIntent, 0);
                            if (list != null && list.size() > 0) {
                                enabled = true;
                            }
                        }
                        if (enabled) {
                            mLaunchButton.setOnClickListener(InstallAppProgress.this);
                        } else {
                            mLaunchButton.setEnabled(false);
                        }
                    } else if (msg.arg1 == PackageInstaller.STATUS_FAILURE_STORAGE){
                        //【3.2】空间不足安装失败！
                        showDialogInner(DLG_OUT_OF_SPACE);
                        return;
                    } else {
                        //【3.2】空间不足安装失败！
                        ((ImageView)findViewById(R.id.center_icon))
                                .setImageDrawable(getDrawable(R.drawable.ic_report_problem_92));
                        centerExplanationLabel = getExplanationFromErrorCode(msg.arg1);
                        centerTextLabel = R.string.install_failed;
                        mLaunchButton.setVisibility(View.GONE);
                    }
                    if (centerExplanationLabel != -1) {
                        mExplanationTextView.setText(centerExplanationLabel);
                        findViewById(R.id.center_view).setVisibility(View.GONE);
                        ((TextView)findViewById(R.id.explanation_status)).setText(centerTextLabel);
                        findViewById(R.id.explanation_view).setVisibility(View.VISIBLE);
                    } else {
                        ((TextView)findViewById(R.id.center_text)).setText(centerTextLabel);
                        findViewById(R.id.center_view).setVisibility(View.VISIBLE);
                        findViewById(R.id.explanation_view).setVisibility(View.GONE);
                    }
                    mDoneButton.setOnClickListener(InstallAppProgress.this);
                    mOkPanel.setVisibility(View.VISIBLE);
                    break;
                default:
                    break;
            }
        }
    };
```
不多说了！

## 5.3 doPackageStage

到这里，已经和 pm install 中很类似了：
```java
    private void doPackageStage(PackageManager pm, PackageInstaller.SessionParams params) {
        final PackageInstaller packageInstaller = pm.getPackageInstaller();
        PackageInstaller.Session session = null;
        try {
            final String packageLocation = mPackageURI.getPath();
            final File file = new File(packageLocation);
            //【1】这里奉陪了一个 sessionid 给本次安装，这个方法我们在 pm install 中分析过！
            final int sessionId = packageInstaller.createSession(params);
            final byte[] buffer = new byte[65536];
            
            //【2】这里根据上面分配的 sessionid，创建了一个 Session，这个方法我们在 pm install 中分析过！
            session = packageInstaller.openSession(sessionId);

            final InputStream in = new FileInputStream(file);
            final long sizeBytes = file.length();
            final OutputStream out = session.openWrite("PackageInstaller", 0, sizeBytes);
            try {
                int c;
                while ((c = in.read(buffer)) != -1) {
                    out.write(buffer, 0, c);
                    if (sizeBytes > 0) {
                        final float fraction = ((float) c / (float) sizeBytes);
                        session.addProgress(fraction);
                    }
                }
                session.fsync(out);
            } finally {
                IoUtils.closeQuietly(in);
                IoUtils.closeQuietly(out);
            }

            //【2】创建了一个 PendingIntent 封装了上面的 action： com.android.packageinstaller.ACTION_INSTALL_COMMIT
            Intent broadcastIntent = new Intent(BROADCAST_ACTION);
            PendingIntent pendingIntent = PendingIntent.getBroadcast(
                    InstallAppProgress.this /*context*/,
                    sessionId,
                    broadcastIntent,
                    PendingIntent.FLAG_UPDATE_CURRENT);
    
            //【4】提交事务，触发安装，这个方法我们在 pm install 中分析过！
            // 这里将 PendingIntent 的 IntentSender 传递给了系统进程，后续系统进程会通过 binder 通信触发 intent！
            session.commit(pendingIntent.getIntentSender());
        } catch (IOException e) {
            //【*5.2.3】异常情况，回调并结束！！
            onPackageInstalled(PackageInstaller.STATUS_FAILURE);
        } finally {
            IoUtils.closeQuietly(session);
        }
    }
```
按照结束后，pendingIntent 会被触发，然后发送 com.android.packageinstaller.ACTION_INSTALL_COMMIT 广播！


# 6 onCreateDialog -> notify user

下面我们来看下 onCreateDialog 的函数逻辑，这个会在安装过程中提示用户：

```java
    @Override
    public Dialog onCreateDialog(int id, Bundle bundle) {
        switch (id) {
        //【1】提示本次安装了来自位置来源！
        case DLG_UNKNOWN_SOURCES:
            return new AlertDialog.Builder(this)
                    .setMessage(R.string.unknown_apps_dlg_text)
                    .setNegativeButton(R.string.cancel, new DialogInterface.OnClickListener() {
                        public void onClick(DialogInterface dialog, int which) {
                            Log.i(TAG, "Finishing off activity so that user can navigate to settings manually");
                            finishAffinity();
                        }})
                    .setPositiveButton(R.string.settings, new DialogInterface.OnClickListener() {
                        public void onClick(DialogInterface dialog, int which) {
                            Log.i(TAG, "Launching settings");
                            //【6.1】设置未知来源开关！
                            launchSecuritySettings();
                        }
                    })
                    .setOnCancelListener(this)
                    .create();
                    
        //【2】当前用户是在另外用户的 profile 中，同时用户关闭了未知来源设置
        // 此时不能继续安装！
        case DLG_ADMIN_RESTRICTS_UNKNOWN_SOURCES:
            return new AlertDialog.Builder(this)
                    .setMessage(R.string.unknown_apps_admin_dlg_text)
                    .setPositiveButton(android.R.string.ok, new DialogInterface.OnClickListener() {
                        public void onClick(DialogInterface dialog, int which) {
                            finish();
                        }
                    })
                    .setOnCancelListener(this)
                    .create();
        
        //【3】当前用户是在另外用户的 profile 中，同时用户关闭了未知来源设置
        // 此时不能继续安装！！
        case DLG_PACKAGE_ERROR :
            return new AlertDialog.Builder(this)
                    .setMessage(R.string.Parse_error_dlg_text)
                    .setPositiveButton(R.string.ok, new DialogInterface.OnClickListener() {
                        public void onClick(DialogInterface dialog, int which) {
                            finish();
                        }
                    })
                    .setOnCancelListener(this)
                    .create();
                    
        //【4】安装空间不足，此时不能继续安装！！
        case DLG_OUT_OF_SPACE:
            CharSequence appTitle = mPm.getApplicationLabel(mPkgInfo.applicationInfo);
            String dlgText = getString(R.string.out_of_space_dlg_text,
                    appTitle.toString());
            return new AlertDialog.Builder(this)
                    .setMessage(dlgText)
                    .setPositiveButton(R.string.manage_applications, new DialogInterface.OnClickListener() {
                        public void onClick(DialogInterface dialog, int which) {
                            //【4.1】进入空间管理界面，用户可以选择释放空间！
                            Intent intent = new Intent("android.intent.action.MANAGE_PACKAGE_STORAGE");
                            intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
                            startActivity(intent);
                            finish();
                        }
                    })
                    .setNegativeButton(R.string.cancel, new DialogInterface.OnClickListener() {
                        public void onClick(DialogInterface dialog, int which) {
                            Log.i(TAG, "Canceling installation");
                            finish();
                        }
                })
                  .setOnCancelListener(this)
                  .create();
    
        //【5】安装失败！！
        case DLG_INSTALL_ERROR :
            CharSequence appTitle1 = mPm.getApplicationLabel(mPkgInfo.applicationInfo);
            String dlgText1 = getString(R.string.install_failed_msg,
                    appTitle1.toString());
            return new AlertDialog.Builder(this)
                    .setNeutralButton(R.string.ok, new DialogInterface.OnClickListener() {
                        public void onClick(DialogInterface dialog, int which) {
                            finish();
                        }
                    })
                    .setMessage(dlgText1)
                    .setOnCancelListener(this)
                    .create();
                    
        //【6】这个是 WEAR 可穿戴设备上的逻辑，这里不关注！
        case DLG_NOT_SUPPORTED_ON_WEAR:
            return new AlertDialog.Builder(this)
                    .setMessage(R.string.wear_not_allowed_dlg_text)
                    .setPositiveButton(R.string.ok, new DialogInterface.OnClickListener() {
                        public void onClick(DialogInterface dialog, int which) {
                            setResult(RESULT_OK);
                            clearCachedApkIfNeededAndFinish();
                        }
                    })
                    .setOnCancelListener(this)
                    .create();
       }
       return null;
    }
```
不多说了！

## 6.1 launchSecuritySettings

进入设置，让用户选择是否打开未知来源设置：
```java
    private void launchSecuritySettings() {
        Intent launchSettingsIntent = new Intent(Settings.ACTION_SECURITY_SETTINGS);
        startActivityForResult(launchSettingsIntent, REQUEST_ENABLE_UNKNOWN_SOURCES);
    }
```