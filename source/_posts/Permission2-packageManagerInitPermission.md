# Permission第 2 篇 - PackageManager 对权限的初始化流程
title: Permission第 2 篇 - PackageManager 对权限的初始化流程
date: 2017/05/13 20:46:25
catalog: true
categories:
- AndroidFramework源码分析
- Permission权限管理
tags: Permission权限管理
---

[toc]

基于  Android7.1.1 源码，分析权限管理机制

# 0 综述

在分析 PackageManagerService 的时候，我们详细的分析了 PackageManagerService 的启动流程，包括安装信息解析，应用程序包的解析等等！

其中，涉及到了对系统中权限的解析和配置，但是由于 PackageManagerService 的启动流程过于负责，所以在 PackageManagerService 相关文章中，并没有对其进行启动的总结！

本系列文章我们详细的总结下权限相关的内容！

# 1 添加共享 uid

在 PMS 启动的开始，创建了 Settings 对象，同时添加了一些系统定义的共享 uid！

```java
        mSettings = new Settings(mPackages);
        mSettings.addSharedUserLPw("android.uid.system", Process.SYSTEM_UID,
                ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
        mSettings.addSharedUserLPw("android.uid.phone", RADIO_UID,
                ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
        mSettings.addSharedUserLPw("android.uid.log", LOG_UID,
                ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
        mSettings.addSharedUserLPw("android.uid.nfc", NFC_UID,
                ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
        mSettings.addSharedUserLPw("android.uid.bluetooth", BLUETOOTH_UID,
                ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
        mSettings.addSharedUserLPw("android.uid.shell", SHELL_UID,
                ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
```
addSharedUserLPw 方法会创建 SharedUserSetting 对象，保存到 Settings.mSharedUsers 中！

同时也会根据 uid 的取值添加到 Settings.mUserIds 或者 Settings.mOtherUserIds 中，取值判断的依据是：uid >= Process.FIRST_APPLICATION_UID ()！

- 如果是应用程序的 uid，会被添加到 mUserIds 中！
- 如果是系统服务/进程的 uid，会被添加到 mOtherUserIds 中！

因为，这里添加的 5 种共享 uid 都是系统定义的，所以会添加到 Settings.mOtherUserIds 中！

# 2 解析系统配置信息
```java
        SystemConfig systemConfig = SystemConfig.getInstance();
        mGlobalGids = systemConfig.getGlobalGids();
        mSystemPermissions = systemConfig.getSystemPermissions();
        mAvailableFeatures = systemConfig.getAvailableFeatures();
```
这个过程会解析以下目录中的文件：

```xml
/etc/sysconfig：ALLOW_ALL
/etc/permissions：ALLOW_ALL
/odm/etc/sysconfig：ALLOW_LIBS | ALLOW_FEATURES | ALLOW_APP_CONFIGS
/odm/etc/permissions：ALLOW_LIBS | ALLOW_FEATURES | ALLOW_APP_CONFIGS
/oem/etc/sysconfig：ALLOW_FEATURES
/oem/etc/permissions：ALLOW_FEATURES
```
可以看到，和权限相关的目录是：

```xml
/etc/sysconfig：ALLOW_ALL
/etc/permissions：ALLOW_ALL
```
在解析过程中 etc/permissions/platform.xml 文件是被放到了最后处理！

在这个过程中会解析以下和权限相关的属性，我们来看下 platform.xml 文件中和权限相关的内容！

## 2.1 解析 "group"

获得系统中的一些特定的 gid，这些 gid 会和系统权限相互映射：

```java
    <permission name="android.permission.BLUETOOTH_STACK" >
        <group gid="bluetooth" />
        <group gid="wakelock" />
        <group gid="uhid" />
    </permission>
```
group 标签不能单独使用，要和 permission 配合使用！

SystemConfig  会通过 android.os.Process.getGidForName 方法获得字符串 group 对应的 int 值！

getGidForName 是一个 native 方法，它的实际定义是 frameworks/base/core/jni/android_util_Process.cpp 中

```C++
jint android_os_Process_getGidForName(JNIEnv* env, jobject clazz, jstring name) {
    ......    
    const size_t N = name8.size();
    if (N > 0) { 
        const char* str = name8.string();
        for (size_t i=0; i<N; i++) {
            if (str[i] < '0' || str[i] > '9') {
                struct group* grp = getgrnam(str);
                if (grp == NULL) {
                    return -1;
                }    
                return grp->gr_gid;
            }    
        }    
        return atoi(str);
    }    
    return -1;
}
```
这里根据 groupName 调用 getgrnam 获取组信息。我们知道，getgrnam 是一个 C 库函数，在 Linux 中标准定义是根据读取 /etc/group 文件获取组信息，但在 Android 中并没有这个文件，那么这个函数是怎么实现的呢？

该函数是定义在 bionic/libc/bionic/stubs.cpp 文件中，这里我们只关注主要的代码：

```c++
#include "private/android_filesystem_config.h"

static group* android_iinfo_to_group(group* gr, const android_id_info* iinfo) {
  gr->gr_name   = (char*) iinfo->name;
  gr->gr_gid    = iinfo->aid;
  gr->gr_mem[0] = gr->gr_name;
  gr->gr_mem[1] = NULL;
  return gr;
}

static group* android_name_to_group(group* gr, const char* name) {
  for (size_t n = 0; n < android_id_count; ++n) {
    if (!strcmp(android_ids[n].name, name)) {
      return android_iinfo_to_group(gr, android_ids + n);
    }
  }
  return NULL;
}

group* getgrnam(const char* name) { // NOLINT: implementing bad function.
  stubs_state_t* state = __stubs_state();
  if (state == NULL) {
    return NULL;
  }

  if (android_name_to_group(&state->group_, name) != 0) {                         
    return &state->group_;
  }

  return app_id_to_group(app_id_from_name(name), state);
}
```
显而易见，它是遍历 android_ids 数组，查找是否有对应的组。那么 android_ids 定义在那里呢？

在 system\core\include\private\android_filesystem_config.h 文件中：

```c++
static const struct android_id_info android_ids[] = {
    { "root",          AID_ROOT, },

    { "system",        AID_SYSTEM, },

    { "radio",         AID_RADIO, },
    { "bluetooth",     AID_BLUETOOTH, },
    { "graphics",      AID_GRAPHICS, },
    { "input",         AID_INPUT, },
    { "audio",         AID_AUDIO, },
    ... ... ...
    { "everybody",     AID_EVERYBODY, },
    { "misc",          AID_MISC, },
    { "nobody",        AID_NOBODY, },
};
```
可以看到 android_ids 定义了 uid 的字符串名称和对应的 int 值的映射，AID_XXXX 用于指定其对应的 uid/gid：

```java
#define AID_ROOT             0  /* traditional unix root user */

#define AID_SYSTEM        1000  /* system server */

#define AID_RADIO         1001  /* telephony subsystem, RIL */
#define AID_BLUETOOTH     1002  /* bluetooth subsystem */
#define AID_GRAPHICS      1003  /* graphics devices */
#define AID_INPUT         1004  /* input devices */
#define AID_AUDIO         1005  /* audio devices */
... ... .. ...
```

关于 android_filesystem_config.h 中的相关内容，我们后面再分析！

SystemConfig  会将解析到的这些全局特定的 gid 保存到 SystemConfig.mGlobalGids 中！

## 2.2 解析 "permission"

获得系统权限和所属的 gid：

```xml
<permissions>
    <!--解析 permission 标签-->
    <permission name="android.permission.BLUETOOTH_ADMIN" >
        <group gid="net_bt_admin" />
    </permission>
</permissions>
```

下面是解析过程：

- 创建一个 PermissionEntry 对象，用于封装解析到的权限信息！
- 解析 perUser 属性；
- 解析获得该权限映射的 gid，依然是调用了 Process.getGidForName 方法，保存到 PermissionEntry.gids 中！

最后，将解析的数据会被保存到 Settings.mPermissions 中！


## 2.3 解析 "assign-permission"

给特定的 uid 分配指定的权限！ 

```xml
    <assign-permission name="android.permission.MODIFY_AUDIO_SETTINGS" uid="media" />
    <assign-permission name="android.permission.ACCESS_SURFACE_FLINGER" uid="media" />
    <assign-permission name="android.permission.WAKE_LOCK" uid="media" />
```
assign-permission 标签用于给那些运行在特定的 uid 下的，没有对于 package 文件的系统进程赋予更高层次的权限，比如 MediaSever 守护进程！

下面是解析过程：

- 调用 Process.getUidForName 获得字符串 uid 对应的 int 值，Process.getUidForName 方法最终还是调用的前面的 getgrnam 方法！


最后，将解析的数据会被保存到 Settings.mSystemPermissions 中！

## 2.4 数据同步

SystemConfig  在解析完成后，会将 Settings.mSystemPermissions 和 SystemConfig.mGlobalGids 的数据同步保存到 PMS 的集合中：

```java
        mGlobalGids = systemConfig.getGlobalGids();
        mSystemPermissions = systemConfig.getSystemPermissions();
```


# 3 添加配置信息中的系统权限

SystemConfig 在解析完了系统配置的权限和 gid 映射关系，同时我们也得到了一些系统的权限，这里会将这些权限添加到 mSettings.mPermissions 中：

```java
        ArrayMap<String, SystemConfig.PermissionEntry> permConfig
                = systemConfig.getPermissions();
        for (int i = 0; i< permConfig.size(); i++) {
            SystemConfig.PermissionEntry perm = permConfig.valueAt(i);
            BasePermission bp = mSettings.mPermissions.get(perm.name);
            if (bp == null) {
                bp = new BasePermission(perm.name, "android", BasePermission.TYPE_BUILTIN);
                // 将这些系统权限添加到 Settings.mPermissions 中！
                mSettings.mPermissions.put(perm.name, bp);
            }
            if (perm.gids != null) {
                bp.setGids(perm.gids, perm.perUser);
            }
        }
```

这里会创建 BasePermission 实例，指定权限定义者包名为 android，权限类型为 BasePermission.TYPE_BUILTIN！

如果该系统权限有映射特定的 gid，将其添加到 BasePermission.gids 中！



# 4 读取上一次的安装信息

接下来，Settings 会从 Packages.xml 中读取上一次安装的信息！

```xml
<packages>
    <version sdkVersion="27" databaseVersion="3" fingerprint="../release-keys" />
    <version volumeUuid="primary_physical" sdkVersion="27" databaseVersion="27"
             fingerprint=".../release-keys" />
    <permission-trees />
    <permissions>
        <item name="android.permission.REAL_GET_TASKS" package="android" protection="18" />
        <!-- ... ... ...-->
    </permissions>
    <package name="com.android.providers.calendar" codePath="/system/priv-app/CalendarProvider" 
                    nativeLibraryPath="/system/priv-app/CalendarProvider/lib" 
					primaryCpuAbi="armeabi-v7a" publicFlags="940064325" privateFlags="8" 
					ft="1624a4dcee0" it="1624a4dcee0" ut="1624a4dcee0"
					version="0" sharedUserId="10006" isOrphaned="true">
        <sigs count="1">
            <cert index="0" />
        </sigs>
        <perms>
            <item name="android.permission.WRITE_SETTINGS" granted="true" flags="0" />
			<!-- ... ... ...-->
        </perms>
        <proper-signing-keyset identifier="1" />
    </package>
    ... ... ... ..
    <shared-user name="android.uid.bluetooth" userId="1002">
        <sigs count="1">
            <cert index="0" />
        </sigs>
        <perms>
            <item name="android.permission.REAL_GET_TASKS" granted="true" flags="0" />
			<!-- ... ... -->
        </perms>
    </shared-user>
    
</packages>
```
上面这个 xml 文件显示了 packages.xml 中的文件结构，可以看到，他们会有一个解析的顺序！

这个过程会会解析获得很多数据，这里我们只关注和权限相关的数据！

## 4.1 解析 "permissions" 和 "permission-trees"

首先会解析 permission-trees 的内容，然后在解析 permissions 的内容，其保存了上一次安装时的系统和应用定义的所有的权限信息：

```xml
    <permissions>
        <item name="android.permission.REAL_GET_TASKS" package="android" protection="18" />
        <item name="android.permission.SEND_RECEIVE_STK_INTENT" package="com.android.stk" protection="2" />
        <item name="android.permission.ACCESS_CACHE_FILESYSTEM" package="android" protection="18" />
        <item name="android.permission.REMOTE_AUDIO_PLAYBACK" package="android" protection="2" />
        <item name="android.permission.DOWNLOAD_WITHOUT_NOTIFICATION" package="com.android.providers.downloads" />
        <item name="android.permission.REGISTER_WINDOW_MANAGER_LISTENERS" package="android" protection="2" />
        ... ... ...
    </permissions>
```
permissions 标签中保存的是非动态权限信息，permission-trees 中保存的是动态权限信息。

对于非动态权限和动态权限树，会有不同的处理：

- 非动态权限信息最终都会被解析保存到 Setting.mPermissions 中了！！

- 动态权限信息会被保存在 Setting.mPermissionTrees 中了！！

</br>

整个解析过程如下：

- 解析获得权限名 name，权限的定义包名 package，权限类型 type，以及权限的保护级别 protection 并对其修正；
    - 修正就是为了保证附加标志位只能和基本权限类型 signiture 一起使用；  

</br>

- 尝试从 Setttings 的权限集合中获得对应的权限对象 BasePermission 
    - 如果为 null 就会创建新的 BasePermission 对象；
    - 如果已有对应的 BasePermission 对象，但是其 type 不是 BasePermission.TYPE_BUILTIN，也会创建新的 BasePermission 对象 ！

</br>

- 判断权限是不是动态权限：
    - 权限类型 type 是否是 dynamic，如果是，权限类型为 BasePermission.TYPE_DYNAMIC，不是就是 BasePermission.TYPE_NORMAL；
    - 如果是非动态权限，就直接添加到 Setting.mPermissions 中；
    - 如果是动态权限，会在创建一个 PermissionInfo 对象，封装所属权限树的信息，再添加到 Setting.mPermissionTrees 中；


## 4.2 解析 "package"

接着是解析 package 信息，主要是调用 readPackageLPw 方法！

### 4.2.1 Settings.readPackageLPw

对于 readPackageLPw 方法，我们在 PMS 的启动流程中有详细分析，这里我们只关注和权限相关的代码！

```java
    private void readPackageLPw(XmlPullParser parser) throws XmlPullParserException, IOException {
        ... ... ... ...
        try {
            
            //【1】获得应用的 name！
            name = parser.getAttributeValue(null, ATTR_NAME);
            realName = parser.getAttributeValue(null, "realName");
            
            //【2】获得 userId 和 sharedId 的名称, userId 和 sharedIdStr 不能同时存在！
            idStr = parser.getAttributeValue(null, "userId");
            uidError = parser.getAttributeValue(null, "uidError");
            sharedIdStr = parser.getAttributeValue(null, "sharedUserId");
            
            ... ... ...
                        
            //【3】获得 userId 的 int 值，如果 AndroidManifest.xml 有设置 android:sharedUserId 属性，
            // 那么应用的 userId 就为 0！！
            int userId = idStr != null ? Integer.parseInt(idStr) : 0;
            if (resourcePathStr == null) {
                resourcePathStr = codePathStr;
            }
            if (realName != null) {
                realName = realName.intern();
            }
            if (name == null) {
                PackageManagerService.reportSettingsProblem(Log.WARN,
                        "Error in package manager settings: <package> has no name at "
                                + parser.getPositionDescription());
            } else if (codePathStr == null) {
                PackageManagerService.reportSettingsProblem(Log.WARN,
                        "Error in package manager settings: <package> has no codePath at "
                                + parser.getPositionDescription());
            } else if (userId > 0) {
                //【3.1】如果 userId 大于0，说明 package 是独立用户 id
                // 调用 addPackageLPw 方法保存这个有独立的 Linux 用户 ID 的 Package!
                packageSetting = addPackageLPw(name.intern(), realName, new File(codePathStr),
                        new File(resourcePathStr), legacyNativeLibraryPathStr, primaryCpuAbiString,
                        secondaryCpuAbiString, cpuAbiOverrideString, userId, versionCode, pkgFlags,
                        pkgPrivateFlags, parentPackageName, null);

                if (PackageManagerService.DEBUG_SETTINGS)
                    Log.i(PackageManagerService.TAG, "Reading package " + name + ": userId="
                            + userId + " pkg=" + packageSetting);
                if (packageSetting == null) {
                    PackageManagerService.reportSettingsProblem(Log.ERROR, "Failure adding uid "
                            + userId + " while parsing settings at "
                            + parser.getPositionDescription());
                } else {
                    // 设置时间戳，第一次安装时间，最近更新时间
                    packageSetting.setTimeStamp(timeStamp);
                    packageSetting.firstInstallTime = firstInstallTime;
                    packageSetting.lastUpdateTime = lastUpdateTime;
                }
            } else if (sharedIdStr != null) {
                //【3.2】sharedIdStr 不为 null，说明 package 设置了共享用户 id！
                userId = sharedIdStr != null ? Integer.parseInt(sharedIdStr) : 0;
                if (userId > 0) {
                    // 对于共享用户 ID 这种情况，还需要验证其有效性!
                    // 创建一个 PendingPackage 对象，来封装这个有共享用户 ID 的 package 的信息!
                    packageSetting = new PendingPackage(name.intern(), realName, new File(
                            codePathStr), new File(resourcePathStr), legacyNativeLibraryPathStr,
                            primaryCpuAbiString, secondaryCpuAbiString, cpuAbiOverrideString,
                            userId, versionCode, pkgFlags, pkgPrivateFlags, parentPackageName,
                            null);
                       
                    // 设置时间戳，第一次安装时间，最近更新时间！     
                    packageSetting.setTimeStamp(timeStamp);
                    packageSetting.firstInstallTime = firstInstallTime;
                    packageSetting.lastUpdateTime = lastUpdateTime;
                    
                    //【3.2.1】添加到 mPendingPackages 中，因为后续需要确定 shareUserId 的有效性！
                    mPendingPackages.add((PendingPackage) packageSetting);
                    
                    if (PackageManagerService.DEBUG_SETTINGS)
                        Log.i(PackageManagerService.TAG, "Reading package " + name
                                + ": sharedUserId=" + userId + " pkg=" + packageSetting);
                } else {
                    PackageManagerService.reportSettingsProblem(Log.WARN,
                            "Error in package manager settings: package " + name
                                    + " has bad sharedId " + sharedIdStr + " at "
                                    + parser.getPositionDescription());
                }
            } else {
                PackageManagerService.reportSettingsProblem(Log.WARN,
                        "Error in package manager settings: package " + name + " has bad userId "
                                + idStr + " at " + parser.getPositionDescription());
            }
        } catch (NumberFormatException e) {
            PackageManagerService.reportSettingsProblem(Log.WARN,
                    "Error in package manager settings: package " + name + " has bad userId "
                            + idStr + " at " + parser.getPositionDescription());
        }
        
        //【4】接下解析和设置其他属性。
        if (packageSetting != null) {
            ... ... ...
            int outerDepth = parser.getDepth();
            int type;
            while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
                    && (type != XmlPullParser.END_TAG || parser.getDepth() > outerDepth)) {
                if (type == XmlPullParser.END_TAG || type == XmlPullParser.TEXT) {
                    continue;
                }

                String tagName = parser.getName();
                // Legacy
                if (tagName.equals(TAG_DISABLED_COMPONENTS)) {
                    readDisabledComponentsLPw(packageSetting, parser, 0);
                    
                } else if (tagName.equals(TAG_ENABLED_COMPONENTS)) {
                    readEnabledComponentsLPw(packageSetting, parser, 0);
                    
                } else if (tagName.equals("sigs")) { // 解析 "sigs"
                    packageSetting.signatures.readXml(parser, mPastSignatures);
                    
                } else if (tagName.equals(TAG_PERMISSIONS)) { 
                    //【4.2】解析 "perms" 
                    readInstallPermissionsLPr(parser, packageSetting.getPermissionsState());
                    packageSetting.installPermissionsFixed = true;
                    
                } else if (tagName.equals("proper-signing-keyset")) {
                ... ... ...
                }
            }
        } else {
            XmlUtils.skipCurrentTag(parser);
        }
    }
        
```
流程总结：

- 对于分配了独立的 uid 的 package，创建 PackageSetting 对象，添加到 mPackage 中，并且将自身和 uid 的引用关心保存到 mUsersId 或者 mOthersId 中；
- 对于分配了共享的 uid 的 package，创建 PendingPackage 对象，添加到 mPendingPackages 中，后续会对其共享 uid 的有效性进行校验！


在解析过程中，会判断 package 是否是共享 uid 的！

```java
    idStr = parser.getAttributeValue(null, "userId");
    uidError = parser.getAttributeValue(null, "uidError");
    sharedIdStr = parser.getAttributeValue(null, "sharedUserId");
```
userId 和 sharedUserId 只能存在一个，如果 package 有 sharedUserId，那么 userId 会被设置为 0；

- 如果 package 是独立 userId ，那么会立刻创建对应的 PackageSettings 对象！然后会调用 addPackageLPw 方法，将其添加到 Settings.mPackages 中！
   - 同时也会调用 addUserIdLPw 方法，将该 PackageSettings 添加到 Settings.mUserIds/Settings.mOtherUserIds 中，取决于 uid 的范围！

</br>

- 如果 package 是共享 userId，这里会临时创建一个 PendingPackage 对象，添加到 Settings.mPendingPackages 中，因为共享 userId 的有效性没有确定！

### 4.2.1 解析 "perms" 子标签 - 安装时权限

对于上一次安装信息，package 的 perms 子标签保存了该 package 申请的安装时权限，注意是**安装时权限**！

```xml
   <perms>
        <item name="android.permission.REAL_GET_TASKS" granted="true" flags="0" />
        <item name="android.permission.RECEIVE_BOOT_COMPLETED" granted="true" flags="0" />
        <item name="android.permission.INTERNET" granted="true" flags="0" />
        <item name="android.permission.ACCESS_NETWORK_STATE" granted="true" flags="0" />
   </perms>
```
运行时权限在另外的地方保存，我们后面再看！

接着是解析 package 的安装时权限信息，主要是调用 readInstallPermissionsLPr 方法！

#### 4.2.1.1 Settings.readInstallPermissionsLPr

```java
    void readInstallPermissionsLPr(XmlPullParser parser, PermissionsState permissionsState) 
                                                        throws IOException, XmlPullParserException {
        int outerDepth = parser.getDepth();
        int type;
        while ((type=parser.next()) != XmlPullParser.END_DOCUMENT
                && (type != XmlPullParser.END_TAG
                || parser.getDepth() > outerDepth)) {
            if (type == XmlPullParser.END_TAG
                    || type == XmlPullParser.TEXT) {
                continue;
            }
            String tagName = parser.getName();
            if (tagName.equals(TAG_ITEM)) { // 解析 "item"
                String name = parser.getAttributeValue(null, ATTR_NAME); // 接续 "name"，权限名称！
                //【1】从之前解析的 Settings.mPermissions 获得对应的权限，如果没有，就跳过；
                BasePermission bp = mPermissions.get(name);
                if (bp == null) {
                    Slog.w(PackageManagerService.TAG, "Unknown permission: " + name);
                    XmlUtils.skipCurrentTag(parser);
                    continue;
                }
                //【2】解析 "granted" 标签，获得授予情况！
                String grantedStr = parser.getAttributeValue(null, ATTR_GRANTED);
                final boolean granted = grantedStr == null
                        || Boolean.parseBoolean(grantedStr);
                // 解析 "flags"标签
                String flagsStr = parser.getAttributeValue(null, ATTR_FLAGS);
                final int flags = (flagsStr != null)
                        ? Integer.parseInt(flagsStr, 16) : 0;

                //【3】处理权限授予的操作！
                if (granted) { 
                    //【9.1】处理默认授予的情况！
                    if (permissionsState.grantInstallPermission(bp) ==
                            PermissionsState.PERMISSION_OPERATION_FAILURE) {
                        Slog.w(PackageManagerService.TAG, "Permission already added: " + name);
                        XmlUtils.skipCurrentTag(parser);
                    } else {
                        permissionsState.updatePermissionFlags(bp, UserHandle.USER_ALL,
                                PackageManager.MASK_PERMISSION_FLAGS, flags);
                    }
                } else { 
                    //【9.2】处理默认不授予的情况！
                    if (permissionsState.revokeInstallPermission(bp) ==
                            PermissionsState.PERMISSION_OPERATION_FAILURE) {
                        Slog.w(PackageManagerService.TAG, "Permission already added: " + name);
                        XmlUtils.skipCurrentTag(parser);
                    } else {
                        //【9.3】更新应用权限信息！
                        permissionsState.updatePermissionFlags(bp, UserHandle.USER_ALL,
                                PackageManager.MASK_PERMISSION_FLAGS, flags);
                    }
                }
            } else {
                Slog.w(PackageManagerService.TAG, "Unknown element under <permissions>: "
                        + parser.getName());
                XmlUtils.skipCurrentTag(parser);
            }
        }
    }
```

这里解析到的权限授予信息会保存到 PermissionsState 对象中！

- 首先会判断 Settings.mPermissions 中是否有对应的权限，如果没有，就会忽略掉该安装时权限的授予！
- 解析授予状态 granted，值为 true，因为是安装时权限，解析权限的授予标志位信息 flags；
- 接着处理安装时权限的授予情况！

</bar>

PermissionsState permissionsState 用于保存该 package 的权限状态信息，通过 packageSetting.getPermissionsState() 方法获得，如果应用是独立的 uid，那么该方法返回的是自身的 PermissionsState，如果是共享 uid，返回的是 ShareUserId 的 PermissionsState！

```java
    public PermissionsState getPermissionsState() {
        return (sharedUser != null)
                ? sharedUser.getPermissionsState()
                : super.getPermissionsState();
    }
```

PermissionsState 内部有一个 mPermissions 哈希表，key 为权限 name，value 为 PermissionData，PermissionData 用于保存权限的授予情况；

```java
    private ArrayMap<String, PermissionData> mPermissions;
```

接着是处理安装时权限的授予情况：

##### 4.2.1.1.1 PermissionsS.grantInstallPermission

permissionsState.grantInstallPermission 方法用于授予安装时权限：

```java
    private int grantPermission(BasePermission permission, int userId) {
        //【9.1.1】如果之前有权限，本次授予失败！
        if (hasPermission(permission.name, userId)) {
            return PERMISSION_OPERATION_FAILURE;
        }
        //【9.1.2】判断权限 BasePermission 在 userId 下是否有映射 gid，如果有，那 hasGids 为 true！
        final boolean hasGids = !ArrayUtils.isEmpty(permission.computeGids(userId));
        //【9.1.3】如果权限 BasePermission 有映射的 gid，那就计算下该 package 的所属的 Gid 组！
        final int[] oldGids = hasGids ? computeGids(userId) : NO_GIDS;
        //【9.1.4】创建权限的 permissionData 对象，并添加到 PermissionState.mPermissions 中！
        PermissionData permissionData = ensurePermissionData(permission);
        //【9.1.5】授予权限！
        if (!permissionData.grant(userId)) { // 授权失败，就返回 PERMISSION_OPERATION_FAILURE！
            return PERMISSION_OPERATION_FAILURE;
        }
        //【2】如果权限有映射的 gid，在授予权限后，再次计算该 package 的所有映射的 gid！
        // 比较授权前后，gid 组是否发生了变化，如果有，返回 PERMISSION_OPERATION_SUCCESS_GIDS_CHANGED！
        if (hasGids) {
            final int[] newGids = computeGids(userId);
            if (oldGids.length != newGids.length) {
                return PERMISSION_OPERATION_SUCCESS_GIDS_CHANGED;
            }
        }
        return PERMISSION_OPERATION_SUCCESS; // 返回 PERMISSION_OPERATION_SUCCESS！
    }
``` 
因为是安装时权限，所以传入的 userId 为 UserHandle.USER_ALL！

hasPermission 方法用于判断该权限是否已经授予权限，如果已经授予，那就返回 **PERMISSION_OPERATION_FAILURE**！

hasGids 用于判断当前要处理的安装时权限是否有映射的 gid，如果有那个 hasGids 为 true！

如果 hasGids 为 true，那就要计算下该 package 当前所持有的所有权限的映射的 gid 组！

接着，创建权限的 permissionData 对象，并添加到 PermissionState.mPermissions 中，key 为权限名！

permissionData.grant 方法用于设置权限为授予状态：

PermissionData 内部有一个 SparseArray，保存了该权限在不用 userId 下的授予状态，可以知道，对于安装时权限，由于 userid 为 UserHandle.USER_ALL，所以 mUserStates 只有一条数据：

```java
private SparseArray<PermissionState> mUserStates = new SparseArray<>();
```

如果授予失败，比如该权限已经授予等等，那么会返回 **PERMISSION_OPERATION_FAILURE**！！

如果 hasGids 为 true，那么在授予后，再次计算该 package 当前持有的权限映射的所有 gid，如果 gid 发生了变化，那么返回 **PERMISSION_OPERATION_SUCCESS_GIDS_CHANGED**！

如果 gid 没有发生变化，那么返回 **PERMISSION_OPERATION_SUCCESS**，那么会有发生变化的情况吗？是有的，比如应用程序定义的权限是没有映射 gid 的！

如果返回的是 PERMISSION_OPERATION_SUCCESS 或者 PERMISSION_OPERATION_SUCCESS_GIDS_CHANGED ，那么接下来就会保存标志位信息！


##### 4.2.1.1.2 PermissionsS.revokeInstallPermission

permissionsState.revokeInstallPermission 方法用于安装时撤销权限！

```java
    private int revokePermission(BasePermission permission, int userId) {
        //【1】如果没有授予该权限，那就返回 PERMISSION_OPERATION_FAILURE！
        if (!hasPermission(permission.name, userId)) {
            return PERMISSION_OPERATION_FAILURE;
        }

        //【2】同样，计算该权限是否有映射 gid，如果有 gid，计算当前 package 持有的权限映射的所有 gid！
        // 保存到 oldGids 中！
        final boolean hasGids = !ArrayUtils.isEmpty(permission.computeGids(userId));
        final int[] oldGids = hasGids ? computeGids(userId) : NO_GIDS;

        PermissionData permissionData = mPermissions.get(permission.name);

        //【9.2.1】取消权限的授予！
        if (!permissionData.revoke(userId)) {
            return PERMISSION_OPERATION_FAILURE;
        }

        //【9.2.2】如果该权限在所有设备用户下均没有授予该 package，即 mUserStates.size() <= 0；
        // 那么我们要移除该权限！
        if (permissionData.isDefault()) {
            ensureNoPermissionData(permission.name);
        }

        //【3】这里是比较，在撤销安装时权限后，该 package 所持有的权限映射的 gid 是否发生变化！
        // 如果有，返回 PERMISSION_OPERATION_SUCCESS_GIDS_CHANGED！
        if (hasGids) {
            final int[] newGids = computeGids(userId);
            if (oldGids.length != newGids.length) {
                return PERMISSION_OPERATION_SUCCESS_GIDS_CHANGED;
            }
        }

        return PERMISSION_OPERATION_SUCCESS;
    }
```
整个过程和授予很类似，这里不多说！


##### 4.2.1.1.3 PermissionsS.updatePermissionFlags

如果返回的是 PERMISSION_OPERATION_SUCCESS 或者 PERMISSION_OPERATION_SUCCESS_GIDS_CHANGED ，那么接下来就会保存标志位信息！

参数传递：

- userId 传入的是 UserHandle.USER_ALL；
- flagMask 传入的是 PackageManager.MASK_PERMISSION_FLAGS，取值为 0xFF，转为二进制就是 11111
- flagValues 则是解析上次安装信息时，解析该 package 的 perms 标签时，每一个权限的 flags！

```java
    public boolean updatePermissionFlags(BasePermission permission, int userId,
            int flagMask, int flagValues) {
        enforceValidUserId(userId);
  
        final boolean mayChangeFlags = flagValues != 0 || flagMask != 0;

        //【1】如果 mPermissions 为 null，且需要更新 flags，那么会初始化 mPermissions，将 permission 添加进入！
        if (mPermissions == null) {
            if (!mayChangeFlags) {
                return false;
            }
            ensurePermissionData(permission);
        }
        //【2】如果该权限 permission 没有对应的 PermissionData，且需要更新 flags，那么我们会创建 PermissionData 添加！
        PermissionData permissionData = mPermissions.get(permission.name);
        if (permissionData == null) {
            if (!mayChangeFlags) {
                return false;
            }
            permissionData = ensurePermissionData(permission);
        }

        //【3】获得该权限的旧的 flags，用于对比！
        final int oldFlags = permissionData.getFlags(userId);
 
        //【9.3.1】更新权限标志位！
        final boolean updated = permissionData.updateFlags(userId, flagMask, flagValues);

        //【4】如果权限的状态发生了变化，那就判断是否需要重新申请！
        if (updated) {
            final int newFlags = permissionData.getFlags(userId);
            if ((oldFlags & PackageManager.FLAG_PERMISSION_REVIEW_REQUIRED) == 0
                    && (newFlags & PackageManager.FLAG_PERMISSION_REVIEW_REQUIRED) != 0) {
                if (mPermissionReviewRequired == null) {
                    mPermissionReviewRequired = new SparseBooleanArray();
                }
                //【4.1】如果在该 userId 下需要重新申请，那么就将其添加到 mPermissionReviewRequired 中！
                mPermissionReviewRequired.put(userId, true);
            } else if ((oldFlags & PackageManager.FLAG_PERMISSION_REVIEW_REQUIRED) != 0
                    && (newFlags & PackageManager.FLAG_PERMISSION_REVIEW_REQUIRED) == 0) {
                //【4.2】如果在该 userId 下不需要重新申请，那么就将其从 mPermissionReviewRequired 中移除！
                if (mPermissionReviewRequired != null) {
                    mPermissionReviewRequired.delete(userId);
                    if (mPermissionReviewRequired.size() <= 0) {
                        mPermissionReviewRequired = null;
                    }
                }
            }
        }
        return updated;
    }
```
对于 mayChangeFlags，由于我们 flagMask 传入的是 PackageManager.MASK_PERMISSION_FLAGS，那就意味了需要更新 flags！

可以看到，如果 mayChangeFlags 为 true 的话，即使没有权限对应的 PermissionData 也会创建，因为这次会更新 flags！

PermissionData.getFlags 方法会获得权限旧的 flags！

重点的操作在 PermissionData.updateFlag 方法中：

```java
        public boolean updateFlags(int userId, int flagMask, int flagValues) {
            if (isInstallPermission()) {
                // 如果是安装时权限，那么 userId 为 UserHandle.USER_ALL;
                userId = UserHandle.USER_ALL;
            }
            if (!isCompatibleUserId(userId)) {
                // 如果 userId 不兼容，那么会直接返回！
                return false;
            }
            //【1】这里进行 & 操作，保留了 flagValues 和 flagMask 相同的位！
            final int newFlags = flagValues & flagMask;
            //【2】获得该 userId 下的权限状态值！
            PermissionState userState = mUserStates.get(userId);
            if (userState != null) {
                final int oldFlags = userState.mFlags;
                //【2.1】取消旧的 flags，设置新的 flags！
                userState.mFlags = (userState.mFlags & ~flagMask) | newFlags;
                if (userState.isDefault()) {
                    mUserStates.remove(userId);
                }
                //【2.2】判断 flags 是否发生变化，如果发生了变化，那就返回 true！
                return userState.mFlags != oldFlags;
            } else if (newFlags != 0) {
                //【2.3】如果该权限在该 userId 下没有状态信息，那就创建状态信息，直接设置新的 flags，返回 true！
                userState = new PermissionState(mPerm.name);
                userState.mFlags = newFlags;
                mUserStates.put(userId, userState);
                return true;
            }
            return false;
        }
```
这里有一个操作：

- 首先，进行 & 操作，保留了 flagValues 和 flagMask 相同的位，为 newFlags；
- 接着，userState.mFlags 表示旧的 flags，这里在 userState.mFlags 和 ~flagMask 中做了 & 操作，等价于取了 userState.mFlags 和 flagMask 不同的位，然后加上 newFlags，作为新的 flags！

结合前面的参数：

flagMask 传入的是 PackageManager.MASK_PERMISSION_FLAGS 每一个位都是 1，userState.mFlags & ~flagMask 操作清空了旧的 flags；flagValues & flagMask 计算得到的标志位 newFlags 就是 flagValues！

其实本质上就是设置标志位为 xml 中解析的值！


## 4.3 解析 "shared-user"

共享用户相关数据如下：
```xml
    <shared-user name="android.media" userId="10014">
        <sigs count="1">
            <cert index="3" />
        </sigs>
        <perms>
            <item name="android.permission.WRITE_SETTINGS" granted="true" flags="0" />
            <item name="android.permission.RECEIVE_BOOT_COMPLETED" granted="true" flags="0" />
            <item name="android.permission.WRITE_MEDIA_STORAGE" granted="true" flags="0" />
            ... ... 
        </perms>
    </shared-user>
```

接下来是解析共享 uid，以及共享 uid 被授予的安装时权限！

### 4.3.1 Settings.readSharedUserLPw

解析系统定义的共享用户 id：
```java
    private void readSharedUserLPw(XmlPullParser parser) throws XmlPullParserException,IOException {
        String name = null;
        String idStr = null;
        int pkgFlags = 0;
        int pkgPrivateFlags = 0;
        SharedUserSetting su = null;

        try {
            name = parser.getAttributeValue(null, ATTR_NAME); // 获得共享用户的名称
            idStr = parser.getAttributeValue(null, "userId"); // 获得共享用户的id
            int userId = idStr != null ? Integer.parseInt(idStr) : 0;

            if ("true".equals(parser.getAttributeValue(null, "system"))) { // 是否是系统的用户 id！
                pkgFlags |= ApplicationInfo.FLAG_SYSTEM;
            }

            if (name == null) {
                ... ... ...
            } else if (userId == 0) {
                ... ... ...
            } else {
                //【4.3.2】调用 addSharedUserLPw 方法，将这个共享用户和对应的 uid 保存下来！
                if ((su = addSharedUserLPw(name.intern(), userId, pkgFlags, pkgPrivateFlags))
                        == null) {
                    PackageManagerService
                            .reportSettingsProblem(Log.ERROR, "Occurred while parsing settings at "
                                    + parser.getPositionDescription());
                }
            }
        } catch (NumberFormatException e) {
            PackageManagerService.reportSettingsProblem(Log.WARN,
                    "Error in package manager settings: package " + name + " has bad userId "
                            + idStr + " at " + parser.getPositionDescription());
        }

        if (su != null) { // 解析子标签！
            int outerDepth = parser.getDepth();
            int type;

            while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
                    && (type != XmlPullParser.END_TAG || parser.getDepth() > outerDepth)) {
                if (type == XmlPullParser.END_TAG || type == XmlPullParser.TEXT) {
                    continue;
                }

                String tagName = parser.getName();
                if (tagName.equals("sigs")) {
                    su.signatures.readXml(parser, mPastSignatures);
                    
                } else if (tagName.equals("perms")) { // 解析 "perms" 标签
                    //【4.3.3】解析共享用户的权限信息！
                    readInstallPermissionsLPr(parser, su.getPermissionsState());
                    
                } else {
                    PackageManagerService.reportSettingsProblem(Log.WARN,
                            "Unknown element under <shared-user>: " + parser.getName());
                    XmlUtils.skipCurrentTag(parser);

                }
            }
        } else {
            XmlUtils.skipCurrentTag(parser);
        }
    }
```
可以看到，整个过程和解析 package 类似！

addSharedUserLPw 会创建对应的 SharedUserSetting 对象，在解析共享 uid 的安装时权限时，会通过 SharedUserSetting.getPermissionsState() 获得权限管理状态对象！

### 4.3.2 Settings.addSharedUserLPw

addSharedUserLPw 会创建对应的 SharedUserSetting 对象！
```java
    SharedUserSetting addSharedUserLPw(String name, int uid, int pkgFlags, int pkgPrivateFlags) {
        //【1】创建共享用户对应的 SharedUserSetting 对象！
        SharedUserSetting s = mSharedUsers.get(name);
        if (s != null) {
            if (s.userId == uid) {
                return s;
            }
            PackageManagerService.reportSettingsProblem(Log.ERROR,
                    "Adding duplicate shared user, keeping first: " + name);
            return null;
        }
        s = new SharedUserSetting(name, pkgFlags, pkgPrivateFlags);
        s.userId = uid;
        
        //【8.1.1.2.1】根据 uid 的范围，保存到 mUsersId 和 mOthersId 中！
        if (addUserIdLPw(uid, s, name)) {
            // 将其添加到 mSharedUsers 集合中！
            mSharedUsers.put(name, s);
            return s;
        }
        return null;
    }

```
流程总结：

- 解析 "shared-user" ，将其分装成 SharedUserSetting 对象，保存到 Settings.mSharedUsers，并根据 uid 的取值将其保存到 mUserIds 或者 mOtherIds 中！
- 解析 "shared-user" 子标签 "perm" 等等，解析共享用户的权限，每个权限对应一个 PermissionData 对象，保存进入 PermisssionState 中，用于管理共享用户的权限！

# 5 确定共享 uid 有效性

主要逻辑如下，我们知道 mPendingPackages 中包含了 uid 为共享 uid 的 PackageSettings
```java
        //【1】对 mPendingPackages 集合中的需要验证共享用户 id 有效性的 package，进行共享用户 id 有效性的验证!
        final int N = mPendingPackages.size();
        for (int i = 0; i < N; i++) {
            final PendingPackage pp = mPendingPackages.get(i);

            //【8.2.1】看 sharedId 是否能对应找到一个 ShardUserSetting 对象!
            Object idObj = getUserIdLPr(pp.sharedId);
            if (idObj != null && idObj instanceof SharedUserSetting) { // 能找到，说明共享用户 ID 是有效的!
            
                //【8.2.2】创建 PackageSetting！
                PackageSetting p = getPackageLPw(pp.name, null, pp.realName,
                        (SharedUserSetting) idObj, pp.codePath, pp.resourcePath,
                        pp.legacyNativeLibraryPathString, pp.primaryCpuAbiString,
                        pp.secondaryCpuAbiString, pp.versionCode, pp.pkgFlags, pp.pkgPrivateFlags,
                        null, true /* add */, false /* allowInstall */, pp.parentPackageName,
                        pp.childPackageNames);

                if (p == null) {
                    PackageManagerService.reportSettingsProblem(Log.WARN,
                            "Unable to create application package for " + pp.name);
                    continue;
                }
                
                // 将 PendingPackage 的其他数据拷贝到 PackageSetting 中！
                p.copyFrom(pp); 

            } else if (idObj != null) {
                String msg = "Bad package setting: package " + pp.name + " has shared uid "
                        + pp.sharedId + " that is not a shared uid\n";
                mReadMessages.append(msg);
                PackageManagerService.reportSettingsProblem(Log.ERROR, msg);

            } else {
                String msg = "Bad package setting: package " + pp.name + " has shared uid "
                        + pp.sharedId + " that is not defined\n";
                mReadMessages.append(msg);
                PackageManagerService.reportSettingsProblem(Log.ERROR, msg);

            }
        }
        mPendingPackages.clear(); // 清空 mPendingPackages
```
过程很简单，就是从前面解析得到的 Settings.mSharedUsers 中尝试获得 package 的共享 uid 对应的 SharedUserSetting 对象，如果存在，那么就说明共享 uid 是有效的！

getPackageLPw 方法会为那些在 mPendingPackages 中的 PendingPackage 创建对应的 PackageSetting，同时设置他们的共享 uid 为 SharedUserSetting，然后会将其添加到 Settings.mPackages 中！

这里调用 getUserIdLPr 来从 mUserIds 或 mOtherUserIds 中获得一个共享用户对象 SharedUserSetting！


# 6 读取上一次运行时权限信息

```java
        //【8.4】读取每个 user 下的运行时权限信息！
        for (UserInfo user : users) {
            mRuntimePermissionsPersistence.readStateForUserSyncLPr(user.id);
        }
```

接下来是读取运行时权限的信息！

```xml
<?xml version='1.0' encoding='UTF-8' standalone='yes' ?>
<runtime-permissions fingerprint="OPPO/R11s/R11s:8.1.0/OPM1.171019.011/1519198279:user/release-keys">
  <pkg name="com.sohu.inputmethod.sogouoem">
    <item name="android.permission.ACCESS_FINE_LOCATION" granted="true" flags="20" />
    <item name="android.permission.READ_EXTERNAL_STORAGE" granted="true" flags="20" />
    <item name="android.permission.ACCESS_COARSE_LOCATION" granted="true" flags="20" />
    ... ... ...
  </pkg>
  <shared-user name="android.uid.calendar">
    <item name="android.permission.READ_CALENDAR" granted="true" flags="20" />
    <item name="android.permission.ACCESS_FINE_LOCATION" granted="true" flags="20" />
    <item name="android.permission.READ_EXTERNAL_STORAGE" granted="true" flags="20" />
    ... ... ...
</runtime-permissions>
```

## 6.1 RuntimePermissionsPersistence.readStateForUserSyncLPr

该方法会读取指定的 userId 下的运行时权限信息，因为运行时权限每个设备用户下都不一样！

运行时权限记录在 /data/system/users/0/runtime-permissions.xml 文件中！

```java
        public void readStateForUserSyncLPr(int userId) {
            //【1】获得运行时权限文件对象 runtime-permissions.xml！
            File permissionsFile = getUserRuntimePermissionsFile(userId);
            if (!permissionsFile.exists()) {
                return;
            }

            FileInputStream in;
            try {
                in = new AtomicFile(permissionsFile).openRead();
            } catch (FileNotFoundException fnfe) {
                Slog.i(PackageManagerService.TAG, "No permissions state");
                return;
            }

            try {
                XmlPullParser parser = Xml.newPullParser();
                parser.setInput(in, null);
                //【6.2】解析运行时权限数据！
                parseRuntimePermissionsLPr(parser, userId);

            } catch (XmlPullParserException | IOException e) {
                throw new IllegalStateException("Failed parsing permissions file: "
                        + permissionsFile , e);
            } finally {
                IoUtils.closeQuietly(in);
            }
        }
```
解析过程在 parseRuntimePermissionsLPr 方法中！

## 6.2 RuntimePermissionsPersistence.parseRuntimePermissionsLPr

用于解析运行时权限文件，获得上一次安装时的运行时权限相关的信息！
```java
        private void parseRuntimePermissionsLPr(XmlPullParser parser, int userId)
                throws IOException, XmlPullParserException {
            final int outerDepth = parser.getDepth();
            int type;
            while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
                    && (type != XmlPullParser.END_TAG || parser.getDepth() > outerDepth)) {
                if (type == XmlPullParser.END_TAG || type == XmlPullParser.TEXT) {
                    continue;
                }

                switch (parser.getName()) {
                    case TAG_RUNTIME_PERMISSIONS: { //【1】解析 runtime-permissions 标签
                        // 解析 fingerprint 属性，保存到 mFingerprints 中！
                        String fingerprint = parser.getAttributeValue(null, ATTR_FINGERPRINT); 
                        mFingerprints.put(userId, fingerprint); 

                        // 然用用本次启动系统的 fingerprint 判断系统是否有升级，如果没有升级 defaultsGranted 为 true！
                        // 后面默认授予运行时权限会用到！
                        final boolean defaultsGranted = Build.FINGERPRINT.equals(fingerprint);
                        mDefaultPermissionsGranted.put(userId, defaultsGranted);

                    } break;

                    case TAG_PACKAGE: {  //【2】解析 pkg 标签
                        String name = parser.getAttributeValue(null, ATTR_NAME);
                        PackageSetting ps = mPackages.get(name);
                        if (ps == null) {
                            Slog.w(PackageManagerService.TAG, "Unknown package:" + name);
                            XmlUtils.skipCurrentTag(parser);
                            continue;
                        }
                        //【6.2.1】解析并处理 package 的运行时权限授予情况！
                        parsePermissionsLPr(parser, ps.getPermissionsState(), userId);
                    } break;

                    case TAG_SHARED_USER: {  //【3】解析 shared-user 标签
                        String name = parser.getAttributeValue(null, ATTR_NAME);
                        SharedUserSetting sus = mSharedUsers.get(name);
                        if (sus == null) {
                            Slog.w(PackageManagerService.TAG, "Unknown shared user:" + name);
                            XmlUtils.skipCurrentTag(parser);
                            continue;
                        }
                        //【6.2.1】解析并处理 shared-user 的运行时权限授予情况！
                        parsePermissionsLPr(parser, sus.getPermissionsState(), userId);
                    } break;

                    case TAG_RESTORED_RUNTIME_PERMISSIONS: { // restored-perms 标签
                        final String pkgName = parser.getAttributeValue(null, ATTR_PACKAGE_NAME);
                        parseRestoredRuntimePermissionsLPr(parser, pkgName, userId);
                    } break;
                }
            }
        }
```

这里的 mDefaultPermissionsGranted 用于判断是否进行默认运行时权限授予的！


# 7 应用文件扫描

PMS 会按照顺序扫描以下目录：

 - **/vendor/overlay**
 - **/system/framework**
 - **/system/priv-app**
 - **/system/app**
 - **/vendor/app**
 - **/oem/app**

调用 PMS.scanDirTracedLI 进行扫描，需要注意的是被扫描目录的顺序，这个顺序意味着：先被扫描到的文件，就是最终被用到的文件。


## 7.1 系统权限的定义

系统的权限定义在 \android\frameworks\base\core\res\AndroidManifest.xml 文件中！但实际上最终会通过编译生成 framework-res.apk！

我们来看看具体的权限定义：

```java
   <permission-group android:name="android.permission-group.CONTACTS"
        android:icon="@drawable/perm_group_contacts"
        android:label="@string/permgrouplab_contacts"
        android:description="@string/permgroupdesc_contacts"
        android:priority="100" />

    <!-- Allows an application to read the user's contacts data.
        <p>Protection level: dangerous
    -->
    <permission android:name="android.permission.READ_CONTACTS"
        android:permissionGroup="android.permission-group.CONTACTS"
        android:label="@string/permlab_readContacts"
        android:description="@string/permdesc_readContacts"
        android:protectionLevel="dangerous" />
```


## 7.2 系统和应用权限信息的解析

解析系统权限的过程，其实就是解析 framework-res.apk 的过程！

对于这一部分，可以看【应用程序的权限配置和解析】这边博文！


在解析完 Apk 的 AndroidManifest.xml 中定义和申请的权限信息后，会进入扫描的最后阶段 PMS.scanPackageDirtyLI


## 7.3 PMS.scanPackageDirtyLI

无论是系统应用，还是三方应用，最终都会调用 scanPackageDirtyLI 方法处理扫描得到的数据：

这里我们重点看和权限相关的代码段：
```java
    private PackageParser.Package scanPackageDirtyLI(PackageParser.Package pkg,
            final int policyFlags, final int scanFlags, long currentTime, UserHandle user)
            throws PackageManagerException {
        final File scanFile = new File(pkg.codePath);
        ...

        //【×】处理包名为 "android" 的 apk，也就是 framework-res.apk，属于系统平台包！
        if (pkg.packageName.equals("android")) {
        
            synchronized (mPackages) {
                if (mAndroidApplication != null) {
                    Slog.w(TAG, "*************************************************");
                    Slog.w(TAG, "Core android package being redefined.  Skipping.");
                    Slog.w(TAG, " file=" + scanFile);
                    Slog.w(TAG, "*************************************************");
                    throw new PackageManagerException(INSTALL_FAILED_DUPLICATE_PACKAGE,
                            "Core android package being redefined.  Skipping.");
                }

                if ((scanFlags & SCAN_CHECK_ONLY) == 0) { // 进入该分支，设置系统平台包相关的属性！
                    // Set up information for our fall-back user intent resolution activity.
                    mPlatformPackage = pkg;
                    pkg.mVersionCode = mSdkVersion;
                    mAndroidApplication = pkg.applicationInfo;

                    if (!mResolverReplaced) { // 建立 ResolverActivity 的内存对象。
                        mResolveActivity.applicationInfo = mAndroidApplication;
                        mResolveActivity.name = ResolverActivity.class.getName();
                        mResolveActivity.packageName = mAndroidApplication.packageName;
                        mResolveActivity.processName = "system:ui";
                        mResolveActivity.launchMode = ActivityInfo.LAUNCH_MULTIPLE;
                        mResolveActivity.documentLaunchMode = ActivityInfo.DOCUMENT_LAUNCH_NEVER;
                        mResolveActivity.flags = ActivityInfo.FLAG_EXCLUDE_FROM_RECENTS;
                        mResolveActivity.theme = R.style.Theme_Material_Dialog_Alert;
                        mResolveActivity.exported = true;
                        mResolveActivity.enabled = true;
                        mResolveActivity.resizeMode = ActivityInfo.RESIZE_MODE_RESIZEABLE;
                        mResolveActivity.configChanges = ActivityInfo.CONFIG_SCREEN_SIZE
                                | ActivityInfo.CONFIG_SMALLEST_SCREEN_SIZE
                                | ActivityInfo.CONFIG_SCREEN_LAYOUT
                                | ActivityInfo.CONFIG_ORIENTATION
                                | ActivityInfo.CONFIG_KEYBOARD
                                | ActivityInfo.CONFIG_KEYBOARD_HIDDEN;
                        mResolveInfo.activityInfo = mResolveActivity;
                        mResolveInfo.priority = 0;
                        mResolveInfo.preferredOrder = 0;
                        mResolveInfo.match = 0;
                        mResolveComponentName = new ComponentName(
                                mAndroidApplication.packageName, mResolveActivity.name);
                    }
                }
            }
        }

        if (DEBUG_PACKAGE_SCANNING) {
            if ((policyFlags & PackageParser.PARSE_CHATTY) != 0)
                Log.d(TAG, "Scanning package " + pkg.packageName);
        }

        synchronized (mPackages) {
            //【1】如果 PMS.mPackages 已经包含当前的 Package，说明这个包已经被扫描过了，抛异常，不继续处理！
            if (mPackages.containsKey(pkg.packageName)
                    || mSharedLibraries.containsKey(pkg.packageName)) {
                throw new PackageManagerException(INSTALL_FAILED_DUPLICATE_PACKAGE,
                        "Application package " + pkg.packageName
                                + " already installed.  Skipping duplicate.");
            }


            // 系统 apk 不进入这个分支，SCAN_REQUIRE_KNOWN 只有在扫描 data 分区是才会被设置！！
            if ((scanFlags & SCAN_REQUIRE_KNOWN) != 0) {
                if (mExpectingBetter.containsKey(pkg.packageName)) {
                    logCriticalInfo(Log.WARN,
                            "Relax SCAN_REQUIRE_KNOWN requirement for package " + pkg.packageName);
                } else {
                    PackageSetting known = mSettings.peekPackageLPr(pkg.packageName);
                    if (known != null) {
                        if (DEBUG_PACKAGE_SCANNING) {
                            Log.d(TAG, "Examining " + pkg.codePath
                                    + " and requiring known paths " + known.codePathString
                                    + " & " + known.resourcePathString);
                        }
                        if (!pkg.applicationInfo.getCodePath().equals(known.codePathString)
                                || !pkg.applicationInfo.getResourcePath().equals(
                                known.resourcePathString)) {

                            throw new PackageManagerException(INSTALL_FAILED_PACKAGE_CHANGED,
                                    "Application package " + pkg.packageName
                                            + " found at " + pkg.applicationInfo.getCodePath()
                                            + " but expected at " + known.codePathString
                                            + "; ignoring.");
                        }
                    }
                }
            }
        }

        //【×】获得扫描到的 apk 和资源的路径，下面会用到！
        File destCodeFile = new File(pkg.applicationInfo.getCodePath());
        File destResourceFile = new File(pkg.applicationInfo.getResourcePath());

        SharedUserSetting suid = null;
        PackageSetting pkgSetting = null;

        //【×】系统 apk 才能有 mOriginalPackages，mRealPackage 和 mAdoptPermissions！
        if (!isSystemApp(pkg)) {
            pkg.mOriginalPackages = null;
            pkg.mRealPackage = null;
            pkg.mAdoptPermissions = null;
        }

        // Getting the package setting may have a side-effect, so if we
        // are only checking if scan would succeed, stash a copy of the
        // old setting to restore at the end.
        PackageSetting nonMutatedPs = null;


        synchronized (mPackages) {
            //【×】当前的系统 package 是共享 uid 的，要判断其对应的共享 uid 是否存在！
            if (pkg.mSharedUserId != null) {
                suid = mSettings.getSharedUserLPw(pkg.mSharedUserId, 0, 0, true);
                if (suid == null) {
                    throw new PackageManagerException(INSTALL_FAILED_INSUFFICIENT_STORAGE,
                            "Creating application package " + pkg.packageName
                            + " for shared user failed");
                }

                if (DEBUG_PACKAGE_SCANNING) {
                    if ((policyFlags & PackageParser.PARSE_CHATTY) != 0)
                        Log.d(TAG, "Shared UserID " + pkg.mSharedUserId + " (uid=" + suid.userId
                                + "): packages=" + suid.packages);
                }
            }

            ... ... ...
            
            //【3.2.1.7.2】获得当前扫描的这个 package 对应的 packageSetting 对象，如果已经存在就直接返回，不存在就创建！
            // 如果 origPackage 不为 null，创建新的需要重命名为源包的名字！
            pkgSetting = mSettings.getPackageLPw(pkg, origPackage, realName, suid, destCodeFile,
                    destResourceFile, pkg.applicationInfo.nativeLibraryRootDir,
                    pkg.applicationInfo.primaryCpuAbi,
                    pkg.applicationInfo.secondaryCpuAbi,
                    pkg.applicationInfo.flags, pkg.applicationInfo.privateFlags,
                    user, false);
    
            if (pkgSetting == null) {
                throw new PackageManagerException(INSTALL_FAILED_INSUFFICIENT_STORAGE,
                        "Creating application package " + pkg.packageName + " failed");
            }

            //【×】对于系统 package，如果 origPackage 不为 null，改名为源包的名字！
            if (pkgSetting.origPackage != null) {
                pkg.setPackageName(origPackage.name);

                String msg = "New package " + pkgSetting.realName
                        + " renamed to replace old package " + pkgSetting.name;
                reportSettingsProblem(Log.WARN, msg);

                if ((scanFlags & SCAN_CHECK_ONLY) == 0) { // 如果没有设置 SCAN_CHECK_ONLY，记录本次改名！
                    mTransferedPackages.add(origPackage.name);
                }

                // 清空 originPackage 属性！
                pkgSetting.origPackage = null;
            }

            if ((scanFlags & SCAN_CHECK_ONLY) == 0 && realName != null) {
                mTransferedPackages.add(pkg.packageName);
            }

            //【×】如果当前的系统 apk 被覆盖更新过，就添加 ApplicationInfo.FLAG_UPDATED_SYSTEM_APP 标签！
            if (mSettings.isDisabledSystemPackageLPr(pkg.packageName)) { 
                pkg.applicationInfo.flags |= ApplicationInfo.FLAG_UPDATED_SYSTEM_APP;
            }

            if ((policyFlags&PackageParser.PARSE_IS_SYSTEM_DIR) == 0) {
                updateSharedLibrariesLPw(pkg, null);
            }

            if (mFoundPolicyFile) {
                // 给当前的 Package 分配一个标签，用于 seLinux！
                SELinuxMMAC.assignSeinfoValue(pkg);
            }

            pkg.applicationInfo.uid = pkgSetting.appId;
            pkg.mExtras = pkgSetting;

            //【×】处理 keySet 更新和签名校验！
            if (shouldCheckUpgradeKeySetLP(pkgSetting, scanFlags)) {
                if (checkUpgradeKeySetLP(pkgSetting, pkg)) {
                    // 签名正确，更新本地的签名信息！
                    pkgSetting.signatures.mSignatures = pkg.mSignatures;
                } else {
                    // 签名异常处理
                    if ((policyFlags & PackageParser.PARSE_IS_SYSTEM_DIR) == 0) {
                        throw new PackageManagerException(INSTALL_FAILED_UPDATE_INCOMPATIBLE,
                                "Package " + pkg.packageName + " upgrade keys do not match the "
                                + "previously installed version");
                    } else {
                        pkgSetting.signatures.mSignatures = pkg.mSignatures;
                        String msg = "System package " + pkg.packageName
                            + " signature changed; retaining data.";
                        reportSettingsProblem(Log.WARN, msg);
                    }
                }
            } else {
                // 如果不检查 KetSet 更新的话，就直接校验签名！
                try {
                    verifySignaturesLP(pkgSetting, pkg);
                    pkgSetting.signatures.mSignatures = pkg.mSignatures;
                    
                } catch (PackageManagerException e) {
                    if ((policyFlags & PackageParser.PARSE_IS_SYSTEM_DIR) == 0) {
                        throw e;
                    }
                    // 如果签名校验出现问题，这里会先恢复成本次解析的签名！
                    pkgSetting.signatures.mSignatures = pkg.mSignatures;
                    // 但是如果是 sharedUser 的情况，就会报错！
                    if (pkgSetting.sharedUser != null) {
                        if (compareSignatures(pkgSetting.sharedUser.signatures.mSignatures,
                                              pkg.mSignatures) != PackageManager.SIGNATURE_MATCH) {
                            throw new PackageManagerException(
                                    INSTALL_PARSE_FAILED_INCONSISTENT_CERTIFICATES,
                                            "Signature mismatch for shared user: "
                                            + pkgSetting.sharedUser);
                        }
                    }
                    String msg = "System package " + pkg.packageName
                        + " signature changed; retaining data.";
                    reportSettingsProblem(Log.WARN, msg);
                }
            }
            

            ... ... ... ...

            // 同样的，只有系统 apk 才能进入该分支，用于权限继承！
            if ((scanFlags & SCAN_CHECK_ONLY) == 0 && pkg.mAdoptPermissions != null) {
                for (int i = pkg.mAdoptPermissions.size() - 1; i >= 0; i--) {
                    final String origName = pkg.mAdoptPermissions.get(i);
                    final PackageSetting orig = mSettings.peekPackageLPr(origName);
                    if (orig != null) {
                        if (verifyPackageUpdateLPr(orig, pkg)) {
                            Slog.i(TAG, "Adopting permissions from " + origName + " to "
                                    + pkg.packageName);
                            mSettings.transferPermissionsLPw(origName, pkg.packageName);
                        }
                    }
                }
            }
        }

        ... ... ...

        // 如果是系统 package，设置 isOrphaned 的属性为 true！
        if (isSystemApp(pkg)) {
            pkgSetting.isOrphaned = true;
        }

        ... ... ...

        Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "updateSettings"); // 记录 trace 事件！

        boolean createIdmapFailed = false;
        synchronized (mPackages) {
            if (pkgSetting.pkg != null) {
                // Note that |user| might be null during the initial boot scan. If a codePath
                // for an app has changed during a boot scan, it's due to an app update that's
                // part of the system partition and marker changes must be applied to all users.
                maybeRenameForeignDexMarkers(pkgSetting.pkg, pkg,
                    (user != null) ? user : UserHandle.ALL);
            }

            //【important】将新创建或者更新后的 PackageSetting 重新添加到 mSettings 中！
            mSettings.insertPackageSettingLPw(pkgSetting, pkg);
            
            //【important】将本次扫描的 Package 添加到 PMS.mPackages 中去！
            mPackages.put(pkg.applicationInfo.packageName, pkg);
            
            // 将当前的 package 从 mSettings.mPackagesToBeCleaned 中移除，防止数据清除！
            final Iterator<PackageCleanItem> iter = mSettings.mPackagesToBeCleaned.iterator();
            while (iter.hasNext()) {
                PackageCleanItem item = iter.next();
                if (pkgName.equals(item.packageName)) {
                    iter.remove();
                }
            }

            // 处理 package 的第一次安装时间和最近更新时间！
            if (currentTime != 0) {
                if (pkgSetting.firstInstallTime == 0) {
                    pkgSetting.firstInstallTime = pkgSetting.lastUpdateTime = currentTime;

                } else if ((scanFlags&SCAN_UPDATE_TIME) != 0) {
                    pkgSetting.lastUpdateTime = currentTime;

                }
            } else if (pkgSetting.firstInstallTime == 0) {
                pkgSetting.firstInstallTime = pkgSetting.lastUpdateTime = scanFileTime;
                
            } else if ((policyFlags&PackageParser.PARSE_IS_SYSTEM_DIR) != 0) {
                if (scanFileTime != pkgSetting.timeStamp) {
                    pkgSetting.lastUpdateTime = scanFileTime;
                }
            }

            ksms.addScannedPackageLPw(pkg); // 将 package 的 KeySets 添加到 KeySetManagerService 中！
           
            // 处理四大组件！ 
            //【1】处理该 Package 中 的 Provider 信息，添加到 PMS.mProviders 中！
            int N = pkg.providers.size();
            StringBuilder r = null;
            int i;
            for (i=0; i<N; i++) {
                PackageParser.Provider p = pkg.providers.get(i);
                p.info.processName = fixProcessName(pkg.applicationInfo.processName,
                        p.info.processName, pkg.applicationInfo.uid);
                mProviders.addProvider(p); // 添加到 PMS.mProviders 中！
                p.syncable = p.info.isSyncable;
                if (p.info.authority != null) {
                   ... ... ... ...
                }
                if ((policyFlags&PackageParser.PARSE_CHATTY) != 0) {
                    if (r == null) {
                        r = new StringBuilder(256);
                    } else {
                        r.append(' ');
                    }
                    r.append(p.info.name);
                }
            }
            if (r != null) {
                if (DEBUG_PACKAGE_SCANNING) Log.d(TAG, "  Providers: " + r);
            }

            //【2】处理该 Package 中的 Service 信息，添加到 PMS.mService 中！
            N = pkg.services.size();
            r = null;
            for (i=0; i<N; i++) {
                PackageParser.Service s = pkg.services.get(i);
                s.info.processName = fixProcessName(pkg.applicationInfo.processName,
                        s.info.processName, pkg.applicationInfo.uid);
                mServices.addService(s); // 添加到 PMS.mServices 中！
                if ((policyFlags&PackageParser.PARSE_CHATTY) != 0) {
                    if (r == null) {
                        r = new StringBuilder(256);
                    } else {
                        r.append(' ');
                    }
                    r.append(s.info.name);
                }
            }
            if (r != null) {
                if (DEBUG_PACKAGE_SCANNING) Log.d(TAG, "  Services: " + r);
            }

            //【3】处理该 Package 中的 BroadcastReceiver 信息,添加到 PMS.mReceivers 中。
            N = pkg.receivers.size();
            r = null;
            for (i=0; i<N; i++) {
                PackageParser.Activity a = pkg.receivers.get(i);
                a.info.processName = fixProcessName(pkg.applicationInfo.processName,
                        a.info.processName, pkg.applicationInfo.uid);
                mReceivers.addActivity(a, "receiver"); // 添加到 PMS.mReceivers 中！
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
            
            //【4】处理该 Package 中的 activity 信息，添加到 PMS.mActivities 中！
            N = pkg.activities.size();
            r = null;
            for (i=0; i<N; i++) {
                PackageParser.Activity a = pkg.activities.get(i);
                a.info.processName = fixProcessName(pkg.applicationInfo.processName,
                        a.info.processName, pkg.applicationInfo.uid);
                mActivities.addActivity(a, "activity"); // 添加到 PMS.mActivities 中！
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
                if (DEBUG_PACKAGE_SCANNING) Log.d(TAG, "  Activities: " + r);
            }

            //【5】处理该 Package 中的 PermissionGroups 信息，添加到 PMS.mPermissionGroups 中！
            N = pkg.permissionGroups.size();
            r = null;
            for (i=0; i<N; i++) {
                PackageParser.PermissionGroup pg = pkg.permissionGroups.get(i);
                PackageParser.PermissionGroup cur = mPermissionGroups.get(pg.info.name);
                final String curPackageName = cur == null ? null : cur.info.packageName;
                //【5.1】如果 isPackageUpdate 为 true，说明要更新权限组的信息！
                // 如果 cur 为 null，说明是新添加的权限组信息！
                final boolean isPackageUpdate = pg.info.packageName.equals(curPackageName);
                if (cur == null || isPackageUpdate) {
                    mPermissionGroups.put(pg.info.name, pg); // 添加到 PMS.mPermissionGroups 中！
                    if ((policyFlags&PackageParser.PARSE_CHATTY) != 0) {
                        if (r == null) {
                            r = new StringBuilder(256);
                        } else {
                            r.append(' ');
                        }
                        if (isPackageUpdate) {
                            r.append("UPD:");
                        }
                        r.append(pg.info.name);
                    }
                } else {
                    Slog.w(TAG, "Permission group " + pg.info.name + " from package "
                            + pg.info.packageName + " ignored: original from "
                            + cur.info.packageName);
                    if ((policyFlags&PackageParser.PARSE_CHATTY) != 0) {
                        if (r == null) {
                            r = new StringBuilder(256);
                        } else {
                            r.append(' ');
                        }
                        r.append("DUP:");
                        r.append(pg.info.name);
                    }
                }
            }
            if (r != null) {
                if (DEBUG_PACKAGE_SCANNING) Log.d(TAG, "  Permission Groups: " + r);
            }
            
            //【6】处理该 Package 中的定义 Permission 和 Permission-tree 信息！
            N = pkg.permissions.size();
            r = null;
            for (i=0; i<N; i++) {
                PackageParser.Permission p = pkg.permissions.get(i);
                // 默认取消掉 PermissionInfo.FLAG_INSTALLED 标志位！
                p.info.flags &= ~PermissionInfo.FLAG_INSTALLED;

                //【6.1】设置权限所属的 group，前提是只有 Android5.1 以后才支持 Permission Groups！
                if (pkg.applicationInfo.targetSdkVersion > Build.VERSION_CODES.LOLLIPOP_MR1) {
                    p.group = mPermissionGroups.get(p.info.group);
                    if (p.info.group != null && p.group == null) {
                        Slog.w(TAG, "Permission " + p.info.name + " from package "
                                + p.info.packageName + " in an unknown group " + p.info.group);
                    }
                }

                //【6.2】从 Settings 中获得权限管理集合，如果该权限是一个 Permission-tree，
                // 那就返回 mSettings.mPermissionTrees；否则，返回 mSettings.mPermissions！
                ArrayMap<String, BasePermission> permissionMap =
                        p.tree ? mSettings.mPermissionTrees
                                : mSettings.mPermissions;

                //【6.3】尝试获得上次安装时该权限对应的 BasePermission 对象！
                BasePermission bp = permissionMap.get(p.info.name);

                //【6.4】允许系统应用来重新定义非系统权限！
                // 如果上次安装时的 BasePermission 不为 null，但是当前解析的 package 不是上次安装时该权限的定义者！
                if (bp != null && !Objects.equals(bp.sourcePackage, p.info.packageName)) {
                    //【6.4.1】判断上一次安装时，定义该权限的 package 是否是系统应用！
                    // 如果是 currentOwnerIsSystem 为 true！
                    final boolean currentOwnerIsSystem = (bp.perm != null
                            && isSystemApp(bp.perm.owner));
                    
                    // 如果当前解析的定义了该权限的 package 是系统 app，那么进入这里！     
                    if (isSystemApp(p.owner)) {
                        // 如果上次安装时，该权限是一个 BasePermission.TYPE_BUILTIN 系统权限，且 bp.perm 为 null，
                        // 即：拥有者未知，那么这里我们将这个 system package 分配给这个权限！
                        if (bp.type == BasePermission.TYPE_BUILTIN && bp.perm == null) {
                            bp.packageSetting = pkgSetting;
                            bp.perm = p;
                            bp.uid = pkg.applicationInfo.uid;
                            bp.sourcePackage = p.info.packageName;
                            p.info.flags |= PermissionInfo.FLAG_INSTALLED;

                        } else if (!currentOwnerIsSystem) {
                        // 如果上一次安装时，定义该权限的 package 不是系统应用，而定义相同权限的
                        // 本次解析的 package 是系统应用，那么该权限会被系统应用重新定义！
                            String msg = "New decl " + p.owner + " of permission  "
                                    + p.info.name + " is system; overriding " + bp.sourcePackage;
                            reportSettingsProblem(Log.WARN, msg);
                            // 设置 bp 为 null！
                            bp = null;
                        }
                    }
                }

                // 因为 bp 为 null，所谓我们会使用系统应用重新定义权限，如果是 permission-tree，会被添加到 
                // mSettings.mPermissionTrees 中！
                if (bp == null) {
                    bp = new BasePermission(p.info.name, p.info.packageName,
                            BasePermission.TYPE_NORMAL);
                    permissionMap.put(p.info.name, bp);
                }
                // 重新分配拥有者为该系统应用！
                if (bp.perm == null) {
                    if (bp.sourcePackage == null
                            || bp.sourcePackage.equals(p.info.packageName)) {
                        BasePermission tree = findPermissionTreeLP(p.info.name);
                        if (tree == null
                                || tree.sourcePackage.equals(p.info.packageName)) {
                            bp.packageSetting = pkgSetting;
                            bp.perm = p;
                            bp.uid = pkg.applicationInfo.uid;
                            bp.sourcePackage = p.info.packageName;
                            p.info.flags |= PermissionInfo.FLAG_INSTALLED;
                            if ((policyFlags&PackageParser.PARSE_CHATTY) != 0) {
                                if (r == null) {
                                    r = new StringBuilder(256);
                                } else {
                                    r.append(' ');
                                }
                                r.append(p.info.name);
                            }
                        } else {
                            Slog.w(TAG, "Permission " + p.info.name + " from package "
                                    + p.info.packageName + " ignored: base tree "
                                    + tree.name + " is from package "
                                    + tree.sourcePackage);
                        }
                    } else {
                        Slog.w(TAG, "Permission " + p.info.name + " from package "
                                + p.info.packageName + " ignored: original from "
                                + bp.sourcePackage);
                    }
                } else if ((policyFlags&PackageParser.PARSE_CHATTY) != 0) {
                    if (r == null) {
                        r = new StringBuilder(256);
                    } else {
                        r.append(' ');
                    }
                    r.append("DUP:");
                    r.append(p.info.name);
                }
                // 设置 protectionLevel！
                if (bp.perm == p) {
                    bp.protectionLevel = p.info.protectionLevel;
                }
            }

            if (r != null) {
                if (DEBUG_PACKAGE_SCANNING) Log.d(TAG, "  Permissions: " + r);
            }

            N = pkg.instrumentation.size();
            r = null;
            for (i=0; i<N; i++) {
                ... ... ... ...// 这里省略掉，暂时不看！
            }
            if (r != null) {
                if (DEBUG_PACKAGE_SCANNING) Log.d(TAG, "  Instrumentation: " + r);
            }
            
            // 处理该 Package 中的 protectedBroadcasts 信息！
            if (pkg.protectedBroadcasts != null) {
                N = pkg.protectedBroadcasts.size();
                for (i=0; i<N; i++) {
                    mProtectedBroadcasts.add(pkg.protectedBroadcasts.get(i));
                }
            }

            pkgSetting.setTimeStamp(scanFileTime);

            // Create idmap files for pairs of (packages, overlay packages).
            // Note: "android", ie framework-res.apk, is handled by native layers.
            if (pkg.mOverlayTarget != null) {
                // This is an overlay package.
                if (pkg.mOverlayTarget != null && !pkg.mOverlayTarget.equals("android")) {
                    if (!mOverlays.containsKey(pkg.mOverlayTarget)) {
                        mOverlays.put(pkg.mOverlayTarget,
                                new ArrayMap<String, PackageParser.Package>());
                    }
                    ArrayMap<String, PackageParser.Package> map = mOverlays.get(pkg.mOverlayTarget);
                    map.put(pkg.packageName, pkg);
                    PackageParser.Package orig = mPackages.get(pkg.mOverlayTarget);
                    if (orig != null && !createIdmapForPackagePairLI(orig, pkg)) {
                        createIdmapFailed = true;
                    }
                }
            } else if (mOverlays.containsKey(pkg.packageName) &&
                    !pkg.packageName.equals("android")) {
                // This is a regular package, with one or more known overlay packages.
                createIdmapsForPackageLI(pkg);
            }
        }

        Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);

        if (createIdmapFailed) {
            throw new PackageManagerException(INSTALL_FAILED_UPDATE_INCOMPATIBLE,
                    "scanPackageLI failed to createIdmap");
        }
        return pkg;
    }

```
这里会将解析到的 Packages 和 PackageSetting 添加到对应的集合中！

```java
    //【important】将新创建或者更新后的 PackageSetting 重新添加到 mSettings 中！
    mSettings.insertPackageSettingLPw(pkgSetting, pkg);
    //【important】将本次扫描的 Package 添加到 PMS.mPackages 中去！
    mPackages.put(pkg.applicationInfo.packageName, pkg);
```

# 8 更新权限信息

在 PMS 的扫描最后阶段 PMS_SCAN_END，会更新权限信息：

```java
            // 如果从我们上次启动，SDK 平台被改变了，我们需要重新授予应用程序权限。
            // 这里会有一些安全问题，就是可能会有应用通过这种方式获取那些用户没有显式允许的权限
            // 这里可能后续 google 会改善！
            int updateFlags = UPDATE_PERMISSIONS_ALL;
            if (ver.sdkVersion != mSdkVersion) {
                Slog.i(TAG, "Platform changed from " + ver.sdkVersion + " to "
                        + mSdkVersion + "; regranting permissions for internal storage");
                updateFlags |= UPDATE_PERMISSIONS_REPLACE_PKG | UPDATE_PERMISSIONS_REPLACE_ALL;
            }
            
            //【1】更新系统中的权限和权限树，移除无效的权限和权限树，同时更新应用的权限授予情况！
            updatePermissionsLPw(null, null, StorageManager.UUID_PRIVATE_INTERNAL, updateFlags);
```
这里的最重要的方法是 updatePermissionsLPw 方法：

## 8.1 PMS.updatePermissionsLPw

String changingPkg 表示指定的权限发生变化的 package name，可以为 null; PackageParser.Package pkgInfo 表示该 package 的解析信息！

这里的 replaceVolumeUuid 传入的是 StorageManager.UUID_PRIVATE_INTERNAL！

```java
    private void updatePermissionsLPw(String changingPkg,
            PackageParser.Package pkgInfo, String replaceVolumeUuid, int flags) {
        //【1】确保系统中没有失效的权限树；
        Iterator<BasePermission> it = mSettings.mPermissionTrees.values().iterator();
        while (it.hasNext()) {
            final BasePermission bp = it.next();
            if (bp.packageSetting == null) {
                // 如果 packageSetting 为 null，尝试查找定义的 package！
                bp.packageSetting = mSettings.mPackages.get(bp.sourcePackage);
            }

            //【1.1】如果找不到定义该权限树的 package，那就移除该 permission-tree！
            if (bp.packageSetting == null) {
                Slog.w(TAG, "Removing dangling permission tree: " + bp.name
                        + " from package " + bp.sourcePackage);
                it.remove();

            } else if (changingPkg != null && changingPkg.equals(bp.sourcePackage)) {
                // 根据参数，不进入这里，但是我们来分析下这部分代码的意思
                // 如果权限的定义者不在定义这个权限了，那就移除该权限的上一次信息！
                if (pkgInfo == null || !hasPermission(pkgInfo, bp.name)) {
                    Slog.i(TAG, "Removing old permission tree: " + bp.name
                            + " from package " + bp.sourcePackage);
                    flags |= UPDATE_PERMISSIONS_ALL;
                    it.remove();
                }
            }
        }

        //【2】确保系统中所有的动态权限都已经分配给对应的 package，同时系统中也没有失效的权限!
        it = mSettings.mPermissions.values().iterator();
        while (it.hasNext()) {
            final BasePermission bp = it.next();
            //【2.1】如果是动态权限，尝试找到其所属的 permission-tree，并绑定到定义了 tree 的 package 上！
            if (bp.type == BasePermission.TYPE_DYNAMIC) {
                if (DEBUG_SETTINGS) Log.v(TAG, "Dynamic permission: name="
                        + bp.name + " pkg=" + bp.sourcePackage
                        + " info=" + bp.pendingInfo);
    
                if (bp.packageSetting == null && bp.pendingInfo != null) {
                    final BasePermission tree = findPermissionTreeLP(bp.name);
                    if (tree != null && tree.perm != null) {
                        bp.packageSetting = tree.packageSetting;
                        bp.perm = new PackageParser.Permission(tree.perm.owner,
                                new PermissionInfo(bp.pendingInfo));
                        bp.perm.info.packageName = tree.perm.info.packageName;
                        bp.perm.info.name = bp.name;
                        bp.uid = tree.uid;
                    }
                }
            }

            //【2.2】如果权限的定义者为 null，尝试在系统中找到定义者！
            if (bp.packageSetting == null) {
                bp.packageSetting = mSettings.mPackages.get(bp.sourcePackage);
            }
            //【2.3】如果依然找不到定义该权限的 package，那就移除该 permission ！
            if (bp.packageSetting == null) { // 该权限无效，移除它！
                Slog.w(TAG, "Removing dangling permission: " + bp.name
                        + " from package " + bp.sourcePackage);
                it.remove();
            } else if (changingPkg != null && changingPkg.equals(bp.sourcePackage)) { 
                // 根据参数，不进入这里，但是我们来分析下这部分代码的意思
                // 如果权限的定义者不在定义这个权限了，那就移除该权限的上一次信息！
                if (pkgInfo == null || !hasPermission(pkgInfo, bp.name)) {
                    Slog.i(TAG, "Removing old permission: " + bp.name
                            + " from package " + bp.sourcePackage);
                    flags |= UPDATE_PERMISSIONS_ALL;
                    it.remove();
                }
            }
        }

        //【3】如果 flags 设置了 UPDATE_PERMISSIONS_ALL 标志位，那么就更新除了 changingPkg 以外的其他
        // 所有的 package 的权限授予信息 ，特别是替换系统包的授予权限！
        if ((flags & UPDATE_PERMISSIONS_ALL) != 0) {
            for (PackageParser.Package pkg : mPackages.values()) {
                if (pkg != pkgInfo) {
                    final String volumeUuid = getVolumeUuidForPackage(pkg);
                    final boolean replace = ((flags & UPDATE_PERMISSIONS_REPLACE_ALL) != 0)
                            && Objects.equals(replaceVolumeUuid, volumeUuid);
                    //【3.1】重新授予权限！
                    grantPermissionsLPw(pkg, replace, changingPkg);
                }
            }
        }
        //【4】因为这里 pkgInfo 为 null，所以不进入这里！
        // 这里的逻辑是，如果 changingPkg 不为 null，最后也会更新其权限的授予信息！
        if (pkgInfo != null) { 
            // Only replace for packages on requested volume
            final String volumeUuid = getVolumeUuidForPackage(pkgInfo);
            final boolean replace = ((flags & UPDATE_PERMISSIONS_REPLACE_PKG) != 0)
                    && Objects.equals(replaceVolumeUuid, volumeUuid);
            grantPermissionsLPw(pkgInfo, replace, changingPkg);
        }
    }

```
这里就分析到这里！


# 9 持久化权限相关信息

核心方法是在 Settings.writeLPr 中：

```java
Settings.writeLPr
```
下面我们来看下：

## 9.1 Settings.writeLPr
将解析到的安装数据写回到 packages.xml 等文件中！
```java
    void writeLPr() {
        Debug.startMethodTracing("/data/system/packageprof", 8 * 1024 * 1024);
        //【1】下面会先将最新的数据写入到 packages-backup.xml 文件中！
        if (mSettingsFilename.exists()) {
            if (!mBackupSettingsFilename.exists()) {
                if (!mSettingsFilename.renameTo(mBackupSettingsFilename)) {
                    Slog.wtf(PackageManagerService.TAG,
                            "Unable to backup package manager settings, "
                            + " current changes will be lost at reboot");
                    return;
                }
            } else {
                mSettingsFilename.delete();
                Slog.w(PackageManagerService.TAG, "Preserving older settings backup");
            }
        }

        mPastSignatures.clear();

        try {
            FileOutputStream fstr = new FileOutputStream(mSettingsFilename);
            BufferedOutputStream str = new BufferedOutputStream(fstr);

            //XmlSerializer serializer = XmlUtils.serializerInstance();
            XmlSerializer serializer = new FastXmlSerializer();
            serializer.setOutput(str, StandardCharsets.UTF_8.name());
            serializer.startDocument(null, true);
            serializer.setFeature("http://xmlpull.org/v1/doc/features.html#indent-output", true);

            serializer.startTag(null, "packages"); //【2】写入 "packages"！

            ... ... ... ...

            serializer.startTag(null, "permission-trees"); //【3】写入 "permission-trees"
            for (BasePermission bp : mPermissionTrees.values()) {
                writePermissionLPr(serializer, bp);
            }
            serializer.endTag(null, "permission-trees");

            serializer.startTag(null, "permissions"); //【4】写入 "permissions"
            for (BasePermission bp : mPermissions.values()) {
                writePermissionLPr(serializer, bp);
            }
            serializer.endTag(null, "permissions");

            for (final PackageSetting pkg : mPackages.values()) { //【5】写入 "package"
                writePackageLPr(serializer, pkg);
            }

            for (final PackageSetting pkg : mDisabledSysPackages.values()) { //【6】写入 "updated-package"
                writeDisabledSysPackageLPr(serializer, pkg);
            }

            for (final SharedUserSetting usr : mSharedUsers.values()) { //【7】写入 "shared-user"
                serializer.startTag(null, "shared-user");
                serializer.attribute(null, ATTR_NAME, usr.name);
                serializer.attribute(null, "userId",
                        Integer.toString(usr.userId));
                usr.signatures.writeXml(serializer, "sigs", mPastSignatures);
                // 写入 shared-user 的安装时权限！
                writePermissionsLPr(serializer, usr.getPermissionsState()
                        .getInstallPermissionStates());
                serializer.endTag(null, "shared-user");
            }

            ... ... ...

            serializer.endTag(null, "packages"); // 结束 packages-backup.xml 文件的写入！

            ... ... ... ...

            writePackageListLPr(); // 写入 packageslist 文件！
            writeAllUsersPackageRestrictionsLPr(); // 写入偏好设置信息；
            writeAllRuntimePermissionsLPr(); //【8】写入运行时权限授予信息！
            return;

        } catch(XmlPullParserException e) {
            Slog.wtf(PackageManagerService.TAG, "Unable to write package manager settings, "
                    + "current changes will be lost at reboot", e);
        } catch(java.io.IOException e) {
            Slog.wtf(PackageManagerService.TAG, "Unable to write package manager settings, "
                    + "current changes will be lost at reboot", e);
        }
        // Clean up partially written files
        if (mSettingsFilename.exists()) {
            if (!mSettingsFilename.delete()) {
                Slog.wtf(PackageManagerService.TAG, "Failed to clean up mangled file: "
                        + mSettingsFilename);
            }
        }
    }
```
这里我们可以看到，整个的写入顺序！


# 10 默认运行时权限授予

在开机的最后阶段，PMS 会通过 DefaultPermissionGrantPolicy.grantDefaultPermissions 方法来默认授予一些系统应用运行所必须的系统权限

```java
        for (int userId : grantPermissionsUserIds) {
            mDefaultPermissionPolicy.grantDefaultPermissions(userId);
        }
        // 如果没有默认授予运行时权限给指定的 userId，那么我们会直接读取 default-permissions 目录中的文件
        // 初始化 mGrantExceptions，和上面的过程类似，不关注！
        if (grantPermissionsUserIds == EMPTY_INT_ARRAY) {
            mDefaultPermissionPolicy.scheduleReadDefaultPermissionExceptions();
        }
```
