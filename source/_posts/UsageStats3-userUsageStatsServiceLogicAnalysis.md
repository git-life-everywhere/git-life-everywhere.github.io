# UsageStats 第 3 篇 - UserUsageStatsService 逻辑分析
title: UsageStats 第 3 篇 - UserUsageStatsService 逻辑分析
date: 2017/05/03 20:46:25
categories:
- AndroidFramework源码分析
- UsageStats使用状态管理
tags: UsageStats使用状态管理
---


[toc]

基于 Android7.1.1 源码分析 UsageStatsService 的架构和原理！

# 0 综述

UserUsageStatsService 用于保存每个设备用户下的数据信息！每一个 UserUsageStatsService 内部都会有一个 UsageStatsDatabase 对象用于访问本地持久化文件！

本文主要分析 UserUsageStatsService 相关的接口，为 UsageStatsService 的分析提供帮助！

# 1 new UserUsageStatsService - 创建

参数 File usageStatsDir 传入的是 new File(mUsageStatsDir, Integer.toString(userId))。

mUsageStatsDir 就是 /data/system/usagestats，这里我们不考虑多用户，在默认用户下 usageStatsDir 为 /data/system/usagestats/0

```java
    UserUsageStatsService(Context context, int userId, File usageStatsDir,
            StatsUpdatedListener listener) {
        mContext = context;
        //【1.1】用于日期的计算；
        mDailyExpiryDate = new UnixCalendar(0);
        //【1.2】数据库对象，用于加载和更新持久化文件；
        mDatabase = new UsageStatsDatabase(usageStatsDir);
        //【1.3】创建一个 IntervalStats 数组，大小为 4，用于加载每个时间类别的最新数据到内存中！        
        mCurrentStats = new IntervalStats[UsageStatsManager.INTERVAL_COUNT];
        mListener  = listener;
        mLogPrefix = "User[" + Integer.toString(userId) + "] ";
        mUserId = userId; // 对应的 userId
    }
```
这里的 mListener 就是 UsageStatsService，前面我们知道 UsageStatsService 实现了 UserUsageStatsService.StatsUpdatedListener：

```java
private final StatsUpdatedListener mListener;

interface StatsUpdatedListener {
    void onStatsUpdated();
    void onStatsReloaded();
    void onNewUpdate(int mUserId);
}
```

这样，当 UserUsageStatsService 中的数据发生更新后，会通过 StatsUpdatedListener 的接口通知 UsageStatsService！

- **onStatsUpdated**：当使用信息被更新后，触发该回调；
- **onStatsReloaded**：当使用信息重新加载后，触发该回调；
- **onNewUpdate**：当系统发生了升级后，触发该回调；

## 1.1 new UnixCalendar

```java
public class UnixCalendar {
    public static final long DAY_IN_MILLIS = 24 * 60 * 60 * 1000;
    public static final long WEEK_IN_MILLIS = 7 * DAY_IN_MILLIS;
    public static final long MONTH_IN_MILLIS = 30 * DAY_IN_MILLIS;
    public static final long YEAR_IN_MILLIS = 365 * DAY_IN_MILLIS;
    private long mTime;

    public UnixCalendar(long time) {
        mTime = time;
    }

    public void addDays(int val) {
        mTime += val * DAY_IN_MILLIS;
    }

    public void addWeeks(int val) {
        mTime += val * WEEK_IN_MILLIS;
    }

    public void addMonths(int val) {
        mTime += val * MONTH_IN_MILLIS;
    }

    public void addYears(int val) {
        mTime += val * YEAR_IN_MILLIS;
    }

    public void setTimeInMillis(long time) {
        mTime = time;
    }

    public long getTimeInMillis() {
        return mTime;
    }
}
```
UnixCalendar 用于计算日期的变化，代码很简单，不多说了！


## 1.2 new UsageStatsDatabase

```java
    public UsageStatsDatabase(File dir) {
        //【1】创建了一个文件夹数组，对应了本地的四个不同的目录；
        mIntervalDirs = new File[] {
                new File(dir, "daily"),
                new File(dir, "weekly"),
                new File(dir, "monthly"),
                new File(dir, "yearly"),
        };
        //【2】创建版本信息文件！
        mVersionFile = new File(dir, "version");
        //【3】创建一个 TimeSparseArray，长度为 4，用于对不同目录下的文件根据时间排序！
        mSortedStatFiles = new TimeSparseArray[mIntervalDirs.length];
        mCal = new UnixCalendar(0);
    }
```
在默认用户下，这里的 dir 对应的目录是 /data/system/usagestats/0。

同时创建了一个文件夹数组，其实这里可以知道，数据是按 daily，monthly，weekly，yearly 四个文件夹存储的，而每个文件夹中又包含若干个以一个时间戳为名称的文件！

- /data/system/usagestats/0/daily；
- /data/system/usagestats/0/weekly；
- /data/system/usagestats/0/monthly；
- /data/system/usagestats/0/yearly；

同时，又创建了数据库对应的 version 文件：

- /data/system/usagestats/0/version；

我们来看看 daily，monthly，weekly，yearly 四个文件夹中存储的是什么文件：

```java
/data/system/usagestats/0/daily  ls
1526928050937  1527014648681  1527101544569  1527188059056  1527277127932  1527364251915  1527450652825
```
可以看到，每一个文件的名字都是一个时间戳，这个时间戳的表示的是该文件开始记录的时间节点！

我们来看下，这些文件内部的数据结构：
```java
<?xml version='1.0' encoding='utf-8' standalone='yes' ?>
<usagestats version="1" endTime="62979335">
    <packages>
        <package lastTimeActive="1104580" package="com.github.shadowsocks" timeActive="119959" lastEvent="2" />
    </packages>
    <configurations>
        <config lastTimeActive="60709298" timeActive="60690625" count="3" active="true" fs="1065353216" mcc="460" 
                mnc="65535" locales="zh-CN" touch="3" key="1" keyHid="1" hardKeyHid="2" nav="1" navHid="2" 
                ori="1" scrLay="268435810" clrMod="5" ui="17" width="360" height="685" sw="360" 
                density="480" app_bounds="0 0 1080 2136" />
    </configurations>
    <event-log>
        <event time="1104580" package="com.github.shadowsocks" 
               class="com.github.shadowsocks.MainActivity" flags="0" type="2" />
    </event-log>
</usagestats>
```
可以看到，一个文件主要包含三方面的内容：

- **packages**：应用的使用信息；
- **configurations**：配置的使用信息；
- **event-log**：时间上报的历史记录；

下面，会分析到解析过程！

## 1.3 new IntervalStats

```java
class IntervalStats {
    public long beginTime; // 开始记录的时间
    public long endTime; // 当前记录的最大时间
    public long lastTimeSaved; // 最后一次将数据保存到持久化文件的时间！

    // 内存中的 UsageStats 数据！
    public final ArrayMap<String, UsageStats> packageStats = new ArrayMap<>();
    
    // 内存中的 ConfigurationStats 数据！
    public final ArrayMap<Configuration, ConfigurationStats> configurations = new ArrayMap<>();
    public Configuration activeConfiguration;

    // 内存中的 Event 数据！
    public TimeSparseArray<UsageEvents.Event> events;
    
    private final ArraySet<String> mStringCache = new ArraySet<>(); // 用于缓存包名！
    ... ... ...
}
```
IntervalStats 用于将持久化数据缓存到内存中，加快访问效率！

- 应用的数据以 UsageStats 的形式保存在 packageStats 中；
- 配置的数据以 ConfigurationStats 的形式保存在 configurations 中；

IntervalStats 内部也提供了一些 get，udpate 等等的方法，我们后面分析的时候再说！

IntervalStats 数组的长度为 4 ，和 UsageStatsDatabase 中的四个文件一一对应，分别用于保存对应文件的缓存数据！

**每一个 IntervalStats 都对应着一个名字为时间戳的文件**！

# 2 UserUsageStatsService.init - 初始化操作
```java
    void init(final long currentTimeMillis) {
        //【×2.1】初始化 UsageStatsDatabase 实例！
        mDatabase.init(currentTimeMillis);

        //【2】加载本地持久化文件，初始化缓存数据：mCurrentStats！
        int nullCount = 0;
        for (int i = 0; i < mCurrentStats.length; i++) {
            //【×2.2】将持久化文件的数据加载到内存中！
            mCurrentStats[i] = mDatabase.getLatestUsageStats(i);
            if (mCurrentStats[i] == null) {
                nullCount++; // 统计没有对应数据的类别！
            }
        }

        if (nullCount > 0) {
            // 如果 nullCount 大于 0，说明 daily，monthly，weekly，yearly 这四个数据类型，至少有一个没有对应的数据！
            if (nullCount != mCurrentStats.length) {
                // 说明某个类别没有数据！
                Slog.w(TAG, mLogPrefix + "Some stats have no latest available");
            } else {
                // 第一次启动的情况！
            }
 
            //【×2.3】加载最新的使用信息！
            loadActiveStats(currentTimeMillis);
        } else {
            //【×2.4】设置回滚日期！
            updateRolloverDeadline();
        }

        // Now close off any events that were open at the time this was saved.
        //【2】初始化到这里，mCurrentStats 已经保存了每个时间类别下，对应的最新的使用信息！
        // 接着，对 package 的 UsageStats.mLastEvent 取值做一个特殊处理！
        for (IntervalStats stat : mCurrentStats) {
            final int pkgCount = stat.packageStats.size();
            for (int i = 0; i < pkgCount; i++) {
                UsageStats pkgStats = stat.packageStats.valueAt(i);
                
                //【2.1】如果该 package 的最后一个 event 是 MOVE_TO_FOREGROUND 或者 CONTINUE_PREVIOUS_DAY
                // 更新其为 END_OF_DAY！
                if (pkgStats.mLastEvent == UsageEvents.Event.MOVE_TO_FOREGROUND ||
                        pkgStats.mLastEvent == UsageEvents.Event.CONTINUE_PREVIOUS_DAY) {
                        
                    //【×2.5】更新所属 IntervalStats 的属性！
                    stat.update(pkgStats.mPackageName, stat.lastTimeSaved,
                            UsageEvents.Event.END_OF_DAY);
                            
                    //【×2.6】延迟 20mins 通知 UsageStatsService 使用信息发生变化！
                    // 通知一次！
                    notifyStatsChanged();
                }
            }

            //【×2.7】更新配置信息！
            stat.updateConfigurationStats(null, stat.lastTimeSaved);
        }

        //【×2.8】如果发生了系统升级，isNewUpdate 返回 true！
        if (mDatabase.isNewUpdate()) {
            notifyNewUpdate();
        }
    }
```

## 2.1 UsageStatsDatabase.init

init 方法用于初始化 database，我们来看看具体的流程：

```java
    public void init(long currentTimeMillis) {
        synchronized (mLock) {
            //【1】创建 daily，monthly，weekly，yearly 四个文件夹！
            for (File f : mIntervalDirs) {
                f.mkdirs();
                if (!f.exists()) {
                    throw new IllegalStateException("Failed to create directory "
                            + f.getAbsolutePath());
                }
            }
            //【2.1.1】检查 version 变化！
            checkVersionAndBuildLocked();
            //【2.1.2】加载每个时间类别文件夹中的文件到 mSortedStatFiles 中！
            indexFilesLocked();

            //【3】删除那些日期在当前时间以后的文件！
            for (TimeSparseArray<AtomicFile> files : mSortedStatFiles) {
                //【3.1】根据当前时间确定删除文件的参考点！
                final int startIndex = files.closestIndexOnOrAfter(currentTimeMillis);
                if (startIndex < 0) {
                    continue;
                }
                //【3.2】移除那些日期在当前时间之后的文件！
                final int fileCount = files.size();
                for (int i = startIndex; i < fileCount; i++) {
                    files.valueAt(i).delete();
                }

                for (int i = startIndex; i < fileCount; i++) {
                    files.removeAt(i);
                }
            }
        }
    }
```

方法很简单，不多说了！

### 2.1.1 UsageStatsDatabase.checkVersionAndBuildLocked

checkVersionAndBuildLocked 用于检查版本变化！
```java
    private void checkVersionAndBuildLocked() {
        int version;
        String buildFingerprint;
        // 获得当前系统的 Fingerprint！
        String currentFingerprint = getBuildFingerprint();
        mFirstUpdate = true;
        mNewUpdate = true;
        try (BufferedReader reader = new BufferedReader(new FileReader(mVersionFile))) {
            version = Integer.parseInt(reader.readLine());
            //【1】从 /data/system/usagestats/0version 中读取数据库中记录的Fingerprint！
            // 如果可以读取掉，说明不是第一次更新，那么 mFirstUpdate 为 false；
            // 如果当前系统的 Fingerprint 和数据库中的 Fingerprint 一样，说没有发生系统更新，mNewUpdate 为 false；
            buildFingerprint = reader.readLine();
            if (buildFingerprint != null) {
                mFirstUpdate = false;
            }
            if (currentFingerprint.equals(buildFingerprint)) {
                mNewUpdate = false;
            }
        } catch (NumberFormatException | IOException e) {
            version = 0;
        }
        //【2】如果 version 不等于 3，说明存在数据库升级！
        if (version != CURRENT_VERSION) {
            Slog.i(TAG, "Upgrading from version " + version + " to " + CURRENT_VERSION);
            //【2.1.1.1】升级数据库，删除旧文件！
            doUpgradeLocked(version);
        }

        //【3】当 如果 version 不等于 3 或者发生了系统更新，那就需要更新本地文件中的 Fingerprint！
        if (version != CURRENT_VERSION || mNewUpdate) {
            try (BufferedWriter writer = new BufferedWriter(new FileWriter(mVersionFile))) {
                writer.write(Integer.toString(CURRENT_VERSION));
                writer.write("\n");
                writer.write(currentFingerprint);
                writer.write("\n");
                writer.flush();
            } catch (IOException e) {
                Slog.e(TAG, "Failed to write new version");
                throw new RuntimeException(e);
            }
        }
    }
```
这里我们不过多关注！

```java
    private static final int CURRENT_VERSION = 3;
```
我们可以看到 UsageStatsDatabase 中定义了 CURRENT_VERSION 值为 3，表示当前版本号！


#### 2.1.1.1 UsageStatsDatabase.doUpgradeLocked

处理数据库升级！
```java
    private void doUpgradeLocked(int thisVersion) {
        if (thisVersion < 2) {
            // 删除掉 version 小于 2 的数据库！
            Slog.i(TAG, "Deleting all usage stats files");
            for (int i = 0; i < mIntervalDirs.length; i++) {
                File[] files = mIntervalDirs[i].listFiles();
                if (files != null) {
                    for (File f : files) {
                        f.delete();
                    }
                }
            }
        }
    }
```
其实就是删除本地持久化文件！

### 2.1.2 UsageStatsDatabase.indexFilesLocked

indexFilesLocked 方法用于列出时间类别文件夹中的所有文件，并按照时间升序排序！

```java
    private void indexFilesLocked() {
        //【1】创建一个文件过滤器，过滤掉那些文件后缀是 .bak 的备份文件！
        final FilenameFilter backupFileFilter = new FilenameFilter() {
            @Override
            public boolean accept(File dir, String name) {
                return !name.endsWith(BAK_SUFFIX);
            }
        };

        //【2】遍历四个时间类别的文件夹，获得那个类别文件夹下，除去后缀为 .bak 的所有文件，
        for (int i = 0; i < mSortedStatFiles.length; i++) {
            if (mSortedStatFiles[i] == null) {
                mSortedStatFiles[i] = new TimeSparseArray<>();
            } else {
                mSortedStatFiles[i].clear();
            }
            File[] files = mIntervalDirs[i].listFiles(backupFileFilter);
            if (files != null) {
                if (DEBUG) {
                    Slog.d(TAG, "Found " + files.length + " stat files for interval " + i);
                }

                for (File f : files) {
                    final AtomicFile af = new AtomicFile(f);
                    try {
                        //【2.1.2.1】调用 UsageStatsXml.parseBeginTime 计算文件的创建时间，然后
                        // 通过 time —> AtomicFile 映射关系，保存到对应类别的 mSortedStatFiles[i] 中去！
                        mSortedStatFiles[i].put(UsageStatsXml.parseBeginTime(af), af);
                    } catch (IOException e) {
                        Slog.e(TAG, "failed to index file: " + f, e);
                    }
                }
            }
        }
    }
```

这里设置到了一个文件后缀：
```java
    private static final String BAK_SUFFIX = ".bak";
```
UsageStatsDatabase 会自动过滤掉 .bak 结尾的文件，因为 .bak 是备份文件！！

#### 2.1.2.1 UsageStatsXml.parseBeginTime

这里用到了 UsageStatsXml，他是用来专门解析 usage 文件的：
```java
    public static long parseBeginTime(AtomicFile file) throws IOException {
        return parseBeginTime(file.getBaseFile());
    }
```
继续调用：
```java
    public static long parseBeginTime(File file) throws IOException {
        String name = file.getName();

        //【1】如果文件名以 -c 结尾，返回去掉 -c 剩余的内容：
        while (name.endsWith(CHECKED_IN_SUFFIX)) { 
            name = name.substring(0, name.length() - CHECKED_IN_SUFFIX.length());
        }

        try {
            return Long.parseLong(name);
        } catch (NumberFormatException e) {
            throw new IOException(e);
        }
    }
```
这里我们可以看到，其实每一个时间类别的文件夹中的所有文件都是以其创建日期来命名的！

## 2.2 UsageStatsDatabase.getLatestUsageStats - 获得最新的使用信息

返回指定时间类别 intervalType 的最新的使用状态信息！
```java
    public IntervalStats getLatestUsageStats(int intervalType) {
        synchronized (mLock) {
            //【1】校验 intervalType 的取值范围！
            if (intervalType < 0 || intervalType >= mIntervalDirs.length) {
                throw new IllegalArgumentException("Bad interval type " + intervalType);
            }

            final int fileCount = mSortedStatFiles[intervalType].size();
            if (fileCount == 0) {
                return null;
            }

            try {
                //【2】因为 mSortedStatFiles[intervalType] 是按照时间顺序排序的，所以最新的状态信息文件
                // 一定是 fileCount - 1 对应的 AtomicFile!
                final AtomicFile f = mSortedStatFiles[intervalType].valueAt(fileCount - 1);
                IntervalStats stats = new IntervalStats();
                
                //【2.2.1.1】调用 UsageStatsXml.read 从本地文件中读取信息，初始化 IntervalStats 对象！
                UsageStatsXml.read(f, stats);
                return stats;
            } catch (IOException e) {
                Slog.e(TAG, "Failed to read usage stats file", e);
            }
        }
        return null;
    }
```

### 2.2.1 UsageStatsXml.read

read 读取文件！
```java
    public static void read(AtomicFile file, IntervalStats statsOut) throws IOException {
        try {
            FileInputStream in = file.openRead();
            try {
                //【2.1.2.1】调用 parseBeginTime 方法，初始化 statsOut.beginTime，可以看到
                // 文件名的时间戳，就是 statsOut.beginTime 的值！
                statsOut.beginTime = parseBeginTime(file);

                //【2.2.2】调用自身的 read 方法，继续解析；
                read(in, statsOut);
                // 获得文件上一次被更新的时间 statsOut.lastTimeSaved
                statsOut.lastTimeSaved = file.getLastModifiedTime();
            } finally {
                try {
                    in.close();
                } catch (IOException e) {
                    // Empty
                }
            }
        } catch (FileNotFoundException e) {
            Slog.e(TAG, "UsageStats Xml", e);
            throw e;
        }
    }
```
通过这个阶段，我们获得了：

```java
statsOut.beginTime; // 文件开始记录的时间
statsOut.lastTimeSaved; // 文件最新修改的时间
```
以上两个属性！

### 2.2.2 UsageStatsXml.read

继续解析：
```java
    static void read(InputStream in, IntervalStats statsOut) throws IOException {
        XmlPullParser parser = Xml.newPullParser();
        try {
            parser.setInput(in, "utf-8");
            XmlUtils.beginDocument(parser, USAGESTATS_TAG); // 解析 usagestats 标签
            String versionStr = parser.getAttributeValue(null, VERSION_ATTR); // 解析 version 属性
            try {
                switch (Integer.parseInt(versionStr)) {
                    case 1:
                        //【2.2.3】调用 UsageStatsXmlV1.read 继续解析
                        // 只有 version 置为 1 时才会继续解析；
                        UsageStatsXmlV1.read(parser, statsOut);
                        break;

                    default:
                        Slog.e(TAG, "Unrecognized version " + versionStr);
                        throw new IOException("Unrecognized version " + versionStr);
                }
            } catch (NumberFormatException e) {
                Slog.e(TAG, "Bad version");
                throw new IOException(e);
            }
        } catch (XmlPullParserException e) {
            Slog.e(TAG, "Failed to parse Xml", e);
            throw new IOException(e);
        }
    }
```
第二个 read 方法更像是 version 校验！

这里对应的数据是：

```xml
<usagestats version="1" endTime="62979335">
</usagestats>
```

### 2.2.3 UsageStatsXmlV1.read

最后，调用了 UsageStatsXmlV1 的 read 方法：

```java
    public static void read(XmlPullParser parser, IntervalStats statsOut)
            throws XmlPullParserException, IOException {
        //【1】清空 IntervalStats 的内部的集合！
        statsOut.packageStats.clear();
        statsOut.configurations.clear();
        statsOut.activeConfiguration = null;

        if (statsOut.events != null) {
            statsOut.events.clear();
        }
        //【2】计算 statsOut.endTime，等于 statsOut.beginTime 加上 endTime 的值！
        statsOut.endTime = statsOut.beginTime + XmlUtils.readLongAttribute(parser, END_TIME_ATTR);

        int eventCode;
        int outerDepth = parser.getDepth();
        //【3】接下来，根据使用信息的类别，进行不同的处理！
        while ((eventCode = parser.next()) != XmlPullParser.END_DOCUMENT
                && (eventCode != XmlPullParser.END_TAG || parser.getDepth() > outerDepth)) {
            if (eventCode != XmlPullParser.START_TAG) {
                continue;
            }

            final String tag = parser.getName();
            switch (tag) {
                case PACKAGE_TAG: // package 标签
                    //【2.2.3.1】如果该使用信息是 package 的，调用 loadUsageStats 解析！
                    loadUsageStats(parser, statsOut);
                    break;

                case CONFIG_TAG: // config 标签
                    //【2.2.3.2】如果该使用信息是 config 的，调用 loadConfigStats 解析！
                    loadConfigStats(parser, statsOut);
                    break;

                case EVENT_TAG: // event 标签
                    //【2.2.3.3】如果该使用信息是 event 的，调用 loadEvent 解析！
                    loadEvent(parser, statsOut);
                    break;
            }
        }
    }
```
该阶段，我们得到了如下的属性：

- statsOut.endTime，表示该使用信息文件记录的截至时间点，取值为 statsOut.beginTime + XmlUtils.readLongAttribute(parser, END_TIME_ATTR)；

```xml
<usagestats version="1" endTime="62979335">
</usagestats>
```
END_TIME_ATTR 对应着属性：endTime 属性，endTime 其实是一个时间间隔，距离 statsOut.beginTime 的时间间隔，statsOut.beginTime 和 statsOut.endTime 的时间段就是该文件能够记录的数据范围！

接着，解析 package，config 和 event 相关的使用信息！

#### 2.2.3.1 UsageStatsXmlV1.loadUsageStats - 读取 UsageStats

我们来看 package 标签和其属性：
```xml
    <packages>
        <package lastTimeActive="1104580" package="com.github.shadowsocks" timeActive="119959" lastEvent="2" />
    </packages>
```

下面看看 解析 package 的使用信息！

```java
    private static void loadUsageStats(XmlPullParser parser, IntervalStats statsOut)
            throws IOException {
        final String pkg = parser.getAttributeValue(null, PACKAGE_ATTR);
        if (pkg == null) {
            throw new ProtocolException("no " + PACKAGE_ATTR + " attribute present");
        }
        //【2.2.3.1.1】根据给定的 package，创建对应的 UsageStats！
        final UsageStats stats = statsOut.getOrCreateUsageStats(pkg);

        //【1】解析 lastTimeActive 属性，表示的是距离 beginTime 的时间间隔，
        // 初始化 stats.mLastTimeUsed，表示该 package 最后是使用的时间；
        stats.mLastTimeUsed = statsOut.beginTime + XmlUtils.readLongAttribute(
                parser, LAST_TIME_ACTIVE_ATTR);

        //【2】解析 timeActive 属性，获得 package 在前台的总时间，初始化 stats.mTotalTimeInForeground！
        stats.mTotalTimeInForeground = XmlUtils.readLongAttribute(parser, TOTAL_TIME_ACTIVE_ATTR);

        //【3】解析 lastEvent 属性，获得 package 在最后一次触发的 event，保存到 stats.mLastEvent 中！
        stats.mLastEvent = XmlUtils.readIntAttribute(parser, LAST_EVENT_ATTR);
    }
```
首先获得 pkg 对应的 UsageStats 对象，然后解析相关属性！

- lastTimeActive 属性用于计算最后一次处于 activie 的时间，它是一个距离 statsOut.beginTime 的时间间隔！

通过 lastTimeActive + statsOut.beginTime，就能够计算出上一次使用的时间 stats.mLastTimeUsed！

- timeActive 属性表示其在处于 active 总时间，用于计算 stats.mTotalTimeInForeground；

- lastEvent 属性表示上一次该 package 上报的时间类型，用于初始化 stats.mLastEvent；


##### 2.2.3.1.1 IntervalStats.getOrCreateUsageStats
```java
    UsageStats getOrCreateUsageStats(String packageName) {
        //【1】先从 packageStats 中获取 package 对应的 UsageStats！
        UsageStats usageStats = packageStats.get(packageName);
        if (usageStats == null) {
            //【2】创建一个 UsageStats 对象！
            usageStats = new UsageStats();
            
            // 获得包名
            usageStats.mPackageName = getCachedStringRef(packageName);

            //【3】很显然，该 package 的 usageStats.mBeginTimeStamp 和 usageStats.mEndTimeStamp 
            // 和其所属的 IntervalStats 是一样的！
            usageStats.mBeginTimeStamp = beginTime;
            usageStats.mEndTimeStamp = endTime;

            //【4】将其添加到 packageStats 中去！
            packageStats.put(usageStats.mPackageName, usageStats);
        }
        // 返回
        return usageStats;
    }
```
我们知道 IntervalStats.packageStats 中保存的是应用的使用信息！

getCachedStringRef 优先从 IntervalStats.mStringCache 内部缓存中获取！

- 创建了一个 UsageStats 对象，封装该 package 的使用信息！

- 计算 usageStats.mBeginTimeStamp 和 usageStats.mEndTimeStamp，等于所属 IntervalStats.beginTime 和 IntervalStats.beginTime！

- 将新创建的 UsageStats 添加到 packageStats 中！

#### 2.2.3.2 UsageStatsXmlV1.loadConfigStats - 读取 ConfigStats

我们来看看 config 的使用信息：
```java
    <configurations>
        <config lastTimeActive="60709298" timeActive="60690625" count="3" active="true"
                fs="1065353216" mcc="460" mnc="65535" 
				locales="zh-CN" touch="3" key="1" keyHid="1"
				hardKeyHid="2" nav="1" navHid="2" ori="1" scrLay="268435810" 
				clrMod="5" ui="17" width="360" height="685" sw="360"
				density="480" app_bounds="0 0 1080 2136" />
    </configurations>
```

下面我们来出解析 Configuration 的使用信息！
```java
    private static void loadConfigStats(XmlPullParser parser, IntervalStats statsOut)
            throws XmlPullParserException, IOException {
        //【1】创建了一个 Configuration 对象，并 readXmlAttrs 解析和配置文件相关的信息！
        final Configuration config = new Configuration();
        Configuration.readXmlAttrs(parser, config);

        //【2.2.3.1.1】根据给定的 Configuration，创建对应的 ConfigurationStats！
        final ConfigurationStats configStats = statsOut.getOrCreateConfigurationStats(config);

        //【1】解析 lastTimeActive 属性，表示的是距离 beginTime 的时间间隔，
        // 初始化 stats.mLastTimeUsed，表示该 package 最后是使用的时间；
        configStats.mLastTimeActive = statsOut.beginTime + XmlUtils.readLongAttribute(
                parser, LAST_TIME_ACTIVE_ATTR);
                
        //【2】解析 timeActive 属性，获得 Configuration 活跃的总时间，初始化 stats.mTotalTimeActive！
        configStats.mTotalTimeActive = XmlUtils.readLongAttribute(parser, TOTAL_TIME_ACTIVE_ATTR);
        //【3】解析 count 属性，获得 Configuration 活跃的次数，初始化 stats.mActivationCount！
        configStats.mActivationCount = XmlUtils.readIntAttribute(parser, COUNT_ATTR);
        
        //【4】解析 active 属性，判断该 Configuration 是否是活跃状态！
        // 初始化 statsOut.activeConfiguration！
        if (XmlUtils.readBooleanAttribute(parser, ACTIVE_ATTR)) {
            statsOut.activeConfiguration = configStats.mConfiguration;
        }
    }
```

对于 Configuration，由于其属性配置很多，所以这里我们只关注和 UsageStats 相关的属性！

创建了一个 Configuration 对象，表示该配置信息对象，用于保存配置相关的属性！

创建该 config 对象的 ConfigurationStats 对象！

##### 2.2.3.2.1 IntervalStats.getOrCreateConfigurationStats

获取或者创建 Configuration 对应的 ConfigurationStats 对象！
```java
    ConfigurationStats getOrCreateConfigurationStats(Configuration config) {
        //【1】先从 configurations 中获取 package 对应的 ConfigurationStats！
        ConfigurationStats configStats = configurations.get(config);
        if (configStats == null) {
            //【2】如果没有，就创建一个 ConfigurationStats 对象！
            configStats = new ConfigurationStats();
            //【3】很显然，该 package 的 usageStats.mBeginTimeStamp 和 usageStats.mEndTimeStamp 
            // 和其所属的 IntervalStats 是一样的！
            configStats.mBeginTimeStamp = beginTime;
            configStats.mEndTimeStamp = endTime;
            //【4】设置 configStats.mConfiguration；
            configStats.mConfiguration = config;
            //【5】将其添加到 configurations 中！
            configurations.put(config, configStats);
        }
        // 返回
        return configStats;
    }
```

#### 2.2.3.3 UsageStatsXmlV1.loadEvent - 读取 Event

我们来看看 event 相关的数据

```java
    <event-log>
        <event time="5852614" package="com.tencent.mobileqq"
               class="com.tencent.mobileqq.activity.QQLSActivity" flags="0" type="1" />
        <event time="5852617" package="com.tencent.mobileqq" 
               class="com.tencent.mobileqq.activity.QQLSActivity" flags="0" type="2" />
    </event-log>
```
注意：只有 daily 类别的文件才有 event！

解析 UsageEvents 的使用信息！
```jav
    private static void loadEvent(XmlPullParser parser, IntervalStats statsOut)
            throws XmlPullParserException, IOException {
        //【1】解析 package 属性，获得该 event 所属的 pacakge！
        final String packageName = XmlUtils.readStringAttribute(parser, PACKAGE_ATTR);
        if (packageName == null) {
            throw new ProtocolException("no " + PACKAGE_ATTR + " attribute present");
        }
        //【2】解析 class 属性，获得该 event 所属的 pacakge
        final String className = XmlUtils.readStringAttribute(parser, CLASS_ATTR);

        //【2.2.3.3.1】根据给定的 packageName 和 className，创建对应的 Event！
        final UsageEvents.Event event = statsOut.buildEvent(packageName, className);

        //【3】解析 lastTimeActive 属性，表示的是距离 statsOut.beginTime 的时间间隔，
        // 初始化 event.mTimeStamp，表示该 event 的上报时间；
        event.mTimeStamp = statsOut.beginTime + XmlUtils.readLongAttribute(parser, TIME_ATTR);
        
        //【4】解析 type 属性，初始化 event.mEventType，表示该 event 的类型；
        event.mEventType = XmlUtils.readIntAttribute(parser, TYPE_ATTR);
        
        //【5】如果 event type 类型为 CONFIGURATION_CHANGE 或者 SHORTCUT_INVOCATION
        // 还要解析其对应的 Configuration 和 shortcutId 属性！
        switch (event.mEventType) {
            case UsageEvents.Event.CONFIGURATION_CHANGE:
                event.mConfiguration = new Configuration();
                Configuration.readXmlAttrs(parser, event.mConfiguration);
                break;
    
            case UsageEvents.Event.SHORTCUT_INVOCATION:
                final String id = XmlUtils.readStringAttribute(parser, SHORTCUT_ID_ATTR);
                event.mShortcutId = (id != null) ? id.intern() : null;
                break;
        }

        if (statsOut.events == null) { // 如果 IntervalStats.events 为 null，初始化！
            statsOut.events = new TimeSparseArray<>();
        }
        
        //【6】将该 UsageEvents.Event 添加到 IntervalStats.events 中！
        statsOut.events.put(event.mTimeStamp, event);
    }
```
##### 2.2.3.3.1 IntervalStats.buildEvent
```java
    UsageEvents.Event buildEvent(String packageName, String className) {
        //【1】创建一个 UsageEvents.Event 对象！
        UsageEvents.Event event = new UsageEvents.Event();
        //【2】初始化 event.mPackage 和 event.mClass 属性！
        event.mPackage = getCachedStringRef(packageName);
        if (className != null) {
            event.mClass = getCachedStringRef(className);
        }
        return event;
    }
```
该方法只是创建 UsageEvents.Event 对象，但是其并没有将其添加到对应的集合中！

## 2.3 UserUsageStatsService.loadActiveStats

loadActiveStats 用于给没有最新使用信息的时间类别创建新的 IntervalStats！

```java
    private void loadActiveStats(final long currentTimeMillis) {
        //【1】mCurrentStats 用于保存每个时间类别对应的最新的使用信息，这里开始遍历时间类别！
        for (int intervalType = 0; intervalType < mCurrentStats.length; intervalType++) {
        
            //【2.2】通过 getLatestUsageStats 方法获得时间类别的最新使用信息！
            final IntervalStats stats = mDatabase.getLatestUsageStats(intervalType);
            if (stats != null && currentTimeMillis - 500 >= stats.endTime &&
                    currentTimeMillis < stats.beginTime + INTERVAL_LENGTH[intervalType]) {
                //【2】判断下当前时间是否在时间范围之内，如果是，该 IntervalStats 依然可以记录使用信息！
                if (DEBUG) {
                    Slog.d(TAG, mLogPrefix + "Loading existing stats @ " +
                            sDateFormat.format(stats.beginTime) + "(" + stats.beginTime +
                            ") for interval " + intervalType);
                }
                mCurrentStats[intervalType] = stats;
            } else {
                //【3】当前时间已经超过最新的 IntervalStats 能够记录的时间范围！
                // 或者某个时间类别没有对应的使用信息！
                if (DEBUG) {
                    Slog.d(TAG, "Creating new stats @ " +
                            sDateFormat.format(currentTimeMillis) + "(" +
                            currentTimeMillis + ") for interval " + intervalType);
                }
                
                // 创建一个新的 IntervalStats 对象，初始化 beginTime 为当前时间，endTime 为当前时间 + 1
                mCurrentStats[intervalType] = new IntervalStats();
                mCurrentStats[intervalType].beginTime = currentTimeMillis;
                mCurrentStats[intervalType].endTime = currentTimeMillis + 1;
            }
        }
        //【2.3】将 mStatsChanged 置为 false；
        mStatsChanged = false;

        //【2.4】更新回滚时间！
        updateRolloverDeadline();

        //【×2.3.1】通知 UsageStatsService 该 userId 的最新使用信息重新加载了！
        mListener.onStatsReloaded();
    }
```

UserUsageStatsService 内部有一个数组，用于表示每个时间类别对应的时间间隔长度：
```java
    private static final long[] INTERVAL_LENGTH = new long[] {
            UnixCalendar.DAY_IN_MILLIS, UnixCalendar.WEEK_IN_MILLIS,
            UnixCalendar.MONTH_IN_MILLIS, UnixCalendar.YEAR_IN_MILLIS
    };
```
UnixCalendar 我们在 1.1 有分析过，这里就不在多说了！

举个简单的例子，比如我们指定的时间类别是 daily，那么该 daily 类别下的最新数据要满足要求，当前时间必满足以下条件：

```java
currentTimeMillis >= stats.endTime + 500

currentTimeMillis < stats.beginTime + 24 * 60 * 60 * 1000
```
- stats.beginTime 表示的是该 IntervalStats 的创建时间，那么其最多能记录的信息是：stats.beginTime + 24 * 60 * 60 * 1000 之前的信息！

- stats.endTime 表示的是该 IntervalStats 的目前已经记录的时间，那么如果该使用信息能够继续被更新，则有 currentTimeMillis - 500 >= stats.endTime！

其他类别的信息是同样的道理，所以 [stats.endTime + 500, stats.beginTime + 24 * 60 * 60 * 1000) 就是该 IntervalStats 还可以记录的时间范围！



### 2.3.1 UsageStatsService.onStatsReloaded - 回调

```java
    @Override
    public void onStatsReloaded() {
        postOneTimeCheckIdleStates();
    }
```
loadActiveStats 方法根据当前时间，重新加载最新的数据，然后触发 UsageStatsService.onStatsReloaded 回调！

该方法会调用 postOneTimeCheckIdleStates 检查一次 idle 状态！


## 2.4 UserUsageStatsService.updateRolloverDeadline

更新回滚时间！
```java
    private void updateRolloverDeadline() {
        //【1】首先设置 mDailyExpiryDate 为 daily 时间类别下的最新日期的使用信息的开始记录时间点
        mDailyExpiryDate.setTimeInMillis(
                mCurrentStats[UsageStatsManager.INTERVAL_DAILY].beginTime);
        //【2】然后再加一天时间！
        mDailyExpiryDate.addDays(1);
        Slog.i(TAG, mLogPrefix + "Rollover scheduled @ " +
                sDateFormat.format(mDailyExpiryDate.getTimeInMillis()) + "(" +
                mDailyExpiryDate.getTimeInMillis() + ")");
    }
```
updateRolloverDeadline 方法中，将 mDailyExpiryDate 先设置为了 mCurrentStats[UsageStatsManager.INTERVAL_DAILY].beginTime，然后再加了一天！

所以 updateRolloverDeadline 计算的是下一天使用信息的开始记录时间！mDailyExpiryDate 可以看作是否超过一天的临界时间点！


## 2.5 IntervalStats.update

我们看到，在 init 方法中：
```java
    if (pkgStats.mLastEvent == UsageEvents.Event.MOVE_TO_FOREGROUND ||
            pkgStats.mLastEvent == UsageEvents.Event.CONTINUE_PREVIOUS_DAY) {
        ... ... ...
    }
```
只有满足以上条件，才会进入 IntervalStats.update 方法中！

update 用于跟更新 packageName 对应的 UsageStats 相关属性！

- **long timeStamp 传入的是 IntervalStats.lastTimeSaved，表示 IntervalStats 最后被修改/更新的时间点**；
- int eventType 传入的是 UsageEvents.Event.END_OF_DAY，表示本次要设置的 eventType；

```java
    void update(String packageName, long timeStamp, int eventType) {
        //【2.2.3.1.1】获得该 packageName 对应的 UsageStats 实例！
        UsageStats usageStats = getOrCreateUsageStats(packageName);

        //【1】如果本次的 eventType 为  MOVE_TO_BACKGROUND 或者 END_OF_DAY，进入这个 if 条件！
        if (eventType == UsageEvents.Event.MOVE_TO_BACKGROUND ||
                eventType == UsageEvents.Event.END_OF_DAY) {

            //【1.1】并且 UsageStats.mLastEvent 的取值为 MOVE_TO_FOREGROUND 或者 CONTINUE_PREVIOUS_DAY
            // 的话，才修改 package 的 mTotalTimeInForeground！
            if (usageStats.mLastEvent == UsageEvents.Event.MOVE_TO_FOREGROUND ||
                    usageStats.mLastEvent == UsageEvents.Event.CONTINUE_PREVIOUS_DAY) {
    
                //【1.1.1】更新 package 在前台的总时间 mTotalTimeInForeground 为：
                // 在已有基础上再加上 timeStamp  - usageStats.mLastTimeUsed
                usageStats.mTotalTimeInForeground += timeStamp - usageStats.mLastTimeUsed;
            }
        }

        //【2.5.1】如果 eventType 属于 Stateful Event
        // 那就更新该 pacakge 的 usageStats.mLastEvent（上一次事件）为 eventType；
        if (isStatefulEvent(eventType)) {
            usageStats.mLastEvent = eventType;
        }

        //【3】如果 eventType 不是 UsageEvents.Event.SYSTEM_INTERACTION，更新上次使用时间 
        // usageStats.mLastTimeUsed 为 timeStamp！
        // 因为 SYSTEM_INTERACTION 本质上是系统对 package 的操作，不会统计进去！
        if (eventType != UsageEvents.Event.SYSTEM_INTERACTION) {
            usageStats.mLastTimeUsed = timeStamp;
        }
        
        //【4】更新该 package 的 usageStats.mEndTimeStamp 为 timeStamp；
        usageStats.mEndTimeStamp = timeStamp;

        //【5】如果 eventType 为 MOVE_TO_FOREGROUND，该 package 的 usageStats.mLaunchCount（启动次数）加一；
        if (eventType == UsageEvents.Event.MOVE_TO_FOREGROUND) {
            usageStats.mLaunchCount += 1;
        }

        //【6】同时修改 IntervalStats.endTime 也为 timeStamp；
        endTime = timeStamp;
    }
```
整个过程如下：

- 获得该 package 的 UsageStats 对象！

</br>

- 如果本次要更新的 eventType 为  **MOVE_TO_BACKGROUND** / **END_OF_DAY**，并且该 package 的 LastEvent 为 **MOVE_TO_FOREGROUND** / **CONTINUE_PREVIOUS_DAY**
    - 这种情况是 package 从前台退到了后台，
    - 这是我们会更新 package 的在前台的总时间 mTotalTimeInForeground = mTotalTimeInForeground + timeStamp - mLastTimeUsed；

</br>

- 如果本次要更新的 eventType 为 Stateful Event，我们也会更新 package 的 mLastEvent 为 eventType；

</br>

- 如果本次要更新的 eventType 不是 SYSTEM_INTERACTION，那么我们会更新 package 的 mLastTimeUsed 为 timeStamp；

</br>

- 更新 package 的 mEndTimeStamp 为 timeStamp；同时更新对应的 IntervalStats 的 endTime 也为 timeStamp；

</br>

- 如果本次要更新的 eventType 不是 SYSTEM_INTERACTION，那么我们会更新 package 的 mLastTimeUsed 为 timeStamp；同时更新对应的 IntervalStats 的 endTime 也为 timeStamp；

### 2.5.1 IntervalStats.isStatefulEvent
```java
    private boolean isStatefulEvent(int eventType) {
        switch (eventType) {
            case UsageEvents.Event.MOVE_TO_FOREGROUND:
            case UsageEvents.Event.MOVE_TO_BACKGROUND:
            case UsageEvents.Event.END_OF_DAY:
            case UsageEvents.Event.CONTINUE_PREVIOUS_DAY:
                return true;
        }
        return false;
    }
```
判断 eventType 是否是 stateful event！

## 2.6 UserUsageStatsService.notifyStatsChanged

```java
    private void notifyStatsChanged() {
        //【1】只有 mStatsChanged 为 false 时，才通知 UsageStatsService！
        if (!mStatsChanged) {
            //【1.1】设置 mStatsChanged 为 true；
            mStatsChanged = true;
            //【1.2】通知 UsageStatsService，IntervalStats 数据发生了变化！
            mListener.onStatsUpdated();
        }
    }
```
mStatsChanged 表示的是使用信息是否发生变化！

当 IntervalStats 的数据发生变化后，会触发 notifyStatsChanged 方法！

### 2.6.1 UsageStatsService.onStatsUpdated - 回调
```java
    @Override
    public void onStatsUpdated() {
        mHandler.sendEmptyMessageDelayed(MSG_FLUSH_TO_DISK, FLUSH_INTERVAL);
    }
```
UsageStatsService 的 onStatsUpdated 方法触发后，会延迟 20mins 后发送 MSG_FLUSH_TO_DISK 给 H！

这样也保证了不会频繁的通知和保存！！

## 2.7 IntervalStats.updateConfigurationStats

更新指定配置的信息！

- Configuration confi： 是要成为 active config 的配置对象，这里传入的是 null；
- long timeStamp：传入的是 stat.lastTimeSaved（IntervalStats 自身上一次被更新的时间）；

```java
    void updateConfigurationStats(Configuration config, long timeStamp) {
        //【1】如果 activeConfiguration 不为 null，说明已经有配置信息处于 active 状态！
        // 那就更新其时间信息！
        if (activeConfiguration != null) {
            ConfigurationStats activeStats = configurations.get(activeConfiguration);
            //【1.1】更新 active config 总的活跃时间 mTotalTimeActive 为：在其基础上加上
            // timeStamp 减去上一次活跃时间（mLastTimeActive）
            activeStats.mTotalTimeActive += timeStamp - activeStats.mLastTimeActive;

            //【1.2】更新上次活跃时间 mLastTimeActive 为上次被更新的时间（timeStamp）减去 1；
            activeStats.mLastTimeActive = timeStamp - 1;
        }

        //【2】更新 config 指定的配置的信息。因为我们这里传入的是 null，所以不会进入；
        if (config != null) {
            ConfigurationStats configStats = getOrCreateConfigurationStats(config);
            //【2.1】更新 config 上次活跃时间 mLastTimeActive 为 timeStamp
            configStats.mLastTimeActive = timeStamp;
            //【2.2】更新 config 的活跃次数；
            configStats.mActivationCount += 1;
            
            //【2.3】更新 IntervalStats.activeConfiguration 为指定的 config！
            activeConfiguration = configStats.mConfiguration;
        }
        //【3】更新 IntervalStats.endTime 为上一次被更新的时间 ！timeStamp
        endTime = timeStamp;
    }
```
因为这里我们传入的是 Configuration config 为 null，所以只会尝试更新 activeConfiguration 的时间信息！

## 2.8 UserUsageStatsService.notifyNewUpdate

当系统发生了升级后，UserUsageStatsService 会调用 notifyNewUpdate 通知 UsageStatsService!
```java
    private void notifyNewUpdate() {
        //【*2.8.1】这里的 mListener 就是 UsageStatsService！
        mListener.onNewUpdate(mUserId);
    }
```

这里就不多说了！

### 2.8.1 UsageStatsService.onNewUpdate - 回调
```java
    @Override
    public void onNewUpdate(int userId) {
        initializeDefaultsForSystemApps(userId);
    }
```
如果出现了系统升级的情况，那么 UsageStatsService 的 onNewUpdate 方法会调用，然后执行 initializeDefaultsForSystemApps 方法，对系统 App 的信息做初始化！

## 2.9 流程总结

# 3 UserUsageStatsService.onTimeChanged - 处理时间变化

处理时间改变的情况！

```java
    void onTimeChanged(long oldTime, long newTime) {
        //【×3.1】持久化处于内存中的最新数据！
        persistActiveStats();
        //【×3.2】通知数据库时间发生了变化！
        mDatabase.onTimeChanged(newTime - oldTime);
        //【×2.3】再次加载最新日期的数据！
        loadActiveStats(newTime);
    }
```
整个方法的逻辑如下：

- 先将内存中的数据持久化到本地文件中；
- 然后处理下时间变化；
- 再次加载本地数据到内存中；


对于 loadActiveStats 方法，我们在前面是有分析过的，该过程的主要逻辑如下：

- 先将最新日期的内存数据写回本地持久化文件；
- 处理时间变化后，持久化文件的名称修改，然后对改名后的文件重新排序，加载到内存中；
- 重新加载最新日期的使用数据！

## 3.0 调用时机

在 UsageStatsService 的 checkAndGetTimeLocked 会计算当前的实际时间，判断是否有调时发生！

如果有的话，会触发 UserUsageStatsService.onTimeChanged 方法！

```java
    private long checkAndGetTimeLocked() {
        final long actualSystemTime = System.currentTimeMillis();
        final long actualRealtime = SystemClock.elapsedRealtime();
        final long expectedSystemTime = (actualRealtime - mRealTimeSnapshot) + mSystemTimeSnapshot;
        final long diffSystemTime = actualSystemTime - expectedSystemTime;
        if (Math.abs(diffSystemTime) > TIME_CHANGE_THRESHOLD_MILLIS) {
            // The time has changed.
            Slog.i(TAG, "Time changed in UsageStats by " + (diffSystemTime / 1000) + " seconds");
            final int userCount = mUserState.size();
            for (int i = 0; i < userCount; i++) {
                final UserUsageStatsService service = mUserState.valueAt(i);
                //【×3】会触发 onTimeChanged 方法！
                service.onTimeChanged(expectedSystemTime, actualSystemTime);
            }
            mRealTimeSnapshot = actualRealtime;
            mSystemTimeSnapshot = actualSystemTime;
        }
        return actualSystemTime;
    }
```
对于 checkAndGetTimeLocked 的逻辑，这里不再多说！

## 3.1 UserUsageStatsService.persistActiveStats

persistActiveStats 将最新的缓存数据保存到本地文件中！

```java
    void persistActiveStats() {
        //【1】只有当 mStatsChanged 为 true 是才会更新本地文件！
        if (mStatsChanged) {
            Slog.i(TAG, mLogPrefix + "Flushing usage stats to disk");
            try {
                //【×3.1.1】更新数据到本地文件！
                for (int i = 0; i < mCurrentStats.length; i++) {
                    mDatabase.putUsageStats(i, mCurrentStats[i]);
                }
                //【2】更新完成后会将 mStatsChanged 置为 false！
                mStatsChanged = false;
            } catch (IOException e) {
                Slog.e(TAG, mLogPrefix + "Failed to persist active stats", e);
            }
        }
    }
```
执行 persistActiveStats 前，会先做一次判断，如果时间发生了变化，并且内存中的使用信息有更新过，即 mStatsChanged 为 true，才会触发回写本地文件的操作！

mCurrentStats 数组中保存的是当前最新的数据！

### 3.1.1 UsageStatsDatabase.putUsageStats

```java
    public void putUsageStats(int intervalType, IntervalStats stats) throws IOException {
        if (stats == null) return;
        synchronized (mLock) {
            //【1】校验 intervalType 的取值范围！
            if (intervalType < 0 || intervalType >= mIntervalDirs.length) {
                throw new IllegalArgumentException("Bad interval type " + intervalType);
            }
            //【2】获得 IntervalStats 对应的本地文件 AtomicFile 对象！
            // 如果没有找到，就创建一个新的 AtomicFile，并添加到 mSortedStatFiles 中！
            AtomicFile f = mSortedStatFiles[intervalType].get(stats.beginTime);
            if (f == null) {
                f = new AtomicFile(new File(mIntervalDirs[intervalType],
                        Long.toString(stats.beginTime)));
                mSortedStatFiles[intervalType].put(stats.beginTime, f);
            }
            //【3.1.2】调用 UsageStatsXml 的 write 方法将数据写到本地文件中！
            UsageStatsXml.write(f, stats);

            //【4】更新 stats.lastTimeSaved
            stats.lastTimeSaved = f.getLastModifiedTime();
        }
    }
```

### 3.1.2 UsageStatsXml.write

```java
    public static void write(AtomicFile file, IntervalStats stats) throws IOException {
        FileOutputStream fos = file.startWrite();
        try {
            // 调用另外一个 write 方法！
            write(fos, stats);
            file.finishWrite(fos);
            fos = null;
        } finally {
            // When fos is null (successful write), this will no-op
            file.failWrite(fos);
        }
    }
```
调用了另外一个 write 方法：
```java
    static void write(OutputStream out, IntervalStats stats) throws IOException {
        FastXmlSerializer xml = new FastXmlSerializer();
        xml.setOutput(out, "utf-8");
        xml.startDocument("utf-8", true);
        xml.setFeature("http://xmlpull.org/v1/doc/features.html#indent-output", true);
        xml.startTag(null, USAGESTATS_TAG); // 处理 usagestats 标签
        xml.attribute(null, VERSION_ATTR, Integer.toString(CURRENT_VERSION)); // 处理 version 属性，值为 1；
        // 继续调用 UsageStatsXmlV1.write 方法写入！
        UsageStatsXmlV1.write(xml, stats);

        xml.endTag(null, USAGESTATS_TAG);
        xml.endDocument();
    }
```

### 3.1.3 UsageStatsXmlV1.write

该方法最终写入本地文件！

```java
    public static void write(XmlSerializer xml, IntervalStats stats) throws IOException {
        XmlUtils.writeLongAttribute(xml, END_TIME_ATTR, stats.endTime - stats.beginTime);

        xml.startTag(null, PACKAGES_TAG); // 写入 packages 标签
        final int statsCount = stats.packageStats.size();
        for (int i = 0; i < statsCount; i++) {
            //【3.1.3.1】写入 package 的使用信息；
            writeUsageStats(xml, stats, stats.packageStats.valueAt(i));
        }
        xml.endTag(null, PACKAGES_TAG);

        xml.startTag(null, CONFIGURATIONS_TAG); // 写入 configurations 标签
        final int configCount = stats.configurations.size();
        for (int i = 0; i < configCount; i++) {
            //【3.1.3.2】写入 Configuration 的使用信息；
            boolean active = stats.activeConfiguration.equals(stats.configurations.keyAt(i));
            writeConfigStats(xml, stats, stats.configurations.valueAt(i), active);
        }
        xml.endTag(null, CONFIGURATIONS_TAG);

        xml.startTag(null, EVENT_LOG_TAG); // 写入 event-log 标签
        final int eventCount = stats.events != null ? stats.events.size() : 0;
        for (int i = 0; i < eventCount; i++) {
             //【3.1.3.3】写入 Event 的上报信息；
            writeEvent(xml, stats, stats.events.valueAt(i));
        }
        xml.endTag(null, EVENT_LOG_TAG);
    }
```
该过程主要是写入 package，Configuration，Event 的相关信息！

#### 3.1.3.1 UsageStatsXmlV1.writeUsageStats

IntervalStats stats 是使用信息记录文件的缓存对象；UsageStats usageStats 是该 package 的缓存对象！
```java
    private static void writeUsageStats(XmlSerializer xml, final IntervalStats stats,
            final UsageStats usageStats) throws IOException {
        xml.startTag(null, PACKAGE_TAG); // 写入 package 标签；

        // 写入 lastTimeActive 属性，取值为 usageStats.mLastTimeUsed - stats.beginTime；
        XmlUtils.writeLongAttribute(xml, LAST_TIME_ACTIVE_ATTR,
                usageStats.mLastTimeUsed - stats.beginTime);
        // 写入 package 属性，取值为 usageStats.mPackageName；
        XmlUtils.writeStringAttribute(xml, PACKAGE_ATTR, usageStats.mPackageName);
        // 写入 timeActive 属性，取值为 usageStats.mTotalTimeInForeground；
        XmlUtils.writeLongAttribute(xml, TOTAL_TIME_ACTIVE_ATTR, usageStats.mTotalTimeInForeground);
        // 写入 lastEvent 属性，取值为 usageStats.mLastEvent；
        XmlUtils.writeIntAttribute(xml, LAST_EVENT_ATTR, usageStats.mLastEvent);

        xml.endTag(null, PACKAGE_TAG);
    }
```
写入 package 的使用信息：

- 写入 lastTimeActive 属性，取值为 usageStats.mLastTimeUsed - stats.beginTime；
- 写入 package 属性，取值为 usageStats.mPackageName；
- 写入 timeActive 属性，取值为 usageStats.mTotalTimeInForeground；
- 写入 lastEvent 属性，取值为 usageStats.mLastEvent；

#### 3.1.3.2 UsageStatsXmlV1.writeConfigStats

IntervalStats stats 是使用信息记录文件的缓存对象；ConfigurationStats configStats 是该 config 的缓存对象，boolean isActive 表示该 config 是否是处于活跃状态！！
```java
    private static void writeConfigStats(XmlSerializer xml, final IntervalStats stats,
            final ConfigurationStats configStats, boolean isActive) throws IOException {
        xml.startTag(null, CONFIG_TAG); // 写入 configurations 标签

        // 写入 lastTimeActive 属性，取值为 usageStats.mLastTimeUsed - stats.beginTime；
        XmlUtils.writeLongAttribute(xml, LAST_TIME_ACTIVE_ATTR,
                configStats.mLastTimeActive - stats.beginTime);
        // 写入 timeActive 属性，取值为 configStats.mTotalTimeActive；
        XmlUtils.writeLongAttribute(xml, TOTAL_TIME_ACTIVE_ATTR, configStats.mTotalTimeActive);
        // 写入 count 属性，取值为 configStats.mActivationCount；
        XmlUtils.writeIntAttribute(xml, COUNT_ATTR, configStats.mActivationCount);
        // 如果该 config 是当前处于 active 状态，写入 active 属性，true；
        if (isActive) {
            XmlUtils.writeBooleanAttribute(xml, ACTIVE_ATTR, true);
        }

        // 接着就是写入 Configuration 的配置信息了！
        Configuration.writeXmlAttrs(xml, configStats.mConfiguration);

        xml.endTag(null, CONFIG_TAG);
    }
```
写入 Config 的使用信息：

- 写入 lastTimeActive 属性，取值为 usageStats.mLastTimeUsed - stats.beginTime；
- 写入 timeActive 属性，取值为 configStats.mTotalTimeActive；
- 写入 count 属性，取值为 configStats.mActivationCount；
- 如果该 config 是当前处于 active 状态，写入 active 属性，true；
- 接着就是写入 Configuration 的配置信息了！

#### 3.1.3.3 UsageStatsXmlV1.writeEvent

IntervalStats stats 是使用信息记录文件的缓存对象；UsageEvents.Event event 是该 event 的缓存对象！！
```java
    private static void writeEvent(XmlSerializer xml, final IntervalStats stats,
            final UsageEvents.Event event) throws IOException {
        xml.startTag(null, EVENT_TAG); // 写入 event 标签

        // 写入 time 属性，取值为 event.mTimeStamp - stats.beginTime；
        XmlUtils.writeLongAttribute(xml, TIME_ATTR, event.mTimeStamp - stats.beginTime);
        // 写入 package 属性和 class 属性，取值为 event.mPackage 和 event.mClass！
        XmlUtils.writeStringAttribute(xml, PACKAGE_ATTR, event.mPackage);
        if (event.mClass != null) {
            XmlUtils.writeStringAttribute(xml, CLASS_ATTR, event.mClass);
        }
        // 写入 type 属性，取值为 event.mEventType！
        XmlUtils.writeIntAttribute(xml, TYPE_ATTR, event.mEventType);
        // 根据 EventType，写入额外的信息！
        switch (event.mEventType) {
            case UsageEvents.Event.CONFIGURATION_CHANGE:
                if (event.mConfiguration != null) {
                    Configuration.writeXmlAttrs(xml, event.mConfiguration);
                }
                break;
            case UsageEvents.Event.SHORTCUT_INVOCATION:
                if (event.mShortcutId != null) {
                    XmlUtils.writeStringAttribute(xml, SHORTCUT_ID_ATTR, event.mShortcutId);
                }
                break;
        }

        xml.endTag(null, EVENT_TAG);
    }
```
写入 Event 的使用信息：

- 写入 time 属性，取值为 event.mTimeStamp - stats.beginTime；
- 写入 package 属性和 class 属性，取值为 event.mPackage 和 event.mClass！
- 写入 type 属性，取值为 event.mEventType！
- 根据 EventType，写入额外的信息！
    -  Configuration 或者 SHORTCUT！

## 3.2 UsageStatsDatabase.onTimeChanged

long timeDiffMillis 表示时间调整后的差值，因为时间调整了，所以我们同步修改文件的名称，同时删除那些无效的文件！

```java
    public void onTimeChanged(long timeDiffMillis) {
        synchronized (mLock) {
            StringBuilder logBuilder = new StringBuilder();
            logBuilder.append("Time changed by ");
            TimeUtils.formatDuration(timeDiffMillis, logBuilder);
            logBuilder.append(".");

            int filesDeleted = 0;
            int filesMoved = 0;

            //【1】处理 mSortedStatFiles 中所有的 AtomicFile 对象！
            for (TimeSparseArray<AtomicFile> files : mSortedStatFiles) {
                final int fileCount = files.size();
                for (int i = 0; i < fileCount; i++) {
                    //【1.1】通过 valueAt 获得要处理的 AtomicFile 文件，通过 keyAt 可以获得该文件的开始记录时间！
                    final AtomicFile file = files.valueAt(i);
                    //【1.2】计算时间改变后的新时间！
                    final long newTime = files.keyAt(i) + timeDiffMillis;
                    if (newTime < 0) {
                        //【1.2.1】如果 newTime 小于 0，那么该文件无效，删除该文件！
                        filesDeleted++;
                        file.delete();

                    } else {
                        //【1.2.2】这种情况，我们要修改文件的名字，因为文件的名字就是其开始记录时间，
                        // 我们会将文件名改为：newTime + "-c" 的形式！
                        try {
                            file.openRead().close();
                        } catch (IOException e) {
                            // Ignore, this is just to make sure there are no backups.
                        }

                        String newName = Long.toString(newTime);
                        if (file.getBaseFile().getName().endsWith(CHECKED_IN_SUFFIX)) {
                            newName = newName + CHECKED_IN_SUFFIX;
                        }

                        final File newFile = new File(file.getBaseFile().getParentFile(), newName);
                        filesMoved++;
                        file.getBaseFile().renameTo(newFile);
                    }
                }
                files.clear();
            }

            logBuilder.append(" files deleted: ").append(filesDeleted);
            logBuilder.append(" files moved: ").append(filesMoved);
            Slog.i(TAG, logBuilder.toString());

            //【×2.1.2】然后再次调用 indexFilesLocked，对改名后的文件进行重新排序，再次添加到 mSortedStatFiles 中！
            indexFilesLocked();
        }
    }
```
indexFilesLocked 方法我们有讲过，这里就不多说了！


## 3.3 流程总结

# 4 UserUsageStatsService.rolloverStats - 数据回滚

当我们在 report event 的时候，会判断此时记录的时间点是否已经超过了 mDailyExpiryDate 指定的日期，mDailyExpiryDate 前面我们有说过，其作为是否超过一天的临界点！

```java
    if (event.mTimeStamp >= mDailyExpiryDate.getTimeInMillis()) {
        // Need to rollover
        rolloverStats(event.mTimeStamp);
    }
```

如果超过了 1 天，那么我们要关闭当前最新的使用信息文件，这次的记录时间会做为今天的最后一个记录时间，然后下次 report event 时，会创建一个新的文件记录下一天的使用信息！

当我们发现本次记录的时间超过了 mDailyExpiryDate，那么我们要关闭当前最新日期的使用信息文件了，rolloverStats 就发生在此时 ！

```java
    private void rolloverStats(final long currentTimeMillis) {
        final long startTime = SystemClock.elapsedRealtime();
        Slog.i(TAG, mLogPrefix + "Rolling over usage stats");
        
        //【1】获得 dailay 时间类别使用信息中处于 active 状态的 config！ 
        final Configuration previousConfig =
                mCurrentStats[UsageStatsManager.INTERVAL_DAILY].activeConfiguration;
        
        //【2】continuePreviousDay 用于记录那些需要将设置为 CONTINUE_PREVIOUS_DAY 的 package
        ArraySet<String> continuePreviousDay = new ArraySet<>();

        //【3】遍历当前每个时间类别下最新日期的使用信息！
        for (IntervalStats stat : mCurrentStats) {
            final int pkgCount = stat.packageStats.size();
            for (int i = 0; i < pkgCount; i++) {
                UsageStats pkgStats = stat.packageStats.valueAt(i);

                //【3.1】如果 package 的 mLastEvent 是 MOVE_TO_FOREGROUND 或者 CONTINUE_PREVIOUS_DAY！
                // 将其 event 改为 END_OF_DAY 表示一天的记录结束了！
                if (pkgStats.mLastEvent == UsageEvents.Event.MOVE_TO_FOREGROUND ||
                        pkgStats.mLastEvent == UsageEvents.Event.CONTINUE_PREVIOUS_DAY) {

                    //【3.1.1】那么将该 package 添加到 continuePreviousDay 中！
                    continuePreviousDay.add(pkgStats.mPackageName);
                    //【×4.1】同时更新 pacakge 对应的 UsageStats 的信息！
                    stat.update(pkgStats.mPackageName, mDailyExpiryDate.getTimeInMillis() - 1,
                            UsageEvents.Event.END_OF_DAY);

                    //【×2.6】延迟 20mins 通知 UsageStatsService 刷新本地数据，
                    // mStatsChanged 会被设置为 true，只通知一次！
                    notifyStatsChanged();
                }
            }

            //【×4.2】更新 Configuration 的状态信息！
            stat.updateConfigurationStats(null, mDailyExpiryDate.getTimeInMillis() - 1);
        }
        
        //【×3.1】将内存中的最新数据写回持久化文件！
        //【×2.6】会将 mStatsChanged 设置为 false；
        persistActiveStats();
        
        //【×4.2】移除那些日期太旧的本地文件，然后重新加载文件到内存中！
        mDatabase.prune(currentTimeMillis);
        
        //【×2.3】加载最新使用信息到内存中！
        // 对于 daily 时间类别，会创建一个新的 IntervalStats 对象！
        // 对于其他类别，可能会创建一个新的 IntervalStats 对象！
        // 这里也会将 mStatsChanged 设置为 false；
        loadActiveStats(currentTimeMillis);

        //【4】处理之前收集到的 package，将其设置为 CONTINUE_PREVIOUS_DAY！
        final int continueCount = continuePreviousDay.size();
        for (int i = 0; i < continueCount; i++) {
            String name = continuePreviousDay.valueAt(i);

            //【4.1】这里的 beginTime 是 daily 时间类别的新创建的 IntervalStats 的 beginTime 时间；
            // 同时也是其他类别的更新时间点！
            final long beginTime = mCurrentStats[UsageStatsManager.INTERVAL_DAILY].beginTime;
            
            //【4.2】遍历每个时间类别下的最新使用数据 IntervalStats！
            // 对于 daily 时间类别的数据，这里是将该 package 的使用信息更新到新创建的 IntervalStats！
            // 对于其他类别的数据，可能是更新到了新创建的 IntervalStats 中，也可能是修改已有的最新 IntervalStats！
            for (IntervalStats stat : mCurrentStats) {
            
                //【4.2.1】这里再次调用了 update/updateConfigurationStats 方法更新 IntervalStats！
                // 设置其 event 为 CONTINUE_PREVIOUS_DAY，表示继续前一天的记录！
                stat.update(name, beginTime, UsageEvents.Event.CONTINUE_PREVIOUS_DAY);
                stat.updateConfigurationStats(previousConfig, beginTime);
                
                //【×2.6】延迟 20mins 通知 UsageStatsService 刷新本地数据，
                // mStatsChanged 会被设置为 true，只通知一次！
                notifyStatsChanged();
            }
        }
        
        //【×3.1】将内存中的最新数据写回持久化文件！
        persistActiveStats();

        final long totalTime = SystemClock.elapsedRealtime() - startTime;
        Slog.i(TAG, mLogPrefix + "Rolling over usage stats complete. Took " + totalTime
                + " milliseconds");
    }
```

我们来分析下整个方法的逻辑：


我们来分析下一些细节处理：

这里要注意下 loadActiveStats：

- 加载最新使用信息到内存中，在加载最新数据时，会判断当前的时间是否在最新文件可以记录的范围内，如果在的话，就直接加载最新文件的数据，如果不在的话，那就会创建一个新的 IntervalStats，记录下一个时间段的使用信息；

</br>

- 因为我们当前的时间已经超过了 mDailyExpiryDate，而 mDailyExpiryDate 是以一天为临界点的，所以对于 daily 类别的最新文件来说，已经超过了其能够记录的范围，那么会创建一个新的 IntervalStats 对象！

</br>

- 而对于 weekly，monthly，yearly 不一定超过了其最新文件能够记录的时间返回，所以可能返回的依然是当前最新文件的 IntervalStats 对象！

</br>

- loadActiveStats 方法在加载完成数据后，会调用 updateRolloverDeadline 再次将 mDailyExpiryDate 设置到下一天！

## 4.1 IntervalStats.update

这里再次调用了 IntervalStats.update 方法，更新 package 的信息！

同样的，必须满足一下条件才能进入：

```java
    if (pkgStats.mLastEvent == UsageEvents.Event.MOVE_TO_FOREGROUND ||
            pkgStats.mLastEvent == UsageEvents.Event.CONTINUE_PREVIOUS_DAY) {
        ... ... ...
    }
```
和 init 中传入的 timeStamp 不同，这里传入的是：

- **long timeStamp**：  本次更新时间 mDailyExpiryDate.getTimeInMillis() - 1；            
- **int eventType**：   本次更新的事件 UsageEvents.Event.END_OF_DAY
                
```java
    void update(String packageName, long timeStamp, int eventType) {
        //【2.2.3.1.1】获得该 packageName 对应的 UsageStats 实例！
        UsageStats usageStats = getOrCreateUsageStats(packageName);

        //【1】如果 eventType 为  MOVE_TO_BACKGROUND 或者 END_OF_DAY，进入这个 if 条件！
        if (eventType == UsageEvents.Event.MOVE_TO_BACKGROUND ||
                eventType == UsageEvents.Event.END_OF_DAY) {

            //【1.1】这时判断如果 UsageStats.mLastEvent 的取值为 MOVE_TO_FOREGROUND 或者 CONTINUE_PREVIOUS_DAY
            if (usageStats.mLastEvent == UsageEvents.Event.MOVE_TO_FOREGROUND ||
                    usageStats.mLastEvent == UsageEvents.Event.CONTINUE_PREVIOUS_DAY) {
    
                //【1.2】更新 package 在前台的总时间 mTotalTimeInForeground 为：
                // 在已有基础上再加上 timeStamp - usageStats.mLastTimeUsed
                usageStats.mTotalTimeInForeground += timeStamp - usageStats.mLastTimeUsed;
            }
        }

        //【2.5.1】如果 eventType 属于 Stateful Event
        // 那就更新该 pacakge 的 usageStats.mLastEvent（上一次事件）为 eventType；
        if (isStatefulEvent(eventType)) {
            usageStats.mLastEvent = eventType;
        }

        //【3】如果 eventType 不是 UsageEvents.Event.SYSTEM_INTERACTION，更新 usageStats.mLastTimeUsed
        // 上次使用时间为 timeStamp！
        if (eventType != UsageEvents.Event.SYSTEM_INTERACTION) {
            usageStats.mLastTimeUsed = timeStamp;
        }
        
        //【4】更新该 package 的 usageStats.mEndTimeStamp 为 timeStamp 
        usageStats.mEndTimeStamp = timeStamp;

        //【5】如果 eventType 为 MOVE_TO_FOREGROUND，该 package 的 usageStats.mLaunchCount（启动次数）加一；
        if (eventType == UsageEvents.Event.MOVE_TO_FOREGROUND) {
            usageStats.mLaunchCount += 1;
        }

        //【6】同时修改 IntervalStats.endTime 也为 timeStamp
        endTime = timeStamp;
    }
```
函数流程就不在分析了，我们直接看结论：

- usageStats.mTotalTimeInForeground += (mDailyExpiryDate.getTimeInMillis() - 1) - usageStats.mLastTimeUsed
- usageStats.mLastEvent = UsageEvents.Event.END_OF_DAY
- usageStats.mLastTimeUsed = (mDailyExpiryDate.getTimeInMillis() - 1)
- usageStats.mEndTimeStamp = (mDailyExpiryDate.getTimeInMillis() - 1)

- IntervalStats.endTime = (mDailyExpiryDate.getTimeInMillis() - 1)


## 4.2 IntervalStats.updateConfigurationStats

再次调用 updateConfigurationStats 更新指定配置的信息！

- **Configuration config**： 是要成为 active config 的配置对象，这里传入的是 null；
- **long timeStamp**：传入的是 mDailyExpiryDate.getTimeInMillis() - 1；

```java
    void updateConfigurationStats(Configuration config, long timeStamp) {
        //【1】如果 activeConfiguration 不为 null，说明已经有配置信息处于 active 状态！
        // 那就更新其时间信息！
        if (activeConfiguration != null) {
            ConfigurationStats activeStats = configurations.get(activeConfiguration);
            //【1.1】更新 active config 总的活跃时间 mTotalTimeActive 为：在其基础上加上
            // timeStamp 减去上一次活跃时间 mLastTimeActive！
            activeStats.mTotalTimeActive += timeStamp - activeStats.mLastTimeActive;

            //【1.2】更新上次活跃时间 mLastTimeActive 为 timeStamp 减去 1；
            activeStats.mLastTimeActive = timeStamp - 1;
        }

        //【2】更新 config 指定的配置的信息。因为我们这里传入的是 null，所以不会进入；
        if (config != null) {
            ConfigurationStats configStats = getOrCreateConfigurationStats(config);
            //【2.1】更新 config 最新活跃时间 mLastTimeActive 为 timeStamp
            configStats.mLastTimeActive = timeStamp;
            //【2.2】更新 config 的活跃次数；
            configStats.mActivationCount += 1;
            
            //【2.3】更新 IntervalStats.activeConfiguration 为指定的 config！
            activeConfiguration = configStats.mConfiguration;
        }
        //【3】更新 IntervalStats.endTime 为上一次被更新的时间 ！timeStamp
        endTime = timeStamp;
    }
```
因为这里我们传入的是 Configuration config 为 null，所以只会尝试更新 activeConfiguration 的时间信息！

## 4.3 UsageStatsDatabase.prune

prune 方法会移除那些日期太旧的本地文件！
```java
    public void prune(final long currentTimeMillis) {
        synchronized (mLock) {
            //【1】删除 3 年以前的使用信息文件！
            mCal.setTimeInMillis(currentTimeMillis);
            mCal.addYears(-3);
            pruneFilesOlderThan(mIntervalDirs[UsageStatsManager.INTERVAL_YEARLY],
                    mCal.getTimeInMillis());
            //【2】删除 6 个月以前的使用信息文件！
            mCal.setTimeInMillis(currentTimeMillis);
            mCal.addMonths(-6);
            pruneFilesOlderThan(mIntervalDirs[UsageStatsManager.INTERVAL_MONTHLY],
                    mCal.getTimeInMillis());
            //【3】删除 4 周以前的使用信息文件！
            mCal.setTimeInMillis(currentTimeMillis);
            mCal.addWeeks(-4);
            pruneFilesOlderThan(mIntervalDirs[UsageStatsManager.INTERVAL_WEEKLY],
                    mCal.getTimeInMillis());
            //【4】删除 7 天以前的使用信息文件！
            mCal.setTimeInMillis(currentTimeMillis);
            mCal.addDays(-7);
            pruneFilesOlderThan(mIntervalDirs[UsageStatsManager.INTERVAL_DAILY],
                    mCal.getTimeInMillis());

            //【2.1.2】然后重新加载文件到内存中！
            indexFilesLocked();
        }
    }
```

 - 删除 3 年以前的使用信息文件！ 
 - 删除 6 个月以前的使用信息文件！ 
 - 删除 4 周以前的使用信息文件！ 
 - 删除 7 天以前的使用信息文件！
 - 最后要重新加载最新文件到内存中，防止读取到已经被删除的文件！


### 4.3.1 UsageStatsDatabase.pruneFilesOlderThan

```java
    private static void pruneFilesOlderThan(File dir, long expiryTime) {
        File[] files = dir.listFiles();
        if (files != null) {
            for (File f : files) {
                String path = f.getPath();
                //【1】如果文件名中有 .bak 那么属于备份文件，去掉 .bak 得到非备份文件！
                if (path.endsWith(BAK_SUFFIX)) {
                    f = new File(path.substring(0, path.length() - BAK_SUFFIX.length()));
                }
                long beginTime;
                try {
                    //【2.1.2.1】调用 UsageStatsXml.parseBeginTime 获得文件的
                    beginTime = UsageStatsXml.parseBeginTime(f);
                } catch (IOException e) {
                    beginTime = 0;
                }
                //【2】当文件的开始记录时间早于 expiryTime，该文件过期了，删除！
                if (beginTime < expiryTime) {
                    new AtomicFile(f).delete();
                }
            }
        }
    }
```
方法流程简单，不多说了！

## 4.4 流程总结

# 5 UserUsageStatsService.reportEvent - 上报事件

这里我们来看看上报时间的处理：

```java
    void reportEvent(UsageEvents.Event event) {
        if (DEBUG) {
            Slog.d(TAG, mLogPrefix + "Got usage event for " + event.mPackage
                    + "[" + event.mTimeStamp + "]: "
                    + eventToString(event.mEventType));
        }

        //【1】当前记录超过了 mDailyExpiryDate，那么我们要判断下当前的时间点是否超过了每个时间类别下最新文件能够记录的
        // 时间范围，如果超过了，要创建新的 IntervalStats，用下一个阶段的记录！
        if (event.mTimeStamp >= mDailyExpiryDate.getTimeInMillis()) {
            //【×4】对于数据的处理，在 rolloverStats 中！
            rolloverStats(event.mTimeStamp);
        }

        //【2】获得 daily 类别下的最新 IntervalStats，我们要将本次的数据写入到最新的 IntervalStats！！
        final IntervalStats currentDailyStats = mCurrentStats[UsageStatsManager.INTERVAL_DAILY];

        //【3】如果本次的 event 的类型为 CONFIGURATION_CHANGE，并且 currentDailyStats.activeConfiguration 不为 null
        // 那么调整该 event.mConfiguration！
        final Configuration newFullConfig = event.mConfiguration;
        if (event.mEventType == UsageEvents.Event.CONFIGURATION_CHANGE &&
                currentDailyStats.activeConfiguration != null) {
            event.mConfiguration = Configuration.generateDelta(
                    currentDailyStats.activeConfiguration, newFullConfig);
        }

        //【4】将本次 event 添加到 daily 的最新 IntervalStats 中！
        // 如果 eventType 是 SYSTEM_INTERACTION，则不添加！
        if (currentDailyStats.events == null) {
            currentDailyStats.events = new TimeSparseArray<>();
        }
        if (event.mEventType != UsageEvents.Event.SYSTEM_INTERACTION) {
            currentDailyStats.events.put(event.mTimeStamp, event);
        }

        //【5】遍历所有时间类别下的最新使用信息 IntervalStats！
        // 如果 event 类型为 CONFIGURATION_CHANGE，那就只更新 config
        // 如果是其他类型，只更新 UsageStats！
        for (IntervalStats stats : mCurrentStats) {
            if (event.mEventType == UsageEvents.Event.CONFIGURATION_CHANGE) {
                stats.updateConfigurationStats(newFullConfig, event.mTimeStamp);
            } else {
                stats.update(event.mPackage, event.mTimeStamp, event.mEventType);
            }
        }

        //【×2.6】通知 UsageStatsService，使用信息发生了变化！
        notifyStatsChanged();
    }
```
其实可以看到：event 相关的 log 只会记录到 daily 类别的文件中！

notifyStatsChanged 会将 mStatsChanged 置为 true，然后触发 mListener.onStatsUpdated 方法，UsageStatsService.onStatsUpdated 会回调 persistActiveStats 方法，将最新的使用信息保存到本地文件中，最后设置 mStatsChanged 为 false；

```java
    private void notifyStatsChanged() {
        if (!mStatsChanged) {
            mStatsChanged = true;
            mListener.onStatsUpdated();
        }
    }
```

整个过程很简单，不多说了！


# 6 UserUsageStatsService.query - 查询

UserUsageStatsService 提供了三个查询接口：

```java
UserUsageStatsService.queryEvents
UserUsageStatsService.queryUsageStats
UserUsageStatsService.queryStats
```
下面我们来分析下具体的查询过程！

## 6.1 UserUsageStatsService.queryEvents
## 6.2 UserUsageStatsService.queryUsageStats
## 6.3 UserUsageStatsService.queryStats