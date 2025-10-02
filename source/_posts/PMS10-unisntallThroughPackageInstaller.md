# PMS 第 10 篇 - 通过 PackageInstaller 分析 uninstall 过程
title: PMS 第 10 篇 - 通过 PackageInstaller 分析 uninstall 过程
date: 2018/09/10
categories:
- AndroidFramework源码分析
- PackageManager包管理
tags: PackageManager包管理
---

[toc]

基于 Android7.1.1 分析 PackageManagerService 的架构设计！
  
# 0 综述  
  
前面总结了通过 pm uninstall 的方式来卸载一个 apk，下面我们来分析下通过 PackageInstaller 来卸载应用！ 
  
对于用户来说，他们最常用的卸载方式，就是进入应用管理，然后进入指定的应用界面，选择卸载应用：



我们通过 dumpsys window 指令，可以看到这个焦点弹窗：

```shell
sailfish:/ $ dumpsys window | grep mF
    mLastSystemUiFlags=0x8008 mResettingSystemUiFlags=0x0 mForceClearedSystemUiFlags=0x0
    mFocusedWindow=Window{b272ea0 u0 com.google.android.packageinstaller/com.android.packageinstaller.UninstallerActivity}
    mFocusedApp=Token{b7d3821 ActivityRecord{53e1688 u0 com.google.android.packageinstaller/com.android.packageinstaller.UninstallerActivity t55}}
    mForceStatusBar=false mForceStatusBarFromKeyguard=false
             mFillsParent=false mOrientation=-1
             mFillsParent=true mOrientation=-1
             mFillsParent=true mOrientation=-1
             mFillsParent=true mOrientation=-1
             mFillsParent=true mOrientation=-1
             mFillsParent=true mOrientation=5
             mFillsParent=true mOrientation=1
    mPolicyVisibility=false mPolicyVisibilityAfterAnim=false mAppOpVisibility=true parentHidden=false mPermanentlyHidden=false mHiddenWhileSuspended=false mForceHideNonSystemOverlayWindow=false
  mFocusedApp=AppWindowToken{5f12446 token=Token{b7d3821 ActivityRecord{53e1688 u0 com.google.android.packageinstaller/com.android.packageinstaller.UninstallerActivity t55}}}
```

可以看到，这个看起来像弹窗的界面，实际上是一个 Activity：

```java
com.google.android.packageinstaller/com.android.packageinstaller.UninstallerActivity
```

也就是说，拉起了 packageInstaller 去进行卸载操作，当我们点击卸载后，会触发如下的代码：

```java
@VisibleForTesting
void uninstallPkg(String packageName, boolean allUsers, boolean andDisable) {
	stopListeningToPackageRemove();
	// Create new intent to launch Uninstaller activity
	//【1】创建了一个 Uri，封装 packageName 的信息，然后通过 Intent 启动 UninstallerActivity！
	Uri packageUri = Uri.parse("package:" + packageName);
	Intent uninstallIntent = new Intent(Intent.ACTION_UNINSTALL_PACKAGE, packageUri);
	uninstallIntent.putExtra(Intent.EXTRA_UNINSTALL_ALL_USERS, allUsers);

	mMetricsFeatureProvider.action(
			mActivity, MetricsProto.MetricsEvent.ACTION_SETTINGS_UNINSTALL_APP);
	mFragment.startActivityForResult(uninstallIntent, mRequestUninstall);
	mDisableAfterUninstall = andDisable;
}
```

下面我们继续分析：

# 1 UninstallerActivity

## 1.1 onCreate

当我们点击了卸载时，会拉起 PackageInstaller 的 UninstallerActivity 界面：

卸载时，传入的 Uri 的格式如下：

- `package://<packageName>#<className>`，className 是额外的参数，如果被指定，表示用户要卸载的具体的 activity；

```java
    @Override
    public void onCreate(Bundle icicle) {
        super.onCreate(icicle);
        //【1】获得启动的 intent，以及其传递的 Uri！
        final Intent intent = getIntent();
        final Uri packageUri = intent.getData();

        //【2】packageUri 不能为 null；
        if (packageUri == null) {
            Log.e(TAG, "No package URI in intent");
            showAppNotFound();
            return;
        }
		//【3】包名也不能为 null；
        mPackageName = packageUri.getEncodedSchemeSpecificPart();
        if (mPackageName == null) {
            Log.e(TAG, "Invalid package name in URI: " + packageUri);
            showAppNotFound();
            return;
        }
        
        //【4】获得 pms 代理对象；
        final IPackageManager pm = IPackageManager.Stub.asInterface(
                ServiceManager.getService("package"));

        //【*1.1.1】创建一个 DialogInfo 对象，保存安装和显示相关的信息；
        mDialogInfo = new DialogInfo();

        //【5】获得卸载时，指定的用户 user，如果没有指定，默认是当前用户；
        mDialogInfo.user = intent.getParcelableExtra(Intent.EXTRA_USER);
        if (mDialogInfo.user == null) {
            mDialogInfo.user = android.os.Process.myUserHandle();
        }

        //【6】判断是否是从所有 user 下卸载；同时，获得卸载的回调 IBinder 对象；
        mDialogInfo.allUsers = intent.getBooleanExtra(Intent.EXTRA_UNINSTALL_ALL_USERS, false);
        mDialogInfo.callback = intent.getIBinderExtra(PackageInstaller.EXTRA_CALLBACK);

        //【7】获得要卸载的应用信息；
        try {
            mDialogInfo.appInfo = pm.getApplicationInfo(mPackageName,
                    PackageManager.GET_UNINSTALLED_PACKAGES, mDialogInfo.user.getIdentifier());
        } catch (RemoteException e) {
            Log.e(TAG, "Unable to get packageName. Package manager is dead?");
        }
        //【8】应用信息不能为 null
        if (mDialogInfo.appInfo == null) {
            Log.e(TAG, "Invalid packageName: " + mPackageName);
            showAppNotFound();
            return;
        }

        // The class name may have been specified (e.g. when deleting an app from all apps)
        //【9】如果指定了 actiivity，那么要获得该 activity 的信息对象；
        final String className = packageUri.getFragment();
        if (className != null) {
            try {
                mDialogInfo.activityInfo = pm.getActivityInfo(
                        new ComponentName(mPackageName, className), 0,
                        mDialogInfo.user.getIdentifier());
            } catch (RemoteException e) {
                Log.e(TAG, "Unable to get className. Package manager is dead?");
                // Continue as the ActivityInfo isn't critical.
            }
        }
        //【*1.2】继续下一步的处理！
        showConfirmationDialog();
    }
```

### 1.1.1 new DialogInfo

这里创建了一个 DialogInfo 对象，
```java
    static class DialogInfo {
        ApplicationInfo appInfo;
        ActivityInfo activityInfo;
        boolean allUsers;
        UserHandle user;
        IBinder callback;
    }
```

## 1.2 showConfirmationDialog

继续下一步的处理：

```java
    private void showConfirmationDialog() {
        //【*1.2.1】这里是切换到了一个 fragment 来显示信息；
        //【*2】这里会进入到 UninstallAlertDialogFragment 界面中；
        showDialogFragment(new UninstallAlertDialogFragment());
    }
```

### 1.2.1 showDialogFragment

切换显示 Fragment：

```java
    private void showDialogFragment(DialogFragment fragment) {
        FragmentTransaction ft = getFragmentManager().beginTransaction();
        Fragment prev = getFragmentManager().findFragmentByTag("dialog");
        if (prev != null) {
            ft.remove(prev);
        }
        fragment.show(ft, "dialog");
    }
```

## 1.3 startUninstallProgress

开始卸载：

```java
    void startUninstallProgress() {
        //【1】创建 Intent，传递参数：
        Intent newIntent = new Intent(Intent.ACTION_VIEW);
        //【2】要卸载的目标 user！
        newIntent.putExtra(Intent.EXTRA_USER, mDialogInfo.user);
        //【3】是否从所有用户下下载；
        newIntent.putExtra(Intent.EXTRA_UNINSTALL_ALL_USERS, mDialogInfo.allUsers);
        //【4】卸载回调
        newIntent.putExtra(PackageInstaller.EXTRA_CALLBACK, mDialogInfo.callback);
        //【5】要卸载的 app info
        newIntent.putExtra(PackageUtil.INTENT_ATTR_APPLICATION_INFO, mDialogInfo.appInfo);
        if (getIntent().getBooleanExtra(Intent.EXTRA_RETURN_RESULT, false)) {
            newIntent.putExtra(Intent.EXTRA_RETURN_RESULT, true);
            newIntent.addFlags(Intent.FLAG_ACTIVITY_FORWARD_RESULT);
        }
        newIntent.setClass(this, UninstallAppProgress.class);
        //【*3.1】进入 UninstallAppProgress 界面；   
        startActivity(newIntent);
    }
```

启动 UninstallAppProgress activity，进入卸载状态！


# 2 UninstallAlertDialogFragment

UninstallAlertDialogFragment 是 DialogFragment 的子类，实现了 DialogInterface.OnClickListener 接口。

我们去他的 onCreateDialog 方法中看看：

## 2.1 onCreateDialog

该方法会创建一个 Dialog：

```java
        @Override
        public Dialog onCreateDialog(Bundle savedInstanceState) {
            final PackageManager pm = getActivity().getPackageManager();

            //【*1.1】获得前面创建的 DialogInfo 实例
            final DialogInfo dialogInfo = ((UninstallerActivity) getActivity()).mDialogInfo;
            final CharSequence appLabel = dialogInfo.appInfo.loadLabel(pm);

            AlertDialog.Builder dialogBuilder = new AlertDialog.Builder(getActivity());
            StringBuilder messageBuilder = new StringBuilder();

            //【1】如果指定了 activity，那么同时 Activity label 不同于 App label
            // 这里就要显示的通知用户要卸载的 activity 属于该 app！
            if (dialogInfo.activityInfo != null) {
                final CharSequence activityLabel = dialogInfo.activityInfo.loadLabel(pm);
                if (!activityLabel.equals(appLabel)) {
                    // uninstall_activity_text：属于以下应用：
                    messageBuilder.append(
                            getString(R.string.uninstall_activity_text, activityLabel));
                    messageBuilder.append(" ").append(appLabel).append(".\n\n");
                }
            }
            //【2】判断下要卸载的应用是不是安装在 data 分区的 sys app 的更新；
            final boolean isUpdate =
                    ((dialogInfo.appInfo.flags & ApplicationInfo.FLAG_UPDATED_SYSTEM_APP) != 0);
            UserManager userManager = UserManager.get(getActivity());

            if (isUpdate) {
                //【3】如果卸载是安装在 data 分区的 sys app 的更新，那么要根据系统是否是 single user 
                // 提示不同的信息！
                if (isSingleUser(userManager)) {
                    //【*2.1.1】isSingleUser 判断是否是 single user！
                    // 提示："要将此应用替换为出厂版本吗？这样会移除所有数据。"
                    messageBuilder.append(getString(R.string.uninstall_update_text));
                    
                } else {
                    // 提示："要将此应用替换为出厂版本吗？这样会移除所有数据，并会影响此设备的所有用户
                    //（包括已设置工作资料的用户）。"
                    messageBuilder.append(getString(R.string.uninstall_update_text_multiuser));
                    
                }
             
            } else {
                //【4】如果卸载是安装在 data 分区的 user app，那么同样的，要根据系统是否是 single user
                // 以及卸载的 allUsers 参数来做不同的显示！
                if (dialogInfo.allUsers && !isSingleUser(userManager)) {
                    //【4.1】如果是要从所有用户下卸载，同时系统不是 single user 的，那么
                    // 提示："是否要为所有用户卸载此应用？系统将为设备上的所有用户删除此应用及其数据。"
                    messageBuilder.append(getString(R.string.uninstall_application_text_all_users));
                    
                } else if (!dialogInfo.user.equals(android.os.Process.myUserHandle())) {
                    //【4.2】如果指定了要卸载的用户 user，同时该用户不是当前用户 user！
                    // 提示："您要为用户 user 卸载此应用吗？"
                    UserInfo userInfo = userManager.getUserInfo(dialogInfo.user.getIdentifier());
                    messageBuilder.append(
                            getString(R.string.uninstall_application_text_user, userInfo.name));
                            
                } else {
                    //【4.3】其他情况，提示："要卸载此应用吗？"
                    messageBuilder.append(getString(R.string.uninstall_application_text));
                    
                }
            }

            //【5】创建一个 Dialog 并返回！
            dialogBuilder.setTitle(appLabel);
            dialogBuilder.setIcon(dialogInfo.appInfo.loadIcon(pm));
            dialogBuilder.setPositiveButton(android.R.string.ok, this);
            dialogBuilder.setNegativeButton(android.R.string.cancel, this);
            dialogBuilder.setMessage(messageBuilder.toString());
            return dialogBuilder.create();
        }
```
整个过程很简单，不多说了！

### 2.1.1 isSingleUser

判断系统是否是单用户：

```java
        private boolean isSingleUser(UserManager userManager) {
            final int userCount = userManager.getUserCount();
            return userCount == 1
                    || (UserManager.isSplitSystemUser() && userCount == 2);
        }
```

## 2.2 onClick

卸载的关键触发是在 UninstallAlertDialogFragment 的点击事件中：

```java
        @Override
        public void onClick(DialogInterface dialog, int which) {
            if (which == Dialog.BUTTON_POSITIVE) {
                //【*1.3】开始卸载！
                ((UninstallerActivity) getActivity()).startUninstallProgress();
            } else {
                ((UninstallerActivity) getActivity()).dispatchAborted();
            }
        }
```
不多说了！！

# 3 UninstallAppProgress

## 3.1 onCreate

下面是卸载界面的 onCreate 方法：

```java
    @Override
    public void onCreate(Bundle icicle) {
        super.onCreate(icicle);
        //【1】获得启动的 Intent；
        Intent intent = getIntent();
        //【2】获得要卸载的 app info
        mAppInfo = intent.getParcelableExtra(PackageUtil.INTENT_ATTR_APPLICATION_INFO);
        //【3】获得安装回调；
        mCallback = intent.getIBinderExtra(PackageInstaller.EXTRA_CALLBACK);

        //【4】这里是因为 UninstallAppProgress 不支持 onDestroy->onCreate 的数据恢复；
        // 如果是这种情况，结束安装；
        if (icicle != null) {
            mResultCode = PackageManager.DELETE_FAILED_INTERNAL_ERROR;
            //【4.1】如果指定了回调，那么会获得其代理对象，然后触发回调！
            if (mCallback != null) {
                final IPackageDeleteObserver2 observer = IPackageDeleteObserver2.Stub
                        .asInterface(mCallback);
                try {
                    observer.onPackageDeleted(mAppInfo.packageName, mResultCode, null);
                } catch (RemoteException ignored) {
                }
                finish();
            } else {
                //【4.2】如果没有指定回调，那么会发送 resultCode！
                setResultAndFinish(mResultCode);
            }

            return;
        }
        
        //【5】是否是从所有用户下下载；
        mAllUsers = intent.getBooleanExtra(Intent.EXTRA_UNINSTALL_ALL_USERS, false);
        //【6】如果是所有用户，而当前用户不是 AdminUser，那么会抛出异常；
        if (mAllUsers && !UserManager.get(this).isAdminUser()) {
            throw new SecurityException("Only admin user can request uninstall for all users");
        }
        //【7】是否指定了 user，如果没有指定那么默认就是当前 user；如果指定了，那么该 user 必须存在！
        mUser = intent.getParcelableExtra(Intent.EXTRA_USER);
        if (mUser == null) {
            mUser = android.os.Process.myUserHandle();
        } else {
            UserManager userManager = (UserManager) getSystemService(Context.USER_SERVICE);
            List<UserHandle> profiles = userManager.getUserProfiles();
            if (!profiles.contains(mUser)) {
                throw new SecurityException("User " + android.os.Process.myUserHandle() + " can't "
                        + "request uninstall for user " + mUser);
            }
        }
        //【*3.1.1】创建 PackageDeleteObserver 对象，接收卸载的回调；
        PackageDeleteObserver observer = new PackageDeleteObserver();

        // 使窗口透明，直到调用 initView。 在许多情况下，我们可以避免显示UI，因为应用程序很快就会被卸载。 
        // 如果我们显示 UI 并立即删除它，它看起来就像一个闪烁。
        getWindow().setBackgroundDrawable(new ColorDrawable(Color.TRANSPARENT));
        getWindow().setStatusBarColor(Color.TRANSPARENT);
        getWindow().setNavigationBarColor(Color.TRANSPARENT);

        //【*5】执行卸载操作！
        getPackageManager().deletePackageAsUser(mAppInfo.packageName, observer,
                mAllUsers ? PackageManager.DELETE_ALL_USERS : 0, mUser.getIdentifier());

        //【*4.1】延迟 500 ms，发送了 UNINSTALL_IS_SLOW 消息，初始化界面！
        mHandler.sendMessageDelayed(mHandler.obtainMessage(UNINSTALL_IS_SLOW),
                QUICK_INSTALL_DELAY_MILLIS);
    }
```

### 3.1.1 new PackageDeleteObserver

PackageDeleteObserver 接收安装结果！

```java
    class PackageDeleteObserver extends IPackageDeleteObserver.Stub {
        public void packageDeleted(String packageName, int returnCode) {
            //【4.2】卸载完成后，会发送 UNINSTALL_COMPLETE 消息！
            Message msg = mHandler.obtainMessage(UNINSTALL_COMPLETE);
            msg.arg1 = returnCode; // 保存了安装结果码；
            msg.obj = packageName;
            mHandler.sendMessage(msg);
        }
    }
```
这里就不多说了！


## 3.2 initView

```java
    public void initView() {
        if (mIsViewInitialized) {
            return;
        }
        mIsViewInitialized = true;

        // We set the window background to translucent in constructor, revert this
        TypedValue attribute = new TypedValue();
        getTheme().resolveAttribute(android.R.attr.windowBackground, attribute, true);
        if (attribute.type >= TypedValue.TYPE_FIRST_COLOR_INT &&
                attribute.type <= TypedValue.TYPE_LAST_COLOR_INT) {
            getWindow().setBackgroundDrawable(new ColorDrawable(attribute.data));
        } else {
            getWindow().setBackgroundDrawable(getResources().getDrawable(attribute.resourceId,
                    getTheme()));
        }

        getTheme().resolveAttribute(android.R.attr.navigationBarColor, attribute, true);
        getWindow().setNavigationBarColor(attribute.data);

        getTheme().resolveAttribute(android.R.attr.statusBarColor, attribute, true);
        getWindow().setStatusBarColor(attribute.data);
        
        //【1】判断下要卸载的应用是不是安装在 data 分区的 sys app 的更新，用于不同的显示！
        boolean isUpdate = ((mAppInfo.flags & ApplicationInfo.FLAG_UPDATED_SYSTEM_APP) != 0);
        setTitle(isUpdate ? R.string.uninstall_update_title : R.string.uninstall_application_title);

        setContentView(R.layout.uninstall_progress); // 设置布局！
        
        //【2】初始化 view！
        View snippetView = findViewById(R.id.app_snippet);
        //【3】使用 app info 初始化 snippetView，显示 app 信息！
        PackageUtil.initSnippetForInstalledApp(this, mAppInfo, snippetView);

        //【4】初始化设备管理按钮和用户管理按钮，默认是 gone，同时设置点击事件！
        mDeviceManagerButton = (Button) findViewById(R.id.device_manager_button);
        mUsersButton = (Button) findViewById(R.id.users_button);
        mDeviceManagerButton.setVisibility(View.GONE);
        mDeviceManagerButton.setOnClickListener(new OnClickListener() {
            @Override
            public void onClick(View v) {
                Intent intent = new Intent();
                intent.setClassName("com.android.settings",
                        "com.android.settings.Settings$DeviceAdminSettingsActivity");
                intent.setFlags(Intent.FLAG_ACTIVITY_NO_HISTORY | Intent.FLAG_ACTIVITY_NEW_TASK);
                startActivity(intent);
                finish();
            }
        });
        mUsersButton.setVisibility(View.GONE);
        mUsersButton.setOnClickListener(new OnClickListener() {
            @Override
            public void onClick(View v) {
                Intent intent = new Intent(Settings.ACTION_USER_SETTINGS);
                intent.setFlags(Intent.FLAG_ACTIVITY_NO_HISTORY | Intent.FLAG_ACTIVITY_NEW_TASK);
                startActivity(intent);
                finish();
            }
        });
  
        //【5】初始化完成按钮，默认也是 gone 在布局里设置的！
        mOkButton = (Button) findViewById(R.id.ok_button);
        mOkButton.setOnClickListener(this);
    }
```

# 4 mHandler - mainLooper

```java
    private Handler mHandler = new Handler() {
	    ... ... ...
    }
```
mHandler 持有 main thead 的 looper 对象，消息都会发送到主线程操作！

## 4.1 handleMessage[UNINSTALL_IS_SLOW]

发送 UNINSTALL_IS_SLOW 消息，触发 initView 操作：

```java
   case UNINSTALL_IS_SLOW:
       //【*3.2】初始化 view！
       initView();
       break;
```
不多说了！

## 4.2 handleMessage[UNINSTALL_COMPLETE]

发送 UNINSTALL_COMPLETE 消息，处理卸载结果和最终的显示：

```java
   case UNINSTALL_COMPLETE:
       mHandler.removeMessages(UNINSTALL_IS_SLOW);

       if (msg.arg1 != PackageManager.DELETE_SUCCEEDED) {
           //【*3.2】结果不是 success，再次初始化 view！
           initView();
       }

       mResultCode = msg.arg1;
       final String packageName = (String) msg.obj;

       //【1】如果指定了回调，那就通过回调返回结果，并 finish 掉 UninstallAppProgress！
       if (mCallback != null) {
           final IPackageDeleteObserver2 observer = IPackageDeleteObserver2.Stub
                   .asInterface(mCallback);
           try {
               observer.onPackageDeleted(mAppInfo.packageName, mResultCode,
                       packageName);
           } catch (RemoteException ignored) {
           }
           finish();
           return;
       }

       //【2】如果 intent 指定了返回启动结果，那么就通过 intent 返回卸载结果，
       // 并 finish 掉 UninstallAppProgress！
       if (getIntent().getBooleanExtra(Intent.EXTRA_RETURN_RESULT, false)) {
           Intent result = new Intent();
           result.putExtra(Intent.EXTRA_INSTALL_RESULT, mResultCode);
           setResult(mResultCode == PackageManager.DELETE_SUCCEEDED
                   ? Activity.RESULT_OK : Activity.RESULT_FIRST_USER,
                           result);
           finish();
           return;
       }

       //【3】更新界面显示内容；
       final String statusText;
       switch (msg.arg1) {
           //【3.1】卸载成功；
           case PackageManager.DELETE_SUCCEEDED:
               statusText = getString(R.string.uninstall_done);
               // Show a Toast and finish the activity
               Context ctx = getBaseContext();
               Toast.makeText(ctx, statusText, Toast.LENGTH_LONG).show();
               //【3.1.1】返回结果；
               setResultAndFinish(mResultCode);
               return;

           //【3.2】该 apk 是处于 active 状态的设备管理者，不可卸载；
           case PackageManager.DELETE_FAILED_DEVICE_POLICY_MANAGER: {
               UserManager userManager =
                       (UserManager) getSystemService(Context.USER_SERVICE);
               IDevicePolicyManager dpm = IDevicePolicyManager.Stub.asInterface(
                       ServiceManager.getService(Context.DEVICE_POLICY_SERVICE));
               // Find out if the package is an active admin for some non-current user.
               //【3.2.1】获得当前的 user id；
               int myUserId = UserHandle.myUserId();
               UserInfo otherBlockingUser = null;
               //【3.2.2】遍历所有的 user！
               for (UserInfo user : userManager.getUsers()) {
                   //【3.2.2.1】找到不是当前 user 或者不是当前 user 的 profile 的 user！
                   if (isProfileOfOrSame(userManager, myUserId, user.id)) continue;

                   try {
                       //【3.2.2.1】如果该 apk 在这个 user 下是活跃的设备管理者，将其保存到 
                       // otherBlockingUser 中；
                       if (dpm.packageHasActiveAdmins(packageName, user.id)) {
                           otherBlockingUser = user;
                           break;
                       }
                   } catch (RemoteException e) {
                       Log.e(TAG, "Failed to talk to package manager", e);
                   }
               }
               //【3.2.3】如果 otherBlockingUser 为 null，说明 apk 是当前 user 的活跃设备管理者；
               // 如果不为 null，说明在其他 user 下是活跃设备管理者；
               if (otherBlockingUser == null) {
                   Log.d(TAG, "Uninstall failed because " + packageName
                           + " is a device admin");
                   //【3.2.3.1】可以看到，这里 mDeviceManagerButton 是可以点击的！！！
                   mDeviceManagerButton.setVisibility(View.VISIBLE);
                   statusText = getString(
                           R.string.uninstall_failed_device_policy_manager);
               } else {
                   Log.d(TAG, "Uninstall failed because " + packageName
                           + " is a device admin of user " + otherBlockingUser);
                   mDeviceManagerButton.setVisibility(View.GONE);
                   statusText = String.format(
                           getString(R.string.uninstall_failed_device_policy_manager_of_user),
                           otherBlockingUser.name);
               }
               break;
           }
   
           //【3.3】该 apk 被设备管理者标记为不可卸载；
           case PackageManager.DELETE_FAILED_OWNER_BLOCKED: {
               UserManager userManager =
                       (UserManager) getSystemService(Context.USER_SERVICE);
               IPackageManager packageManager = IPackageManager.Stub.asInterface(
                       ServiceManager.getService("package"));
               //【3.3.1】首先找到，在那个 user 下是不可卸载的，只会找到第一个 user！
               List<UserInfo> users = userManager.getUsers();
               int blockingUserId = UserHandle.USER_NULL;
               for (int i = 0; i < users.size(); ++i) {
                   final UserInfo user = users.get(i);
                   try {
                       if (packageManager.getBlockUninstallForUser(packageName,
                               user.id)) {
                           blockingUserId = user.id;
                           break;
                       }
                   } catch (RemoteException e) {
                       // Shouldn't happen.
                       Log.e(TAG, "Failed to talk to package manager", e);
                   }
               }
               int myUserId = UserHandle.myUserId();
               //【3.3.2】如果该 user 是当前 user 或者是当前 user 的 profile，
               // 那么 Device Manager Button 可点击；
               // 否则 Users Button 可点击；
               if (isProfileOfOrSame(userManager, myUserId, blockingUserId)) {
                   mDeviceManagerButton.setVisibility(View.VISIBLE);

               } else {
                   mDeviceManagerButton.setVisibility(View.GONE);
                   mUsersButton.setVisibility(View.VISIBLE);
               }
               // TODO: b/25442806
               if (blockingUserId == UserHandle.USER_SYSTEM) {
                   // 提示："这是您的设备管理员要求必须安装的应用，因此无法卸载。"
                   statusText = getString(R.string.uninstall_blocked_device_owner);
                   
               } else if (blockingUserId == UserHandle.USER_NULL) {
                   Log.d(TAG, "Uninstall failed for " + packageName + " with code "
                           + msg.arg1 + " no blocking user");
                   // 提示："卸载失败。"
                   statusText = getString(R.string.uninstall_failed);
               } else {
                   // 如果是所有用户下卸载，
                   // 提示："这是部分用户或个人资料所需的应用；已为其他用户或个人资料卸载此应用"
                   // 否则提示："这是您的个人资料所需的应用，因此无法卸载。"
                   statusText = mAllUsers
                           ? getString(R.string.uninstall_all_blocked_profile_owner) :
                           getString(R.string.uninstall_blocked_profile_owner);
               }
               break;
           }
          
           default:
               Log.d(TAG, "Uninstall failed for " + packageName + " with code "
                       + msg.arg1);
               statusText = getString(R.string.uninstall_failed);
               break;
       }
       findViewById(R.id.progress_view).setVisibility(View.GONE);
       findViewById(R.id.status_view).setVisibility(View.VISIBLE);
       ((TextView)findViewById(R.id.status_text)).setText(statusText);
   
       //【4】将完成按钮显示出来！
       findViewById(R.id.ok_panel).setVisibility(View.VISIBLE);
       break;
```


# 5 PackageManagerService

最终的卸载，进入了 pms：
```java
    @Override
    public void deletePackageAsUser(String packageName, IPackageDeleteObserver observer, int userId,
            int flags) {
        //【*important】这里和 adb uninstall 的逻辑一样了！    
        //【*5.1】对前面的回调做二次封装；
        deletePackage(packageName, new LegacyPackageDeleteObserver(observer).getBinder(), userId,
                flags);
    }
```

## 5.1 new LegacyPackageDeleteObserver

LegacyPackageDeleteObserver 的定义是在 PackageManager.java 中：

```java
    /** {@hide} */
    public static class LegacyPackageDeleteObserver extends PackageDeleteObserver {
        private final IPackageDeleteObserver mLegacy;

        public LegacyPackageDeleteObserver(IPackageDeleteObserver legacy) {
            mLegacy = legacy;
        }

        @Override
        public void onPackageDeleted(String basePackageName, int returnCode, String msg) {
            if (mLegacy == null) return;
            try {
                //【*3.1.1】回调接口！
                mLegacy.packageDeleted(basePackageName, returnCode);
            } catch (RemoteException ignored) {
            }
        }
    }
```
LegacyPackageDeleteObserver 实际上是对前面的回调的一次封装！



