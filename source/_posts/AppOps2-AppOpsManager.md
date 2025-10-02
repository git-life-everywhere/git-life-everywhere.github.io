# AppOps 第 2 篇 - AppOpsManager 分析
title: AppOps 第 2 篇 - AppOpsManager 分析
date: 2017/08/13 20:46:25
catalog: true
categories:
- AndroidFramework源码分析
- AppOps应用操作管理
tags: AppOps应用操作管理
---

[toc]

本文基于 Android 7.1.1 源码分析，如有错误，欢迎指正，谢谢！

# 0 综述

AppOpsService 实现了大部分的核心功能逻辑，但它不能被其他模块直接调用访问，而是通过 AppOpsManager 提供访问接口。

# 1 重要常量和变量

下面我们来看下 AppOpsManager 中定义的一些重要的变量和常量！

## 1.1 Operation Code
AppOpsManager 一共定义了 64 种 Op Code，我们一起看下：

```java
    // when adding one of these:
    //  - increment _NUM_OP
    //  - add rows to sOpToSwitch, sOpToString, sOpNames, sOpToPerms, sOpDefault
    //  - add descriptive strings to Settings/res/values/arrays.xml
    //  - add the op to the appropriate template in AppOpsState.OpsTemplate (settings app)
    /** @hide No operation specified. */
    public static final int OP_NONE = -1;
    /** @hide Access to coarse location information. */
    public static final int OP_COARSE_LOCATION = 0;
    /** @hide Access to fine location information. */
    public static final int OP_FINE_LOCATION = 1;
    /** @hide Causing GPS to run. */
    public static final int OP_GPS = 2;
    /** @hide */
    public static final int OP_VIBRATE = 3;
    /** @hide */
    public static final int OP_READ_CONTACTS = 4;
    /** @hide */
    public static final int OP_WRITE_CONTACTS = 5;
    /** @hide */
    public static final int OP_READ_CALL_LOG = 6;
    /** @hide */
    public static final int OP_WRITE_CALL_LOG = 7;
    /** @hide */
    public static final int OP_READ_CALENDAR = 8;
    /** @hide */
    public static final int OP_WRITE_CALENDAR = 9;
    /** @hide */
    public static final int OP_WIFI_SCAN = 10;
    /** @hide */
    public static final int OP_POST_NOTIFICATION = 11;
    /** @hide */
    public static final int OP_NEIGHBORING_CELLS = 12;
    /** @hide */
    public static final int OP_CALL_PHONE = 13;
    /** @hide */
    public static final int OP_READ_SMS = 14;
    /** @hide */
    public static final int OP_WRITE_SMS = 15;
    /** @hide */
    public static final int OP_RECEIVE_SMS = 16;
    /** @hide */
    public static final int OP_RECEIVE_EMERGECY_SMS = 17;
    /** @hide */
    public static final int OP_RECEIVE_MMS = 18;
    /** @hide */
    public static final int OP_RECEIVE_WAP_PUSH = 19;
    /** @hide */
    public static final int OP_SEND_SMS = 20;
    /** @hide */
    public static final int OP_READ_ICC_SMS = 21;
    /** @hide */
    public static final int OP_WRITE_ICC_SMS = 22;
    /** @hide */
    public static final int OP_WRITE_SETTINGS = 23;
    /** @hide */
    public static final int OP_SYSTEM_ALERT_WINDOW = 24;
    /** @hide */
    public static final int OP_ACCESS_NOTIFICATIONS = 25;
    /** @hide */
    public static final int OP_CAMERA = 26;
    /** @hide */
    public static final int OP_RECORD_AUDIO = 27;
    /** @hide */
    public static final int OP_PLAY_AUDIO = 28;
    /** @hide */
    public static final int OP_READ_CLIPBOARD = 29;
    /** @hide */
    public static final int OP_WRITE_CLIPBOARD = 30;
    /** @hide */
    public static final int OP_TAKE_MEDIA_BUTTONS = 31;
    /** @hide */
    public static final int OP_TAKE_AUDIO_FOCUS = 32;
    /** @hide */
    public static final int OP_AUDIO_MASTER_VOLUME = 33;
    /** @hide */
    public static final int OP_AUDIO_VOICE_VOLUME = 34;
    /** @hide */
    public static final int OP_AUDIO_RING_VOLUME = 35;
    /** @hide */
    public static final int OP_AUDIO_MEDIA_VOLUME = 36;
    /** @hide */
    public static final int OP_AUDIO_ALARM_VOLUME = 37;
    /** @hide */
    public static final int OP_AUDIO_NOTIFICATION_VOLUME = 38;
    /** @hide */
    public static final int OP_AUDIO_BLUETOOTH_VOLUME = 39;
    /** @hide */
    public static final int OP_WAKE_LOCK = 40;
    /** @hide Continually monitoring location data. */
    public static final int OP_MONITOR_LOCATION = 41;
    /** @hide Continually monitoring location data with a relatively high power request. */
    public static final int OP_MONITOR_HIGH_POWER_LOCATION = 42;
    /** @hide Retrieve current usage stats via {@link UsageStatsManager}. */
    public static final int OP_GET_USAGE_STATS = 43;
    /** @hide */
    public static final int OP_MUTE_MICROPHONE = 44;
    /** @hide */
    public static final int OP_TOAST_WINDOW = 45;
    /** @hide Capture the device's display contents and/or audio */
    public static final int OP_PROJECT_MEDIA = 46;
    /** @hide Activate a VPN connection without user intervention. */
    public static final int OP_ACTIVATE_VPN = 47;
    /** @hide Access the WallpaperManagerAPI to write wallpapers. */
    public static final int OP_WRITE_WALLPAPER = 48;
    /** @hide Received the assist structure from an app. */
    public static final int OP_ASSIST_STRUCTURE = 49;
    /** @hide Received a screenshot from assist. */
    public static final int OP_ASSIST_SCREENSHOT = 50;
    /** @hide Read the phone state. */
    public static final int OP_READ_PHONE_STATE = 51;
    /** @hide Add voicemail messages to the voicemail content provider. */
    public static final int OP_ADD_VOICEMAIL = 52;
    /** @hide Access APIs for SIP calling over VOIP or WiFi. */
    public static final int OP_USE_SIP = 53;
    /** @hide Intercept outgoing calls. */
    public static final int OP_PROCESS_OUTGOING_CALLS = 54;
    /** @hide User the fingerprint API. */
    public static final int OP_USE_FINGERPRINT = 55;
    /** @hide Access to body sensors such as heart rate, etc. */
    public static final int OP_BODY_SENSORS = 56;
    /** @hide Read previously received cell broadcast messages. */
    public static final int OP_READ_CELL_BROADCASTS = 57;
    /** @hide Inject mock location into the system. */
    public static final int OP_MOCK_LOCATION = 58;
    /** @hide Read external storage. */
    public static final int OP_READ_EXTERNAL_STORAGE = 59;
    /** @hide Write external storage. */
    public static final int OP_WRITE_EXTERNAL_STORAGE = 60;
    /** @hide Turned on the screen. */
    public static final int OP_TURN_SCREEN_ON = 61;
    /** @hide Get device accounts. */
    public static final int OP_GET_ACCOUNTS = 62;
    /** @hide Control whether an application is allowed to run in the background. */
    public static final int OP_RUN_IN_BACKGROUND = 63;
```
具体的意思，这里就暂时不解释了！

我们可以看到，原生定义了 64 个 Operation Code，但是也支持扩充，从第一行的 log 中可以看出：如果想扩充，需要更新相应的集合！

同时，AppOpsManager 内部也定义了一些数组：

### 1.1.1 数组 RUNTIME_PERMISSIONS_OPS

```java
    private static final int[] RUNTIME_PERMISSIONS_OPS = {
            // Contacts
            OP_READ_CONTACTS,
            OP_WRITE_CONTACTS,
            OP_GET_ACCOUNTS,
            // Calendar
            OP_READ_CALENDAR,
            OP_WRITE_CALENDAR,
            // SMS
            OP_SEND_SMS,
            OP_RECEIVE_SMS,
            OP_READ_SMS,
            OP_RECEIVE_WAP_PUSH,
            OP_RECEIVE_MMS,
            OP_READ_CELL_BROADCASTS,
            // Storage
            OP_READ_EXTERNAL_STORAGE,
            OP_WRITE_EXTERNAL_STORAGE,
            // Location
            OP_COARSE_LOCATION,
            OP_FINE_LOCATION,
            // Phone
            OP_READ_PHONE_STATE,
            OP_CALL_PHONE,
            OP_READ_CALL_LOG,
            OP_WRITE_CALL_LOG,
            OP_ADD_VOICEMAIL,
            OP_USE_SIP,
            OP_PROCESS_OUTGOING_CALLS,
            // Microphone
            OP_RECORD_AUDIO,
            // Camera
            OP_CAMERA,
            // Body sensors
            OP_BODY_SENSORS
    };
```
RUNTIME_PERMISSIONS_OPS 中定义了一些运行时权限相关的 Op，从名字我们能很容易看出他们映射的是什么运行时权限！

### 1.1.2 数组 sOpToSwitch

```java
    private static int[] sOpToSwitch = new int[] {
            OP_COARSE_LOCATION,
            OP_COARSE_LOCATION,
            OP_COARSE_LOCATION,
            OP_VIBRATE,
            OP_READ_CONTACTS,
            OP_WRITE_CONTACTS,
            OP_READ_CALL_LOG,
            OP_WRITE_CALL_LOG,
            OP_READ_CALENDAR,
            OP_WRITE_CALENDAR,
            OP_COARSE_LOCATION,
            OP_POST_NOTIFICATION,
            OP_COARSE_LOCATION,
            OP_CALL_PHONE,
            OP_READ_SMS,
            OP_WRITE_SMS,
            OP_RECEIVE_SMS,
            OP_RECEIVE_SMS,
            OP_RECEIVE_SMS,
            OP_RECEIVE_SMS,
            OP_SEND_SMS,
            OP_READ_SMS,
            OP_WRITE_SMS,
            OP_WRITE_SETTINGS,
            OP_SYSTEM_ALERT_WINDOW,
            OP_ACCESS_NOTIFICATIONS,
            OP_CAMERA,
            OP_RECORD_AUDIO,
            OP_PLAY_AUDIO,
            OP_READ_CLIPBOARD,
            OP_WRITE_CLIPBOARD,
            OP_TAKE_MEDIA_BUTTONS,
            OP_TAKE_AUDIO_FOCUS,
            OP_AUDIO_MASTER_VOLUME,
            OP_AUDIO_VOICE_VOLUME,
            OP_AUDIO_RING_VOLUME,
            OP_AUDIO_MEDIA_VOLUME,
            OP_AUDIO_ALARM_VOLUME,
            OP_AUDIO_NOTIFICATION_VOLUME,
            OP_AUDIO_BLUETOOTH_VOLUME,
            OP_WAKE_LOCK,
            OP_COARSE_LOCATION,
            OP_COARSE_LOCATION,
            OP_GET_USAGE_STATS,
            OP_MUTE_MICROPHONE,
            OP_TOAST_WINDOW,
            OP_PROJECT_MEDIA,
            OP_ACTIVATE_VPN,
            OP_WRITE_WALLPAPER,
            OP_ASSIST_STRUCTURE,
            OP_ASSIST_SCREENSHOT,
            OP_READ_PHONE_STATE,
            OP_ADD_VOICEMAIL,
            OP_USE_SIP,
            OP_PROCESS_OUTGOING_CALLS,
            OP_USE_FINGERPRINT,
            OP_BODY_SENSORS,
            OP_READ_CELL_BROADCASTS,
            OP_MOCK_LOCATION,
            OP_READ_EXTERNAL_STORAGE,
            OP_WRITE_EXTERNAL_STORAGE,
            OP_TURN_SCREEN_ON,
            OP_GET_ACCOUNTS,
            OP_RUN_IN_BACKGROUND,
    };
```
数组 sOpToSwitch 的长度是 64，保存的是 Op 和 Op 的映射，下标 i 是具体的 Operation，而 value 对应 Operation 的允许情况将决定下标对应的 Operation 的允许情况！

一般情况下，是一对一的映射关系，比如 OP_RUN_IN_BACKGROUND 只和自身相对应，而 OP_COARSE_LOCATION，OP_FINE_LOCATION 和 OP_GPS 则是由 OP_COARSE_LOCATION 是否被运允许来统一决定！！

映射到界面上，就是一个 op 所谓 switch 开关决定一个或者多个 op 的允许情况！

### 1.1.3 数组 sOpToString
```java
    private static String[] sOpToString = new String[] {
            OPSTR_COARSE_LOCATION,
            OPSTR_FINE_LOCATION,
            null,
            null,
            OPSTR_READ_CONTACTS,
            OPSTR_WRITE_CONTACTS,
            OPSTR_READ_CALL_LOG,
            OPSTR_WRITE_CALL_LOG,
            OPSTR_READ_CALENDAR,
            OPSTR_WRITE_CALENDAR,
            null,
            null,
            null,
            OPSTR_CALL_PHONE,
            OPSTR_READ_SMS,
            null,
            OPSTR_RECEIVE_SMS,
            null,
            OPSTR_RECEIVE_MMS,
            OPSTR_RECEIVE_WAP_PUSH,
            OPSTR_SEND_SMS,
            null,
            null,
            OPSTR_WRITE_SETTINGS,
            OPSTR_SYSTEM_ALERT_WINDOW,
            null,
            OPSTR_CAMERA,
            OPSTR_RECORD_AUDIO,
            null,
            null,
            null,
            null,
            null,
            null,
            null,
            null,
            null,
            null,
            null,
            null,
            null,
            OPSTR_MONITOR_LOCATION,
            OPSTR_MONITOR_HIGH_POWER_LOCATION,
            OPSTR_GET_USAGE_STATS,
            null,
            null,
            null,
            OPSTR_ACTIVATE_VPN,
            null,
            null,
            null,
            OPSTR_READ_PHONE_STATE,
            OPSTR_ADD_VOICEMAIL,
            OPSTR_USE_SIP,
            null,
            OPSTR_USE_FINGERPRINT,
            OPSTR_BODY_SENSORS,
            OPSTR_READ_CELL_BROADCASTS,
            OPSTR_MOCK_LOCATION,
            OPSTR_READ_EXTERNAL_STORAGE,
            OPSTR_WRITE_EXTERNAL_STORAGE,
            null,
            OPSTR_GET_ACCOUNTS,
            null,
    };
```
数组 sOpToString 的长度是 64，保存的是 Op 和对应的字符串常量的映射，如果没有对应的字符串相映射，可以为 null；

```java
    /** Access to coarse location information. */
    public static final String OPSTR_COARSE_LOCATION = "android:coarse_location";
    /** Access to fine location information. */
    public static final String OPSTR_FINE_LOCATION =
            "android:fine_location";
    /** Continually monitoring location data. */
    public static final String OPSTR_MONITOR_LOCATION
            = "android:monitor_location";
    /** Continually monitoring location data with a relatively high power request. */
    public static final String OPSTR_MONITOR_HIGH_POWER_LOCATION
    ... ... ...
```
可以看出来，这个 op 对应的 str 的格式是：“android:xxxxxx”

### 1.1.4 数组 sOpPerms
```java
    private static String[] sOpPerms = new String[] {
            android.Manifest.permission.ACCESS_COARSE_LOCATION,
            android.Manifest.permission.ACCESS_FINE_LOCATION,
            null,
            android.Manifest.permission.VIBRATE,
            android.Manifest.permission.READ_CONTACTS,
            android.Manifest.permission.WRITE_CONTACTS,
            android.Manifest.permission.READ_CALL_LOG,
            android.Manifest.permission.WRITE_CALL_LOG,
            android.Manifest.permission.READ_CALENDAR,
            android.Manifest.permission.WRITE_CALENDAR,
            android.Manifest.permission.ACCESS_WIFI_STATE,
            null, // no permission required for notifications
            null, // neighboring cells shares the coarse location perm
            android.Manifest.permission.CALL_PHONE,
            android.Manifest.permission.READ_SMS,
            null, // no permission required for writing sms
            android.Manifest.permission.RECEIVE_SMS,
            android.Manifest.permission.RECEIVE_EMERGENCY_BROADCAST,
            android.Manifest.permission.RECEIVE_MMS,
            android.Manifest.permission.RECEIVE_WAP_PUSH,
            android.Manifest.permission.SEND_SMS,
            android.Manifest.permission.READ_SMS,
            null, // no permission required for writing icc sms
            android.Manifest.permission.WRITE_SETTINGS,
            android.Manifest.permission.SYSTEM_ALERT_WINDOW,
            android.Manifest.permission.ACCESS_NOTIFICATIONS,
            android.Manifest.permission.CAMERA,
            android.Manifest.permission.RECORD_AUDIO,
            null, // no permission for playing audio
            null, // no permission for reading clipboard
            null, // no permission for writing clipboard
            null, // no permission for taking media buttons
            null, // no permission for taking audio focus
            null, // no permission for changing master volume
            null, // no permission for changing voice volume
            null, // no permission for changing ring volume
            null, // no permission for changing media volume
            null, // no permission for changing alarm volume
            null, // no permission for changing notification volume
            null, // no permission for changing bluetooth volume
            android.Manifest.permission.WAKE_LOCK,
            null, // no permission for generic location monitoring
            null, // no permission for high power location monitoring
            android.Manifest.permission.PACKAGE_USAGE_STATS,
            null, // no permission for muting/unmuting microphone
            null, // no permission for displaying toasts
            null, // no permission for projecting media
            null, // no permission for activating vpn
            null, // no permission for supporting wallpaper
            null, // no permission for receiving assist structure
            null, // no permission for receiving assist screenshot
            Manifest.permission.READ_PHONE_STATE,
            Manifest.permission.ADD_VOICEMAIL,
            Manifest.permission.USE_SIP,
            Manifest.permission.PROCESS_OUTGOING_CALLS,
            Manifest.permission.USE_FINGERPRINT,
            Manifest.permission.BODY_SENSORS,
            Manifest.permission.READ_CELL_BROADCASTS,
            null,
            Manifest.permission.READ_EXTERNAL_STORAGE,
            Manifest.permission.WRITE_EXTERNAL_STORAGE,
            null, // no permission for turning the screen on
            Manifest.permission.GET_ACCOUNTS,
            null, // no permission for running in background
    };
```
数组 sOpPerms 的长度是 64，保存的是 Op 和对应权限的映射，如果没有对应的权限相映射，可以为 null；

可以看到，AppOps 不仅映射 dangerous 权限，还映射了 signature 权限！

### 1.1.4 数组 sOpRestrictions
```java
    private static String[] sOpRestrictions = new String[] {
            UserManager.DISALLOW_SHARE_LOCATION, //COARSE_LOCATION
            UserManager.DISALLOW_SHARE_LOCATION, //FINE_LOCATION
            UserManager.DISALLOW_SHARE_LOCATION, //GPS
            null, //VIBRATE
            null, //READ_CONTACTS
            null, //WRITE_CONTACTS
            UserManager.DISALLOW_OUTGOING_CALLS, //READ_CALL_LOG
            UserManager.DISALLOW_OUTGOING_CALLS, //WRITE_CALL_LOG
            null, //READ_CALENDAR
            null, //WRITE_CALENDAR
            UserManager.DISALLOW_SHARE_LOCATION, //WIFI_SCAN
            null, //POST_NOTIFICATION
            null, //NEIGHBORING_CELLS
            null, //CALL_PHONE
            UserManager.DISALLOW_SMS, //READ_SMS
            UserManager.DISALLOW_SMS, //WRITE_SMS
            UserManager.DISALLOW_SMS, //RECEIVE_SMS
            null, //RECEIVE_EMERGENCY_SMS
            UserManager.DISALLOW_SMS, //RECEIVE_MMS
            null, //RECEIVE_WAP_PUSH
            UserManager.DISALLOW_SMS, //SEND_SMS
            UserManager.DISALLOW_SMS, //READ_ICC_SMS
            UserManager.DISALLOW_SMS, //WRITE_ICC_SMS
            null, //WRITE_SETTINGS
            UserManager.DISALLOW_CREATE_WINDOWS, //SYSTEM_ALERT_WINDOW
            null, //ACCESS_NOTIFICATIONS
            UserManager.DISALLOW_CAMERA, //CAMERA
            UserManager.DISALLOW_RECORD_AUDIO, //RECORD_AUDIO
            null, //PLAY_AUDIO
            null, //READ_CLIPBOARD
            null, //WRITE_CLIPBOARD
            null, //TAKE_MEDIA_BUTTONS
            null, //TAKE_AUDIO_FOCUS
            UserManager.DISALLOW_ADJUST_VOLUME, //AUDIO_MASTER_VOLUME
            UserManager.DISALLOW_ADJUST_VOLUME, //AUDIO_VOICE_VOLUME
            UserManager.DISALLOW_ADJUST_VOLUME, //AUDIO_RING_VOLUME
            UserManager.DISALLOW_ADJUST_VOLUME, //AUDIO_MEDIA_VOLUME
            UserManager.DISALLOW_ADJUST_VOLUME, //AUDIO_ALARM_VOLUME
            UserManager.DISALLOW_ADJUST_VOLUME, //AUDIO_NOTIFICATION_VOLUME
            UserManager.DISALLOW_ADJUST_VOLUME, //AUDIO_BLUETOOTH_VOLUME
            null, //WAKE_LOCK
            UserManager.DISALLOW_SHARE_LOCATION, //MONITOR_LOCATION
            UserManager.DISALLOW_SHARE_LOCATION, //MONITOR_HIGH_POWER_LOCATION
            null, //GET_USAGE_STATS
            UserManager.DISALLOW_UNMUTE_MICROPHONE, // MUTE_MICROPHONE
            UserManager.DISALLOW_CREATE_WINDOWS, // TOAST_WINDOW
            null, //PROJECT_MEDIA
            null, // ACTIVATE_VPN
            UserManager.DISALLOW_WALLPAPER, // WRITE_WALLPAPER
            null, // ASSIST_STRUCTURE
            null, // ASSIST_SCREENSHOT
            null, // READ_PHONE_STATE
            null, // ADD_VOICEMAIL
            null, // USE_SIP
            null, // PROCESS_OUTGOING_CALLS
            null, // USE_FINGERPRINT
            null, // BODY_SENSORS
            null, // READ_CELL_BROADCASTS
            null, // MOCK_LOCATION
            null, // READ_EXTERNAL_STORAGE
            null, // WRITE_EXTERNAL_STORAGE
            null, // TURN_ON_SCREEN
            null, // GET_ACCOUNTS
            null, // RUN_IN_BACKGROUND
    };
```
数组 sOpRestrictions 的长度为 64，用来指定 Op 是否应该受到用户限制的影响。每一个 Op 都应该和一个用户限制相映射，如果某个 Op 不受任何用户限制的影响，那么 value 为 null。

### 1.1.5 数组 sOpAllowSystemRestrictionBypass

```
    private static boolean[] sOpAllowSystemRestrictionBypass = new boolean[] {
            true, //COARSE_LOCATION
            true, //FINE_LOCATION
            false, //GPS
            false, //VIBRATE
            false, //READ_CONTACTS
            false, //WRITE_CONTACTS
            false, //READ_CALL_LOG
            false, //WRITE_CALL_LOG
            false, //READ_CALENDAR
            false, //WRITE_CALENDAR
            true, //WIFI_SCAN
            false, //POST_NOTIFICATION
            false, //NEIGHBORING_CELLS
            false, //CALL_PHONE
            false, //READ_SMS
            false, //WRITE_SMS
            false, //RECEIVE_SMS
            false, //RECEIVE_EMERGECY_SMS
            false, //RECEIVE_MMS
            false, //RECEIVE_WAP_PUSH
            false, //SEND_SMS
            false, //READ_ICC_SMS
            false, //WRITE_ICC_SMS
            false, //WRITE_SETTINGS
            true, //SYSTEM_ALERT_WINDOW
            false, //ACCESS_NOTIFICATIONS
            false, //CAMERA
            false, //RECORD_AUDIO
            false, //PLAY_AUDIO
            false, //READ_CLIPBOARD
            false, //WRITE_CLIPBOARD
            false, //TAKE_MEDIA_BUTTONS
            false, //TAKE_AUDIO_FOCUS
            false, //AUDIO_MASTER_VOLUME
            false, //AUDIO_VOICE_VOLUME
            false, //AUDIO_RING_VOLUME
            false, //AUDIO_MEDIA_VOLUME
            false, //AUDIO_ALARM_VOLUME
            false, //AUDIO_NOTIFICATION_VOLUME
            false, //AUDIO_BLUETOOTH_VOLUME
            false, //WAKE_LOCK
            false, //MONITOR_LOCATION
            false, //MONITOR_HIGH_POWER_LOCATION
            false, //GET_USAGE_STATS
            false, //MUTE_MICROPHONE
            true, //TOAST_WINDOW
            false, //PROJECT_MEDIA
            false, //ACTIVATE_VPN
            false, //WALLPAPER
            false, //ASSIST_STRUCTURE
            false, //ASSIST_SCREENSHOT
            false, //READ_PHONE_STATE
            false, //ADD_VOICEMAIL
            false, // USE_SIP
            false, // PROCESS_OUTGOING_CALLS
            false, // USE_FINGERPRINT
            false, // BODY_SENSORS
            false, // READ_CELL_BROADCASTS
            false, // MOCK_LOCATION
            false, // READ_EXTERNAL_STORAGE
            false, // WRITE_EXTERNAL_STORAGE
            false, // TURN_ON_SCREEN
            false, // GET_ACCOUNTS
            false, // RUN_IN_BACKGROUND
    };
```
数组 sOpAllowSystemRestrictionBypass 的长度为 64，数组下标表示具体的 Op，值表示相应的 Operation 是否允许 system 和 system ui 要绕开用户限制！

### 1.1.5 数组 sOpDefaultMode
```java
    private static int[] sOpDefaultMode = new int[] {
            AppOpsManager.MODE_ALLOWED,
            AppOpsManager.MODE_ALLOWED,
            AppOpsManager.MODE_ALLOWED,
            AppOpsManager.MODE_ALLOWED,
            AppOpsManager.MODE_ALLOWED,
            AppOpsManager.MODE_ALLOWED,
            AppOpsManager.MODE_ALLOWED,
            AppOpsManager.MODE_ALLOWED,
            AppOpsManager.MODE_ALLOWED,
            AppOpsManager.MODE_ALLOWED,
            AppOpsManager.MODE_ALLOWED,
            AppOpsManager.MODE_ALLOWED,
            AppOpsManager.MODE_ALLOWED,
            AppOpsManager.MODE_ALLOWED,
            AppOpsManager.MODE_ALLOWED,
            AppOpsManager.MODE_IGNORED, // OP_WRITE_SMS
            AppOpsManager.MODE_ALLOWED,
            AppOpsManager.MODE_ALLOWED,
            AppOpsManager.MODE_ALLOWED,
            AppOpsManager.MODE_ALLOWED,
            AppOpsManager.MODE_ALLOWED,
            AppOpsManager.MODE_ALLOWED,
            AppOpsManager.MODE_ALLOWED,
            AppOpsManager.MODE_DEFAULT, // OP_WRITE_SETTINGS
            AppOpsManager.MODE_DEFAULT, // OP_SYSTEM_ALERT_WINDOW
            AppOpsManager.MODE_ALLOWED,
            AppOpsManager.MODE_ALLOWED,
            AppOpsManager.MODE_ALLOWED,
            AppOpsManager.MODE_ALLOWED,
            AppOpsManager.MODE_ALLOWED,
            AppOpsManager.MODE_ALLOWED,
            AppOpsManager.MODE_ALLOWED,
            AppOpsManager.MODE_ALLOWED,
            AppOpsManager.MODE_ALLOWED,
            AppOpsManager.MODE_ALLOWED,
            AppOpsManager.MODE_ALLOWED,
            AppOpsManager.MODE_ALLOWED,
            AppOpsManager.MODE_ALLOWED,
            AppOpsManager.MODE_ALLOWED,
            AppOpsManager.MODE_ALLOWED,
            AppOpsManager.MODE_ALLOWED,
            AppOpsManager.MODE_ALLOWED,
            AppOpsManager.MODE_ALLOWED,
            AppOpsManager.MODE_DEFAULT, // OP_GET_USAGE_STATS
            AppOpsManager.MODE_ALLOWED,
            AppOpsManager.MODE_ALLOWED,
            AppOpsManager.MODE_IGNORED, // OP_PROJECT_MEDIA
            AppOpsManager.MODE_IGNORED, // OP_ACTIVATE_VPN
            AppOpsManager.MODE_ALLOWED,
            AppOpsManager.MODE_ALLOWED,
            AppOpsManager.MODE_ALLOWED,
            AppOpsManager.MODE_ALLOWED,
            AppOpsManager.MODE_ALLOWED,
            AppOpsManager.MODE_ALLOWED,
            AppOpsManager.MODE_ALLOWED,
            AppOpsManager.MODE_ALLOWED,
            AppOpsManager.MODE_ALLOWED,
            AppOpsManager.MODE_ALLOWED,
            AppOpsManager.MODE_ERRORED,  // OP_MOCK_LOCATION
            AppOpsManager.MODE_ALLOWED,
            AppOpsManager.MODE_ALLOWED,
            AppOpsManager.MODE_ALLOWED,  // OP_TURN_ON_SCREEN
            AppOpsManager.MODE_ALLOWED,
            AppOpsManager.MODE_ALLOWED,  // OP_RUN_IN_BACKGROUND
    };
```
数组 sOpDefaultMode 的长度为 64，用来指定每个 Operation 的默认允许情况（Mode）。

### 1.1.5 数组 sOpDisableReset
```java
    private static boolean[] sOpDisableReset = new boolean[] {
            false,
            false,
            false,
            false,
            false,
            false,
            false,
            false,
            false,
            false,
            false,
            false,
            false,
            false,
            false,
            true,      // OP_WRITE_SMS
            false,
            false,
            false,
            false,
            false,
            false,
            false,
            false,
            false,
            false,
            false,
            false,
            false,
            false,
            false,
            false,
            false,
            false,
            false,
            false,
            false,
            false,
            false,
            false,
            false,
            false,
            false,
            false,
            false,
            false,
            false,
            false,
            false,
            false,
            false,
            false,
            false,
            false,
            false,
            false,
            false,
            false,
            false,
            false,
            false,
            false,
            false,
            false,
    };

```
数组 sOpDisableReset 的长度为 64，用来指定是否允许在重置所有应用偏好设置后，重置 Operation 的授予情况，true 表示禁止重置，false 表示允许重置。

### 1.1.7 哈希表 sOpStrToOp 和 sRuntimePermToOp 
```java
    // 用于保存 Op name 和 Op Code 的映射关系！
    private static HashMap<String, Integer> sOpStrToOp = new HashMap<>();

    // 用于保存运行时权限和 Op Code 的映射关系！
    private static HashMap<String, Integer> sRuntimePermToOp = new HashMap<>();
```

AppOpsManager 有一个静态代码块，static 静态代码块中，会对 AppOpsManager 的每个数组的长度进行 check，如果与 _NUM_OP 值不一致，就会抛异常，会导致开不了机。

同时也会对 sOpStrToOp 和 sRuntimePermToOp 进行初始化！
```java
    static {
        if (sOpToSwitch.length != _NUM_OP) {
            throw new IllegalStateException("sOpToSwitch length " + sOpToSwitch.length
                    + " should be " + _NUM_OP);
        }
        if (sOpToString.length != _NUM_OP) {
            throw new IllegalStateException("sOpToString length " + sOpToString.length
                    + " should be " + _NUM_OP);
        }
        if (sOpNames.length != _NUM_OP) {
            throw new IllegalStateException("sOpNames length " + sOpNames.length
                    + " should be " + _NUM_OP);
        }
        if (sOpPerms.length != _NUM_OP) {
            throw new IllegalStateException("sOpPerms length " + sOpPerms.length
                    + " should be " + _NUM_OP);
        }
        if (sOpDefaultMode.length != _NUM_OP) {
            throw new IllegalStateException("sOpDefaultMode length " + sOpDefaultMode.length
                    + " should be " + _NUM_OP);
        }
        if (sOpDisableReset.length != _NUM_OP) {
            throw new IllegalStateException("sOpDisableReset length " + sOpDisableReset.length
                    + " should be " + _NUM_OP);
        }
        if (sOpRestrictions.length != _NUM_OP) {
            throw new IllegalStateException("sOpRestrictions length " + sOpRestrictions.length
                    + " should be " + _NUM_OP);
        }
        if (sOpAllowSystemRestrictionBypass.length != _NUM_OP) {
            throw new IllegalStateException("sOpAllowSYstemRestrictionsBypass length "
                    + sOpRestrictions.length + " should be " + _NUM_OP);
        }
        //【1】初始化 sOpStrToOp
        for (int i=0; i<_NUM_OP; i++) {
            if (sOpToString[i] != null) {
                sOpStrToOp.put(sOpToString[i], i);
            }
        }
        //【2】初始化 sRuntimePermToOp
        for (int op : RUNTIME_PERMISSIONS_OPS) {
            if (sOpPerms[op] != null) {
                sRuntimePermToOp.put(sOpPerms[op], op);
            }
        }
    }
```

## 1.2 Operation Mode

每一个 Operation Code 都会有如下的 4 中控制状态：

```java
    public static final int MODE_ALLOWED = 0;
```
该值是 checkOp/noteOp/startOp 方法的返回结果之一，同时也是 Operation 的控制状态之一，表示调用者允许执行相应的操作！

```java
    public static final int MODE_IGNORED = 1;
```
该值是 checkOp/noteOp/startOp 方法的返回结果之一，同时也是 Operation 的控制状态之一，表示调用者不允许执行相应的操作！

返回该值不会导致应用因为没权限而 crash！

```java
    public static final int MODE_DEFAULT = 3;
```
该值是 checkOp/noteOp/startOp 方法的返回结果之一，表示给定的调用者应该进行权限检查。这种模式通常不被使用; 它只能与 appop 权限一起使用，调用者必须明确检查并处理它。

```java
    public static final int MODE_ERRORED = 2;
```
该值是 checkOpNoThrow/noteOpNoThrow/startOpNoThrow 方法的返回结果之一，表示调用者不允许执行相应的操作！

返回该值的同时，会抛出异常 SecurityException，导致应用因为没权限而 crash！

## 1.3 属性相关方法


### 1.3.1 opToSwitch
```java
    public static int opToSwitch(int op) {
        return sOpToSwitch[op];
    }
```
该方法会返回控制给定 op 的 op 开关（也是一个 op）

### 1.3.2 opToName
```java
    public static String opToName(int op) {
        if (op == OP_NONE) return "NONE";
        return op < sOpNames.length ? sOpNames[op] : ("Unknown(" + op + ")");
    }
    
    public static int strDebugOpToOp(String op) {
        for (int i=0; i<sOpNames.length; i++) {
            if (sOpNames[i].equals(op)) {
                return i;
            }
        }
        throw new IllegalArgumentException("Unknown operation string: " + op);
    }
```
上述两个方法用于在给定 op 和其对应的字符串名称之间相互转换，用于 debug，通过数组 sOpNames 获得对应的！

### 1.3.3 opToName
```java
    public static String opToPermission(int op) {
        return sOpPerms[op];
    }
```
该方法会返回给定 op 对应的权限，如果没有返回 null，通过数组 sOpPerms 获得权限！

### 1.3.4 opToRestriction
```java
    public static String opToRestriction(int op) {
        return sOpRestrictions[op];
    }
```
该方法会返回和给定 op 相关联的用户限制，如果没有返回 null，通过数组 sOpRestrictions 获得权限！

### 1.3.5 permissionToOpCode
```java
    public static int permissionToOpCode(String permission) {
        Integer boxedOpCode = sRuntimePermToOp.get(permission);
        return boxedOpCode != null ? boxedOpCode : OP_NONE;
    }
```
该方法会返回和指定运行时权限相关联的 Operation，如果没有返回 null，通过数组 sRuntimePermToOp 获得权限！

### 1.3.6 opAllowSystemBypassRestriction
```java
    public static boolean opAllowSystemBypassRestriction(int op) {
        return sOpAllowSystemRestrictionBypass[op];
    }
```
该方法用于判断指定的 Op 是否允许系统（和系统UI）绕过用户对操作的限制！

### 1.3.7 opToDefaultMode
```java
    public static int opToDefaultMode(int op) {
        return sOpDefaultMode[op];
    }
```
该方法用于返回指定的 op 的默认控制情况！

### 1.3.8 opAllowsReset
```java
    /** @hide */
    public static boolean opAllowsReset(int op) {
        return !sOpDisableReset[op];
    }
```
该方法用于判断指定的 op 的是否允许自身的状态被重置！

### 1.3.9 permissionToOp
```java
    /** @hide */
    public static String permissionToOp(String permission) {
        final Integer opCode = sRuntimePermToOp.get(permission);
        if (opCode == null) {
            return null;
        }
        return sOpToString[opCode];
    }
```
该方法用于获得指定的 permission 对应的 op 的名称！

### 1.3.10 strOpToOp
```java
    /** @hide */
    public static int strOpToOp(String op) {
        Integer val = sOpStrToOp.get(op);
        if (val == null) {
            throw new IllegalArgumentException("Unknown operation string: " + op);
        }
        return val;
    }
```
该方法用于获得指定的 op 字符串名称对应的 Operation Code！

#2 重要函数和接口

AppOpsManager 提供了大量的接口，用于设置和检查权限：

## 2.1 setUidMode 相关方法

设置 uid 的 mode：

```java
/** @hide */
public void setUidMode(int code, int uid, int mode) {
    try {
        mService.setUidMode(code, uid, mode);
    } catch (RemoteException e) {
        throw e.rethrowFromSystemServer();
    }
}

/** @hide */
@SystemApi
public void setUidMode(String appOp, int uid, int mode) {
    try {
        mService.setUidMode(AppOpsManager.strOpToOp(appOp), uid, mode);
    } catch (RemoteException e) {
        throw e.rethrowFromSystemServer();
    }
}
```
setUidMode 方法能够给属于该 uid 的所有应用程序的指定 op 设置控制状态！

## 2.2 setUserRestriction 相关方法

设置用户限制：
```java
    /** @hide */
    public void setUserRestriction(int code, boolean restricted, IBinder token) {
        //【1】调用了第二个方法！
        setUserRestriction(code, restricted, token, /*exceptionPackages*/null);
    }

    /** @hide */
    public void setUserRestriction(int code, boolean restricted, IBinder token,
            String[] exceptionPackages) {
        //【2】调用了第三个方法！
        setUserRestrictionForUser(code, restricted, token, exceptionPackages, mContext.getUserId());
    }

    /** @hide */
    public void setUserRestrictionForUser(int code, boolean restricted, IBinder token,
            String[] exceptionPackages, int userId) {
        try {
            mService.setUserRestriction(code, restricted, token, userId, exceptionPackages);
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
```
setUserRestriction 方法用于给指定的 op 设置用户限制！


## 2.3 checkOp 相关方法

检查权限：

```java
    public int checkOp(String op, int uid, String packageName) {
        return checkOp(strOpToOp(op), uid, packageName);
    }
    
    public int checkOp(int op, int uid, String packageName) {
        try {
            int mode = mService.checkOperation(op, uid, packageName);
            if (mode == MODE_ERRORED) {
                throw new SecurityException(buildSecurityExceptionMsg(op, uid, packageName));
            }
            return mode;
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
```
checkOp 用于判断 package 是否允许执行相应的操作，如果不允许，会抛出 SecurityException 异常

```java
    public int checkOpNoThrow(String op, int uid, String packageName) {
        return checkOpNoThrow(strOpToOp(op), uid, packageName);
    }
    
    /** @hide */
    public int checkOpNoThrow(int op, int uid, String packageName) {
        try {
            return mService.checkOperation(op, uid, packageName);
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }

```
checkOpNoThrow 用于判断 package 是否允许执行相应的操作，如果不允许，不会抛出 SecurityException 异常，会返回 MODE_ERRORED！

## 2.4 noteOp 相关方法

检查权限，在检验后会做记录校验结果：

```java
    public int noteOp(String op, int uid, String packageName) {
        return noteOp(strOpToOp(op), uid, packageName);
    }
    
    /** @hide */    
    public int noteOp(int op, int uid, String packageName) {
        try {
            int mode = mService.noteOperation(op, uid, packageName);
            if (mode == MODE_ERRORED) {
                throw new SecurityException(buildSecurityExceptionMsg(op, uid, packageName));
            }
            return mode;
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
```
noteOp 和 checkOp 类似，但是在检验后会做记录校验结果，在结果为 MODE_ERRORED 的情况下，会抛出异常 SecurityException。

```java
    public int noteOpNoThrow(String op, int uid, String packageName) {
        return noteOpNoThrow(strOpToOp(op), uid, packageName);
    }
    
    /** @hide */
    public int noteOpNoThrow(int op, int uid, String packageName) {
        try {
            return mService.noteOperation(op, uid, packageName);
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }

```
noteOp 用于判断 package 是否允许执行相应的操作，在结果为 MODE_ERRORED 的情况下，不会抛出异常 SecurityException。

## 2.5 noteProxyOp 相关方法

检查跨进程 ipc 通信时，调用方 proxiedPackageName 是否有权限，在检验后会做记录校验结果：

```java
    public int noteProxyOp(int op, String proxiedPackageName) {
        //【1】调用另外一个 noteProxyOpNoThrow 方法！
        int mode = noteProxyOpNoThrow(op, proxiedPackageName);
        if (mode == MODE_ERRORED) {
            throw new SecurityException("Proxy package " + mContext.getOpPackageName()
                    + " from uid " + Process.myUid() + " or calling package "
                    + proxiedPackageName + " from uid " + Binder.getCallingUid()
                    + " not allowed to perform " + sOpNames[op]);
        }
        return mode;
    }
```
另外一个方法：
```java
    public int noteProxyOpNoThrow(int op, String proxiedPackageName) {
        try {
            //【1】调用服务端的 noteProxyOperation 方法！
            return mService.noteProxyOperation(op, mContext.getOpPackageName(),
                    Binder.getCallingUid(), proxiedPackageName);
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
```
不多说了！

## 2.6 startOp 相关方法

用于监控长时间的 op 操作！

```java
    /** @hide */
    public int startOp(int op) {
        return startOp(op, Process.myUid(), mContext.getOpPackageName());
    }

    public int startOp(int op, int uid, String packageName) {
        try {
            int mode = mService.startOperation(getToken(mService), op, uid, packageName);
            if (mode == MODE_ERRORED) {
                throw new SecurityException(buildSecurityExceptionMsg(op, uid, packageName));
            }
            return mode;
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
```
startOp 用于报告应用程序已开始执行长时间运行的操作。 必须同时传入要检查的应用程序的 uid 和 packageName; 这个函数将验证这两个匹配，如果不匹配，则返回 MODE_IGNORED 。

如果此调用成功，则此应用程序的最后执行时间将更新为当前时间，并且该操作将被标记为 “正在运行”。 

在这种情况下，稍后必须调用 finishOp 来报告应用程序何时不再执行操作。

```java
    /** @hide */
    public int startOpNoThrow(int op, int uid, String packageName) {
        try {
            return mService.startOperation(getToken(mService), op, uid, packageName);
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
```
startOpNoThrow 和 startOp 类似，只是当结果为 MODE_ERRORED 时，不会抛出异常！

startOpNoThrow 方法调用的是 AppOpsService 的 startOperation 方法，第一个参数传入的是 getToken(mService)，这个方法会返回一个 ClientState 对象！


## 2.7 finishOp 相关方法

结束 startOp 方法监控的操作：

```java
    /** @hide */
    public void finishOp(int op) {
        finishOp(op, Process.myUid(), mContext.getOpPackageName());
    }
    
    public void finishOp(String op, int uid, String packageName) {
        finishOp(strOpToOp(op), uid, packageName);
    }
    
    /** @hide */
    public void finishOp(int op, int uid, String packageName) {
        try {
            mService.finishOperation(getToken(mService), op, uid, packageName);
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
```
报告应用程序不再执行之前使用 startOp 开始的操作，此处提供的参数必须与 startOp 时传递的参数完全相同。


## 2.8 startWatchingMode 相关方法

这里我们一起看下 stopWatchingMode 方法！

- **startWatchingMode 方法**

```java
    public void startWatchingMode(String op, String packageName,
            final OnOpChangedListener callback) {
        startWatchingMode(strOpToOp(op), packageName, callback);
    }

    /** @hide */
    public void startWatchingMode(int op, String packageName, final OnOpChangedListener callback) {
        synchronized (mModeWatchers) {
            IAppOpsCallback cb = mModeWatchers.get(callback);
            if (cb == null) {
                cb = new IAppOpsCallback.Stub() {
                    public void opChanged(int op, int uid, String packageName) {
                        if (callback instanceof OnOpChangedInternalListener) {
                            ((OnOpChangedInternalListener)callback).onOpChanged(op, packageName);
                        }
                        if (sOpToString[op] != null) {
                            callback.onOpChanged(sOpToString[op], packageName);
                        }
                    }
                };
                mModeWatchers.put(callback, cb);
            }
            try {
                //【1】调用了 AppOpsService 的相关接口！
                mService.startWatchingMode(op, packageName, cb);
            } catch (RemoteException e) {
                throw e.rethrowFromSystemServer();
            }
       
```
startWatchingMode 监视给定应用程序中给定操作的模式的变化！当发生变化后会回调 OnOpChangedListener 接口！

- **stopWatchingMode 方法**

```java
    public void stopWatchingMode(OnOpChangedListener callback) {
        synchronized (mModeWatchers) {
            IAppOpsCallback cb = mModeWatchers.get(callback);
            if (cb != null) {
                try {
                    mService.stopWatchingMode(cb);
                } catch (RemoteException e) {
                    throw e.rethrowFromSystemServer();
                }
            }
        }
    }
```
停止监控。 与 callback 回调相关的所有监控都将被删除！


## 2.9 setMode 相关方法

同样是设置 op 的 mode，setMode 方法选择更多些：

```java
    /** @hide */
    public void setMode(int code, int uid, String packageName, int mode) {
        try {
            mService.setMode(code, uid, packageName, mode);
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
```