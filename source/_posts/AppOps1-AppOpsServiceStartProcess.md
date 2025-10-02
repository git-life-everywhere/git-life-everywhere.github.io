# AppOps 第 1 篇 - AppOpsService 的启动
title: AppOps 第 1 篇 - AppOpsService 的启动
date: 2017/08/01 20:46:25
catalog: true
categories:
- AndroidFramework源码分析
- AppOps应用操作管理
tags: AppOps应用操作管理
---

[toc]

本文基于 Android 7.1.1 源码分析，如有错误，欢迎指正，谢谢！

# 0 综述

AppOpsService 的初始化是在 ActivityManagerService 启动的时候：

```java
    public ActivityManagerService(Context systemContext) {
        ... ... ...   
        mHandlerThread = new ServiceThread(TAG,
                android.os.Process.THREAD_PRIORITY_FOREGROUND, false /*allowIo*/);
        mHandlerThread.start();
        mHandler = new MainHandler(mHandlerThread.getLooper());
        
        ... ... ... 
        // TODO: Move creation of battery stats service outside of activity manager service.
        File dataDir = Environment.getDataDirectory();
        File systemDir = new File(dataDir, "system");
        systemDir.mkdirs();

        ... ... ...
        //【1】创建了 AppOpsService 对象！
        mAppOpsService = new AppOpsService(new File(systemDir, "appops.xml"), mHandler);
        //【2】启动监听，开始监听是否有应用要在后台运行！
        mAppOpsService.startWatchingMode(AppOpsManager.OP_RUN_IN_BACKGROUND, null,
                new IAppOpsCallback.Stub() {
                    @Override public void opChanged(int op, int uid, String packageName) {
                        if (op == AppOpsManager.OP_RUN_IN_BACKGROUND && packageName != null) {
                            if (mAppOpsService.checkOperation(op, uid, packageName)
                                    != AppOpsManager.MODE_ALLOWED) {
                                runInBackgroundDisabled(uid);
                            }
                        }
                    }
                });
        ... ... ...
    }
```

# 1 new AppOpsService

创建 AppOpsService 的时候，传入了一个 File 对象，用于访问 /data/system/appops.xml 文件！

```java
    public AppOpsService(File storagePath, Handler handler) {
        mFile = new AtomicFile(storagePath);
        mHandler = handler;
        //【2】从 /data/system/appops.xml 文件中读取记录！
        readState();
    }
```
我们来看下，appops.xml 文件的内容：

```java
<?xml version='1.0' encoding='utf-8' standalone='yes' ?>
<app-ops>
    <uid n="1001">
        <op m="0" n="15"/>
    </uid>
    <pkg n="android">
        <uid n="1000" p="true">
            <op n="0" pu="0" t="1514736165168"/>
            <op n="1" pu="0" t="1514738472778"/>
            <op d="133" n="3" t="1514752786349"/>
            <op n="24" r="1514738089301"/>
            <op n="40" t="1514756174224"/>
            <op d="20027926" n="41" t="1514736155960"/>
            <op n="61" pu="0" t="1514756174065"/>
        </uid>
    </pkg>
    <pkg n="com.android.settings">
        <uid n="1000" p="true">
            <op d="135" n="3" t="1514738402132"/>
            <op m="0" n="15"/>
            <op n="59" pp="com.android.providers.media" pu="0" t="1514738336945"/>
            <op n="60" pu="0" t="1514736151388"/>
            <op n="63" pu="0" t="1514749896628"/>
            <op n="73" pu="0" t="1514749896649"/>
        </uid>
    </pkg>
    <pkg n="com.android.systemui">
        <uid n="1000" p="true">
            <op d="74" n="3" t="1514752786572"/>
            <op n="24" r="1514741582746"/>
            <op d="98" n="40" t="1514752835557"/>
            <op n="59" pu="0" t="493576354"/>
            <op n="60" pu="0" t="493576354"/>
        </uid>
    </pkg>
</app-ops>
```
内容我们先简单的分析下：

- uid 用于记录和 uid 相关的限制信息；
- pkg 用于记录和具体应用的限制信息；


# 2 AppOpsService.readState

AppOpsService 启动的时候，会读取本地持久化文件，恢复上一次的数据！
```java
    void readState() {
        synchronized (mFile) {
            synchronized (this) {
                FileInputStream stream;
                try {
                    stream = mFile.openRead();
                } catch (FileNotFoundException e) {
                    Slog.i(TAG, "No existing app ops " + mFile.getBaseFile() + "; starting empty");
                    return;
                }
                boolean success = false;
                mUidStates.clear();
                try {
                    XmlPullParser parser = Xml.newPullParser();
                    parser.setInput(stream, StandardCharsets.UTF_8.name());
                    int type;
                    while ((type = parser.next()) != XmlPullParser.START_TAG
                            && type != XmlPullParser.END_DOCUMENT) {
                        ;
                    }

                    if (type != XmlPullParser.START_TAG) {
                        throw new IllegalStateException("no start tag found");
                    }

                    int outerDepth = parser.getDepth();
                    while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
                            && (type != XmlPullParser.END_TAG || parser.getDepth() > outerDepth)) {
                        if (type == XmlPullParser.END_TAG || type == XmlPullParser.TEXT) {
                            continue;
                        }

                        String tagName = parser.getName();
                        if (tagName.equals("pkg")) {
                            //【1】读取 pkg 标签；
                            readPackage(parser);
                        } else if (tagName.equals("uid")) {
                            //【2】读取 uid 标签；
                            readUidOps(parser);
                        } else {
                            Slog.w(TAG, "Unknown element under <app-ops>: "
                                    + parser.getName());
                            XmlUtils.skipCurrentTag(parser);
                        }
                    }
                    success = true;
                } catch (IllegalStateException e) {
                    Slog.w(TAG, "Failed parsing " + e);
                } catch (NullPointerException e) {
                    Slog.w(TAG, "Failed parsing " + e);
                } catch (NumberFormatException e) {
                    Slog.w(TAG, "Failed parsing " + e);
                } catch (XmlPullParserException e) {
                    Slog.w(TAG, "Failed parsing " + e);
                } catch (IOException e) {
                    Slog.w(TAG, "Failed parsing " + e);
                } catch (IndexOutOfBoundsException e) {
                    Slog.w(TAG, "Failed parsing " + e);
                } finally {
                    if (!success) {
                        mUidStates.clear();
                    }
                    try {
                        stream.close();
                    } catch (IOException e) {
                    }
                }
            }
        }
    }
```
readState 会 pkg 和 uid 标签的数据，然后保存到内部的数据结构中！

## 2.1 AppOpsService.readUid[1] - 读取 uid 的 op 信息

首先，解析的是 uid 信息！

```java
    <uid n="1001">
        <op m="0" n="15"/>
    </uid>
```
解释一下：

- uid 的 n 属性表示 uid 的取值；
- op 的 n 属性表示 op 的数值，m 表示 op 的 mode 值；

我们来看看解析的过程！

```java
    void readUidOps(XmlPullParser parser) throws NumberFormatException,
            XmlPullParserException, IOException {
        final int uid = Integer.parseInt(parser.getAttributeValue(null, "n"));
        int outerDepth = parser.getDepth();
        int type;
        while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
                && (type != XmlPullParser.END_TAG || parser.getDepth() > outerDepth)) {
            if (type == XmlPullParser.END_TAG || type == XmlPullParser.TEXT) {
                continue;
            }

            String tagName = parser.getName();
            if (tagName.equals("op")) { // 解析 op 子标签！
                final int code = Integer.parseInt(parser.getAttributeValue(null, "n")); //
                final int mode = Integer.parseInt(parser.getAttributeValue(null, "m"));
                //【2.1.1】调用 getUidStateLocked 获得 uid 对应的状态信息，如果没有，就会创建一个！
                UidState uidState = getUidStateLocked(uid, true);
                if (uidState.opModes == null) {
                    uidState.opModes = new SparseIntArray();
                }
                uidState.opModes.put(code, mode);
            } else {
                Slog.w(TAG, "Unknown element under <uid-ops>: "
                        + parser.getName());
                XmlUtils.skipCurrentTag(parser);
            }
        }
    }
```
我们看到，创建了一个 UidState 对象，封装对 uid 的 op 控制，uidState.opModes 则保存了该 uid 的所有 op 和控制情况的映射关系！

### 2.1.1 getUidStateLocked
```java
    private UidState getUidStateLocked(int uid, boolean edit) {
        // 通过 uid，获得对应的 UidState 对象！
        UidState uidState = mUidStates.get(uid);
        if (uidState == null) {
            if (!edit) {
                return null;
            }
            //【2.1.2】创建一个新的 UidState，并添加到 mUidStates 中！
            uidState = new UidState(uid);
            mUidStates.put(uid, uidState);
        }
        return uidState;
    }
```

AppOpsService 有一个内部的 SparseArray，用于保存 uid 和其对应的 UidState 的映射关系！
```java
    private final SparseArray<UidState> mUidStates = new SparseArray<>();
```

### 2.1.2 new UidState
```java
    private static final class UidState {
        public final int uid; // uid 值
        public ArrayMap<String, Ops> pkgOps; //【2.1.2.1】属于 uid 的所有 package 和其 Op 集合的映射关系！
        public SparseIntArray opModes; // 用于保存该 uid 的 op 集合 ！

        public UidState(int uid) {
            this.uid = uid;
        }

        public void clear() {
            pkgOps = null;
            opModes = null;
        }

        public boolean isDefault() {
            return (pkgOps == null || pkgOps.isEmpty())
                    && (opModes == null || opModes.size() <= 0);
        }
    }
```
UidState.pkgOps 是一个 ArrayMap，key 为 packageName，而 value 为 Ops，Ops 用于封装该 package 所有的 Op 控制数据！

这里涉及到了一个 Ops 类，我们来看下他的结构!

#### 2.1.2.1 new Ops

可以看到 Ops 本质上是一个 SparseArray，用于记录 package 的所有 Op 控制信息！
```java
    public final static class Ops extends SparseArray<Op> {
        public final String packageName; // 所属的 package
        public final UidState uidState; // 所属的 UidState
        public final boolean isPrivileged;

        public Ops(String _packageName, UidState _uidState, boolean _isPrivileged) {
            packageName = _packageName;
            uidState = _uidState;
            isPrivileged = _isPrivileged;
        }
    }
```
保存的规则是：下标为 op，值为控制情况！

## 2.2 AppOpsService.readPackage  - 读取 package 的 op 信息

我们来看下 readPackage 解析的数据：

```java
    <pkg n="com.android.settings"> // readPackage 解析
        <uid n="1000" p="true"> // readUid[2] 解析
            <op d="135" n="3" t="1514738402132"/>
            <op m="0" n="15"/>
            <op n="59" pp="com.android.providers.media" pu="0" t="1514738336945"/>
            <op n="60" pu="0" t="1514736151388"/>
            <op n="63" pu="0" t="1514749896628"/>
            <op n="73" pu="0" t="1514749896649"/>
        </uid>
    </pkg>
```
解释一下：

- pkg 的 n 属性表示包名；
- uid 的 n 表示应用包所属的 uid 值，uid 的 p 表示应用包是否是特权应用；
- op 的 n 属性表示 op 的数值，m 表示 op 的 mode 值；


首先是解析 pkg 标签！
```java
    void readPackage(XmlPullParser parser) throws NumberFormatException,
            XmlPullParserException, IOException {
        //【1】获取包名 package name！
        String pkgName = parser.getAttributeValue(null, "n");
        int outerDepth = parser.getDepth();
        int type;
        while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
                && (type != XmlPullParser.END_TAG || parser.getDepth() > outerDepth)) {
            if (type == XmlPullParser.END_TAG || type == XmlPullParser.TEXT) {
                continue;
            }

            String tagName = parser.getName();
            if (tagName.equals("uid")) { 
                //【2.1.1】这里的 uid 标签是 pkg 标签的子标签！
                readUid(parser, pkgName);
            } else {
                Slog.w(TAG, "Unknown element under <pkg>: "
                        + parser.getName());
                XmlUtils.skipCurrentTag(parser);
            }
        }
    }
```
继续来看：

### 2.2.1 AppOpsService.readUid[2]

然后是解析子 uid 标签！
```java
    void readUid(XmlPullParser parser, String pkgName) throws NumberFormatException,
            XmlPullParserException, IOException {
        //【1】解析 n 属性，uid；
        int uid = Integer.parseInt(parser.getAttributeValue(null, "n"));
        //【2】解析 p 属性，是否是特权应用；
        String isPrivilegedString = parser.getAttributeValue(null, "p");
        boolean isPrivileged = false;
        if (isPrivilegedString == null) {
            try {
                IPackageManager packageManager = ActivityThread.getPackageManager();
                if (packageManager != null) {
                    //【3】获得 pkg 对应的 ApplicationInfo 对象！
                    ApplicationInfo appInfo = ActivityThread.getPackageManager()
                            .getApplicationInfo(pkgName, 0, UserHandle.getUserId(uid));
                    if (appInfo != null) {
                        isPrivileged = (appInfo.privateFlags
                                & ApplicationInfo.PRIVATE_FLAG_PRIVILEGED) != 0;
                    }
                } else {
                    return;
                }
            } catch (RemoteException e) {
                Slog.w(TAG, "Could not contact PackageManager", e);
            }
        } else {
            //【4】是否是特权应用！
            isPrivileged = Boolean.parseBoolean(isPrivilegedString);
        }
        int outerDepth = parser.getDepth();
        int type;
        while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
                && (type != XmlPullParser.END_TAG || parser.getDepth() > outerDepth)) {
            if (type == XmlPullParser.END_TAG || type == XmlPullParser.TEXT) {
                continue;
            }

            String tagName = parser.getName();
            //【5】解析 op 标签！
            if (tagName.equals("op")) {
                //【2.2.1.1】创建一个 Op 对象，封装 op 标签的属性！
                // 解析 n 属性，表示 op 的类别，保存到 Op.op！
                Op op = new Op(uid, pkgName, Integer.parseInt(parser.getAttributeValue(null, "n")));
                String mode = parser.getAttributeValue(null, "m"); // 解析 m 属性，用于获取该 op 的 mode 情况！
                if (mode != null) {
                    op.mode = Integer.parseInt(mode);
                }
                String time = parser.getAttributeValue(null, "t"); // 解析 t 属性，用于获取该 op 的授予情况！
                if (time != null) {
                    op.time = Long.parseLong(time);
                }
                time = parser.getAttributeValue(null, "r");  // 解析 r 属性，用于获取该 op 的拒绝时间！
                if (time != null) {
                    op.rejectTime = Long.parseLong(time);
                }
                String dur = parser.getAttributeValue(null, "d");  // 解析 d 属性，用于获取该 op 的时间间隔！
                if (dur != null) {
                    op.duration = Integer.parseInt(dur);
                }
                String proxyUid = parser.getAttributeValue(null, "pu");  // 解析 pu 属性，用于获取该 op 的代理 uid！
                if (proxyUid != null) {
                    op.proxyUid = Integer.parseInt(proxyUid);
                }
                String proxyPackageName = parser.getAttributeValue(null, "pp");  // 解析 pp 属性，用于获取该 op 的代理 pkg！
                if (proxyPackageName != null) {
                    op.proxyPackageName = proxyPackageName;
                }
                
                //【2.1.1】获得 uid 对应的 UidState 状态对象！
                UidState uidState = getUidStateLocked(uid, true);
                if (uidState.pkgOps == null) {
                    // 如果 uidState.pkgOps 为 null，初始化！
                    uidState.pkgOps = new ArrayMap<>();
                }
                //【2.2.1.1】从 uidState.pkgOps 获得该 package 的 Ops 集合
                // 如果为 null，会创建一个新的 Ops！
                Ops ops = uidState.pkgOps.get(pkgName);
                if (ops == null) {
                    ops = new Ops(pkgName, uidState, isPrivileged);
                    // 将其添加到 uidState.pkgOps 中！
                    uidState.pkgOps.put(pkgName, ops);
                }
                // 保存 op 类别和对应的控制情况！
                ops.put(op.op, op);
            } else {
                Slog.w(TAG, "Unknown element under <pkg>: "
                        + parser.getName());
                XmlUtils.skipCurrentTag(parser);
            }
        }
    }
```
整个过程很简单，不多说了！

#### 2.2.1.1 new Op

创建一个 Op 对象，封装 package 的每一个 Operation 信息！
```java
    public final static class Op {
        public final int uid; // 所属 uid
        public final String packageName; // 所属 package
        public int proxyUid = -1;
        public String proxyPackageName;
        public final int op; // 操作码，表示一个访问操作！
        public int mode; // 操作的控制信息！
        public int duration;
        public long time;
        public long rejectTime;
        public int nesting;

        public Op(int _uid, String _packageName, int _op) {
            uid = _uid;
            packageName = _packageName;
            op = _op;
            mode = AppOpsManager.opToDefaultMode(op); // 计算 op 的默认控制状态！
        }
    }
```

# 3 AppOpsService.startWatchingMode

在 AMS 的构造函数中，接着启动了对 AppOpsManager.OP_RUN_IN_BACKGROUND op 操作的监听！
```java
    mAppOpsService.startWatchingMode(AppOpsManager.OP_RUN_IN_BACKGROUND, null,
            new IAppOpsCallback.Stub() {
                @Override public void opChanged(int op, int uid, String packageName) {
                    if (op == AppOpsManager.OP_RUN_IN_BACKGROUND && packageName != null) {
                        if (mAppOpsService.checkOperation(op, uid, packageName)
                                != AppOpsManager.MODE_ALLOWED) {
                            runInBackgroundDisabled(uid);
                        }
                    }
                }
            });
```
可以看到，这里传入了一个回调：
```java
    new IAppOpsCallback.Stub() {
        @Override public void opChanged(int op, int uid, String packageName) {
            //【1】当 op 为 AppOpsManager.OP_RUN_IN_BACKGROUND 时，并且 packageName 不为 null
            // 这时候会检查 packageName 是有允许在后台运行！
            if (op == AppOpsManager.OP_RUN_IN_BACKGROUND && packageName != null) {
                //【1.1】检查是否有权限执行 op 操作，关于这部分内容，我们后面再分析！
                if (mAppOpsService.checkOperation(op, uid, packageName)
                        != AppOpsManager.MODE_ALLOWED) {
                    //【3.2.1】如果不允许，ams 会 stop 掉该进程！
                    runInBackgroundDisabled(uid);
                }
            }
        }
    }
```
这个 Callback 是一个 IAppOpsCallback.Stub 对象，用于跨进程通信，为什么是个桩对象呢？我们后面会知道！

op 参数表示的是要监听的 op 操作；packageName 表示要监听的具体的 package，这里传入的是 null；callback 表示当 op 操作发生改变时，要执行的回调！

```java
    @Override
    public void startWatchingMode(int op, String packageName, IAppOpsCallback callback) {
        if (callback == null) {
            return;
        }
        synchronized (this) {
            op = (op != AppOpsManager.OP_NONE) ? AppOpsManager.opToSwitch(op) : op;
            //【×3.1】将 callback 转为 Binder 对象，封装为一个 Callback 对象，添加到 mModeWatchers 中！
            Callback cb = mModeWatchers.get(callback.asBinder());
            if (cb == null) {
                // 创建了一个 Callback 对象，用于监听远程 Binder 对象的死亡布告对象
                // 这样当 IAppOpsCallback.Stub 死亡后，可以停止监听！
                cb = new Callback(callback);
                // 将 IAppOpsCallback.Stub 和 Callback 的映射保存到 mModeWatchers 中！
                mModeWatchers.put(callback.asBinder(), cb);
            }

            //【3.2】将要监听的 op 操作和对应的监听回调 Callback 的映射关系保存到 mOpModeWatchers 中！
            if (op != AppOpsManager.OP_NONE) {
                ArrayList<Callback> cbs = mOpModeWatchers.get(op);
                if (cbs == null) {
                    cbs = new ArrayList<Callback>();
                    mOpModeWatchers.put(op, cbs);
                }
                cbs.add(cb);
            }
            //【3.3】将要监听的 package 和对应的监听回调 Callback 的映射关系保存到 mPackageModeWatchers 中！
            if (packageName != null) {
                ArrayList<Callback> cbs = mPackageModeWatchers.get(packageName);
                if (cbs == null) {
                    cbs = new ArrayList<Callback>();
                    mPackageModeWatchers.put(packageName, cbs);
                }
                cbs.add(cb);
            }
        }
    }
```
AppOpsService 内部有多个集合：
```java
    //【1】用与 IBinder 作和对应的回调 Callback 的映射关系！
    final ArrayMap<IBinder, Callback> mModeWatchers
            = new ArrayMap<IBinder, Callback>();
```
用于保存远程回调 IAppOpsCallback.Stub 和其对应的死亡仆告对象 Callback 的映射关系，当 IAppOpsCallback.Stub 死亡后，Callback.binderDied 会被触发，然

```java
    //【1】用与保存 op 操作和对应的回调 Callback 的映射关系！
    final SparseArray<ArrayList<Callback>> mOpModeWatchers
            = new SparseArray<ArrayList<Callback>>();
    //【2】用与保存 package 和对应的回调 Callback 的映射关系！
    final ArrayMap<String, ArrayList<Callback>> mPackageModeWatchers
            = new ArrayMap<String, ArrayList<Callback>>();
```
而 mOpModeWatchers 和 mPackageModeWatchers 则是从不同的角度来建立监听关系：mOpModeWatchers 是从具体 op 的角度，而 mPackageModeWatchers 则是从 package 的角度！

可以看到 Callback 实例的作用是，作为远程回调的死亡仆告对象，用于停止监听！！

## 3.1 new Callback - 死亡回调

我们来看下 Callback 对象的创建，同时 Callback 又是一个 DeathRecipient 对象，用来监听远程 IAppOpsCallback 桩对象是否死亡！
```java
    public final class Callback implements DeathRecipient {
        final IAppOpsCallback mCallback;

        public Callback(IAppOpsCallback callback) {
            mCallback = callback;
            try {
                //【1】将自身注册为一个死亡仆告对象！
                mCallback.asBinder().linkToDeath(this, 0);
            } catch (RemoteException e) {
            }
        }

        public void unlinkToDeath() { //【2】解除注册！
            mCallback.asBinder().unlinkToDeath(this, 0);
        }

        @Override
        public void binderDied() { //【3.1.1】当远程桩对象死亡后，停止监听！
            stopWatchingMode(mCallback);
        }
    }
```
当远程的 IAppOpsCallback.Stub 死亡后，AppOpsService.stopWatchingMode 会被执行！

### 3.1.1 AppOpsService.stopWatchingMode

```java
    @Override
    public void stopWatchingMode(IAppOpsCallback callback) {
        if (callback == null) {
            return;
        }
        synchronized (this) {
            //【1】从 mModeWatchers 中移除 Callback！
            Callback cb = mModeWatchers.remove(callback.asBinder());
            if (cb != null) {
                // 解除死亡仆告对象注册！
                cb.unlinkToDeath();
                //【2】从 mOpModeWatchers 移除该 Callback！
                for (int i=mOpModeWatchers.size()-1; i>=0; i--) {
                    ArrayList<Callback> cbs = mOpModeWatchers.valueAt(i);
                    cbs.remove(cb);
                    if (cbs.size() <= 0) {
                        mOpModeWatchers.removeAt(i);
                    }
                }
                //【3】从 mPackageModeWatchers 移除该 Callback！
                for (int i=mPackageModeWatchers.size()-1; i>=0; i--) {
                    ArrayList<Callback> cbs = mPackageModeWatchers.valueAt(i);
                    cbs.remove(cb);
                    if (cbs.size() <= 0) {
                        mPackageModeWatchers.removeAt(i);
                    }
                }
            }
        }
    }
```
流程很简单，不多说了！

## 3.2 AMS 对 OP_RUN_IN_BACKGROUND 的处理

再次回到 AMS 的构造器中去看看：

```java
    //【1】当 op 为 AppOpsManager.OP_RUN_IN_BACKGROUND 时，并且 packageName 不为 null
    // 这时候会检查 packageName 是有允许在后台运行！
    if (op == AppOpsManager.OP_RUN_IN_BACKGROUND && packageName != null) {
        //【1.1】检查是否有权限执行 op 操作，关于这部分内容，我们后面再分析！
        if (mAppOpsService.checkOperation(op, uid, packageName)
                != AppOpsManager.MODE_ALLOWED) {
            //【3.2.1】如果不允许，ams 会 stop 掉该进程！
            runInBackgroundDisabled(uid);
        }
    }
```
AMS 中并没有监听某个具体的 package 的 op 变化，所以 packageName 传入的是 null；AMS 关心的是 AppOpsManager.OP_RUN_IN_BACKGROUND 操作的变化！

当有 op 发生变化后，opChanged 方法会被调用，这时 AMS 会判断如果 op 为 AppOpsManager.OP_RUN_IN_BACKGROUND，同时 packageName 不为 null；

这时，AMS 会调用 mAppOpsService.checkOperation 判断该应用是否允许后台运行，如果不允许，那么，会调用 runInBackgroundDisabled 方法 stop 掉进程！

### 3.2.1 runInBackgroundDisabled

```java
    final void runInBackgroundDisabled(int uid) {
        synchronized (this) {
            UidRecord uidRec = mActiveUids.get(uid);
            //【1】如果该 uid 仍然在运行，那么只有其处于 idle 状态才能 stop 进程！
            if (uidRec != null) {
                if (uidRec.idle) {
                    //【3.2.2】stop 进程！
                    doStopUidLocked(uidRec.uid, uidRec);
                }
            } else {
                //【2】如果 uid 不再运行，那就 stop 进程！
                doStopUidLocked(uid, null);
            }
        }
    }
```
### 3.2.2 doStopUidLocked
```java
    final void doStopUidLocked(int uid, final UidRecord uidRec) {
        mServices.stopInBackgroundLocked(uid); // stop 掉该进程！
        enqueueUidChangeLocked(uidRec, uid, UidRecord.CHANGE_IDLE);
    }
```
这里就不多说了！


# 4 AppOpsService.publish - publish 到 ServiceManager 中！

我们回到 ActivityManagerService 的 start 方法中去看看：
```java
    private void start() {
        Process.removeAllProcessGroups();
        mProcessCpuThread.start();

        mBatteryStatsService.publish(mContext);
        //【1】调用了 AppOpsService 方法！
        mAppOpsService.publish(mContext);
        Slog.d("AppOps", "AppOpsService published");
        LocalServices.addService(ActivityManagerInternal.class, new LocalService());
    }
```
我们来看看 AppOpsService.publish！
```java
    public void publish(Context context) {
        mContext = context;
        ServiceManager.addService(Context.APP_OPS_SERVICE, asBinder());
    }
```
将自身注册进入 ServiceManager 中！

# 5 AppOpsService.systemReady

我们回到 ActivityManagerService 的 systemReady 方法中去看看：

```java

    public void systemReady(final Runnable goingCallback) {
        synchronized(this) {
            if (mSystemReady) {
                // If we're done calling all the receivers, run the next "boot phase" passed in
                // by the SystemServer
                if (goingCallback != null) {
                    goingCallback.run();
                }
                return;
            }

            mLocalDeviceIdleController
                    = LocalServices.getService(DeviceIdleController.LocalService.class);

            // Make sure we have the current profile info, since it is needed for security checks.
            mUserController.onSystemReady();
            mRecentTasks.onSystemReadyLocked();
            //【1】调用了 mAppOpsService 的 systemReady 方法！
            mAppOpsService.systemReady();
            mSystemReady = true;
        }
        ... ... ... ...
    }

```
去看看 mAppOpsService 的 systemReady 方法：
```java
    public void systemReady() {
        synchronized (this) {
            boolean changed = false;
            for (int i = mUidStates.size() - 1; i >= 0; i--) {
                UidState uidState = mUidStates.valueAt(i);
                //【×5.1】获取属于该 uid 的所有 PacakgeName！
                String[] packageNames = getPackagesForUid(uidState.uid);
                if (ArrayUtils.isEmpty(packageNames)) {
                    //【1】如果属于该 uid 的 package 为 null，从 mUidStates 中移除！
                    uidState.clear();
                    mUidStates.removeAt(i);
                    changed = true;
                    continue;
                }
                // 如果 uidState.pkgOps 为 null，跳过！
                ArrayMap<String, Ops> pkgs = uidState.pkgOps;
                if (pkgs == null) {
                    continue;
                }

                //【2】我们知道 uidState.pkgOps 保存了该 uid 下所有的 package 的 op 信息！
                Iterator<Ops> it = pkgs.values().iterator();
                while (it.hasNext()) { // 遍历每一个 package 的 Op 信息！
                    Ops ops = it.next();
                    int curUid = -1;
                    try {
                        //【2.1】首先，通过包名获得 package 的 uid！
                        curUid = AppGlobals.getPackageManager().getPackageUid(ops.packageName,
                                PackageManager.MATCH_UNINSTALLED_PACKAGES,
                                UserHandle.getUserId(ops.uidState.uid));
                    } catch (RemoteException ignored) {
                    }
                    //【2.2】然后和 op 所属的 uid 进行比较，如果不相等，说明 package 信息发生了变化！
                    // 移除旧的 Ops 集合！
                    if (curUid != ops.uidState.uid) {
                        Slog.i(TAG, "Pruning old package " + ops.packageName
                                + "/" + ops.uidState + ": new uid=" + curUid);
                        it.remove();
                        changed = true;
                    }
                }

                if (uidState.isDefault()) { 
                    //【3】移除那些没有对应 Package 和 Ops 的 UidState！
                    mUidStates.removeAt(i);
                }
            }
            if (changed) { 
                //【5.2】如果 changed 为 true，表示 UidState，或者 Ops 发生了变化，更新本地持久化文件！
                scheduleFastWriteLocked();
            }
        }

        MountServiceInternal mountServiceInternal = LocalServices.getService(
                MountServiceInternal.class);
        mountServiceInternal.addExternalStoragePolicy(
                new MountServiceInternal.ExternalStorageMountPolicy() {
                    @Override
                    public int getMountMode(int uid, String packageName) {
                        if (Process.isIsolated(uid)) {
                            return Zygote.MOUNT_EXTERNAL_NONE;
                        }
                        if (noteOperation(AppOpsManager.OP_READ_EXTERNAL_STORAGE, uid,
                                packageName) != AppOpsManager.MODE_ALLOWED) {
                            return Zygote.MOUNT_EXTERNAL_NONE;
                        }
                        if (noteOperation(AppOpsManager.OP_WRITE_EXTERNAL_STORAGE, uid,
                                packageName) != AppOpsManager.MODE_ALLOWED) {
                            return Zygote.MOUNT_EXTERNAL_READ;
                        }
                        return Zygote.MOUNT_EXTERNAL_WRITE;
                    }

                    @Override
                    public boolean hasExternalStorage(int uid, String packageName) {
                        final int mountMode = getMountMode(uid, packageName);
                        return mountMode == Zygote.MOUNT_EXTERNAL_READ
                                || mountMode == Zygote.MOUNT_EXTERNAL_WRITE;
                    }
                });
    }
```

## 5.1 getPackagesForUid

获取 uid 所有的 package：

```java
    private static String[] getPackagesForUid(int uid) {
        String[] packageNames = null;
        try {
            packageNames = AppGlobals.getPackageManager().getPackagesForUid(uid);
        } catch (RemoteException e) {
            /* ignore - local call */
        }
        if (packageNames == null) {
            return EmptyArray.STRING;
        }
        return packageNames;
    }
```
以数组形式放回！

## 5.2 scheduleFastWriteLocked

更新本地的持久化文件！
```java
    private void scheduleFastWriteLocked() {
        if (!mFastWriteScheduled) {
            mWriteScheduled = true;
            mFastWriteScheduled = true;
            mHandler.removeCallbacks(mWriteRunner);
            mHandler.postDelayed(mWriteRunner, 10*1000); // 
        }
    }
```
这里的 mHandler 是由 AMS 创建的一个子线程的 Handler！

```java
    final Runnable mWriteRunner = new Runnable() {
        public void run() {
            synchronized (AppOpsService.this) {
                mWriteScheduled = false;
                mFastWriteScheduled = false;
                //【1】创建一个 AsyncTask 异步任务对象！
                AsyncTask<Void, Void, Void> task = new AsyncTask<Void, Void, Void>() {
                    @Override protected Void doInBackground(Void... params) {
                        //【5.3】writeState 更新持久化文件！
                        writeState();
                        return null;
                    }
                };
                //【2】执行任务！
                task.executeOnExecutor(AsyncTask.THREAD_POOL_EXECUTOR, (Void[])null);
            }
        }
    };
```
继续来看！！

## 5.3 writeState

我们来看下，如何更新持久化文件！
```java
    void writeState() {
        synchronized (mFile) {
            //【5.2.1】获得所有 package 的 Op 信息！
            List<AppOpsManager.PackageOps> allOps = getPackagesForOps(null);

            FileOutputStream stream;
            try {
                stream = mFile.startWrite();
            } catch (IOException e) {
                Slog.w(TAG, "Failed to write state: " + e);
                return;
            }

            try {
                XmlSerializer out = new FastXmlSerializer();
                out.setOutput(stream, StandardCharsets.UTF_8.name());
                out.startDocument(null, true);
                out.startTag(null, "app-ops"); //【1】写入 app-ops 标签！

                final int uidStateCount = mUidStates.size();
                for (int i = 0; i < uidStateCount; i++) { //【2】首先写入 uid 标签！
                    UidState uidState = mUidStates.valueAt(i);
                    if (uidState.opModes != null && uidState.opModes.size() > 0) {
                        out.startTag(null, "uid");
                        out.attribute(null, "n", Integer.toString(uidState.uid)); //【2.1】写入 n 属性！
                        SparseIntArray uidOpModes = uidState.opModes;
                        final int opCount = uidOpModes.size();
                        for (int j = 0; j < opCount; j++) {
                            final int op = uidOpModes.keyAt(j);
                            final int mode = uidOpModes.valueAt(j);
                            out.startTag(null, "op"); //【2.2】写入 op 子标签，和对应的 n，m 属性！
                            out.attribute(null, "n", Integer.toString(op));
                            out.attribute(null, "m", Integer.toString(mode));
                            out.endTag(null, "op");
                        }
                        out.endTag(null, "uid");
                    }
                }

                if (allOps != null) { // 如果 allOps 不为 null，那就开始写入 pkg 标签！
                    String lastPkg = null;
                    for (int i=0; i<allOps.size(); i++) {
                        AppOpsManager.PackageOps pkg = allOps.get(i);
                        if (!pkg.getPackageName().equals(lastPkg)) {
                            if (lastPkg != null) {
                                out.endTag(null, "pkg");
                            }
                            lastPkg = pkg.getPackageName();
                            out.startTag(null, "pkg"); // 写入 pkg 标签
                            out.attribute(null, "n", lastPkg); // 写入 n 属性，包名
                        }
                        out.startTag(null, "uid"); // 写入 uid 子标签
                        out.attribute(null, "n", Integer.toString(pkg.getUid())); // 写入 n 属性，uid 的值
                        synchronized (this) {
                            Ops ops = getOpsRawLocked(pkg.getUid(), pkg.getPackageName(), false);
                            // 写入 p 属性，表示是否是 Privileged！
                            if (ops != null) {
                                out.attribute(null, "p", Boolean.toString(ops.isPrivileged));
                            } else {
                                out.attribute(null, "p", Boolean.toString(false));
                            }
                        }
                        List<AppOpsManager.OpEntry> ops = pkg.getOps(); // 处理该 package 的 Ops！
                        for (int j=0; j<ops.size(); j++) {
                            AppOpsManager.OpEntry op = ops.get(j);
                            out.startTag(null, "op"); // 写入 op 标签；
                            out.attribute(null, "n", Integer.toString(op.getOp())); // 写入 n 属性，op 码；
                            // 写入 m 属性，表示控制状态，这里先调用了 AppOpsManager.opToDefaultMode 来计算下
                            // 默认的控制状态。如果 op 的控制状态不等于默认值才会写入！
                            if (op.getMode() != AppOpsManager.opToDefaultMode(op.getOp())) {
                                out.attribute(null, "m", Integer.toString(op.getMode()));
                            }
                            // 写入 t 属性！
                            long time = op.getTime();
                            if (time != 0) {
                                out.attribute(null, "t", Long.toString(time));
                            }
                            // 写入 r 属性！                            
                            time = op.getRejectTime();
                            if (time != 0) {
                                out.attribute(null, "r", Long.toString(time));
                            }
                            // 写入 d 属性！                               
                            int dur = op.getDuration();
                            if (dur != 0) {
                                out.attribute(null, "d", Integer.toString(dur));
                            }
                            // 写入 pu 属性！                               
                            int proxyUid = op.getProxyUid();
                            if (proxyUid != -1) {
                                out.attribute(null, "pu", Integer.toString(proxyUid));
                            }
                            // 写入 pp 属性！   
                            String proxyPackageName = op.getProxyPackageName();
                            if (proxyPackageName != null) {
                                out.attribute(null, "pp", proxyPackageName);
                            }
                            out.endTag(null, "op"); 
                        }
                        out.endTag(null, "uid");
                    }
                    if (lastPkg != null) {
                        out.endTag(null, "pkg");
                    }
                }

                out.endTag(null, "app-ops");
                out.endDocument();
                mFile.finishWrite(stream);
            } catch (IOException e) {
                Slog.w(TAG, "Failed to write state, restoring backup.", e);
                mFile.failWrite(stream);
            }
        }
    }
```
writeState 的流程还是很简单的！

### 5.3.1 getPackagesForOps

该方法用于获得所有 package 的 Op 信息，参数 ops 表示只收集其指定的 op 信息！

```java
    @Override
    public List<AppOpsManager.PackageOps> getPackagesForOps(int[] ops) {
        //【1】首先，强制检查调用者是否有 GET_APP_OPS_STATS 权限！
        mContext.enforcePermission(android.Manifest.permission.GET_APP_OPS_STATS,
                Binder.getCallingPid(), Binder.getCallingUid(), null);
        ArrayList<AppOpsManager.PackageOps> res = null;
        synchronized (this) {
            final int uidStateCount = mUidStates.size();
            for (int i = 0; i < uidStateCount; i++) {
                UidState uidState = mUidStates.valueAt(i);
                if (uidState.pkgOps == null || uidState.pkgOps.isEmpty()) {
                    continue;
                }
                //【1.1】首先，获得 uidState.pkgOps 中保存的该 uid 下，所有的 package 的 Ops！
                ArrayMap<String, Ops> packages = uidState.pkgOps;
                final int packageCount = packages.size();
                for (int j = 0; j < packageCount; j++) {
                    //【1.2】处理每一个 package 的 Ops！
                    Ops pkgOps = packages.valueAt(j); 
                    //【5.3.1.1】调用 collectOps 方法,收集 package 的 Op 信息，封装为 OpEntry 对象！！
                    ArrayList<AppOpsManager.OpEntry> resOps = collectOps(pkgOps, ops);
                    if (resOps != null) {
                        if (res == null) {
                            res = new ArrayList<AppOpsManager.PackageOps>();
                        }
                        //【5.3.1.2】创建一个 PackageOps 对象，封装该 package 的所有 Op 信息
                        // 最后添加到 res 列表中！
                        AppOpsManager.PackageOps resPackage = new AppOpsManager.PackageOps(
                                pkgOps.packageName, pkgOps.uidState.uid, resOps);
                        res.add(resPackage);
                    }
                }
            }
        }
        // 返回列表！
        return res;
    }
```
#### 5.3.1.1 collectOps

参数 int[] ops 用于指定特定的 Op，这样我们会只收集这些特定的 Ops，当然这里我们要收集所有 package 的 Op 信息，所以 int[] ops 传入 null；

```java
    private ArrayList<AppOpsManager.OpEntry> collectOps(Ops pkgOps, int[] ops) {
        ArrayList<AppOpsManager.OpEntry> resOps = null;
        if (ops == null) { //【1】ops 为 null，我们收集该 package 的所有 Ops！
            resOps = new ArrayList<AppOpsManager.OpEntry>();
            for (int j=0; j<pkgOps.size(); j++) {
                Op curOp = pkgOps.valueAt(j); // 处理每一个 Op 对象！
                //【5.3.1.1.1】创建一个  AppOpsManager.OpEntry 对象，并添加到 resOps 中！
                resOps.add(new AppOpsManager.OpEntry(curOp.op, curOp.mode, curOp.time,
                        curOp.rejectTime, curOp.duration, curOp.proxyUid,
                        curOp.proxyPackageName));
            }
        } else {
            for (int j=0; j<ops.length; j++) {
                Op curOp = pkgOps.get(ops[j]);
                if (curOp != null) { //【2】ops 不为 null，我们收集数组 ops 指定的 Op！
                    if (resOps == null) {
                        resOps = new ArrayList<AppOpsManager.OpEntry>();
                    }
                    resOps.add(new AppOpsManager.OpEntry(curOp.op, curOp.mode, curOp.time,
                            curOp.rejectTime, curOp.duration, curOp.proxyUid,
                            curOp.proxyPackageName));
                }
            }
        }
        //【3】返回 resOps
        return resOps;
    }
```
collectOps 方法很简单，不多说了！

##### 5.3.1.1.1 new AppOpsManager.OpEntry


```java
public static class OpEntry implements Parcelable {
    private final int mOp;
    private final int mMode;
    private final long mTime;
    private final long mRejectTime;
    private final int mDuration;
    private final int mProxyUid;
    private final String mProxyPackageName;

    public OpEntry(int op, int mode, long time, long rejectTime, int duration,
            int proxyUid, String proxyPackage) {
        mOp = op;
        mMode = mode;
        mTime = time;
        mRejectTime = rejectTime;
        mDuration = duration;
        mProxyUid = proxyUid;
        mProxyPackageName = proxyPackage;
    }
    
    ... ... ... ...// 省略 get 和 set，以及 Parcelable 相关方法！
}
```
OpEntry 的结构很简单，不多说了！

#### 5.3.1.2 new AppOpsManager.PackageOps

```java
    public static class PackageOps implements Parcelable {
        private final String mPackageName; // package 
        private final int mUid; // uid 
        private final List<OpEntry> mEntries; // 持有的所有 Op 对应的 OpEntry 对象！

        public PackageOps(String packageName, int uid, List<OpEntry> entries) {
            mPackageName = packageName;
            mUid = uid;
            mEntries = entries;
        }
        ... ... ... ...// 省略 get 和 set，以及 Parcelable 相关方法！
    }
```
PackageOps 的结构很简单，不多说了！


### 5.3.2 getOpsRawLocked

getOpsRawLocked 方法用于获得 package 的 Ops 列表！
```java
    private Ops getOpsRawLocked(int uid, String packageName, boolean edit) {
        //【1】获得 uid 对应的 UidState！
        UidState uidState = getUidStateLocked(uid, edit);
        if (uidState == null) {
            return null;
        }

        if (uidState.pkgOps == null) {
            if (!edit) {
                return null;
            }
            uidState.pkgOps = new ArrayMap<>();
        }
        //【2】获得 package 对应的 Ops 集合！
        Ops ops = uidState.pkgOps.get(packageName);
        if (ops == null) { // 处理 package 的 ops 为 null 的特殊情况！
            if (!edit) {
                return null;
            }
            boolean isPrivileged = false;
            //【3】如果 uid 不为 null，需要校验 uid 是否一致！
            if (uid != 0) {
                final long ident = Binder.clearCallingIdentity();
                try {
                    int pkgUid = -1;
                    try {
                        //【3.1】获得该 package 的 ApplicationInfo 对象！
                        ApplicationInfo appInfo = ActivityThread.getPackageManager()
                                .getApplicationInfo(packageName,
                                        PackageManager.MATCH_DEBUG_TRIAGED_MISSING,
                                        UserHandle.getUserId(uid));
                        //【3.2】获得 package 的 uid，并判断是否是 FLAG_PRIVILEGED 的！
                        if (appInfo != null) {
                            pkgUid = appInfo.uid;
                            isPrivileged = (appInfo.privateFlags
                                    & ApplicationInfo.PRIVATE_FLAG_PRIVILEGED) != 0;
                        } else {
                            // 如果 appInfo 为 null，那就判断一些特殊 uid 的情况！
                            if ("media".equals(packageName)) {
                                pkgUid = Process.MEDIA_UID;
                                isPrivileged = false;
                            } else if ("audioserver".equals(packageName)) {
                                pkgUid = Process.AUDIOSERVER_UID;
                                isPrivileged = false;
                            } else if ("cameraserver".equals(packageName)) {
                                pkgUid = Process.CAMERASERVER_UID;
                                isPrivileged = false;
                            }
                        }
                    } catch (RemoteException e) {
                        Slog.w(TAG, "Could not contact PackageManager", e);
                    }
                    //【3.3】检查 uid 是否发生变化，如果有变化，无效，那就返回 null！
                    if (pkgUid != uid) {
                        RuntimeException ex = new RuntimeException("here");
                        ex.fillInStackTrace();
                        Slog.w(TAG, "Bad call: specified package " + packageName
                                + " under uid " + uid + " but it is really " + pkgUid, ex);
                        return null;
                    }
                } finally {
                    Binder.restoreCallingIdentity(ident);
                }
            }
            //【4】如果该 package 的 Ops 为 null，那就会
            // 给该 package 重新创建一个 Ops，并添加到所属的 uidState.pkgOps 中！
            ops = new Ops(packageName, uidState, isPrivileged);
            uidState.pkgOps.put(packageName, ops);
        }
        return ops; // 返回该 package 的 Ops 列表！
    }
```
方法很简单，不多说了！