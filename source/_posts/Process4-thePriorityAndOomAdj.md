# Process篇 4 - 进程的 priority 和 oomAdj 简析
title: Process篇 4 - 进程的 priority 和 oomAdj 简析
date: 2016/04/13 20:46:25
catalog: true
categories:
- AndroidFramework源码分析
- Process进程
tags: Process进程
---

基于 `Android 7.1.1` 源码，分析和总结进程相关的知识！ 

本文参考：

```java
https://developer.android.com/guide/components/processes-and-threads.html
```


# 1 进程的重要性层次

进程的重要性层次一共有 `5` 级，以下的部分内容截取自 `Android Developer Guide`！

> **From Android Developer Guide**

**【前台进程 Foreground Process】**

**定义：**用户当前操作所必需的进程。

- 如果一个进程满足以下任一条件，即视为前台进程：
  - 托管用户正在交互的 `Activity`。（即已调用 `Activity` 的 `onResume()` 方法）
  - 托管某个 `Service`，后者绑定到用户正在交互的 `Activity`。
  - 托管正在 “前台” 运行的 `Service`。（服务已调用 `startForeground()`）
  - 托管正执行一个生命周期回调的 `Service`。（比如：`onCreate()`，`onStart()`或 `onDestroy()`）
  - 托管正执行其 `onReceive()` 方法的 `BroadcastReceiver`

通常，在任意给定时间前台进程都为数不多。前台进程的优先级最高，只有在内存不足，万不得已的时候，系统才会杀掉他们！

</br>

**【可见进程 Visible Process】**

**定义：**没有任何前台组件、但仍会影响用户在屏幕上所见内容的进程。

- 如果一个进程满足以下任一条件，即视为可见进程：
   - 托管不在前台、但仍对用户可见的 `Activity`（已调用其 `onPause()` 方法）。
      - 例如，如果前台 `Activity` 启动了一个对话框，允许在其后显示上一个 `Activity`，则有可能会发生这种情况。
   - 托管绑定到可见（或前台）`Activity` 的 `Service`。

可见进程被视为是极其重要的进程，除非为了维持所有前台进程同时运行而必须终止，否则系统不会终止这些进程。

</br>

**【服务进程 Service Process】**

**定义：**正在运行已使用 `startService()` 方法启动的服务且不属于上述两个更高类别进程的进程，那就属于服务进程！

可以看到管服务进程与用户所见内容没有直接关联，但是它们通常在执行一些用户关心的操作（例如，在后台播放音乐或从网络下载数据）。因此，除非内存不足以维持所有前台进程和可见进程同时运行，否则系统会让服务进程保持运行状态。

</br>

**【后台进程 Background Process】**

**定义：**包含目前对用户不可见的 `Activity` 的进程（已调用 `Activity` 的 `onStop()` 方法）。

这些进程对用户体验没有直接影响，系统可能随时终止它们，以回收内存供前台进程、可见进程或服务进程使用。

通常会有很多后台进程在运行，因此它们会保存在 `LRU` （最近最少使用）列表中，以确保包含用户最近查看的 `Activity` 的进程最后一个被终止。

如果某个 `Activity` 正确实现了生命周期方法，并保存了其当前状态，则终止其进程不会对用户体验产生明显影响，因为当用户导航回该 `Activity` 时，`Activity` 会恢复其所有可见状态。

</br>

**【空进程 Empty Process】**

**定义：**不含任何活动应用组件的进程。

保留这种进程的的唯一目的是用作缓存，以缩短下次在其中运行组件所需的启动时间。为使总体系统资源在进程缓存和底层内核缓存之间保持平衡，系统往往会终止这些进程。

> **End From**

通过 `Android developer guide` 我们能可以知道进程优先级的 `5` 分类以及其满足的条件，这里我用一张思维导图简单的总结一下：

<img src="https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/23/aonaotu-download.png" alt="aonaotu-download.png-252.2kB" style="zoom:50%;" />


注意：这只是一个粗略的划分。其实，在系统的内部实现中，优先级远不止这么五种。

# 2 进程的优先级和状态

在 `ActivityManager.java` 中，我们可以通过一个方法来获得系统中所有正在运行中的应用进程的信息：

```java
    public List<RunningAppProcessInfo> getRunningAppProcesses() {
        try {
            return ActivityManagerNative.getDefault().getRunningAppProcesses();
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
```
该方法会返回一个 `List` 列表，每一个进程都被封装成了一个 `RunningAppProcessInfo` 对象，`RunningAppProcessInfo` 中有几个重要的成员变量，用于表示进程的重要性和状态；

```java
        public int importance;
        public int processState;
```
其中，`processState` 表示的是进程的状态，系统是通过这个值来动态调节进程的优先级的；`importance` 则是进程的重要级别，它是系统暴露给外部的参数，用于获得自身或者其他一个应用进程的重要性级别，下面我们来看看二者的关系和定义！

## 2.1 进程的优先级

首先，来看看 `importance`，其表示进程的重要性级别，取值有如下的 `10` 种，定义在 `ActivityManager.java` 中！

```java
        // 该进程持有前台 UI 界面，该 UI 界面正在和用户交互！
        public static final int IMPORTANCE_FOREGROUND = 100;

        // 该进程运行着一个前台服务，比如进程在播放音乐，即使用户没有在应用中，
        // 这种状态表明进程正在做一些用户非常关心的事情！
        public static final int IMPORTANCE_FOREGROUND_SERVICE = 125;

        // 该进程虽然用户无法直接感知到，但在某种程度上他们仍然可以察觉到!
        public static final int IMPORTANCE_PERCEPTIBLE = 130;

        // 该进程运行着一个前台 UI 界面，但是由于设备进入了睡眠状态，因此对用户不可见，用户此时无法和其交互！
        // 用户也无法感知该进程，但是当用户唤醒设备后，用户又可以与之交互，所以这种状态下的进程也是很重要的！
        public static final int IMPORTANCE_TOP_SLEEPING = 150;
        
        // 该进程中运行的应用，我们不能保存它的状态，所以当该进程退到后台时，我们不能杀死它！
        public static final int IMPORTANCE_CANT_SAVE_STATE = 170;

        // 该进程运行着一些没有在前台，但是对用户是可见的内容，其内部可能运行着一个在前台 UI 下面的窗口
        //（其处于暂停状态，系统将其状态保存了下来，其没有和用户交互，但是一定程度来说对用户是可见的）；
        // 其内部也可能运行着在系统控制下的服务！
        public static final int IMPORTANCE_VISIBLE = 200;

        // 服务进程，该进程包含一些在后台运行的服务，这些服务用户是无感知的，所以它们可以由系统相对自由地杀死！
        public static final int IMPORTANCE_SERVICE = 300;

        // 后台进程，用户完全无法感知到进程的存在
        public static final int IMPORTANCE_BACKGROUND = 400;

        // 空进程
        public static final int IMPORTANCE_EMPTY = 500;

        // 此进程不存在
        public static final int IMPORTANCE_GONE = 1000;
```

## 2.2 进程的状态

接着，我们来看看 `processState`，其表示进程的状态，其取值范围有如下的 `17` 种，同样定义在 `ActivityManager.java` 中！

进程的状态是从 `android` 系统管理角度进行的分级，和 `lmk` 的分级不同！

```java
    // 进程不存在！
    public static final int PROCESS_STATE_NONEXISTENT = -1;

    // persist 系统进程
    public static final int PROCESS_STATE_PERSISTENT = 0;

    // persist 系统进程，并且正在进行 UI 相关的操作！
    public static final int PROCESS_STATE_PERSISTENT_UI = 1;

    // 该进程拥有当前用户可见的 top activity，其他可见的 activity 都在其下面！
    public static final int PROCESS_STATE_TOP = 2;

    // 该进程拥有一个前台服务，该服务是由系统绑定的！
    public static final int PROCESS_STATE_BOUND_FOREGROUND_SERVICE = 3;

    // 该进程拥有一个前台服务！
    public static final int PROCESS_STATE_FOREGROUND_SERVICE = 4;

    // 该进程拥有当前用户可见的 top activity，和 PROCESS_STATE_TOP 一样！
    // 但此时设备处于休眠状态！
    public static final int PROCESS_STATE_TOP_SLEEPING = 5;

    // 对用户很重要的进程，用户可感知其存在
    public static final int PROCESS_STATE_IMPORTANT_FOREGROUND = 6;

    // 对用户很重要的进程，用户不可感知其存在
    public static final int PROCESS_STATE_IMPORTANT_BACKGROUND = 7;

    // 后台进程，正在执行备份 / 恢复操作！
    public static final int PROCESS_STATE_BACKUP = 8;

    // 后台进程，但是我们不能恢复其状态，所以尽量避免杀死该进程
    public static final int PROCESS_STATE_HEAVY_WEIGHT = 9;
    
    // 后台进程，正在运行一个服务，执行某个操作！
    public static final int PROCESS_STATE_SERVICE = 10;

    // 后台进程，正在运行一个广播接收者，处理广播！
    public static final int PROCESS_STATE_RECEIVER = 11;

    // 后台进程，桌面 home activity 位于该进程中
    public static final int PROCESS_STATE_HOME = 12;

    // 后台进程，拥有上一次显示给用户的 activity
    public static final int PROCESS_STATE_LAST_ACTIVITY = 13;

    // 缓存进程，供以后使用，内部包含 activity。
    public static final int PROCESS_STATE_CACHED_ACTIVITY = 14;

    // 缓存进程，供以后使用，其为另一个内含 activity 的缓存进程的 client 端进程！
    public static final int PROCESS_STATE_CACHED_ACTIVITY_CLIENT = 15;

    // 缓存进程，供以后使用，内部为空
    public static final int PROCESS_STATE_CACHED_EMPTY = 16;
```


我们知道，每一个进程在系统中都有一个 `ProcessRecord` 对象与之对应，其内部封装着该进程的数据和组件信息，`ProcessRecord` 也有一些成员变量，其取值也是上面的这 `17` 种！

```java
  int curProcState = PROCESS_STATE_NONEXISTENT; // Currently computed process state
  int repProcState = PROCESS_STATE_NONEXISTENT; // Last reported process state
  int setProcState = PROCESS_STATE_NONEXISTENT; // Last set process state in process tracker
  int pssProcState = PROCESS_STATE_NONEXISTENT; // Currently requesting pss for
```
`ActivityManagerService` 就是通过这些变量来设置和调整进程的状态的，这个我们后面再看！



## 2.3 进程优先级和状态的转换

在系统中 `processState` 和 `importance` 是有相应的映射和转换关系的，其映射方法的定义在 `ActivityManager.java` 中！

```java
    @SystemApi
    public int getPackageImportance(String packageName) {
        try {
            //【1】获得进程的状态值！
            int procState = ActivityManagerNative.getDefault().getPackageProcessState(packageName,
                    mContext.getOpPackageName());
            //【2】将进程的状态映射为所属的重要性级别
            return RunningAppProcessInfo.procStateToImportance(procState);
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
```

最终的映射函数在 `RunningAppProcessInfo.procStateToImportance`：

```java
        public static int procStateToImportance(int procState) {
            //【1】将进程的状态转为对应的重要性级别！
            if (procState == PROCESS_STATE_NONEXISTENT) { // -1
                return IMPORTANCE_GONE;
                
            } else if (procState >= PROCESS_STATE_HOME) { // 12
                return IMPORTANCE_BACKGROUND;
                
            } else if (procState >= PROCESS_STATE_SERVICE) { // 10
                return IMPORTANCE_SERVICE;
                
            } else if (procState > PROCESS_STATE_HEAVY_WEIGHT) { // 9
                return IMPORTANCE_CANT_SAVE_STATE;
                
            } else if (procState >= PROCESS_STATE_IMPORTANT_BACKGROUND) { // 7
                return IMPORTANCE_PERCEPTIBLE;
                
            } else if (procState >= PROCESS_STATE_IMPORTANT_FOREGROUND) { // 6
                return IMPORTANCE_VISIBLE;
                
            } else if (procState >= PROCESS_STATE_TOP_SLEEPING) { // 5
                return IMPORTANCE_TOP_SLEEPING;
                
            } else if (procState >= PROCESS_STATE_FOREGROUND_SERVICE) { // 4
                return IMPORTANCE_FOREGROUND_SERVICE;
                
            } else {
                return IMPORTANCE_FOREGROUND;
            }
        }
```

通过该方法，我们可以知道，某个应用进程所属的重要性级别了！

**这里不解的是，对应关系中忽略掉了一些 `IMPORTANCE` 级别，不只是为什么！**


# 3 进程的 oomAdj 值

说到 `oomAdj`，我们就要谈到 `Android` 中的一个很重要的机制 `Low Memory Killer`，也叫做低内存杀手！对于 `Low Memory Killer`，我后面会单独写一篇文章来分析和总结！

`Low Memory Killer` 基于 `Linux` 的 `OOM` 机制，在 `Linux` 中，内存是以页面为单位分配的，当申请页面分配时如果内存不足会通过以下流程选择 `bad` 进程来杀掉从而释放内存：

```c
alloc_pages -> out_of_memory() -> select_bad_process() -> badness()
```

而 `Low Memory Killer` 杀进程依据就是进程的 `oom_adj` 值，`oom_adj` 越小越不容易被杀死，对于 `oomAdj` 值，定义在 `ProcessList.java` 中;

```java
    // 初始化值
    static final int INVALID_ADJ = -10000;

    // 该值用于用于某些我们还没法确定具体值的地方，一般是指当一个进程要被缓存了，但我们还不知道要给它分配的
    // 缓存进程的 adj 的具体值！
    static final int UNKNOWN_ADJ = 1001;

    // 缓存进程，其 adj 的最大致和最小值，该进程仅托管不可见的 activity，他们容易就会被杀死！
    static final int CACHED_APP_MAX_ADJ = 906;
    static final int CACHED_APP_MIN_ADJ = 900;

    // B list 列表中的服务所在进程，该进程中的服务很老旧，使用率小，不像 A list 中的服务那样活跃和使用频繁！
    static final int SERVICE_B_ADJ = 800;

    // 用户所在的上一个应用的进程
    static final int PREVIOUS_APP_ADJ = 700;

    // 托管着桌面 HOME 应用的进程，我们要尽量避免杀掉它，即使其退到了后台！因为用户和其交互的频率很高！
    static final int HOME_APP_ADJ = 600;

    // 托管着应用 Service 的进程，杀掉这类进程对用户不会有太大的影响！
    static final int SERVICE_ADJ = 500;

    // 高权重 "heavy-weight" 应用所在进程，虽然该进程是在后台，但是我们要避免杀掉它
    // 该 adj 是 "system/rootdir/init.rc" 在启动时设置的！
    static final int HEAVY_WEIGHT_APP_ADJ = 400;

    // 正在执行备份 backup 操作的进程，
    // 虽然杀掉这类进程不会导致严重问题，但是会导致备份的数据异常，所以杀掉它不是明智之举！
    static final int BACKUP_APP_ADJ = 300;

    // 仅持有用户可感知组件的进程，比如在后台播放音乐，虽然其对用户不是立即可见，但是我们也要比避免杀死它！
    static final int PERCEPTIBLE_APP_ADJ = 200;

    // 持有对用户可见的 activity 的进程，我们也要避免杀死他们！
    static final int VISIBLE_APP_ADJ = 100;
    static final int VISIBLE_APP_LAYER_MAX = PERCEPTIBLE_APP_ADJ - VISIBLE_APP_ADJ - 1;

    // 运行着当前前台应用的进程，我们不能杀掉这类进程！
    static final int FOREGROUND_APP_ADJ = 0;

    // 被系统进程或者 persistent 进程绑定的进程！
    static final int PERSISTENT_SERVICE_ADJ = -700;

    // 系统 persistent 进程，比如 telephony，这种类型的进程系统一定是不会杀死的！
    static final int PERSISTENT_PROC_ADJ = -800;

    // 系统进程，默认分配！
    static final int SYSTEM_ADJ = -900;

    // native 进程，不受系统的管控！
    static final int NATIVE_ADJ = -1000;
```

我们在前面知道了进程的重要性级别和进程的状态的映射关系，那么对于进程的状态和进程的 `oomAdj` 之间是否也有一定的联系呢？这里就要谈到 `oomAdj` 的调度算法，我会单独开一篇文章来分析！

接下来，我介绍下 `ProcessRecord` 中和 `oomAdj` 相关的几个重要变量，我们后续分析 `oomAdj` 的时候会遇到：

```java
    int maxAdj;                 // 该进程最大的 Adj，进程的 adj 的设定不能超过该值
    
    int curRawAdj;              // 在第一个过程评估后该进程具有的 adj 值，
    int setRawAdj;              // 上一次经过第一次评估后该进程具有的 adj 值，curRawAdj 被更新后，旧值会保存到 setRawAdj 
    
    int curAdj;                 // 进程当前的 adj 值，经过了两个过程评估，这才是进程实际的 adj 值
    int setAdj;                 // 上一次设置的 adj 值， curAdj 被更新了，旧值会被保存到 setAdj 中！
    
    int verifiedAdj;            // The last adjustment that was verified as actually being set
    
    int curSchedGroup;          // Currently desired scheduling class
    int setSchedGroup;          // Last set to background scheduling class
```

对于 `low memory killer`，当系统的可用内存不够的时候，他会根据一定的策略来杀掉进程，释放内存，针对 `low memory killer` 我后续也会单独写一篇博文来分析！

# 4 ActivityManager 相关接口

```java
    /** @hide Should this process state be considered a background state? */
    public static final boolean isProcStateBackground(int procState) {
        return procState >= PROCESS_STATE_BACKUP;
    }
```
该方法用来判断，一个进程是否是后台进程！！

http://melove.net/blog/2017/03/android-daemon-service-1488942411000.html

http://www.cnblogs.com/angeldevil/category/336548.html

http://blog.csdn.net/ccjhdopc/article/details/52818012

http://blog.csdn.net/ccjhdopc/article/details/52818012


[1]: 