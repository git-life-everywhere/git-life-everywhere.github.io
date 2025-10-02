# Permission第 3 篇 - checkPermissions 权限检查
title: Permission第 3 篇 - checkPermissions 权限检查
date: 2017/07/06 20:46:25
catalog: true
categories:
- AndroidFramework源码分析
- Permission权限管理
tags: Permission权限管理
---

[toc]

# 0 综述

基于 Android 7.1.1 源码，分析系统的权限管理机制！

下面来看下系统中的 check permission 相关的接口：

# 1 Context 权限检查接口 - ContextImpl

在应用中申请权限离不开 Context，Context 中暴露了多个检查权限相关的接口，具体的实现是在 ContextImpl 中！

## 1.1 checkCallingPermission

检查来自其他进程的调用者的是否有权限，该接口不能用于检查自身是否具有权限！

```java
    @Override
    public int checkCallingPermission(String permission) {
        if (permission == null) {
            throw new IllegalArgumentException("permission is null");
        }

        int pid = Binder.getCallingPid();
        if (pid != Process.myPid()) {
            //【*1.3】调用自身的 checkPermission 方法！
            return checkPermission(permission, pid, Binder.getCallingUid());
        }
        return PackageManager.PERMISSION_DENIED;
    }
```

应用程序权限检查，最终需要进入框架层！

## 1.2 checkCallingOrSelfPermission

该方法和 checkCallingPermission 不同，他还可以检查自身是否具有权限！
```java
    @Override
    public int checkCallingOrSelfPermission(String permission) {
        if (permission == null) {
            throw new IllegalArgumentException("permission is null");
        }
        //【*1.3】调用自身的 checkPermission 方法！
        return checkPermission(permission, Binder.getCallingPid(),
                Binder.getCallingUid());
    }
```

## 1.3 checkPermission

- **checkPermission[3]**

该方法需要指定 pid 和 uid：

```java
    @Override
    public int checkPermission(String permission, int pid, int uid) {
        if (permission == null) {
            throw new IllegalArgumentException("permission is null");
        }

        try {
            //【*2.2】调用了 ams 的 checkPermission 接口
            return ActivityManagerNative.getDefault().checkPermission(
                    permission, pid, uid);
        } catch (RemoteException e) {
            return PackageManager.PERMISSION_DENIED;
        }
    }
```

- **checkPermission[4]**

还有一个重载函数，需要额外传入一个 callerToken 实例，这里是一个 hide 方法，目前是隐藏的，只能由系统调用：

```java
    /** @hide */
    @Override
    public int checkPermission(String permission, int pid, int uid, IBinder callerToken) {
        if (permission == null) {
            throw new IllegalArgumentException("permission is null");
        }

        try {
            //【*2.10】调用了 ams 的 checkPermissionWithToken 接口
            return ActivityManagerNative.getDefault().checkPermissionWithToken(
                    permission, pid, uid, callerToken);
        } catch (RemoteException e) {
            return PackageManager.PERMISSION_DENIED;
        }
    }
```

## 1.4 Check Uri Permissions

ContextImpl 还提供了检查 Uri Permissions 的接口，这些接口依赖于一个 modeFlags 标志位，取值有如下：

```java
Intent.FLAG_GRANT_READ_URI_PERMISSION
Intent.FLAG_GRANT_WRITE_URI_PERMISSION
```
下面我们来看下相关的接口的逻辑：

### 1.4.1 checkCallingUriPermission

同样的，这个方法只能检查其他进程是否具有访问 uri 权限！

```java
    @Override
    public int checkCallingUriPermission(Uri uri, int modeFlags) {
        int pid = Binder.getCallingPid();
        if (pid != Process.myPid()) {
            //【*1.4.3】调用 ContextImpl 的 checkUriPermission 方法！
            return checkUriPermission(uri, pid,
                    Binder.getCallingUid(), modeFlags);
        }
        return PackageManager.PERMISSION_DENIED;
    }
```
### 1.4.2 checkCallingOrSelfUriPermission

同样的，这个方法能检查其他进程或者自身是否具有访问 uri 权限！

```java
    @Override
    public int checkCallingOrSelfUriPermission(Uri uri, int modeFlags) {
        //【*1.4.3】调用 ContextImpl 的 checkUriPermission 方法！
        return checkUriPermission(uri, Binder.getCallingPid(),
                Binder.getCallingUid(), modeFlags);
    }
```

### 1.4.3 checkUriPermission

checkUriPermission 方法一共有三个重载实现：

- **checkUriPermission[6]**

```java
    @Override
    public int checkUriPermission(Uri uri, String readPermission,
            String writePermission, int pid, int uid, int modeFlags) {
        if (DEBUG) {
            Log.i("foo", "checkUriPermission: uri=" + uri + "readPermission="
                    + readPermission + " writePermission=" + writePermission
                    + " pid=" + pid + " uid=" + uid + " mode" + modeFlags);
        }
        //【1】如果 flags 设置了 FLAG_GRANT_READ_URI_PERMISSION 标志位，那么
        // 如果 readPermission 不为 null，且 pid uid 是有这个 readPermission 权限的；
        // 那就返回 PERMISSION_GRANTED
        if ((modeFlags & Intent.FLAG_GRANT_READ_URI_PERMISSION) != 0) {
            if (readPermission == null
                    || checkPermission(readPermission, pid, uid)
                    == PackageManager.PERMISSION_GRANTED) {
                return PackageManager.PERMISSION_GRANTED;
            }
        }
        //【2】如果 flags 设置了 FLAG_GRANT_WRITE_URI_PERMISSION 标志位，那么
        // 如果 writePermission 不为 null，且 pid uid 是有这个 writePermission 权限的；
        // 那就返回 PERMISSION_GRANTED
        if ((modeFlags & Intent.FLAG_GRANT_WRITE_URI_PERMISSION) != 0) {
            if (writePermission == null
                    || checkPermission(writePermission, pid, uid)
                    == PackageManager.PERMISSION_GRANTED) {
                return PackageManager.PERMISSION_GRANTED;
            }
        }
        //【*self-other】上述条件不满足的话，会进入到这部分中，调用另外一个重载的函数：
        return uri != null ? checkUriPermission(uri, pid, uid, modeFlags)
                : PackageManager.PERMISSION_DENIED;
    }
```
- **checkUriPermission[4]**

第二个 checkUriPermission 会被第一个方法调用：

```java
    @Override
    public int checkUriPermission(Uri uri, int pid, int uid, int modeFlags) {
        try {
            //【*2.9】调用 ams 的 checkUriPermission 方法！
            return ActivityManagerNative.getDefault().checkUriPermission(
                    ContentProvider.getUriWithoutUserId(uri), pid, uid, modeFlags,
                    resolveUserId(uri), null);
        } catch (RemoteException e) {
            return PackageManager.PERMISSION_DENIED;
        }
    }
```

- **checkUriPermission[5]**

第三个 checkUriPermission 方法，需要传入一个 IBinder token 对象，这是一个 hide 方法，只能由系统调用！

```java
    /** @hide */
    @Override
    public int checkUriPermission(Uri uri, int pid, int uid, int modeFlags, IBinder callerToken) {
        try {
            //【*2.9】调用 ams 的 checkUriPermission 方法！
            return ActivityManagerNative.getDefault().checkUriPermission(
                    ContentProvider.getUriWithoutUserId(uri), pid, uid, modeFlags,
                    resolveUserId(uri), callerToken);
        } catch (RemoteException e) {
            return PackageManager.PERMISSION_DENIED;
        }
    }
```


# 2 框架层权限检查接口 - ActivityManagerService

我们来看看 Android 7.1.1 中框架层提供了那些用于校验权限的接口，这些接口主要位于 ActivityManager，ActivityManagerService 中，我们做一个简单的归类！


- 访问 ContentProvider 的权限校验；

```java
String checkContentProviderPermissionLocked(ProviderInfo cpi, ProcessRecord r, int userId, boolean checkUser)
```

- 判断 pid，uid 对应的进程是否具有某个权限；

```java
 // 检测调用者的权限，第一个方法会调用第二个方法！
int checkCallingPermission(String permission) {...}
int checkPermission(String permission, int pid, int uid) {...}
```

- 检测组件的权限；

```java
 // 检测组件的权限
int checkComponentPermission(String permission, int pid, int uid, int owningUid, boolean exported)  {...}
```

- 检测 Uri 相关的权限；

```java
int checkGrantUriPermission(int callingUid, String targetPkg, Uri uri, 
                                                    final int modeFlags, int userId)  {...}

NeededUriGrants checkGrantUriPermissionFromIntentLocked(int callingUid, String targetPkg, Intent intent,
                                                    int mode, NeededUriGrants needed, int targetUserId)  {...}
            
int checkGrantUriPermissionLocked(int callingUid, String targetPkg, GrantUri grantUri, 
                                                    final int modeFlags, int lastTargetUid)  {...}

int checkUriPermission(Uri uri, int pid, int uid, final int modeFlags, int userId, IBinder callerToken) {...}
            
boolean checkUriPermissionLocked(GrantUri grantUri, int uid, final int modeFlags) {...}

boolean checkHoldingPermissionsLocked(IPackageManager pm, ProviderInfo pi, GrantUri grantUri,
                                                                    int uid, final int modeFlags)  {...}
                                
boolean checkHoldingPermissionsInternalLocked(IPackageManager pm, ProviderInfo pi, GrantUri grantUri,
                                        int uid, final int modeFlags, boolean considerUidPermissions)  {...}
```

- 检测 Binder 对象的权限；

```java
int checkPermissionWithToken(String permission, int pid, int uid, IBinder callerToken)  {...}
```

同时，系统也提供了一些强制检查权限的接口，以 enforce 开头，我们在很多的系统调用过程中都会看到，这些接口会调用上面的接口来查询权限是否授予，没有授予会抛出异常，这里就不再列举！

## 2.1 checkCallingPermission

该方法用于判断调用者是否具有指定的权限，内部会自动通过 Binder 相关接口获得调用者的 pid 和 uid！！
```java
    int checkCallingPermission(String permission) {
        //【*2.2】这里调用了 checkPermission 方法！
        return checkPermission(permission,
                Binder.getCallingPid(),
                UserHandle.getAppId(Binder.getCallingUid()));
    }
```
该方法内部会通过 Binder.getCallingPid() 和 Binder.getCallingUid() 方法，计算 pid 和 uid！

### 2.1.1 调用时机

checkCallingPermission 方法系统中调用的很多，一些重要服务都会调用它来检查权限，比如 InputManagerService，WindowManagerService，NotificationManagerService，DisplayManagerService 等等，下面只简单列出来几个：

#### 2.1.1.1 NotificationManagerService.enforcePolicyAccess

```java
    private void enforcePolicyAccess(String pkg, String method) {
        //【1】检查调用者是否有 MANAGE_NOTIFICATIONS 权限
        if (PackageManager.PERMISSION_GRANTED == getContext().checkCallingPermission(
                android.Manifest.permission.MANAGE_NOTIFICATIONS)) {
            return;
        }
        checkCallerIsSameApp(pkg);
        if (!checkPolicyAccess(pkg)) {
            Slog.w(TAG, "Notification policy access denied calling " + method);
            throw new SecurityException("Notification policy access denied");
        }
    }
```
NotificationManagerService 有很多方法设计到了权限校验，这里只是举例展示。

#### 2.1.1.2 InputManagerService.addKeyboardLayoutForInputDevice

```java
    @Override // Binder call
    public void addKeyboardLayoutForInputDevice(InputDeviceIdentifier identifier,
            String keyboardLayoutDescriptor) {
        //【1】检查调用者是否有 SET_KEYBOARD_LAYOUT 权限
        if (!checkCallingPermission(android.Manifest.permission.SET_KEYBOARD_LAYOUT,
                "addKeyboardLayoutForInputDevice()")) {
            throw new SecurityException("Requires SET_KEYBOARD_LAYOUT permission");
        }
        ... ... ... ...
    }
```
InputManagerService 有很多方法设计到了权限校验，这里只是举例展示。

#### 2.1.1.3 WindowManagerService.startAppFreezingScreen

```java
    @Override
    public void startAppFreezingScreen(IBinder token, int configChanges) {
        //【1】检查调用者是否有 MANAGE_APP_TOKENS 权限
        if (!checkCallingPermission(android.Manifest.permission.MANAGE_APP_TOKENS,
                "setAppFreezingScreen()")) {
            throw new SecurityException("Requires MANAGE_APP_TOKENS permission");
        }

        synchronized(mWindowMap) {
            if (configChanges == 0 && okToDisplay()) {
                if (DEBUG_ORIENTATION) Slog.v(TAG_WM, "Skipping set freeze of " + token);
                return;
            }

            AppWindowToken wtoken = findAppWindowToken(token);
            if (wtoken == null || wtoken.appToken == null) {
                Slog.w(TAG_WM, "Attempted to freeze screen with non-existing app token: " + wtoken);
                return;
            }
            final long origId = Binder.clearCallingIdentity();
            startAppFreezingScreenLocked(wtoken);
            Binder.restoreCallingIdentity(origId);
        }
    }
```
WindowManagerService 有很多方法设计到了权限校验，这里只是举例展示。



## 2.2 checkPermission

该方法用于判断 pid uid 是否有具体的权限，调用这个方法必须要知道对应的 pid 和 uid：

```java
    @Override
    public int checkPermission(String permission, int pid, int uid) {
        if (permission == null) {
            return PackageManager.PERMISSION_DENIED;
        }
        //【*2.12】调用了 checkComponentPermission 方法！
        return checkComponentPermission(permission, pid, uid, -1, true);
    }
```

### 2.2.1 调用时机

checkPermission 方法的调用也很多，所以简单的看几个：

#### 2.2.1.1 ActivityManagerService.isGetTasksAllowed

```java
    private boolean isGetTasksAllowed(String caller, int callingPid, int callingUid) {
        //【1】检查调用者是否具有 REAL_GET_TASKS 和 GET_TASKS 权限！
        boolean allowed = checkPermission(android.Manifest.permission.REAL_GET_TASKS,
                callingPid, callingUid) == PackageManager.PERMISSION_GRANTED;
        if (!allowed) {
            if (checkPermission(android.Manifest.permission.GET_TASKS,
                    callingPid, callingUid) == PackageManager.PERMISSION_GRANTED) {
                try {
                    if (AppGlobals.getPackageManager().isUidPrivileged(callingUid)) {
                        allowed = true;
                        if (DEBUG_TASKS) Slog.w(TAG, caller + ": caller " + callingUid
                                + " is using old GET_TASKS but privileged; allowing");
                    }
                } catch (RemoteException e) {
                }
            }
        }
        if (!allowed) {
            if (DEBUG_TASKS) Slog.w(TAG, caller + ": caller " + callingUid
                    + " does not hold REAL_GET_TASKS; limiting output");
        }
        return allowed;
    }
```

### 2.2.2 跨进程调用

调用方法：
```java
try {
    ActivityManagerNative.getDefault().checkPermission(...);
} catch (RemoteException e) {
    throw e.rethrowFromSystemServer();
}
```
具体流程：

- **ActivityManagerProxy**

```java
    public int checkPermission(String permission, int pid, int uid)
            throws RemoteException {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IActivityManager.descriptor);
        data.writeString(permission);
        data.writeInt(pid);
        data.writeInt(uid);
        mRemote.transact(CHECK_PERMISSION_TRANSACTION, data, reply, 0);
        reply.readException();
        int res = reply.readInt();
        data.recycle();
        reply.recycle();
        return res;
    }

```

- **ActivityManagerNative**

```java
        case CHECK_PERMISSION_TRANSACTION: {
            data.enforceInterface(IActivityManager.descriptor);
            String perm = data.readString();
            int pid = data.readInt();
            int uid = data.readInt();
            int res = checkPermission(perm, pid, uid);
            reply.writeNoException();
            reply.writeInt(res);
            return true;
        }
```


## 2.3 checkContentProviderPermissionLocked

checkContentProviderPermissionLocked 方法用于判断：

- 进程 ProcessRecord 是否可以访问目标 ContentProvider 对象，该方法返回 null 表示该进程是有这个权限的！

```java
    private final String checkContentProviderPermissionLocked(
            ProviderInfo cpi, ProcessRecord r, int userId, boolean checkUser) {
        //【1】获得进程的 pid 和 uid！
        final int callingPid = (r != null) ? r.pid : Binder.getCallingPid();
        final int callingUid = (r != null) ? r.uid : Binder.getCallingUid();
        boolean checkedGrants = false;

        //【2】如果 checkUser 为 true 的话，会进入下面的分支，检查 userId
        if (checkUser) {
            // Looking for cross-user grants before enforcing the typical cross-users permissions
            int tmpTargetUserId = mUserController.unsafeConvertIncomingUserLocked(userId);
            if (tmpTargetUserId != UserHandle.getUserId(callingUid)) {
                if (checkAuthorityGrants(callingUid, cpi, tmpTargetUserId, checkUser)) {
                    return null;
                }
                checkedGrants = true;
            }
            userId = mUserController.handleIncomingUser(callingPid, callingUid, userId, false,
                    ALLOW_NON_FULL, "checkContentProviderPermissionLocked " + cpi.authority, null);
            if (userId != tmpTargetUserId) {
                // When we actually went to determine the final targer user ID, this ended
                // up different than our initial check for the authority.  This is because
                // they had asked for USER_CURRENT_OR_SELF and we ended up switching to
                // SELF.  So we need to re-check the grants again.
                checkedGrants = false;
            }
        }
        //【*2.12】检查访问进程是否具有读权限！
        if (checkComponentPermission(cpi.readPermission, callingPid, callingUid,
                cpi.applicationInfo.uid, cpi.exported)
                == PackageManager.PERMISSION_GRANTED) {
            return null;
        }
        //【*2.12】检查访问进程是否具有写权限！
        if (checkComponentPermission(cpi.writePermission, callingPid, callingUid,
                cpi.applicationInfo.uid, cpi.exported)
                == PackageManager.PERMISSION_GRANTED) {
            return null;
        }
        //【3】检查访问进程是否具有 pathPermissions，每个 path 都可以指定 read 和 write 权限！
        PathPermission[] pps = cpi.pathPermissions;
        if (pps != null) {
            int i = pps.length;
            while (i > 0) {
                i--;
                PathPermission pp = pps[i];
                //【*2.12】检查访问进程是否具有 path 读权限！
                String pprperm = pp.getReadPermission();
                if (pprperm != null && checkComponentPermission(pprperm, callingPid, callingUid,
                        cpi.applicationInfo.uid, cpi.exported)
                        == PackageManager.PERMISSION_GRANTED) {
                    return null;
                }
                //【*2.12】检查访问进程是否具有 path 写权限！
                String ppwperm = pp.getWritePermission();
                if (ppwperm != null && checkComponentPermission(ppwperm, callingPid, callingUid,
                        cpi.applicationInfo.uid, cpi.exported)
                        == PackageManager.PERMISSION_GRANTED) {
                    return null;
                }
            }
        }
        if (!checkedGrants && checkAuthorityGrants(callingUid, cpi, userId, checkUser)) {
            return null;
        }

        String msg;
        //【4】最后校验该 ContentProvider 是否是 exported 的！
        if (!cpi.exported) {
            msg = "Permission Denial: opening provider " + cpi.name
                    + " from " + (r != null ? r : "(null)") + " (pid=" + callingPid
                    + ", uid=" + callingUid + ") that is not exported from uid "
                    + cpi.applicationInfo.uid;
        } else {
            msg = "Permission Denial: opening provider " + cpi.name
                    + " from " + (r != null ? r : "(null)") + " (pid=" + callingPid
                    + ", uid=" + callingUid + ") requires "
                    + cpi.readPermission + " or " + cpi.writePermission;
        }
        Slog.w(TAG, msg);
        return msg;
    }
```

我们看到了这里调用了 checkComponentPermission 方法，判断访问者对该 ContentProvider 是否具有读或写的权限！


### 2.3.1 调用时机

#### 2.3.1.1 ActivityManagerService.getContentProviderImpl

该方法主要是在访问 provider 的时候做权限校验的，调用的方法在 getContentProviderImpl 方法;

```java
    private ContentProviderHolder getContentProviderImpl(IApplicationThread caller,
            String name, IBinder token, boolean stable, int userId) {
        ContentProviderRecord cpr;
        ContentProviderConnection conn = null;
        ProviderInfo cpi = null;

        synchronized(this) {
            ... ... ...
            boolean providerRunning = cpr != null && cpr.proc != null && !cpr.proc.killed;
            if (providerRunning) {
                cpi = cpr.info;
                String msg;
                checkTime(startTime, "getContentProviderImpl: before checkContentProviderPermission");
                //【*2.3】校验是否有访问 provider 的权限；
                if ((msg = checkContentProviderPermissionLocked(cpi, r, userId, checkCrossUser))
                        != null) {
                    throw new SecurityException(msg);
                }
                checkTime(startTime, "getContentProviderImpl: after checkContentProviderPermission");
                ... ... ... ...
            }

            if (!providerRunning) {
                ... ... ... ...
                String msg;
                checkTime(startTime, "getContentProviderImpl: before checkContentProviderPermission");
                //【*2.3】同上；
                if ((msg = checkContentProviderPermissionLocked(cpi, r, userId, !singleton))
                        != null) {
                    throw new SecurityException(msg);
                }
                checkTime(startTime, "getContentProviderImpl: after checkContentProviderPermission");
                ... ... ... ...
            }
            checkTime(startTime, "getContentProviderImpl: done!");
        }
        ... ... ... ...
        return cpr != null ? cpr.newHolder(conn) : null;
    }
```
getContentProviderImpl 方法我就不多说了，之前分析总结过！

## 2.4 checkGrantUriPermission

判断 callingUid 是否能够授予 targetPkg 访问 uri 的权限，modeFlags 指定了访问模式！

```java
    public int checkGrantUriPermission(int callingUid, String targetPkg, Uri uri,
            final int modeFlags, int userId) {
        enforceNotIsolatedCaller("checkGrantUriPermission");
        synchronized(this) {
            //【*2.6】这里调用了 checkGrantUriPermissionLocked 方法！
            return checkGrantUriPermissionLocked(callingUid, targetPkg,
                    new GrantUri(userId, uri, false), modeFlags, -1);
        }
    }
```
该方法会对参数做一个简单的封装，然后调用 checkGrantUriPermissionLocked 继续处理！

### 2.4.1 调用时机

#### 2.4.1.1 Clipboard.setPrimaryClip

这里我们通过一个简单的例子来看下这个方法的使用，Android 系统有一个 Clipboard，当我们机型拷贝的时候会调用其 setPrimaryClip 方法：

```java
    public void setPrimaryClip(ClipData clip, String callingPackage) {
        synchronized (this) {
            if (clip != null && clip.getItemCount() <= 0) {
               throw new IllegalArgumentException("No items");
            }
            final int callingUid = Binder.getCallingUid();
            //【1】当然这边是对调用者是否有 write clipboard 的 op 操作进行检查和记录！
            if (mAppOps.noteOp(AppOpsManager.OP_WRITE_CLIPBOARD, callingUid,
                    callingPackage) != AppOpsManager.MODE_ALLOWED) {
                return;
            }
            //【2】检查调用者是否是 data 的所有者！
            checkDataOwnerLocked(clip, callingUid);
            ... ... ...
```
调用了 checkDataOwnerLocked 方法：
```java
    private final void checkDataOwnerLocked(ClipData data, int uid) {
        final int N = data.getItemCount();
        for (int i=0; i<N; i++) {
            //【1】对所有的 data 进行检查！ 
            checkItemOwnerLocked(data.getItemAt(i), uid);
        }
    }
```
调用了 checkItemOwnerLocked 方法：
```java
    private final void checkUriOwnerLocked(Uri uri, int uid) {
        //【1】如果 uri 的 scheme 不是 content 的话，不会检查；
        if (!"content".equals(uri.getScheme())) {
            return;
        }
        long ident = Binder.clearCallingIdentity();
        try {
            //【2】校验下 uid 是否有权限访问 uri！
            mAm.checkGrantUriPermission(uid, null, ContentProvider.getUriWithoutUserId(uri),
                    Intent.FLAG_GRANT_READ_URI_PERMISSION,
                    ContentProvider.getUserIdFromUri(uri, UserHandle.getUserId(uid)));
        } catch (RemoteException e) {
        } finally {
            Binder.restoreCallingIdentity(ident);
        }
    }
```
这个调用流程很清晰！

#### 2.4.1.2 VoiceInteractionSessionConnection.deliverSessionDataLocked

还有一个是语音识别服务，需要传递数据的时候：

```java
    private void deliverSessionDataLocked(AssistDataForActivity assistDataForActivity) {
        Bundle assistData = assistDataForActivity.data.getBundle(
                VoiceInteractionSession.KEY_DATA);
        AssistStructure structure = assistDataForActivity.data.getParcelable(
                VoiceInteractionSession.KEY_STRUCTURE);
        AssistContent content = assistDataForActivity.data.getParcelable(
                VoiceInteractionSession.KEY_CONTENT);
        //【1】获得目标 uid！
        int uid = assistDataForActivity.data.getInt(Intent.EXTRA_ASSIST_UID, -1);
        if (uid >= 0 && content != null) {
            //【1.1】获得要传递的 intent，遍历处理内部的 ClipData 数据！
            Intent intent = content.getIntent();
            if (intent != null) {
                ClipData data = intent.getClipData();
                if (data != null && Intent.isAccessUriMode(intent.getFlags())) {
                    //【1.1.1】由于目标 uid 需要访问数据，所以这里会尝试授予 uid 对应的 uri 权限！
                    grantClipDataPermissions(data, intent.getFlags(), uid,
                            mCallingUid, mSessionComponentName.getPackageName());
                }
            }
            //【1.2】同理！
            ClipData data = content.getClipData();
            if (data != null) {
                grantClipDataPermissions(data,
                        Intent.FLAG_GRANT_READ_URI_PERMISSION,
                        uid, mCallingUid, mSessionComponentName.getPackageName());
            }
        }
        try {
            //【2】处理语音相关事务！
            mSession.handleAssist(assistData, structure, content,
                    assistDataForActivity.activityIndex, assistDataForActivity.activityCount);
        } catch (RemoteException e) {
        }
    }
```
调用 grantClipDataPermissions 方法，这里的 srcUid 是 EXTRA_ASSIST_UID 对应的 uid！！
```java
    void grantClipDataPermissions(ClipData data, int mode, int srcUid, int destUid,
            String destPkg) {
        final int N = data.getItemCount();
        for (int i=0; i<N; i++) {
            //【1】继续调用：
            grantClipDataItemPermission(data.getItemAt(i), mode, srcUid, destUid, destPkg);
        }
    }
```
再调用 grantClipDataItemPermission 方法：
```java
    void grantClipDataItemPermission(ClipData.Item item, int mode, int srcUid, int destUid,
            String destPkg) {
        if (item.getUri() != null) {
            //【1】继续调用，授予 uid 对应的 uri 权限！
            grantUriPermission(item.getUri(), mode, srcUid, destUid, destPkg);
        }
        Intent intent = item.getIntent();
        if (intent != null && intent.getData() != null) {
            grantUriPermission(intent.getData(), mode, srcUid, destUid, destPkg);
        }
    }
```
继续调用 grantUriPermission 方法：
```java
    void grantUriPermission(Uri uri, int mode, int srcUid, int destUid, String destPkg) {
        if (!"content".equals(uri.getScheme())) {
            return;
        }
        long ident = Binder.clearCallingIdentity();
        try {
            //【1】做一次检查，如果有异常，那么终止操作！
            mAm.checkGrantUriPermission(srcUid, null, ContentProvider.getUriWithoutUserId(uri),
                    mode, ContentProvider.getUserIdFromUri(uri, UserHandle.getUserId(srcUid)));
            // No security exception, do the grant.
            int sourceUserId = ContentProvider.getUserIdFromUri(uri, mUser);
            uri = ContentProvider.getUriWithoutUserId(uri);
            //【2】授予 srcUid 访问 uri 的权限！
            mAm.grantUriPermissionFromOwner(mPermissionOwner, srcUid, destPkg,
                    uri, Intent.FLAG_GRANT_READ_URI_PERMISSION, sourceUserId, mUser);
        } catch (RemoteException e) {
        } catch (SecurityException e) {
            Slog.w(TAG, "Can't propagate permission", e);
        } finally {
            Binder.restoreCallingIdentity(ident);
        }
    }
```
不多说了！！

### 2.4.2 跨进程调用

调用方法：
```java
try {
    ActivityManagerNative.getDefault().checkGrantUriPermission(...);
} catch (RemoteException e) {
    throw e.rethrowFromSystemServer();
}
```
具体流程：

- **ActivityManagerProxy**

```java
    public int checkGrantUriPermission(int callingUid, String targetPkg,
            Uri uri, int modeFlags, int userId) throws RemoteException {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IActivityManager.descriptor);
        data.writeInt(callingUid);
        data.writeString(targetPkg);
        uri.writeToParcel(data, 0);
        data.writeInt(modeFlags);
        data.writeInt(userId);
        mRemote.transact(CHECK_GRANT_URI_PERMISSION_TRANSACTION, data, reply, 0);
        reply.readException();
        int res = reply.readInt();
        data.recycle();
        reply.recycle();
        return res;
    }
```

- **ActivityManagerNative**

```java
        case CHECK_GRANT_URI_PERMISSION_TRANSACTION: {
            data.enforceInterface(IActivityManager.descriptor);
            int callingUid = data.readInt();
            String targetPkg = data.readString();
            Uri uri = Uri.CREATOR.createFromParcel(data);
            int modeFlags = data.readInt();
            int userId = data.readInt();
            int res = checkGrantUriPermission(callingUid, targetPkg, uri, modeFlags, userId);
            reply.writeNoException();
            reply.writeInt(res);
            return true;
        }
```


## 2.5 checkGrantUriPermissionFromIntentLocked

该方法用于判断是否能够授予 targetPkg 访问 uri 的权限，uri 来自接收到的 intent；

和 2.6 的 checkGrantUriPermissionLocked 很类似，区别是 checkGrantUriPermissionLocked 直接操作的是 Uri；而该方法直接作用于 Intent！

返回 null 表示不需要授予权限！

```java
    NeededUriGrants checkGrantUriPermissionFromIntentLocked(int callingUid,
            String targetPkg, Intent intent, int mode, NeededUriGrants needed, int targetUserId) {
        if (DEBUG_URI_PERMISSION) Slog.v(TAG_URI_PERMISSION,
                "Checking URI perm to data=" + (intent != null ? intent.getData() : null)
                + " clip=" + (intent != null ? intent.getClipData() : null)
                + " from " + intent + "; flags=0x"
                + Integer.toHexString(intent != null ? intent.getFlags() : 0));

        //【1】如果 targetPkg / intent 为 null，则抛出异常！
        if (targetPkg == null) {
            throw new NullPointerException("targetPkg");
        }
        if (intent == null) {
            return null;
        }
        //【2】获得 intent 中的 Uri 和 ClipData 数据，如果都为 null，抛出异常！
        Uri data = intent.getData();
        ClipData clip = intent.getClipData();
        if (data == null && clip == null) {
            return null;
        }

        //【3】获得 uri 所在的 userId，如果 contentUserHint 等于 UserHandle.USER_CURRENT，
        // 说明该 intent 没有离开当前 userId！
        int contentUserHint = intent.getContentUserHint();
        if (contentUserHint == UserHandle.USER_CURRENT) {
            contentUserHint = UserHandle.getUserId(callingUid);
        }
        //【4】计算目标 uid，如果已经知道需要什么权限，即 NeededUriGrants 不为 null，那么通过 NeededUriGrants.targetUid
        // 获得目标 uid，否则，通过 targetPkg 计算 targetUid！
        final IPackageManager pm = AppGlobals.getPackageManager();
        int targetUid;
        if (needed != null) {
            targetUid = needed.targetUid;
        } else {
            try {
                targetUid = pm.getPackageUid(targetPkg, MATCH_DEBUG_TRIAGED_MISSING,
                        targetUserId);
            } catch (RemoteException ex) {
                return null;
            }
            if (targetUid < 0) {
                if (DEBUG_URI_PERMISSION) Slog.v(TAG_URI_PERMISSION,
                        "Can't grant URI permission no uid for: " + targetPkg
                        + " on user " + targetUserId);
                return null;
            }
        }
        //【5】首先处理 data 数据，获得传递的 uri，调用 checkGrantUriPermissionLocked 判断是否需要授予 uri 权限！
        if (data != null) {
            GrantUri grantUri = GrantUri.resolve(contentUserHint, data);
            //【*2.6】首先判断是否有访问 data 指定的数据！
            targetUid = checkGrantUriPermissionLocked(callingUid, targetPkg, grantUri, mode,
                    targetUid);
            //【5.1】如果返回值大于 0，说明需要授予权限，创建 NeededUriGrants 对象，记录下！
            if (targetUid > 0) {
                if (needed == null) {
                    needed = new NeededUriGrants(targetPkg, targetUid, mode);
                }
                needed.add(grantUri);
            }
        }
        //【6】接着处理 ClipData 数据，遍历 ClipData 中的所有 item，处理其 uri 和 intent 数据！
        if (clip != null) {
            for (int i=0; i<clip.getItemCount(); i++) {
                Uri uri = clip.getItemAt(i).getUri();
                if (uri != null) {
                    //【*2.6】如果其 uri 不为 null，检查是否有权限访问该 url，同样是 checkGrantUriPermissionLocked
                    // 方法！
                    GrantUri grantUri = GrantUri.resolve(contentUserHint, uri);
                    targetUid = checkGrantUriPermissionLocked(callingUid, targetPkg, grantUri, mode,
                            targetUid);
                    if (targetUid > 0) {
                        if (needed == null) {
                            needed = new NeededUriGrants(targetPkg, targetUid, mode);
                        }
                        needed.add(grantUri);
                    }
                } else {
                    //【6.1】如果其 clipIntent 不为 null，递归检查该 intent!
                    Intent clipIntent = clip.getItemAt(i).getIntent();
                    if (clipIntent != null) {
                        //【*2.6】递归检查！ 
                        NeededUriGrants newNeeded = checkGrantUriPermissionFromIntentLocked(
                                callingUid, targetPkg, clipIntent, mode, needed, targetUserId);
                        if (newNeeded != null) {
                            needed = newNeeded;
                        }
                    }
                }
            }
        }
        //【7】返回 NeededUriGrants 对象，该对象保存了需要授权访问的所有 url！！
        return needed;
    }
```
该方法会返回一个 NeededUriGrants 对象，用于保存该 intent 携带的所有需要授权访问了 GrantUri 对象，包括其 ClipData 中的 intent 携带的 uri：

```java
    static class NeededUriGrants extends ArrayList<GrantUri> {
        final String targetPkg;
        final int targetUid;
        final int flags;

        NeededUriGrants(String targetPkg, int targetUid, int flags) {
            this.targetPkg = targetPkg;
            this.targetUid = targetUid;
            this.flags = flags;
        }
    }
```
这里的 mode 可以设置下面的两个标志位！

```java
    public static final int FLAG_GRANT_READ_URI_PERMISSION = 0x00000001;

    public static final int FLAG_GRANT_WRITE_URI_PERMISSION = 0x00000002;
```
如果设置该标记，Intent 的接受者将会被赋予读/写 Intent 中 uri 数据和 Intent.ClipData 中的 uri 的权限。

当应用到 Intent 的 ClipData 时，其携带的所有 uri 都会被 intent 接收者读/写，同时其会递归处理其携带的所有 intent！

其实，mode 的值是为了标志授予的权限类型是读还是写的，都设置的话，那就是可读可写！

### 2.5.1 调用时机

该方法的调用实际比较少，ActiveServices 和 ActivityManagerService 中：

#### 2.5.1.1 ActiveServices.startServiceLocked

```java
    ComponentName startServiceLocked(IApplicationThread caller, Intent service, String resolvedType,
            int callingPid, int callingUid, String callingPackage, final int userId)
            throws TransactionTooLargeException {
        if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE, "startService: " + service
                + " type=" + resolvedType + " args=" + service.getExtras());
        ... ... ... ...

        if (!r.startRequested) {
            final long token = Binder.clearCallingIdentity();
            try {
                // Before going further -- if this app is not allowed to run in the
                // background, then at this point we aren't going to let it period.
                final int allowed = mAm.checkAllowBackgroundLocked(
                        r.appInfo.uid, r.packageName, callingPid, true);
                if (allowed != ActivityManager.APP_START_MODE_NORMAL) {
                    Slog.w(TAG, "Background start not allowed: service "
                            + service + " to " + r.name.flattenToShortString()
                            + " from pid=" + callingPid + " uid=" + callingUid
                            + " pkg=" + callingPackage);
                    return null;
                }
            } finally {f
                Binder.restoreCallingIdentity(token);
            }
        }
        //【1】校验了下是否需要授予 uri 权限！
        NeededUriGrants neededGrants = mAm.checkGrantUriPermissionFromIntentLocked(
                callingUid, r.packageName, service, service.getFlags(), null, r.userId);

        ... ... ... ...

        return startServiceInnerLocked(smap, service, r, callerFg, addToStarting);
    }
```
这里就不多说了！

## 2.6 checkGrantUriPermissionLocked

判断是否需要授予 targetPkg 访问 uri 的权限，modeFlags 指定了访问模式， callingUid 是调用者 uid！

如果 callingUid 不被允许，会抛出 SecurityException 异常！ 如果方法返回了 targetPkg 的 uid，说明需要授予权限；如果返回 -1，表示不需要或者无法授予（比如目标包已经有权限访问 uri）！

lastTargetUid 表示 targetPkg 的 uid，如果已经知道了 targetPkg.uid，那么就设置 lastTargetUid 为 targetPkg.uid 即可；如果不知道，那么设置为 -1，方法内部会计算出对应的 uid！

```java
    int checkGrantUriPermissionLocked(int callingUid, String targetPkg, GrantUri grantUri,
            final int modeFlags, int lastTargetUid) {
        //【1】判断 modeFlags 是否至少设置了 Intent.FLAG_GRANT_WRITE_URI_PERMISSION 或者
        // Intent.FLAG_GRANT_READ_URI_PERMISSION，如果都没有，返回 -1!
        if (!Intent.isAccessUriMode(modeFlags)) {
            return -1;
        }

        if (targetPkg != null) {
            if (DEBUG_URI_PERMISSION) Slog.v(TAG_URI_PERMISSION,
                    "Checking grant " + targetPkg + " permission to " + grantUri);
        }

        final IPackageManager pm = AppGlobals.getPackageManager();

        //【2】如果 uri 不是和 content 相关的，异常，返回 -1；
        if (!ContentResolver.SCHEME_CONTENT.equals(grantUri.uri.getScheme())) {
            if (DEBUG_URI_PERMISSION) Slog.v(TAG_URI_PERMISSION,
                    "Can't grant URI permission for non-content URI: " + grantUri);
            return -1;
        }

        //【3】无法通过 uri 找到一个 ContentProvider，异常，返回 -1；
        final String authority = grantUri.uri.getAuthority();
        final ProviderInfo pi = getProviderInfoLocked(authority, grantUri.sourceUserId,
                MATCH_DEBUG_TRIAGED_MISSING);
        if (pi == null) {
            Slog.w(TAG, "No content provider found for permission check: " +
                    grantUri.uri.toSafeString());
            return -1;
        }

        //【4】计算 targetPkg 的 uid！
        int targetUid = lastTargetUid;
        if (targetUid < 0 && targetPkg != null) {
            try {
                targetUid = pm.getPackageUid(targetPkg, MATCH_DEBUG_TRIAGED_MISSING,
                        UserHandle.getUserId(callingUid));
                if (targetUid < 0) {
                    if (DEBUG_URI_PERMISSION) Slog.v(TAG_URI_PERMISSION,
                            "Can't grant URI permission no uid for: " + targetPkg);
                    return -1;
                }
            } catch (RemoteException ex) {
                return -1;
            }
        }

        if (targetUid >= 0) {
            // 一般情况 targetUid >= 0，说明 targetPkg 是存在的，那么就判断下 targetPkg 是否已经持有该权限！
            //【*2.7】这里调用了 checkHoldingPermissionsLocked 判断是否已经有权限，如果已经持有，返回 -1；
            if (checkHoldingPermissionsLocked(pm, pi, grantUri, targetUid, modeFlags)) {
                if (DEBUG_URI_PERMISSION) Slog.v(TAG_URI_PERMISSION,
                        "Target " + targetPkg + " already has full permission to " + grantUri);
                return -1;
            }
        } else {
            //【5】如果 targetUid < 0，说明没有 targetPkg，
            // 此时如果 ContentProvider 指定了读写权限，那就要进一步判断是否要授予权限；如果没有指定读写权限
            // 就要看 provider 是否是暴露的，如果是 export 的，那么可以直接访问，无需授权，返回 -1；
            // 否则，需要继续判断；
            boolean allowed = pi.exported;
            if ((modeFlags & Intent.FLAG_GRANT_READ_URI_PERMISSION) != 0) {
                if (pi.readPermission != null) {
                    allowed = false;
                }
            }
            if ((modeFlags & Intent.FLAG_GRANT_WRITE_URI_PERMISSION) != 0) {
                if (pi.writePermission != null) {
                    allowed = false;
                }
            }
            if (allowed) {
                return -1;
            }
        }

        //【6】这里来处理一种跨用户访问的特殊情况：
        // 如果 targetUid 和 uri 不再同一个 userId 下，并且当前 userId 下的应用不需要任何权限就可以访问该 uri，
        // 那么我们也会授予 targetPkg 访问 uri 的权限，
        // 即使 provider 没有授予该权限，这种情况 specialCrossUserGrant 为 true！
        //【*2.8】这里调用了 checkHoldingPermissionsInternalLocked 方法！
        boolean specialCrossUserGrant = UserHandle.getUserId(targetUid) != grantUri.sourceUserId
                && checkHoldingPermissionsInternalLocked(pm, pi, grantUri, callingUid,
                modeFlags, false /*without considering the uid permissions*/);

        //【7】一般情况下，我们的访问是不会是跨用户的，继续判断 provider 是否允许授予 uri 权限！
        if (!specialCrossUserGrant) {
            //【7.1】provider 不允许授予 uri 权限，抛出异常！
            if (!pi.grantUriPermissions) {
                throw new SecurityException("Provider " + pi.packageName
                        + "/" + pi.name
                        + " does not allow granting of Uri permissions (uri "
                        + grantUri + ")");
            }
            //【7.2】provider 允许授予 uri 权限，如果其指定了 uriPermissionPatterns，判断下要访问呢的 url
            // 是不是能够匹配 uriPermissionPatterns，如果不行，那就说明不允许访问，抛出异常！
            if (pi.uriPermissionPatterns != null) {
                final int N = pi.uriPermissionPatterns.length;
                boolean allowed = false;
                for (int i=0; i<N; i++) {
                    if (pi.uriPermissionPatterns[i] != null
                            && pi.uriPermissionPatterns[i].match(grantUri.uri.getPath())) {
                        allowed = true;
                        break;
                    }
                }
                if (!allowed) {
                    throw new SecurityException("Provider " + pi.packageName
                            + "/" + pi.name
                            + " does not allow granting of permission to path of Uri "
                            + grantUri);
                }
            }
        }

        //【8】这里判断了下调用者自身 callingUid 是否有权限访问该 Uri，如果 callingUid 不是 system uid 的话
        // 就会进行检查！
        if (UserHandle.getAppId(callingUid) != Process.SYSTEM_UID) {
            //【*2.7】同样的，这里调用了 checkHoldingPermissionsLocked 方法判断是否已经有权限！
            if (!checkHoldingPermissionsLocked(pm, pi, grantUri, callingUid, modeFlags)) {
                //【*2.10】调用了 checkUriPermissionLocked 判断是否有 uri 权限！
                if (!checkUriPermissionLocked(grantUri, callingUid, modeFlags)) {
                    throw new SecurityException("Uid " + callingUid
                            + " does not have permission to uri " + grantUri);
                }
            }
        }
        //【9】最后返回 targetUid，如果大于 0，说明需要授予权限！！
        return targetUid;
    }
```
我们看到这里会有几种情况出现异常：

- uri 所属的 provider 的 grantUriPermissions 为 false，说明 provider 禁止 uri 访问，会抛出异常；
- 如果 provider 的 grantUriPermissions 为 true，但是 provider 的 uriPermissionPatterns 中的 uri 没有和要访问的 uri 相匹配的，抛出异常；
- 如果调用者接口的程序，不具有对该 uri 的访问权限，那么就会抛出异常；

这里也用到了那 2 个 flags，不在多说了！

### 2.6.1 调用时机

该方法的调用比较少，在 ActivityManagerService 中：

- 一个是 2.4 checkGrantUriPermission 方法；
- 一个是 2.5 checkGrantUriPermissionFromIntentLocked 方法；
- 一个是在 grantUriPermissionLocked 方法，关于 grant 后面单独分析；

## 2.7 checkHoldingPermissionsLocked

判断 uid 是否已经有访问 uri 的权限！
```java
    private final boolean checkHoldingPermissionsLocked(
            IPackageManager pm, ProviderInfo pi, GrantUri grantUri, int uid, final int modeFlags) {
        if (DEBUG_URI_PERMISSION) Slog.v(TAG_URI_PERMISSION,
                "checkHoldingPermissionsLocked: uri=" + grantUri + " uid=" + uid);
        //【1】首先，如果 uid 和 uri 的 userId 不相等的话，需要检查 uid 是否具有跨用户的权限！
        if (UserHandle.getUserId(uid) != grantUri.sourceUserId) {
            //【*2.12】这里调用了 checkComponentPermission 方法，如果没有跨用户的权限，那肯定是不会有
            // 访问 url 的权限的！！
            if (ActivityManager.checkComponentPermission(INTERACT_ACROSS_USERS, uid, -1, true)
                    != PERMISSION_GRANTED) {
                return false;
            }
        }
        //【*2.8】如果是同一用户下的，调用 checkHoldingPermissionsInternalLocked 方法进一步处理，
        // 这里 considerUidPermissions 为 true！表示需要校验 uid 权限！
        return checkHoldingPermissionsInternalLocked(pm, pi, grantUri, uid, modeFlags, true);
    }

```

### 2.7.1 调用时机

该方法的调用比较少，在 ActivityManagerService 中：

- 一个是 2.6 checkGrantUriPermissionLocked 方法；
- 一个是在 revokeUriPermissionLocked 方法，关于 revoke 后面单独分析；

## 2.8 checkHoldingPermissionsInternalLocked

判断 uid 是否已经具有访问 uri 的权限，该接口并不检查是否是跨用户的调用！

considerUidPermissions 参数表示是否检查 uid 权限，一般我们是会传入 true 的！

```java
    private final boolean checkHoldingPermissionsInternalLocked(IPackageManager pm, ProviderInfo pi,
            GrantUri grantUri, int uid, final int modeFlags, boolean considerUidPermissions) {
        //【1】如果 provider 的 uid 和访问者 uid 一样，同一应用，那是有权限访问的，返回 true；
        // 如果 provider.exported 为 false，那么，其没有权限访问，返回 false；
        if (pi.applicationInfo.uid == uid) {
            return true;
        } else if (!pi.exported) {
            return false;
        }

        //【2】解析 modeFlags 中的 read 和 write 标志位，如果设置了该标志位，则为 false
        // 那么，下面会进行权限检查！
        boolean readMet = (modeFlags & Intent.FLAG_GRANT_READ_URI_PERMISSION) == 0;
        boolean writeMet = (modeFlags & Intent.FLAG_GRANT_WRITE_URI_PERMISSION) == 0;
        try {
            //【2.1】首先判断 uid 是否持有 read 和 write 的 top-level 的权限，该权限是在 <provider> 标签中设定的！
            // 当然判断的前提是 considerUidPermissions 为 true！
            if (!readMet && pi.readPermission != null && considerUidPermissions
                    && (pm.checkUidPermission(pi.readPermission, uid) == PERMISSION_GRANTED)) {
                readMet = true;
            }
            if (!writeMet && pi.writePermission != null && considerUidPermissions
                    && (pm.checkUidPermission(pi.writePermission, uid) == PERMISSION_GRANTED)) {
                writeMet = true;
            }

            //【2.2】如果 provider 没有设置 top-level 的权限的话，allowDefaultRead 和 allowDefaultWrite 为 true！
            // 否则，为 false！
            boolean allowDefaultRead = pi.readPermission == null;
            boolean allowDefaultWrite = pi.writePermission == null;

            //【2.3】下面是非 top-level 的权限检查！
            // 如果 provider 设置了 path permission，那还要判断 uid 其是否持有对应的 path permission！
            // 当然判断的前提是 considerUidPermissions 为 true！
            final PathPermission[] pps = pi.pathPermissions;
            if (pps != null) {
                final String path = grantUri.uri.getPath();
                int i = pps.length;
                //【2.3.1】只有 readMet/writeMet 为 false 时，才会做 path permission 校验！
                // 因为如果 readMet | writeMet 为 true 的话，说明 uid 持有 top-level 的权限！
                while (i > 0 && (!readMet || !writeMet)) {
                    i--;
                    PathPermission pp = pps[i];
                    if (pp.match(path)) {
                        if (!readMet) {
                            final String pprperm = pp.getReadPermission();
                            if (DEBUG_URI_PERMISSION) Slog.v(TAG_URI_PERMISSION,
                                    "Checking read perm for " + pprperm + " for " + pp.getPath()
                                    + ": match=" + pp.match(path)
                                    + " check=" + pm.checkUidPermission(pprperm, uid));
                            if (pprperm != null) {
                                //【2.3.1】当 considerUidPermissions 为 true，并且 uid 有权限，那么 readMet 为 true
                                // 表示可读，否则 readMet == allowDefaultRead == false；
                                //【*3.1】调用了 pms 的 checkUidPermission 接口！
                                if (considerUidPermissions && pm.checkUidPermission(pprperm, uid)
                                        == PERMISSION_GRANTED) {
                                    readMet = true;
                                } else {
                                    allowDefaultRead = false;
                                }
                            }
                        }
                        if (!writeMet) {
                            final String ppwperm = pp.getWritePermission();
                            if (DEBUG_URI_PERMISSION) Slog.v(TAG_URI_PERMISSION,
                                    "Checking write perm " + ppwperm + " for " + pp.getPath()
                                    + ": match=" + pp.match(path)
                                    + " check=" + pm.checkUidPermission(ppwperm, uid));
                            if (ppwperm != null) {
                                //【2.3.2】当 considerUidPermissions 为 true，并且 uid 有权限，那么 writeMet 为 true
                                // 表示可写；否则 writeMet == allowDefaultWrite == false；
                                //【*3.1】调用了 pms 的 checkUidPermission 接口！
                                if (considerUidPermissions && pm.checkUidPermission(ppwperm, uid)
                                        == PERMISSION_GRANTED) {
                                    writeMet = true;
                                } else {
                                    allowDefaultWrite = false;
                                }
                            }
                        }
                    }
                }
            }

            //【3】如果 allowDefaultRead/allowDefaultWrite 为 true 了，设置 readMet/writeMet 为 true！
            if (allowDefaultRead) readMet = true;
            if (allowDefaultWrite) writeMet = true;

        } catch (RemoteException e) {
            return false;
        }
        // 最后返回 readMet && writeMet，如果为 true，表示其具有访问 uri的权限！
        return readMet && writeMet;
    }
```
通过逻辑可以知道：

- 如果 considerUidPermissions 为 true，那么会调用 checkUidPermission 进行 uid 权限的校验!

- 如果 considerUidPermissions 为 false 的话，如果 uid 想要对 provider 持有访问权限的话，只能是

```java
pi.readPermission == null;
pi.writePermission == null;
pi.pathPermissions == null;
pi.exported == true
```
也就是说，provider 不设置任何权限，同时是暴露的！！

### 2.8.1 调用时机

该方法的调用比较少，在 ActivityManagerService 中：

- 一个是 2.7 checkHoldingPermissionsLocked 方法中；
- 一个是在 revokeUriPermissionLocked 方法，关于 revoke 后面单独分析；

## 2.9 checkUriPermission

用于检查 uid 是否持有 uri permissions：

```java
    public int checkUriPermission(Uri uri, int pid, int uid,
            final int modeFlags, int userId, IBinder callerToken) {
        enforceNotIsolatedCaller("checkUriPermission");
        //【1】通过 IBinder 对象获得调用者进程的 pid 和 uid！
        Identity tlsIdentity = sCallerIdentity.get();
        if (tlsIdentity != null && tlsIdentity.token == callerToken) {
            uid = tlsIdentity.uid;
            pid = tlsIdentity.pid;
        }

        //【2】如果 pid 是系统进程自身，默认授予！
        if (pid == MY_PID) {
            return PackageManager.PERMISSION_GRANTED;
        }
        synchronized (this) {
            //【*2.10】调用 checkUriPermissionLocked 继续检查是否具有 uri 权限！
            return checkUriPermissionLocked(new GrantUri(userId, uri, false), uid, modeFlags)
                    ? PackageManager.PERMISSION_GRANTED
                    : PackageManager.PERMISSION_DENIED;
        }
    }
```
这里的 sCallerIdentity 是一个线程本地变量：
```java
    private static final ThreadLocal<Identity> sCallerIdentity = new ThreadLocal<Identity>();
```
我们通过传入的 IBinder 对象和 Identity.token 进行匹配，然后就可以获得调用者的进程 pid 和 uid！

### 2.9.1 调用时机 - 跨进程

该方法主要是从 Context 的 checkUriPermission 调用过来的！

调用方法：
```java
try {
    ActivityManagerNative.getDefault().checkUriPermission(...);
} catch (RemoteException e) {
    throw e.rethrowFromSystemServer();
}
```
具体流程：

- **ActivityManagerProxy**

```java
    public int checkUriPermission(Uri uri, int pid, int uid, int mode, int userId,
            IBinder callerToken) throws RemoteException {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IActivityManager.descriptor);
        uri.writeToParcel(data, 0);
        data.writeInt(pid);
        data.writeInt(uid);
        data.writeInt(mode);
        data.writeInt(userId);
        data.writeStrongBinder(callerToken);
        mRemote.transact(CHECK_URI_PERMISSION_TRANSACTION, data, reply, 0);
        reply.readException();
        int res = reply.readInt();
        data.recycle();
        reply.recycle();
        return res;
    }
```

- **ActivityManagerNative**

```java
        case CHECK_URI_PERMISSION_TRANSACTION: {
            data.enforceInterface(IActivityManager.descriptor);
            Uri uri = Uri.CREATOR.createFromParcel(data);
            int pid = data.readInt();
            int uid = data.readInt();
            int mode = data.readInt();
            int userId = data.readInt();
            IBinder callerToken = data.readStrongBinder();
            //【*1.9】继续调用；
            int res = checkUriPermission(uri, pid, uid, mode, userId, callerToken);
            reply.writeNoException();
            reply.writeInt(res);
            return true;
        }
```



## 2.10 checkUriPermissionLocked

用于检查 uid 是否已经被授予uri permissions！

```java
    private final boolean checkUriPermissionLocked(GrantUri grantUri, int uid,
            final int modeFlags) {
        final boolean persistable = (modeFlags & Intent.FLAG_GRANT_PERSISTABLE_URI_PERMISSION) != 0;
        final int minStrength = persistable ? UriPermission.STRENGTH_PERSISTABLE
                : UriPermission.STRENGTH_OWNED;

        //【1】如果 uid 是 0，为 root 用户，默认是授予，返回 true！
        if (uid == 0) {
            return true;
        }
        //【2】获得 uid 已经被授予的 UriPermission，如果没有，返回 false，表示没有授予权限！！
        final ArrayMap<GrantUri, UriPermission> perms = mGrantedUriPermissions.get(uid);
        if (perms == null) return false;

        //【3】精确匹配 uri！
        final UriPermission exactPerm = perms.get(grantUri);
        if (exactPerm != null && exactPerm.getStrength(modeFlags) >= minStrength) {
            return true;
        }

        //【4】前缀匹配 uri！
        final int N = perms.size();
        for (int i = 0; i < N; i++) {
            final UriPermission perm = perms.valueAt(i);
            if (perm.uri.prefix && grantUri.uri.isPathPrefixMatch(perm.uri.uri)
                    && perm.getStrength(modeFlags) >= minStrength) {
                return true;
            }
        }

        return false;
    }
```

对于 modeFlags，除了能设置上面的 2 个标志之外，还可以设置下面的标志位：

```java
Intent.FLAG_GRANT_PERSISTABLE_URI_PERMISSION
```
FLAG_GRANT_READ_URI_PERMISSION 跟 FLAG_GRANT_WRITE_URI_PERMISSION 是用于授予临时权限，如果和 FLAG_GRANT_PERSISTABLE_URI_PERMISSION 结合使用，URI 权限会持久存在，重启也无法撤消，直到明确的用 revokeUriPermission(Uri, int) 撤销。 

这个 flag 只提供可能持久授权。但是接收的应用必须调用 ContentResolver.takePersistableUriPermission(Uri, int) 方法实现 。


### 2.10.1 调用时机

该方法的调用比较少，在 ActivityManagerService 中：

- 一个是 2.9 checkUriPermission 方法中；
- 一个是 2.10 checkUriPermissionLocked 方法中；

## 2.11 checkPermissionWithToken

检查 IBinder 对象对应的进程是否具有指定权限！

```java
    @Override
    public int checkPermissionWithToken(String permission, int pid, int uid, IBinder callerToken) {
        if (permission == null) {
            return PackageManager.PERMISSION_DENIED;
        }

        //【1】计算 IBinder 对象对应的进程的 pid 和 uid！
        Identity tlsIdentity = sCallerIdentity.get();
        if (tlsIdentity != null && tlsIdentity.token == callerToken) {
            Slog.d(TAG, "checkComponentPermission() adjusting {pid,uid} to {"
                    + tlsIdentity.pid + "," + tlsIdentity.uid + "}");
            uid = tlsIdentity.uid;
            pid = tlsIdentity.pid;
        }
        //【*2.12】调用 checkComponentPermission 方法，继续检查：
        return checkComponentPermission(permission, pid, uid, -1, true);
    }
```

### 2.11.1 跨进程调用

调用方法：
```java
try {
    ActivityManagerNative.getDefault().checkPermissionWithToken(...);
} catch (RemoteException e) {
    throw e.rethrowFromSystemServer();
}
```
具体流程：

- **ActivityManagerProxy**

```java
    public int checkPermissionWithToken(String permission, int pid, int uid, IBinder callerToken)
            throws RemoteException {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IActivityManager.descriptor);
        data.writeString(permission);
        data.writeInt(pid);
        data.writeInt(uid);
        data.writeStrongBinder(callerToken);
        mRemote.transact(CHECK_PERMISSION_WITH_TOKEN_TRANSACTION, data, reply, 0);
        reply.readException();
        int res = reply.readInt();
        data.recycle();
        reply.recycle();
        return res;
    }
```

- **ActivityManagerNative**

```java
    case CHECK_PERMISSION_WITH_TOKEN_TRANSACTION: {
        data.enforceInterface(IActivityManager.descriptor);
        String perm = data.readString();
        int pid = data.readInt();
        int uid = data.readInt();
        IBinder token = data.readStrongBinder();
        //【*2.11】继续调用；
        int res = checkPermissionWithToken(perm, pid, uid, token);
        reply.writeNoException();
        reply.writeInt(res);
        return true;
    }
```


## 2.12 checkComponentPermission
```java
    int checkComponentPermission(String permission, int pid, int uid,
            int owningUid, boolean exported) {
        if (pid == MY_PID) { // 如果是进程自身，是持有权限的！
            return PackageManager.PERMISSION_GRANTED;
        }
        //【*2.12.1】调用 ActivityManager 的 checkComponentPermission 方法！
        return ActivityManager.checkComponentPermission(permission, uid,
                owningUid, exported);
    }
```

### 2.12.1 ActivityM.checkComponentPermission

这个方法是一个 hide 方法，只能由系统调用！

```java
    /** @hide */
    public static int checkComponentPermission(String permission, int uid,
            int owningUid, boolean exported) {
        //【1】如果是 root 用户或者 system 用户，
        final int appId = UserHandle.getAppId(uid);
        if (appId == Process.ROOT_UID || appId == Process.SYSTEM_UID) {
            return PackageManager.PERMISSION_GRANTED;
        }
        //【2】隔离进程无权限！
        if (UserHandle.isIsolated(uid)) {
            return PackageManager.PERMISSION_DENIED;
        }
        //【3】如果是相同 uid，有权限！
        if (owningUid >= 0 && UserHandle.isSameApp(uid, owningUid)) {
            return PackageManager.PERMISSION_GRANTED;
        }
        //【4】如果组件不是暴露的，那就无权限！
        if (!exported) {
            /*
            RuntimeException here = new RuntimeException("here");
            here.fillInStackTrace();
            Slog.w(TAG, "Permission denied: checkComponentPermission() owningUid=" + owningUid,
                    here);
            */
            return PackageManager.PERMISSION_DENIED;
        }
        if (permission == null) {
            return PackageManager.PERMISSION_GRANTED;
        }
        //【*3.1】调用 pms 的 checkUidPermission 方法继续检查权限！
        try {
            return AppGlobals.getPackageManager()
                    .checkUidPermission(permission, uid);
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
```

### 2.12.2 ActivityM.checkUidPermission

同时 ActivityManager 也有 checkUidPermission 方法，这方法是一个 hide 方法，只能由系统调用！

该方法会直接访问 pms 的接口！

```java
    /** @hide */
    public static int checkUidPermission(String permission, int uid) {
        try {
            //【*3.1】调用 pms 的 checkUidPermission 方法继续检查权限！
            return AppGlobals.getPackageManager()
                    .checkUidPermission(permission, uid);
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
```

# 3 权限检查关键接口 - PackageManagerService

通过分析系统中的一些 checkPermissions 接口，我们发现最终都调用了 PackageManagerService.checkUidPermission 方法，下面我们去看下：


## 3.1 checkUidPermission

用于检查 uid 是否具有权限 permName！

```java
    @Override
    public int checkUidPermission(String permName, int uid) {
        final int userId = UserHandle.getUserId(uid);
        //【1】首先是校验 userId 有效性！
        if (!sUserManager.exists(userId)) {
            return PackageManager.PERMISSION_DENIED;
        }
        synchronized (mPackages) {
            //【2】尝试根据 uid 获得其 PackageSetting 或者 SharedUserSetting 对象，这取决于 uid 是独立还是共享的！
            Object obj = mSettings.getUserIdLPr(UserHandle.getAppId(uid));
            if (obj != null) {
                final SettingBase ps = (SettingBase) obj;
                //【2.1】获得权限管理对象 PermissionsState！
                final PermissionsState permissionsState = ps.getPermissionsState();
                // 如果该权限确实授予了该 uid，返回 PackageManager.PERMISSION_GRANTED！
                if (permissionsState.hasPermission(permName, userId)) {
                    return PackageManager.PERMISSION_GRANTED;
                }
                
                //【2.2】这里是处理一种特殊的权限！
                if (Manifest.permission.ACCESS_COARSE_LOCATION.equals(permName) && permissionsState
                        .hasPermission(Manifest.permission.ACCESS_FINE_LOCATION, userId)) {
                    return PackageManager.PERMISSION_GRANTED;
                }
            } else {
                //【3】mSystemPermissions 中保存的是系统 uid 和其权限的对应关系！
                ArraySet<String> perms = mSystemPermissions.get(uid);
                if (perms != null) {
                    if (perms.contains(permName)) {
                        return PackageManager.PERMISSION_GRANTED;
                    }
                    if (Manifest.permission.ACCESS_COARSE_LOCATION.equals(permName) && perms
                            .contains(Manifest.permission.ACCESS_FINE_LOCATION)) {
                        return PackageManager.PERMISSION_GRANTED;
                    }
                }
            }
        }

        return PackageManager.PERMISSION_DENIED;
    }
```
我们来看下 getUserIdLPr 方法的逻辑：

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
- **uid 的取值范围**：

从 Process.FIRST_APPLICATION_UID（10000） 开始的 uid 是普通 Android 的应用程序，即使用户安装的应用程序，最大值为 Process.LAST_APPLICATION_UID（19999）！

而 Process.FIRST_APPLICATION_UID 以下的 uid 都是系统应用和系统进程的！

- **PackageSetting 和 SharedUserSetting**：

如果应用程序是独立 uid 的话，那么其会有一个 PackageSetting 对象，然后会根据其 uid 的类型，添加到 mUserIds 或者 mOtherUserIds；

如果应用程序是共享 uid 的话，除了会创建 PackageSetting 对象，还会创建一个 SharedUserSetting 对象，用来封装共享 uid 的信息 ，然后会根据其 uid 的类型，将 SharedUserSetting 对象添加到 mUserIds 或者 mOtherUserIds；

- **权限的处理**：

如果应用是独立 uid 的话，那么 getUserIdLPr 返回是 PackageSetting 对象！

如果应用是共享 uid 的话，那么 getUserIdLPr 返回的是 SharedUserSetting 对象，这是因为共享同一个 uid 的应用持有相同的权限！


PackageSetting 和 SharedUserSetting 都有一个 PermissionsState 对象，用于封装 package 在不同 userId 下的权限授予情况！


## 3.2 checkPermission

PMS 还有另外一个用于检查权限的方法，判断包名对应的应用程序是否有权限！

```java
    @Override
    public int checkPermission(String permName, String pkgName, int userId) {
        if (!sUserManager.exists(userId)) {
            return PackageManager.PERMISSION_DENIED;
        }

        synchronized (mPackages) {
            //【1】获得该 pkg 对应的 Package 实例！
            final PackageParser.Package p = mPackages.get(pkgName);
            if (p != null && p.mExtras != null) {
                final PackageSetting ps = (PackageSetting) p.mExtras;
                final PermissionsState permissionsState = ps.getPermissionsState();
                //【1.1】判断 permissionsState 中该权限状态是否是 true！
                if (permissionsState.hasPermission(permName, userId)) {
                    return PackageManager.PERMISSION_GRANTED;
                }
                //【1.2】对于 ACCESS_COARSE_LOCATION 比较特殊，如果应用已经有了 ACCESS_FINE_LOCATION
                // 那么应用也会有 ACCESS_COARSE_LOCATION 权限！
                if (Manifest.permission.ACCESS_COARSE_LOCATION.equals(permName) && permissionsState
                        .hasPermission(Manifest.permission.ACCESS_FINE_LOCATION, userId)) {
                    return PackageManager.PERMISSION_GRANTED;
                }
            }
        }

        return PackageManager.PERMISSION_DENIED;
    }
```

该方法用于判断 pkgName 在 userId 下，是否有权限 permName！

### 3.2.1 调用时机

PMS 的接口可以被其他 check 接口调用，同时我们也可以直接调用其进行权限检查：

#### 3.2.1.1 BroadcastQueue.processNextBroadcast

在我们发送广播的时候，如果广播指定了接收者的权限，那么我们会校验

```java
    final void processNextBroadcast(boolean fromMsg) {
        ... ... ...
           if (!skip && info.activityInfo.applicationInfo.uid != Process.SYSTEM_UID &&
                r.requiredPermissions != null && r.requiredPermissions.length > 0) {
                for (int i = 0; i < r.requiredPermissions.length; i++) {
                    String requiredPermission = r.requiredPermissions[i];
                    try {
                        perm = AppGlobals.getPackageManager().
                                checkPermission(requiredPermission,
                                        info.activityInfo.applicationInfo.packageName,
                                        UserHandle
                                                .getUserId(info.activityInfo.applicationInfo.uid));
                    } catch (RemoteException e) {
                        perm = PackageManager.PERMISSION_DENIED;
                    }
                    ... ... ...
                }
            }
        ... ... ...
    }
```













