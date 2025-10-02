# PMS 第 6 篇 - PMS_READY 阶段
title: PMS 第 6 篇 - PMS_READY 阶段
date: 2018/04/28
categories:
- AndroidFramework源码分析
- PackageManager包管理
tags: PackageManager包管理
---

[toc]

基于 Android 7.1.1 源码分析 PackageManagerService 的架构和逻辑实现，本文是作者原创，转载请说明出处！

# 0 综述

最后我们会进入 PMS 初始化的最后阶段：

```java
            ... ... ... ...// 第五阶段
            EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_PMS_READY,
                    SystemClock.uptimeMillis());
            
            //【1】获得 PMS 需要一些包名，比如 PackageInstaller，IntentFilterVerifier 等等！
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
            
            //【2】建立 PackageInstallerService 服务对象，用于 package 的安装！
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

        // 执行 GC 操作，回收系统资源！
        Runtime.getRuntime().gc();

        // The initial scanning above does many calls into installd while
        // holding the mPackages lock, but we're mostly interested in yelling
        // once we have a booted system.
        mInstaller.setWarnIfHeld(mPackages);

        //【3】将自身加入本地服务管理中，方便系统自身访问！
        LocalServices.addService(PackageManagerInternal.class, new PackageManagerInternalImpl());
        
    }
```

这个阶段逻辑很简单，我们不多分析！

# 1 获得一些 PMS 会用到的应用的包名

## 1.1 PackageMS.getRequiredButNotReallyRequiredVerifierLPr

```java
    private @Nullable String getRequiredButNotReallyRequiredVerifierLPr() {
        // android.intent.action.PACKAGE_NEEDS_VERIFICATION
        final Intent intent = new Intent(Intent.ACTION_PACKAGE_NEEDS_VERIFICATION);

        final List<ResolveInfo> matches = queryIntentReceiversInternal(intent, PACKAGE_MIME_TYPE,
                MATCH_SYSTEM_ONLY | MATCH_DIRECT_BOOT_AWARE | MATCH_DIRECT_BOOT_UNAWARE,
                UserHandle.USER_SYSTEM);
        if (matches.size() == 1) {
            return matches.get(0).getComponentInfo().packageName;
        } else if (matches.size() == 0) {
            Log.e(TAG, "There should probably be a verifier, but, none were found");
            return null;
        }
        throw new RuntimeException("There must be exactly one verifier; found " + matches);
    }
```


## 1.2 PackageMS.getRequiredInstallerLPr

获得 packageInstaller 的包名！
```java
    private @NonNull String getRequiredInstallerLPr() {
        final Intent intent = new Intent(Intent.ACTION_INSTALL_PACKAGE);
        intent.addCategory(Intent.CATEGORY_DEFAULT);
        intent.setDataAndType(Uri.fromFile(new File("foo.apk")), PACKAGE_MIME_TYPE);

        final List<ResolveInfo> matches = queryIntentActivitiesInternal(intent, PACKAGE_MIME_TYPE,
                MATCH_SYSTEM_ONLY | MATCH_DIRECT_BOOT_AWARE | MATCH_DIRECT_BOOT_UNAWARE,
                UserHandle.USER_SYSTEM);
        if (matches.size() == 1) {
            ResolveInfo resolveInfo = matches.get(0);
            if (!resolveInfo.activityInfo.applicationInfo.isPrivilegedApp()) {
                throw new RuntimeException("The installer must be a privileged app");
            }
            return matches.get(0).getComponentInfo().packageName;
        } else {
            throw new RuntimeException("There must be exactly one installer; found " + matches);
        }
    }
```

## 1.3 PackageMS.getRequiredUninstallerLPr

```java
    private @NonNull String getRequiredUninstallerLPr() {
        final Intent intent = new Intent(Intent.ACTION_UNINSTALL_PACKAGE);
        intent.addCategory(Intent.CATEGORY_DEFAULT);
        intent.setData(Uri.fromParts(PACKAGE_SCHEME, "foo.bar", null));

        final ResolveInfo resolveInfo = resolveIntent(intent, null,
                MATCH_SYSTEM_ONLY | MATCH_DIRECT_BOOT_AWARE | MATCH_DIRECT_BOOT_UNAWARE,
                UserHandle.USER_SYSTEM);
        if (resolveInfo == null ||
                mResolveActivity.name.equals(resolveInfo.getComponentInfo().name)) {
            throw new RuntimeException("There must be exactly one uninstaller; found "
                    + resolveInfo);
        }
        return resolveInfo.getComponentInfo().packageName;
    }
```

## 1.4 PackageMS.getIntentFilterVerifierComponentNameLPr

```java
    private @NonNull ComponentName getIntentFilterVerifierComponentNameLPr() {
        final Intent intent = new Intent(Intent.ACTION_INTENT_FILTER_NEEDS_VERIFICATION);

        final List<ResolveInfo> matches = queryIntentReceiversInternal(intent, PACKAGE_MIME_TYPE,
                MATCH_SYSTEM_ONLY | MATCH_DIRECT_BOOT_AWARE | MATCH_DIRECT_BOOT_UNAWARE,
                UserHandle.USER_SYSTEM);
        ResolveInfo best = null;
        final int N = matches.size();
        for (int i = 0; i < N; i++) {
            final ResolveInfo cur = matches.get(i);
            final String packageName = cur.getComponentInfo().packageName;
            if (checkPermission(android.Manifest.permission.INTENT_FILTER_VERIFICATION_AGENT,
                    packageName, UserHandle.USER_SYSTEM) != PackageManager.PERMISSION_GRANTED) {
                continue;
            }

            if (best == null || cur.priority > best.priority) {
                best = cur;
            }
        }

        if (best != null) {
            return best.getComponentInfo().getComponentName();
        } else {
            throw new RuntimeException("There must be at least one intent filter verifier");
        }
    }
```

## 1.5 PackageMS.getRequiredSharedLibraryLPr


```java
    /**
     * This is a library that contains components apps can invoke. For
     * example, a services for apps to bind to, or standard chooser UI,
     * etc. This library is versioned and backwards compatible. Clients
     * should check its version via {@link android.ext.services.Version
     * #getVersionCode()} and avoid calling APIs added in later versions.
     *
     * @hide
     */
    public static final String SYSTEM_SHARED_LIBRARY_SERVICES = "android.ext.services";

    /**
     * This is a library that contains components apps can dynamically
     * load. For example, new widgets, helper classes, etc. This library
     * is versioned and backwards compatible. Clients should check its
     * version via {@link android.ext.shared.Version#getVersionCode()}
     * and avoid calling APIs added in later versions.
     *
     * @hide
     */
    public static final String SYSTEM_SHARED_LIBRARY_SHARED = "android.ext.shared";
```
```java
    mServicesSystemSharedLibraryPackageName = getRequiredSharedLibraryLPr(
            PackageManager.SYSTEM_SHARED_LIBRARY_SERVICES);
    mSharedSystemSharedLibraryPackageName = getRequiredSharedLibraryLPr(
            PackageManager.SYSTEM_SHARED_LIBRARY_SHARED);
```                        
```java
    private @NonNull String getRequiredSharedLibraryLPr(String libraryName) {
        synchronized (mPackages) {
            SharedLibraryEntry libraryEntry = mSharedLibraries.get(libraryName);
            if (libraryEntry == null) {
                throw new IllegalStateException("Missing required shared library:" + libraryName);
            }
            return libraryEntry.apk;
        }
    }

```
# 2 new PackageInstallerService - 创建安装服务对象

创建 PackageInstallerService 服务对象，用于应用的安装，关于安装，这个我们先不关注！ 

```java
    public PackageInstallerService(Context context, PackageManagerService pm) {
        mContext = context;
        mPm = pm; // pms 的应用！

        mInstallThread = new HandlerThread(TAG); // 一个线程，用于处理 install 操作！
        mInstallThread.start();

        mInstallHandler = new Handler(mInstallThread.getLooper()); // 该线程的 Handler！
        //【2.1】创建了一个 Callbacks 回调对象，用于处理事务的变化，通知需要监听事务变化的远程对象！
        mCallbacks = new Callbacks(mInstallThread.getLooper());

        // 用于保存安装事务项的文件对象！
        mSessionsFile = new AtomicFile(
                new File(Environment.getDataSystemDirectory(), "install_sessions.xml"));
        mSessionsDir = new File(Environment.getDataSystemDirectory(), "install_sessions");
        mSessionsDir.mkdirs();

        synchronized (mSessions) {
            //【2.2】读取安装事务！
            readSessionsLocked();

            reconcileStagesLocked(StorageManager.UUID_PRIVATE_INTERNAL, false /*isEphemeral*/);
            reconcileStagesLocked(StorageManager.UUID_PRIVATE_INTERNAL, true /*isEphemeral*/);

            final ArraySet<File> unclaimedIcons = newArraySet(
                    mSessionsDir.listFiles());

            // Ignore stages and icons claimed by active sessions
            for (int i = 0; i < mSessions.size(); i++) {
                final PackageInstallerSession session = mSessions.valueAt(i);
                unclaimedIcons.remove(buildAppIconFile(session.sessionId));
            }

            // Clean up orphaned icons
            for (File icon : unclaimedIcons) {
                Slog.w(TAG, "Deleting orphan icon " + icon);
                icon.delete();
            }
        }
    }
```
在 PackageInstallerService 有如下的集合，来保存安装事务！
```java
    // 所有的事务，有效和无效的
    @GuardedBy("mSessions")
    private final SparseBooleanArray mAllocatedSessions = new SparseBooleanArray();

    // 用于保存那些有效的安装事务！
    @GuardedBy("mSessions")
    private final SparseArray<PackageInstallerSession> mSessions = new SparseArray<>();

    // 用于保存那些无效的历史事务！
    @GuardedBy("mSessions")
    private final SparseArray<PackageInstallerSession> mHistoricalSessions = new SparseArray<>();
```
在创建 PackageInstallerService 的时候会调用 readSessionsLocked 读取重启之前的所有安装事务！

## 2.1 new Callback

Callback 本质上也是一个 Handler 对象，其 Looper 对象来自 mInstallThread.getLooper()，用于处理耗时操作：

```java
    private static class Callbacks extends Handler {
        public Callbacks(Looper looper) {
            super(looper);
        }
    }
```

Callback 用于处理和 session 相关的消息：
```java
        private static final int MSG_SESSION_CREATED = 1;
        private static final int MSG_SESSION_BADGING_CHANGED = 2;
        private static final int MSG_SESSION_ACTIVE_CHANGED = 3;
        private static final int MSG_SESSION_PROGRESS_CHANGED = 4;
        private static final int MSG_SESSION_FINISHED = 5;
```
本质上讲，Callback 只是这些消息的中转站，它会将消息发送给注册到其内部的远程回调接口：

```java
        private final RemoteCallbackList<IPackageInstallerCallback>
                mCallbacks = new RemoteCallbackList<>();
```

CallBack 内部提供了注册相关的接口：

```java
        public void register(IPackageInstallerCallback callback, int userId) {
            mCallbacks.register(callback, new UserHandle(userId));
        }

        public void unregister(IPackageInstallerCallback callback) {
            mCallbacks.unregister(callback);
        }
```
我们来看下，其实如何处理回调的，当有 Session 发生变化后，下面的接口会被触发调用！
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
```
这些接口会发送消息，处理仍然是在 CallBack 中：

```java
        @Override
        public void handleMessage(Message msg) {
            final int userId = msg.arg2;
            final int n = mCallbacks.beginBroadcast();
            for (int i = 0; i < n; i++) {
                final IPackageInstallerCallback callback = mCallbacks.getBroadcastItem(i);
                final UserHandle user = (UserHandle) mCallbacks.getBroadcastCookie(i);
                // TODO: dispatch notifications for slave profiles
                if (userId == user.getIdentifier()) {
                    try {
                        // 调用 invokeCallback 分发 session 变化的消息！
                        invokeCallback(callback, msg);
                    } catch (RemoteException ignored) {
                    }
                }
            }
            mCallbacks.finishBroadcast();
        }
```
invokeCallback 方法将 session 变化的消息会转发给所有注册到其内部的远程回调接口！
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


## 2.2 PackageInstallerService.readSessionsLocked[0]

readSessionsLocked 用来读取事务！
```java
    private void readSessionsLocked() {
        if (LOGD) Slog.v(TAG, "readSessionsLocked()");

        mSessions.clear();

        FileInputStream fis = null;
        try {
            fis = mSessionsFile.openRead();
            final XmlPullParser in = Xml.newPullParser();
            in.setInput(fis, StandardCharsets.UTF_8.name());

            int type;
            while ((type = in.next()) != END_DOCUMENT) {
                if (type == START_TAG) {
                    final String tag = in.getName();
                    if (TAG_SESSION.equals(tag)) { // session 标签，用户封装指定的安装事务！
                        //【2.2】读取安装事务，返回 PackageInstallerSession 对象！
                        final PackageInstallerSession session = readSessionLocked(in);

                        // 计算事务的有效性！
                        final long age = System.currentTimeMillis() - session.createdMillis;
                        final boolean valid;
                        // 如果大于 MAX_AGE_MILLIS 3 天，那就是无效的，否则是有效的！
                        if (age >= MAX_AGE_MILLIS) {
                            Slog.w(TAG, "Abandoning old session first created at "
                                    + session.createdMillis);
                            valid = false;
                        } else {
                            valid = true;
                        }

                        if (valid) {
                            // 有效的事务
                            mSessions.put(session.sessionId, session);
                        } else {
                            // 无效的事务，用于 debug，比如 dumpsys 等等！
                            mHistoricalSessions.put(session.sessionId, session);
                        }
                        // 所有的事务！
                        mAllocatedSessions.put(session.sessionId, true);
                    }
                }
            }
        } catch (FileNotFoundException e) {
            // Missing sessions are okay, probably first boot
        } catch (IOException | XmlPullParserException e) {
            Slog.wtf(TAG, "Failed reading install sessions", e);
        } finally {
            IoUtils.closeQuietly(fis);
        }
    }
```
判断一个事务是否有效的依据是，其实是否是在 3 天时间间隔之前创建的！
```java
    private static final long MAX_AGE_MILLIS = 3 * DateUtils.DAY_IN_MILLIS;
```

### 2.2.1 PackageInstallerService.readSessionsLocked[1]

我们看下解析单个事务的具体过程！

```java
    private PackageInstallerSession readSessionLocked(XmlPullParser in) throws IOException,
            XmlPullParserException {
        final int sessionId = readIntAttribute(in, ATTR_SESSION_ID); // sessionId 属性
        final int userId = readIntAttribute(in, ATTR_USER_ID); // userId 属性 
        final String installerPackageName = readStringAttribute(in, ATTR_INSTALLER_PACKAGE_NAME); // installerPackageName 属性
        final int installerUid = readIntAttribute(in, ATTR_INSTALLER_UID, mPm.getPackageUid(
                installerPackageName, PackageManager.MATCH_UNINSTALLED_PACKAGES, userId)); // installerUid 属性
        final long createdMillis = readLongAttribute(in, ATTR_CREATED_MILLIS); // createdMillis 属性
        final String stageDirRaw = readStringAttribute(in, ATTR_SESSION_STAGE_DIR); // sessionStageDir 属性
        final File stageDir = (stageDirRaw != null) ? new File(stageDirRaw) : null; 
        final String stageCid = readStringAttribute(in, ATTR_SESSION_STAGE_CID); // sessionStageCid 属性
        final boolean prepared = readBooleanAttribute(in, ATTR_PREPARED, true); // prepared 属性
        final boolean sealed = readBooleanAttribute(in, ATTR_SEALED); // sealed 属性

        //【2.2.1】创建了一个 SessionParams 对象，封装事务详细参数
        final SessionParams params = new SessionParams(
                SessionParams.MODE_INVALID);
        params.mode = readIntAttribute(in, ATTR_MODE); // mode 属性
        params.installFlags = readIntAttribute(in, ATTR_INSTALL_FLAGS); // installFlags 属性
        params.installLocation = readIntAttribute(in, ATTR_INSTALL_LOCATION); // installLocation 属性
        params.sizeBytes = readLongAttribute(in, ATTR_SIZE_BYTES); // sizeBytes 属性
        params.appPackageName = readStringAttribute(in, ATTR_APP_PACKAGE_NAME); // appPackageName 属性
        params.appIcon = readBitmapAttribute(in, ATTR_APP_ICON); // appIcon 属性
        params.appLabel = readStringAttribute(in, ATTR_APP_LABEL); // appLabel 属性
        params.originatingUri = readUriAttribute(in, ATTR_ORIGINATING_URI); // originatingUri 属性
        params.originatingUid =
                readIntAttribute(in, ATTR_ORIGINATING_UID, SessionParams.UID_UNKNOWN); // originatingUid 属性
        params.referrerUri = readUriAttribute(in, ATTR_REFERRER_URI); // referrerUri 属性
        params.abiOverride = readStringAttribute(in, ATTR_ABI_OVERRIDE); // abiOverride 属性
        params.volumeUuid = readStringAttribute(in, ATTR_VOLUME_UUID); // volumeUuid 属性
        //【2.2.2】读取授予的运行时权限！
        params.grantedRuntimePermissions = readGrantedRuntimePermissions(in);

        final File appIconFile = buildAppIconFile(sessionId);
        if (appIconFile.exists()) {
            params.appIcon = BitmapFactory.decodeFile(appIconFile.getAbsolutePath());
            params.appIconLastModified = appIconFile.lastModified();
        }
        //【2.2.3】创建事务对象 PackageInstallerSession！
        return new PackageInstallerSession(mInternalCallback, mContext, mPm,
                mInstallThread.getLooper(), sessionId, userId, installerPackageName, installerUid,
                params, createdMillis, stageDir, stageCid, prepared, sealed);
    }
```

方法步骤：

- 解析 xml；
- 创建事务对象 PackageInstallerSession；

#### 2.2.1.1 new SessionParams

```java
        public SessionParams(int mode) {
            this.mode = mode;
        }
```
创建了一个 SessionParams，用于封装事务参数！

#### 2.2.1.2 PackageInstallerService.readGrantedRuntimePermissions

readGrantedRuntimePermissions 方法会读取授予的运行时权限！
```java
    private static String[] readGrantedRuntimePermissions(XmlPullParser in)
            throws IOException, XmlPullParserException {
        List<String> permissions = null;

        final int outerDepth = in.getDepth();
        int type;
        while ((type = in.next()) != XmlPullParser.END_DOCUMENT
                && (type != XmlPullParser.END_TAG || in.getDepth() > outerDepth)) {
            if (type == XmlPullParser.END_TAG || type == XmlPullParser.TEXT) {
                continue;
            }
            if (TAG_GRANTED_RUNTIME_PERMISSION.equals(in.getName())) { // 读取 granted-runtime-permission 子标签！
                String permission = readStringAttribute(in, ATTR_NAME); // 读取 name 属性，即运行时权限名
                if (permissions == null) {
                    permissions = new ArrayList<>();
                }
                // 将权限名保存到 permissions，并返回！！
                permissions.add(permission);
            }
        }

        if (permissions == null) {
            return null;
        }

        String[] permissionsArray = new String[permissions.size()];
        permissions.toArray(permissionsArray);
        return permissionsArray;
    }
```
方法很简单，这就不多说了！

#### 2.2.1.3 new PackageInstallerSession

创建一个 PackageInstallerSession，参数来自前面的解析！
```java
    public PackageInstallerSession(PackageInstallerService.InternalCallback callback,
            Context context, PackageManagerService pm, Looper looper, int sessionId, int userId,
            String installerPackageName, int installerUid, SessionParams params, long createdMillis,
            File stageDir, String stageCid, boolean prepared, boolean sealed) {
        mCallback = callback; // InternalCallback 对象，用于监听和相应 session 的变化！！
        mContext = context;
        mPm = pm;
        mHandler = new Handler(looper, mHandlerCallback);

        this.sessionId = sessionId;
        this.userId = userId;
        this.installerPackageName = installerPackageName;
        this.installerUid = installerUid;
        this.params = params;
        this.createdMillis = createdMillis;
        this.stageDir = stageDir;
        this.stageCid = stageCid;

        if ((stageDir == null) == (stageCid == null)) {
            throw new IllegalArgumentException(
                    "Exactly one of stageDir or stageCid stage must be set");
        }

        mPrepared = prepared;
        mSealed = sealed;

        // Device owners are allowed to silently install packages, so the permission check is
        // waived if the installer is the device owner.
        DevicePolicyManager dpm = (DevicePolicyManager) mContext.getSystemService(
                Context.DEVICE_POLICY_SERVICE);
        final boolean isPermissionGranted =
                (mPm.checkUidPermission(android.Manifest.permission.INSTALL_PACKAGES, installerUid)
                        == PackageManager.PERMISSION_GRANTED);
        final boolean isInstallerRoot = (installerUid == Process.ROOT_UID); // 安装者是否是 root 级别！
        final boolean forcePermissionPrompt =
                (params.installFlags & PackageManager.INSTALL_FORCE_PERMISSION_PROMPT) != 0;
        mIsInstallerDeviceOwner = (dpm != null) && dpm.isDeviceOwnerAppOnCallingUser(
                installerPackageName);
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
在创建 PackageInstallerSession 的时候，我们传入了一个 mInternalCallback，他是 PackageInstallerService 的成员变量：

```java
    private final InternalCallback mInternalCallback = new InternalCallback();
```

这个接口用于监听和相应 Sessions 的变化，当 Sessions 发生变化后，会通过该接口，将相应的消息抓发给 Callback，Callback 会将消息再次转发给远程回调接口！
```java
    class InternalCallback {
        public void onSessionBadgingChanged(PackageInstallerSession session) {
            mCallbacks.notifySessionBadgingChanged(session.sessionId, session.userId);
            writeSessionsAsync();
        }

        public void onSessionActiveChanged(PackageInstallerSession session, boolean active) {
            mCallbacks.notifySessionActiveChanged(session.sessionId, session.userId, active);
        }

        // 安装事务的进度发生变化！
        public void onSessionProgressChanged(PackageInstallerSession session, float progress) {
            mCallbacks.notifySessionProgressChanged(session.sessionId, session.userId, progress);
        }

        // 处理事务完成的消息！
        public void onSessionFinished(final PackageInstallerSession session, boolean success) {
            mCallbacks.notifySessionFinished(session.sessionId, session.userId, success);

            mInstallHandler.post(new Runnable() {
                @Override
                public void run() {
                    synchronized (mSessions) {
                        // 将已经完成的消息移除 mSessions，加入到 mHistoricalSessions 中！
                        mSessions.remove(session.sessionId);
                        mHistoricalSessions.put(session.sessionId, session);

                        final File appIconFile = buildAppIconFile(session.sessionId);
                        if (appIconFile.exists()) {
                            appIconFile.delete();
                        }

                        writeSessionsLocked(); // 更新 sessions 变化到本地文件中！
                    }
                }
            });
        }
        
        public void onSessionPrepared(PackageInstallerSession session) {
            // We prepared the destination to write into; we want to persist
            // this, but it's not critical enough to block for.
            writeSessionsAsync(); // 将 sessions 数据写到本地文件中！
        }

        public void onSessionSealedBlocking(PackageInstallerSession session) {
            // It's very important that we block until we've recorded the
            // session as being sealed, since we never want to allow mutation
            // after sealing.
            synchronized (mSessions) {
                writeSessionsLocked();
            }
        }
    }
```
关于 Session，我们想讲这么多，在分析安装的过程中，我们在深入分析，我们只需要知道，每一个安装任务都会封装成一个 Session 即可！


# 3 new PackageManagerInternalImpl - 系统进程内部调用

在 pms 的构造器结尾，调用了 LocalServices，将一个 PackageManagerInternalImpl 对象注册到本地服务管理中，这种方式用于系统进程内部服务间的通信，因为不需要使用 Binder 线程，所以效率更高！

```java
        //【3】将自身加入本地服务管理中，方便系统自身访问！
        LocalServices.addService(PackageManagerInternal.class, new PackageManagerInternalImpl());
```
LocalServices 内部保存着一个静态的 ArrayMap 来保存所有注册到其内部的服务：

```java
    private static final ArrayMap<Class<?>, Object> sLocalServiceObjects =
            new ArrayMap<Class<?>, Object>();
```
通过其内部的 addService 方法，将系统服务添加进来！
```java
    public static <T> void addService(Class<T> type, T service) {
        synchronized (sLocalServiceObjects) {
            if (sLocalServiceObjects.containsKey(type)) {
                throw new IllegalStateException("Overriding service registration");
            }
            sLocalServiceObjects.put(type, service);
        }
    }
```
PackageManagerInternalImpl 对象继承了 PackageManagerInternal 并实现了其抽象方法，其内部有多个方案来供其他服务访问 PMS：

## 3.1 注册系统服务 provider

后续会有其他的系统服务启动，他们会把自身的 provider 注册进入 PMS，后续 PMS 会授予他们权限！

```java
    private class PackageManagerInternalImpl extends PackageManagerInternal {
        // 创建 LocationManagerService 时会调用该方法，LocationManagerService 会创建一个 PackagesProvider 对象！
        // 用于封装其 provider name！
        @Override
        public void setLocationPackagesProvider(PackagesProvider provider) {
            synchronized (mPackages) {
                // 保存到 DefaultPermissionGrantPolicy.mLocationPackagesProvider 变量中！
                mDefaultPermissionPolicy.setLocationPackagesProviderLPw(provider);
            }
        }
        
        // 创建 VoiceInteractionManagerService 时会调用该方法！
        @Override
        public void setVoiceInteractionPackagesProvider(PackagesProvider provider) {
            synchronized (mPackages) {
                // 保存到 DefaultPermissionGrantPolicy.mVoiceInteractionPackagesProvider 变量中！
                mDefaultPermissionPolicy.setVoiceInteractionPackagesProviderLPw(provider);
            }
        }

        // 创建 TelecomLoaderService 时会调用该方法，TelecomLoaderService 创建时,
        // 会调用 registerDefaultAppProviders，该方法会触发注册 provider！
        @Override
        public void setSmsAppPackagesProvider(PackagesProvider provider) {
            synchronized (mPackages) {
                // 保存到 DefaultPermissionGrantPolicy.mSmsAppPackagesProvider 变量中！
                mDefaultPermissionPolicy.setSmsAppPackagesProviderLPw(provider);
            }
        }

        // 创建 TelecomLoaderService 时会调用该方法，同上！
        @Override
        public void setDialerAppPackagesProvider(PackagesProvider provider) {
            synchronized (mPackages) {
                // 保存到 DefaultPermissionGrantPolicy.mDialerAppPackagesProvider 变量中！
                mDefaultPermissionPolicy.setDialerAppPackagesProviderLPw(provider);
            }
        }

        // 创建 TelecomLoaderService 时会调用该方法，同上！
        @Override
        public void setSimCallManagerPackagesProvider(PackagesProvider provider) {
            synchronized (mPackages) {
                // 保存到 DefaultPermissionGrantPolicy.mSimCallManagerPackagesProvider 变量中！
                mDefaultPermissionPolicy.setSimCallManagerPackagesProviderLPw(provider);
            }
        }

        // 创建 ContentService 时会调用该方法，注册 provider！
        @Override
        public void setSyncAdapterPackagesprovider(SyncAdapterPackagesProvider provider) {
            synchronized (mPackages) {
                // 保存到 DefaultPermissionGrantPolicy.mSyncAdapterPackagesProvider 变量中！
                mDefaultPermissionPolicy.setSyncAdapterPackagesProviderLPw(provider);
            }
        }

        ... ... ... ...
    }
```

上面的内容，后面会遇到！

## 3.2 授予默认权限接口

```java
    private class PackageManagerInternalImpl extends PackageManagerInternal {
        ... ... ... ...

        @Override
        public void grantDefaultPermissionsToDefaultSmsApp(String packageName, int userId) {
            synchronized (mPackages) {
                mDefaultPermissionPolicy.grantDefaultPermissionsToDefaultSmsAppLPr(
                        packageName, userId);
            }
        }

        @Override
        public void grantDefaultPermissionsToDefaultDialerApp(String packageName, int userId) {
            synchronized (mPackages) {
                mSettings.setDefaultDialerPackageNameLPw(packageName, userId);
                mDefaultPermissionPolicy.grantDefaultPermissionsToDefaultDialerAppLPr(
                        packageName, userId);
            }
        }

        @Override
        public void grantDefaultPermissionsToDefaultSimCallManager(String packageName, int userId) {
            synchronized (mPackages) {
                mDefaultPermissionPolicy.grantDefaultPermissionsToDefaultSimCallManagerLPr(
                        packageName, userId);
            }
        }

        ... ... ... ...
    }
```

以上方法都是在 TelecomLoaderService 服务中调用的！


# 4 PackageMS.updatePackagesIfNeeded - 执行 Odex 优化

在 SystemServer.startOtherServices 中，会调用 updatePackagesIfNeeded 更新 package！

```java
        if (!mOnlyCore) {
            Trace.traceBegin(Trace.TRACE_TAG_SYSTEM_SERVER, "UpdatePackagesIfNeeded");
            try {
                mPackageManagerService.updatePackagesIfNeeded();
            } catch (Throwable e) {
                reportWtf("update packages", e);
            }
            Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);
        }
```
我们去看看 updatePackagesIfNeeded 方法，该方法的主要作用是对应用进行 Odex 优化！
```java
    @Override
    public void updatePackagesIfNeeded() {
        enforceSystemOrRoot("Only the system can request package update");
        
        //【1】是否是通过 OTA 升级的！
        boolean causeUpgrade = isUpgrade();

        //【2】是否是第一次启动，包括出场第一次开机，factory reset，如果是从 Android N 升级上来的，也看作是第一次启动！
        boolean causeFirstBoot = isFirstBoot() || mIsPreNUpgrade;

        // We need to re-extract after a pruned cache, as AoT-ed files will be out of date.
        boolean causePrunedCache = VMRuntime.didPruneDalvikCache();
        
        //【1】如果不是上面几种情况，不处理！
        if (!causeUpgrade && !causeFirstBoot && !causePrunedCache) {
            return;
        }
        //【4.1】获得要执行 odex 优化的 package！
        List<PackageParser.Package> pkgs;
        synchronized (mPackages) {
            pkgs = PackageManagerServiceUtils.getPackagesForDexopt(mPackages.values(), this);
        }

        final long startTime = System.nanoTime();
        //【4.2】执行 odex 优化！
        final int[] stats = performDexOptUpgrade(pkgs, mIsPreNUpgrade /* showDialog */,
                    getCompilerFilterForReason(causeFirstBoot ? REASON_FIRST_BOOT : REASON_BOOT));

        final int elapsedTimeSeconds =
                (int) TimeUnit.NANOSECONDS.toSeconds(System.nanoTime() - startTime);

        //【4】记录执行结果！
        MetricsLogger.histogram(mContext, "opt_dialog_num_dexopted", stats[0]);
        MetricsLogger.histogram(mContext, "opt_dialog_num_skipped", stats[1]);
        MetricsLogger.histogram(mContext, "opt_dialog_num_failed", stats[2]);
        MetricsLogger.histogram(mContext, "opt_dialog_num_total", getOptimizablePackages().size());
        MetricsLogger.histogram(mContext, "opt_dialog_time_s", elapsedTimeSeconds);
    }
```
整个过程就是收集所有的 Pacakge，然后执行 Odex 优化！

可以看到，只有

## 4.1 PackageManagerServiceUtils.getPackagesForDexopt

getPackagesForDexopt 会对系统中所有扫描到的 package 进行排序！
```java
    public static List<PackageParser.Package> getPackagesForDexopt(
            Collection<PackageParser.Package> packages,
            PackageManagerService packageManagerService) {
        //【1】result 用于保存排序后的所有结果，remainingPkgs 用于保存没有排序的 package
        // sortTemp 用于缓存排序的结果！
        ArrayList<PackageParser.Package> remainingPkgs = new ArrayList<>(packages); // 初始化 remainingPkgs！
        LinkedList<PackageParser.Package> result = new LinkedList<>();
        ArrayList<PackageParser.Package> sortTemp = new ArrayList<>(remainingPkgs.size());

        //【4.1.1】调用 applyPackageFilter 设置优先级，然后排序！
        //【4.1.1.1】开始第一阶段排序，收集那些 pkg.coreApp 为 true 的应用，进行排序，然后添加到 result 中！
        applyPackageFilter((pkg) -> pkg.coreApp, result, remainingPkgs, sortTemp,
                packageManagerService);

        //【4.1.1.2】开始第二个阶段排序，条件是是否监听 ACTION_PRE_BOOT_COMPLETED 这个广播！
        // 手机监听该广播的 package，进行排序，结果添加到 result 中！
        Intent intent = new Intent(Intent.ACTION_PRE_BOOT_COMPLETED);
        final ArraySet<String> pkgNames = getPackageNamesForIntent(intent, UserHandle.USER_SYSTEM);
        applyPackageFilter((pkg) -> pkgNames.contains(pkg.packageName), result, remainingPkgs,
                sortTemp, packageManagerService);

        //【4.1.1.3】开始第三个阶段排序，条件是 package 是否被其他应用加载或使用，排序后结果添加到 result 中！
        applyPackageFilter((pkg) -> PackageDexOptimizer.isUsedByOtherApps(pkg), result,
                remainingPkgs, sortTemp, packageManagerService);

        //【2】处理剩余的仍然没有被处理的 Package！
        Predicate<PackageParser.Package> remainingPredicate;
        if (!remainingPkgs.isEmpty() && packageManagerService.isHistoricalPackageUsageAvailable()) {
            if (DEBUG_DEXOPT) {
                Log.i(TAG, "Looking at historical package use");
            }

            //【1】对剩余的 package 进行比较，找到最新使用时间最晚的那个 package！
            PackageParser.Package lastUsed = Collections.max(remainingPkgs, (pkg1, pkg2) ->
                    Long.compare(pkg1.getLatestForegroundPackageUseTimeInMills(),
                            pkg2.getLatestForegroundPackageUseTimeInMills()));
                            
            if (DEBUG_DEXOPT) {
                Log.i(TAG, "Taking package " + lastUsed.packageName + " as reference in time use");
            }
            
            long estimatedPreviousSystemUseTime =
                    lastUsed.getLatestForegroundPackageUseTimeInMills();
            //【2】确定过滤器
            if (estimatedPreviousSystemUseTime != 0) {
                //【2.1】剩余 package 的排序条件为：该应用最新使用时间距离安装超过了 7 天！
                final long cutoffTime = estimatedPreviousSystemUseTime - SEVEN_DAYS_IN_MILLISECONDS;
                remainingPredicate =
                        (pkg) -> pkg.getLatestForegroundPackageUseTimeInMills() >= cutoffTime;
            } else {
                //【2.2】剩余 package 的排序条件为 true，即：不过滤，处理剩下的所有 package！
                remainingPredicate = (pkg) -> true;
            }

            // 先对剩余的 package 进行一次时间排序，目的是使用！
            sortPackagesByUsageDate(remainingPkgs, packageManagerService);
        } else {
            //【2.3】剩余 package 的排序条件为 true，即：不过滤，处理剩下的所有 package
            remainingPredicate = (pkg) -> true;
        }
        //【4.1.1.4】最后一次排序，处理剩余的仍然没有被处理的 Package，满足上面条件的 package！！
        applyPackageFilter(remainingPredicate, result, remainingPkgs, sortTemp,
                packageManagerService);

        if (DEBUG_DEXOPT) {
            Log.i(TAG, "Packages to be dexopted: " + packagesToString(result));
            Log.i(TAG, "Packages skipped from dexopt: " + packagesToString(remainingPkgs));
        }
        //【3】返回需要做 Odex 的排序过的 Package！
        return result;
    }

```
对于 getPackagesForDexopt 方法的分析就到这里！

### 4.1.1 PackageManagerServiceUtils.applyPackageFilter

applyPackageFilter 方法通过传入的过滤条件，收集满足条件的 package，对其进行排序！
```java
    private static void applyPackageFilter(Predicate<PackageParser.Package> filter,
            Collection<PackageParser.Package> result,
            Collection<PackageParser.Package> packages,
            @NonNull List<PackageParser.Package> sortTemp,
            PackageManagerService packageManagerService) {
        //【1】收集那些满足 filter 设定的条件的 package，添加到 sortTemp！
        for (PackageParser.Package pkg : packages) {
            if (filter.test(pkg)) {
                sortTemp.add(pkg);
            }
        }
        //【4.1.1.1】对 sortTemp 中的 package 进行排序！
        sortPackagesByUsageDate(sortTemp, packageManagerService);
        
        //【2】从 packages （就是 remainingPkgs）中移除 sortTemp 包含的 pacakge！
        packages.removeAll(sortTemp);
        
        //【3】将 sortTemp 中排好序的所有 package，添加到 result 中！
        for (PackageParser.Package pkg : sortTemp) {
            result.add(pkg);
            //【3.1】找到和该 pkg 共享非系统库文件的 package，也添加到 result 中！
            // 同时从 packages （就是 remainingPkgs）中也移除这部分 package！
            Collection<PackageParser.Package> deps =
                    packageManagerService.findSharedNonSystemLibraries(pkg);
            if (!deps.isEmpty()) {
                deps.removeAll(result);
                result.addAll(deps);
                packages.removeAll(deps);
            }
        }

        sortTemp.clear();
    }
```
过程很简单不多说了！

#### 4.1.1.1 PackageManagerServiceUtils.sortPackagesByUsageDate

该方法的作用是对 package 列表进行排序！
```java
    public static void sortPackagesByUsageDate(List<PackageParser.Package> pkgs,
            PackageManagerService packageManagerService) {
        if (!packageManagerService.isHistoricalPackageUsageAvailable()) {
            return;
        }
        //【1】排序依据是：package 在前台使用是时间！
        Collections.sort(pkgs, (pkg1, pkg2) ->
                Long.compare(pkg2.getLatestForegroundPackageUseTimeInMills(),
                        pkg1.getLatestForegroundPackageUseTimeInMills()));
    }
```

## 4.2 PackageMS.performDexOptUpgrade

performDexOptUpgrade 用于执行 Odex 优化操作！

对于参数 showDialog：传入的是 mIsPreNUpgrade，表示是否是从 N 升级上来的！

对于参数 compilerFilter：传入的是 getCompilerFilterForReason(causeFirstBoot ? REASON_FIRST_BOOT : REASON_BOOT)!
```java
    private int[] performDexOptUpgrade(List<PackageParser.Package> pkgs, boolean showDialog,
            String compilerFilter) {

        int numberOfPackagesVisited = 0;
        int numberOfPackagesOptimized = 0;
        int numberOfPackagesSkipped = 0;
        int numberOfPackagesFailed = 0;
        final int numberOfPackagesToDexopt = pkgs.size();

        //【All】遍历所有要执行 Odex 的所有 Package！
        for (PackageParser.Package pkg : pkgs) {
            numberOfPackagesVisited++;
            //【1】跳过那些不能做 Odex 优化的 Package！
            if (!PackageDexOptimizer.canOptimizePackage(pkg)) {
                if (DEBUG_DEXOPT) {
                    Log.i(TAG, "Skipping update of of non-optimizable app " + pkg.packageName);
                }
                numberOfPackagesSkipped++;
                continue;
            }

            if (DEBUG_DEXOPT) {
                Log.i(TAG, "Updating app " + numberOfPackagesVisited + " of " +
                        numberOfPackagesToDexopt + ": " + pkg.packageName);
            }
            //【2】在做 Odex 优化的时候，是否显示界面进行提示！
            if (showDialog) {
                try {
                    ActivityManagerNative.getDefault().showBootMessage(
                            mContext.getResources().getString(R.string.android_upgrading_apk,
                                    numberOfPackagesVisited, numberOfPackagesToDexopt), true);
                } catch (RemoteException e) {
                }
                synchronized (mPackages) {
                    mDexOptDialogShown = true;
                }
            }

            // If the OTA updates a system app which was previously preopted to a non-preopted state
            // the app might end up being verified at runtime. That's because by default the apps
            // are verify-profile but for preopted apps there's no profile.
            // Do a hacky check to ensure that if we have no profiles (a reasonable indication
            // that before the OTA the app was preopted) the app gets compiled with a non-profile
            // filter (by default interpret-only).
            // Note that at this stage unused apps are already filtered.
            //【3】设置执行 Odex 优化的时候的 compilerFilter 类型！
            if (isSystemApp(pkg) &&
                    DexFile.isProfileGuidedCompilerFilter(compilerFilter) &&
                    !Environment.getReferenceProfile(pkg.packageName).exists()) {
                compilerFilter = getNonProfileGuidedCompilerFilter(compilerFilter);
            }

            // checkProfiles is false to avoid merging profiles during boot which
            // might interfere with background compilation (b/28612421).
            // Unfortunately this will also means that "pm.dexopt.boot=speed-profile" will
            // behave differently than "pm.dexopt.bg-dexopt=speed-profile" but that's a
            // trade-off worth doing to save boot time work.
            //【4.2.2】执行 Odex
            int dexOptStatus = performDexOptTraced(pkg.packageName,
                    false /* checkProfiles */,
                    compilerFilter,
                    false /* force */);
            switch (dexOptStatus) {
                case PackageDexOptimizer.DEX_OPT_PERFORMED:
                    numberOfPackagesOptimized++;
                    break;
                case PackageDexOptimizer.DEX_OPT_SKIPPED:
                    numberOfPackagesSkipped++;
                    break;
                case PackageDexOptimizer.DEX_OPT_FAILED:
                    numberOfPackagesFailed++;
                    break;
                default:
                    Log.e(TAG, "Unexpected dexopt return code " + dexOptStatus);
                    break;
            }
        }

        return new int[] { numberOfPackagesOptimized, numberOfPackagesSkipped,
                numberOfPackagesFailed };
    }
```

### 4.2.1 PackageManagerServiceCompilerMapping.getCompilerFilterForReason

这里调用了 getCompilerFilterForReason，根据执行的原因选择合适的 compileFilter：
```java
    public static String getCompilerFilterForReason(int reason) {
        //【4.2.1.1】通过 reason 来获得 compileFilter
        return getAndCheckValidity(reason);
    }
```
如果是 first boot，则为 REASON_FIRST_BOOT，否则为 REASON_BOOT！

```java
    // Compilation reasons.
    public static final int REASON_FIRST_BOOT = 0;
    public static final int REASON_BOOT = 1;
    public static final int REASON_INSTALL = 2;
    public static final int REASON_BACKGROUND_DEXOPT = 3;
    public static final int REASON_AB_OTA = 4;
    public static final int REASON_NON_SYSTEM_LIBRARY = 5;
    public static final int REASON_SHARED_APK = 6;
    public static final int REASON_FORCED_DEXOPT = 7;
    public static final int REASON_CORE_APP = 8;
```
对于 reason 的取值如上，其实通过名字，很容易就能看到，每种 reason 的使用类型！

#### 4.2.1.1 PackageManagerServiceCompilerMapping.getAndCheckValidity

检查 reason 对应的有效性！

```java
    private static String getAndCheckValidity(int reason) {
        //【1】校验 reason 对应的 compileFilter 是否有效！
        String sysPropValue = SystemProperties.get(getSystemPropertyName(reason));
        if (sysPropValue == null || sysPropValue.isEmpty() ||
                !DexFile.isValidCompilerFilter(sysPropValue)) {
            //【1.1】这里会通过 DexFile.isValidCompilerFilter 校验！
            throw new IllegalStateException("Value \"" + sysPropValue +"\" not valid "
                    + "(reason " + REASON_STRINGS[reason] + ")");
        }

        //【2】如果 reason 是 REASON_SHARED_APK 和 REASON_FORCED_DEXOPT，还要校验下其是否映射到一个 
        // profile-guided filter！
        switch (reason) {
            case PackageManagerService.REASON_SHARED_APK:
            case PackageManagerService.REASON_FORCED_DEXOPT:
                if (DexFile.isProfileGuidedCompilerFilter(sysPropValue)) {
                    throw new IllegalStateException("\"" + sysPropValue + "\" is profile-guided, "
                            + "but not allowed for " + REASON_STRINGS[reason]);
                }
                break;
        }
        //【3】返回我们需要的 compileFilter！
        return sysPropValue;
    }
```
每一个有效的 reason 都会对应一个系统属性名，PackageManagerServiceCompilerMapping 内置了如下的系统属性名关键字：
```java
    static final String REASON_STRINGS[] = {
            "first-boot", "boot", "install", "bg-dexopt", "ab-ota", "nsys-library", "shared-apk",
            "forced-dexopt", "core-app"
    };
```
如果是 **REASON_FIRST_BOOT**，对应的关键字就是 "first-boot"，其他的以此类推！
```java
    private static String getSystemPropertyName(int reason) {
        if (reason < 0 || reason >= REASON_STRINGS.length) {
            throw new IllegalArgumentException("reason " + reason + " invalid");
        }

        return "pm.dexopt." + REASON_STRINGS[reason];
    }
```
然后，我们就可以获得系统属性名了，格式为：pm.dexopt.REASON_STRINGS[reason]，比如对于 REASON_FIRST_BOOT，对应的系统属性名为 ：pm.dexopt.first-boot！

然后我们通过该系统属性名，获得对应的属性 compileFilter！

### 4.2.2 PackageMS.performDexOptTraced

执行 Odex 优化操作！

```java
    private int performDexOptTraced(String packageName,
                boolean checkProfiles, String targetCompilerFilter, boolean force) {
        Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "dexopt");
        try {
            //【4.2.2.1】执行 Odex 优化！
            return performDexOptInternal(packageName, checkProfiles,
                    targetCompilerFilter, force);
        } finally {
            Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
        }
    }
```
继续调用 performDexOptTraced 方法！

#### 4.2.2.1 PackageMS.performDexOptTraced

```java
    private int performDexOptInternal(String packageName,
                boolean checkProfiles, String targetCompilerFilter, boolean force) {
        PackageParser.Package p;
        synchronized (mPackages) {
            //【1】获得 package 对应的数据对象 PackageParser.Package！
            // 找不到，返回 PackageDexOptimizer.DEX_OPT_FAILED！
            p = mPackages.get(packageName);
            if (p == null) {
                // Package could not be found. Report failure.
                return PackageDexOptimizer.DEX_OPT_FAILED;
            }
            mPackageUsage.maybeWriteAsync(mPackages);
            mCompilerStats.maybeWriteAsync();
        }
        long callingId = Binder.clearCallingIdentity();
        try {
            synchronized (mInstallLock) {
                //【4.2.2.2】调用 performDexOptInternalWithDependenciesLI 继续执行！
                return performDexOptInternalWithDependenciesLI(p, checkProfiles,
                        targetCompilerFilter, force);
            }
        } finally {
            Binder.restoreCallingIdentity(callingId);
        }
    }
```
继续调用 performDexOptInternalWithDependenciesLI

#### 4.2.2.2 PackageMS.performDexOptInternalWithDependenciesLI

根据前面的参数传递：checkProfiles 的值为 false；force 的值为 false：
```java
    private int performDexOptInternalWithDependenciesLI(PackageParser.Package p,
                boolean checkProfiles, String targetCompilerFilter,
            boolean force) {
        //【1】根据参数 force 的值，选择不同的 PackageDexOptimizer，绝大多数情况下，选择的是
        // PackageDexOptimizer！
        PackageDexOptimizer pdo = force
                ? new PackageDexOptimizer.ForcedUpdatePackageDexOptimizer(mPackageDexOptimizer)
                : mPackageDexOptimizer;

        //【2】先对该 package 所依赖的共享库所属 pacakge 进行 Odex 优化！
        Collection<PackageParser.Package> deps = findSharedNonSystemLibraries(p);
        final String[] instructionSets = getAppDexInstructionSets(p.applicationInfo);
        if (!deps.isEmpty()) {
            for (PackageParser.Package depPackage : deps) {
                // TODO: Analyze and investigate if we (should) profile libraries.
                // Currently this will do a full compilation of the library by default.
                pdo.performDexOpt(depPackage, null /* sharedLibraries */, instructionSets,
                        false /* checkProfiles */,
                        getCompilerFilterForReason(REASON_NON_SYSTEM_LIBRARY),
                        getOrCreateCompilerPackageStats(depPackage));
            }
        }
        //【4.2.2.3】对该 package 进行 Odex 优化！
        return pdo.performDexOpt(p, p.usesLibraryFiles, instructionSets, checkProfiles,
                targetCompilerFilter, getOrCreateCompilerPackageStats(p));
    }
```
如果 force 的值为 false，那么选择的是 mPackageDexOptimizer，mPackageDexOptimizer 我们都知道，在 PackageManagerService 的初始化的时候，会创建对应的 PackageDexOptimizer 对象！

绝大数情况，force 都是 false；为 true 的情况很少使用，主要用于 cmd test！

当然，最终调用了：
```java
PackageDexOptimizer.performDexOpt
```
performDexOpt 方法进行 Odex 优化！

#### 4.2.2.3 PackageDexOptimizer.performDexOpt

```java
    int performDexOpt(PackageParser.Package pkg, String[] sharedLibraries,
            String[] instructionSets, boolean checkProfiles, String targetCompilationFilter,
            CompilerStats.PackageStats packageStats) {
        synchronized (mInstallLock) {
            //【1】申请 WakeLock！
            final boolean useLock = mSystemReady;
            if (useLock) {
                mDexoptWakeLock.setWorkSource(new WorkSource(pkg.applicationInfo.uid));
                mDexoptWakeLock.acquire();
            }
            try {
                //【4.2.2.3.1】调用 performDexOptLI 执行 Odex 优化！
                return performDexOptLI(pkg, sharedLibraries, instructionSets, checkProfiles,
                        targetCompilationFilter, packageStats);
            } finally {
                if (useLock) {
                    mDexoptWakeLock.release();
                }
            }
        }
    }
```

##### 4.2.2.3.1 PackageDexOptimizer.performDexOptLI

performDexOptLI 是执行 Odex 的重要方法：
```java
    private int performDexOptLI(PackageParser.Package pkg, String[] sharedLibraries,
            String[] targetInstructionSets, boolean checkProfiles, String targetCompilerFilter,
            CompilerStats.PackageStats packageStats) {
        final String[] instructionSets = targetInstructionSets != null ?
                targetInstructionSets : getAppDexInstructionSets(pkg.applicationInfo);
        //【1】如果该 package 没有 ApplicationInfo.FLAG_HAS_CODE 该标志，
        // 就不能执行 Odex 优化，返回 DEX_OPT_SKIPPED！
        if (!canOptimizePackage(pkg)) {
            return DEX_OPT_SKIPPED;
        }
        //【2】获得该应用的 apk 文件所在路径！
        final List<String> paths = pkg.getAllCodePathsExcludingResourceOnly();
        final int sharedGid = UserHandle.getSharedAppGid(pkg.applicationInfo.uid);

        //【3】如果该应用被其他的应用使用，不能用 profile-guided
        boolean isProfileGuidedFilter = DexFile.isProfileGuidedCompilerFilter(targetCompilerFilter);
        if (isProfileGuidedFilter && isUsedByOtherApps(pkg)) {
            checkProfiles = false;

            targetCompilerFilter = getNonProfileGuidedCompilerFilter(targetCompilerFilter);
            if (DexFile.isProfileGuidedCompilerFilter(targetCompilerFilter)) {
                throw new IllegalStateException(targetCompilerFilter);
            }
            isProfileGuidedFilter = false;
        }

        // If we're asked to take profile updates into account, check now.
        boolean newProfile = false;
        if (checkProfiles && isProfileGuidedFilter) {
            // Merge profiles, see if we need to do anything.
            try {
                newProfile = mInstaller.mergeProfiles(sharedGid, pkg.packageName);
            } catch (InstallerException e) {
                Slog.w(TAG, "Failed to merge profiles", e);
            }
        }

        final boolean vmSafeMode = (pkg.applicationInfo.flags & ApplicationInfo.FLAG_VM_SAFE_MODE) != 0;
        final boolean debuggable = (pkg.applicationInfo.flags & ApplicationInfo.FLAG_DEBUGGABLE) != 0;

        boolean performedDexOpt = false;
        boolean successfulDexOpt = true;

        //【5】处理所有指令集下的  Odex 优化！
        final String[] dexCodeInstructionSets = getDexCodeInstructionSets(instructionSets);
        for (String dexCodeInstructionSet : dexCodeInstructionSets) {
            for (String path : paths) {
                int dexoptNeeded;
                try {
                    //【5.1】判断是否有必要做 Odex 优化，结果保存到 dexoptNeeded！
                    dexoptNeeded = DexFile.getDexOptNeeded(path,
                            dexCodeInstructionSet, targetCompilerFilter, newProfile);
                } catch (IOException ioe) {
                    Slog.w(TAG, "IOException reading apk: " + path, ioe);
                    return DEX_OPT_FAILED;
                }
                dexoptNeeded = adjustDexoptNeeded(dexoptNeeded); // 返回自身！
                if (PackageManagerService.DEBUG_DEXOPT) {
                    Log.i(TAG, "DexoptNeeded for " + path + "@" + targetCompilerFilter + " is " +
                            dexoptNeeded);
                }

                final String dexoptType;
                String oatDir = null;
                //【5.2】处理 dexoptNeeded 结果！
                switch (dexoptNeeded) {
                    case DexFile.NO_DEXOPT_NEEDED: 
                        // 如果是 NO_DEXOPT_NEEDED，不需要优化，跳过；
                        continue;
                    case DexFile.DEX2OAT_NEEDED: 
                        // 如果是 DEX2OAT_NEEDED，需要优化，类型为 dex2oat！
                        //【4.2.2.3.1.1】并创建 oat 目标目录！ 
                        dexoptType = "dex2oat";
                        oatDir = createOatDirIfSupported(pkg, dexCodeInstructionSet);
                        break;
                    case DexFile.PATCHOAT_NEEDED:
                        // 如果是 PATCHOAT_NEEDED，需要优化，类型为 patchoat！
                        dexoptType = "patchoat";
                        break;
                    case DexFile.SELF_PATCHOAT_NEEDED:
                        // 如果是 SELF_PATCHOAT_NEEDED，需要优化，类型为 self patchoat！
                        dexoptType = "self patchoat";
                        break;
                    default:
                        throw new IllegalStateException("Invalid dexopt:" + dexoptNeeded);
                }

                String sharedLibrariesPath = null;
                if (sharedLibraries != null && sharedLibraries.length != 0) {
                    StringBuilder sb = new StringBuilder();
                    for (String lib : sharedLibraries) {
                        if (sb.length() != 0) {
                            sb.append(":");
                        }
                        sb.append(lib);
                    }
                    sharedLibrariesPath = sb.toString();
                }
                
                Log.i(TAG, "Running dexopt (" + dexoptType + ") on: " + path + " pkg="
                        + pkg.applicationInfo.packageName + " isa=" + dexCodeInstructionSet
                        + " vmSafeMode=" + vmSafeMode + " debuggable=" + debuggable
                        + " target-filter=" + targetCompilerFilter + " oatDir = " + oatDir
                        + " sharedLibraries=" + sharedLibrariesPath);
                        
                // Profile guide compiled oat files should not be public.
                final boolean isPublic = !pkg.isForwardLocked() && !isProfileGuidedFilter;
                final int profileFlag = isProfileGuidedFilter ? DEXOPT_PROFILE_GUIDED : 0;
                final int dexFlags = adjustDexoptFlags(
                        ( isPublic ? DEXOPT_PUBLIC : 0)
                        | (vmSafeMode ? DEXOPT_SAFEMODE : 0)
                        | (debuggable ? DEXOPT_DEBUGGABLE : 0)
                        | profileFlag
                        | DEXOPT_BOOTCOMPLETE);

                try {
                    long startTime = System.currentTimeMillis();
                    //【5.3】执行 Odex 优化！
                    mInstaller.dexopt(path, sharedGid, pkg.packageName, dexCodeInstructionSet,
                            dexoptNeeded, oatDir, dexFlags, targetCompilerFilter, pkg.volumeUuid,
                            sharedLibrariesPath);
                    performedDexOpt = true;

                    if (packageStats != null) {
                        long endTime = System.currentTimeMillis();
                        packageStats.setCompileTime(path, (int)(endTime - startTime));
                    }
                } catch (InstallerException e) {
                    Slog.w(TAG, "Failed to dexopt", e);
                    successfulDexOpt = false; // 如果有异常，successfulDexOpt 为 false！
                }
            }
        }
        //【6】处理 Odex 优化结果，如果 successfulDexOpt 为 true，说明执行 Odex 的过程中没有发生错误！
        // 然后还要处理 performedDexOpt，如果为 true，表示 Odex 优化成功，返回 DEX_OPT_PERFORMED！
        // 否则，直接返回 DEX_OPT_SKIPPED 表示无需 Odex 优化，直接跳过！
        if (successfulDexOpt) {
            return performedDexOpt ? DEX_OPT_PERFORMED : DEX_OPT_SKIPPED;
        } else {
            return DEX_OPT_FAILED;
        }
    }

```
整个流程很清晰，不多说！

###### 4.2.2.3.1.1 PackageDexOptimizer.createOatDirIfSupported

createOatDirIfSupported 方法用于创建 oat 目录！
```java
    @Nullable
    private String createOatDirIfSupported(PackageParser.Package pkg, String dexInstructionSet) {
        //【1】如果应用不能有 oat 文件，返回 null！
        if (!pkg.canHaveOatDir()) {
            return null;
        }
        File codePath = new File(pkg.codePath);
        if (codePath.isDirectory()) {
            //【2】创建 oat 目录！
            File oatDir = getOatDir(codePath);
            try {
                // 这里的 oatDir 为 /data/app/<packageName>/oat
                // 而 dexInstructionSet 为 arm/arm64
                mInstaller.createOatDir(oatDir.getAbsolutePath(), dexInstructionSet);
            } catch (InstallerException e) {
                Slog.w(TAG, "Failed to create oat dir", e);
                return null;
            }
            return oatDir.getAbsolutePath();
        }
        return null;
    }
```


PackageParser.Package.canHaveOatDir 方法用于判断该应用是否能够有 oat 目录！
```java
        public boolean canHaveOatDir() {
            // 要能有 oat 目录，必须要满足 2 个条件；
            // 1、不是 system app，或者是被更新过的 system app；
            // 2、同时不能是 forward-locked app，并且不能被安装在 external 上!
            return (!isSystemApp() || isUpdatedSystemApp())
                    && !isForwardLocked() && !applicationInfo.isExternalAsec();
        }
```

# 5 PackageMS.systemReady - 进入最终阶段

在 SystemServer.startOtherServices 中，最后，会调用 systemReady 方法，进入 PackageManagerService 启动的最终阶段！
```hava
        Trace.traceBegin(Trace.TRACE_TAG_SYSTEM_SERVER, "MakePackageManagerServiceReady");
        try {
            mPackageManagerService.systemReady();
        } catch (Throwable e) {
            reportWtf("making Package Manager Service ready", e);
        }
```

当系统准备完成后，会调用到 PackageManagerService 的 systemReady 方法！
```java
    @Override
    public void systemReady() {
        mSystemReady = true;

        // Disable any carrier apps. We do this very early in boot to prevent the apps from being
        // disabled after already being started.
        CarrierAppUtils.disableCarrierAppsUntilPrivileged(mContext.getOpPackageName(), this,
                mContext.getContentResolver(), UserHandle.USER_SYSTEM);

        // Read the compatibilty setting when the system is ready.
        boolean compatibilityModeEnabled = android.provider.Settings.Global.getInt(
                mContext.getContentResolver(),
                android.provider.Settings.Global.COMPATIBILITY_MODE, 1) == 1;
        PackageParser.setCompatibilityModeEnabled(compatibilityModeEnabled);
        if (DEBUG_SETTINGS) {
            Log.d(TAG, "compatibility mode:" + compatibilityModeEnabled);
        }
        //【1】该集合表示需要默认授予运行时权限的 userId！
        int[] grantPermissionsUserIds = EMPTY_INT_ARRAY;

        synchronized (mPackages) {
            //【1】移除那些已经不存在的 preferred activity 
            ArrayList<PreferredActivity> removed = new ArrayList<PreferredActivity>();
            for (int i=0; i<mSettings.mPreferredActivities.size(); i++) {
                PreferredIntentResolver pir = mSettings.mPreferredActivities.valueAt(i);
                removed.clear();
                for (PreferredActivity pa : pir.filterSet()) {
                    if (mActivities.mActivities.get(pa.mPref.mComponent) == null) {
                        removed.add(pa);
                    }
                }
                // 移除这些不存在的 preferred activity 在系统中的关联！
                if (removed.size() > 0) {
                    for (int r=0; r<removed.size(); r++) {
                        PreferredActivity pa = removed.get(r);
                        Slog.w(TAG, "Removing dangling preferred activity: "
                                + pa.mPref.mComponent);
                        pir.removeFilter(pa);
                    }
                    // 更新偏好设置！
                    mSettings.writePackageRestrictionsLPr(
                            mSettings.mPreferredActivities.keyAt(i));
                }
            }
            
            // 返回需要授予默认权限的 userId 集合！，其实就是访问 mDefaultPermissionsGranted 在指定的 userId 下的值
            // 如果指定 userId 返回 false，表示发生了系统升级，那么我们会执行默认运行时权限授予！！
            for (int userId : UserManagerService.getInstance().getUserIds()) {
                if (!mSettings.areDefaultRuntimePermissionsGrantedLPr(userId)) {
                    grantPermissionsUserIds = ArrayUtils.appendInt(
                            grantPermissionsUserIds, userId);
                }
            }
        }
        
        // 通知 userManager 系统准备完毕！
        sUserManager.systemReady();

        //【5.1】执行默认授予运行时权限的操作！
        for (int userId : grantPermissionsUserIds) {
            mDefaultPermissionPolicy.grantDefaultPermissions(userId);
        }

        // 如果没有默认授予运行时权限给指定的 userId，那么我们会直接读取 default-permissions 目录中的文件
        // 初始化 mGrantExceptions，和上面的过程类似，不关注！
        if (grantPermissionsUserIds == EMPTY_INT_ARRAY) {
            // 该方法会发送 MSG_READ_DEFAULT_PERMISSION_EXCEPTIONS 消息给内部的 Handler
            // 然后会调用：readDefaultPermissionExceptionsLPw 方法！
            mDefaultPermissionPolicy.scheduleReadDefaultPermissionExceptions();
        }

        // 处理 mPostSystemReadyMessages 中的 START_CLEANING_PACKAGE 消息，这些消息是在 delete/remove package
        // 的时候添加进来的！！
        if (mPostSystemReadyMessages != null) {
            for (Message msg : mPostSystemReadyMessages) {
                msg.sendToTarget();
            }
            mPostSystemReadyMessages = null;
        }

        // 注册存储监听器，持续监听存储变化！
        final StorageManager storage = mContext.getSystemService(StorageManager.class);
        storage.registerListener(mStorageListener);

        // 通知 PackageInstallerService 和 PackageDexOptimizer 系统准备完成！
        // PackageDexOptimizer 是用于执行 odex 优化的，后续我们会看到！
        mInstallerService.systemReady();
        mPackageDexOptimizer.systemReady();

        MountServiceInternal mountServiceInternal = LocalServices.getService(
                MountServiceInternal.class);
        mountServiceInternal.addExternalStoragePolicy(
                new MountServiceInternal.ExternalStorageMountPolicy() {
            @Override
            public int getMountMode(int uid, String packageName) {
                if (Process.isIsolated(uid)) {
                    return Zygote.MOUNT_EXTERNAL_NONE;
                }
                if (checkUidPermission(WRITE_MEDIA_STORAGE, uid) == PERMISSION_GRANTED) {
                    return Zygote.MOUNT_EXTERNAL_DEFAULT;
                }
                if (checkUidPermission(READ_EXTERNAL_STORAGE, uid) == PERMISSION_DENIED) {
                    return Zygote.MOUNT_EXTERNAL_DEFAULT;
                }
                if (checkUidPermission(WRITE_EXTERNAL_STORAGE, uid) == PERMISSION_DENIED) {
                    return Zygote.MOUNT_EXTERNAL_READ;
                }
                return Zygote.MOUNT_EXTERNAL_WRITE;
            }

            @Override
            public boolean hasExternalStorage(int uid, String packageName) {
                return true;
            }
        });

        //【5.2】清楚哪些无效不用的用户和应用！
        reconcileUsers(StorageManager.UUID_PRIVATE_INTERNAL);
        reconcileApps(StorageManager.UUID_PRIVATE_INTERNAL);
    }

```


## 5.1 DefaultPermissionGrantPolicy.grantDefaultPermissions - 默认运行时权限授予

前面有讲过 DefaultPermissionGrantPolicy 用来处理默认授予系统应用相应的运行时权限的！
```java
    public void grantDefaultPermissions(int userId) {
        //【5.1.1】默认授予系统组件和 pri app 相应的运行时权限！
        grantPermissionsToSysComponentsAndPrivApps(userId);
        //【5.1.2】默认授予系统中的一些重要应用包相应的运行时权限！
        grantDefaultSystemHandlerPermissions(userId);
        //【5.1.3】处理之前授予异常的情况！
        grantDefaultPermissionExceptions(userId);
    }
```
我们来一个一个分析！

下面分析过程中，我们把 DefaultPermissionGrantPolicy 简称为 DPGrantPolicy

### 5.1.1 DPGrantPolicy.grantPermissionsToSysComponentsAndPrivApps

授予系统组件和私有应用默认权限！

```java
    private void grantPermissionsToSysComponentsAndPrivApps(int userId) {
        Log.i(TAG, "Granting permissions to platform components for user " + userId);

        synchronized (mService.mPackages) {
            for (PackageParser.Package pkg : mService.mPackages.values()) {
                //【5.1.1.1/2】跳过那些非系统或者非私有应用，或者不支持运行时权限，或者没有请求运行时权限的应用
                if (!isSysComponentOrPersistentPlatformSignedPrivAppLPr(pkg)
                        || !doesPackageSupportRuntimePermissions(pkg)
                        || pkg.requestedPermissions.isEmpty()) {
                    continue;
                }
                Set<String> permissions = new ArraySet<>();
                final int permissionCount = pkg.requestedPermissions.size();
                for (int i = 0; i < permissionCount; i++) {
                    String permission = pkg.requestedPermissions.get(i);
                    BasePermission bp = mService.mSettings.mPermissions.get(permission);
                    // 如果是运行时权限的话，添加到集合中！
                    if (bp != null && bp.isRuntime()) {
                        permissions.add(permission);
                    }
                }
                //【5.1.1.3】默认授予运行时权限，不需要请求！！
                if (!permissions.isEmpty()) {
                    grantRuntimePermissionsLPw(pkg, permissions, true, userId);
                }
            }
        }
    }
```
直接授予权限即可！

#### 5.1.1.1 DPGrantPolicy.isSysComponentOrPersistentPlatformSignedPrivAppLPr
```java
    private boolean isSysComponentOrPersistentPlatformSignedPrivAppLPr(PackageParser.Package pkg) {
        //【1】apk 的 uid 小于 FIRST_APPLICATION_UID，那么返回 true，不跳过！
        if (UserHandle.getAppId(pkg.applicationInfo.uid) < FIRST_APPLICATION_UID) {
            return true;
        }
        //【2】如果不是 privileged App，返回 false，会跳过！
        if (!pkg.isPrivilegedApp()) {
            return false;
        }
        //【3】如果其没有 ApplicationInfo.FLAG_PERSISTENT 标志位，返回 false，会跳过！
        PackageSetting sysPkg = mService.mSettings.getDisabledSystemPkgLPr(pkg.packageName);
        if (sysPkg != null && sysPkg.pkg != null) {
            if ((sysPkg.pkg.applicationInfo.flags & ApplicationInfo.FLAG_PERSISTENT) == 0) {
                return false;
            }
        } else if ((pkg.applicationInfo.flags & ApplicationInfo.FLAG_PERSISTENT) == 0) {
            return false;
        }
        // 最后，匹配签名，如果应用是系统平台签名，返回 true，不跳过！
        return PackageManagerService.compareSignatures(mService.mPlatformPackage.mSignatures,
                pkg.mSignatures) == PackageManager.SIGNATURE_MATCH;
    }
```
上面的逻辑就不多说了！


#### 5.1.1.2 DPGrantPolicy.doesPackageSupportRuntimePermissions

判断该 package 是否支持运行时权限！
```java
    private static boolean doesPackageSupportRuntimePermissions(PackageParser.Package pkg) {
        return pkg.applicationInfo.targetSdkVersion > Build.VERSION_CODES.LOLLIPOP_MR1;
    }
```
即：如果目标 sdk 大于 Android5.1，那就是支持，返回 true！


### 5.1.2 DPGrantPolicy.grantDefaultSystemHandlerPermissions
```java
    private void grantDefaultSystemHandlerPermissions(int userId) {
        Log.i(TAG, "Granting permissions to default platform handlers for user " + userId);
        
        // 这些 PackagesProvider 就是其他服务通过 PackageManagerInternalImpl 注册到 pms 内部的！
        final PackagesProvider locationPackagesProvider;
        final PackagesProvider voiceInteractionPackagesProvider;
        final PackagesProvider smsAppPackagesProvider;
        final PackagesProvider dialerAppPackagesProvider;
        final PackagesProvider simCallManagerPackagesProvider;
        final SyncAdapterPackagesProvider syncAdapterPackagesProvider;

        synchronized (mService.mPackages) {
            locationPackagesProvider = mLocationPackagesProvider;
            voiceInteractionPackagesProvider = mVoiceInteractionPackagesProvider;
            smsAppPackagesProvider = mSmsAppPackagesProvider;
            dialerAppPackagesProvider = mDialerAppPackagesProvider;
            simCallManagerPackagesProvider = mSimCallManagerPackagesProvider;
            syncAdapterPackagesProvider = mSyncAdapterPackagesProvider;
        }

        String[] voiceInteractPackageNames = (voiceInteractionPackagesProvider != null)
                ? voiceInteractionPackagesProvider.getPackages(userId) : null;
        String[] locationPackageNames = (locationPackagesProvider != null)
                ? locationPackagesProvider.getPackages(userId) : null;
        String[] smsAppPackageNames = (smsAppPackagesProvider != null)
                ? smsAppPackagesProvider.getPackages(userId) : null;
        String[] dialerAppPackageNames = (dialerAppPackagesProvider != null)
                ? dialerAppPackagesProvider.getPackages(userId) : null;
        String[] simCallManagerPackageNames = (simCallManagerPackagesProvider != null)
                ? simCallManagerPackagesProvider.getPackages(userId) : null;
        String[] contactsSyncAdapterPackages = (syncAdapterPackagesProvider != null) ?
                syncAdapterPackagesProvider.getPackages(ContactsContract.AUTHORITY, userId) : null;
        String[] calendarSyncAdapterPackages = (syncAdapterPackagesProvider != null) ?
                syncAdapterPackagesProvider.getPackages(CalendarContract.AUTHORITY, userId) : null;

        synchronized (mService.mPackages) {
            //【2】默认授予 PackageInstaller STORAGE_PERMISSIONS 中的所有权限！
            PackageParser.Package installerPackage = getSystemPackageLPr(
                    mService.mRequiredInstallerPackage);
            if (installerPackage != null
                    && doesPackageSupportRuntimePermissions(installerPackage)) {
                grantRuntimePermissionsLPw(installerPackage, STORAGE_PERMISSIONS, true, userId);
            }

            //【3】默认授予 PackageVerifier STORAGE_PERMISSIONS，PHONE_PERMISSIONS，SMS_PERMISSIONS 中的所有权限！
            PackageParser.Package verifierPackage = getSystemPackageLPr(
                    mService.mRequiredVerifierPackage);
            if (verifierPackage != null
                    && doesPackageSupportRuntimePermissions(verifierPackage)) {
                grantRuntimePermissionsLPw(verifierPackage, STORAGE_PERMISSIONS, true, userId);
                grantRuntimePermissionsLPw(verifierPackage, PHONE_PERMISSIONS, false, userId);
                grantRuntimePermissionsLPw(verifierPackage, SMS_PERMISSIONS, false, userId);
            }
            //【4】默认授予 SetupWizard PHONE_PERMISSIONS，CONTACTS_PERMISSIONS，LOCATION_PERMISSIONS 
            // CAMERA_PERMISSIONS中的所有权限！
            PackageParser.Package setupPackage = getSystemPackageLPr(
                    mService.mSetupWizardPackage);
            if (setupPackage != null
                    && doesPackageSupportRuntimePermissions(setupPackage)) {
                grantRuntimePermissionsLPw(setupPackage, PHONE_PERMISSIONS, userId);
                grantRuntimePermissionsLPw(setupPackage, CONTACTS_PERMISSIONS, userId);
                grantRuntimePermissionsLPw(setupPackage, LOCATION_PERMISSIONS, userId);
                grantRuntimePermissionsLPw(setupPackage, CAMERA_PERMISSIONS, userId);
            }

            //【5】默认授予 Camera CAMERA_PERMISSIONS，MICROPHONE_PERMISSIONS，STORAGE_PERMISSIONS 中的所有权限！
            Intent cameraIntent = new Intent(MediaStore.ACTION_IMAGE_CAPTURE);
            PackageParser.Package cameraPackage = getDefaultSystemHandlerActivityPackageLPr(
                    cameraIntent, userId);
            if (cameraPackage != null
                    && doesPackageSupportRuntimePermissions(cameraPackage)) {
                grantRuntimePermissionsLPw(cameraPackage, CAMERA_PERMISSIONS, userId);
                grantRuntimePermissionsLPw(cameraPackage, MICROPHONE_PERMISSIONS, userId);
                grantRuntimePermissionsLPw(cameraPackage, STORAGE_PERMISSIONS, userId);
            }

            //【6】默认授予 Media provier STORAGE_PERMISSIONS 中的所有权限！
            PackageParser.Package mediaStorePackage = getDefaultProviderAuthorityPackageLPr(
                    MediaStore.AUTHORITY, userId);
            if (mediaStorePackage != null) {
                grantRuntimePermissionsLPw(mediaStorePackage, STORAGE_PERMISSIONS, true, userId);
            }

            //【7】默认授予 download provier STORAGE_PERMISSIONS 中的所有权限！
            PackageParser.Package downloadsPackage = getDefaultProviderAuthorityPackageLPr(
                    "downloads", userId);
            if (downloadsPackage != null) {
                grantRuntimePermissionsLPw(downloadsPackage, STORAGE_PERMISSIONS, true, userId);
            }

            //【8】默认授予 Downloads UI STORAGE_PERMISSIONS 中的所有权限！
            Intent downloadsUiIntent = new Intent(DownloadManager.ACTION_VIEW_DOWNLOADS);
            PackageParser.Package downloadsUiPackage = getDefaultSystemHandlerActivityPackageLPr(
                    downloadsUiIntent, userId);
            if (downloadsUiPackage != null
                    && doesPackageSupportRuntimePermissions(downloadsUiPackage)) {
                grantRuntimePermissionsLPw(downloadsUiPackage, STORAGE_PERMISSIONS, true, userId);
            }

            //【9】默认授予 Storage provider STORAGE_PERMISSIONS 中的所有权限！
            PackageParser.Package storagePackage = getDefaultProviderAuthorityPackageLPr(
                    "com.android.externalstorage.documents", userId);
            if (storagePackage != null) {
                grantRuntimePermissionsLPw(storagePackage, STORAGE_PERMISSIONS, true, userId);
            }

            //【10】默认授予 CertInstaller STORAGE_PERMISSIONS 中的所有权限！
            Intent certInstallerIntent = new Intent(Credentials.INSTALL_ACTION);
            PackageParser.Package certInstallerPackage = getDefaultSystemHandlerActivityPackageLPr(
                    certInstallerIntent, userId);
            if (certInstallerPackage != null
                    && doesPackageSupportRuntimePermissions(certInstallerPackage)) {
                grantRuntimePermissionsLPw(certInstallerPackage, STORAGE_PERMISSIONS, true, userId);
            }

            //【11】默认授予 dialer CONTACTS_PERMISSIONS，SMS_PERMISSIONS，
            // MICROPHONE_PERMISSIONS，CAMERA_PERMISSIONS中的所有权限！
            if (dialerAppPackageNames == null) {
                Intent dialerIntent = new Intent(Intent.ACTION_DIAL);
                PackageParser.Package dialerPackage = getDefaultSystemHandlerActivityPackageLPr(
                        dialerIntent, userId);
                if (dialerPackage != null) {
                    grantDefaultPermissionsToDefaultSystemDialerAppLPr(dialerPackage, userId);
                }
            } else {
                for (String dialerAppPackageName : dialerAppPackageNames) {
                    PackageParser.Package dialerPackage = getSystemPackageLPr(dialerAppPackageName);
                    if (dialerPackage != null) {
                        grantDefaultPermissionsToDefaultSystemDialerAppLPr(dialerPackage, userId);
                    }
                }
            }
            //【12】默认授予 Sim call manager PHONE_PERMISSIONS，SMS_PERMISSIONS
            // MICROPHONE_PERMISSIONS，CAMERA_PERMISSIONS中的所有权限！
            if (simCallManagerPackageNames != null) {
                for (String simCallManagerPackageName : simCallManagerPackageNames) {
                    PackageParser.Package simCallManagerPackage =
                            getSystemPackageLPr(simCallManagerPackageName);
                    if (simCallManagerPackage != null) {
                        grantDefaultPermissionsToDefaultSimCallManagerLPr(simCallManagerPackage,
                                userId);
                    }
                }
            }
            //【13】默认授予 SMS PHONE_PERMISSIONS，CONTACTS_PERMISSIONS，SMS_PERMISSIONS 中的所有权限！
            if (smsAppPackageNames == null) {
                Intent smsIntent = new Intent(Intent.ACTION_MAIN);
                smsIntent.addCategory(Intent.CATEGORY_APP_MESSAGING);
                PackageParser.Package smsPackage = getDefaultSystemHandlerActivityPackageLPr(
                        smsIntent, userId);
                if (smsPackage != null) {
                   grantDefaultPermissionsToDefaultSystemSmsAppLPr(smsPackage, userId);
                }
            } else {
                for (String smsPackageName : smsAppPackageNames) {
                    PackageParser.Package smsPackage = getSystemPackageLPr(smsPackageName);
                    if (smsPackage != null) {
                        grantDefaultPermissionsToDefaultSystemSmsAppLPr(smsPackage, userId);
                    }
                }
            }
            //【14】默认授予 Cell Broadcast Receiver SMS_PERMISSIONS 中的所有权限！
            Intent cbrIntent = new Intent(Intents.SMS_CB_RECEIVED_ACTION);
            PackageParser.Package cbrPackage =
                    getDefaultSystemHandlerActivityPackageLPr(cbrIntent, userId);
            if (cbrPackage != null && doesPackageSupportRuntimePermissions(cbrPackage)) {
                grantRuntimePermissionsLPw(cbrPackage, SMS_PERMISSIONS, userId);
            }
            //【15】默认授予 Carrier Provisioning Service SMS_PERMISSIONS 中的所有权限！
            Intent carrierProvIntent = new Intent(Intents.SMS_CARRIER_PROVISION_ACTION);
            PackageParser.Package carrierProvPackage =
                    getDefaultSystemHandlerServicePackageLPr(carrierProvIntent, userId);
            if (carrierProvPackage != null && doesPackageSupportRuntimePermissions(carrierProvPackage)) {
                grantRuntimePermissionsLPw(carrierProvPackage, SMS_PERMISSIONS, false, userId);
            }
            //【15】默认授予 Calendar CALENDAR_PERMISSIONS 中的所有权限！
            Intent calendarIntent = new Intent(Intent.ACTION_MAIN);
            calendarIntent.addCategory(Intent.CATEGORY_APP_CALENDAR);
            PackageParser.Package calendarPackage = getDefaultSystemHandlerActivityPackageLPr(
                    calendarIntent, userId);
            if (calendarPackage != null
                    && doesPackageSupportRuntimePermissions(calendarPackage)) {
                grantRuntimePermissionsLPw(calendarPackage, CALENDAR_PERMISSIONS, userId);
                grantRuntimePermissionsLPw(calendarPackage, CONTACTS_PERMISSIONS, userId);
            }
            //【16】默认授予 Calendar provider CONTACTS_PERMISSIONS，CALENDAR_PERMISSIONS，
            // STORAGE_PERMISSIONS 中的所有权限！
            PackageParser.Package calendarProviderPackage = getDefaultProviderAuthorityPackageLPr(
                    CalendarContract.AUTHORITY, userId);
            if (calendarProviderPackage != null) {
                grantRuntimePermissionsLPw(calendarProviderPackage, CONTACTS_PERMISSIONS, userId);
                grantRuntimePermissionsLPw(calendarProviderPackage, CALENDAR_PERMISSIONS,
                        true, userId);
                grantRuntimePermissionsLPw(calendarProviderPackage, STORAGE_PERMISSIONS, userId);
            }
            //【17】默认授予 Calendar provider sync adapters CALENDAR_PERMISSIONS 中的所有权限！
            List<PackageParser.Package> calendarSyncAdapters = getHeadlessSyncAdapterPackagesLPr(
                    calendarSyncAdapterPackages, userId);
            final int calendarSyncAdapterCount = calendarSyncAdapters.size();
            for (int i = 0; i < calendarSyncAdapterCount; i++) {
                PackageParser.Package calendarSyncAdapter = calendarSyncAdapters.get(i);
                if (doesPackageSupportRuntimePermissions(calendarSyncAdapter)) {
                    grantRuntimePermissionsLPw(calendarSyncAdapter, CALENDAR_PERMISSIONS, userId);
                }
            }
            //【18】默认授予 Contacts CONTACTS_PERMISSIONS， PHONE_PERMISSIONS 中的所有权限！
            Intent contactsIntent = new Intent(Intent.ACTION_MAIN);
            contactsIntent.addCategory(Intent.CATEGORY_APP_CONTACTS);
            PackageParser.Package contactsPackage = getDefaultSystemHandlerActivityPackageLPr(
                    contactsIntent, userId);
            if (contactsPackage != null
                    && doesPackageSupportRuntimePermissions(contactsPackage)) {
                grantRuntimePermissionsLPw(contactsPackage, CONTACTS_PERMISSIONS, userId);
                grantRuntimePermissionsLPw(contactsPackage, PHONE_PERMISSIONS, userId);
            }
            //【19】默认授予 Contacts provider sync adapters CONTACTS_PERMISSIONS 中的所有权限！
            List<PackageParser.Package> contactsSyncAdapters = getHeadlessSyncAdapterPackagesLPr(
                    contactsSyncAdapterPackages, userId);
            final int contactsSyncAdapterCount = contactsSyncAdapters.size();
            for (int i = 0; i < contactsSyncAdapterCount; i++) {
                PackageParser.Package contactsSyncAdapter = contactsSyncAdapters.get(i);
                if (doesPackageSupportRuntimePermissions(contactsSyncAdapter)) {
                    grantRuntimePermissionsLPw(contactsSyncAdapter, CONTACTS_PERMISSIONS, userId);
                }
            }
            //【20】默认授予 Contacts provider CONTACTS_PERMISSIONS，PHONE_PERMISSIONS，
            // STORAGE_PERMISSIONS 中的所有权限！
            PackageParser.Package contactsProviderPackage = getDefaultProviderAuthorityPackageLPr(
                    ContactsContract.AUTHORITY, userId);
            if (contactsProviderPackage != null) {
                grantRuntimePermissionsLPw(contactsProviderPackage, CONTACTS_PERMISSIONS,
                        true, userId);
                grantRuntimePermissionsLPw(contactsProviderPackage, PHONE_PERMISSIONS,
                        true, userId);
                grantRuntimePermissionsLPw(contactsProviderPackage, STORAGE_PERMISSIONS, userId);
            }
            //【21】默认授予 Device provisioning CONTACTS_PERMISSIONS 中的所有权限！
            Intent deviceProvisionIntent = new Intent(
                    DevicePolicyManager.ACTION_PROVISION_MANAGED_DEVICE);
            PackageParser.Package deviceProvisionPackage =
                    getDefaultSystemHandlerActivityPackageLPr(deviceProvisionIntent, userId);
            if (deviceProvisionPackage != null
                    && doesPackageSupportRuntimePermissions(deviceProvisionPackage)) {
                grantRuntimePermissionsLPw(deviceProvisionPackage, CONTACTS_PERMISSIONS, userId);
            }
            //【22】默认授予 Maps App LOCATION_PERMISSIONS 中的所有权限！
            Intent mapsIntent = new Intent(Intent.ACTION_MAIN);
            mapsIntent.addCategory(Intent.CATEGORY_APP_MAPS);
            PackageParser.Package mapsPackage = getDefaultSystemHandlerActivityPackageLPr(
                    mapsIntent, userId);
            if (mapsPackage != null
                    && doesPackageSupportRuntimePermissions(mapsPackage)) {
                grantRuntimePermissionsLPw(mapsPackage, LOCATION_PERMISSIONS, userId);
            }
            //【23】默认授予 Gallery App STORAGE_PERMISSIONS 中的所有权限！
            Intent galleryIntent = new Intent(Intent.ACTION_MAIN);
            galleryIntent.addCategory(Intent.CATEGORY_APP_GALLERY);
            PackageParser.Package galleryPackage = getDefaultSystemHandlerActivityPackageLPr(
                    galleryIntent, userId);
            if (galleryPackage != null
                    && doesPackageSupportRuntimePermissions(galleryPackage)) {
                grantRuntimePermissionsLPw(galleryPackage, STORAGE_PERMISSIONS, userId);
            }
            //【24】默认授予 Email App CONTACTS_PERMISSIONS，CALENDAR_PERMISSIONS 中的所有权限！
            Intent emailIntent = new Intent(Intent.ACTION_MAIN);
            emailIntent.addCategory(Intent.CATEGORY_APP_EMAIL);
            PackageParser.Package emailPackage = getDefaultSystemHandlerActivityPackageLPr(
                    emailIntent, userId);
            if (emailPackage != null
                    && doesPackageSupportRuntimePermissions(emailPackage)) {
                grantRuntimePermissionsLPw(emailPackage, CONTACTS_PERMISSIONS, userId);
                grantRuntimePermissionsLPw(emailPackage, CALENDAR_PERMISSIONS, userId);
            }
            //【25】默认授予 Browser App LOCATION_PERMISSIONS 中的所有权限！
            PackageParser.Package browserPackage = null;
            String defaultBrowserPackage = mService.getDefaultBrowserPackageName(userId);
            if (defaultBrowserPackage != null) {
                browserPackage = getPackageLPr(defaultBrowserPackage);
            }
            if (browserPackage == null) {
                Intent browserIntent = new Intent(Intent.ACTION_MAIN);
                browserIntent.addCategory(Intent.CATEGORY_APP_BROWSER);
                browserPackage = getDefaultSystemHandlerActivityPackageLPr(
                        browserIntent, userId);
            }
            if (browserPackage != null
                    && doesPackageSupportRuntimePermissions(browserPackage)) {
                grantRuntimePermissionsLPw(browserPackage, LOCATION_PERMISSIONS, userId);
            }
            //【26】默认授予 Voice interaction CONTACTS_PERMISSIONS，CALENDAR_PERMISSIONS，MICROPHONE_PERMISSIONS
            // PHONE_PERMISSIONS，SMS_PERMISSIONS，LOCATION_PERMISSIONS 中的所有权限！
            if (voiceInteractPackageNames != null) {
                for (String voiceInteractPackageName : voiceInteractPackageNames) {
                    PackageParser.Package voiceInteractPackage = getSystemPackageLPr(
                            voiceInteractPackageName);
                    if (voiceInteractPackage != null
                            && doesPackageSupportRuntimePermissions(voiceInteractPackage)) {
                        grantRuntimePermissionsLPw(voiceInteractPackage,
                                CONTACTS_PERMISSIONS, userId);
                        grantRuntimePermissionsLPw(voiceInteractPackage,
                                CALENDAR_PERMISSIONS, userId);
                        grantRuntimePermissionsLPw(voiceInteractPackage,
                                MICROPHONE_PERMISSIONS, userId);
                        grantRuntimePermissionsLPw(voiceInteractPackage,
                                PHONE_PERMISSIONS, userId);
                        grantRuntimePermissionsLPw(voiceInteractPackage,
                                SMS_PERMISSIONS, userId);
                        grantRuntimePermissionsLPw(voiceInteractPackage,
                                LOCATION_PERMISSIONS, userId);
                    }
                }
            }
            //【26】默认授予 Voice recognition MICROPHONE_PERMISSIONS 中的所有权限！
            Intent voiceRecoIntent = new Intent("android.speech.RecognitionService");
            voiceRecoIntent.addCategory(Intent.CATEGORY_DEFAULT);
            PackageParser.Package voiceRecoPackage = getDefaultSystemHandlerServicePackageLPr(
                    voiceRecoIntent, userId);
            if (voiceRecoPackage != null
                    && doesPackageSupportRuntimePermissions(voiceRecoPackage)) {
                grantRuntimePermissionsLPw(voiceRecoPackage, MICROPHONE_PERMISSIONS, userId);
            }
            //【27】默认授予 Location CONTACTS_PERMISSIONS... 中的所有权限！
            if (locationPackageNames != null) {
                for (String packageName : locationPackageNames) {
                    PackageParser.Package locationPackage = getSystemPackageLPr(packageName);
                    if (locationPackage != null
                            && doesPackageSupportRuntimePermissions(locationPackage)) {
                        grantRuntimePermissionsLPw(locationPackage, CONTACTS_PERMISSIONS, userId);
                        grantRuntimePermissionsLPw(locationPackage, CALENDAR_PERMISSIONS, userId);
                        grantRuntimePermissionsLPw(locationPackage, MICROPHONE_PERMISSIONS, userId);
                        grantRuntimePermissionsLPw(locationPackage, PHONE_PERMISSIONS, userId);
                        grantRuntimePermissionsLPw(locationPackage, SMS_PERMISSIONS, userId);
                        grantRuntimePermissionsLPw(locationPackage, LOCATION_PERMISSIONS,
                                true, userId);
                        grantRuntimePermissionsLPw(locationPackage, CAMERA_PERMISSIONS, userId);
                        grantRuntimePermissionsLPw(locationPackage, SENSORS_PERMISSIONS, userId);
                        grantRuntimePermissionsLPw(locationPackage, STORAGE_PERMISSIONS, userId);
                    }
                }
            }
            //【28】默认授予 Music STORAGE_PERMISSIONS 中的所有权限！
            Intent musicIntent = new Intent(Intent.ACTION_VIEW);
            musicIntent.addCategory(Intent.CATEGORY_DEFAULT);
            musicIntent.setDataAndType(Uri.fromFile(new File("foo.mp3")),
                    AUDIO_MIME_TYPE);
            PackageParser.Package musicPackage = getDefaultSystemHandlerActivityPackageLPr(
                    musicIntent, userId);
            if (musicPackage != null
                    && doesPackageSupportRuntimePermissions(musicPackage)) {
                grantRuntimePermissionsLPw(musicPackage, STORAGE_PERMISSIONS, userId);
            }

            // 这里是针对于 Android Wear 设备的，不关注！
            if (mService.hasSystemFeature(PackageManager.FEATURE_WATCH, 0)) {
                Intent homeIntent = new Intent(Intent.ACTION_MAIN);
                homeIntent.addCategory(Intent.CATEGORY_HOME_MAIN);

                PackageParser.Package wearHomePackage = getDefaultSystemHandlerActivityPackageLPr(
                        homeIntent, userId);

                if (wearHomePackage != null
                        && doesPackageSupportRuntimePermissions(wearHomePackage)) {
                    grantRuntimePermissionsLPw(wearHomePackage, CONTACTS_PERMISSIONS, false,
                            userId);
                    grantRuntimePermissionsLPw(wearHomePackage, PHONE_PERMISSIONS, true, userId);
                    grantRuntimePermissionsLPw(wearHomePackage, MICROPHONE_PERMISSIONS, false,
                            userId);
                    grantRuntimePermissionsLPw(wearHomePackage, LOCATION_PERMISSIONS, false,
                            userId);
                }
            }
            //【29】默认授予 Print Spooler LOCATION_PERMISSIONS 中的所有权限！
            PackageParser.Package printSpoolerPackage = getSystemPackageLPr(
                    PrintManager.PRINT_SPOOLER_PACKAGE_NAME);
            if (printSpoolerPackage != null
                    && doesPackageSupportRuntimePermissions(printSpoolerPackage)) {
                grantRuntimePermissionsLPw(printSpoolerPackage, LOCATION_PERMISSIONS, true, userId);
            }
            //【30】默认授予 EmergencyInfo CONTACTS_PERMISSIONS, PHONE_PERMISSIONS 中的所有权限！
            Intent emergencyInfoIntent = new Intent(TelephonyManager.ACTION_EMERGENCY_ASSISTANCE);
            PackageParser.Package emergencyInfoPckg = getDefaultSystemHandlerActivityPackageLPr(
                    emergencyInfoIntent, userId);
            if (emergencyInfoPckg != null
                    && doesPackageSupportRuntimePermissions(emergencyInfoPckg)) {
                grantRuntimePermissionsLPw(emergencyInfoPckg, CONTACTS_PERMISSIONS, true, userId);
                grantRuntimePermissionsLPw(emergencyInfoPckg, PHONE_PERMISSIONS, true, userId);
            }
            //【31】默认授予 NFC CONTACTS_PERMISSIONS, PHONE_PERMISSIONS 中的所有权限！
            Intent nfcTagIntent = new Intent(Intent.ACTION_VIEW);
            nfcTagIntent.setType("vnd.android.cursor.item/ndef_msg");
            PackageParser.Package nfcTagPkg = getDefaultSystemHandlerActivityPackageLPr(
                    nfcTagIntent, userId);
            if (nfcTagPkg != null
                    && doesPackageSupportRuntimePermissions(nfcTagPkg)) {
                grantRuntimePermissionsLPw(nfcTagPkg, CONTACTS_PERMISSIONS, false, userId);
                grantRuntimePermissionsLPw(nfcTagPkg, PHONE_PERMISSIONS, false, userId);
            }
            //【32】默认授予 Storage Manager STORAGE_PERMISSIONS 中的所有权限！
            Intent storageManagerIntent = new Intent(StorageManager.ACTION_MANAGE_STORAGE);
            PackageParser.Package storageManagerPckg = getDefaultSystemHandlerActivityPackageLPr(
                    storageManagerIntent, userId);
            if (storageManagerPckg != null
                    && doesPackageSupportRuntimePermissions(storageManagerPckg)) {
                grantRuntimePermissionsLPw(storageManagerPckg, STORAGE_PERMISSIONS, true, userId);
            }
            
            //【5.1.2.2】最后调用了 Settings.onDefaultRuntimePermissionsGrantedLPr 方法，
            // 再次保存运行时权限授予情况！！！
            mService.mSettings.onDefaultRuntimePermissionsGrantedLPr(userId);
        }
    }

```
我们可以看到，这个阶段 DefaultPermissionGrantPolicy 默认授予了很多系统组件很多必须的运行时权限！

涉及到的 DefaultPermissionGrantPolicy 内部方法也很多，但是**最终都是调用了其内部的 grantRuntimePermissionsLPw 方法**，该方法，我们签名有分析过！

这里就不在跟踪了！

#### 5.1.2.1 权限集合

DefaultPermissionGrantPolicy 内部有多个 Set 集合，保存了要默认授予给应用的运行时权限！
```java
    private static final Set<String> PHONE_PERMISSIONS = new ArraySet<>();
    static {
        PHONE_PERMISSIONS.add(Manifest.permission.READ_PHONE_STATE);
        PHONE_PERMISSIONS.add(Manifest.permission.CALL_PHONE);
        PHONE_PERMISSIONS.add(Manifest.permission.READ_CALL_LOG);
        PHONE_PERMISSIONS.add(Manifest.permission.WRITE_CALL_LOG);
        PHONE_PERMISSIONS.add(Manifest.permission.ADD_VOICEMAIL);
        PHONE_PERMISSIONS.add(Manifest.permission.USE_SIP);
        PHONE_PERMISSIONS.add(Manifest.permission.PROCESS_OUTGOING_CALLS);
    }

    private static final Set<String> CONTACTS_PERMISSIONS = new ArraySet<>();
    static {
        CONTACTS_PERMISSIONS.add(Manifest.permission.READ_CONTACTS);
        CONTACTS_PERMISSIONS.add(Manifest.permission.WRITE_CONTACTS);
        CONTACTS_PERMISSIONS.add(Manifest.permission.GET_ACCOUNTS);
    }

    private static final Set<String> LOCATION_PERMISSIONS = new ArraySet<>();
    static {
        LOCATION_PERMISSIONS.add(Manifest.permission.ACCESS_FINE_LOCATION);
        LOCATION_PERMISSIONS.add(Manifest.permission.ACCESS_COARSE_LOCATION);
    }

    private static final Set<String> CALENDAR_PERMISSIONS = new ArraySet<>();
    static {
        CALENDAR_PERMISSIONS.add(Manifest.permission.READ_CALENDAR);
        CALENDAR_PERMISSIONS.add(Manifest.permission.WRITE_CALENDAR);
    }

    private static final Set<String> SMS_PERMISSIONS = new ArraySet<>();
    static {
        SMS_PERMISSIONS.add(Manifest.permission.SEND_SMS);
        SMS_PERMISSIONS.add(Manifest.permission.RECEIVE_SMS);
        SMS_PERMISSIONS.add(Manifest.permission.READ_SMS);
        SMS_PERMISSIONS.add(Manifest.permission.RECEIVE_WAP_PUSH);
        SMS_PERMISSIONS.add(Manifest.permission.RECEIVE_MMS);
        SMS_PERMISSIONS.add(Manifest.permission.READ_CELL_BROADCASTS);
    }

    private static final Set<String> MICROPHONE_PERMISSIONS = new ArraySet<>();
    static {
        MICROPHONE_PERMISSIONS.add(Manifest.permission.RECORD_AUDIO);
    }

    private static final Set<String> CAMERA_PERMISSIONS = new ArraySet<>();
    static {
        CAMERA_PERMISSIONS.add(Manifest.permission.CAMERA);
    }

    private static final Set<String> SENSORS_PERMISSIONS = new ArraySet<>();
    static {
        SENSORS_PERMISSIONS.add(Manifest.permission.BODY_SENSORS);
    }

    private static final Set<String> STORAGE_PERMISSIONS = new ArraySet<>();
    static {
        STORAGE_PERMISSIONS.add(Manifest.permission.READ_EXTERNAL_STORAGE);
        STORAGE_PERMISSIONS.add(Manifest.permission.WRITE_EXTERNAL_STORAGE);
    }
```
这里只是简单的列举出来！
#### 5.1.2.2 Settings.onDefaultRuntimePermissionsGrantedLPr

```java
    void onDefaultRuntimePermissionsGrantedLPr(int userId) {
        mRuntimePermissionsPersistence
                .onDefaultRuntimePermissionsGrantedLPr(userId);
    }
```

对于 RuntimePermissionsPersistence 我们前面有说过，这里会调用其 onDefaultRuntimePermissionsGrantedLPr 方法，重新保存一次运行时权限授予情况！

代码就不在跟踪了！

### 5.1.3 DPGrantPolicy.grantDefaultPermissionExceptions

grantDefaultPermissionExceptions 用于处理那些授予异常的情况，系统会将授予异常的权限保存到一个文件中！

```java
    private void grantDefaultPermissionExceptions(int userId) {
        synchronized (mService.mPackages) {
            mHandler.removeMessages(MSG_READ_DEFAULT_PERMISSION_EXCEPTIONS);
            //【5.1.3.1】通过本地文件读取默认运行时权限的授予！
            if (mGrantExceptions == null) {
                mGrantExceptions = readDefaultPermissionExceptionsLPw();
            }
            // mGrantExceptions 仅在第一次读取之前为空，然后它作为应为每个用户执行的默认授权的缓存。
            // 如果有条目，则应用程序位于系统映像上并支持运行时权限。
            Set<String> permissions = null;
            final int exceptionCount = mGrantExceptions.size();
            for (int i = 0; i < exceptionCount; i++) {
                String packageName = mGrantExceptions.keyAt(i);
                PackageParser.Package pkg = getSystemPackageLPr(packageName);
                List<DefaultPermissionGrant> permissionGrants = mGrantExceptions.valueAt(i);
                final int permissionGrantCount = permissionGrants.size();
                for (int j = 0; j < permissionGrantCount; j++) {
                    DefaultPermissionGrant permissionGrant = permissionGrants.get(j);
                    if (permissions == null) {
                        permissions = new ArraySet<>();
                    } else {
                        permissions.clear();
                    }
                    permissions.add(permissionGrant.name);
                    //【5.1.1.3】再次调用 grantRuntimePermissionsLPw 授予该运行时权限！
                    grantRuntimePermissionsLPw(pkg, permissions, false,
                            permissionGrant.fixed, userId);
                }
            }
        }
    }
```
#### 5.1.3.1 DPGrantPolicy.readDefaultPermissionExceptionsLPw
```java
    private @NonNull ArrayMap<String, List<DefaultPermissionGrant>>
            readDefaultPermissionExceptionsLPw() {
        //【1】获得 /etc/default-permissions 文件目录对象！
        File dir = new File(Environment.getRootDirectory(), "etc/default-permissions");
        if (!dir.exists() || !dir.isDirectory() || !dir.canRead()) {
            return new ArrayMap<>(0);
        }

        File[] files = dir.listFiles();
        if (files == null) {
            return new ArrayMap<>(0);
        }

        ArrayMap<String, List<DefaultPermissionGrant>> grantExceptions = new ArrayMap<>();

        for (File file : files) {
            if (!file.getPath().endsWith(".xml")) {
                Slog.i(TAG, "Non-xml file " + file + " in " + dir + " directory, ignoring");
                continue;
            }
            if (!file.canRead()) {
                Slog.w(TAG, "Default permissions file " + file + " cannot be read");
                continue;
            }
            try (
                InputStream str = new BufferedInputStream(new FileInputStream(file))
            ) {
                XmlPullParser parser = Xml.newPullParser();
                parser.setInput(str, null);
                //【5.1.3.1.1】parse 解析该目录下的每一个 xml 文件，将结果保存到 grantExceptions 中！
                parse(parser, grantExceptions);
            } catch (XmlPullParserException | IOException e) {
                Slog.w(TAG, "Error reading default permissions file " + file, e);
            }
        }

        return grantExceptions;
    }

```
##### 5.1.3.1.1 DPGrantPolicy.parse
```java
    private void parse(XmlPullParser parser, Map<String, List<DefaultPermissionGrant>>
            outGrantExceptions) throws IOException, XmlPullParserException {
        final int outerDepth = parser.getDepth();
        int type;
        while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
                && (type != XmlPullParser.END_TAG || parser.getDepth() > outerDepth)) {
            if (type == XmlPullParser.END_TAG || type == XmlPullParser.TEXT) {
                continue;
            }
            if (TAG_EXCEPTIONS.equals(parser.getName())) { // 标签 exceptions
                //【5.1.3.1.2】parseExceptions 继续解析
                parseExceptions(parser, outGrantExceptions);
            } else {
                Log.e(TAG, "Unknown tag " + parser.getName());
            }
        }
    }
```
##### 5.1.3.1.2 DPGrantPolicy.parseExceptions

```java
    private void parseExceptions(XmlPullParser parser, Map<String, List<DefaultPermissionGrant>>
            outGrantExceptions) throws IOException, XmlPullParserException {
        final int outerDepth = parser.getDepth();
        int type;
        while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
                && (type != XmlPullParser.END_TAG || parser.getDepth() > outerDepth)) {
            if (type == XmlPullParser.END_TAG || type == XmlPullParser.TEXT) {
                continue;
            }
            if (TAG_EXCEPTION.equals(parser.getName())) { // exception 标签名；
                String packageName = parser.getAttributeValue(null, ATTR_PACKAGE); // package 属性名；

                List<DefaultPermissionGrant> packageExceptions =
                        outGrantExceptions.get(packageName);
                if (packageExceptions == null) {
                    //【1】要处理的应用必须是 system app！
                    PackageParser.Package pkg = getSystemPackageLPr(packageName);
                    if (pkg == null) {
                        Log.w(TAG, "Unknown package:" + packageName);
                        XmlUtils.skipCurrentTag(parser);
                        continue;
                    }

                    //【2】必须支持运行时权限！
                    if (!doesPackageSupportRuntimePermissions(pkg)) {
                        Log.w(TAG, "Skipping non supporting runtime permissions package:"
                                + packageName);
                        XmlUtils.skipCurrentTag(parser);
                        continue;
                    }
                    //【3】创建一个 list，保存权限信息！
                    packageExceptions = new ArrayList<>();
                    outGrantExceptions.put(packageName, packageExceptions);
                }
                //【5.1.3.1.3】调用 parsePermission 继续解析！
                parsePermission(parser, packageExceptions);
            } else {
                Log.e(TAG, "Unknown tag " + parser.getName() + "under <exceptions>");
            }
        }
    }
```
##### 5.1.3.1.3 DPGrantPolicy.parsePermission

```java
    private void parsePermission(XmlPullParser parser, List<DefaultPermissionGrant>
            outPackageExceptions) throws IOException, XmlPullParserException {
        final int outerDepth = parser.getDepth();
        int type;
        while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
                && (type != XmlPullParser.END_TAG || parser.getDepth() > outerDepth)) {
            if (type == XmlPullParser.END_TAG || type == XmlPullParser.TEXT) {
                continue;
            }

            if (TAG_PERMISSION.contains(parser.getName())) { // permission 标签！
                String name = parser.getAttributeValue(null, ATTR_NAME); // name 属性
                if (name == null) {
                    Log.w(TAG, "Mandatory name attribute missing for permission tag");
                    XmlUtils.skipCurrentTag(parser);
                    continue;
                }

                final boolean fixed = XmlUtils.readBooleanAttribute(parser, ATTR_FIXED); // fixed 属性

                //【5.1.3.1.4】封装为一个 DefaultPermissionGrant 对象！
                DefaultPermissionGrant exception = new DefaultPermissionGrant(name, fixed);
                outPackageExceptions.add(exception);
            } else {
                Log.e(TAG, "Unknown tag " + parser.getName() + "under <exception>");
            }
        }
    }
```
##### 5.1.3.1.4 new DefaultPermissionGrant

```java
    private static final class DefaultPermissionGrant {
        final String name;
        final boolean fixed;

        public DefaultPermissionGrant(String name, boolean fixed) {
            this.name = name;
            this.fixed = fixed;
        }
    }
```
DefaultPermissionGrant 对象的属性简单，这里不多看了！

### 5.1.4 DPGrantPolicy.grantRuntimePermissionsLPw - 默认授予核心方法

默认授予运行时权限，调用的是内部的两个 grantRuntimePermissionsLPw 方法：

```java
[1]private void grantRuntimePermissionsLPw(PackageParser.Package pkg, Set<String> permissions,
            boolean systemFixed, int userId) {...}

[2]private void grantRuntimePermissionsLPw(PackageParser.Package pkg, Set<String> permissions,
            boolean systemFixed, boolean isDefaultPhoneOrSms, int userId) {...}
```
其中，方法一会调用方法二，这是参数 boolean systemFixed 设置的是 false！

```java
    private void grantRuntimePermissionsLPw(PackageParser.Package pkg, Set<String> permissions,
            boolean systemFixed, int userId) {
        grantRuntimePermissionsLPw(pkg, permissions, systemFixed, false, userId);
    }
```

最后调用的是五参的同名方法：systemFixed 表示是否设置 system fix，isDefaultPhoneOrSms 表示是否是默认的 phone or sms app，permissions 为该 package 要被默认授予的运行时权限集合：
```java
    private void grantRuntimePermissionsLPw(PackageParser.Package pkg, Set<String> permissions,
            boolean systemFixed, boolean isDefaultPhoneOrSms, int userId) {
        //【1】再次校验应用是否有请求权限！
        if (pkg.requestedPermissions.isEmpty()) {
            return;
        }
        //【2】获得本次扫描的 system apk 的权限！
        List<String> requestedPermissions = pkg.requestedPermissions;
        Set<String> grantablePermissions = null;

        //【3】如果是默认的 phone app 或者 sms app，如果其更新过，我们会默认授予更新过的 app 申明的权限
        // 如果这是一个被更新过的其他类型的 system app，我们只会默认授予是那些 system 分区的被更新过得 app 申明过的权限！
        if (!isDefaultPhoneOrSms && pkg.isUpdatedSystemApp()) {
            PackageSetting sysPs = mService.mSettings.getDisabledSystemPkgLPr(pkg.packageName);
            if (sysPs != null) {
                if (sysPs.pkg.requestedPermissions.isEmpty()) {
                    return;
                }
                if (!requestedPermissions.equals(sysPs.pkg.requestedPermissions)) {
                    // 表示本次扫描的 package 申明过的权限；
                    grantablePermissions = new ArraySet<>(requestedPermissions);
                    // 替换为被更新过的 system app 申明的权限！！
                    requestedPermissions = sysPs.pkg.requestedPermissions;
                }
            }
        }

        //【4】遍历该 package 申明过的所有权限；
        final int grantablePermissionCount = requestedPermissions.size();
        for (int i = 0; i < grantablePermissionCount; i++) {
            String permission = requestedPermissions.get(i);
            //【4.1】如果这是一个被更新过的 system app，跳过那些 data 分区新 app 没有声明的权限！
            if (grantablePermissions != null && !grantablePermissions.contains(permission)) {
                continue;
            }

            //【4.2】接下来，就是授予权限权限的过程了！
            if (permissions.contains(permission)) {
                //【5.1.1.3.1】获得权限的 flags；
                final int flags = mService.getPermissionFlags(permission, pkg.packageName, userId);

                // If any flags are set to the permission, then it is either set in
                // its current state by the system or device/profile owner or the user.
                // In all these cases we do not want to clobber the current state.
                // Unless the caller wants to override user choices. The override is
                // to make sure we can grant the needed permission to the default
                // sms and phone apps after the user chooses this in the UI.
                //【4.2.1】如果该权限的 flags 没有设置任何标志位，或者是默认的 phone app 或者 sms app 会进入 IF 分支！
                if (flags == 0 || isDefaultPhoneOrSms) {
                    //【4.2.1.1】对于 isDefaultPhoneOrSms 我们会再判断下是否设置了 system fix 或者 policy fix 标志位!
                    // 如果设置了，那么我们不会修改标志位！
                    final int fixedFlags = PackageManager.FLAG_PERMISSION_SYSTEM_FIXED
                            | PackageManager.FLAG_PERMISSION_POLICY_FIXED;
                    if ((flags & fixedFlags) != 0) {
                        continue;
                    }
                    //【4.2.1.2】这里调用了 PMS 的 grantRuntimePermission 方法，授予权限，不多说了！
                    mService.grantRuntimePermission(pkg.packageName, permission, userId);
                    if (DEBUG) {
                        Log.i(TAG, "Granted " + (systemFixed ? "fixed " : "not fixed ")
                                + permission + " to default handler " + pkg.packageName);
                    }
                    
                    //【4.2.1.3】更新该运行时权限的 flags，先只设置 FLAG_PERMISSION_GRANTED_BY_DEFAULT 标志位
                    // 如果为 system fix，还会设置 FLAG_PERMISSION_SYSTEM_FIXED 位！
                    int newFlags = PackageManager.FLAG_PERMISSION_GRANTED_BY_DEFAULT;
                    if (systemFixed) {
                        newFlags |= PackageManager.FLAG_PERMISSION_SYSTEM_FIXED;
                    }
                    // 这里调用了 PMS 的 updatePermissionFlags 方法，更新权限的 flags 为 newFlags！
                    mService.updatePermissionFlags(permission, pkg.packageName,
                            newFlags, newFlags, userId);
                }

                //【4.2.2】如果权限被设置了 FLAG_PERMISSION_GRANTED_BY_DEFAULT 和 FLAG_PERMISSION_SYSTEM_FIXED 标志位
                // 但是本次授予是 no system fix，那么我们就要去掉 system fix 标志位！
                if ((flags & PackageManager.FLAG_PERMISSION_GRANTED_BY_DEFAULT) != 0
                        && (flags & PackageManager.FLAG_PERMISSION_SYSTEM_FIXED) != 0
                        && !systemFixed) {
                    if (DEBUG) {
                        Log.i(TAG, "Granted not fixed " + permission + " to default handler "
                                + pkg.packageName);
                    }
                    //【4.2.2.1】更新权限的 flags 去掉 system fix 标志位！
                    mService.updatePermissionFlags(permission, pkg.packageName,
                            PackageManager.FLAG_PERMISSION_SYSTEM_FIXED, 0, userId);
                }
            }
        }
    }
```
该方法我们就分析到这里！

#### 5.1.4.1 PackageManagerS.getPermissionFlags

该方法用于获得权限的 flags；
```java
    @Override
    public int getPermissionFlags(String name, String packageName, int userId) {
        if (!sUserManager.exists(userId)) {
            return 0;
        }
        //【1】首先是校验是否具有相应的权限！
        enforceGrantRevokeRuntimePermissionPermissions("getPermissionFlags");
        enforceCrossUserPermission(Binder.getCallingUid(), userId,
                true /* requireFullPermission */, false /* checkShell */,
                "getPermissionFlags");

        synchronized (mPackages) {
            final PackageParser.Package pkg = mPackages.get(packageName);
            if (pkg == null) {
                return 0;
            }

            final BasePermission bp = mSettings.mPermissions.get(name);
            if (bp == null) {
                return 0;
            }

            SettingBase sb = (SettingBase) pkg.mExtras;
            if (sb == null) {
                return 0;
            }
            //【2】查询权限的 Flags！
            PermissionsState permissionsState = sb.getPermissionsState();
            return permissionsState.getPermissionFlags(name, userId);
        }
    }
```
##### 5.1.4.1.1 PermissionsState.getPermissionFlags
```java
    public int getPermissionFlags(String name, int userId) {
        //【1】如果是运行时权限的话，那就返回运行时权限的 flags
        PermissionState installPermState = getInstallPermissionState(name);
        if (installPermState != null) {
            return installPermState.getFlags();
        }
        //【2】如果是安装时权限的话，那就返回安装时权限的 flags
        PermissionState runtimePermState = getRuntimePermissionState(name, userId);
        if (runtimePermState != null) {
            return runtimePermState.getFlags();
        }
        return 0;
    }
```
下面是具体的获取运行时权限或者安装时权限的权限状态的方法，很简单，不多说了！
```java
    public PermissionState getInstallPermissionState(String name) {
        return getPermissionState(name, UserHandle.USER_ALL);
    }

    public PermissionState getRuntimePermissionState(String name, int userId) {
        enforceValidUserId(userId);
        return getPermissionState(name, userId);
    }
```

## 5.2 清除那些陈旧不用的用户和应用

代码段如下：
```java
        reconcileUsers(StorageManager.UUID_PRIVATE_INTERNAL);
        reconcileApps(StorageManager.UUID_PRIVATE_INTERNAL);
```

### 5.2.1 PackageManagerService.reconcileUsers

该方法会检测清除不用的设备用户：
```java
    private void reconcileUsers(String volumeUuid) {
        final List<File> files = new ArrayList<>();
        // 收集 /data/user_de 目录下的所有文件；
        Collections.addAll(files, FileUtils
                .listFilesOrEmpty(Environment.getDataUserDeDirectory(volumeUuid)));
        // 收集 /data/user 目录下的所有文件；
        Collections.addAll(files, FileUtils
                .listFilesOrEmpty(Environment.getDataUserCeDirectory(volumeUuid)));
        // 收集 /data/system_de 目录下的所有文件；
        Collections.addAll(files, FileUtils
                .listFilesOrEmpty(Environment.getDataSystemDeDirectory()));
        // 收集 /data/system 目录下的所有文件；
        Collections.addAll(files, FileUtils
                .listFilesOrEmpty(Environment.getDataSystemCeDirectory()));
        for (File file : files) {
            if (!file.isDirectory()) continue;

            final int userId;
            final UserInfo info;
            try {
                userId = Integer.parseInt(file.getName());
                // 尝试获得设备用户对应的 UserInfo！
                info = sUserManager.getUserInfo(userId);
            } catch (NumberFormatException e) {
                Slog.w(TAG, "Invalid user directory " + file);
                continue;
            }
            // 判断用户是否无效！
            boolean destroyUser = false;
            if (info == null) {
                logCriticalInfo(Log.WARN, "Destroying user directory " + file
                        + " because no matching user was found");
                destroyUser = true;
            } else if (!mOnlyCore) {
                try {
                    UserManagerService.enforceSerialNumber(file, info.serialNumber);
                } catch (IOException e) {
                    logCriticalInfo(Log.WARN, "Destroying user directory " + file
                            + " because we failed to enforce serial number: " + e);
                    destroyUser = true;
                }
            }
            //【5.2.1.1】如果设备用户无效了，清楚该用户的所有数据！
            if (destroyUser) {
                synchronized (mInstallLock) {
                    destroyUserDataLI(volumeUuid, userId,
                            StorageManager.FLAG_STORAGE_DE | StorageManager.FLAG_STORAGE_CE);
                }
            }
        }
    }
```
第一步，收集了 /data/user_de，/data/user，/data/system_de 和 /data/system 目录下的文件，这些文件的名字，都是以 userId 开头的！

#### 5.2.1.1 PackageManagerService.destroyUserDataLI
```java
    private void destroyUserDataLI(String volumeUuid, int userId, int flags) {
        final StorageManager storage = mContext.getSystemService(StorageManager.class);
        try {
            //【1】通过 Installer 来删除该用户下的 app data, profile data, and media data
            mInstaller.destroyUserData(volumeUuid, userId, flags);

            //【2】删除 system data！
            if (Objects.equals(volumeUuid, StorageManager.UUID_PRIVATE_INTERNAL)) {
                if ((flags & StorageManager.FLAG_STORAGE_DE) != 0) {
                    FileUtils.deleteContentsAndDir(Environment.getUserSystemDirectory(userId));
                    FileUtils.deleteContentsAndDir(Environment.getDataSystemDeDirectory(userId));
                }
                if ((flags & StorageManager.FLAG_STORAGE_CE) != 0) {
                    FileUtils.deleteContentsAndDir(Environment.getDataSystemCeDirectory(userId));
                }
            }
            //【3】取消挂载！
            storage.destroyUserStorage(volumeUuid, userId, flags);

        } catch (Exception e) {
            logCriticalInfo(Log.WARN,
                    "Failed to destroy user " + userId + " on volume " + volumeUuid + ": " + e);
        }
    }
```
该函数的作用很简单，不多说了！

### 5.2.2 PackageManagerService.reconcileApps

清除那些在该用户下已经卸载的，无效的应用！

```java
    private void reconcileApps(String volumeUuid) {
        //【1】手机 /data/app 目录下的所有文件
        final File[] files = FileUtils
                .listFilesOrEmpty(Environment.getDataAppDirectory(volumeUuid));
        for (File file : files) {
            final boolean isPackage = (isApkFile(file) || file.isDirectory())
                    && !PackageInstallerService.isStageName(file.getName());
            if (!isPackage) {
                // Ignore entries which are not packages
                continue;
            }

            try {
                // 再次解析该 package，返回 PackageLite 对象！
                final PackageLite pkg = PackageParser.parsePackageLite(file,
                        PackageParser.PARSE_MUST_BE_APK);
                //【4.2.2.1】判断该 package 是否有效，无效会抛出一个异常！
                assertPackageKnown(volumeUuid, pkg.packageName);

            } catch (PackageParserException | PackageManagerException e) {
                logCriticalInfo(Log.WARN, "Destroying " + file + " due to: " + e);
                synchronized (mInstallLock) {
                    //【4.2.2.2】package 无效，删除该 apk！
                    removeCodePathLI(file);
                }
            }
        }
    }
```

#### 5.2.2.1 PackageManagerService.assertPackageKnown
```java
    private void assertPackageKnown(String volumeUuid, String packageName)
            throws PackageManagerException {
        synchronized (mPackages) {
            //【1】如果重命名过了，转为以前的名字；
            packageName = normalizePackageNameLPr(packageName);
            final PackageSetting ps = mSettings.mPackages.get(packageName);
            //【2】如果系统中没有该 apk 的安装信息，或者其所处的 volume 异常，那么无效，抛出异常！
            if (ps == null) {
                throw new PackageManagerException("Package " + packageName + " is unknown");
            } else if (!TextUtils.equals(volumeUuid, ps.volumeUuid)) {
                throw new PackageManagerException(
                        "Package " + packageName + " found on unknown volume " + volumeUuid
                                + "; expected volume " + ps.volumeUuid);
            }
        }
    }
```
该方法用于判断 package 是否有效！

#### 5.2.2.2 PackageManagerService.removeCodePathLI
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
该方法删除指定的 apk 文件！