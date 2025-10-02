# BroadcastReceiver篇 4 - BroadcastReceiver 静态注册
title: BroadcastReceiver篇 4 - BroadcastReceiver 静态注册
date: 2016/05/08 20:46:25
catalog: true
categories:
- AndroidFramework源码分析
- BroadcastReceiver广播接收者
tags: BroadcastReceiver广播接收者
---

[toc]

本文基于 Android 7.1.1 源码，分析 BroadcastReceiver 的静态注册过程，转载请说明出处，谢谢！

# 0 综述

广播接收者除了动态注册之外，还有静态注册，就是在 AndroidManifest.xml 文件中进行配置！

```java
<receiver 
    android:enabled=["true" | "false"]
    android:exported=["true" | "false"]
    android:icon="drawable resource"
    android:label="string resource"
    android:name="string"
    android:permission="string"
    android:process="string" >
    <intent-filter>
        <action android:name="android.intent.action.BOOT_COMPLETED" />
    </intent-filter>
</receiver>
```

我们来简单地看看这些属性的意思： 

- android:enabled：此 broadcastReceiver 是否可用，默认值为 true。
- android:exported：此 broadcastReceiver 能否接收其它 App 的发出的广播，如果标签中定义了 intent-filter 字段，则此值默认值为 true，否则为 false。
- android:name：此 BroadcastReceiver 的组件名。
- android:permission：广播发送方应该具有的权限； 
- android:process：表示 broadcastReceiver 运行的进程，BroadcastReceiver 默认运行在当前 app 的进程中，也可以通过此字段指定其运行于其它独立的进程。
- intent-filter：用于指定该broadcastReceiver接收广播的类型。


静态注册的 BroadcastReceiver 不能作为内部类，必须要单独作为一个 BroadcastReceiver.java 文件！

其实，如果大家对于 PMS 熟悉的话，应该很清楚了，对于静态注册的方式，是会通过 PMS 的解析来获得 BroadcastReceiver 的信息的！！下面我们就到 PMS 中去看看：

# 1 PackageParser.parseBaseApplication - 解析 application

解析的关键的代码段如下，我们简单的回顾下：
```java
    private boolean parseBaseApplication(Package owner, Resources res,
            XmlResourceParser parser, int flags, String[] outError)
        throws XmlPullParserException, IOException {
        ... ... ... ...
        while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
                && (type != XmlPullParser.END_TAG || parser.getDepth() > innerDepth)) {
            if (type == XmlPullParser.END_TAG || type == XmlPullParser.TEXT) {
                continue;
            }

            String tagName = parser.getName();
            if (tagName.equals("activity")) {
                Activity a = parseActivity(owner, res, parser, flags, outError, false,
                        owner.baseHardwareAccelerated);
                if (a == null) {
                    mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
                    return false;
                }

                owner.activities.add(a);
            
            //【2.1】解析 "receiver"，获得静态注册的广播接收者的信息！
            } else if (tagName.equals("receiver")) {
                Activity a = parseActivity(owner, res, parser, flags, outError, true, false);
                if (a == null) {
                    mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
                    return false;
                }

                //【1】将解析到的 receiver 加入到 Package owner 中！
                owner.receivers.add(a);

            } 
            ... ... ...
        }
        ... ... ... ...
    }
```
可以看到，这里会解析静态注册的广播接收者，并将其信息封装成 Activity 对象保存到了 Package.receivers 对象中去！

我们进入到 PackageParser 的 parseActivity 中去看看！

## 2.1 PackageParser.parseActivity

```java
    private Activity parseActivity(Package owner, Resources res,
            XmlResourceParser parser, int flags, String[] outError,
            boolean receiver, boolean hardwareAccelerated)
            throws XmlPullParserException, IOException {

        TypedArray sa = res.obtainAttributes(parser, R.styleable.AndroidManifestActivity);

        if (mParseActivityArgs == null) {
            mParseActivityArgs = new ParseComponentArgs(owner, outError,
                    R.styleable.AndroidManifestActivity_name,
                    R.styleable.AndroidManifestActivity_label,
                    R.styleable.AndroidManifestActivity_icon,
                    R.styleable.AndroidManifestActivity_roundIcon,
                    R.styleable.AndroidManifestActivity_logo,
                    R.styleable.AndroidManifestActivity_banner,
                    mSeparateProcesses,
                    R.styleable.AndroidManifestActivity_process,
                    R.styleable.AndroidManifestActivity_description,
                    R.styleable.AndroidManifestActivity_enabled);
        }

        mParseActivityArgs.tag = receiver ? "<receiver>" : "<activity>";
        mParseActivityArgs.sa = sa;
        mParseActivityArgs.flags = flags;
        
        //【1】创建了一个 Activity 对象！
        Activity a = new Activity(mParseActivityArgs, new ActivityInfo());
        
        if (outError[0] != null) {
            sa.recycle();
            return null;
        }

        ... ... ... ...

        int outerDepth = parser.getDepth();
        int type;
        while ((type=parser.next()) != XmlPullParser.END_DOCUMENT
               && (type != XmlPullParser.END_TAG
                       || parser.getDepth() > outerDepth)) {
            if (type == XmlPullParser.END_TAG || type == XmlPullParser.TEXT) {
                continue;
            }
            
            
            // 解析 "intent-filter" 标签！
            if (parser.getName().equals("intent-filter")) {

                //【2】创建 ActivityIntentInfo 对象！
                ActivityIntentInfo intent = new ActivityIntentInfo(a);
                
                //【2.2】解析 "intent-filter" 的子标签！
                if (!parseIntent(res, parser, true, true, intent, outError)) {
                    return null;
                }
                if (intent.countActions() == 0) {
                    Slog.w(TAG, "No actions in intent filter at "
                            + mArchiveSourcePath + " "
                            + parser.getPositionDescription());
                } else {

                    //【4】如果设置了 intent 过滤属性，即 intent.countActions() != 0：
                    a.intents.add(intent);
                }
            } else if (!receiver && parser.getName().equals("preferred")) { // 这个是和 activity 相关的
                ActivityIntentInfo intent = new ActivityIntentInfo(a);
                if (!parseIntent(res, parser, false, false, intent, outError)) {
                    return null;
                }
                if (intent.countActions() == 0) {
                    Slog.w(TAG, "No actions in preferred at "
                            + mArchiveSourcePath + " "
                            + parser.getPositionDescription());
                } else {
                    if (owner.preferredActivityFilters == null) {
                        owner.preferredActivityFilters = new ArrayList<ActivityIntentInfo>();
                    }
                    owner.preferredActivityFilters.add(intent);
                }
            } else if (parser.getName().equals("meta-data")) {
                if ((a.metaData = parseMetaData(res, parser, a.metaData,
                        outError)) == null) {
                    return null;
                }
            } else if (!receiver && parser.getName().equals("layout")) {
                parseLayout(res, parser, a);
            } else {
                ... ... ... ...
            }
        }

        if (!setExported) {
            a.info.exported = a.intents.size() > 0;
        }

        return a;
    }
```

## 2.2 PackageParser.parseIntent

我们来看看 parseIntent 方法中做过了些什么！

```java
    private boolean parseIntent(Resources res, XmlResourceParser parser,
            boolean allowGlobs, boolean allowAutoVerify, IntentInfo outInfo, String[] outError)
            throws XmlPullParserException, IOException {
        
        //【1】这里是解析 "intent-filter" 标签的属性，我们这里不重点关注！
        TypedArray sa = res.obtainAttributes(parser,
                com.android.internal.R.styleable.AndroidManifestIntentFilter);

        ... ... ... ...

        sa.recycle();

        int outerDepth = parser.getDepth();
        int type;
        while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
                && (type != XmlPullParser.END_TAG || parser.getDepth() > outerDepth)) {
            if (type == XmlPullParser.END_TAG || type == XmlPullParser.TEXT) {
                continue;
            }
           
            String nodeName = parser.getName();

            //【2】解析 "action"
            if (nodeName.equals("action")) {
                String value = parser.getAttributeValue(
                        ANDROID_RESOURCES, "name");
                if (value == null || value == "") {
                    outError[0] = "No value supplied for <android:name>";
                    return false;
                }
                XmlUtils.skipCurrentTag(parser);
                //【1】设置 action 属性；
                outInfo.addAction(value);

            //【3】解析 "category"
            } else if (nodeName.equals("category")) {
                String value = parser.getAttributeValue(
                        ANDROID_RESOURCES, "name"); // android:na
                if (value == null || value == "") {
                    outError[0] = "No value supplied for <android:name>";
                    return false;
                }
                XmlUtils.skipCurrentTag(parser);
                //【2】设置 category 属性；
                outInfo.addCategory(value);
            
            //【4】解析 "data"
            } else if (nodeName.equals("data")) {
                sa = res.obtainAttributes(parser,
                        com.android.internal.R.styleable.AndroidManifestData);

                String str = sa.getNonConfigurationString(
                        com.android.internal.R.styleable.AndroidManifestData_mimeType, 0);
                if (str != null) {
                    try {
                        //【4.2】设置 dataType 属性；
                        outInfo.addDataType(str);
                    } catch (IntentFilter.MalformedMimeTypeException e) {
                        outError[0] = e.toString();
                        sa.recycle();
                        return false;
                    }
                }

                ... ... ... ...

                sa.recycle();
                XmlUtils.skipCurrentTag(parser);
            } else if (!RIGID_PARSER) {
                Slog.w(TAG, "Unknown element under <intent-filter>: "
                        + parser.getName() + " at " + mArchiveSourcePath + " "
                        + parser.getPositionDescription());
                XmlUtils.skipCurrentTag(parser);
            } else {
                outError[0] = "Bad element under <intent-filter>: " + parser.getName();
                return false;
            }
        }

        outInfo.hasDefault = outInfo.hasCategory(Intent.CATEGORY_DEFAULT);

        if (DEBUG_PARSER) {

            ... ... ...

        }

        return true;
    }

```
其实看到这里，我们已经能够看出静态注册的 broadcastReceiver 是如何被 PMS 解析了，解析的数据都会被保存到 PackageParser.Package 中！

# 2 PMS.scanPackageDirtyLI

接下来，PMS 会进一步处理解析的数据！
```java
    private PackageParser.Package scanPackageDirtyLI(PackageParser.Package pkg,
            final int policyFlags, final int scanFlags, long currentTime, UserHandle user)
            throws PackageManagerException {

            ... ... ...
            
            //【1】将解析到的 BroadcastRecord 数据保存到 PMS 的内部变量 mReceivers 中！
            N = pkg.receivers.size();
            r = null;
            for (i=0; i<N; i++) {

                //【2】获得该 package 中静态注册的广播接收者的信息对象 Activity；
                PackageParser.Activity a = pkg.receivers.get(i);

                a.info.processName = fixProcessName(pkg.applicationInfo.processName,
                        a.info.processName, pkg.applicationInfo.uid);
                
                //【3】添加到 PMS.mReceivers 中;
                mReceivers.addActivity(a, "receiver");
                
                if ((policyFlags&PackageParser.PARSE_CHATTY) != 0) {
                    if (r == null) {
                        r = new StringBuilder(256);
                    } else {
                        r.append(' ');
                    }
                    r.append(a.info.name);
                }
            }

            if (r != null) {
                if (DEBUG_PACKAGE_SCANNING) Log.d(TAG, "  Receivers: " + r);
            }
            
            ... ...

    }
```

将之前是扫描解析到的信息对象 Activity，保存到 PMS.mReceivers 对象中，mReceivers 是一个 ActivityIntentResolver 类的对象，用于保存系统中定义的所有的静态接收者：

```java
    // All available receivers, for your resolving pleasure.
    final ActivityIntentResolver mReceivers =
            new ActivityIntentResolver();
```
我们继续分析！

## 2.1 PMS.ActivityIntentResolver


我们来看下 ActivityIntentResolver 是如何处理静态接收者的注册的！

```java
    final class ActivityIntentResolver
            extends IntentResolver<PackageParser.ActivityIntentInfo, ResolveInfo> {

        ... ... ... ...

        public final void addActivity(PackageParser.Activity a, String type) {
            //【1】key 为广播接收者组件名，value 为 PackageParser.Activity 对象，保存到内部的 mActivities 哈希表中！
            mActivities.put(a.getComponentName(), a);

            if (DEBUG_SHOW_INFO)
                Log.v(
                TAG, "  " + type + " " +
                (a.info.nonLocalizedLabel != null ? a.info.nonLocalizedLabel : a.info.name) + ":");
            if (DEBUG_SHOW_INFO)
                Log.v(TAG, "    Class=" + a.info.name);
            
            //【2】处理对应的 ActivityIntentInfo 对象！
            final int NI = a.intents.size();
            for (int j=0; j<NI; j++) {
                PackageParser.ActivityIntentInfo intent = a.intents.get(j);

                if ("activity".equals(type)) { // 因为我们的 type 是 "receiver" ，所以不进入该分支！
                    final PackageSetting ps =
                            mSettings.getDisabledSystemPkgLPr(intent.activity.info.packageName);
                    final List<PackageParser.Activity> systemActivities =
                            ps != null && ps.pkg != null ? ps.pkg.activities : null;
                    adjustPriority(systemActivities, intent);
                }

                if (DEBUG_SHOW_INFO) {
                    Log.v(TAG, "    IntentFilter:");
                    intent.dump(new LogPrinter(Log.VERBOSE, TAG), "      ");
                }
                if (!intent.debugCheck()) {
                    Log.w(TAG, "==> For Activity " + a.info.name);
                }
                
                //【2.2】调用了父类的 addFilter 方法；
                addFilter(intent);
            }
        }

        ... ... ... ...

        // 内部集合，存储解析到的组件！
        private final ArrayMap<ComponentName, PackageParser.Activity> mActivities
                = new ArrayMap<ComponentName, PackageParser.Activity>();
        private int mFlags;            
    }        

```

ActivityIntentResolver 继承了 IntentResolver，IntentResolver 是一个模板类：

## 2.2 IntentResolver.addFilter

这里 F 是 PackageParser.ActivityIntentInfo， R 是 ResolveInfo：
```java
public abstract class IntentResolver<F extends IntentFilter, R extends Object> {

    //【1】调用了 addFilter 方法！
    public void addFilter(F f) {
        if (localLOGV) {
            Slog.v(TAG, "Adding filter: " + f);
            f.dump(new LogPrinter(Log.VERBOSE, TAG, Log.LOG_ID_SYSTEM), "      ");
            Slog.v(TAG, "    Building Lookup Maps:");
        }
        
        //【2】添加到内部总集合 mFilters 中！
        mFilters.add(f);
        //【3】根据 filter 属性配置，添加到不同的子集合中！
        //【3.1】处理 Scheme；
        int numS = register_intent_filter(f, f.schemesIterator(),
                mSchemeToFilter, "      Scheme: ");
        //【3.2】处理 Type；
        int numT = register_mime_types(f, "      Type: ");
        //【3.3】处理 Action；
        if (numS == 0 && numT == 0) {
            register_intent_filter(f, f.actionsIterator(),
                    mActionToFilter, "      Action: ");
        }
        //【3.4】处理 TypedAction；
        if (numT != 0) {
            register_intent_filter(f, f.actionsIterator(),
                    mTypedActionToFilter, "      TypedAction: ");
        }
    }
    
    private final ArraySet<F> mFilters = new ArraySet<F>(); // 所有注册的 filter
}
```
这里我们就是实现了对静态注册的广播接收者的处理！！

### 2.2.1 IntentResolver.register_intent_filter

register_intent_filter 用于注册 filter ，并返回注册的数量，对于参数，我们以 mSchemeToFilter 为例：

**F filter**：这里是我们解析的 intentFilter 的 ActivityIntentInfo 对象！

**Iterator<String> i**：f.schemesIterator()，是一个 Scheme 迭代器，用于遍历该 filter 设置的所有的 Scheme；

**ArrayMap<String, F[]> dest**：是我们要加入集合，这里是 mSchemeToFilter；

**String prefix**：用于 log，不过多关注！


```java
    private final int register_intent_filter(F filter, Iterator<String> i,
            ArrayMap<String, F[]> dest, String prefix) {
        if (i == null) {
            return 0;
        }

        int num = 0;
        while (i.hasNext()) {
            //【1】这里返回的就是 android:name 属性对应的值！
            String name = i.next();
            num++;
            if (localLOGV) Slog.v(TAG, prefix + name);
            //【2.2.2】继续处理！
            addFilter(dest, name, filter);
        }
        return num;
    }
```
### 2.2.2 IntentResolver.addFilter
```java
    private final void addFilter(ArrayMap<String, F[]> map, String name, F filter) {
        F[] array = map.get(name);
        if (array == null) {
            array = newArray(2);
            map.put(name,  array);
            array[0] = filter; //【1】保存到 array[0] 中！
        } else {
            final int N = array.length;
            int i = N;
            while (i > 0 && array[i-1] == null) {
                i--;
            }
            if (i < N) {
                //【2】插入到末尾！
                array[i] = filter;
            } else {
                //【3】扩充数组为之前的 3/2 倍，插入到末尾！
                F[] newa = newArray((N*3)/2);
                System.arraycopy(array, 0, newa, 0, N);
                newa[N] = filter;
                map.put(name, newa);
            }
        }
    }
```

当我们要发送广播时，是要查找当前系统中所有已经静态注册的广播接收者的，那么我们去看看查询相关的方法：

# 3 PMS.queryIntentReceivers - 查询静态接收者

查询静态注册的广播接收者方法如下：
```java
    @Override
    public @NonNull ParceledListSlice<ResolveInfo> queryIntentReceivers(Intent intent,
            String resolvedType, int flags, int userId) {

        return new ParceledListSlice<>(
                queryIntentReceiversInternal(intent, resolvedType, flags, userId));
    }
```

最终会调用：
```java
    private @NonNull List<ResolveInfo> queryIntentReceiversInternal(Intent intent,
            String resolvedType, int flags, int userId) {

        //【1】如果不存在这个设备用户 id，就返回一个数据为 null 的 list；
        if (!sUserManager.exists(userId)) return Collections.emptyList();

        flags = updateFlagsForResolve(flags, userId, intent);

        ComponentName comp = intent.getComponent();
        if (comp == null) {
            if (intent.getSelector() != null) {
                intent = intent.getSelector();
                comp = intent.getComponent();
            }
        }

        //【2】如果 Intent 设置了组件名，就通过组件名直接获取！
        if (comp != null) {
            List<ResolveInfo> list = new ArrayList<ResolveInfo>(1);
            //【3.1】创建 ResolveInfo 的 list 列表，用于保存匹配结果！
            ActivityInfo ai = getReceiverInfo(comp, flags, userId);
            if (ai != null) {
                ResolveInfo ri = new ResolveInfo();
                ri.activityInfo = ai;
                list.add(ri);
            }
            return list;
        }

        synchronized (mPackages) {

            //【2】如果 Intent 没有设置组件名和包名，就调用 queryIntent 进行查询！
            String pkgName = intent.getPackage();
            if (pkgName == null) {
                //【3.2】使用 queryIntent 方法！
                return mReceivers.queryIntent(intent, resolvedType, flags, userId);
            }

            //【3】如果 Intent 没有设置组件名，但是设置了包名，就通过之前的扫描解析信息，查询！
            final PackageParser.Package pkg = mPackages.get(pkgName);
            if (pkg != null) {
                //【3.3】使用 queryIntentForPackage 根据包名查询，mReceivers 中保存了解析到的所有的；
                return mReceivers.queryIntentForPackage(intent, resolvedType, flags, pkg.receivers,
                        userId);
            }

            return Collections.emptyList();
        }
    }
```

下面我们一个一个去看看：

## 3.1 PackageManagerS.getReceiverInfo - 组件名查询
```java
    @Override
    public ActivityInfo getReceiverInfo(ComponentName component, int flags, int userId) {
        if (!sUserManager.exists(userId)) return null;
        
        flags = updateFlagsForComponent(flags, userId, component);
        enforceCrossUserPermission(Binder.getCallingUid(), userId,
                false /* requireFullPermission */, false /* checkShell */, "get receiver info");
        synchronized (mPackages) {
            //【1】直接通过组件名获得对应的 PackageParser.Activity 对象，封装了 receiver 的信息！
            PackageParser.Activity a = mReceivers.mActivities.get(component);
            if (DEBUG_PACKAGE_INFO) Log.v(
                TAG, "getReceiverInfo " + component + ": " + a);
            if (a != null && mSettings.isEnabledAndMatchLPr(a.info, flags, userId)) {
            
                //【2】如果对应的 PackageSetting 对象为 null，说明应用没有安装，返回 null
                PackageSetting ps = mSettings.mPackages.get(component.getPackageName());
                if (ps == null) return null;
                
                //【3.1.1】调用 PackageParser 的 generateActivityInfo 方法继续查询！
                return PackageParser.generateActivityInfo(a, flags, ps.readUserState(userId),
                        userId);
            }
        }
        return null;
    }
```
继续看：

### 3.1.1 PackageParser.generateActivityInfo

```java
    public static final ActivityInfo generateActivityInfo(Activity a, int flags,
            PackageUserState state, int userId) {
        if (a == null) return null;
        //【1】判断应用是否安装或者隐藏；
        if (!checkUseInstalledOrHidden(flags, state)) {
            return null;
        }
        //【2】判断是否需要 copy 一份数据，如果不需要就直接返回 mReceivers 中解析到的！
        if (!copyNeeded(flags, a.owner, state, a.metaData, userId)) {
            return a.info;
        }

        //【3】获得 Activity 内部的 ActivityInfo 对象的拷贝！
        ActivityInfo ai = new ActivityInfo(a.info);
        ai.metaData = a.metaData;
        //【4】拷贝封装需要的信息
        ai.applicationInfo = generateApplicationInfo(a.owner, flags, state, userId);
        return ai;
    }
```
这里很简单，返回了之前解析获得的广播接收者的 ActivityInfo 对象 ai！

然后，创建 ResolveInfo ri = new ResolveInfo() 对象，然后初始化 ri.activityInfo = ai，将 ResolveInfo 对象 ri 添加到 List<ResolveInfo> 返回！

## 3.2 ActivityIntentResolver.queryIntent - 意图匹配查询

这里先会调用子类 ActivityIntentResolver 的 queryIntent 方法！

```java
        public List<ResolveInfo> queryIntent(Intent intent, String resolvedType, int flags,
                int userId) {
            //【1】userId 不存在，返回 null
            if (!sUserManager.exists(userId)) return null;
            mFlags = flags;
            
            //【3.2.1】调用父类的 queryIntent 方法继续查询！
            return super.queryIntent(intent, resolvedType,
                    (flags & PackageManager.MATCH_DEFAULT_ONLY) != 0, userId);
        }
```
进入父类 IntentResolver：

### 3.2.1 IntentResolver.queryIntent

这里的模板参数 R 为 ResolveInfo 类：
```java
    public List<R> queryIntent(Intent intent, String resolvedType, boolean defaultOnly,
            int userId) {
        String scheme = intent.getScheme();

        //【1】最终要返回的 ArrayList<R> list 集合！
        ArrayList<R> finalList = new ArrayList<R>();

        final boolean debug = localLOGV ||
                ((intent.getFlags() & Intent.FLAG_DEBUG_LOG_RESOLUTION) != 0);

        if (debug) Slog.v(
            TAG, "Resolving type=" + resolvedType + " scheme=" + scheme
            + " defaultOnly=" + defaultOnly + " userId=" + userId + " of " + intent);

        F[] firstTypeCut = null;
        F[] secondTypeCut = null;
        F[] thirdTypeCut = null;
        F[] schemeCut = null;

        //【2】如果 intent 包含一个 MIME type，我们会手机所有能匹配该 MIME 的 filters！
        // MIME 是描述消息内容类型的因特网标准，格式如下：application/pdf
        if (resolvedType != null) {
            int slashpos = resolvedType.indexOf('/');
            //【2.1】校验指定的 MIME 的格式是否正确，必须要有 / 隔开！
            if (slashpos > 0) {
                //【2.1.1】获得 MIME 的前半部分：application，针对其取值做不同的处理！
                final String baseType = resolvedType.substring(0, slashpos);
                if (!baseType.equals("*")) {
                    //【2.1.1.1】如果 baseType 不为 *，说明 intent 可能指定了一个 MIME，接下来，我们继续处理！
                    if (resolvedType.length() != slashpos+2
                            || resolvedType.charAt(slashpos+1) != '*') {
                        // Not a wild card, so we can just look for all filters that
                        // completely match or wildcards whose base type matches.
                        //【2.1.1.1.1】如果 resolvedType
                        firstTypeCut = mTypeToFilter.get(resolvedType);
                        if (debug) Slog.v(TAG, "First type cut: " + Arrays.toString(firstTypeCut));
                        secondTypeCut = mWildTypeToFilter.get(baseType);
                        if (debug) Slog.v(TAG, "Second type cut: "
                                + Arrays.toString(secondTypeCut));

                    } else {
                        //【2.1.1.1.2】如果 resolvedType 的长度等于 slashpos + 2，且 slashpos + 1 为 "*"
                        // 那么其能够匹配所有的 "baseType/*"
                        firstTypeCut = mBaseTypeToFilter.get(baseType);
                        if (debug) Slog.v(TAG, "First type cut: " + Arrays.toString(firstTypeCut));
                        secondTypeCut = mWildTypeToFilter.get(baseType);
                        if (debug) Slog.v(TAG, "Second type cut: "
                                + Arrays.toString(secondTypeCut));

                    }
                    // Any */* types always apply, but we only need to do this
                    // if the intent type was not already */*.
                    thirdTypeCut = mWildTypeToFilter.get("*");
                    if (debug) Slog.v(TAG, "Third type cut: " + Arrays.toString(thirdTypeCut));

                } else if (intent.getAction() != null) {
                    //【2.1.1.2】如果 baseType 为 *，说明是通过正则表达式匹配任何类型的 filters
                    // 这种情况会有风险，那就使用 action 来匹配！
                    firstTypeCut = mTypedActionToFilter.get(intent.getAction());
                    if (debug) Slog.v(TAG, "Typed Action list: " + Arrays.toString(firstTypeCut));

                }
            }
        }

        //【3】如果 intent 设置了 scheme（data URI），收集所有能匹配该 scheme 的 filter！
        if (scheme != null) {
            schemeCut = mSchemeToFilter.get(scheme);
            if (debug) Slog.v(TAG, "Scheme list: " + Arrays.toString(schemeCut));
        }

        //【4】intent 没有设置 Scheme 和 MIME，那就匹配 action！
        if (resolvedType == null && scheme == null && intent.getAction() != null) {
            firstTypeCut = mActionToFilter.get(intent.getAction());
            if (debug) Slog.v(TAG, "Action list: " + Arrays.toString(firstTypeCut));
        }


        //【3.4】开始进一步的匹配，将结果保存到 finalList 中！
        FastImmutableArraySet<String> categories = getFastIntentCategories(intent);
        if (firstTypeCut != null) {
            buildResolveList(intent, categories, debug, defaultOnly,
                    resolvedType, scheme, firstTypeCut, finalList, userId);
        }
        if (secondTypeCut != null) {
            buildResolveList(intent, categories, debug, defaultOnly,
                    resolvedType, scheme, secondTypeCut, finalList, userId);
        }
        if (thirdTypeCut != null) {
            buildResolveList(intent, categories, debug, defaultOnly,
                    resolvedType, scheme, thirdTypeCut, finalList, userId);
        }
        if (schemeCut != null) {
            buildResolveList(intent, categories, debug, defaultOnly,
                    resolvedType, scheme, schemeCut, finalList, userId);
        }
        //【3.5】对结果进行过滤；
        filterResults(finalList);
        //【3.6】对结果进行排序；
        sortResults(finalList);

        if (debug) {
            Slog.v(TAG, "Final result list:");
            for (int i=0; i<finalList.size(); i++) {
                Slog.v(TAG, "  " + finalList.get(i));
            }
        }
        return finalList;
    }

```
最后返回匹配的 intent filter！

## 3.3 ActivityIntentResolver.queryIntentForPackage - 包名查询

参数：

- **ArrayList<PackageParser.Activity> packageActivities**：传入 PackageParser.Package.receivers list 集合！
```java
        public List<ResolveInfo> queryIntentForPackage(Intent intent, String resolvedType,
                int flags, ArrayList<PackageParser.Activity> packageActivities, int userId) {
            //【1】userId 不存在，返回 null；
            if (!sUserManager.exists(userId)) return null;
            
            //【2】如果 packageActivities 为 null，返回 null；
            if (packageActivities == null) {
                return null;
            }
            mFlags = flags;
            final boolean defaultOnly = (flags&PackageManager.MATCH_DEFAULT_ONLY) != 0;

            final int N = packageActivities.size();

            ArrayList<PackageParser.ActivityIntentInfo[]> listCut =
                new ArrayList<PackageParser.ActivityIntentInfo[]>(N);
            
            //【3】这里是找到 packageActivities 列表中所有 Activity 对象的 ActivityIntentInfo 数组！
            // 将其收集起来，保存到 listCut 列表中！
            ArrayList<PackageParser.ActivityIntentInfo> intentFilters;
            for (int i = 0; i < N; ++i) {
                intentFilters = packageActivities.get(i).intents;
                if (intentFilters != null && intentFilters.size() > 0) {

                    PackageParser.ActivityIntentInfo[] array =
                            new PackageParser.ActivityIntentInfo[intentFilters.size()];
                    intentFilters.toArray(array);
                    listCut.add(array);
                }
            }
            
            //【3.3.1】调用父类的 queryIntentFromList 方法继续查询！
            return super.queryIntentFromList(intent, resolvedType, defaultOnly, listCut, userId);
        }
```

进入父类 IntentResolver 中：

### 3.3.1 ActivityIntentResolver.queryIntentFromList

这里的模板参数 R 为 ResolveInfo 类：
```java
    public List<R> queryIntentFromList(Intent intent, String resolvedType, 
            boolean defaultOnly, ArrayList<F[]> listCut, int userId) {
        //【1】用于保存查询的结果！
        ArrayList<R> resultList = new ArrayList<R>();

        final boolean debug = localLOGV ||
                ((intent.getFlags() & Intent.FLAG_DEBUG_LOG_RESOLUTION) != 0);

        FastImmutableArraySet<String> categories = getFastIntentCategories(intent);
        final String scheme = intent.getScheme();
        int N = listCut.size();
        for (int i = 0; i < N; ++i) {
            //【3.4】遍历 ArrayList<PackageParser.ActivityIntentInfo[]> 列表 listCut
            // 处理每一个 ActivityIntentInfo 数组，也就是 IntentFliter 数组！
            buildResolveList(intent, categories, debug, defaultOnly,
                    resolvedType, scheme, listCut.get(i), resultList, userId);
        }
        filterResults(resultList);

        //【3】对结果进行排序！
        sortResults(resultList);
        return resultList;
    }
```

## 3.4 IntentResolver.buildResolveList

下面我们来看一下，查询静态注册的广播接收者信息的最重要的一个方法，如何通过 `ActivityIntentInfo[]` 数组 src，生成 `List<ResolveList>` 列表 dest：

```java
    private void buildResolveList(Intent intent, FastImmutableArraySet<String> categories,
            boolean debug, boolean defaultOnly,
            String resolvedType, String scheme, F[] src, List<R> dest, int userId) {
        //【1】获得广播 Intent 的 action！
        final String action = intent.getAction();
        //【2】获得广播 Intent 的 uri 数据！
        final Uri data = intent.getData();
        //【3】获得广播 Intent 的 package 数据！
        final String packageName = intent.getPackage();
        //【4】广播是否设置了 FLAG_EXCLUDE_STOPPED_PACKAGES 标志！
        final boolean excludingStopped = intent.isExcludingStopped();

        final Printer logPrinter;
        final PrintWriter logPrintWriter;
        if (debug) {
            logPrinter = new LogPrinter(Log.VERBOSE, TAG, Log.LOG_ID_SYSTEM);
            logPrintWriter = new FastPrintWriter(logPrinter);
        } else {
            logPrinter = null;
            logPrintWriter = null;
        }

        final int N = src != null ? src.length : 0;
        boolean hasNonDefaults = false;
        int i;
        F filter;
        
        //【1】遍历传入的 ActivityIntentInfo[] 数组 src，选择能接受当前广播 Intent 的 ActivityIntentInfo！
        for (i=0; i<N && (filter=src[i]) != null; i++) {
            int match;
            if (debug) Slog.v(TAG, "Matching against filter " + filter);
            
            //【3.4.1】如果广播设置了 FLAG_EXCLUDE_STOPPED_PACKAGES 标签，那就要判断 filter 所属的应用是否被 stop 了！
            // isFilterStopped 方法是 IntentResolver 的子类中实现！
            // 如果满足，那么这个应用是不能接收到当前的广播的！
            if (excludingStopped && isFilterStopped(filter, userId)) {
                if (debug) {
                    Slog.v(TAG, "  Filter's target is stopped; skipping");
                }
                continue;
            }

            //【3.4.2】如果广播 Intent 设置了包名，那就要判断当前 ActivityIntentInfo 所属的目标应用包名是否一致
            // 不一致，说明这个不是目标应用，isPackageForFilter 也是在 IntentResolver 的子类中实现！
            if (packageName != null && !isPackageForFilter(packageName, filter)) {
                if (debug) {
                    Slog.v(TAG, "  Filter is not from package " + packageName + "; skipping");
                }
                continue;
            }

            if (filter.getAutoVerify()) {
                if (localVerificationLOGV || debug) {
                    Slog.v(TAG, "  Filter verified: " + isFilterVerified(filter));
                    int authorities = filter.countDataAuthorities();
                    for (int z = 0; z < authorities; z++) {
                        Slog.v(TAG, "   " + filter.getDataAuthority(z).getHost());
                    }
                }
            }

            //【3.4.3】allowFilterResult 方法也是在 IntentResolver 的子类中实现！
            // 判断我们是否已经添加过这个 filter！
            if (!allowFilterResult(filter, dest)) {
                if (debug) {
                    Slog.v(TAG, "  Filter's target already added");
                }
                continue;
            }
            
            //【2】进行匹配操作，结果返回值 match >= 0，说明匹配成功！
            match = filter.match(action, resolvedType, scheme, data, categories, TAG);
            if (match >= 0) {
                if (debug) Slog.v(TAG, "  Filter matched!  match=0x" +
                        Integer.toHexString(match) + " hasDefault="
                        + filter.hasCategory(Intent.CATEGORY_DEFAULT));
                if (!defaultOnly || filter.hasCategory(Intent.CATEGORY_DEFAULT)) {
                    //【3.4.4】创建 ResolveList 对象！
                    // newResult 方法也是在 IntentResolver 的子类中实现中实现
                    final R oneResult = newResult(filter, match, userId);
                    if (oneResult != null) {
                        //【2.2】添加到目标集合 dest 中！
                        dest.add(oneResult);
                        if (debug) {
                            dumpFilter(logPrintWriter, "    ", filter);
                            logPrintWriter.flush();
                            filter.dump(logPrinter, "    ");
                        }
                    }
                } else {
                    hasNonDefaults = true;
                }
            } else {
                if (debug) {
                    String reason;
                    switch (match) {
                        case IntentFilter.NO_MATCH_ACTION: reason = "action"; break;
                        case IntentFilter.NO_MATCH_CATEGORY: reason = "category"; break;
                        case IntentFilter.NO_MATCH_DATA: reason = "data"; break;
                        case IntentFilter.NO_MATCH_TYPE: reason = "type"; break;
                        default: reason = "unknown reason"; break;
                    }
                    Slog.v(TAG, "  Filter did not match: " + reason);
                }
            }
        }

        if (debug && hasNonDefaults) {
            if (dest.size() == 0) {
                Slog.v(TAG, "resolveIntent failed: found match, but none with CATEGORY_DEFAULT");
            } else if (dest.size() > 1) {
                Slog.v(TAG, "resolveIntent: multiple matches, only some with CATEGORY_DEFAULT");
            }
        }
    }
```
这个方法流程很简单，我们来看一些细节！

### 3.4.1 ActivityIntentResolver ActivityIntentResolver.isFilterStopped
```java
        @Override
        protected boolean isFilterStopped(PackageParser.ActivityIntentInfo filter, int userId) {
            if (!sUserManager.exists(userId)) return true;
            PackageParser.Package p = filter.activity.owner;
            if (p != null) {
                PackageSetting ps = (PackageSetting)p.mExtras;
                if (ps != null) {
                    return (ps.pkgFlags&ApplicationInfo.FLAG_SYSTEM) == 0
                            && ps.getStopped(userId);
                }
            }
            return false;
        }
```
isFilterStopped 方法用来判断 filter 所属的应用是否被强行停止了，返回 true 的情况如下：

- 设备用户 id 不存在；
- filter 所属的应用是非系统应用，且应用的 stoped 属性被置为 true；

该方法返回  true，表示该应用在该设备用户下，被强制停止了！

### 3.4.2 ActivityIntentResolver.isPackageForFilter
```java
        @Override
        protected boolean isPackageForFilter(String packageName,
                PackageParser.ActivityIntentInfo info) {
            //【1】就是匹配下 package name！
            return packageName.equals(info.activity.owner.packageName);
        }
```
这个方法很简单，不用多说了，就是判断 Intent 设置的包名和 filter 所属的应用的包名是不是一样的！一样的话，说明该应用是广播的目标应用！

### 3.4.3 ActivityIntentResolver.allowFilterResult
```java
        @Override
        protected boolean allowFilterResult(
                PackageParser.ActivityIntentInfo filter, List<ResolveInfo> dest) {
            
            //【1】获得 filter 所属的组件的 ActivityInfo 对象！
            ActivityInfo filterAi = filter.activity.info;
            for (int i=dest.size()-1; i>=0; i--) {
                ActivityInfo destAi = dest.get(i).activityInfo;
                if (destAi.name == filterAi.name
                        && destAi.packageName == filterAi.packageName) {
                    return false;
                }
            }
            return true;
        }
```
该方法的作用是是否重复添加过同一个 filter，判断依据是：要被添加的 filter 的组件名和包名是否和 dest 集合中的某个 filter 一样！！

ActivityInfo 是对应组件的信息对象，ActivityInfo.name 这个值来自于 android:name 属性！！

### 3.4.4 ActivityIntentResolver.newResult

最后，就是将 ActivityIntentInfo 对象封装为 ResolveInfo 对象了！
```java
        @Override
        protected ResolveInfo newResult(PackageParser.ActivityIntentInfo info,
                int match, int userId) {
            
            //【1】如果设备用户不存在，返回 null；
            if (!sUserManager.exists(userId)) return null;
            if (!mSettings.isEnabledAndMatchLPr(info.activity.info, mFlags, userId)) {
                return null;
            }
            final PackageParser.Activity activity = info.activity;
            PackageSetting ps = (PackageSetting) activity.owner.mExtras;
            if (ps == null) {
                return null;
            }
            
            //【2】获得组件新的 ActivityInfo 拷贝对象 ai！
            ActivityInfo ai = PackageParser.generateActivityInfo(activity, mFlags,
                    ps.readUserState(userId), userId);

            if (ai == null) {
                return null;
            }
            
            //【3】创建一个 ResolveInfo 对象 res，并设置 res.activityInfo 为 ai；
            final ResolveInfo res = new ResolveInfo();
            res.activityInfo = ai;
            if ((mFlags&PackageManager.GET_RESOLVED_FILTER) != 0) {
                res.filter = info;
            }
            if (info != null) {
                res.handleAllWebDataURI = info.handleAllWebDataURI();
            }
            res.priority = info.getPriority();
            res.preferredOrder = activity.owner.mPreferredOrder;
            //System.out.println("Result: " + res.activityInfo.className +
            //                   " = " + res.priority);
            res.match = match;
            res.isDefault = info.hasDefault;
            res.labelRes = info.labelRes;
            res.nonLocalizedLabel = info.nonLocalizedLabel;
            if (userNeedsBadging(userId)) {
                res.noResourceId = true;
            } else {
                res.icon = info.icon;
            }
            res.iconResourceId = info.icon;
            //【4】是否是系统应用！
            res.system = res.activityInfo.applicationInfo.isSystemApp();
            //【5】返回我们创建的新的 ResolveInfo 对象！
            return res;
        }

```

到这里，我们就知道了如何查询系统中所有注册过的广播接收者了！

# 4 总结！

我们来看看广播的静态注册的类关系图：

## 4.1 静态注册类关系图

PMS 解析了静态注册的广播接收者后，数据结婚间的关系图如下：

<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/23/广播静态注册-20221024234258349.png" alt="广播静态注册.png-169.5kB" style="zoom:50%;" /> 

## 4.2 查询注册结果关系图