# ContentProvider篇 2 - ContentProvider 的启动
title: ContentProvider篇  2 - ContentProvider 的启动
date: 2018/04/13 20:46:25
catalog: true
categories:
- AndroidFramework源码分析
- ContentProvider内容提供者
tags: ContentProvider内容提供者
---

[toc]

基于 **Android 7.1.1**，分析 ContentProvider 的架构和原理。


# 0 综述

ContentProvider 是进程间通信的利器之一，充当数据存储和出具共享的中间者，其和核心是 Binder 和匿名共享内存！

我们在访问一个 ContentProvider 的时候，一般情况下都会先拉起该 ContentProvider 所在的进程，然后对其的增删改查：

```java
ContentResolver cr = getContentResolver();
Cursor cursor = cr.query(uri, null, null, null, null);
```
下面我们来分析下，其主要的流程：

# 1 ContextImpl

## 1.1 getContentResolver

```java
    @Override
    public ContentResolver getContentResolver() {
        return mContentResolver;
    }
```
这里的 mContentResolver 是一个 ApplicationContentResolver 的实例：

```java
    private final ApplicationContentResolver mContentResolver;
```

其创建是在 ContentImpl 初始化的时候：

```java
    private ContextImpl(@Nullable ContextImpl container, @NonNull ActivityThread mainThread,
            @NonNull LoadedApk packageInfo, @Nullable String splitName,
            @Nullable IBinder activityToken, @Nullable UserHandle user, int flags,
            @Nullable ClassLoader classLoader) {

        ... ... ... ...// 省略掉和 provider 无关的代码！

        //【1.2】创建 ApplicationContentResolver 实例！
        mContentResolver = new ApplicationContentResolver(this, mainThread, user);
    }
```

## 1.2 new ApplicationContentResolver

可以看到 ApplicationContentResolver 继承了 ContentResolver

```java
    private static final class ApplicationContentResolver extends ContentResolver {
        private final ActivityThread mMainThread;
        private final UserHandle mUser;

        public ApplicationContentResolver(
                Context context, ActivityThread mainThread, UserHandle user) {
            super(context);
            mMainThread = Preconditions.checkNotNull(mainThread);
            mUser = Preconditions.checkNotNull(user);
        }
        ... ... ...
    }
```
同时，ApplicationContentResolver 继承了 ContentResolver 中和增删改查相关的接口：

```java
public final @Nullable Cursor query(...) {...}
public final int update(...) {...}
public final @Nullable Uri insert(...) {...}
... ... ...
```

下面，我们通过分析 query 接口，来了解下 ContentProvider 的启动流程！


# 2 ContentResolver


## 2.1 query - 查询

下面我们来看下查询的调用！

```java
    public final @Nullable Cursor query(final @RequiresPermission.Read @NonNull Uri uri,
            @Nullable String[] projection, @Nullable String selection,
            @Nullable String[] selectionArgs, @Nullable String sortOrder,
            @Nullable CancellationSignal cancellationSignal) {
        //【1】检查 uri 是否为 null；
        Preconditions.checkNotNull(uri, "uri");
        
        //【×3.1】获得 unstable Provider 对象！
        IContentProvider unstableProvider = acquireUnstableProvider(uri);
        if (unstableProvider == null) {
            return null;
        }

        IContentProvider stableProvider = null;
        Cursor qCursor = null;
        try {
            long startTime = SystemClock.uptimeMillis();

            ICancellationSignal remoteCancellationSignal = null;
            if (cancellationSignal != null) {
                cancellationSignal.throwIfCanceled();
                remoteCancellationSignal = unstableProvider.createCancellationSignal();
                cancellationSignal.setRemote(remoteCancellationSignal);
            }
            try {
                //【2】通过 unstable Provider 查询，并返回游标 Cursor！
                qCursor = unstableProvider.query(mPackageName, uri, projection,
                        selection, selectionArgs, sortOrder, remoteCancellationSignal);

            } catch (DeadObjectException e) {
                //【3】如果 query 时远程进程死亡，我们会尝试恢复，但是恢复时，我们会获取 stable provider！
                
                //【*3.2】处理 unstable Provider 进程死亡的清理操作！！
                unstableProviderDied(unstableProvider);
    
                //【×3.3】获得 stable Provider 对象！
                stableProvider = acquireProvider(uri);
    
                if (stableProvider == null) {
                    return null;
                }
                //【4】通过 stable Provider 查询，并返回游标 Cursor！
                qCursor = stableProvider.query(mPackageName, uri, projection,
                        selection, selectionArgs, sortOrder, remoteCancellationSignal);
            }
            if (qCursor == null) {
                return null;
            }

            //【5】强制执行 query 的操作，可能会失败抛出运行时异常！！
            qCursor.getCount();
            long durationMillis = SystemClock.uptimeMillis() - startTime;
            maybeLogQueryToEventLog(durationMillis, uri, projection, selection, sortOrder);

            //【6】将查询结果 Cursor 封装成 CursorWrapperInner，并返回！！
            // 可以看到，我们最后访问的其实是 CursorWrapperInner！！
            final IContentProvider provider = (stableProvider != null) ? stableProvider
                    : acquireProvider(uri);
            final CursorWrapperInner wrapper = new CursorWrapperInner(qCursor, provider);
            stableProvider = null;
            qCursor = null;
            return wrapper;

        } catch (RemoteException e) {
            //【7】如果抛出运行时异常，ams 会杀掉该进程！
            return null;
        } finally {
            if (qCursor != null) {
                qCursor.close();
            }
            if (cancellationSignal != null) {
                cancellationSignal.setRemote(null);
            }
            if (unstableProvider != null) {
                releaseUnstableProvider(unstableProvider);
            }
            if (stableProvider != null) {
                releaseProvider(stableProvider);
            }
        }
    }
```
这里简单的说下，stable provider 和 unstable provider 的区别：

- 获取 stable provider 的进程会因为 provider 所在进程死亡而被杀死；而获取 unstable provider 的进程则不会出现这种情况；

具体的原因我们后面再分析！

这个流程很简单，不多说了！！

### 2.1.1 acquireUnstableProvider - 获取 unstable provider

```java
    public final IContentProvider acquireUnstableProvider(Uri uri) {
        //【1】如果 uri 的 scheme 不是 "content"，那就返回 null！
        if (!SCHEME_CONTENT.equals(uri.getScheme())) {
            return null;
        }
        //【2】如果 uri 没有指定 authority，也返回 null；
        String auth = uri.getAuthority();
        if (auth != null) {
            //【×3.1】继续调用！
            return acquireUnstableProvider(mContext, uri.getAuthority());
        }
        return null;
    }
```

### 2.1.2 acquireProvider - 获取 stable provider

```java
    public final IContentProvider acquireProvider(Uri uri) {
        //【1】如果 uri 的 scheme 不是 "content"，那就返回 null！
        if (!SCHEME_CONTENT.equals(uri.getScheme())) {
            return null;
        }
        //【2】如果 uri 没有指定 authority，也返回 null；
        final String auth = uri.getAuthority();
        if (auth != null) {
            //【×3.3】继续调用！
            return acquireProvider(mContext, auth);
        }
        return null;
    }
```

# 3 ApplicationContentResolver

## 3.1 acquireUnstableProvider  - 获取 unstable provider

获取 unstable provider

```java
    @Override
    protected IContentProvider acquireUnstableProvider(Context c, String auth) {
        //【×4.1】进入 ActivityThread
        return mMainThread.acquireProvider(c,
                ContentProvider.getAuthorityWithoutUserId(auth),
                resolveUserIdFromAuthority(auth), false);
    }
```

## 3.2 unstableProviderDied

处理 unstable provider 所在进程死亡后的工作！

```java
    @Override
    public void unstableProviderDied(IContentProvider icp) {
        //【×4.2】进入 ActivityThread
        mMainThread.handleUnstableProviderDied(icp.asBinder(), true);
    }
```


## 3.3 acquireProvider  - 获取 stable provider

获取 stable provider

```java
        @Override
        protected IContentProvider acquireProvider(Context context, String auth) {
            //【×4.1】进入 ActivityThread
            return mMainThread.acquireProvider(context,
                    ContentProvider.getAuthorityWithoutUserId(auth),
                    resolveUserIdFromAuthority(auth), true);
        }
```

可以看到 ApplicationContentResolver 最终都访问了 ActivityThread 中的逻辑！


# 4 ActivityThread

## 4.1 acquireProvider - 获取 provider

无论是 unstable provider 还是 stable provider，最终都会调用 ActivityThread 的 acquireProvider 方法，参数传递：

- **String auth**：authority 属性；
- **int userId**：设备用户 id；
- **boolean stable**：provider 的类型；

```java
    public final IContentProvider acquireProvider(
            Context c, String auth, int userId, boolean stable) {
        //【×4.1.1】尝试查询一个已经存在的 provider，如果有，那就返回这个 provider！
        final IContentProvider provider = acquireExistingProvider(c, auth, userId, stable);
        if (provider != null) {
            return provider;
        }

        IActivityManager.ContentProviderHolder holder = null;
        try {
            //【×5.1】如果没有已经存在的，那就需要通过 ams 来获取！
            holder = ActivityManagerNative.getDefault().getContentProvider(
                    getApplicationThread(), auth, userId, stable);
        } catch (RemoteException ex) {
            throw ex.rethrowFromSystemServer();
        }
        if (holder == null) {
            Slog.e(TAG, "Failed to find provider info for " + auth);
            return null;
        }

        //【×6.4】增加引用计数，将 provider 封装为 ContentProviderHolder 然后返回！
        holder = installProvider(c, holder, holder.info,
                true /*noisy*/, holder.noReleaseNeeded, stable);
        //【1】最后返回 holder.provider！
        return holder.provider;
    }
```
关于 holder 和 holder.provider 我们后面再分析！

继续来看：

### 4.1.1 acquireExistingProvider

尝试获得已经存在的一个 provider：

```java
    public final IContentProvider acquireExistingProvider(
            Context c, String auth, int userId, boolean stable) {
        synchronized (mProviderMap) {
            //【1】根据 authority 属性创建 ProviderKey 对象！
            final ProviderKey key = new ProviderKey(auth, userId);

            //【2】尝试获得已经存在的 ProviderClientRecord 客户端对象！
            final ProviderClientRecord pr = mProviderMap.get(key);
            if (pr == null) {
                return null;
            }

            //【3】判断 provdier 所在的进程是否死亡，这里的 provider 其实是一个 Transport 对象，
            // 目标进程的 provider 被拉起是会创建！
            IContentProvider provider = pr.mProvider;
            IBinder jBinder = provider.asBinder();
            if (!jBinder.isBinderAlive()) {
                //【3.1】如果 provider 所在进程死亡了，那么就返回 null！
                Log.i(TAG, "Acquiring provider " + auth + " for user " + userId
                        + ": existing object's process dead");
                //【×4.1.1.1】处理进程死亡后，unstable providre 的引用计数！
                handleUnstableProviderDiedLocked(jBinder, true);
                return null;
            }

            //【4】如果可以找到已经存在的合适的 provider，那么就增加客户端的引用计数！
            ProviderRefCount prc = mProviderRefCountMap.get(jBinder);
            if (prc != null) {
                //【×4.1.1.2】增加客户端的引用计数！
                incProviderRefLocked(prc, stable);
            }
            return provider;
        }
    }
```
主要逻辑如下：

- 尝试从 ActivityThread 的 mProviderMap 查询已存在相对应的 provider，若不存在，则返回 null；
- 如果 ActivityThread 存在该 provider ，但其所在的进程已经死亡，则调用 handleUnstableProviderDiedLocked 清理 provider 信息，并返回 null；
- 当 provider 记录存在，并且其进程仍然存活，则在 provider 引用计数不为空时继续增加引用计数。然后返回当前进程已经存在的 provider。



#### 4.1.1.1 handleUnstableProviderDiedLocked

如果 provdier 所在的进程已经死亡，那么我们会将该 provider 从引用它的进程中的相关结构体中移除！

boolean fromClient 是否是客户端移除的！

```java
    final void handleUnstableProviderDiedLocked(IBinder provider, boolean fromClient) {
        //【1】获得该进程对该 provider 的引用对象！
        ProviderRefCount prc = mProviderRefCountMap.get(provider);
        if (prc != null) {
            if (DEBUG_PROVIDER) Slog.v(TAG, "Cleaning up dead provider "
                    + provider + " " + prc.holder.info.name);
            //【1.1】从引用计数 mProviderRefCountMap 中移除该 provider 的信息！
            mProviderRefCountMap.remove(provider);
            for (int i = mProviderMap.size() - 1; i>=0; i--) {
                ProviderClientRecord pr = mProviderMap.valueAt(i);
                if (pr != null && pr.mProvider.asBinder() == provider) {
                    Slog.i(TAG, "Removing dead content provider:" + pr.mProvider.toString());
                    //【1.2】从 mProviderMap 中移除该 provider 的信息！
                    mProviderMap.removeAt(i);
                }
            }

            if (fromClient) {
                try {
                    //【5】通知 ams 该 provider 已经死亡！
                    ActivityManagerNative.getDefault().unstableProviderDied(
                            prc.holder.connection);
                } catch (RemoteException e) {
                    //do nothing content provider object is dead any way
                }
            }
        }
    }
```
这里的逻辑就不多说了！


#### 4.1.1.2 incProviderRefLocked

增加 provider 的引用计数， 参数 boolean stable 表示的是稳定引用，还是非稳定引用：

```java
    private final void incProviderRefLocked(ProviderRefCount prc, boolean stable) {
        //【1】stable 引用进入这里！
        if (stable) {
            prc.stableCount += 1; // 增加引用计数
            if (prc.stableCount == 1) {
                int unstableDelta;
                if (prc.removePending) { // 如果此时我们正在移除 provider，那就取消移除！
                    unstableDelta = -1;
                    if (DEBUG_PROVIDER) {
                        Slog.v(TAG, "incProviderRef: stable "
                                + "snatched provider from the jaws of death");
                    }
                    prc.removePending = false;
                    //【1.1】移除 REMOVE_PROVIDER 消息！
                    mH.removeMessages(H.REMOVE_PROVIDER, prc);
                } else {
                    unstableDelta = 0;
                }
                try {
                    if (DEBUG_PROVIDER) {
                        Slog.v(TAG, "incProviderRef Now stable - "
                                + prc.holder.info.name + ": unstableDelta="
                                + unstableDelta);
                    }
                    //【×7.2】通知 ams，增加框架层的引用信息！
                    ActivityManagerNative.getDefault().refContentProvider(
                            prc.holder.connection, 1, unstableDelta);
                } catch (RemoteException e) {
                }
            }
        } else {
        //【2】no stable 引用进入这里；
            prc.unstableCount += 1; // 增加引用计数
            if (prc.unstableCount == 1) {
                if (prc.removePending) { // 如果此时我们正在移除 provider，那就取消移除！
                    if (DEBUG_PROVIDER) {
                        Slog.v(TAG, "incProviderRef: unstable "
                                + "snatched provider from the jaws of death");
                    }
                    //【2.1】移除 REMOVE_PROVIDER 消息！
                    prc.removePending = false;
                    mH.removeMessages(H.REMOVE_PROVIDER, prc);
                } else {
                    try {
                        if (DEBUG_PROVIDER) {
                            Slog.v(TAG, "incProviderRef: Now unstable - "
                                    + prc.holder.info.name);
                        }
                        //【×7.2】通知 ams，增加框架层的引用信息！
                        ActivityManagerNative.getDefault().refContentProvider(
                                prc.holder.connection, 0, 1);
                    } catch (RemoteException e) {
                    }
                }
            }
        }
    }
```
可以看到，如果是第一次引用，那么会做一些额外的处理！


## 4.2 handleUnstableProviderDied

处理 unstable provider 死亡的清理操作，boolean fromClient 表示本次操作是否来自客户端进程，通过前面的参数传递                   

```java
    final void handleUnstableProviderDied(IBinder provider, boolean fromClient) {
        synchronized (mProviderMap) {
            //【×4.2.1】继续分析：
            handleUnstableProviderDiedLocked(provider, fromClient);
        }
    }
```
这里就不多说了，继续看！


### 4.2.1 handleUnstableProviderDiedLocked

根据参数，这里的 boolean fromClient 为 true，当该进程持有的 unstable provider 所在进程死亡后，
```java
    final void handleUnstableProviderDiedLocked(IBinder provider, boolean fromClient) {
        //【1】尝试从当前进程的引用集合中获得该 provider 的引用计数对象，并移除！
        ProviderRefCount prc = mProviderRefCountMap.get(provider);
        if (prc != null) {
            if (DEBUG_PROVIDER) Slog.v(TAG, "Cleaning up dead provider "
                    + provider + " " + prc.holder.info.name);
            mProviderRefCountMap.remove(provider);
            for (int i=mProviderMap.size()-1; i>=0; i--) {
                //【1.2】移除该进程持有的 provider 的 ProviderClientRecord 客户端对象！
                ProviderClientRecord pr = mProviderMap.valueAt(i);
                if (pr != null && pr.mProvider.asBinder() == provider) {
                    Slog.i(TAG, "Removing dead content provider:" + pr.mProvider.toString());
                    mProviderMap.removeAt(i);
                }
            }

            if (fromClient) {
                try {
                    //【×7.4】通知 ams，客户端进程移除了该 provider 相关的引用计数和客户端实例！
                    ActivityManagerNative.getDefault().unstableProviderDied(
                            prc.holder.connection);
                } catch (RemoteException e) {
                    //do nothing content provider object is dead any way
                }
            }
        }
    }
```
这里就不多说了！

# 5 ActivityManagerService  - 系统进程1

当我们拉起了 Content provider 所在的进程，并且执行了一些初始化的操作后，就需要将创建的 provider 注册到系统进程中进行管理！

## 5.1 getContentProvider[Impl]

下面我们来看 ActivityManagerService 中的 getContentProvider 方法：

```JAVA
    @Override
    public final ContentProviderHolder getContentProvider(
            IApplicationThread caller, String name, int userId, boolean stable) {
        enforceNotIsolatedCaller("getContentProvider");
        if (caller == null) {
            String msg = "null IApplicationThread when getting content provider "
                    + name;
            Slog.w(TAG, msg);
            throw new SecurityException(msg);
        }
        //【1】调用了另外一个 getContentProviderImpl 方法！
        return getContentProviderImpl(caller, name, null, stable, userId);
    }
```
继续来看，getContentProviderImpl 方法的逻辑很长，

```java
    private ContentProviderHolder getContentProviderImpl(IApplicationThread caller,
            String name, IBinder token, boolean stable, int userId) {
        //【1】在 ams 中每一个的 provider 都是以 ContentProviderRecord 的形式保存的!
        ContentProviderRecord cpr;
        //【2】用于保存和 provider 的连接关系！
        ContentProviderConnection conn = null;
        //【3】provider 的信息对象，在应用程序安装的时候会解析保存！
        ProviderInfo cpi = null;

        synchronized(this) {
            long startTime = SystemClock.uptimeMillis();
            //【4】判断访问者进程 r 是否存在，如果不存在，那就抛出异常，并返回！
            ProcessRecord r = null;
            if (caller != null) {
                r = getRecordForAppLocked(caller);
                if (r == null) {
                    throw new SecurityException(
                            "Unable to find app for caller " + caller
                          + " (pid=" + Binder.getCallingPid()
                          + ") when getting content provider " + name);
                }
            }

            boolean checkCrossUser = true;

            checkTime(startTime, "getContentProviderImpl: getProviderByName");

            //【5】判断该 provider 是否已经 publish 到该 userId 中了！
            cpr = mProviderMap.getProviderByName(name, userId);

            // 如果在 userId 下并没有 publish，那么就判断下，该 provider 是否只是给 USER_SYSTEM 使用的
            // 如果是，那么就要校验下是否是单例模式！
            if (cpr == null && userId != UserHandle.USER_SYSTEM) {
                cpr = mProviderMap.getProviderByName(name, UserHandle.USER_SYSTEM);
                if (cpr != null) {
                    cpi = cpr.info;
                    //【5.1】如果该 provider 是给 USER_SYSTEM 使用的，那么只有该 provider 是单例模式，并且
                    // 本地调用对于单例是有效的，那么才可以使用该 provider！
                    //【×5.1.1.1】通过 isSingleton 判断是否是单例模式！
                    //【×5.1.1.2】通过 isValidSingletonCall 方法判断方法访问单例 pvodier 
                    if (isSingleton(cpi.processName, cpi.applicationInfo,
                            cpi.name, cpi.flags)
                            && isValidSingletonCall(r.uid, cpi.applicationInfo.uid)) {
                        //【5.2】单例模式的 provider 运行在默认用户，所以 userId 被设置为了 USER_SYSTEM！
                        // 同时其他的用户可以访问！
                        userId = UserHandle.USER_SYSTEM;
                        checkCrossUser = false; // 不用校验 user 了！
                    } else {
                        cpr = null; // 不是单例模式，进入这里！
                        cpi = null;
                    }
                }
            }

            //【6】判断 provider 是否正在运行！
            boolean providerRunning = cpr != null && cpr.proc != null && !cpr.proc.killed;

            //【7】如果 provider 正在运行，即： cpr 不为 null，同时其进程存在，那么就直接返回该 provider！
            if (providerRunning) {
                cpi = cpr.info;
                String msg;
                checkTime(startTime, "getContentProviderImpl: before checkContentProviderPermission");
                //【×5.1.1.3】校验 provider 权限！
                if ((msg = checkContentProviderPermissionLocked(cpi, r, userId, checkCrossUser))
                        != null) {
                    throw new SecurityException(msg);
                }
                checkTime(startTime, "getContentProviderImpl: after checkContentProviderPermission");
                
                //【×5.1.1.4】如果访问者进程存在，并且 provider 能够在访问者的进程中运行的话，进入这里！
                // 该 provider 支持 multi process，或者 provider 和访问者属于同一进程，并且所属 userId 相同！
                if (r != null && cpr.canRunHere(r)) {
                    //【×5.1.1.5】那么这里会通过 provider 在 ams 中的 ContentProviderRecord 实例，创建一个
                    // ContentProviderHolder 实例，同时设置 holder.connection 和 holder.provider 为 null；
                    // 因为访问者会建立自己的本地 provider！
                    ContentProviderHolder holder = cpr.newHolder(null);
                    holder.provider = null;
                    return holder;
                }

                final long origId = Binder.clearCallingIdentity();
                checkTime(startTime, "getContentProviderImpl: incProviderCountLocked");

                //【×5.1.1.6】增加 provider 引用计数，并返回连接对象 ContentProviderConnection！
                conn = incProviderCountLocked(r, cpr, token, stable);
                if (conn != null && (conn.stableCount+conn.unstableCount) == 1) {
                    //【8】如果是第一次连接，那么会判断下访问者进程的 adj，如果优先级是可感知进程，或者比可感知进程高
                    // 那么会调整 provider 所在进程在 lruProcess 中的位置！
                    if (cpr.proc != null && r.setAdj <= ProcessList.PERCEPTIBLE_APP_ADJ) {
                        checkTime(startTime, "getContentProviderImpl: before updateLruProcess");
                        updateLruProcessLocked(cpr.proc, false, null);
                        checkTime(startTime, "getContentProviderImpl: after updateLruProcess");
                    }
                }
                checkTime(startTime, "getContentProviderImpl: before updateOomAdj");

                //【9】调整 provider 所在进程的优先级！
                final int verifiedAdj = cpr.proc.verifiedAdj;
                boolean success = updateOomAdjLocked(cpr.proc);
                
                //【10】异常校验，如果优先级调整结果是成功的，但是 provider 的进程 adj 校验不过，并且 provider 
                // 的进程不处于 alive 状态，那么这属于异常情况！
                if (success && verifiedAdj != cpr.proc.setAdj && !isProcessAliveLocked(cpr.proc)) {
                    success = false;
                }

                //【11】更新使用情况！
                maybeUpdateProviderUsageStatsLocked(r, cpr.info.packageName, name);

                checkTime(startTime, "getContentProviderImpl: after updateOomAdj");
                if (DEBUG_PROVIDER) Slog.i(TAG_PROVIDER, "Adjust success: " + success);

                //【12】如果 success 为 false，说明出现了异常，比如 provider 的进程被杀死了。
                // 那么我们需要启动一个新的进程，并且确认 provider 进程的死亡不会杀掉我们的进程！
                if (!success) {
                    Slog.i(TAG, "Existing provider " + cpr.name.flattenToShortString()
                            + " is crashing; detaching " + r);

                    //【×5.1.1.7】减少 provider 引用计数，因为前面尝试增加了！
                    // 并判断取消的引用是否是该进程对 provider 的最后引用！
                    boolean lastRef = decProviderCountLocked(conn, cpr, token, stable);
                    checkTime(startTime, "getContentProviderImpl: before appDied");

                    //【12.1】这个是处理 provider 进程死亡后的相关工作，这里不关注！
                    appDiedLocked(cpr.proc);

                    checkTime(startTime, "getContentProviderImpl: after appDied");
                    if (!lastRef) {
                        //【12.2】如果取消的引用不是该进程对 provider 的最后引用，返回一个 null！
                        return null;
                    }

                    //【12.3】设置 providerRunning 为 false，conn 为 null，下面我们会继续处理！
                    providerRunning = false;
                    conn = null;
                } else {
                    cpr.proc.verifiedAdj = cpr.proc.setAdj;
                }

                Binder.restoreCallingIdentity(origId);
            }

            //【8】如果 provider 没有在运行，进入这里！！
            if (!providerRunning) {
                try {
                    checkTime(startTime, "getContentProviderImpl: before resolveContentProvider");
                    //【8.1】查询 provider 的信息；
                    cpi = AppGlobals.getPackageManager().
                        resolveContentProvider(name,
                            STOCK_PM_FLAGS | PackageManager.GET_URI_PERMISSION_PATTERNS, userId);
                    checkTime(startTime, "getContentProviderImpl: after resolveContentProvider");
                } catch (RemoteException ex) {
                }
                if (cpi == null) {
                    return null;
                }

                //【8.2】如果 provider 是单例模式，同时且满足访问单例的条件，那么我们才允许访问 provider！
                //【×5.1.1.1】通过 isSingleton 判断是否是单例模式！
                //【×5.1.1.2】通过 isValidSingletonCall 方法判断是否可以访问单例 provider！
                boolean singleton = isSingleton(cpi.processName, cpi.applicationInfo,
                        cpi.name, cpi.flags)
                        && isValidSingletonCall(r.uid, cpi.applicationInfo.uid);
                if (singleton) { // 如果是单例访问，那么 userId 为 USER_SYSTEM！
                    userId = UserHandle.USER_SYSTEM;
                }

                //【8.2】获得 provider 在该 userId 下的 ApplicationInfo
                cpi.applicationInfo = getAppInfoForUser(cpi.applicationInfo, userId);
                checkTime(startTime, "getContentProviderImpl: got app info for user");

                String msg;
                checkTime(startTime, "getContentProviderImpl: before checkContentProviderPermission");

                //【×5.1.1.3】检验访问者是否有权限访问 provider！
                if ((msg = checkContentProviderPermissionLocked(cpi, r, userId, !singleton))
                        != null) {
                    throw new SecurityException(msg);
                }

                checkTime(startTime, "getContentProviderImpl: after checkContentProviderPermission");

                //【8.3】如果该 provider 是运行在其他进程中的，但是系统进程没有启动完成，
                // 那么也无法启动 provider 所在进程，抛出异常！！
                if (!mProcessesReady
                        && !cpi.processName.equals("system")) {
                    throw new IllegalArgumentException(
                            "Attempt to launch content provider before system ready");
                }

                //【8.4】确保 provider 所在的设备用户处于运行状态！
                if (!mUserController.isUserRunningLocked(userId, 0)) {
                    Slog.w(TAG, "Unable to launch app "
                            + cpi.applicationInfo.packageName + "/"
                            + cpi.applicationInfo.uid + " for provider "
                            + name + ": user " + userId + " is stopped");
                    return null;
                }
                //【8.5】封装 provider 的组件信息！
                ComponentName comp = new ComponentName(cpi.packageName, cpi.name);
                checkTime(startTime, "getContentProviderImpl: before getProviderByClass");
                
                //【×5.1.2.1】尝试通过组件名获得 provider 实例！
                cpr = mProviderMap.getProviderByClass(comp, userId);
                checkTime(startTime, "getContentProviderImpl: after getProviderByClass");
                final boolean firstClass = cpr == null;

                //【8.6】如果该 provider 并没有 publish 到系统中，说明这是第一次！
                // 那么我们会创建一个 ContentProviderRecord 实例！
                if (firstClass) {
                    final long ident = Binder.clearCallingIdentity();

                    //【8.6.1】如果运行任何组件前要重新确认权限，那么这里会拉起权限确认！
                    if (Build.PERMISSIONS_REVIEW_REQUIRED) {
                        if (!requestTargetProviderPermissionsReviewIfNeededLocked(cpi, r, userId)) {
                            return null;
                        }
                    }

                    try {
                        checkTime(startTime, "getContentProviderImpl: before getApplicationInfo");
                        //【8.6.2】获得应用程序的 ApplicationInfo 实例！！
                        ApplicationInfo ai =
                            AppGlobals.getPackageManager().
                                getApplicationInfo(
                                        cpi.applicationInfo.packageName,
                                        STOCK_PM_FLAGS, userId);
                        checkTime(startTime, "getContentProviderImpl: after getApplicationInfo");
                        if (ai == null) {
                            Slog.w(TAG, "No package info for content provider "
                                    + cpi.name);
                            return null;
                        }
                        //【8.6.3】根据本次访问的 userId，动态调整 ApplicationInfo 的信息！
                        ai = getAppInfoForUser(ai, userId);

                        //【×5.1.2.2】创建 provider 的 ContentProviderRecord 实例！！
                        cpr = new ContentProviderRecord(this, cpi, ai, comp, singleton);
                    } catch (RemoteException ex) {
                    } finally {
                        Binder.restoreCallingIdentity(ident);
                    }
                }

                checkTime(startTime, "getContentProviderImpl: now have ContentProviderRecord");
                
                //【×5.1.1.4】如果访问者进程存在，并且 provider 能够在访问者的进程中运行的话，进入这里！
                // 该 provider 支持 multi process，或者 provider 和访问者属于同一进程，并且所属 userId 相同！
                if (r != null && cpr.canRunHere(r)) {

                    //【×5.1.1.5】那么这里会通过 provider 在 ams 中的 ContentProviderRecord 实例，创建一个
                    // ContentProviderHolder 实例，同时设置 holder.connection 和 holder.provider 为 null；
                    //（holder.provider 在前面 new ContentProviderRecord 时就是 null 的）
                    // 因为访问者会建立自己的本地 provider！！
                    return cpr.newHolder(null);
                }

                if (DEBUG_PROVIDER) Slog.w(TAG_PROVIDER, "LAUNCHING REMOTE PROVIDER (myuid "
                            + (r != null ? r.uid : null) + " pruid " + cpr.appInfo.uid + "): "
                            + cpr.info.name + " callers=" + Debug.getCallers(6));

                //【8.7】判断 provider 是否正在启动中，每一个正在启动的 provider 都会被加入到 
                // mLaunchingProviders 列表中！！！
                final int N = mLaunchingProviders.size();
                int i;
                for (i = 0; i < N; i++) {
                    //【8.7.1】如果该 provider 正在启动，那么 i 不会超过 N！
                    if (mLaunchingProviders.get(i) == cpr) {
                        break;
                    }
                }

                //【8.8】如果 provider 还没有启动，那么我们会先启动 orovider！
                if (i >= N) {
                    final long origId = Binder.clearCallingIdentity();
                    try {
                        try {
                            checkTime(startTime, "getContentProviderImpl: before set stopped state");

                            //【8.8.1】provider 所属的 package 不能处于 stop 状态！
                            AppGlobals.getPackageManager().setPackageStoppedState(
                                    cpr.appInfo.packageName, false, userId);

                            checkTime(startTime, "getContentProviderImpl: after set stopped state");
                        } catch (RemoteException e) {
                        } catch (IllegalArgumentException e) {
                            Slog.w(TAG, "Failed trying to unstop package "
                                    + cpr.appInfo.packageName + ": " + e);
                        }

                        checkTime(startTime, "getContentProviderImpl: looking for process record");

                        //【8.8.2】获得 provider 所属的进程！
                        ProcessRecord proc = getProcessRecordLocked(
                                cpi.processName, cpr.appInfo.uid, false);

                        //【8.8.3】如果 provider 所属进程已经启动了，那我们就拉起该 provider！！
                        if (proc != null && proc.thread != null && !proc.killed) {
                            if (DEBUG_PROVIDER) Slog.d(TAG_PROVIDER,
                                    "Installing in existing process " + proc);

                            if (!proc.pubProviders.containsKey(cpi.name)) {
                                checkTime(startTime, "getContentProviderImpl: scheduling install");
    
                                //【8.8.3.1】如果该 provider 不在该进程的 pubProviders，将其添加其中！
                                proc.pubProviders.put(cpi.name, cpr);
                                try {
                                    //【×6.1】进入 provider 所在进程，拉起 provider！
                                    proc.thread.scheduleInstallProvider(cpi);
                                } catch (RemoteException e) {
                                }
                            }
                        } else {
                            checkTime(startTime, "getContentProviderImpl: before start process");
                            
                            //【8.8.4】如果该 provider 所在进程未启动，那么就启动所在进程！
                            proc = startProcessLocked(cpi.processName,
                                    cpr.appInfo, false, 0, "content provider",
                                    new ComponentName(cpi.applicationInfo.packageName,
                                            cpi.name), false, false, false);
                            checkTime(startTime, "getContentProviderImpl: after start process");
                            if (proc == null) {
                                Slog.w(TAG, "Unable to launch app "
                                        + cpi.applicationInfo.packageName + "/"
                                        + cpi.applicationInfo.uid + " for provider "
                                        + name + ": process is bad");
                                return null;
                            }
                        }
                        //【8.8.4.1】设置 cpr.launchingApp 为 proc，表示正在等待该进程启动！
                        cpr.launchingApp = proc;
                        //【8.8.4.2】将该 provider 加入到 mLaunchingProviders 中，表示其正在启动！
                        mLaunchingProviders.add(cpr);
                    } finally {
                        Binder.restoreCallingIdentity(origId);
                    }
                }

                checkTime(startTime, "getContentProviderImpl: updating data structures");

                //【9】将 provider 加入的 mProviderMap 中！
                if (firstClass) {
                    mProviderMap.putProviderByClass(comp, cpr);
                }
                mProviderMap.putProviderByName(name, cpr);
                
                //【×5.1.1.6】增加 provider 引用计数，并返回连接对象 ContentProviderConnection！！
                conn = incProviderCountLocked(r, cpr, token, stable);
                if (conn != null) {
                    conn.waiting = true;
                }
            }
            checkTime(startTime, "getContentProviderImpl: done!");
        }

        //【9】等待 provider publish 完成！
        synchronized (cpr) {
            //【9.1】当 cpr.provider 为 null 的时候，说明 provider 还没有 publish 完成，所以这里会持续等待！！
            while (cpr.provider == null) {
                if (cpr.launchingApp == null) {
                    //【9.3】 cpr.launchingApp 表示其正在等待启动的进程，如果为 null，说明无法启动 provider 所在
                    // 的进程，那么就返回 null！
                    Slog.w(TAG, "Unable to launch app "
                            + cpi.applicationInfo.packageName + "/"
                            + cpi.applicationInfo.uid + " for provider "
                            + name + ": launching app became null");
                    EventLog.writeEvent(EventLogTags.AM_PROVIDER_LOST_PROCESS,
                            UserHandle.getUserId(cpi.applicationInfo.uid),
                            cpi.applicationInfo.packageName,
                            cpi.applicationInfo.uid, name);
                    return null;
                }
                try {
                    if (DEBUG_MU) Slog.v(TAG_MU,
                            "Waiting to start provider " + cpr
                            + " launchingApp=" + cpr.launchingApp);
                    //【9.4】如果此时创建了连接对象，那么设置 conn.waiting 为 true，表示等待 provider 的 publish！！ 
                    if (conn != null) {
                        conn.waiting = true;
                    }
                    cpr.wait(); // Binder 线程加入了该对象的等待队列中等待条件满足！
                } catch (InterruptedException ex) {
                } finally {
                    if (conn != null) {
                        conn.waiting = false;
                    }
                }
            }
        }
        //【10】provider 启动完成了，返回一个 Holder 对象，此时我们已经创建了连接对象 conn！
        // 同时 cpr.provider 也不为 null；
        return cpr != null ? cpr.newHolder(conn) : null;
    }
```
在 ams 中有如下的集合：

```java
    //【1】用于保存正在启动的 provider！
    final ArrayList<ContentProviderRecord> mLaunchingProviders
            = new ArrayList<ContentProviderRecord>();
```


### 5.1.1  Provider 正在运行的情况

#### 5.1.1.1 isSingleton

判断该 provider 时候是否是单例模式！
```java
    boolean isSingleton(String componentProcessName, ApplicationInfo aInfo,
            String className, int flags) {
        boolean result = false;
        if (UserHandle.getAppId(aInfo.uid) >= Process.FIRST_APPLICATION_UID) {
            //【1】如果 provider 所属应用的 uid 大于等于 FIRST_APPLICATION_UID，那么其必须要设置 FLAG_SINGLE_USER
            // 标志位，同时也要被授予 INTERACT_ACROSS_USERS 权限才行！
            if ((flags & ServiceInfo.FLAG_SINGLE_USER) != 0) {
                if (ActivityManager.checkUidPermission(
                        INTERACT_ACROSS_USERS,
                        aInfo.uid) != PackageManager.PERMISSION_GRANTED) {
                    ComponentName comp = new ComponentName(aInfo.packageName, className);
                    String msg = "Permission Denial: Component " + comp.flattenToShortString()
                            + " requests FLAG_SINGLE_USER, but app does not hold "
                            + INTERACT_ACROSS_USERS;
                    Slog.w(TAG, msg);
                    throw new SecurityException(msg);
                }

                result = true;
            }
        } else if ("system".equals(componentProcessName)) {
            //【2】如果 1 满足，那么 provider 所在进程必须是 system 进程！
            result = true;

        } else if ((flags & ServiceInfo.FLAG_SINGLE_USER) != 0) {
            //【3】如果 1 和 2 都不满足，那么必须是 Phone app 或者 persistent apps，才能提供单例 provider！
            result = UserHandle.isSameApp(aInfo.uid, Process.PHONE_UID)
                    || (aInfo.flags & ApplicationInfo.FLAG_PERSISTENT) != 0;

        }
        if (DEBUG_MU) Slog.v(TAG_MU,
                "isSingleton(" + componentProcessName + ", " + aInfo + ", " + className + ", 0x"
                + Integer.toHexString(flags) + ") = " + result);

        return result;
    }
```
**属性：**

android:singleUser，取值为 true 或者 false，如果为 true，那么该 privider 将会是一个单例组件，系统中将有且只会存在一个单例组件运行在所有的设备用户下！



判断一个 provider 是否是单例模式，要满足一下条件之一：

- 如果 provider 所属 appId 大于等于 FIRST_APPLICATION_UID，并且其 flags 设置了 FLAG_SINGLE_USER 位，同时其被授予了 INTERACT_ACROSS_USERS 权限；

</br>

- 如果条件 1 不满足，那么如果 provider 所属进程是 system 进程，那么其就是单例的！

</br>

- 条件 1，2 都不满足，如果 provider 所属进程不是系统进程，同时其设置了 FLAG_SINGLE_USER 位，那么其所属应必须用是 phone app 或者是 persistent app，那么才是单例的！ 


逻辑很简单，不多说了！

#### 5.1.1.2 isValidSingletonCall

用于判断调用单例 provider 的操作是否有效：

```java
    boolean isValidSingletonCall(int callingUid, int componentUid) {
        int componentAppId = UserHandle.getAppId(componentUid);
        //【1】对于单例模式，调用者和 provider 必须属于同一个应用，或者 provider 组件属于 system/phone
        // 或者 provider 组件有垮用户的权限！
        return UserHandle.isSameApp(callingUid, componentUid)
                || componentAppId == Process.SYSTEM_UID
                || componentAppId == Process.PHONE_UID
                || ActivityManager.checkUidPermission(INTERACT_ACROSS_USERS_FULL, componentUid)
                        == PackageManager.PERMISSION_GRANTED;
    }
```
判断是否可以访问单例 provider，至少要满足以下的条件之一：

- 单例的 provider 组件和调用者是同一个 app；
- 单例的 provider 是 system uid 或者 phone uid；
- 单例的 provider 所属应用有 INTERACT_ACROSS_USERS_FULL 的权限！

不多说了！

#### 5.1.1.3 checkContentProviderPermissionLocked

```java
    private final String checkContentProviderPermissionLocked(
            ProviderInfo cpi, ProcessRecord r, int userId, boolean checkUser) {
        final int callingPid = (r != null) ? r.pid : Binder.getCallingPid();
        final int callingUid = (r != null) ? r.uid : Binder.getCallingUid();
        boolean checkedGrants = false;
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
        if (checkComponentPermission(cpi.readPermission, callingPid, callingUid,
                cpi.applicationInfo.uid, cpi.exported)
                == PackageManager.PERMISSION_GRANTED) {
            return null;
        }
        if (checkComponentPermission(cpi.writePermission, callingPid, callingUid,
                cpi.applicationInfo.uid, cpi.exported)
                == PackageManager.PERMISSION_GRANTED) {
            return null;
        }

        PathPermission[] pps = cpi.pathPermissions;
        if (pps != null) {
            int i = pps.length;
            while (i > 0) {
                i--;
                PathPermission pp = pps[i];
                String pprperm = pp.getReadPermission();
                if (pprperm != null && checkComponentPermission(pprperm, callingPid, callingUid,
                        cpi.applicationInfo.uid, cpi.exported)
                        == PackageManager.PERMISSION_GRANTED) {
                    return null;
                }
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

#### 5.1.1.4 ContentProviderRecord.canRunHere

判断该 provider 是否可以在指定的进程中运行：
```java
    public boolean canRunHere(ProcessRecord app) {
        //【1】provider 设置了 multiprocess 属性，或者 provider 的进程是该进程
        // 并且该 provider 所属 userId 和该进程所属应用程序的 userId 一样（相同用户下）！
        return (info.multiprocess || info.processName.equals(app.processName))
                && uid == app.info.uid;
    }
```
不多说了！

#### 5.1.1.5 ContentProviderRecord.newHolder

根据系统中已经 publish 的 provider，创建一个 ContentProviderHolder 对象！

```java
    public ContentProviderHolder newHolder(ContentProviderConnection conn) {
        ContentProviderHolder holder = new ContentProviderHolder(info);
        holder.provider = provider;
        holder.noReleaseNeeded = noReleaseNeeded;
        holder.connection = conn;
        return holder;
    }
```
关于 ContentProviderHolder，我们后面会分析！

#### 5.1.1.6 incProviderCountLocked

增加 provider 的引用计数，参数 r 是访问 provider 的进程：

```java
    ContentProviderConnection incProviderCountLocked(ProcessRecord r,
            final ContentProviderRecord cpr, IBinder externalProcessToken, boolean stable) {
        //【1】判断下访问者进程是否存在！
        if (r != null) {
            for (int i = 0; i < r.conProviders.size(); i++) {
                //【1.1】如果进程存在（当然是存在的），那就依次遍历其持有的 provider connection 对象！
                // 找到该进程持有的对该 provider 的连接对象！
                ContentProviderConnection conn = r.conProviders.get(i);
                if (conn.provider == cpr) {
                    if (DEBUG_PROVIDER) Slog.v(TAG_PROVIDER,
                            "Adding provider requested by "
                            + r.processName + " from process "
                            + cpr.info.processName + ": " + cpr.name.flattenToShortString()
                            + " scnt=" + conn.stableCount + " uscnt=" + conn.unstableCount);
                    //【1.2】如果是访问的是 stable provider，增加 stable count！
                    if (stable) {
                        conn.stableCount++;
                        conn.numStableIncs++;
                    } else {
                    //【1.3】如果是访问的是 unstable provider，增加 unstable count！
                        conn.unstableCount++;
                        conn.numUnstableIncs++;
                    }
                    //【1.4】返回该 connection 对象！
                    return conn;
                }
            }
            //【*5.1.1.6.1】如果找不到，说明是第一次访问该 provider，那就先创建
            // 一个 ContentProviderConnection 对象！！
            ContentProviderConnection conn = new ContentProviderConnection(cpr, r);
            //【1.5】增加引用计数！
            if (stable) {
                conn.stableCount = 1;
                conn.numStableIncs = 1;
            } else {
                conn.unstableCount = 1;
                conn.numUnstableIncs = 1;
            }
            //【1.6】将其添加到 ContentProviderRecord.connections 集合中；
            cpr.connections.add(conn);
            //【1.7】将其添加到 ProcessRecord.conProviders 集合中；
            r.conProviders.add(conn);
            startAssociationLocked(r.uid, r.processName, r.curProcState,
                    cpr.uid, cpr.name, cpr.info.processName); // 用于记录进程间的关联性的，这里先不关注！
            //【1.8】返回该新建的连接对象！
            return conn;
        }
        cpr.addExternalProcessHandleLocked(externalProcessToken);
        return null;
    }
```
不多说了！

##### 5.1.1.6.1 new ContentProviderConnection

创建连接对象，本质上是一个 Binder 对象：
```java
public final class ContentProviderConnection extends Binder {
    public final ContentProviderRecord provider; // provider 对象；
    public final ProcessRecord client; // 连接到该 provider 的进程；
    public final long createTime;
    public int stableCount; // stable provider 的引用计数；
    public int unstableCount; // unstable provider 的引用计数；
 
    public boolean waiting; // 是否正在等待 provider publish，被锁保护！
    public boolean dead; // provider 是否死亡！

    public int numStableIncs; // 用于调试！
    public int numUnstableIncs;
    
    public ContentProviderConnection(ContentProviderRecord _provider, ProcessRecord _client) {
        provider = _provider;
        client = _client;
        createTime = SystemClock.elapsedRealtime();
    }
    ... ... ...
}
```
数据结构都很简答，不多说了！！


#### 5.1.1.7 decProviderCountLocked

减少 provider 的引用计数，参数 conn 是该 provider 的连接对象：

```java
    boolean decProviderCountLocked(ContentProviderConnection conn,
            ContentProviderRecord cpr, IBinder externalProcessToken, boolean stable) {
        if (conn != null) {
            //【1】获得该连接对象链接的 provider 对爱 ContentProviderRecord！
            cpr = conn.provider;
            if (DEBUG_PROVIDER) Slog.v(TAG_PROVIDER,
                    "Removing provider requested by "
                    + conn.client.processName + " from process "
                    + cpr.info.processName + ": " + cpr.name.flattenToShortString()
                    + " scnt=" + conn.stableCount + " uscnt=" + conn.unstableCount);
            //【2】减少该连接对象的相应引用计数！
            if (stable) {
                conn.stableCount--;
            } else {
                conn.unstableCount--;
            }
            //【3】判断是否是该 provider 的最后一个引用，即本次减少引用后，该连接对象的 stable count 
            // 和 unstable count 都是 0。
            if (conn.stableCount == 0 && conn.unstableCount == 0) {
                //【3.1】ContentProviderRecord.connections 移除该连接对象；
                cpr.connections.remove(conn);
                //【3.2】从该连接对象的所属进程中移除自身！
                conn.client.conProviders.remove(conn);
                if (conn.client.setProcState < ActivityManager.PROCESS_STATE_LAST_ACTIVITY) {
                    // The client is more important than last activity -- note the time this
                    // is happening, so we keep the old provider process around a bit as last
                    // activity to avoid thrashing it.
                    if (cpr.proc != null) {
                        cpr.proc.lastProviderTime = SystemClock.uptimeMillis();
                    }
                }
                stopAssociationLocked(conn.client.uid, conn.client.processName, cpr.uid, cpr.name);
                //【3.3】返回 true，说明取消的是最后一个引用！
                return true;
            }
            return false;
        }
        cpr.removeExternalProcessHandleLocked(externalProcessToken);
        return false;
    }
```

不多说了！


### 5.1.2  Provider 没在运行的情况

#### 5.1.2.1 ProviderMap

在 ams 有一个这样的数据结构：

```java
        ProviderMap mProviderMap = new ProviderMap(this);
```
用来记录系统中所有 publish 的 provider：

```java
public final class ProviderMap {

    private static final String TAG = "ProviderMap";

    private static final boolean DBG = false;

    private final ActivityManagerService mAm;

    //【1】单例 provdier 集合，key 为 authority，value 为 ContentProviderRecord 实例！
    private final HashMap<String, ContentProviderRecord> mSingletonByName
            = new HashMap<String, ContentProviderRecord>();
    private final HashMap<ComponentName, ContentProviderRecord> mSingletonByClass
            = new HashMap<ComponentName, ContentProviderRecord>();
    //【2】非单例 provdier 集合，key 为 userId，value 是 authority 和 ContentProviderRecord 的映射！
    private final SparseArray<HashMap<String, ContentProviderRecord>> mProvidersByNamePerUser
            = new SparseArray<HashMap<String, ContentProviderRecord>>();
    private final SparseArray<HashMap<ComponentName, ContentProviderRecord>> mProvidersByClassPerUser
            = new SparseArray<HashMap<ComponentName, ContentProviderRecord>>();

    ProviderMap(ActivityManagerService am) {
        mAm = am;
    }
    ... ... ...
}
```
这里就不多说了！

同时也提供了如下的 get set 接口：

 5.1.2.2.1 ProviderMap.get


#### 5.1.2.2 new ContentProviderRecord

创建一个 ContentProviderRecord 实例，用于在系统进程中描述一个 provider！

```java
final class ContentProviderRecord {
    final ActivityManagerService service; // ams 实力
    
    public final ProviderInfo info; // provider 的信息对象；
    final int uid; // 所属应用的 uid；
    final ApplicationInfo appInfo; // 所属应用程序的信息；
    final ComponentName name; // provider 对应的组件名实例；
    final boolean singleton; // 是否是单例模式；
    public IContentProvider provider;
    public boolean noReleaseNeeded;

    //【1】所有连接到该 provider 的进程的连接对象！
    final ArrayList<ContentProviderConnection> connections
            = new ArrayList<ContentProviderConnection>();
            
    //final HashSet<ProcessRecord> clients = new HashSet<ProcessRecord>();
    // Handles for non-framework processes supported by this provider
    HashMap<IBinder, ExternalProcessHandle> externalProcessTokenToHandle;
    
    // Count for external process for which we have no handles.
    int externalProcessNoHandleCount;
    ProcessRecord proc; //【2】所在进程
    ProcessRecord launchingApp; //【3】等待启动的进程！
    String stringName;
    String shortStringName;

    public ContentProviderRecord(ActivityManagerService _service, ProviderInfo _info,
            ApplicationInfo ai, ComponentName _name, boolean _singleton) {
        service = _service;
        info = _info;
        uid = ai.uid;
        appInfo = ai;
        name = _name;
        singleton = _singleton;
        noReleaseNeeded = uid == 0 || uid == Process.SYSTEM_UID;
    }
    ... ... ...
}
```
继续分析！

### 5.1.3 ActivityManagerProxy.getContentProvider

对于 getContentProvider，我们再去看下 ActivityManagerProxy 中是如何调用的：

```java
    public ContentProviderHolder getContentProvider(IApplicationThread caller,
            String name, int userId, boolean stable) throws RemoteException {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IActivityManager.descriptor);
        data.writeStrongBinder(caller != null ? caller.asBinder() : null);
        data.writeString(name);
        data.writeInt(userId);
        data.writeInt(stable ? 1 : 0);
        mRemote.transact(GET_CONTENT_PROVIDER_TRANSACTION, data, reply, 0);
        reply.readException();
        int res = reply.readInt();
        ContentProviderHolder cph = null;
        if (res != 0) {
            //【*5.1.3.1】调用了 ContentProviderHolder.CREATOR 的 createFromParcel 方法！
            cph = ContentProviderHolder.CREATOR.createFromParcel(reply);
        }
        data.recycle();
        reply.recycle();
        return cph;
    }
```
getContentProvider 会返回一个 ContentProviderHolder 实例，ContentProviderHolder 看源码，实现了 Parcelable 接口，可以序列化!

#### 5.1.3.1 ContentProviderHolder.CREATOR.createFromParcel

我们来看下 ContentProviderHolder.CREATOR 的 createFromParcel 方法：

```java
    public static final Parcelable.Creator<ContentProviderHolder> CREATOR
            = new Parcelable.Creator<ContentProviderHolder>() {
        @Override
        public ContentProviderHolder createFromParcel(Parcel source) {
            //【*5.1.3.2】这里是通过服务端进程返回的 Parcel，再创建了一个 ContentProviderHolder！
            return new ContentProviderHolder(source);
        }

        @Override
        public ContentProviderHolder[] newArray(int size) {
            return new ContentProviderHolder[size];
        }
    };
```

#### 5.1.3.2 new ContentProviderHolder[Parcel]

这里通过 ContentProviderHolder 另外一个构造器创建了访问者进程中的 ContentProviderHolder 实例！

```java
    private ContentProviderHolder(Parcel source) {
        info = ProviderInfo.CREATOR.createFromParcel(source);
        //【1】初始化 provider 实例！
        provider = ContentProviderNative.asInterface(
                source.readStrongBinder());
        connection = source.readStrongBinder();
        noReleaseNeeded = source.readInt() != 0;
    }
```
我们知道在 provider 的宿主进程里，这里的 provider 是 Transport 实例！！

但是在访问者进程中，这里的 provider 是一个 ContentProviderProxy 实例，也就是 Transport 的客户端代理对象！

这样访问者进程就可以通过 ContentProviderProxy -> Transport 来跨进程通信啦！！

# 6 ActivityThread

## 6.1 provider 未启动但是其进程已经启动

### 6.1.1 scheduleInstallProvider

发送了 INSTALL_PROVIDER 消息：
```java
        @Override
        public void scheduleInstallProvider(ProviderInfo provider) {
            //【1】发送了 INSTALL_PROVIDER 消息！
            sendMessage(H.INSTALL_PROVIDER, provider);
        }
```

### 6.1.2 H.handleMessage[INSTALL_PROVIDER]

```java
        case INSTALL_PROVIDER:
            //【6.1.3】安装 provider！
            handleInstallProvider((ProviderInfo) msg.obj);
            break;
```

### 6.1.3 handleInstallProvider


```java
    public void handleInstallProvider(ProviderInfo info) {
        final StrictMode.ThreadPolicy oldPolicy = StrictMode.allowThreadDiskWrites();
        try {
            //【×6.3】安装 content provider！
            installContentProviders(mInitialApplication, Lists.newArrayList(info));
        } finally {
            StrictMode.setThreadPolicy(oldPolicy);
        }
    }
```

这里就不多说了！！


## 6.2 provider 未启动同时其进程也未启动

### 6.2.1 AMS.attachApplicationLocked

startProcessLocked 方法，先会调用 attachApplicationLocked 方法，这里省略掉了和 ContentProvider 无关的逻辑和代码：

attachApplicationLocked 我们在进程的启动中有分析过！

```java
    private final boolean attachApplicationLocked(IApplicationThread thread,
            int pid) {
        ... ... ... ...
        //【×6.2.1.1】获得进程中需要 install 和 publish 的 provider！
        boolean normalMode = mProcessesReady || isAllowedWhileBooting(app.info);
        List<ProviderInfo> providers = normalMode ? generateApplicationProvidersLocked(app) : null;

        //【×6.2.1.2】如果有 provider 正在等待该进程启动，那么就设置超时消息！
        if (providers != null && checkAppInLaunchingProvidersLocked(app)) {
            Message msg = mHandler.obtainMessage(CONTENT_PROVIDER_PUBLISH_TIMEOUT_MSG);
            msg.obj = app;
            mHandler.sendMessageDelayed(msg, CONTENT_PROVIDER_PUBLISH_TIMEOUT);
        }

        ... ... ... ...

        try {
            ... ... ... ...

            //【×6.2.2】bind 应用程序进程，这个地方，我们将 providers 作为参数传递到了 bindApplication
            // 方法里！
            thread.bindApplication(processName, appInfo, providers, app.instrumentationClass,
                    profilerInfo, app.instrumentationArguments, app.instrumentationWatcher,
                    app.instrumentationUiAutomationConnection, testMode,
                    mBinderTransactionTrackingEnabled, enableTrackAllocation,
                    isRestrictedBackupMode || !normalMode, app.persistent,
                    new Configuration(mConfiguration), app.compat,
                    getCommonServicesLocked(app.isolated),
                    mCoreSettingsObserver.getCoreSettingsLocked());

            //【1】更新 lru 进程列表！
            updateLruProcessLocked(app, false, null);
            app.lastRequestedGc = app.lastLowMemory = SystemClock.uptimeMillis();
        } catch (Exception e) {
            Slog.wtf(TAG, "Exception thrown during bind of " + app, e);
            app.resetPackageList(mProcessStats);
            app.unlinkDeathRecipient();
            startProcessLocked(app, "bind fail", processName);
            return false;
        }

        ... ... ...

        return true;
    }
```

#### 6.2.1.1 generateApplicationProvidersLocked

收集该进程中需要启动的 provider！
```java
    private final List<ProviderInfo> generateApplicationProvidersLocked(ProcessRecord app) {
        List<ProviderInfo> providers = null;
        try {
            //【1】获得该进程中所有 provider！
            providers = AppGlobals.getPackageManager()
                    .queryContentProviders(app.processName, app.uid,
                            STOCK_PM_FLAGS | PackageManager.GET_URI_PERMISSION_PATTERNS
                                    | MATCH_DEBUG_TRIAGED_MISSING)
                    .getList();
        } catch (RemoteException ex) {
        }

        if (DEBUG_MU) Slog.v(TAG_MU,
                "generateApplicationProvidersLocked, app.info.uid = " + app.uid);

        int userId = app.userId;

        //【2】遍历获得的所有的 provider！
        if (providers != null) {
            int N = providers.size();
            app.pubProviders.ensureCapacity(N + app.pubProviders.size());
            for (int i=0; i<N; i++) {
                ProviderInfo cpi =
                    (ProviderInfo)providers.get(i);
                
                //【×5.1.1.1】判断 provider 是否是单例模式的！
                boolean singleton = isSingleton(cpi.processName, cpi.applicationInfo,
                        cpi.name, cpi.flags);

                //【2.1】如果 provider 是单例模式，那么他们只能在默认用户下的进程中运行，如果
                // 进程的所属用户不是默认用户，那就不能运行！
                if (singleton && UserHandle.getUserId(app.uid) != UserHandle.USER_SYSTEM) {
                    //【2.1.1】将该 provider 从集合中移除！
                    providers.remove(i);
                    N--;
                    i--;
                    continue;
                }
                
                //【2.3】创建 provider 对应的组件对象，并创建 provider 对应的 ContentProviderRecord 实例
                // 将其添加到 mProviderMap 和 mProviderMap 中！
                ComponentName comp = new ComponentName(cpi.packageName, cpi.name);
                ContentProviderRecord cpr = mProviderMap.getProviderByClass(comp, userId);
                if (cpr == null) {
                    cpr = new ContentProviderRecord(this, cpi, app.info, comp, singleton);
                    mProviderMap.putProviderByClass(comp, cpr);
                }
                if (DEBUG_MU) Slog.v(TAG_MU,
                        "generateApplicationProvidersLocked, cpi.uid = " + cpr.uid);
                
                //【2.4】将该 provdier 添加到所属进程的 app.pubProviders 集合中！
                app.pubProviders.put(cpi.name, cpr);

                //【2.5】如果 provider 不是 multiprocess；
                // 或者 provider 是 multiprocess 同时其宿主进程名不是 android，我们会将其所属包名加入到该进程中！
                if (!cpi.multiprocess || !"android".equals(cpi.packageName)) {
                    // 如果是平台组件的话，那么是不会加入的，因为其属于框架层！
                    app.addPackage(cpi.applicationInfo.packageName, cpi.applicationInfo.versionCode,
                            mProcessStats);
                }
                notifyPackageUse(cpi.applicationInfo.packageName,
                                 PackageManager.NOTIFY_PACKAGE_USE_CONTENT_PROVIDER);
            }
        }
        //【2】返回要启动的 provider！
        return providers;
    }
```

这里就不多说了！！

#### 6.2.1.2 checkAppInLaunchingProvidersLocked

判断是否有 content provider 等待该进程的启动！

```java
    boolean checkAppInLaunchingProvidersLocked(ProcessRecord app) {
        //【1】遍历 mLaunchingProviders 集合，如果有 provider 所属进程属于该 process，返回 true！
        for (int i = mLaunchingProviders.size() - 1; i >= 0; i--) {
            ContentProviderRecord cpr = mLaunchingProviders.get(i);
            if (cpr.launchingApp == app) {
                return true;
            }
        }
        return false;
    }
```


### 6.2.2 handleBindApplication

startProcessLocked 方法，最后会调用 handleBindApplication 方法，这里省略掉了和 ContentProvider 无关的逻辑和代码：

handleBindApplication 我们在进程的启动中有分析过！

```java
    private void handleBindApplication(AppBindData data) {
        ... ... ... ...

        final StrictMode.ThreadPolicy savedPolicy = StrictMode.allowThreadDiskWrites();
        try {
            Application app = data.info.makeApplication(data.restrictedBackupMode, null);
            mInitialApplication = app;

            if (!data.restrictedBackupMode) {
                if (!ArrayUtils.isEmpty(data.providers)) {
                    //【×6.3】安装 content provider！！
                    installContentProviders(app, data.providers);
                    mH.sendEmptyMessageDelayed(H.ENABLE_JIT, 10*1000);
                }
            }

            try {
                mInstrumentation.onCreate(data.instrumentationArgs);
            }
            catch (Exception e) {
                throw new RuntimeException(
                    "Exception thrown in onCreate() of "
                    + data.instrumentationName + ": " + e.toString(), e);
            }

            try {
                mInstrumentation.callApplicationOnCreate(app);
            } catch (Exception e) {
                if (!mInstrumentation.onException(app, e)) {
                    throw new RuntimeException(
                        "Unable to create application " + app.getClass().getName()
                        + ": " + e.toString(), e);
                }
            }
        } finally {
            StrictMode.setThreadPolicy(savedPolicy);
        }
    }
```

## 6.3 installContentProviders - 安装 provider

安装 contentprovider：
```java
    private void installContentProviders(
            Context context, List<ProviderInfo> providers) {
        //【1】该进程中的所有 provider！
        final ArrayList<IActivityManager.ContentProviderHolder> results =
            new ArrayList<IActivityManager.ContentProviderHolder>();

        for (ProviderInfo cpi : providers) {
            if (DEBUG_PROVIDER) {
                StringBuilder buf = new StringBuilder(128);
                buf.append("Pub ");
                buf.append(cpi.authority);
                buf.append(": ");
                buf.append(cpi.name);
                Log.i(TAG, buf.toString());
            }
            //【×6.4】封装 provider！
            IActivityManager.ContentProviderHolder cph = installProvider(context, null, cpi,
                    false /*noisy*/, true /*noReleaseNeeded*/, true /*stable*/);
            
            //【2】设置 ContentProviderHolder 的 noReleaseNeeded 属性
            // 同时将其添加到 results 集合中！
            if (cph != null) {
                cph.noReleaseNeeded = true;
                results.add(cph);
            }
        }

        try {
            //【×7.1】将 provider 注册到 ams 中！
            ActivityManagerNative.getDefault().publishContentProviders(
                getApplicationThread(), results);
        } catch (RemoteException ex) {
            throw ex.rethrowFromSystemServer();
        }
    }
```
这里我们要注意下参数：

- **Context context**：进程的上下文环境！
- **IActivityManager.ContentProviderHolder holder**：null
- **ProviderInfo info**：provider 的信息对象；
- **boolean noisy**：false
- **boolean noReleaseNeeded**：true
- **boolean stable**：true



## 6.4 installProvider

封装 provider，返回对应的 ContentProviderHolder 实例，注意，这里由于 provider 的进程刚启动，所以 installProvider 的 

IActivityManager.ContentProviderHolder holder 参数为 null！
```java
    private IActivityManager.ContentProviderHolder installProvider(Context context,
            IActivityManager.ContentProviderHolder holder, ProviderInfo info,
            boolean noisy, boolean noReleaseNeeded, boolean stable) {
        ContentProvider localProvider = null;
        IContentProvider provider;

        //【1】如果 if 条件满足，说明进程要创建自己的本地 provider 实例！！
        if (holder == null || holder.provider == null) {
            if (DEBUG_PROVIDER || noisy) {
                Slog.d(TAG, "Loading provider " + info.authority + ": "
                        + info.name);
            }
            //【1.1】获取 Context 上下文环境！
            Context c = null;
            ApplicationInfo ai = info.applicationInfo;
            if (context.getPackageName().equals(ai.packageName)) {
                c = context;
            } else if (mInitialApplication != null &&
                    mInitialApplication.getPackageName().equals(ai.packageName)) {
                c = mInitialApplication;
            } else {
                try {
                    c = context.createPackageContext(ai.packageName,
                            Context.CONTEXT_INCLUDE_CODE);
                } catch (PackageManager.NameNotFoundException e) {
                }
            }
            if (c == null) {
                Slog.w(TAG, "Unable to get context for package " +
                      ai.packageName +
                      " while loading content provider " +
                      info.name);
                return null;
            }
            try {
                final java.lang.ClassLoader cl = c.getClassLoader();
                //【×6.4.1】第一次创建时，会通过反射，创建 ContentProvider 对象！
                localProvider = (ContentProvider)cl.
                    loadClass(info.name).newInstance();
                    
                //【×6.4.2】获得内部的 IContentProvider 对象，用于 Binder 通信；
                // 返回的是其内部的一个 Transport 实例！
                provider = localProvider.getIContentProvider();
                if (provider == null) {
                    Slog.e(TAG, "Failed to instantiate class " +
                          info.name + " from sourceDir " +
                          info.applicationInfo.sourceDir);
                    return null;
                }
                if (DEBUG_PROVIDER) Slog.v(
                    TAG, "Instantiating local provider " + info.name);

                //【*6.4.3】设置 Context，解析保存 providerInfo 中的信息！
                localProvider.attachInfo(c, info);

            } catch (java.lang.Exception e) {
                if (!mInstrumentation.onException(null, e)) {
                    throw new RuntimeException(
                            "Unable to get provider " + info.name
                            + ": " + e.toString(), e);
                }
                return null;
            }
        } else {
            //【1.2】如果 if 条件不满足，说明进程不需要创建本地 provider 而是需要获得远程的 provider 连接对象！
            // holder.provider 就是 6.4.2.1 的 Transport 实例，后面我们再看！！
            provider = holder.provider;
            if (DEBUG_PROVIDER) Slog.v(TAG, "Installing external provider " + info.authority + ": "
                    + info.name);
        }

        IActivityManager.ContentProviderHolder retHolder;

        synchronized (mProviderMap) {
            if (DEBUG_PROVIDER) Slog.v(TAG, "Checking to add " + provider
                    + " / " + info.name);
            //【2】获得 provider 的 Transport 对象！
            IBinder jBinder = provider.asBinder();

            //【3】如果进程需要创建自己的 local provider（比如 multi process 情况下），
            // 那么此时 localProvider 不为 null！
            if (localProvider != null) {
                //【3.1】创建对应的 ComponentName！
                ComponentName cname = new ComponentName(info.packageName, info.name);

                //【3.2】判断当前进程中是否已经有该 provider 对应的 ProviderClientRecord，当然，
                // 如果是第一次创建，是不会有的；
                ProviderClientRecord pr = mLocalProvidersByName.get(cname);
                if (pr != null) {
                    if (DEBUG_PROVIDER) {
                        Slog.v(TAG, "installProvider: lost the race, "
                                + "using existing local provider");
                    }

                    //【3.2.1】获得其 Transport 实例
                    provider = pr.mProvider;

                } else {
                    //【*6.4.4】如果本地还未创建，那就创建 ContentProviderHolder 对象！
                    // 并设置 holder.provider！！
                    holder = new IActivityManager.ContentProviderHolder(info);
                    holder.provider = provider;
                    holder.noReleaseNeeded = true;

                    //【*6.4.5】处理 authority 返回一个 ProviderClientRecord 对象！
                    pr = installProviderAuthoritiesLocked(provider, localProvider, holder);
                    
                    //【3.3】将应用关系保存到进程内部的指定集合中！
                    mLocalProviders.put(jBinder, pr);
                    mLocalProvidersByName.put(cname, pr);
                }

                //【2.4】将创建的 Holder 保存到 retHolder 中；
                retHolder = pr.mHolder;

            } else {
                //【4】如果进程不需要创建自己的 local provider，而是需要访问远程的 provider，
                // 那么我们需要和远程的 provider，建立引用关系！

                //【4.1】尝试根据 provider 的 Transport 对象，获得其对应的引用计数对象！
                ProviderRefCount prc = mProviderRefCountMap.get(jBinder);
                if (prc != null) {
                    //【2.5.1】如果引用计数对象存在，那就增加引用计数！
                    if (DEBUG_PROVIDER) {
                        Slog.v(TAG, "installProvider: lost the race, updating ref count");
                    }

                    if (!noReleaseNeeded) { // 如果需要 release 的话，我们会增加引用，同时释放掉旧的引用；
                        //【1.3.6】增加引用计数！
                        incProviderRefLocked(prc, stable);
                        try {
                            //【×2.2】释放旧的引用！
                            ActivityManagerNative.getDefault().removeContentProvider(
                                    holder.connection, stable);
                        } catch (RemoteException e) {
                        }
                    }
                } else {
                    //【2.5.2】如果引用计数对象为 null，那就会创建引用计数对象，这里会分为稳定引用和不稳定引用！
                    ProviderClientRecord client = installProviderAuthoritiesLocked(
                            provider, localProvider, holder);
                    
                    //【2.3.7】创建引用计数对象！
                    if (noReleaseNeeded) { // 如果这个引用无需释放，那么会设置为 1000！
                        prc = new ProviderRefCount(holder, client, 1000, 1000);
                    } else {
                        prc = stable
                                ? new ProviderRefCount(holder, client, 1, 0)
                                : new ProviderRefCount(holder, client, 0, 1);
                    }
                    // 将 Transport 和引用计数对象的映射关系保存到 mProviderRefCountMap 中！
                    mProviderRefCountMap.put(jBinder, prc);
                }
                retHolder = prc.holder;
            }
        }
        //【3】返回创建的 Holder！
        return retHolder;
    }
```
在 ActivityThread 中有如下和 provider 相关的集合：

```java
    //【1】用于保存 Transport 和对其他进程中的 provider 的引用计数 ProviderRefCount 的映射关系！
    final ArrayMap<IBinder, ProviderRefCount> mProviderRefCountMap
                                                = new ArrayMap<IBinder, ProviderRefCount>();

    //【2】用于保存 Transport 和 ProviderClientRecord 的映射关系！
    final ArrayMap<IBinder, ProviderClientRecord> mLocalProviders
                                                = new ArrayMap<IBinder, ProviderClientRecord>();

    //【3】用于保存 provide ComponentName 和 ProviderClientRecord 的映射关系！
    final ArrayMap<ComponentName, ProviderClientRecord> mLocalProvidersByName 
                                                = new ArrayMap<ComponentName, ProviderClientRecord>();
```


### 6.4.1 new ContentProvider

创建 ContentProvider 实例，表示该进程中的 provider！

```java
public abstract class ContentProvider implements ComponentCallbacks2 {
    private static final String TAG = "ContentProvider";

    private Context mContext = null; // provider 所在进程的上下文环境；
    private int mMyUid; // 所在进程 uid

    private String mAuthority;
    private String[] mAuthorities;
    private String mReadPermission; // 读权限
    private String mWritePermission; // 写权限
    private PathPermission[] mPathPermissions; // path 权限
    private boolean mExported; // 是否 export 属性
    private boolean mNoPerms;
    private boolean mSingleUser; // 是否是 single user

    private final ThreadLocal<String> mCallingPackage = new ThreadLocal<String>(); // 访问该 provider 的应用包名

    private Transport mTransport = new Transport(); // 这个很重要，下面会分析！

    public ContentProvider() {
    }
    ... ... ...
}
```
这里不多说！

### 6.4.2 ContentProvider.getIContentProvider

返回内部的一个 Transport 实例，该实例实现了 IContentProvider 接口！

```java
    public IContentProvider getIContentProvider() {
        //【*6.4.2.1】返回内部的 Transport 实例！
        return mTransport;
    }
```

#### 6.4.2.1 new Transport

```java
    class Transport extends ContentProviderNative {
        AppOpsManager mAppOpsManager = null;
        int mReadOp = AppOpsManager.OP_NONE;
        int mWriteOp = AppOpsManager.OP_NONE;
        ... ... ...
    }
```

Transport 继承了 ContentProviderNative，其本质上是一个 Binder 对象，用于跨进程通信，作为 Binder 通信的服务端：

```java
abstract public class ContentProviderNative extends Binder implements IContentProvider {
    public ContentProviderNative()
    {
        attachInterface(this, descriptor);
    }

    static public IContentProvider asInterface(IBinder obj)
    {
        if (obj == null) {
            return null;
        }
        IContentProvider in =
            (IContentProvider)obj.queryLocalInterface(descriptor);
        if (in != null) {
            return in;
        }

        return new ContentProviderProxy(obj);
    }

    public abstract String getProviderName();

    @Override
    public boolean onTransact(int code, Parcel data, Parcel reply, int flags)
            throws RemoteException {
        ... ... ... // 省略这部分逻辑，后面再分析！
    }

    public IBinder asBinder()
    {
        return this; // 返回的是自身！
    }
}
```
继续来看！！

### 6.4.3 ContentProvider.attachInfo[2]->[3]

attachInfo 方法的作用是，解析 provider 的属性！

```java
    public void attachInfo(Context context, ProviderInfo info) {
        //【1】调用三参数方法！
        attachInfo(context, info, false);
    }
```
继续分析：
```java
    private void attachInfo(Context context, ProviderInfo info, boolean testing) {
        mNoPerms = testing;

        if (mContext == null) {
            //【1】将当前进程的 Context 保存到 mContext 中！
            mContext = context;
            if (context != null) {
                //【2】初始化内部 Transport 的 appOps 属性！
                mTransport.mAppOpsManager = (AppOpsManager) context.getSystemService(
                        Context.APP_OPS_SERVICE);
            }
            //【3】初始化 uid
            mMyUid = Process.myUid();
            if (info != null) {
                //【*6.4.3.1】解析 provider 的属性！
                setReadPermission(info.readPermission);
                setWritePermission(info.writePermission);
                setPathPermissions(info.pathPermissions);
                mExported = info.exported;
                mSingleUser = (info.flags & ProviderInfo.FLAG_SINGLE_USER) != 0;
                setAuthorities(info.authority);
            }
            //【*6.4.3.2】拉起 provider 的 onCreate 方法！
            ContentProvider.this.onCreate();
        }
    }
```

#### 6.4.3.1 parse attribute

- **读/写权限**
```java
    protected final void setReadPermission(@Nullable String permission) {
        //【1】读权限；
        mReadPermission = permission;
    }
```

```java
    protected final void setWritePermission(@Nullable String permission) {
        //【1】写权限；
        mWritePermission = permission;
    }
```

- **Path 权限**
```java
    protected final void setPathPermissions(@Nullable PathPermission[] permissions) {
        //【1】path  权限；
        mPathPermissions = permissions;
    }
```

- **authority 权限**
```java
    protected final void setAuthorities(String authorities) {
        if (authorities != null) {
            if (authorities.indexOf(';') == -1) {
                mAuthority = authorities;
                mAuthorities = null;
            } else {
                mAuthority = null;
                mAuthorities = authorities.split(";");
            }
        }
    }
```
不多说了！


#### 6.4.3.2 ContentProvider.onCreate - 生命周期方法 onCreate

拉起 provider 的 onCreate 方法：

```java
    public abstract boolean onCreate();
```
当然，这个方法是抽象方法，因为我们 ContentProvider 这个类本身就是抽喜类，我们要实现自己的 provider 实例！！


### 6.4.4 new ContentProviderHolder

创建 ContentProviderHolder 对象！！

```java
    public static class ContentProviderHolder implements Parcelable {
        public final ProviderInfo info; // provider 数据对象！
        public IContentProvider provider; // 就是 Transport 对象！
        public IBinder connection; // provider 连接对象！
        public boolean noReleaseNeeded;

        public ContentProviderHolder(ProviderInfo _info) {
            info = _info;
        }
        ... ... ...
    }
```
看到了 ContentProviderHolder 是实现了 Parcelable 接口，可以跨进程传输！！

不多说，继续看：

### 6.4.5 installProviderAuthoritiesLocked

处理 provider 的 authority 属性，同时创建 ProviderClientRecord 对象并返回：

```java
    private ProviderClientRecord installProviderAuthoritiesLocked(IContentProvider provider,
            ContentProvider localProvider, IActivityManager.ContentProviderHolder holder) {
        //【1】获得 provider 的 authority 属性！
        final String auths[] = holder.info.authority.split(";");
        final int userId = UserHandle.getUserId(holder.info.applicationInfo.uid); // 应用程序的目标 userId
        
        //【*6.4.5.1】创建了 provider 对应的 ProviderClientRecord 对象！
        final ProviderClientRecord pcr = new ProviderClientRecord(
                auths, provider, localProvider, holder);
                
        //【2】将该 provider 的 authority 和其 ProviderClientRecord 的映射关系，保存到
        // mProviderMap 中！
        for (String auth : auths) {
            //【*6.4.5.2】创建 ProviderKey 对象！
            final ProviderKey key = new ProviderKey(auth, userId);
            final ProviderClientRecord existing = mProviderMap.get(key);

            if (existing != null) {
                //【2.1】已存在，就不会添加！
                Slog.w(TAG, "Content provider " + pcr.mHolder.info.name
                        + " already published as " + auth);
            } else {
                mProviderMap.put(key, pcr);
            }
        }
        //【3】返回 ProviderClientRecord 实例！
        return pcr;
    }
```

在 ActivityThread 中有一个 mProviderMap 哈希表：
```java
    //【1】保存该 authority 属性和其所属 provider 的 ProviderClientRecord 映射关系！
    final ArrayMap<ProviderKey, ProviderClientRecord> mProviderMap
        = new ArrayMap<ProviderKey, ProviderClientRecord>();
```
继续看！

#### 6.4.5.1 new ProviderClientRecord

创建 ProviderClientRecord 实例！

```java
    final class ProviderClientRecord {
        final String[] mNames; // authority 属性！
        final IContentProvider mProvider; // Transport 对象；
        final ContentProvider mLocalProvider; // ContentProvider 实例；
        final IActivityManager.ContentProviderHolder mHolder; // Holder 对象；

        ProviderClientRecord(String[] names, IContentProvider provider,
                ContentProvider localProvider,
                IActivityManager.ContentProviderHolder holder) {
            mNames = names;
            mProvider = provider;
            mLocalProvider = localProvider;
            mHolder = holder;
        }
    }
```
不多说！！

#### 6.4.5.2 new ProviderKey

创建 authority 属性对应的 ProviderKey 实例！

```java
    private static final class ProviderKey {
        final String authority; // authority 属性
        final int userId;

        public ProviderKey(String authority, int userId) {
            this.authority = authority;
            this.userId = userId;
        }
        ... ... ...
    }
```
不多说！


### 6.4.6 incProviderRefLocked

增加 provider 的引用计数， 参数 boolean stable 表示的是稳定引用，还是非稳定引用：

```java
    private final void incProviderRefLocked(ProviderRefCount prc, boolean stable) {
        //【1】stable 引用进入这里！
        if (stable) {
            prc.stableCount += 1; // 增加客户端引用计数
            if (prc.stableCount == 1) {
                int unstableDelta;
                if (prc.removePending) { // 如果此时我们正在移除 provider，那就取消移除！
                    unstableDelta = -1;
                    if (DEBUG_PROVIDER) {
                        Slog.v(TAG, "incProviderRef: stable "
                                + "snatched provider from the jaws of death");
                    }
                    prc.removePending = false;
                    mH.removeMessages(H.REMOVE_PROVIDER, prc);
                } else {
                    unstableDelta = 0;
                }
                try {
                    if (DEBUG_PROVIDER) {
                        Slog.v(TAG, "incProviderRef Now stable - "
                                + prc.holder.info.name + ": unstableDelta="
                                + unstableDelta);
                    }
                    //【*7.3】增加系统进程中 provider 的引用计数！
                    ActivityManagerNative.getDefault().refContentProvider(
                            prc.holder.connection, 1, unstableDelta);
                } catch (RemoteException e) {
                }
            }
        } else {
        //【2】no stable 引用进入这里；
            prc.unstableCount += 1; // 增加客户端引用计数
            if (prc.unstableCount == 1) {
                if (prc.removePending) { // 如果此时我们正在移除 provider，那就取消移除！
                    if (DEBUG_PROVIDER) {
                        Slog.v(TAG, "incProviderRef: unstable "
                                + "snatched provider from the jaws of death");
                    }
                    prc.removePending = false;
                    mH.removeMessages(H.REMOVE_PROVIDER, prc);
                } else {
                    try {
                        if (DEBUG_PROVIDER) {
                            Slog.v(TAG, "incProviderRef: Now unstable - "
                                    + prc.holder.info.name);
                        }
                        //【*7.3】增加系统进程中 provider 的引用计数！
                        ActivityManagerNative.getDefault().refContentProvider(
                                prc.holder.connection, 0, 1);
                    } catch (RemoteException e) {
                    }
                }
            }
        }
    }
```
可以看到，如果是第一次引用，那么会做一些额外的处理！

### 6.4.7 new ProviderRefCount

创建一个 provider 的引用计数对象：

```java
    private static final class ProviderRefCount {
        public final IActivityManager.ContentProviderHolder holder;
        public final ProviderClientRecord client;
        public int stableCount;
        public int unstableCount;

        // 如果为 true，那么 stable 和 unstable 的引用计数都为 0，同时 ams 准备移除引用计数
        // 但是在 ams 中依然持有一个 unstable 的引用！
        public boolean removePending;

        ProviderRefCount(IActivityManager.ContentProviderHolder inHolder,
                ProviderClientRecord inClient, int sCount, int uCount) {
            holder = inHolder;
            client = inClient;
            stableCount = sCount;
            unstableCount = uCount;
        }
    }
```
先看到这里！


# 7 ActivityManagerService - 系统进程2

这里我们又从 provider 进程进入了系统进程！


## 7.1 publishContentProviders

publishContentProviders 用于将 provider publish 到系统中！

首先来看下 ActivityManagerNative 中的代码，
                                                                            
```java
    case PUBLISH_CONTENT_PROVIDERS_TRANSACTION: {
        data.enforceInterface(IActivityManager.descriptor);
        IBinder b = data.readStrongBinder();
        //【1】获得 provider 所在的进程的 ApplicationThreadProxy
        IApplicationThread app = ApplicationThreadNative.asInterface(b);
        //【2】这里是从 provider 的 Holder 对象！
        ArrayList<ContentProviderHolder> providers =
            data.createTypedArrayList(ContentProviderHolder.CREATOR);
        //【3】调用 ams 的 publishContentProviders 方法！
        publishContentProviders(app, providers);
        reply.writeNoException();
        return true;
    }
```
接着，进入了 ActivityManagerService 中：
```java
    public final void publishContentProviders(IApplicationThread caller,
            List<ContentProviderHolder> providers) {
        if (providers == null) {
            return;
        }

        enforceNotIsolatedCaller("publishContentProviders");
        synchronized (this) {
            //【1】获得 provider 所在的进程 ProcessRecord！
            final ProcessRecord r = getRecordForAppLocked(caller);
            if (DEBUG_MU) Slog.v(TAG_MU, "ProcessRecord uid = " + r.uid);
            if (r == null) {
                throw new SecurityException(
                        "Unable to find app for caller " + caller
                      + " (pid=" + Binder.getCallingPid()
                      + ") when publishing content providers");
            }

            final long origId = Binder.clearCallingIdentity();

            final int N = providers.size();
            for (int i = 0; i < N; i++) {
                ContentProviderHolder src = providers.get(i);
                if (src == null || src.info == null || src.provider == null) {
                    continue;
                }
                //【2】我们知道 provider 对应的 ContentProviderRecord 在前面就已经添加到 r.pubProviders 中了！
                ContentProviderRecord dst = r.pubProviders.get(src.info.name);
                if (DEBUG_MU) Slog.v(TAG_MU, "ContentProviderRecord uid = " + dst.uid);
                if (dst != null) {
                    //【2.1】将 ContentProviderRecord 实例添加到 ProviderMap 中！
                    ComponentName comp = new ComponentName(dst.info.packageName, dst.info.name);
                    mProviderMap.putProviderByClass(comp, dst);
                    String names[] = dst.info.authority.split(";");
                    for (int j = 0; j < names.length; j++) {
                        mProviderMap.putProviderByName(names[j], dst);
                    }

                    int launchingCount = mLaunchingProviders.size();
                    int j;
                    //【2.2】判断其是否在 mLaunchingProviders 中，如果有，那就从中移除！
                    boolean wasInLaunchingProviders = false;
                    for (j = 0; j < launchingCount; j++) {
                        if (mLaunchingProviders.get(j) == dst) {
                            mLaunchingProviders.remove(j);
                            wasInLaunchingProviders = true;
                            j--;
                            launchingCount--;
                        }
                    }
                    //【2.3】移除 provider publish 超时消息！
                    if (wasInLaunchingProviders) {
                        mHandler.removeMessages(CONTENT_PROVIDER_PUBLISH_TIMEOUT_MSG, r);
                    }
                    synchronized (dst) {
                        //【2.4】更新 ContentProviderRecord 中的属性，包括 dst.provider 等等！
                        dst.provider = src.provider;
                        dst.proc = r;
                        //【important】这里调用了 Object.notifyAll 方法，唤醒在等待队列中等待的 Binder 线程
                        // 这里就回到了 5.1 getContentProvider[Impl] 方法中了！
                        dst.notifyAll();
                    }
                    // 更新进程的优先级！
                    updateOomAdjLocked(r);
                    // 更新进程的使用情况！
                    maybeUpdateProviderUsageStatsLocked(r, src.info.packageName,
                            src.info.authority);
                }
            }

            Binder.restoreCallingIdentity(origId);
        }
    }
```
不多说了！

## 7.2 removeContentProvider

减少系统进程中的 provider 的引用计数！

```java
        case REMOVE_CONTENT_PROVIDER_TRANSACTION: {
            data.enforceInterface(IActivityManager.descriptor);
            IBinder b = data.readStrongBinder();
            boolean stable = data.readInt() != 0;
            //【1】调用了 removeContentProvider 方法！
            removeContentProvider(b, stable);
            reply.writeNoException();
            return true;
        }
```
继续来看：
```java
    public void removeContentProvider(IBinder connection, boolean stable) {
        enforceNotIsolatedCaller("removeContentProvider");
        long ident = Binder.clearCallingIdentity();
        try {
            synchronized (this) {
                ContentProviderConnection conn;
                try {
                    //【1】获得 ContentProviderConnection 连接对象！
                    conn = (ContentProviderConnection)connection;
                } catch (ClassCastException e) {
                    String msg ="removeContentProvider: " + connection
                            + " not a ContentProviderConnection";
                    Slog.w(TAG, msg);
                    throw new IllegalArgumentException(msg);
                }
                if (conn == null) {
                    throw new NullPointerException("connection is null");
                }
                //【×5.1.1.7】减少对 provider 的引用计数！
                if (decProviderCountLocked(conn, null, null, stable)) {
                    updateOomAdjLocked(); // 调整进程优先级！
                }
            }
        } finally {
            Binder.restoreCallingIdentity(ident);
        }
    }
```
这里就不多说了！


## 7.3 refContentProvider

增加系统进程中的 provider 的引用计数！

```java
    public boolean refContentProvider(IBinder connection, int stable, int unstable)
            throws RemoteException {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IActivityManager.descriptor);
        data.writeStrongBinder(connection);
        data.writeInt(stable);
        data.writeInt(unstable);
        mRemote.transact(REF_CONTENT_PROVIDER_TRANSACTION, data, reply, 0);
        reply.readException();
        boolean res = reply.readInt() != 0;
        data.recycle();
        reply.recycle();
        return res;
    }
```
继续来看：

```java
    public boolean refContentProvider(IBinder connection, int stable, int unstable) {
        //【1】获得 ContentProviderConnection 连接对象！
        ContentProviderConnection conn;
        try {
            conn = (ContentProviderConnection)connection;
        } catch (ClassCastException e) {
            String msg ="refContentProvider: " + connection
                    + " not a ContentProviderConnection";
            Slog.w(TAG, msg);
            throw new IllegalArgumentException(msg);
        }
        if (conn == null) {
            throw new NullPointerException("connection is null");
        }

        synchronized (this) {
            if (stable > 0) { // 统计总的引用数，debug 用不关注！
                conn.numStableIncs += stable;
            }
            //【2】调整 stable connect 计数！
            stable = conn.stableCount + stable;
            if (stable < 0) {
                throw new IllegalStateException("stableCount < 0: " + stable);
            }
            
            if (unstable > 0) { // 统计总的引用数，debug 用不关注！
                conn.numUnstableIncs += unstable;
            }
            //【3】调整 unstable connect 计数！
            unstable = conn.unstableCount + unstable;
            if (unstable < 0) {
                throw new IllegalStateException("unstableCount < 0: " + unstable);
            }

            if ((stable+unstable) <= 0) {
                throw new IllegalStateException("ref counts can't go to zero here: stable="
                        + stable + " unstable=" + unstable);
            }
            //【4】更新 ContentProviderConnection 中的引用计数！
            conn.stableCount = stable;
            conn.unstableCount = unstable;
            return !conn.dead;
        }
    }
```
不多说了！

## 7.4 unstableProviderDied

处理 unstable provider 进程的死亡后事：

```java
    public void unstableProviderDied(IBinder connection) {
        ContentProviderConnection conn;
        try {
            //【1】获得引用的连接对象！
            conn = (ContentProviderConnection)connection;
        } catch (ClassCastException e) {
            String msg ="refContentProvider: " + connection
                    + " not a ContentProviderConnection";
            Slog.w(TAG, msg);
            throw new IllegalArgumentException(msg);
        }
        if (conn == null) {
            throw new NullPointerException("connection is null");
        }

        //【2】获得该 conn 对象绑定着的 provider，这里是返回 provider 内部的 Transport 实例！
        IContentProvider provider;
        synchronized (this) {
            provider = conn.provider.provider;
        }

        if (provider == null) {
            return;
        }

        //【3】判读客户端的连接是否仍然存在，如果是，那就直接返回！
        if (provider.asBinder().pingBinder()) {
            synchronized (this) {
                Slog.w(TAG, "unstableProviderDied: caller " + Binder.getCallingUid()
                        + " says " + conn + " died, but we don't agree");
                return;
            }
        }

        //【4】从处理进程死亡的后事！
        synchronized (this) {
            //【4.1】发生了神奇的变化，返回！
            if (conn.provider.provider != provider) {
                return;
            }
            //【4.2】如果进程已经被清理了，那就返回！
            ProcessRecord proc = conn.provider.proc;
            if (proc == null || proc.thread == null) {
                return;
            }

            Slog.i(TAG, "Process " + proc.processName + " (pid " + proc.pid
                    + ") early provider death");
            final long ident = Binder.clearCallingIdentity();
            try {
                //【4.3】处理进程的死亡后事！
                appDiedLocked(proc);
            } finally {
                Binder.restoreCallingIdentity(ident);
            }
        }
    }
```
不多说了！


# 8 总结

我们来看下 provider 相关的类图：

<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/23/ContentProvider-N-20221024234755771.png" alt="ContentProvider-N.png-486.9kB" style="zoom:50%;" />

