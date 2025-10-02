# Permission第 1 篇 - Permission 配置和解析
title: Permission第 1 篇 - Permission 配置和解析
date: 2017/05/09 20:46:25
catalog: true
categories:
- AndroidFramework源码分析
- Permission权限管理
tags: Permission权限管理
---

[toc]

# 0 综述

基于 Android 7.1.1，分析权限管理相关知识，下面关于 xml 配置的内容来自于 Android developer 文档，在加上我自身的理解！


# 1 Application 的权限配置和解析

## 1.1 权限配置

### 1.1.1 manifest 配置

和 Application 权限相关配置相关的内容主要有 2 部分：

- 一个是在 `<manifest></manifest>` 中定义的和权限相关的配置；
- 一个是在 `<application></application>` 中定义的和权限相关的配置；

```xml
    <uses-permission android:name="string"
                     android:maxSdkVersion="integer" />
    
    <permission android:description="string resource"
                android:icon="drawable resource"
                android:label="string resource"
                android:name="string"
                android:permissionFlags="[costsMoney | removed | installed ]"
                android:permissionGroup="string"
                android:protectionLevel="[normal | dangerous | signature | ...]" />

    <permission-tree android:icon="drawable resource"
                     android:label="string resource"
                     android:name="string" />
                     
    <permission-group android:description="string resource"
                      android:icon="drawable resource"
                      android:label="string resource"
                      android:name="string" 
                      android:parsePermissionGroup="string"
                      android:priority="string" />
```
下面我们来解释一下，每个标签的意思：

- **uses-permission**

用来申请应用要使用的权限，包括安装时权限和运行时权限！

| 属性名 | 解释  |
|--------|:----  |
| **android:name** | 表示申请的权限名，可以是应用自身通过 `<permission>` 元素定义的权限、另一个应用定义的权限，或者一个标准系统权限！ | 
| **android:maxSdkVersion**|不常用，表示应用需要申请该权限的最高 API 级别，高于这个值就无需申请权限！|  


- **permission**

声明一个权限，可用于限制对此应用程序或其他应用程序的特定组件或功能的访问。

| 属性名 | 解释  |
|--------|:----  |
| **android:description** | 用于显示给用户的权限描述，解释权限 | 
| **android:icon** | 表示权限的图标| 
| **android:label** | 权限的可视化名称，用于显示给用户| 
| **android:name** | 权限的名称，用于在代码中引用该权限，例如， `<uses-permission>` 元素，应用组件的 `android:permission` 属性 | 
| **android:permissionFlags** | 权限的额外标志位| 
| **android:permissionGroup** | 权限所属权限组的名称，必须是此应用程序或其他应用程序中的 `<permission-group>` 元素声明的组名 。如果未设置此属性，则权限不属于组|
| **android:protectionLevel** | 权限保护级别，每个保护级别由基本权限类型和零个或多个标志组成。例如，"dangerous"保护级别没有标志。相反，保护级别 "signature\|privileged" 是"signature"基本权限类型和 "privileged"标志的组合。 | 

以上就是和 permission 相关的配置！

- **permission-group**

为相关权限的逻辑分组声明一个名称。权限通过其 android:permissionGroup 属性加入权限组。权限组的所有权限会在用户界面中一起呈现。

其本身并未声明权限，只是申明了一个可以放置权限的类别。

| 属性名 | 解释  |
|--------|:----  |
| **android:description** | 用于显示给用户的组描述| 
| **android:icon** | 表示权限组的图标| 
| **android:label** | 权限组的可视化名称，用于显示给用户| 
| **android:name** | 组的名称，用于在代码中引用该权限组，例如， `<permission>` 的 `android:android:permissionGroup` 属性 | 

继续来看：

- **permission-tree**

权限树和权限组很类似，该标签用于声明权限树的基本名称，定义了权限树的应用程序拥有权限树名称的所有权，可以调用 PackageManager.addPermission() 向树中动态添加权限！

此元素本身并未声明权限，只是定义了可以动态放置权限的名称空间。

| 属性名 | 解释  |
|--------|:----  |
| **android:icon** | 表示树中所有权限的图标；| 
| **android:label** | 用户可读的组名称；| 
| **android:name** | 权限树的名称，为树中所有权限名称的前缀，用于在代码中引用该权限组，该名称的路径中必须包含至少两个以句点分隔的段| 


### 1.1.2 application 配置
```xml
	<application
        android:permission="" >
	</application>
```

- **android:permission**

permission 属性表示访问该 application 应该具有的权限！

## 1.2 权限解析

下面，我们深入到 PMS 中，看下 Application 权限信息是如何解析的，这部分从两方面来看：

- manifest 的属性配置解析；
- application 的属性配置解析；

### 1.2.1 PackageParser.parseBaseApkCommon

我们先来看看 manifest 的属性配置解析！

```java
    private Package parseBaseApkCommon(Package pkg, Set<String> acceptedTags, Resources res,
            XmlResourceParser parser, int flags, String[] outError) throws XmlPullParserException,
            IOException {
        ... ... ... ...

        int outerDepth = parser.getDepth();
        while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
                && (type != XmlPullParser.END_TAG || parser.getDepth() > outerDepth)) {
            if (type == XmlPullParser.END_TAG || type == XmlPullParser.TEXT) {
                continue;
            }

            String tagName = parser.getName();

            if (acceptedTags != null && !acceptedTags.contains(tagName)) {
                Slog.w(TAG, "Skipping unsupported element under <manifest>: "
                        + tagName + " at " + mArchiveSourcePath + " "
                        + parser.getPositionDescription());
                XmlUtils.skipCurrentTag(parser);
                continue;
            }

            if (tagName.equals(TAG_APPLICATION)) { //【1】解析 application 标签属性！
                if (foundApp) {
                    if (RIGID_PARSER) {
                        outError[0] = "<manifest> has more than one <application>";
                        mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
                        return null;
                    } else {
                        Slog.w(TAG, "<manifest> has more than one <application>");
                        XmlUtils.skipCurrentTag(parser);
                        continue;
                    }
                }

                foundApp = true;

                //【*1.2.2】调用 parseBaseApplication 解析 application 属性！
                if (!parseBaseApplication(pkg, res, parser, flags, outError)) {
                    return null;
                }
            } else if (
            
                ... ... ...
            
            } else if (tagName.equals(TAG_PERMISSION_GROUP)) {
                //【*1.2.1.1】解析 permission-group 标签属性
                if (parsePermissionGroup(pkg, flags, res, parser, outError) == null) {
                    return null;
                }
            } else if (tagName.equals(TAG_PERMISSION)) {
                //【*1.2.1.2】解析 permission 标签属性
                if (parsePermission(pkg, res, parser, outError) == null) {
                    return null;
                }
            } else if (tagName.equals(TAG_PERMISSION_TREE)) { 
                //【*1.2.1.3】解析 permission-tree 标签属性
                if (parsePermissionTree(pkg, res, parser, outError) == null) {
                    return null;
                }
            } else if (tagName.equals(TAG_USES_PERMISSION)) {
                //【*1.2.1.4】解析 uses-permission 标签属性
                if (!parseUsesPermission(pkg, res, parser)) {
                    return null;
                }
            } else if (tagName.equals(TAG_USES_PERMISSION_SDK_M)
                    || tagName.equals(TAG_USES_PERMISSION_SDK_23)) { 
                //【*1.2.1.4】解析 uses-permission-sdk-m/23 标签属性
                if (!parseUsesPermission(pkg, res, parser)) {
                    return null;
                }
            } else if (
            
                ... ... ... ...
            
            } else if (tagName.equals(TAG_ADOPT_PERMISSIONS)) {
                //【2】解析 adopt-permissions 标签
                sa = res.obtainAttributes(parser, // 解析源包属性
                        com.android.internal.R.styleable.AndroidManifestOriginalPackage);

                String name = sa.getNonConfigurationString( // 解析权限名属性
                        com.android.internal.R.styleable.AndroidManifestOriginalPackage_name, 0);

                sa.recycle();

                if (name != null) {
                    //【2.1】加入到 Package.mAdoptPermissions 集合中！
                    if (pkg.mAdoptPermissions == null) {
                        pkg.mAdoptPermissions = new ArrayList<String>();
                    }
                    pkg.mAdoptPermissions.add(name);
                }

                XmlUtils.skipCurrentTag(parser);

            } else if (
            
                ... ... ... ...
                
            }
        }

        ... ... ... ...

        //【3】处理一些新添加的权限，如果当前应用的 targetSdkVersion 小于这些权限的 sdkVersion！
        // 我们会默认将其添加到 Package.requestedPermissions 中！
        final int NP = PackageParser.NEW_PERMISSIONS.length;
        StringBuilder implicitPerms = null;
        for (int ip = 0; ip < NP; ip++) {
            final PackageParser.NewPermissionInfo npi
                    = PackageParser.NEW_PERMISSIONS[ip];
            //【3.1】如果应用的 target SDK Version 高于权限的该权限的 version，不做处理
            if (pkg.applicationInfo.targetSdkVersion >= npi.sdkVersion) {
                break;
            }
            if (!pkg.requestedPermissions.contains(npi.name)) {
                if (implicitPerms == null) {
                    implicitPerms = new StringBuilder(128);
                    implicitPerms.append(pkg.packageName);
                    implicitPerms.append(": compat added ");
                } else {
                    implicitPerms.append(' ');
                }
                implicitPerms.append(npi.name);
                pkg.requestedPermissions.add(npi.name);
            }
        }
        if (implicitPerms != null) {
            Slog.i(TAG, implicitPerms.toString());
        }

        //【4】处理一些相互依赖的权限，这里除了有 sdkVersion 限制之外，
        // 还有一个判断 rootPerm 是否在 requestedPermissions 列表中的判断 ！
        final int NS = PackageParser.SPLIT_PERMISSIONS.length;
        for (int is = 0; is < NS; is++) {
            final PackageParser.SplitPermissionInfo spi
                    = PackageParser.SPLIT_PERMISSIONS[is];
            //【4.1】如果应用的 target SDK Version 高于权限的该权限的 version，不做处理，
            // 或者应用没有请求这个根权限，不用处理！
            if (pkg.applicationInfo.targetSdkVersion >= spi.targetSdk
                    || !pkg.requestedPermissions.contains(spi.rootPerm)) {
                continue;
            }
            for (int in=0; in < spi.newPerms.length; in++) {
                final String perm = spi.newPerms[in];
                if (!pkg.requestedPermissions.contains(perm)) {
                    pkg.requestedPermissions.add(perm);
                }
            }
        }

        ... ... ...

        return pkg;
    }
```
最后，处理了下在指定版本中出现的新权限的添加！

```java
    public static final PackageParser.NewPermissionInfo NEW_PERMISSIONS[] =
        new PackageParser.NewPermissionInfo[] {
            new PackageParser.NewPermissionInfo(android.Manifest.permission.WRITE_EXTERNAL_STORAGE,
                    android.os.Build.VERSION_CODES.DONUT, 0),
            new PackageParser.NewPermissionInfo(android.Manifest.permission.READ_PHONE_STATE,
                    android.os.Build.VERSION_CODES.DONUT, 0)
    };
```

Build.VERSION_CODES.DONUT 的值为 4，表示 Android 1.6，意思是如果是 Android 1.6 之前话，应用如果没有申请 NEW_PERMISSIONS 的权限，那么我们会默认给他申请，因为我们关注的是 Android 7.1.1 那么这里我们不会进入！

对于 SPLIT_PERMISSIONS 列表中的权限，保存了 root 权限和相互依赖的权限，以及首次出现的平台！

举个例子，如果应用只是申请了 WRITE_EXTERNAL_STORAGE，即使其在 Manifest 中没有申明 READ_EXTERNAL_STORAGE 权限，我们也会将其主动加入到 Package.requestedPermissions！！
```java
    public static final PackageParser.SplitPermissionInfo SPLIT_PERMISSIONS[] =
        new PackageParser.SplitPermissionInfo[] {
            new PackageParser.SplitPermissionInfo(android.Manifest.permission.WRITE_EXTERNAL_STORAGE,
                    new String[] { android.Manifest.permission.READ_EXTERNAL_STORAGE },
                    android.os.Build.VERSION_CODES.CUR_DEVELOPMENT+1),
            new PackageParser.SplitPermissionInfo(android.Manifest.permission.READ_CONTACTS,
                    new String[] { android.Manifest.permission.READ_CALL_LOG },
                    android.os.Build.VERSION_CODES.JELLY_BEAN),
            new PackageParser.SplitPermissionInfo(android.Manifest.permission.WRITE_CONTACTS,
                    new String[] { android.Manifest.permission.WRITE_CALL_LOG },
                    android.os.Build.VERSION_CODES.JELLY_BEAN)
    };
```
Build.VERSION_CODES.JELLY_BEAN 的值为 16，表示 Android 4.1；

Build.VERSION_CODES.CUR_DEVELOPMENT 值为 10000，表示开发者版本！

这段代码的意思是：如果是 Android 4.1 之前，且应用申请了 SPLIT_PERMISSIONS 中指定的根权限，那么我们也会将其对应的子权限添加到其申请列表中，因为我们关注的是 Android 7.1.1 那么这里我们不会进入！


#### 1.2.1.1 PackageP.parsePermissionGroup

parsePermissionGroup 用于处理 permission-group 的解析：

```java
    private PermissionGroup parsePermissionGroup(Package owner, int flags, Resources res,
            XmlResourceParser parser, String[] outError)
            throws XmlPullParserException, IOException {
        //【1】创建 PermissionGroup 对象！
        PermissionGroup perm = new PermissionGroup(owner);
        TypedArray sa = res.obtainAttributes(parser,
                com.android.internal.R.styleable.AndroidManifestPermissionGroup);
        if (!parsePackageItemInfo(owner, perm.info, outError,
                "<permission-group>", sa, true /*nameRequired*/,
                com.android.internal.R.styleable.AndroidManifestPermissionGroup_name,
                com.android.internal.R.styleable.AndroidManifestPermissionGroup_label,
                com.android.internal.R.styleable.AndroidManifestPermissionGroup_icon,
                com.android.internal.R.styleable.AndroidManifestPermissionGroup_roundIcon,
                com.android.internal.R.styleable.AndroidManifestPermissionGroup_logo,
                com.android.internal.R.styleable.AndroidManifestPermissionGroup_banner)) {
            sa.recycle();
            mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
            return null;
        }

        perm.info.descriptionRes = sa.getResourceId(
                com.android.internal.R.styleable.AndroidManifestPermissionGroup_description, 0);
        
         //【2】解析 permissionGroupFlags 属性；
        perm.info.flags = sa.getInt(
                com.android.internal.R.styleable.AndroidManifestPermissionGroup_permissionGroupFlags, 0);
         //【3】解析 priority 属性；
        perm.info.priority = sa.getInt(
                com.android.internal.R.styleable.AndroidManifestPermissionGroup_priority, 0);

        //【4】只有系统应用才能设置权限组的优先级
        if (perm.info.priority > 0 && (flags & PARSE_IS_SYSTEM) == 0) {
            perm.info.priority = 0;
        }

        sa.recycle();

        if (!parseAllMetaData(res, parser, "<permission-group>", perm,
                outError)) {
            mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
            return null;
        }

        owner.permissionGroups.add(perm);

        return perm;
    }
```
只有系统应用才能设置权限组的优先级！

创建了 PermissionGroup 对象！
```java
    public final static class PermissionGroup extends Component<IntentInfo> {
        public final PermissionGroupInfo info;

        public PermissionGroup(Package _owner) {
            super(_owner);
            info = new PermissionGroupInfo();
        }

        public PermissionGroup(Package _owner, PermissionGroupInfo _info) {
            super(_owner);
            info = _info;
        }

        public void setPackageName(String packageName) {
            super.setPackageName(packageName);
            info.packageName = packageName;
        }

        public String toString() {
            return "PermissionGroup{"
                + Integer.toHexString(System.identityHashCode(this))
                + " " + info.name + "}";
        }
    }
```

最终的解析结果会保存到 Package.permissionGroups 中！

#### 1.2.1.2 PackageP.parsePermission
parsePermission 用于处理 permission 的解析：

```java
    private Permission parsePermission(Package owner, Resources res,
            XmlResourceParser parser, String[] outError)
        throws XmlPullParserException, IOException {
        //【1】创建一个 Permission 对象，封装 permission-tree 
        Permission perm = new Permission(owner);

        TypedArray sa = res.obtainAttributes(parser,
                com.android.internal.R.styleable.AndroidManifestPermission);

        if (!parsePackageItemInfo(owner, perm.info, outError,
                "<permission>", sa, true /*nameRequired*/,
                com.android.internal.R.styleable.AndroidManifestPermission_name,
                com.android.internal.R.styleable.AndroidManifestPermission_label,
                com.android.internal.R.styleable.AndroidManifestPermission_icon,
                com.android.internal.R.styleable.AndroidManifestPermission_roundIcon,
                com.android.internal.R.styleable.AndroidManifestPermission_logo,
                com.android.internal.R.styleable.AndroidManifestPermission_banner)) {
            sa.recycle();
            mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
            return null;
        }

        //【2】解析权限所属的组信息！
        perm.info.group = sa.getNonResourceString(
                com.android.internal.R.styleable.AndroidManifestPermission_permissionGroup);
        if (perm.info.group != null) {
            perm.info.group = perm.info.group.intern();
        }

        perm.info.descriptionRes = sa.getResourceId(
                com.android.internal.R.styleable.AndroidManifestPermission_description,
                0);
        //【3】解析权限的保护级别，默认为 normal！
        perm.info.protectionLevel = sa.getInt(
                com.android.internal.R.styleable.AndroidManifestPermission_protectionLevel,
                PermissionInfo.PROTECTION_NORMAL);

        //【4】解析权限的 flags，默认为 0！
        perm.info.flags = sa.getInt(
                com.android.internal.R.styleable.AndroidManifestPermission_permissionFlags, 0);

        sa.recycle();

        if (perm.info.protectionLevel == -1) {
            outError[0] = "<permission> does not specify protectionLevel";
            mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
            return null;
        }
        
        //【5】将 signatureOrSystem 转为 signature|privileged
        perm.info.protectionLevel = PermissionInfo.fixProtectionLevel(perm.info.protectionLevel);

        //【6】这里是检查保护级别的有效性！
        // 如果 protectionLevel 设置了额外的标志位，那么其基本权限类型必须是 signature！！
        if ((perm.info.protectionLevel & PermissionInfo.PROTECTION_MASK_FLAGS) != 0) {
            if ((perm.info.protectionLevel & PermissionInfo.PROTECTION_MASK_BASE) !=
                    PermissionInfo.PROTECTION_SIGNATURE) {
                outError[0] = "<permission>  protectionLevel specifies a flag but is "
                        + "not based on signature type";
                mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
                return null;
            }
        }

        if (!parseAllMetaData(res, parser, "<permission>", perm, outError)) { // 解析 meta-data 属性！
            mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
            return null;
        }

        owner.permissions.add(perm);

        return perm;
    }
```
这里会涉及到 2 个标志位：
```java
    // 用于获取基本权限类型
    public static final int PROTECTION_MASK_BASE = 0xf;

    // 用于获取额外的标志位
    public static final int PROTECTION_MASK_FLAGS = 0xff0;
```
我们可以通过如下方式获得基本权限类型和附加标志位：
```java
protectionLevel & PermissionInfo.PROTECTION_MASK_BASE // 获取基本权限类型；
protectionLevel & PermissionInfo.PROTECTION_MASK_FLAGS // 获取额外的标志位；
```

fixProtectionLevel 的意义就是将 signatureOrSystem 转为 signature|privileged 而已！
```java
    /** @hide */
    public static int fixProtectionLevel(int level) {
        if (level == PROTECTION_SIGNATURE_OR_SYSTEM) {
            level = PROTECTION_SIGNATURE | PROTECTION_FLAG_PRIVILEGED;
        }
        return level;
    }
```
如果 protectionLevel 设置了额外的标志位，那么其基本权限类型必须是 signature！！

解析的结果会封装到一个 Permission 对象中！
```java
    public final static class Permission extends Component<IntentInfo> {
        public final PermissionInfo info; // 用于保存该权限的信息！
        public boolean tree; // 是否是权限树，是权限则为 false；
        public PermissionGroup group;

        public Permission(Package _owner) {
            super(_owner);
            info = new PermissionInfo();
        }

        public Permission(Package _owner, PermissionInfo _info) {
            super(_owner);
            info = _info;
        }

        public void setPackageName(String packageName) {
            super.setPackageName(packageName);
            info.packageName = packageName;
        }

        public String toString() {
            return "Permission{"
                + Integer.toHexString(System.identityHashCode(this))
                + " " + info.name + "}";
        }
    }
```

最终的解析结果会保存到 Package.permissions 中！


#### 1.2.1.3 PackageP.parsePermissionTree

parsePermissionTree 用于处理 permission-tree 的解析：
```java
    private Permission parsePermissionTree(Package owner, Resources res,
            XmlResourceParser parser, String[] outError)
        throws XmlPullParserException, IOException {
        //【1】创建一个 Permission 对象，封装 permission-tree 
        Permission perm = new Permission(owner);

        TypedArray sa = res.obtainAttributes(parser,
                com.android.internal.R.styleable.AndroidManifestPermissionTree);

        if (!parsePackageItemInfo(owner, perm.info, outError,
                "<permission-tree>", sa, true /*nameRequired*/,
                com.android.internal.R.styleable.AndroidManifestPermissionTree_name,
                com.android.internal.R.styleable.AndroidManifestPermissionTree_label,
                com.android.internal.R.styleable.AndroidManifestPermissionTree_icon,
                com.android.internal.R.styleable.AndroidManifestPermissionTree_roundIcon,
                com.android.internal.R.styleable.AndroidManifestPermissionTree_logo,
                com.android.internal.R.styleable.AndroidManifestPermissionTree_banner)) {
            sa.recycle();
            mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
            return null;
        }

        sa.recycle();

        //【2】校验权限树的名称合法性，必须是至少由两个 “.” 分割开的段！
        int index = perm.info.name.indexOf('.');
        if (index > 0) {
            index = perm.info.name.indexOf('.', index+1);
        }
        if (index < 0) {
            outError[0] = "<permission-tree> name has less than three segments: "
                + perm.info.name;
            mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
            return null;
        }

        perm.info.descriptionRes = 0;
        perm.info.protectionLevel = PermissionInfo.PROTECTION_NORMAL; // 默认为 nomal
        perm.tree = true; // 设置 perm.tree，表示其为权限树！

        if (!parseAllMetaData(res, parser, "<permission-tree>", perm,
                outError)) {
            mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
            return null;
        }

        owner.permissions.add(perm);

        return perm;
    }

```
我们看到，对于权限树，也是会创建一个 Permission 对象！

最终的解析结果会保存到 Package.permissions 中！


#### 1.2.1.4 PackageP.parseUsesPermission

parseUsesPermission 用于处理 uses-permission 的解析：

```java
    private boolean parseUsesPermission(Package pkg, Resources res, XmlResourceParser parser)
            throws XmlPullParserException, IOException {
        TypedArray sa = res.obtainAttributes(parser,
                com.android.internal.R.styleable.AndroidManifestUsesPermission);

        String name = sa.getNonResourceString( // 解析 name 属性；
                com.android.internal.R.styleable.AndroidManifestUsesPermission_name);

        int maxSdkVersion = 0;
        TypedValue val = sa.peekValue( // 解析 maxSdkVersion 属性；
                com.android.internal.R.styleable.AndroidManifestUsesPermission_maxSdkVersion);
        if (val != null) {
            if (val.type >= TypedValue.TYPE_FIRST_INT && val.type <= TypedValue.TYPE_LAST_INT) {
                maxSdkVersion = val.data;
            }
        }

        sa.recycle();

        if ((maxSdkVersion == 0) || (maxSdkVersion >= Build.VERSION.RESOURCES_SDK_INT)) {
            if (name != null) {
                int index = pkg.requestedPermissions.indexOf(name);
                if (index == -1) {
                    // 将该 package 
                    pkg.requestedPermissions.add(name.intern());
                } else {
                    Slog.w(TAG, "Ignoring duplicate uses-permissions/uses-permissions-sdk-m: "
                            + name + " in package: " + pkg.packageName + " at: "
                            + parser.getPositionDescription());
                }
            }
        }

        XmlUtils.skipCurrentTag(parser);
        return true;
    }
```
最终的解析结果会保存到 Package.requestedPermissions 中！


### 1.2.2 PackageParser.parseBaseApplication

我们先来看看 application 的属性配置解析！

```java
    private boolean parseBaseApplication(Package owner, Resources res,
            XmlResourceParser parser, int flags, String[] outError)
        throws XmlPullParserException, IOException {
        //【1.1】获得 package 的 ApplicationInfo 成员变量，封装 Application 标签的信息
        final ApplicationInfo ai = owner.applicationInfo;
        final String pkgName = owner.applicationInfo.packageName;

        ... ... ...

        //【1.2】解析 android:permission 属性
        String str;
        str = sa.getNonConfigurationString(
                com.android.internal.R.styleable.AndroidManifestApplication_permission, 0);
        ai.permission = (str != null && str.length() > 0) ? str.intern() : null;

        ... ... ...
        
        while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
                && (type != XmlPullParser.END_TAG || parser.getDepth() > innerDepth)) {
            if (type == XmlPullParser.END_TAG || type == XmlPullParser.TEXT) {
                continue;
            }

            String tagName = parser.getName();
            if (tagName.equals("activity")) { //【2】解析 activity 属性
                Activity a = parseActivity(owner, res, parser, flags, outError, false,
                        owner.baseHardwareAccelerated);
                if (a == null) {
                    mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
                    return false;
                }

                owner.activities.add(a);

            } else if (tagName.equals("receiver")) { //【3】解析 receiver 属性
                Activity a = parseActivity(owner, res, parser, flags, outError, true, false);
                if (a == null) {
                    mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
                    return false;
                }

                owner.receivers.add(a);

            } else if (tagName.equals("service")) { //【4】解析 service 属性
                Service s = parseService(owner, res, parser, flags, outError);
                if (s == null) {
                    mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
                    return false;
                }

                owner.services.add(s);

            } else if (tagName.equals("provider")) { //【5】解析 provider 属性
                Provider p = parseProvider(owner, res, parser, flags, outError);
                if (p == null) {
                    mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
                    return false;
                }

                owner.providers.add(p);

            } else if (
            
                ... ... ...
            
            }
        }

        modifySharedLibrariesForBackwardCompatibility(owner);

        if (hasDomainURLs(owner)) {
            owner.applicationInfo.privateFlags |= ApplicationInfo.PRIVATE_FLAG_HAS_DOMAIN_URLS;
        } else {
            owner.applicationInfo.privateFlags &= ~ApplicationInfo.PRIVATE_FLAG_HAS_DOMAIN_URLS;
        }

        return true;
    }

```
最后，解析的 android:permission 属性会保存到 Provider.ApplicationInfo.permission 中去！

# 2 Activity 的权限配置和解析

## 2.1 权限配置

在 Activity 中有如下的权限相关配置：

```android
android:exported=["true" | "false"]
android:permission="string"
```

- exported 属性

exported 属性决定了该 activity 是否暴露给其他应用！

- permission 属性

permission 属性表示启动该 activity 应该具有的权限

## 2.2 权限解析

### 2.2.1 PackageParser.parseActivity

Activity 使用是 parseActivity 方法解析，对于 Activity，boolean receiver 取值为 false：

```java 
    private Activity parseActivity(Package owner, Resources res,
            XmlResourceParser parser, int flags, String[] outError,
            boolean receiver, boolean hardwareAccelerated)
            throws XmlPullParserException, IOException {
        ... ... ...

        //【1】创建了一个 Activity 对象，表示 Receiver  组件，创建了 ActivityInfo 对象，封装 Receiver 信息！
        Activity a = new Activity(mParseActivityArgs, new ActivityInfo());
        if (outError[0] != null) {
            sa.recycle();
            return null;
        }
        
        //【2】解析 android:exported 属性！
        boolean setExported = sa.hasValue(R.styleable.AndroidManifestActivity_exported);
        if (setExported) {
            a.info.exported = sa.getBoolean(R.styleable.AndroidManifestActivity_exported, false);
        }

        ... ... ...

        //【3】解析 android:permission 属性！
        String str;
        str = sa.getNonConfigurationString(R.styleable.AndroidManifestActivity_permission, 0);
        if (str == null) {
            a.info.permission = owner.applicationInfo.permission;
        } else {
            a.info.permission = str.length() > 0 ? str.toString().intern() : null;
        }

        ... ... ...
        
        if (!setExported) { 
            //【4】如果没有显式设置 android:exported，取决于是否设置了意图过滤器；
            s.info.exported = s.intents.size() > 0;
        }
        
        return a;
    }
```
**属性解析**：

- android:exported 属性的值会保存到 Activity.ActivityInfo.exported 中；
- android:permission 会保存到 Activity.ActivityInfo.permission 中；

android:permission 使用的权限必须在 permission 标签中定义才可以使用！

# 3 Service 的权限配置和解析

## 3.1 权限配置
```xml
<service
         android:exported=["true" | "false"]
         android:permission="string" >
    . . .
</service>
```
- **android:exported**

其他应用程序的组件是否可以调用服务或与其进行交互，为 true 表示可以，为 false 表示不可以。

当值为 false 时，只有具有相同 uid 的应用程序或该应用程序的组件才能启动服务或绑定到该服务。

默认值取决于服务是否包含意图过滤器，如果包含，则为 true；否则为 false；

- **android:permission**

启动服务或绑定服务所必须具有的权限；

如果未设置此属性，则由该 application 元素 permission 属性设置的权限将 应用于该服务。如果两个属性均未设置，则该服务不受权限保护。

## 3.2 权限解析

### 3.2.1 PackageParser.parseService

Service 使用是 parseService 方法解析:

```java
    private Service parseService(Package owner, Resources res,
            XmlResourceParser parser, int flags, String[] outError)
            throws XmlPullParserException, IOException {
        TypedArray sa = res.obtainAttributes(parser,
                com.android.internal.R.styleable.AndroidManifestService);

        ... ... ...

        //【1】创建 Service 对象，代表 Service 组件，创建 ServiceInfo 对象，封装 Service 的信息！
        Service s = new Service(mParseServiceArgs, new ServiceInfo());
        if (outError[0] != null) {
            sa.recycle();
            return null;
        }
        //【2】解析 android:exported 属性；
        boolean setExported = sa.hasValue(
                com.android.internal.R.styleable.AndroidManifestService_exported);
        if (setExported) {
            s.info.exported = sa.getBoolean(
                    com.android.internal.R.styleable.AndroidManifestService_exported, false);
        }
        //【3】解析 android:permission 属性；
        String str = sa.getNonConfigurationString(
                com.android.internal.R.styleable.AndroidManifestService_permission, 0);
        if (str == null) {
            s.info.permission = owner.applicationInfo.permission;
        } else {
            s.info.permission = str.length() > 0 ? str.toString().intern() : null;
        }

        ... ... ... ...

        if (!setExported) { 
            //【4】如果没有显式设置 android:exported，取决于是否设置了意图过滤器；
            s.info.exported = s.intents.size() > 0;
        }

        return s;
    }
```

# 4 Receiver 的权限配置和解析

## 4.1 权限配置

```xml
<receiver 
          android:exported=["true" | "false"]
          android:permission="string" >
    . . .
</receiver>
```

- **android:exported**

广播接收器是否可以接收来自其应用程序之外的消息的消息 ，为 true 表示可以，为 false 表示不可以。

当值为 false 时，则广播接收器可以接收的唯一消息是由具有相同 uid 的应用程序或该应用程序自身的组件发送的消息。

默认值取决于广播接收者是否包含意图过滤器，如果包含，则为 true；否则为 false；

- **android:permission**

向该广播接收者发送消息必须具有的权限；

如果未设置此属性，则该 application 元素 permission 属性设置的权限将 应用于广播接收器。如果两个属性均未设置，则接收方不受权限保护。

## 4.2 权限解析

我们回到 PMS 中去，看下应用程序的 Receiver 组件是如何被解析的，我们重点关注和权限相关的：

### 4.2.1 PackageParser.parseActivity

Receiver 和 Activity 使用是同一个方法，对于 Receiver，boolean receiver 取值为 true；

```java 
    private Activity parseActivity(Package owner, Resources res,
            XmlResourceParser parser, int flags, String[] outError,
            boolean receiver, boolean hardwareAccelerated)
            throws XmlPullParserException, IOException {
        ... ... ...

        //【1】创建了一个 Activity 对象，表示 Receiver  组件，创建了 ActivityInfo 对象，封装 Receiver 信息！
        Activity a = new Activity(mParseActivityArgs, new ActivityInfo());
        if (outError[0] != null) {
            sa.recycle();
            return null;
        }
        
        //【2】解析 android:exported 属性！
        boolean setExported = sa.hasValue(R.styleable.AndroidManifestActivity_exported);
        if (setExported) {
            a.info.exported = sa.getBoolean(R.styleable.AndroidManifestActivity_exported, false);
        }

        ... ... ...

        //【3】解析 android:permission 属性！
        String str;
        str = sa.getNonConfigurationString(R.styleable.AndroidManifestActivity_permission, 0);
        if (str == null) {
            a.info.permission = owner.applicationInfo.permission;
        } else {
            a.info.permission = str.length() > 0 ? str.toString().intern() : null;
        }

        ... ... ...
        
        if (!setExported) { 
            //【4】如果没有显式设置 android:exported，取决于是否设置了意图过滤器；
            s.info.exported = s.intents.size() > 0;
        }

        return a;
    }
```
**属性解析**：

- android:exported 属性的值会保存到 Activity.ActivityInfo.exported 中；
- android:permission 会保存到 Activity.ActivityInfo.permission 中；

# 5 Provider 的权限配置和解析

## 5.1 权限配置

```xml
<provider android:exported=["true" | "false"]
          android:grantUriPermissions=["true" | "false"]
          android:permission="string"
          android:readPermission="string"
          android:writePermission="string" >
</provider>
```
和 provider 相关的权限配置比较多，我们来分别看下：


### 5.1.1 **android:exported**

provider 是否被其他应用程序使用：

- 如果为 true， provider 可被其他应用程序使用。任何应用程序都可以使用提供程序的内容 URI 来访问它，但必须持有 provider 指定的权限。
- 如果为 false，provider 不能被其他应用程序使用，只有具有与提供者相同的 uid 的应用程序或者自身才有权访问它。

因为此属性是在 API 级别 17 中引入的，在 API 17 及以后，默认为 false！

### 5.1.2 **android:permission**

应用程序读取或者写入 provider 必须要有的权限，readPermission和 writePermission 设置的权限优先级更高！

如果 readPermission 属性也被设置，则它控制查询权限。如果 writePermission 属性被设置，它将控制访问权限。

### 5.1.3 **android:readPermission**

应用程序读取 provider 数据必须要有的权限。

### 5.1.4 **android:writePermission**

应用程序向 provider 写入数据必须要有的权限。

### 5.1.5 **android:grantUriPermissions**

用于临时授予应用程序访问该 provider 的全部 uri 的权限！

该属性默认为 false，通过该属性，那些没有被授予 permission/readPermission/writePermission 指定的权限的应用可以临时访问 provider！

- 如果为 true，那么应用有权限临时访问该 provider 的所有数据；

- 如果为 false，那么应用将不会被授予临时权限，除非 provider 还设置 grant-uri-permission，那应用程序将有权限访问 grant-uri-permission 指定的特定数据子集！

通过这种方式，可以让应用程序组件能够一次性地临时访问受权限保护的数据！ 

**eg：**当电子邮件包含附件时，邮件应用程序可能会调用适当的查看器来打开它，即使查看器没有一般的权限来查看所有内容提供者的数据。

可以通过设置 Intent.FLAG_GRANT_READ_URI_PERMISSION 和 Intent.FLAG_GRANT_WRITE_URI_PERMISSION 标志来授予对指定 Uri 的一次性的访问权限 。

**eg：**邮件应用程序会将 FLAG_GRANT_READ_URI_PERMISSION 标志位传递给 Context.startActivity() 的 Intent，被启动的 actitivity 会获得上面的标志位和 Uri，从而具有对 uri 的读权限！

### 5.1.6 provider 子标签

```xml
    <path-permission android:path="string"
                 android:pathPrefix="string"
                 android:pathPattern="string"
                 android:permission="string"
                 android:readPermission="string"
                 android:writePermission="string" />

    <grant-uri-permission android:path="string"
                      android:pathPrefix="string" 
                      android:pathPattern="string"/>
```
grant-uri-permission 和 path-permission 均为 provider 的子标签！

#### 5.1.6.1 **grant-uri-permission** 

用于临时授予应用程序访问 provider 中指定 uri 的权限！

如果 provider 的 android:grantUriPermissions 属性为 true，那么应用程序可以临时访问该 provider 的所有 Uri 数据；

如果 android:grantUriPermissions 属性为 false，那么应用程序只能临时访问通过 grant-uri-permission 标签指定的特定 uri 子集！

一个 provider 可以定义多个 grant-uri-permission，每一个 grant-uri-permission 申明一个特定 uri 数据子集！

| 属性名 | 解释  |
|--------|:----  |
| android:path | provider 的数据子集的完整 uri 路径| 
| android:pathPrefix | provider 数据子集的 uri 路径的初始部分| 
| android:pathPattern | provider 数据子集的完整 uri 路径，但可以使用通配符| 

如果设置了 android:path，临时访问权限只能被授予给 path 指定的数据子集；

如果设置了 android:pathPrefix，临时访问权限将会被授予给具有 pathPrefix 初始部分的所有数据子集；


#### 5.1.6.2 **path-permission** 

定义 provider 中特定数据子集的路径和所需权限，用于开放部分 Uri 数据的读写权限！

一个 provider 可以定义多个 path-permission，每一个 path-permission 申明一个特定 uri 数据子集！

| 属性名 | 解释  |
|--------|:----  |
| android:path | provider 数据子集的完整 uri 路径| 
| android:pathPrefix | provider 数据子集的 uri 路径的初始部分| 
| android:pathPattern | provider 数据子集的完整 uri 路径，但可以使用通配符 | 
| android:permission | 读写权限，readPermission 和 writePermission 优先级| 
| android:readPermission | 读数据的权限|
| android:writePermission | 写数据的权限|

如果设置了 android:path，临时访问权限只能被授予给 path 指定的数据子集；

如果设置了 android:pathPrefix，临时访问权限将会被授予给具有 pathPrefix 初始部分的所有数据子集；

## 5.2 权限解析

我们回到 PMS 中去，看下应用程序的 Provider 组件是如何被解析的，我们重点关注和权限相关的：

### 5.2.1 PackageParser.parseProvider
```java
    private Provider parseProvider(Package owner, Resources res,
            XmlResourceParser parser, int flags, String[] outError)
            throws XmlPullParserException, IOException {
        TypedArray sa = res.obtainAttributes(parser,
                com.android.internal.R.styleable.AndroidManifestProvider);
        ... ... ...

        //【1】创建一个 Provider 对象和 ProviderInfo 对象！
        Provider p = new Provider(mParseProviderArgs, new ProviderInfo());
        if (outError[0] != null) {
            sa.recycle();
            return null;
        }

        boolean providerExportedDefault = false;
        if (owner.applicationInfo.targetSdkVersion < Build.VERSION_CODES.JELLY_BEAN_MR1) {
            providerExportedDefault = true;
        }
        //【2】解析 android:exported 属性！
        p.info.exported = sa.getBoolean(
                com.android.internal.R.styleable.AndroidManifestProvider_exported,
                providerExportedDefault);
                
        ... ... ...

        //【3】解析 android:permission 属性！
        String permission = sa.getNonConfigurationString(
                com.android.internal.R.styleable.AndroidManifestProvider_permission, 0);
        //【4】解析 android:readPermission 属性！
        String str = sa.getNonConfigurationString(
                com.android.internal.R.styleable.AndroidManifestProvider_readPermission, 0);
        if (str == null) {
            str = permission;
        }
        if (str == null) {
            p.info.readPermission = owner.applicationInfo.permission;
        } else {
            p.info.readPermission =
                str.length() > 0 ? str.toString().intern() : null;
        }
        //【5】解析 android:writePermission 属性！
        str = sa.getNonConfigurationString(
                com.android.internal.R.styleable.AndroidManifestProvider_writePermission, 0);
        if (str == null) {
            str = permission;
        }
        if (str == null) {
            p.info.writePermission = owner.applicationInfo.permission;
        } else {
            p.info.writePermission =
                str.length() > 0 ? str.toString().intern() : null;
        }
        //【6】解析 android:grantUriPermissions 属性！
        p.info.grantUriPermissions = sa.getBoolean(
                com.android.internal.R.styleable.AndroidManifestProvider_grantUriPermissions,
                false);

        ... ... ...

        //【*5.2.2】调用 parseProviderTags 来解析子标签！
        if (!parseProviderTags(res, parser, p, outError)) {
            return null;
        }

        return p;
    }
```

这里我们看到：

- 在 API 17 以前，exported 默认是 true，因为这个属性是在 API 17 才引入的！
- readPermission 和 writePermission 的优先级高于 permission，如果没有定义


属性的处理：

- android:grantUriPermissions 属性的值会保存到 Provider.ProviderInfo.grantUriPermissions 中；
- android:exported 属性的值会保存到 Provider.ProviderInfo.exported 中；
- android:permission 属性和 android:readPermission 属性会保存到 Provider.ProviderInfo.readPermission 中；
- android:permission 属性和 android:writePermission 属性会保存到 Provider.ProviderInfo.writePermission 中；

### 5.2.2 PackageParser.parseProviderTags

在 parseProviderTags 中会对 grant-uri-permission 和 path-permission 进行解析！

```java
    private boolean parseProviderTags(Resources res,
            XmlResourceParser parser, Provider outInfo, String[] outError)
            throws XmlPullParserException, IOException {
        int outerDepth = parser.getDepth();
        int type;
        while ((type=parser.next()) != XmlPullParser.END_DOCUMENT
               && (type != XmlPullParser.END_TAG
                       || parser.getDepth() > outerDepth)) {
            if (type == XmlPullParser.END_TAG || type == XmlPullParser.TEXT) {
                continue;
            }

            if (parser.getName().equals("intent-filter")) {
                ProviderIntentInfo intent = new ProviderIntentInfo(outInfo);
                if (!parseIntent(res, parser, true, false, intent, outError)) {
                    return false;
                }
                outInfo.intents.add(intent);

            } else if (parser.getName().equals("meta-data")) {
                if ((outInfo.metaData=parseMetaData(res, parser,
                        outInfo.metaData, outError)) == null) {
                    return false;
                }
                
            //【1】解析 grant-uri-permission
            } else if (parser.getName().equals("grant-uri-permission")) {
                // 获得其所有属性！
                TypedArray sa = res.obtainAttributes(parser,
                        com.android.internal.R.styleable.AndroidManifestGrantUriPermission);
                //【1.1】这里创建了一个 PatternMatcher 对象，用于封装 grant-uri-permission 的解析内容；
                PatternMatcher pa = null;
                //【1.2】pathPattern 优先级最高，接下来是 pathPrefix，最后是 pathPattern，优先级高会覆盖优先级低：
                String str = sa.getNonConfigurationString(
                        com.android.internal.R.styleable.AndroidManifestGrantUriPermission_path, 0); // path 属性
                if (str != null) {
                    pa = new PatternMatcher(str, PatternMatcher.PATTERN_LITERAL);
                }

                str = sa.getNonConfigurationString( // pathPrefix 属性
                        com.android.internal.R.styleable.AndroidManifestGrantUriPermission_pathPrefix, 0);
                if (str != null) {
                    pa = new PatternMatcher(str, PatternMatcher.PATTERN_PREFIX);
                }

                str = sa.getNonConfigurationString(  // pathPattern 属性
                        com.android.internal.R.styleable.AndroidManifestGrantUriPermission_pathPattern, 0);
                if (str != null) {
                    pa = new PatternMatcher(str, PatternMatcher.PATTERN_SIMPLE_GLOB);
                }

                sa.recycle();

                if (pa != null) {
                    //【1.3】如果 provider 设置了以上三个属性（至少是一个），
                    // 这里可以看出 provider 是支持多个 grant-uri-permission 的！
                    if (outInfo.info.uriPermissionPatterns == null) {
                        outInfo.info.uriPermissionPatterns = new PatternMatcher[1];
                        outInfo.info.uriPermissionPatterns[0] = pa;
                    } else {
                        final int N = outInfo.info.uriPermissionPatterns.length;
                        PatternMatcher[] newp = new PatternMatcher[N+1];
                        System.arraycopy(outInfo.info.uriPermissionPatterns, 0, newp, 0, N);
                        newp[N] = pa;
                        outInfo.info.uriPermissionPatterns = newp;
                    }
                    //【1.4】由于 provider 设置了 grant-uri-permission，也将 info.grantUriPermissions 置为 true；
                    outInfo.info.grantUriPermissions = true;
                } else {
                    if (!RIGID_PARSER) {
                        Slog.w(TAG, "Unknown element under <path-permission>: "
                                + parser.getName() + " at " + mArchiveSourcePath + " "
                                + parser.getPositionDescription());
                        XmlUtils.skipCurrentTag(parser);
                        continue;
                    } else {
                        outError[0] = "No path, pathPrefix, or pathPattern for <path-permission>";
                        return false;
                    }
                }
                XmlUtils.skipCurrentTag(parser);

            //【2】解析 path-permission
            } else if (parser.getName().equals("path-permission")) {
                //【2.1】获得其所有属性！
                TypedArray sa = res.obtainAttributes(parser,
                        com.android.internal.R.styleable.AndroidManifestPathPermission);
                //【2.2】这里创建了一个 PathPermission 对象，用于封装 path-permission 的解析内容；
                PathPermission pa = null;
                //【2.3】解析 readPermission 和 writePermission！
                String permission = sa.getNonConfigurationString(
                        com.android.internal.R.styleable.AndroidManifestPathPermission_permission, 0);
                String readPermission = sa.getNonConfigurationString(
                        com.android.internal.R.styleable.AndroidManifestPathPermission_readPermission, 0);
                if (readPermission == null) {
                    readPermission = permission;
                }
                String writePermission = sa.getNonConfigurationString(
                        com.android.internal.R.styleable.AndroidManifestPathPermission_writePermission, 0);
                if (writePermission == null) {
                    writePermission = permission;
                }

                boolean havePerm = false;
                if (readPermission != null) {
                    readPermission = readPermission.intern();
                    havePerm = true;
                }
                if (writePermission != null) {
                    writePermission = writePermission.intern();
                    havePerm = true;
                }

                if (!havePerm) {
                    if (!RIGID_PARSER) {
                        Slog.w(TAG, "No readPermission or writePermssion for <path-permission>: "
                                + parser.getName() + " at " + mArchiveSourcePath + " "
                                + parser.getPositionDescription());
                        XmlUtils.skipCurrentTag(parser);
                        continue;
                    } else {
                        outError[0] = "No readPermission or writePermssion for <path-permission>";
                        return false;
                    }
                }
                //【2.4】pathPattern 优先级最高，接下来是 pathPrefix，最后是 pathPattern，优先级高覆盖优先级低！
                String path = sa.getNonConfigurationString(
                        com.android.internal.R.styleable.AndroidManifestPathPermission_path, 0);
                if (path != null) {
                    pa = new PathPermission(path,
                            PatternMatcher.PATTERN_LITERAL, readPermission, writePermission);
                }

                path = sa.getNonConfigurationString(
                        com.android.internal.R.styleable.AndroidManifestPathPermission_pathPrefix, 0);
                if (path != null) {
                    pa = new PathPermission(path,
                            PatternMatcher.PATTERN_PREFIX, readPermission, writePermission);
                }

                path = sa.getNonConfigurationString(
                        com.android.internal.R.styleable.AndroidManifestPathPermission_pathPattern, 0);
                if (path != null) {
                    pa = new PathPermission(path,
                            PatternMatcher.PATTERN_SIMPLE_GLOB, readPermission, writePermission);
                }

                sa.recycle();
        
                if (pa != null) {
                    //【2.5】如果 provider 设置了以上三个属性（至少是一个），
                    // 这里可以看出 provider 是支持多个 path-permission 的！
                    if (outInfo.info.pathPermissions == null) {
                        outInfo.info.pathPermissions = new PathPermission[1];
                        outInfo.info.pathPermissions[0] = pa;
                    } else {
                        final int N = outInfo.info.pathPermissions.length;
                        PathPermission[] newp = new PathPermission[N+1];
                        System.arraycopy(outInfo.info.pathPermissions, 0, newp, 0, N);
                        newp[N] = pa;
                        outInfo.info.pathPermissions = newp;
                    }
                } else {
                    if (!RIGID_PARSER) {
                        Slog.w(TAG, "No path, pathPrefix, or pathPattern for <path-permission>: "
                                + parser.getName() + " at " + mArchiveSourcePath + " "
                                + parser.getPositionDescription());
                        XmlUtils.skipCurrentTag(parser);
                        continue;
                    }
                    outError[0] = "No path, pathPrefix, or pathPattern for <path-permission>";
                    return false;
                }
                XmlUtils.skipCurrentTag(parser);

            } else {
                if (!RIGID_PARSER) {
                    Slog.w(TAG, "Unknown element under <provider>: "
                            + parser.getName() + " at " + mArchiveSourcePath + " "
                            + parser.getPositionDescription());
                    XmlUtils.skipCurrentTag(parser);
                    continue;
                } else {
                    outError[0] = "Bad element under <provider>: " + parser.getName();
                    return false;
                }
            }
        }
        return true;
    }
```

#### 5.2.2.1 解析 grant-uri-permission

我们可以看到，三个属性中 pathPattern 优先级最高，接下来是 pathPrefix，最后是 pathPattern，优先级高的会覆盖优先级低的属性：

```java
    //【1】path 属性；
    String str = sa.getNonConfigurationString(
            com.android.internal.R.styleable.AndroidManifestGrantUriPermission_path, 0);
    if (str != null) {
        pa = new PatternMatcher(str, PatternMatcher.PATTERN_LITERAL);
    }
    //【2】pathPrefix 属性；
    str = sa.getNonConfigurationString(
            com.android.internal.R.styleable.AndroidManifestGrantUriPermission_pathPrefix, 0);
    if (str != null) {
        pa = new PatternMatcher(str, PatternMatcher.PATTERN_PREFIX);
    }
    //【3】pathPattern 属性；
    str = sa.getNonConfigurationString(
            com.android.internal.R.styleable.AndroidManifestGrantUriPermission_pathPattern, 0);
    if (str != null) {
        pa = new PatternMatcher(str, PatternMatcher.PATTERN_SIMPLE_GLOB);
    }
```
最后会创建一个 PatternMatcher 对象！

```java
    public PatternMatcher(String pattern, int type) {
        mPattern = pattern;
        mType = type;
        if (mType == PATTERN_ADVANCED_GLOB) {
            mParsedPattern = parseAndVerifyAdvancedPattern(pattern);
        } else {
            mParsedPattern = null;
        }
    }
```

如果 provider 设置了 grant-uri-permission，也会将前面的 p.info.grantUriPermissions 置为 true，即使应用没有显式设置 android:grantUriPermissions 的值；

解析得到的 PatternMatcher 会保存到 Provider.ProviderInfo.uriPermissionPatterns 数组中；

#### 5.2.2.2 解析 patch-permission

同样的，三个属性中 pathPattern 优先级最高，接下来是 pathPrefix，最后是 pathPattern，优先级高的会覆盖优先级低的属性！

最后会创建一个 PathPermission 对象，PathPermission 是 PatternMatcher 的子类！
```java
public class PathPermission extends PatternMatcher {

    public PathPermission(String pattern, int type, String readPermission,
            String writePermission) {
        super(pattern, type);
        mReadPermission = readPermission;
        mWritePermission = writePermission;
    }
}
```
所以，大家应该明白 String pattern, int type 是如何处理的了！

解析得到的 PathPermission 会保存到 Provider.ProviderInfo.pathPermissions 数组中；


# 6 权限知识点总结

我们知道，权限的 protectionLevel 分为基本权限类型和附加的标志位：

## 6.1 基本权限类型

下表显示了所有基本权限类型：

|基本权限类型 |取值 |解释 |
|--------|:----:|:----|
| **normal**|0| 默认值，这种权限是风险较低的权限，安装时，系统会自动授予这种类型的权限给请求应用程序。| 
| **dangerous** |1| 这种权限是风险较高的权限，因此系统可能不会自动将其授予请求的应用程序，而是需要应用显式的申请。| 
| **signature** |2| 仅当请求权限的应用程序使用与声明权限的应用程序相同的证书签名时，系统才会授予该权限。如果签名匹配，系统会自动授予权限，而不需要通知用户或要求用户明确批准；如果签名不匹配，一些 signature 权限需要用户显式申请。| 
| **signatureOrSystem** |3| 该权限只会授予和声明权限的应用程序相同的证书签名的应用或者系统分区应用，避免使用此选项，因为 signature 保护级别应该足以满足大多数需求，并且可以工作，而不管应用程序的安装位置。**在 API 级别 23 中已弃用**，现在用 "signature\|privileged" 替代； “ signatureOrSystem” 许可用于某些特定情况，其中多个供应商将应用程序内置到系统映像中，并且需要明确地共享特定功能，因为它们正在构建在一起。 | 

小小的思考：

 - normal 类型的权限，无论是系统应用还是三方应用都会默认授予；
 - dangerous 类型的权限，无论是系统应用还是三方应用都需要显式申请，但是系统会有一个类会给某些系统应用默认授予这些权限；
 - 那么 signature 类型的权限的意义就很显然了，对于系统定义的 signature 权限，系统应用会默认授予；而对于三方应用，一些  signature 权限其无法申请，而另外一些  signature 权限，三方应用可以通过一定的方式申请，这些权限是一些特殊的权限！

## 6.2 附加标志位

对于 signature 类型的权限，可以配合下面的特殊的附加标志位：

|附加标志位|取值|解释 |
|--------|:----:|:----|
| **appop**| 40| 该权限与用于控制访问的 app op 密切相关，也就是我们说的 AppOpsManager。| 
| **development**| 20|此权限也**可以授予**（可选）给开发应用程序。| 
| **installer**| 100|此权限可以**自动授予**安装软件包的系统应用程序，就是我们的 PackageInstaller | 
| **instant**| 1000|此权限**可被授予**给即时应用程序（instant app） | 
| **oem**| 4000|| 
| **pre23**| 80| 此权限会自动授予低于 M（在引入运行时权限之前）的 API 级别的应用程序。| 
| **preinstalled**| 400| 此权限可以**自动授予** system 分区预先安装的任何应用程序（不仅仅是特权应用程序）。| 
| **privileged**| 10| 此权限也**可以授予**安装在 system 分区的特权应用。请避免使用此标志位，因为 signature 保护级别已足以满足大多数需求，并且无论应用程序的安装位置如何，都能正常工作。 此权限标志用于某些特殊情况，其中多个供应商将应用程序内置到系统映像中，需要明确地共享特定功能。| 
| **runtime**| 2000| 此权限**只能被授予**适用于运行时权限（M 及以上）的应用程序| 
| **setup**| 800| 此权限可以被**自动授予**给开机向导应用程序（setup wizard app）| 
| **system**| 10| 在 API 级别 23 中已弃用，用 privileged 替代 | 
| **textClassifier**| 10000	| 此权限可以被**自动授予**给系统默认的文本解析器 | 
| **vendorPrivileged**| 8000| 此权限可以被授予给供应商（vendor）分区中的特权应用程序。| 
| **verifier**| 200| 此权限可以被**自动授予**给验证软件包的系统应用程序。 | 

注：如果保护级别设置了额外的标志位，那么其基本权限类型必须是 signature！

## 6.3 权限组

任何权限都可以属于某个权限组，所有的 dangerous 权限都有所属的组，但是，权限组只有在申请的权限是 dangerous 权限是才会对用户体验有影响！

在 Android 6.0 （API level 23）以及以上，如果应用的 targetSdkVersion >= 23 ，系统是这样处理运行时权限的申请的：

- 如果应用程序没有被授予权限组中的任何运行时权限，在申请权限时，系统会向用户显示描述应用程序要访问的权限组的权限请求对话框。 但是该对话框不会描述该组中的特定权限的信息。如果用户同意授予权限，该权限将被授予。
- 如果应用程序已经被授予同一权限组中的某个权限，则对于其他权限，系统会立即授予权限，而不与用户进行任何交互。 例如：如果应用程序先前已请求并已获得READ_CONTACTS权限，然后它请求WRITE_CONTACTS，则系统会立即授予该权限，而不向用户显示权限对话框。

在 Android 5.1 （API level 22）以及以下，或者应用的 targetSdkVersion <= 22：

- 系统会在应用安装的时候授予所有的运行时权限；同时，也是只会告诉用户应用程序需要哪些权限组，而不是单个权限。


## 6.4 特殊权限

### 6.4.1 signature with appop

signature 权限中有几个权限特殊，signature 权限本身是安装时权限，但是当其设置了 appop 标志位的时候，其可以通过 appOps 动态管理。

Android7.1.1 提供了如下 signature appop 权限：

```xml
    <permission android:name="android.permission.WRITE_SETTINGS"
        android:label="@string/permlab_writeSettings"
        android:description="@string/permdesc_writeSettings"
        android:protectionLevel="signature|preinstalled|appop|pre23" />

    <permission android:name="android.permission.SYSTEM_ALERT_WINDOW"
        android:label="@string/permlab_systemAlertWindow"
        android:description="@string/permdesc_systemAlertWindow"
        android:protectionLevel="signature|preinstalled|appop|pre23|development" />

    <permission android:name="android.permission.PACKAGE_USAGE_STATS"
        android:protectionLevel="signature|privileged|development|appop" />
```
这三个权限，可以通过 AppOps 的方式来授予那些签名不匹配的应用！


### 6.4.1 signature with bind service

signature 内部还有一些权限的使用方式也很特殊，应用需要配置这个权限，但是应用自身并不是权限的申请方：
```java
BIND_ACCESSIBILITY_SERVICE
BIND_AUTOFILL_SERVICE
BIND_CARRIER_SERVICES
BIND_CHOOSER_TARGET_SERVICE
BIND_CONDITION_PROVIDER_SERVICE
BIND_DEVICE_ADMIN
BIND_DREAM_SERVICE
BIND_INCALL_SERVICE
BIND_INPUT_METHOD
BIND_MIDI_DEVICE_SERVICE
BIND_NFC_SERVICE
BIND_NOTIFICATION_LISTENER_SERVICE
BIND_PRINT_SERVICE
BIND_SCREENING_SERVICE
BIND_TELECOM_CONNECTION_SERVICE
BIND_TEXT_SERVICE
BIND_TV_INPUT
BIND_VISUAL_VOICEMAIL_SERVICE
BIND_VOICE_INTERACTION
BIND_VPN_SERVICE
BIND_VR_LISTENER_SERVICE
BIND_WALLPAPER
```
这些权限的使用后面会单独写一篇博文记录下！

