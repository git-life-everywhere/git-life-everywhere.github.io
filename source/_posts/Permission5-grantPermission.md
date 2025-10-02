# Permission第 5 篇 - grantPermission 权限授予
title: Permission第 5 篇 - grantPermission 权限授予
date: 2017/08/25 20:46:25
catalog: true
categories:
- AndroidFramework源码分析
- Permission权限管理
tags: Permission权限管理
---

[TOC]

# 0 综述

基于 Android 7.1.1 源码，分析系统的权限管理机制！

本片文章总结一下权限授予的相关接口。

对于 dangrous 权限，应用程序需要显式的弹出弹窗，让用户主动的选择。当用户撤销或者授予时，会主动的调用！

但是对于 normal permission，siginature permission 以及 uri permission 都是无需弹窗申请的！

那么我们来看下系统中有那些和其相关的接口！

# 1 grantPermission - ContextImpl 

在 ContextImpl 中提供了一个 grantUriPermission 方法：

## 1.1 grantUriPermission

该方法由于是在 ContextImpl 暴露的，所以应用程序可以通过该方法 grant uri permission 给其他应用程序：

```java
    @Override
    public void grantUriPermission(String toPackage, Uri uri, int modeFlags) {
         try {
            //【*2.1】继续授予：
            ActivityManagerNative.getDefault().grantUriPermission(
                    mMainThread.getApplicationThread(), toPackage,
                    ContentProvider.getUriWithoutUserId(uri), modeFlags, resolveUserId(uri));
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
```
不多说了!

# 2 grantPermission - ActivityManagerService 

ActivityManagerService 内部提供了一些和 uri permission 相关的接口：

## 2.1 grantUriPermission

授予一个 uri permisson 给 target package，这个方法授予的 permission 是一个 global permisson；

因为其在调用 grantUriPermissionLocked 时传入的 UriPermissionOwner owner 为 null；

```java
    @Override
    public void grantUriPermission(IApplicationThread caller, String targetPkg, Uri uri,
            final int modeFlags, int userId) {
        enforceNotIsolatedCaller("grantUriPermission");
        //【*2.1.1】创建了一个 GrantUri 对象！
        GrantUri grantUri = new GrantUri(userId, uri, false);
        synchronized(this) {
            final ProcessRecord r = getRecordForAppLocked(caller);
            if (r == null) {
                throw new SecurityException("Unable to find app for caller "
                        + caller
                        + " when granting permission to uri " + grantUri);
            }
            if (targetPkg == null) {
                throw new IllegalArgumentException("null target");
            }
            if (grantUri == null) {
                throw new IllegalArgumentException("null uri");
            }
            //【1】强制检查下 intent 的 flags 标志位！
            Preconditions.checkFlagsArgument(modeFlags, Intent.FLAG_GRANT_READ_URI_PERMISSION
                    | Intent.FLAG_GRANT_WRITE_URI_PERMISSION
                    | Intent.FLAG_GRANT_PERSISTABLE_URI_PERMISSION
                    | Intent.FLAG_GRANT_PREFIX_URI_PERMISSION);
            //【*2.2】调用了另外一个方法！
            grantUriPermissionLocked(r.uid, targetPkg, grantUri, modeFlags, null,
                    UserHandle.getUserId(r.uid));
        }
    }
```

### 2.1.1 new GrantUri

GrantUri 实例是对 uri 信息的简单封装：

```java
    public static class GrantUri {
        public final int sourceUserId;
        public final Uri uri;
        public boolean prefix;

        public GrantUri(int sourceUserId, Uri uri, boolean prefix) {
            this.sourceUserId = sourceUserId;
            this.uri = uri;
            this.prefix = prefix;
        }
```


### 2.1.2 跨进程调用

ContextImpl 的 grantUriPermission 可以跨进程调用该方法：

- **ActivityManagerProxy**

```java
    public void grantUriPermission(IApplicationThread caller, String targetPkg,
            Uri uri, int mode, int userId) throws RemoteException {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IActivityManager.descriptor);
        data.writeStrongBinder(caller.asBinder());
        data.writeString(targetPkg);
        uri.writeToParcel(data, 0);
        data.writeInt(mode);
        data.writeInt(userId);
        mRemote.transact(GRANT_URI_PERMISSION_TRANSACTION, data, reply, 0);
        reply.readException();
        data.recycle();
        reply.recycle();
    }
```

- **ActivityManagerNative**

```java
        case GRANT_URI_PERMISSION_TRANSACTION: {
            data.enforceInterface(IActivityManager.descriptor);
            IBinder b = data.readStrongBinder();
            IApplicationThread app = ApplicationThreadNative.asInterface(b);
            String targetPkg = data.readString();
            Uri uri = Uri.CREATOR.createFromParcel(data);
            int mode = data.readInt();
            int userId = data.readInt();
            grantUriPermission(app, targetPkg, uri, mode, userId);
            reply.writeNoException();
            return true;
        }
```


## 2.2 grantUriPermissionLocked - 内部调用

该方法的特点是，系统在调用时，会持有 AMS 大锁，所以此时系统进程的其他需要 AMS 大锁的线程会等待：

```java
    void grantUriPermissionLocked(int callingUid, String targetPkg, GrantUri grantUri,
            final int modeFlags, UriPermissionOwner owner, int targetUserId) {
        if (targetPkg == null) {
            throw new NullPointerException("targetPkg");
        }
        int targetUid;
        final IPackageManager pm = AppGlobals.getPackageManager();
        try {
            targetUid = pm.getPackageUid(targetPkg, MATCH_DEBUG_TRIAGED_MISSING, targetUserId);
        } catch (RemoteException ex) {
            return;
        }
        //【1】检查是否需要授予 uri 权限，
        targetUid = checkGrantUriPermissionLocked(callingUid, targetPkg, grantUri, modeFlags,
                targetUid);
        if (targetUid < 0) {
            return;
        }
        //【*2.3】调用了另外一个方法！
        grantUriPermissionUncheckedLocked(targetUid, targetPkg, grantUri, modeFlags,
                owner);
    }
```
这里调用了 checkGrantUriPermissionLocked 方法，检查是否需要授予 uri 权限！

checkGrantUriPermissionLocked 方法会判断是否已经有 uri 权限，以及发送包含 uri 的 intent 程序是否有权限访问 uri 等等！！

只有发送方对 uri 有权限，接收者才能被授予权限！！

### 2.2.1 调用时机

 - 第二个是在 1.1 grantUriPermission 中调用的；
 - 第二个是在 1.6 grantUriPermissionFromOwner 中调用的；

该方法也是一个内部方法！

## 2.3 grantUriPermissionUncheckedLocked - 内部调用，核心接口

这个方法不会做一些 check 操作，所以最好不要直接调用，默认发送方对 uri 是有权限的：

```java
    void grantUriPermissionUncheckedLocked(int targetUid, String targetPkg, GrantUri grantUri,
            final int modeFlags, UriPermissionOwner owner) {
        //【1】这个方法之前有讲过，判断是否设置了和 uri 下相关的标志位！
        if (!Intent.isAccessUriMode(modeFlags)) {
            return;
        }

        if (DEBUG_URI_PERMISSION) Slog.v(TAG_URI_PERMISSION,
                "Granting " + targetPkg + "/" + targetUid + " permission to " + grantUri);

        final String authority = grantUri.uri.getAuthority();
        //【2】根据 uri 找到对应的 provider！
        final ProviderInfo pi = getProviderInfoLocked(authority, grantUri.sourceUserId,
                MATCH_DEBUG_TRIAGED_MISSING);
        if (pi == null) {
            Slog.w(TAG, "No content provider found for grant: " + grantUri.toSafeString());
            return;
        }
        //【3】判断是否仅仅匹配 uri 前缀！
        if ((modeFlags & Intent.FLAG_GRANT_PREFIX_URI_PERMISSION) != 0) {
            grantUri.prefix = true;
        }
        //【*2.3.1】找到已存在或者新建的 UriPermission 实例；
        final UriPermission perm = findOrCreateUriPermissionLocked(
                pi.packageName, targetPkg, targetUid, grantUri);
        //【*2.3.2】授予该权限；
        perm.grantModes(modeFlags, owner);
    }
```
所以我们看到，对于 uri permission，其管理是在 AMS 中的！


### 2.3.1 findOrCreateUriPermissionLocked

找到一个已存在或者新建的 UriPermission 实例：

```java
    private UriPermission findOrCreateUriPermissionLocked(String sourcePkg,
            String targetPkg, int targetUid, GrantUri grantUri) {
        //【1】尝试找到 targetUid 被授予的所有 uri permission 集合
        ArrayMap<GrantUri, UriPermission> targetUris = mGrantedUriPermissions.get(targetUid);
        //【2】如果没有的话，下面会做初始化工作！
        if (targetUris == null) {
            targetUris = Maps.newArrayMap();
            mGrantedUriPermissions.put(targetUid, targetUris);
        }
        //【*2.3.1.1】同理，创建一个新的 UriPermission！
        UriPermission perm = targetUris.get(grantUri);
        if (perm == null) {
            perm = new UriPermission(sourcePkg, targetPkg, targetUid, grantUri);
            targetUris.put(grantUri, perm);
        }
        //【3】返回 UriPermission 实例！
        return perm;
    }
```

在 ActivityManagerService 内部是有一个 mGrantedUriPermissions 数组：

```java
    //【1】用于保存所有已经被授予的 uri permissions；
    @GuardedBy("this")
            mGrantedUriPermissions = new SparseArray<ArrayMap<GrantUri, UriPermission>>();
```
SparseArray 的下标是被授予 uri permission 的目标 uid，值是一个 ArrayMap<GrantUri, UriPermission>，保存了 uri 和对应的 UriPermission 的映射关系！

#### 2.3.1.1 new UriPermission

创建一个更新的 UriPermission：

```java
final class UriPermission {
    private static final String TAG = "UriPermission";

    public static final int STRENGTH_NONE = 0;
    public static final int STRENGTH_OWNED = 1;
    public static final int STRENGTH_GLOBAL = 2;
    public static final int STRENGTH_PERSISTABLE = 3;

    final int targetUserId; // 目标 userId；
    final String sourcePkg; // 源 pkg；
    final String targetPkg; // 目标 pkg；

    // 目标 uid，用于缓存 targetPkg 对应的 uid，不会被 persist；
    final int targetUid;

    final GrantUri uri; // 对应的 uri；

    // 允许模式，是其他四个 mode 的结合，所有的 uri permission 都要用该 mode 标志处理权限检查和授予；
    int modeFlags = 0;

    // 允许模式，当该模式被设置，表示该 uri permission 有非全局的显式 owner；
    int ownedModeFlags = 0;
    // 允许模式，当该模式被设置，表示该 uri permission 是全局 global 的；
    int globalModeFlags = 0;
    
    // 允许模式，当该模式被设置，表示该 uri permission 可能需要被持久化；
    int persistableModeFlags = 0;
    // 允许模式，当该模式被设置，表示该 uri permission 需要被持久化；
    int persistedModeFlags = 0;

    /**
     * Timestamp when {@link #persistedModeFlags} was first defined in
     * {@link System#currentTimeMillis()} time base.
     */
    long persistedCreateTime = INVALID_TIME;

    private static final long INVALID_TIME = Long.MIN_VALUE;

    private ArraySet<UriPermissionOwner> mReadOwners; // 保存所有读权限的拥有者；
    private ArraySet<UriPermissionOwner> mWriteOwners; // 保存所有写权限的拥有者；

    private String stringName;

    UriPermission(String sourcePkg, String targetPkg, int targetUid, GrantUri uri) {
        this.targetUserId = UserHandle.getUserId(targetUid);
        this.sourcePkg = sourcePkg;
        this.targetPkg = targetPkg;
        this.targetUid = targetUid;
        this.uri = uri;
    }
    ... ... ... ...
}
```
上面只是简单的介绍了下 UriPermission  内部的一些主要的属性，后面我们遇到了具体的逻辑时再分析！

### 2.3.2 UriPermission.grantModes

授予 uri permission：

```java
    void grantModes(int modeFlags, UriPermissionOwner owner) {
        //【1】判断是否需要 persist 该权限！
        final boolean persistable = (modeFlags & Intent.FLAG_GRANT_PERSISTABLE_URI_PERMISSION) != 0;

        //【2】只保留 modeFlags 中的 FLAG_GRANT_READ_URI_PERMISSION 和 FLAG_GRANT_WRITE_URI_PERMISSION 位！
        modeFlags &= (Intent.FLAG_GRANT_READ_URI_PERMISSION
                | Intent.FLAG_GRANT_WRITE_URI_PERMISSION);

        //【2】如果 intent 设置了 FLAG_GRANT_PERSISTABLE_URI_PERMISSION 标志位；
        // 说明该权限可能需要 persisit 所以设置 persistableModeFlags 标志位
        if (persistable) {
            persistableModeFlags |= modeFlags;
        }
        
        //【3】如果 owner 为 null，那么该权限就是一个全局 uri permission！
        // 设置 globalModeFlags 标志位；
        if (owner == null) {
            globalModeFlags |= modeFlags;

        } else {
            //【4】如果传入了指定的 owner，那么该权限就不是一个全局 uri permission！
            if ((modeFlags & Intent.FLAG_GRANT_READ_URI_PERMISSION) != 0) {
                //【*2.3.2.1】添加读权限的拥有者；
                addReadOwner(owner);
            }
            if ((modeFlags & Intent.FLAG_GRANT_WRITE_URI_PERMISSION) != 0) {
                //【*2.3.2.2】添加写权限的拥有者；
                addWriteOwner(owner);
            }
        }
        //【*2.3.2.3】更新 modeFlags 标志位；
        updateModeFlags ();
    }
```
- **方法流程**：

首先，判断是否需要 persist，同时对传入的 modeFlags 作如下处理：
```java
    modeFlags &= (Intent.FLAG_GRANT_READ_URI_PERMISSION | Intent.FLAG_GRANT_WRITE_URI_PERMISSION);
```
只保留 modeFlags 中的这两位，也就是说，此时，modeFlags 要么择其一，要么二者都有；

如果 modeFlags 需要 persistable，那么设置 persistableModeFlags；

接着，判断是否指定 UriPermissionOwner：

- 如果没有，那么该权限就是 global 的，更新 globalModeFlags 标志位；
- 如果有，说明这是一个有指定 owner 的权限，那就添加到如下集合中：

```java
    private ArraySet<UriPermissionOwner> mReadOwners; // 保存所有非全局读权限的拥有者
    private ArraySet<UriPermissionOwner> mWriteOwners; // 保存所有非全局写权限的拥有者
```
同时，更新 ownedModeFlags 的标志位（只有在初始化的 mReadOwners 和 mWriteOwners，第一次添加 owner 时)

最后，调用 updateModeFlags 更新 modeFlags 


- **总结**：

其实整个过程就是对传入的 modeFlags 的标志位进行一一拆解，然后保存到 ownedModeFlags | globalModeFlags | persistableModeFlags | persistedModeFlags 中，最后在 updateModeFlags 中，用这几个标志位更新 modeFlags；



#### 2.3.2.1 addReadOwner

UriPermission 内部方法，，用于添加一个对该 uri 有非全局，读权限的拥有者；
```java
    private void addReadOwner(UriPermissionOwner owner) {
        //【1】初始化 mReadOwners， 
        if (mReadOwners == null) {
            mReadOwners = Sets.newArraySet();
            //【1.1】更新 ownedModeFlags 标志位；
            ownedModeFlags |= Intent.FLAG_GRANT_READ_URI_PERMISSION;
            //【*2.3.2.3】ownedModeFlags 变化了，更新 modeFlags；
            updateModeFlags();
        }
        //【2】建立相互引用关系！
        if (mReadOwners.add(owner)) {
            owner.addReadPermission(this);
        }
    }
```
因为 mReadOwners 默认为 null，所以如果有最开始要初始化 mReadOwners，我们可以看到：

- 只有在第一次初始化 mReadOwners 时，才会设置 ownedModeFlags 标志位；
- 以后在添加新的 owner 时，就不在处理了！


#### 2.3.2.2 addWriteOwner

UriPermission 内部方法，用于添加一个对该 uri 有非全局，写权限的拥有者；

```java
    private void addWriteOwner(UriPermissionOwner owner) {
        //【1】初始化 mWriteOwners，
        if (mWriteOwners == null) {
            mWriteOwners = Sets.newArraySet();
            //【1.1】更新 ownedModeFlags 标志位；
            ownedModeFlags |= Intent.FLAG_GRANT_WRITE_URI_PERMISSION;
            //【*2.3.2.3】ownedModeFlags 变化了，更新 modeFlags；
            updateModeFlags();
        }
        //【2】建立相互引用关系！
        if (mWriteOwners.add(owner)) {
            owner.addWritePermission(this);
        }
    }
```
因为 mWriteOwners 默认为 null，所以如果有最开始要初始化 mWriteOwners，我们可以看到：

- 只有在第一次初始化 mWriteOwners 时，才会设置 ownedModeFlags 标志位；
- 以后在添加新的 owner 时，就不在处理了！

#### 2.3.2.3 updateModeFlags

UriPermission 内部方法，更新 modeFlags 标志位；

```java
    private void updateModeFlags() {
        final int oldModeFlags = modeFlags;
        //【1】可以看到，就是使用其他四个标志位更新 modeFlags！
        modeFlags = ownedModeFlags | globalModeFlags | persistableModeFlags | persistedModeFlags;

        if (Log.isLoggable(TAG, Log.VERBOSE) && (modeFlags != oldModeFlags)) {
            Slog.d(TAG,
                    "Permission for " + targetPkg + " to " + uri + " is changing from 0x"
                            + Integer.toHexString(oldModeFlags) + " to 0x"
                            + Integer.toHexString(modeFlags),
                    new Throwable());
        }
    }
```
不多说了！


### 2.3.3 调用时机

 - 第二个是在 1.2 grantUriPermissionLocked 中调用的；
 - 第二个是在 1.5 grantUriPermissionUncheckedFromIntentLocked 中调用的；

可以看出，均是内部调用，因为该方法不做 check 所以，不建议直接调用；


## 2.4 grantUriPermissionFromIntentLocked

通过 intent 授予 targetPkg uri permission，该方法会先去 check 看是否需要授予权限：

```java
    void grantUriPermissionFromIntentLocked(int callingUid,
            String targetPkg, Intent intent, UriPermissionOwner owner, int targetUserId) {
        //【1】检查是否需要授予权限；
        NeededUriGrants needed = checkGrantUriPermissionFromIntentLocked(callingUid, targetPkg,
                intent, intent != null ? intent.getFlags() : 0, null, targetUserId);
        if (needed == null) {
            //【1.1】如果不需要，那就返回！
            return;
        }
        //【*2.5】进一步授予 uri pemission；
        grantUriPermissionUncheckedFromIntentLocked(needed, owner);
    }
```

### 2.4.1 调用时机

我们来看下该方法的调用时机，框架中主要的调用均是在和 activity 启动，finish，已经结果返回相关的地方：

#### 2.4.1.1 finishActivityResultsLocked - ActivityStack

在退出一个 activity 的时候，如果我们要返回结果给启动者 activity 的时候：

```java
    final void finishActivityResultsLocked(ActivityRecord r, int resultCode, Intent resultData) {
        ActivityRecord resultTo = r.resultTo;
        if (resultTo != null) {
            if (DEBUG_RESULTS) Slog.v(TAG_RESULTS, "Adding result to " + resultTo
                    + " who=" + r.resultWho + " req=" + r.requestCode
                    + " res=" + resultCode + " data=" + resultData);
            if (resultTo.userId != r.userId) {
                if (resultData != null) {
                    resultData.prepareToLeaveUser(r.userId);
                }
            }
            //【1】关键点，判断了下启动者 acitivty 的 uid 是否大于 0；
            if (r.info.applicationInfo.uid > 0) {
                //【*2.4】授予启动者 acitivty 访问 uri 的权限，传入的 callingUid 是被启动的 activity 的 uid；
                //【*2.4.1.3.1】获得 activity 对应的 UriPermissionsOwner；
                mService.grantUriPermissionFromIntentLocked(r.info.applicationInfo.uid,
                        resultTo.packageName, resultData,
                        resultTo.getUriPermissionsLocked(), resultTo.userId);
            }
            resultTo.addResultLocked(r, r.resultWho, r.requestCode, resultCode,
                                     resultData);
            r.resultTo = null;
        }
        else if (DEBUG_RESULTS) Slog.v(TAG_RESULTS, "No result destination from " + r);
        
        r.results = null;
        r.pendingResults = null;
        r.newIntents = null;
        r.icicle = null;
    }
```

#### 2.4.1.2 sendActivityResultLocked  - ActivityStack

发送启动返回结果给 activity：

```java
    void sendActivityResultLocked(int callingUid, ActivityRecord r,
            String resultWho, int requestCode, int resultCode, Intent data) {
        if (callingUid > 0) {
            //【*2.4】授予 acitivty 访问 uri 的权限！
            //【*2.4.1.3.1】获得 activity 对应的 UriPermissionsOwner；
            mService.grantUriPermissionFromIntentLocked(callingUid, r.packageName,
                    data, r.getUriPermissionsLocked(), r.userId);
        }

        if (DEBUG_RESULTS) Slog.v(TAG, "Send activity result to " + r
                + " : who=" + resultWho + " req=" + requestCode
                + " res=" + resultCode + " data=" + data);
        if (mResumedActivity == r && r.app != null && r.app.thread != null) {
            try {
                ArrayList<ResultInfo> list = new ArrayList<ResultInfo>();
                list.add(new ResultInfo(resultWho, requestCode,
                        resultCode, data));
                //【1】发送启动结果！
                r.app.thread.scheduleSendResult(r.appToken, list);
                return;
            } catch (Exception e) {
                Slog.w(TAG, "Exception thrown sending result to " + r, e);
            }
        }

        r.addResultLocked(null, resultWho, requestCode, resultCode, data);
    }
```
不多说了！

#### 2.4.1.3 deliverNewIntentLocked  - ActivityRecord

该方法会拉起一个 activity 的 onNewIntent 方法：
```java
    final void deliverNewIntentLocked(int callingUid, Intent intent, String referrer) {
        //【*2.4】授予要启动的 activity 访问 intent 中 uri 的权限！
        //【*2.4.1.3.1】获得 activity 对应的 UriPermissionsOwner
        service.grantUriPermissionFromIntentLocked(callingUid, packageName,
                intent, getUriPermissionsLocked(), userId);
        final ReferrerIntent rintent = new ReferrerIntent(intent, referrer);
        boolean unsent = true;
        final ActivityStack stack = task.stack;
        final boolean isTopActivityInStack =
                stack != null && stack.topRunningActivityLocked() == this;
        final boolean isTopActivityWhileSleeping =
                service.isSleepingLocked() && isTopActivityInStack;
        if ((state == ActivityState.RESUMED || state == ActivityState.PAUSED
                || isTopActivityWhileSleeping) && app != null && app.thread != null) {
            try {
                ArrayList<ReferrerIntent> ar = new ArrayList<>(1);
                ar.add(rintent);
                //【1】拉起 onNewIntent 方法：
                app.thread.scheduleNewIntent(
                        ar, appToken, state == ActivityState.PAUSED /* andPause */);
                unsent = false;
            } catch (RemoteException e) {
                Slog.w(TAG, "Exception thrown sending new intent to " + this, e);
            } catch (NullPointerException e) {
                Slog.w(TAG, "Exception thrown sending new intent to " + this, e);
            }
        }
        if (unsent) {
            addNewIntentLocked(rintent);
        }
    }
```

##### 2.4.1.3.1 getUriPermissionsLocked

同样的，ActivityRecord 内部也有一个 getUriPermissionsLocked 方法，用来获得其对应的 UriPermissionOwner 实例：

```java
    UriPermissionOwner getUriPermissionsLocked() {
        if (uriPermissions == null) {
            //【*2.6.1.1】创建一个新的 UriPermissionOwner 实例！                                                               
            uriPermissions = new UriPermissionOwner(service, this);
        }
        return uriPermissions;
    }
```

#### 2.4.1.4 startActivityUnchecked - ActivityStarter

在启动一个新的 acitivity 的时候：

```java
    private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask) {
        setInitialState(r, options, inTask, doResume, startFlags, sourceRecord, voiceSession,
                voiceInteractor);
        computeLaunchingTaskFlags();
        computeSourceStack();
        mIntent.setFlags(mLaunchFlags);
        mReusedActivity = getReusableIntentActivity();
        final int preferredLaunchStackId =
                (mOptions != null) ? mOptions.getLaunchStackId() : INVALID_STACK_ID;

        ... ... ...

        //【*2.4】授予要启动的 activity 访问 intent 中 uri 的权限！
        //【*2.4.1.3.1】获得 activity 对应的 UriPermissionsOwner
        mService.grantUriPermissionFromIntentLocked(mCallingUid, mStartActivity.packageName,
                mIntent, mStartActivity.getUriPermissionsLocked(), mStartActivity.userId);

        ... ... ...

        return START_SUCCESS;
    }
```
不多说了！

## 2.5 grantUriPermissionUncheckedFromIntentLocked

通过 intent 授予 uri permission，但是该方法不会 check 是否需要授予权限，所以尽量不要直接调用这个函数；

```java
    void grantUriPermissionUncheckedFromIntentLocked(NeededUriGrants needed,
            UriPermissionOwner owner) {
        if (needed != null) {
            for (int i=0; i<needed.size(); i++) {
                GrantUri grantUri = needed.get(i);
                //【*2.3】调用了 grantUriPermissionUncheckedLocked 方法，继续授予！
                grantUriPermissionUncheckedLocked(needed.targetUid, needed.targetPkg,
                        grantUri, needed.flags, owner);
            }
        }
    }
```
首先遍历所有的 GrantUri，处理每一个 GrantUri 的授予！

### 2.5.1 调用时机

#### 2.5.1.1 sendServiceArgsLocked

在我们启动 Service 的时候，我们会将启动参数 StartItems 发送给 Service：

```java
    private final void sendServiceArgsLocked(ServiceRecord r, boolean execInFg,
            boolean oomAdjusted) throws TransactionTooLargeException {
        final int N = r.pendingStarts.size();
        if (N == 0) {
            return;
        }
        //【1】如果 Service 有等待处理的启动项目！
        while (r.pendingStarts.size() > 0) {
            Exception caughtException = null;
            ServiceRecord.StartItem si = null;
            try {
                si = r.pendingStarts.remove(0);
                if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "Sending arguments to: "
                        + r + " " + r.intent + " args=" + si.intent);
                if (si.intent == null && N > 1) {
                    continue;
                }
                si.deliveredTime = SystemClock.uptimeMillis();
                r.deliveredStarts.add(si);
                si.deliveryCount++;
                //【1】关键地方是在这里，如果启动项有 neededGrants，那么会授予被启动这 uri permission！
                if (si.neededGrants != null) {
                    //【*2.5】授予 uri 权限！
                    //【*2.5.1.2】通过 StartItem 创建一个新的 UriPermissionOwner 实例！
                    mAm.grantUriPermissionUncheckedFromIntentLocked(si.neededGrants,
                            si.getUriPermissionsLocked());
                }
                //【2】超时处理；
                bumpServiceExecutingLocked(r, execInFg, "start");
                if (!oomAdjusted) {
                    oomAdjusted = true;
                    mAm.updateOomAdjLocked(r.app);
                }
                int flags = 0;
                if (si.deliveryCount > 1) {
                    flags |= Service.START_FLAG_RETRY;
                }
                if (si.doneExecutingCount > 0) {
                    flags |= Service.START_FLAG_REDELIVERY;
                }
                //【3】启动服务！
                r.app.thread.scheduleServiceArgs(r, si.taskRemoved, si.id, flags, si.intent);
            } catch (TransactionTooLargeException e) {
                if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "Transaction too large: intent="
                        + si.intent);
                caughtException = e;
            } catch (RemoteException e) {
                // Remote process gone...  we'll let the normal cleanup take care of this.
                if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "Crashed while sending args: " + r);
                caughtException = e;
            } catch (Exception e) {
                Slog.w(TAG, "Unexpected exception", e);
                caughtException = e;
            }
            ... ... ... ...
        }
    }
```
逻辑很简单，不多说了！

#### 2.5.1.2 StartItem.getUriPermissionsLocked

```java
        UriPermissionOwner getUriPermissionsLocked() {
            if (uriPermissions == null) {
                //【*2.6.1.1】新建一个 UriPermissionOwner 实例！
                uriPermissions = new UriPermissionOwner(sr.ams, this);
            }
            return uriPermissions;
        }
```

## 2.6 grantUriPermissionFromOwner

授予 UriPermissionOwner owner 访问 uri 的 permission，这里的 UriPermissionOwner 是通过 IBinder token 转换而来！

```java
    @Override
    public void grantUriPermissionFromOwner(IBinder token, int fromUid, String targetPkg, Uri uri,
            final int modeFlags, int sourceUserId, int targetUserId) {
        //【1】计算下 targetUserId！
        targetUserId = mUserController.handleIncomingUser(Binder.getCallingPid(),
                Binder.getCallingUid(), targetUserId, false, ALLOW_FULL_ONLY,
                "grantUriPermissionFromOwner", null);
        synchronized(this) {
            //【*2.6.1.3】通过 token 找到对应的 UriPermissionOwner，如果没有就会抛出异常！
            UriPermissionOwner owner = UriPermissionOwner.fromExternalToken(token);
            if (owner == null) {
                throw new IllegalArgumentException("Unknown owner: " + token);
            }
            //【1.1】fromUid 表示该 grant 操作来自哪个 uid；
            // Binder.getCallingUid() 表示调用该方法的 uid，一般情况 fromUid 和 callingUid 应该是一样的；
            // 如果不一样，callingUid 必须是 system uid！
            if (fromUid != Binder.getCallingUid()) {
                if (Binder.getCallingUid() != Process.myUid()) {
                    throw new SecurityException("nice try");
                }
            }
            if (targetPkg == null) {
                throw new IllegalArgumentException("null target");
            }
            if (uri == null) {
                throw new IllegalArgumentException("null uri");
            }
            //【*2.2】继续处理 grant 操作！
            grantUriPermissionLocked(fromUid, targetPkg, new GrantUri(sourceUserId, uri, false),
                    modeFlags, owner, targetUserId);
        }
    }
```
可以看到，grantUriPermissionFromOwner 可以提供跨进程的支持，也就是说应用程序可以通过 grantUriPermissionFromOwner 来授予其他应用 uri 权限；

同时，我们还看到要调用该方法，必须要存在一个 UriPermissionOwner，我们去看下如何创建一个 UriPermissionOwner：

### 2.6.1 newUriPermissionOwner

同样的，AMS 提供了一个 newUriPermissionOwner 方法，来实现创建一个新的 UriPermissionOwner，其支持跨进程调用：

```java
    @Override
    public IBinder newUriPermissionOwner(String name) {
        enforceNotIsolatedCaller("newUriPermissionOwner");
        synchronized(this) {
            //【*2.6.1.1】创建一个新的 UriPermissionOwner；
            UriPermissionOwner owner = new UriPermissionOwner(this, name);
            //【*2.6.1.2】返回一个新的
            return owner.getExternalTokenLocked();
        }
    }
```

#### 2.6.1.1 new UriPermissionOwner

新建一个 UriPermissionOwner 实例：

```java
final class UriPermissionOwner {
    final ActivityManagerService service; // ams 对象！
    final Object owner; // 用于表示所有者，

    Binder externalToken; // 用于 token，跨进程的表示同一个 owner

    private ArraySet<UriPermission> mReadPerms; // 该 owner 持有的所有的 read uri permission
    private ArraySet<UriPermission> mWritePerms; // 该 owner 持有的所有的 write uri permission

    // 内部类，用于初始化 externalToken！
    class ExternalToken extends Binder {
        UriPermissionOwner getOwner() {
            return UriPermissionOwner.this;
        }
    }

    UriPermissionOwner(ActivityManagerService service, Object owner) {
        this.service = service;
        this.owner = owner;
    }
```

#### 2.6.1.2 getExternalTokenLocked

返回 UriPermissionOwner 的 Binder 对象 token：

```java
    Binder getExternalTokenLocked() {
        if (externalToken == null) {
            externalToken = new ExternalToken();
        }
        return externalToken;
    }
```
不多说了！

#### 2.6.1.3 fromExternalToken

UriPermissionOwner 有一个方法，通过 token 找到对应的 UriPermissionOwner 实例：
```java
    static UriPermissionOwner fromExternalToken(IBinder token) {
        if (token instanceof ExternalToken) {
            return ((ExternalToken)token).getOwner();
        }
        return null;
    }
```

不多说了！

### 2.6.2 调用时机

我们来看下该方法的具体调用，用 VoiceInteraction 举个例子！

#### 2.6.2.1 new VoiceInteractionSessionConnection

首先，在创建 VoiceInteractionSessionConnection 时，我们为这个 ComponentName component 对应的组件创建了一个 UriPermissionOwner，并返回了其 Binder token！

```java
    public VoiceInteractionSessionConnection(Object lock, ComponentName component, int user,
            Context context, Callback callback, int callingUid, Handler handler) {
        mLock = lock;
        mSessionComponentName = component;
        mUser = user;
        mContext = context;
        mCallback = callback;
        mCallingUid = callingUid;
        mHandler = handler;
        mAm = ActivityManagerNative.getDefault();
        ... ... ...
        IBinder permOwner = null;
        try {
            //【1】为这个 ComponentName component 对应的组件创建了一个 UriPermissionOwner，
            // 并返回了其 Binder token！
            permOwner = mAm.newUriPermissionOwner("voicesession:"
                    + component.flattenToShortString());
        } catch (RemoteException e) {
            Slog.w("voicesession", "AM dead", e);
        }
        mPermissionOwner = permOwner;
        ... ... ...
    }
```

#### 2.6.2.2 deliverSessionDataLocked - VoiceInteractionSessionConnection

在 VoiceInteractionSessionConnection 的 deliverSessionDataLocked 分发语音事务是，会授予 uri permission！

```java
    private void deliverSessionDataLocked(AssistDataForActivity assistDataForActivity) {
        Bundle assistData = assistDataForActivity.data.getBundle(
                VoiceInteractionSession.KEY_DATA);
        AssistStructure structure = assistDataForActivity.data.getParcelable(
                VoiceInteractionSession.KEY_STRUCTURE);
        AssistContent content = assistDataForActivity.data.getParcelable(
                VoiceInteractionSession.KEY_CONTENT);
        int uid = assistDataForActivity.data.getInt(Intent.EXTRA_ASSIST_UID, -1);
        if (uid >= 0 && content != null) {
            Intent intent = content.getIntent();
            if (intent != null) {
                ClipData data = intent.getClipData();
                if (data != null && Intent.isAccessUriMode(intent.getFlags())) {
                    //【1】授予权限！
                    grantClipDataPermissions(data, intent.getFlags(), uid,
                            mCallingUid, mSessionComponentName.getPackageName());
                }
            }
            ClipData data = content.getClipData();
            if (data != null) {
                //【1】授予权限！
                grantClipDataPermissions(data,
                        Intent.FLAG_GRANT_READ_URI_PERMISSION,
                        uid, mCallingUid, mSessionComponentName.getPackageName());
            }
        }
        try {
            mSession.handleAssist(assistData, structure, content,
                    assistDataForActivity.activityIndex, assistDataForActivity.activityCount);
        } catch (RemoteException e) {
        }
    }
```
这里我们省略掉中间过程，大家可以请自去看源码：

```java
grantClipDataPermissions(...)  -> grantClipDataItemPermission(...) -> grantUriPermission(...)
```

#### 2.6.2.3 grantUriPermission - VoiceInteractionSessionConnection

grant 指定的 uid uri permission：
```java
    void grantUriPermission(Uri uri, int mode, int srcUid, int destUid, String destPkg) {
        if (!"content".equals(uri.getScheme())) {
            return;
        }
        long ident = Binder.clearCallingIdentity();
        try {
            //【1】这里首先检查了 srcUid 是否已经有权限，这里我们在 check 的时候有讲过！
            // 如果该方法不抛出异常，那就可以 grant！
            mAm.checkGrantUriPermission(srcUid, null, ContentProvider.getUriWithoutUserId(uri),
                    mode, ContentProvider.getUserIdFromUri(uri, UserHandle.getUserId(srcUid)));

            int sourceUserId = ContentProvider.getUserIdFromUri(uri, mUser);
            uri = ContentProvider.getUriWithoutUserId(uri);

            //【*1.6】授予 uri 权限！
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
这里传入的 mPermissionOwner 是我们前面创建的 Binder token！

### 2.6.3 跨进程通信


ActivityManagerProxy 中提供了相关的接口：

- **ActivityManagerProxy**

```java
    public void grantUriPermissionFromOwner(IBinder owner, int fromUid, String targetPkg,
            Uri uri, int mode, int sourceUserId, int targetUserId) throws RemoteException {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IActivityManager.descriptor);
        data.writeStrongBinder(owner);
        data.writeInt(fromUid);
        data.writeString(targetPkg);
        uri.writeToParcel(data, 0);
        data.writeInt(mode);
        data.writeInt(sourceUserId);
        data.writeInt(targetUserId);
        //【1】这里我改正了错误：
        mRemote.transact(GRANT_URI_PERMISSION_FROM_OWNER_TRANSACTION, data, reply, 0);
        reply.readException();
        data.recycle();
        reply.recycle();
    }
```
7.1.1 的源码有一个问题：

```java
GRANT_URI_PERMISSION_FROM_OWNER_TRANSACTION 被写成了 GRANT_URI_PERMISSION_TRANSACTION
```
我个人觉得是一个错误！

- **ActivityManagerNative**

```java
        case GRANT_URI_PERMISSION_FROM_OWNER_TRANSACTION: {
            data.enforceInterface(IActivityManager.descriptor);
            IBinder owner = data.readStrongBinder();
            int fromUid = data.readInt();
            String targetPkg = data.readString();
            Uri uri = Uri.CREATOR.createFromParcel(data);
            int mode = data.readInt();
            int sourceUserId = data.readInt();
            int targetUserId = data.readInt();
            grantUriPermissionFromOwner(owner, fromUid, targetPkg, uri, mode, sourceUserId,
                    targetUserId);
            reply.writeNoException();
            return true;
        }
```

# 3 grantPermission - PackageManagerService

权限的授予核心接口是在 PackageManagerService 中，包括运行时和安装时权限：

## 3.1 grantPermissionsLPw - 权限更新时调用重新授予

在更新过权限后，会再次授予应用权限，因为可能有些权限已经失效并且被移除！

这里的 replace 为 true，由于我们是更新所有的 pacakge，所以 packageOfInterest 均为 null，参数 pkg 是需要更新权限授予信息的应用！

该方法主要是在 updatePermissionsLPw 方法

```java
    private void grantPermissionsLPw(PackageParser.Package pkg, boolean replace,
            String packageOfInterest) {
        //【1】如果该应用的 PackageSetting 没有，那么不处理！
        final PackageSetting ps = (PackageSetting) pkg.mExtras;
        if (ps == null) {
            return;
        }

        Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "grantPermissions");
        
        //【2】获得该 package 的权限管理对象，此时这个权限管理对象是解析上次安装信息时恢复的！
        // 我们将其保存到 origPermissions 中，作为原始数据！
        PermissionsState permissionsState = ps.getPermissionsState();
        PermissionsState origPermissions = permissionsState;

        final int[] currentUserIds = UserManagerService.getInstance().getUserIds();

        boolean runtimePermissionsRevoked = false;
        int[] changedRuntimePermissionUserIds = EMPTY_INT_ARRAY;

        boolean changedInstallPermission = false;

        //【3】如果 replace 的值为 true，进入这里！
        if (replace) {
            // 设置 installPermissionsFixed 为 true，因为下面会对权限信息调整！
            ps.installPermissionsFixed = false;
            if (!ps.isSharedUser()) {
                //【3.1】如果该 package 不是共享 uid 的，那就重置 permissionsState，但是会将
                // 原始信息 copy到 origPermissions 中！
                origPermissions = new PermissionsState(permissionsState);
                permissionsState.reset();

            } else {
                //【3.2】如果该 package 是共享 uid 的，那么我们会拒绝那些不用的共享 uid 相关权限信息！
                changedRuntimePermissionUserIds = revokeUnusedSharedUserPermissionsLPw(
                        ps.sharedUser, UserManagerService.getInstance().getUserIds());
                if (!ArrayUtils.isEmpty(changedRuntimePermissionUserIds)) {
                    runtimePermissionsRevoked = true;
                }
            }
        }

        permissionsState.setGlobalGids(mGlobalGids);

        //【4】处理该 package 请求的所有权限！
        final int N = pkg.requestedPermissions.size();
        for (int i=0; i < N; i++) {
            final String name = pkg.requestedPermissions.get(i);
            final BasePermission bp = mSettings.mPermissions.get(name);

            if (DEBUG_INSTALL) {
                Log.i(TAG, "Package " + pkg.packageName + " checking " + name + ": " + bp);
            }
            //【4.1】如果该权限没有定义，那么跳过！！
            if (bp == null || bp.packageSetting == null) {
                if (packageOfInterest == null || packageOfInterest.equals(pkg.packageName)) {
                    Slog.w(TAG, "Unknown permission " + name
                            + " in package " + pkg.packageName);
                }
                continue;
            }

            final String perm = bp.name;
            boolean allowedSig = false;
            int grant = GRANT_DENIED; // 用于保存每个权限的授予情况！

            //【4.2】如果申请的权限设置了 PROTECTION_FLAG_APPOP 标志位，
            // 我们会将其添加到 mAppOpPermissionPackages 进行监控！
            if ((bp.protectionLevel & PermissionInfo.PROTECTION_FLAG_APPOP) != 0) {
                ArraySet<String> pkgs = mAppOpPermissionPackages.get(bp.name);
                if (pkgs == null) {
                    pkgs = new ArraySet<>();
                    mAppOpPermissionPackages.put(bp.name, pkgs);
                }
                pkgs.add(pkg.packageName);
            }

            //【4.3】计算权限的基本类型，并判断应用是否支持运行时权限！！！
            final int level = bp.protectionLevel & PermissionInfo.PROTECTION_MASK_BASE;
            final boolean appSupportsRuntimePermissions = pkg.applicationInfo.targetSdkVersion
                    >= Build.VERSION_CODES.M;

            //【4.4】根据权限的基本类型，判断权限的授予情况！！
            switch (level) {
                case PermissionInfo.PROTECTION_NORMAL: { 
                    //【4.4.1】基本类型为 normal，为安装时权限，默认授予！
                    grant = GRANT_INSTALL;
                } break;

                case PermissionInfo.PROTECTION_DANGEROUS: {
                    //【4.4.2】基本类型为 dangerous，那么我们要分不同的情况！！
                    if (!appSupportsRuntimePermissions && !Build.PERMISSIONS_REVIEW_REQUIRED) {
                        //【4.4.2.1】如果 legacy apps，同时也不需要在组件启动之前重新 review 权限
                        // 说明应用的 targetSDKVersion < 23 是 legacy apps，那么运行时权限视为安装时权限；
                        grant = GRANT_INSTALL;
                        
                    } else if (origPermissions.hasInstallPermission(bp.name)) {
                        //【4.4.2.2】如果应用支持安装是权限，或者需要在组件启动之前重新 review 权限
                        // 并且该运行时权限之前是属于安装时权限，那么就要升级为运行时权限（系统升级会出现）
                        grant = GRANT_UPGRADE;
                        
                    } else if (mPromoteSystemApps
                            && isSystemApp(ps)
                            && mExistingSystemPackages.contains(ps.name)) {
                        //【4.4.2.3】如果是从 API 22 升级上来的话，对于 Promote System Apps，
                        // 那么安装时权限就要升级为运行时权限；
                        grant = GRANT_UPGRADE;
                        
                    } else {
                        //【4.4.2.4】其他情况，直接视为安装时权限！
                        grant = GRANT_RUNTIME;
                        
                    }
                } break;

                case PermissionInfo.PROTECTION_SIGNATURE: {
                    //【4.4.3】基本类型为 signature 的权限，需要校验签名！！
                    //【*1.2.2-important】校验签名是否匹配，如果是的话，是为安装时权限！
                    allowedSig = grantSignaturePermission(perm, pkg, bp, origPermissions);
                    if (allowedSig) {
                        // 如果签名校验通过，直接视为安装时权限！
                        grant = GRANT_INSTALL;
                    }
                } break;
            }

            if (DEBUG_INSTALL) {
                Log.i(TAG, "Package " + pkg.packageName + " granting " + perm);
            }

            //【4.5】处理授予情况，如果授予情况不是拒绝 GRANT_DENIED，进入 IF 分支；
            // 如果授予情况是拒绝 GRANT_DENIED，进入 ELSE 分支！
            if (grant != GRANT_DENIED) {
                if (!isSystemApp(ps) && ps.installPermissionsFixed) {
                    //【4.5.1】如果这是一个已经被安装了的非系统的应用，同时本次申请的权限之前不属于安装时权限
                    // 除非该权限属于 new platform permission 不然的话，我们是不会授予他这个权限的；
                    if (!allowedSig && !origPermissions.hasInstallPermission(perm)) {
                        if (!isNewPlatformPermissionForPackage(perm, pkg)) {
                            grant = GRANT_DENIED;
                        }
                    }
                }
                //【4.5.2】处理权限的授予状态！
                switch (grant) {
                    case GRANT_INSTALL: { // 处理安装时权限；
                        //【4.5.2.1.1】首先，判断该权限之前是否是运行时权限，如果是的话，那就要撤销之前的授予！
                        for (int userId : UserManagerService.getInstance().getUserIds()) {
                            if (origPermissions.getRuntimePermissionState(
                                    bp.name, userId) != null) {
                                // 撤销权限，并取消所有的标志位；
                                origPermissions.revokeRuntimePermission(bp, userId);
                                origPermissions.updatePermissionFlags(bp, userId,
                                      PackageManager.MASK_PERMISSION_FLAGS, 0);
                                // 将运行时权限授予状态发生变化的 userId 记录下来，后面用于持久化数据！
                                changedRuntimePermissionUserIds = ArrayUtils.appendInt(
                                        changedRuntimePermissionUserIds, userId);
                            }
                        }
                        //【4.5.2.1.2】授予安装时权限；
                        if (permissionsState.grantInstallPermission(bp) !=
                                PermissionsState.PERMISSION_OPERATION_FAILURE) {
                            changedInstallPermission = true;
                        }
                    } break;

                    case GRANT_RUNTIME: { // 处理运行时权限；
                        //【4.5.2.1.3】授予之前被授予的运行时权限；
                        for (int userId : UserManagerService.getInstance().getUserIds()) {
                            //【4.5.2.1.3.1】获得每个 userId 下的该运行时权限对象和其 flags；
                            PermissionState permissionState = origPermissions
                                    .getRuntimePermissionState(bp.name, userId);
                            int flags = permissionState != null
                                    ? permissionState.getFlags() : 0;
                            
                            //【4.5.2.1.3.2】如果本次授予的权限是已经申请的，那么我们就再次授予！
                            if (origPermissions.hasRuntimePermission(bp.name, userId)) {
                                //【4.5.2.1.3.2.1】授予运行时权限，如果失败的话，意味着本次无法授予，那就更新
                                // changedRuntimePermissionUserIds 列表；
                                if (permissionsState.grantRuntimePermission(bp, userId) ==
                                        PermissionsState.PERMISSION_OPERATION_FAILURE) {
                                    changedRuntimePermissionUserIds = ArrayUtils.appendInt(
                                            changedRuntimePermissionUserIds, userId);
                                }
                                //【4.5.2.1.3.2.2】如果应用支持运行时权限，该权限之前是需要 review 的
                                // 那么这里会取消 review 的标志位！
                                if (Build.PERMISSIONS_REVIEW_REQUIRED
                                        && appSupportsRuntimePermissions
                                        && (flags & PackageManager
                                                .FLAG_PERMISSION_REVIEW_REQUIRED) != 0) {
                                    flags &= ~PackageManager.FLAG_PERMISSION_REVIEW_REQUIRED;
                                    // 标志位变了，那就更新 changedRuntimePermissionUserIds 列表；
                                    changedRuntimePermissionUserIds = ArrayUtils.appendInt(
                                            changedRuntimePermissionUserIds, userId);
                                }
    
                            } else if (Build.PERMISSIONS_REVIEW_REQUIRED
                                    && !appSupportsRuntimePermissions) {
                                //【4.5.2.1.3.2】如果应用不支持运行时权限，但是系统需要权限 review
                                // 每一个被授予的新的运行时权限，都要被 review！
                                // 如果是系统权限，强制增加 FLAG_PERMISSION_REVIEW_REQUIRED 标志位
                                if (PLATFORM_PACKAGE_NAME.equals(bp.sourcePackage)) {
                                    if ((flags & FLAG_PERMISSION_REVIEW_REQUIRED) == 0) {
                                        flags |= FLAG_PERMISSION_REVIEW_REQUIRED;
                                        // 标志位变了，那就更新 changedRuntimePermissionUserIds 列表；
                                        changedRuntimePermissionUserIds = ArrayUtils.appendInt(
                                                changedRuntimePermissionUserIds, userId);
                                    }
                                }
                                // 授予这个新的运行时权限，如果授予失败，更新 changedRuntimePermissionUserIds 列表；
                                if (permissionsState.grantRuntimePermission(bp, userId)
                                        != PermissionsState.PERMISSION_OPERATION_FAILURE) {
                                    changedRuntimePermissionUserIds = ArrayUtils.appendInt(
                                            changedRuntimePermissionUserIds, userId);
                                }
                            }
                            //【4.5.2.1.3.2.1】更新权限的 flags！
                            permissionsState.updatePermissionFlags(bp, userId, flags, flags);

                        }
                    } break;

                    case GRANT_UPGRADE: { // 处理安装时权限升级为运行时权限
                        //【4.5.2.1.4】授予安装时升级为运行时的权限；
                        // 获得权限管理对象和 flags；
                        PermissionState permissionState = origPermissions
                                .getInstallPermissionState(bp.name);
                        final int flags = permissionState != null ? permissionState.getFlags() : 0;

                        //【4.5.2.1.4.1】撤销之前的安装时权限，并清空 flags；
                        if (origPermissions.revokeInstallPermission(bp)
                                != PermissionsState.PERMISSION_OPERATION_FAILURE) {
                            origPermissions.updatePermissionFlags(bp, UserHandle.USER_ALL,
                                    PackageManager.MASK_PERMISSION_FLAGS, 0);
                            changedInstallPermission = true;
                        }

                        //【4.5.2.1.4.2】如果该权限没有设置 FLAG_PERMISSION_REVOKE_ON_UPGRADE，
                        // 表示在升级后可以授予，那么我们就授予运行时权限；
                        if ((flags & PackageManager.FLAG_PERMISSION_REVOKE_ON_UPGRADE) == 0) {
                            for (int userId : currentUserIds) {
                                if (permissionsState.grantRuntimePermission(bp, userId) !=
                                        PermissionsState.PERMISSION_OPERATION_FAILURE) {
                                    // 将旧的 flags 更新到运行时权限；
                                    permissionsState.updatePermissionFlags(bp, userId,
                                            flags, flags);
                                    // 标志位变了，那就更新 changedRuntimePermissionUserIds 列表；
                                    changedRuntimePermissionUserIds = ArrayUtils.appendInt(
                                            changedRuntimePermissionUserIds, userId);
                                }
                            }
                        }

                    } break;

                    default: {
                        if (packageOfInterest == null
                                || packageOfInterest.equals(pkg.packageName)) {
                            Slog.w(TAG, "Not granting permission " + perm
                                    + " to package " + pkg.packageName
                                    + " because it was previously installed without");
                        }
                    } break;
                }
            } else {
                //【4.6】处理撤销情况，撤销该权限，同时清空所有的标志位；
                if (permissionsState.revokeInstallPermission(bp) !=
                        PermissionsState.PERMISSION_OPERATION_FAILURE) {
                    // 清空标志位；
                    permissionsState.updatePermissionFlags(bp, UserHandle.USER_ALL,
                            PackageManager.MASK_PERMISSION_FLAGS, 0);
                    changedInstallPermission = true;
                    Slog.i(TAG, "Un-granting permission " + perm
                            + " from package " + pkg.packageName
                            + " (protectionLevel=" + bp.protectionLevel
                            + " flags=0x" + Integer.toHexString(pkg.applicationInfo.flags)
                            + ")");
                } else if ((bp.protectionLevel&PermissionInfo.PROTECTION_FLAG_APPOP) == 0) {
                    //【4.6.1】如果撤销失败，但是该权限和一个 appOp 相关联，这种情况可以不处理。因为
                    // 对于这类权限，我们不处理，因为我们会通过 appOps 进行控制，通过 ui 给用户选择；
                    if (packageOfInterest == null || packageOfInterest.equals(pkg.packageName)) {
                        Slog.w(TAG, "Not granting permission " + perm
                                + " to package " + pkg.packageName
                                + " (protectionLevel=" + bp.protectionLevel
                                + " flags=0x" + Integer.toHexString(pkg.applicationInfo.flags)
                                + ")");
                    }
                }
            }
        }
        if ((changedInstallPermission || replace) && !ps.installPermissionsFixed &&
                !isSystemApp(ps) || isUpdatedSystemApp(ps)){
            // 这是我们第一次听说过这个包，所以我们现在选择的权限是固定的，直到明确更改为止。
            // 没看懂这段逻辑...
            ps.installPermissionsFixed = true;
        }

        //【5】持久化变化调整后的运行时权限数据！
        for (int userId : changedRuntimePermissionUserIds) {
            mSettings.writeRuntimePermissionsForUserLPr(userId, runtimePermissionsRevoked);
        }

        Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
    }
```

### 3.2.1 grantSignaturePermission - Signature 权限授予核心接口

grantSignaturePermission 方法用于判断签名是否能校验通过，我们在这里先忽略掉签名校验的过程，假设要么校验成功，要么校验失败！

```java
    private boolean grantSignaturePermission(String perm, PackageParser.Package pkg,
            BasePermission bp, PermissionsState origPermissions) {
        boolean allowed;
        //【1】判断应用的签名和权限定义者签名或者平台签名是否匹配！
        allowed = (compareSignatures(
                bp.packageSetting.signatures.mSignatures, pkg.mSignatures)
                        == PackageManager.SIGNATURE_MATCH)
                || (compareSignatures(mPlatformPackage.mSignatures, pkg.mSignatures)
                        == PackageManager.SIGNATURE_MATCH);
                        
        //【2】如果和权限定义者签名或者平台签名不匹配，并且该权限是 PRIVILEGED 的，进行进一步的判断！
        if (!allowed && (bp.protectionLevel
                & PermissionInfo.PROTECTION_FLAG_PRIVILEGED) != 0) {
            
            if (isSystemApp(pkg)) { // 对于 PRIVILEGED 类型的权限，首先必须是系统应用！
                //【2.1】如果该应用是一个被更新过的系统应用，那么只有更新前的旧 apk 被授予了这个权限，
                // 新的 apk 才能被授予这个权限！
                if (pkg.isUpdatedSystemApp()) {
                    // 获得更新前的安装数据(system)！
                    final PackageSetting sysPs = mSettings
                            .getDisabledSystemPkgLPr(pkg.packageName);

                    if (sysPs != null && sysPs.getPermissionsState().hasInstallPermission(perm)) {
                        //【2.1.1】如果更新前，其已经被授予了该权限，并且其是一个特权应用，
                        // 那么本次也允许授予！！
                        if (sysPs.isPrivileged()) {
                            allowed = true;
                        }
                    } else {
                        //【2.1.2】如果更新前，并没有被授予该权限，该权限是更新后新申请的权限！
                        // 那就要看之前是否有请求该权限，只有请求了该权限，才允许授予；
                        if (sysPs != null && sysPs.pkg != null && sysPs.isPrivileged()) {
                            for (int j = 0; j < sysPs.pkg.requestedPermissions.size(); j++) {
                                if (perm.equals(sysPs.pkg.requestedPermissions.get(j))) {
                                    allowed = true;
                                    break;
                                }
                            }
                        }
    
                        // 此外，如果系统映像上的特权父包或其父包任何子代请求特权权限，则更新的子包也可以获得该许可。
                        if (pkg.parentPackage != null) {
                            final PackageSetting disabledSysParentPs = mSettings
                                    .getDisabledSystemPkgLPr(pkg.parentPackage.packageName);
                            if (disabledSysParentPs != null && disabledSysParentPs.pkg != null
                                    && disabledSysParentPs.isPrivileged()) {
                                if (isPackageRequestingPermission(disabledSysParentPs.pkg, perm)) {
                                    allowed = true;
                                } else if (disabledSysParentPs.pkg.childPackages != null) {
                                    final int count = disabledSysParentPs.pkg.childPackages.size();
                                    for (int i = 0; i < count; i++) {
                                        PackageParser.Package disabledSysChildPkg =
                                                disabledSysParentPs.pkg.childPackages.get(i);
                                        if (isPackageRequestingPermission(disabledSysChildPkg,
                                                perm)) {
                                            allowed = true;
                                            break;
                                        }
                                    }
                                }
                            }
                        }
                    }
                } else {
                    //【2.2】如果该应用是一个没有被更新过的系统应用，那就要判读其是否是特权应用！
                    allowed = isPrivilegedApp(pkg);
                }
            }
        }
        
        //【3】经过了上面的判断，依然不允许，接着进行下一步的判断，下面会根据额外的标志位继续判断！
        if (!allowed) {
            if (!allowed && (bp.protectionLevel
                    & PermissionInfo.PROTECTION_FLAG_PRE23) != 0
                    && pkg.applicationInfo.targetSdkVersion < Build.VERSION_CODES.M) {
                //【3.1】PROTECTION_FLAG_PRE23 指定此权限会自动授予低于 M（在引入运行时权限之前）的 API 级别的应用程序。
                // 如果应用的 targetSdkVersion 是小于 M 的，那么我们授予他签名权限！
                allowed = true;
            }

            if (!allowed && (bp.protectionLevel & PermissionInfo.PROTECTION_FLAG_INSTALLER) != 0
                    && pkg.packageName.equals(mRequiredInstallerPackage)) {
                //【3.2】PROTECTION_FLAG_INSTALLER 指定此权限可以自动授予包软件应用程序。
                // 而此时的 apk 就是 packageInstaller，那么授予他签名权限！
                allowed = true;
            }

            if (!allowed && (bp.protectionLevel & PermissionInfo.PROTECTION_FLAG_VERIFIER) != 0
                    && pkg.packageName.equals(mRequiredVerifierPackage)) {
                //【3.3】PROTECTION_FLAG_INSTALLER 指定此权限可以被自动授予给验证软件包的系统应用程序。
                // 而此时的 apk 就是 VerifierPackage，那么授予他签名权限！
                allowed = true;
            }

            if (!allowed && (bp.protectionLevel
                    & PermissionInfo.PROTECTION_FLAG_PREINSTALLED) != 0
                    && isSystemApp(pkg)) {
                //【3.4】PROTECTION_FLAG_PREINSTALLED 指定此权限可以自动授予 system 分区预先安装的任何应用程序
                // 而此时的 apk 就是 system app，那么授予他签名权限！
                allowed = true;
            }

            if (!allowed && (bp.protectionLevel
                    & PermissionInfo.PROTECTION_FLAG_DEVELOPMENT) != 0) {
                //【3.4】PROTECTION_FLAG_DEVELOPMENT 指定此权限可以授予（可选）给开发应用程序
                // 而上一次应用是被授予了这个权限，那么一次也不被授予
                allowed = origPermissions.hasInstallPermission(perm);
            }

            if (!allowed && (bp.protectionLevel & PermissionInfo.PROTECTION_FLAG_SETUP) != 0
                    && pkg.packageName.equals(mSetupWizardPackage)) {
                //【3.5】PROTECTION_FLAG_SETUP 指定此权限可以被自动授予给开机向导应用程序
                // 而此时的 apk 就是开机向导，那么授予他签名权限！.
                allowed = true;
            }
        }
        return allowed;
    }
```
可以看到，这里对 Signature 类型的权限做了详细的处理！

## 3.2 grantRuntimePermission - 运行时权限授予核心接口

授予运行时权限，其主要是在应用运行的时候，显示一个伪弹窗的 acitivity

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
            if (sb == null) { //【6】升级包未安装过，异常！
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

## 3.3 grantRequestedRuntimePermissions - 安装时授予运行时权限

在签名分析 pkg install 的时候，我们有看 handlePackagePostInstall 方法：

```java
    private void handlePackagePostInstall(PackageInstalledInfo res, boolean grantPermissions,
            boolean killApp, String[] grantedPermissions,
            boolean launchedForRestore, String installerPackage,
            IPackageInstallObserver2 installObserver) {
        if (res.returnCode == PackageManager.INSTALL_SUCCEEDED) {
            // Send the removed broadcasts
            if (res.removedInfo != null) {
                res.removedInfo.sendPackageRemovedBroadcasts(killApp);
            }

            //【1】如果我们指定了在安装时授予其运行时权限，那么就会调用 grantRequestedRuntimePermissions 方法：
            if (grantPermissions && res.pkg.applicationInfo.targetSdkVersion
                    >= Build.VERSION_CODES.M) {
                grantRequestedRuntimePermissions(res.pkg, res.newUsers, grantedPermissions);
            }
```
继续来分析：

```java
    private void grantRequestedRuntimePermissions(PackageParser.Package pkg, int[] userIds,
            String[] grantedPermissions) {
        //【3.3.1】授予每个设备用户下的运行时权限！
        for (int userId : userIds) {
            grantRequestedRuntimePermissionsForUser(pkg, userId, grantedPermissions);
        }

        // We could have touched GID membership, so flush out packages.list
        synchronized (mPackages) {
            mSettings.writePackageListLPr();
        }
    }
```
继续跟踪：

### 3.3.1 grantRequestedRuntimePermissionsForUser

下面的代码很简单：

```java
    private void grantRequestedRuntimePermissionsForUser(PackageParser.Package pkg, int userId,
            String[] grantedPermissions) {
        SettingBase sb = (SettingBase) pkg.mExtras;
        if (sb == null) {
            return;
        }

        PermissionsState permissionsState = sb.getPermissionsState();

        //【1】对于 installer 他没有权限去改变 SYSTEM_FIXED 和 POLICY_FIXED 的权限状态！
        final int immutableFlags = PackageManager.FLAG_PERMISSION_SYSTEM_FIXED
                | PackageManager.FLAG_PERMISSION_POLICY_FIXED;

        for (String permission : pkg.requestedPermissions) {
            final BasePermission bp;
            synchronized (mPackages) {
                bp = mSettings.mPermissions.get(permission);
            }
            if (bp != null && (bp.isRuntime() || bp.isDevelopment())
                    && (grantedPermissions == null
                           || ArrayUtils.contains(grantedPermissions, permission))) {
                final int flags = permissionsState.getPermissionFlags(permission, userId);
                //【2】禁止修改！
                if ((flags & immutableFlags) == 0) {
                    //【×3.2】授予运行时权限！
                    grantRuntimePermission(pkg.packageName, permission, userId);
                }
            }
        }
    }

```