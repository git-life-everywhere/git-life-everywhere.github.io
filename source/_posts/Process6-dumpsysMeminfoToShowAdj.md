# Process篇 6 - 从 dumpsys meminfo 看进程的优先级
title: Process篇 6 - 从 dumpsys meminfo 看进程的优先级
date: 2016/06/13 20:46:25
catalog: true
categories:
- AndroidFramework源码分析
- Process进程
tags: Process进程
---

# 0 前言

基于 android 7.1.1 源码，分析和总结 Process 相关知识！

# 1 dumpsys meminfo --oom

`dumpsys meminfo` 可以来看系统的内存使用情况，这里我们重点关注：`Total PSS by OOM adjustment`:

默认的 `dumpsys meminfo` 是出了可以看 `Total PSS by OOM adjustment`，还可以看 `Total PSS by process` 等等详细的信息，这里我们只关注第一种，这里有一个很方便的指令：

`adb shell dumpsys meminfo --oom` 

下面是该指令的输出：

```java
Applications Memory Usage (in Kilobytes):
Uptime: 37282469 Realtime: 62579406

     ... ... ... ...

Total PSS by OOM adjustment:
    304,190K: Native
         38,170K: android.hardware.camera.provider@2.4-service (pid 683)
         17,815K: logd (pid 539)
         15,833K: vendor.oppo.hardware.biometrics.face@1.0-service (pid 1109)
         15,014K: surfaceflinger (pid 721)
          7,927K: android.hardware.audio@2.0-service (pid 681)
          7,318K: webview_zygote32 (pid 2635)
          7,105K: rild (pid 1083)
          6,857K: media.codec (pid 1082)
          ... ... ... ...
  
    119,314K: System
        119,314K: system (pid 1703)

    168,951K: Persistent
         94,755K: com.android.systemui (pid 2598)
         23,766K: com.android.phone (pid 2738)
         11,445K: .dataservices (pid 2706)
          7,254K: com.oppo.multimedia.dirac (pid 3493)
          ... ... ... ...

     95,474K: Foreground
         33,792K: com.coloros.safecenter:clear_filter (pid 2514)
         26,509K: com.oppo.ota:ui (pid 15668 / activities)
         13,879K: com.coloros.securitypermission (pid 3140)
         12,447K: com.oppo.ota (pid 15686)
          8,847K: com.nearme.romupdate (pid 2491)
          ... ... ... ...
          
    524,433K: Visible
        105,184K: com.tencent.mm:push (pid 7125)
        100,550K: com.tencent.mm:exdevice (pid 5237)
         62,929K: com.oppo.launcher (pid 3325 / activities)
         27,815K: android.process.contacts (pid 4085)
         24,425K: com.coloros.safecenter (pid 3708)
         23,279K: com.coloros.mcs (pid 4122)
         21,670K: android.process.acore (pid 4310)
         20,998K: com.coloros.recents (pid 3267 / activities)
         ... ... ... ...
          
     39,159K: Perceptible
         31,468K: com.sohu.inputmethod.sogouoem (pid 2584)
          7,691K: com.amap.android.location (pid 4132)
          
    103,667K: A Services
         72,943K: com.kuaikan.comic:monitorService (pid 4938)
         30,724K: com.kuaikan.comic:QS (pid 5207)
         
    174,752K: Previous
        147,972K: com.tencent.mm (pid 7318)
         26,780K: com.android.settings (pid 13815 / activities)
         
    489,783K: B Services
        370,782K: com.kuaikan.comic (pid 4869 / activities)
         28,757K: com.android.mms (pid 5120)
         27,198K: com.kuaikan.comic:QALSERVICE (pid 5063)
         15,458K: com.coloros.selfcheck (pid 15626 / activities)
         15,422K: com.tencent.mobileqq:MSF (pid 10558)
         14,523K: com.nearme.statistics.rom (pid 2556)
          9,211K: com.coloros.gallery3d (pid 16053)
          4,513K: com.coloros.activation (pid 5301)
          3,919K: com.coloros.usbselection (pid 16123)
          
     76,443K: Cached
         39,483K: com.coloros.aiservice (pid 7038)
         17,041K: android.process.media (pid 13010)
         10,643K: com.android.providers.downloads (pid 3251)
          4,979K: com.coloros.securepay (pid 9412)
          4,297K: com.android.defcontainer (pid 15323)

Total RAM: 3,831,972K (status normal)
 Free RAM: 1,734,943K (   76,527K cached pss + 1,319,364K cached kernel +    24,584K ion cached +   314,468K free)
 Used RAM: 2,459,876K (2,083,100K used pss +   376,776K kernel)
 Lost RAM:  -979,823K
     ZRAM:   616,976K physical used for 1,361,056K in swap (2,097,148K total swap)
   Tuning: 384 (large 512), oom   322,560K, restore limit   107,520K (high-end-gfx)
```

我们可以看到，这部分，系统是根据进程的 `adj` 将其分类，然后根据不同的分类，显示出了不同类别进程的内存使用情况！

这里我们关心的是 `adb shell dumpsys meminfo --oom`  是如何归类不同类型的进程的！



# 2 dumpApplicationMemoryUsage

`adb shell dumpsys meminfo --oom` 最终会调用 `dumpApplicationMemoryUsage` 函数，这里我们重点关注和 `--oom` 相关的逻辑：

这里我们先来看几个和该指令相关的常量;

```java
    static final int[] DUMP_MEM_OOM_ADJ = new int[] {
            ProcessList.NATIVE_ADJ,
            ProcessList.SYSTEM_ADJ, ProcessList.PERSISTENT_PROC_ADJ,
            ProcessList.PERSISTENT_SERVICE_ADJ, ProcessList.FOREGROUND_APP_ADJ,
            ProcessList.VISIBLE_APP_ADJ, ProcessList.PERCEPTIBLE_APP_ADJ,
            ProcessList.BACKUP_APP_ADJ, ProcessList.HEAVY_WEIGHT_APP_ADJ,
            ProcessList.SERVICE_ADJ, ProcessList.HOME_APP_ADJ,
            ProcessList.PREVIOUS_APP_ADJ, ProcessList.SERVICE_B_ADJ, ProcessList.CACHED_APP_MIN_ADJ
    };

    static final String[] DUMP_MEM_OOM_LABEL = new String[] {
            "Native",
            "System", "Persistent", "Persistent Service", "Foreground",
            "Visible", "Perceptible",
            "Heavy Weight", "Backup",
            "A Services", "Home",
            "Previous", "B Services", "Cached"
    };
```

`DUMP_MEM_OOM_ADJ` 中封装的是 `oom adj`，`DUMP_MEM_OOM_LABEL` 中封装的是 `lebal`，这个看命令输出，很容易猜到是什么意思！

下面我们来重点分析方法！！

```java
    final void dumpApplicationMemoryUsage(FileDescriptor fd,
            PrintWriter pw, String prefix, String[] args, boolean brief, PrintWriter categoryPw) {
            
        ... ... ... ...
        
        // 解析指令，因为我们有 `--oom`，所以 oomOnly 为 true！
        else if ("--oom".equals(opt)) {
                oomOnly = true;
            } 
            ... ... ... ...
        }

        long uptime = SystemClock.uptimeMillis();
        long realtime = SystemClock.elapsedRealtime();
        final long[] tmpLong = new long[1];

        // 收集进程，返回的就是 mLruProcesses 的拷贝！
        ArrayList<ProcessRecord> procs = collectProcesses(pw, opti, packages, args);

        ... ... ... ...
        
        // 创建一些数组，用于封装信息，数组长度均是 DUMP_MEM_OOM_LABEL.length: 14！
        long oomPss[] = new long[DUMP_MEM_OOM_LABEL.length];
        long oomSwapPss[] = new long[DUMP_MEM_OOM_LABEL.length];
        ArrayList<MemItem>[] oomProcs = (ArrayList<MemItem>[])
                new ArrayList[DUMP_MEM_OOM_LABEL.length];

        long totalPss = 0;
        long totalSwapPss = 0;
        long cachedPss = 0;
        long cachedSwapPss = 0;
        boolean hasSwapPss = false;

        Debug.MemoryInfo mi = null;
        
        // 逆序遍历进程集合！
        for (int i = procs.size() - 1 ; i >= 0 ; i--) {
            // 获得进程对象！
            final ProcessRecord r = procs.get(i);
            final IApplicationThread thread;
            final int pid;
            final int oomAdj;
            final boolean hasActivities;
            synchronized (this) {
                thread = r.thread;
                pid = r.pid; // 收集 pid！
                oomAdj = r.getSetAdjWithServices(); // 获得该进程的 oomAdj！
                hasActivities = r.activities.size() > 0; // 该进程是否正在运行 activity！
            }

            if (thread != null) {
                ... ... ... ...

                if (!isCheckinRequest && mi != null) {
                    totalPss += myTotalPss;
                    totalSwapPss += myTotalSwapPss;
                    MemItem pssItem = new MemItem(r.processName + " (pid " + pid +
                            (hasActivities ? " / activities)" : ")"), r.processName, myTotalPss,
                            myTotalSwapPss, pid, hasActivities);

                   ... ... ... ...
                    
                    //【1】便利 oomPss 数组，匹配 oomAdj！
                    for (int oomIndex = 0; oomIndex < oomPss.length; oomIndex++) {

                        //【1.1】当此时已经到了 oomPss 的最后一个元素，或者
                        // 该进程的 oomAdj 处于 [ DUMP_MEM_OOM_ADJ[oomIndex], DUMP_MEM_OOM_ADJ[oomIndex + 1] ) 之间！
                        // 这是我们匹配到了合适的 oomAdj，进入以下逻辑！
                        if (oomIndex == (oomPss.length - 1)
                                || (oomAdj >= DUMP_MEM_OOM_ADJ[oomIndex]
                                        && oomAdj < DUMP_MEM_OOM_ADJ[oomIndex + 1])) {
                            
                            // oomPss[oomIndex] 累加上 myTotalPss！
                            oomPss[oomIndex] += myTotalPss;
                            // oomSwapPss[oomIndex] 累加上 myTotalSwapPss！
                            oomSwapPss[oomIndex] += myTotalSwapPss;

                            // 创建一个 ArrayList 列表，用于封装进程的内存对象！
                            if (oomProcs[oomIndex] == null) {
                                oomProcs[oomIndex] = new ArrayList<MemItem>();
                            }

                            // 将 pssItem 添加爱到该 ArrayList 中去！
                            oomProcs[oomIndex].add(pssItem);
                            break;
                        }
                    }
                }
            }
        }
        
        
        // 下面这段逻辑很重要，决定了 adb shell dumpsys meminfo --oom 是如何对进程划分的！
        if (!isCheckinRequest && procs.size() > 1 && !packages) {
            ... ... ... ...

            ArrayList<MemItem> oomMems = new ArrayList<MemItem>();

            // 遍历 oomPss 数组
            for (int j = 0; j < oomPss.length; j++) {
                if (oomPss[j] != 0) {
                    // 获得 label ，因为 isCompact 为 false，所以为 UMP_MEM_OOM_LABEL[j]！
                    String label = isCompact ? DUMP_MEM_OOM_COMPACT_LABEL[j]
                            : DUMP_MEM_OOM_LABEL[j];

                    // 创建总的 MemItem 对象，用于封装每种类型下的进程的信息；
                    // oomPss[j] 表示这个 label 下的总物理内存；
                    // oomSwapPss[j] 表示该 label 下的总交换物理内存；
                    // DUMP_MEM_OOM_ADJ[j] 表示该进程类别的起始 oom adj；
                    MemItem item = new MemItem(label, label, oomPss[j], oomSwapPss[j],
                            DUMP_MEM_OOM_ADJ[j]);
                            
                    // subitems 则用来表示属于该类别的所有进程信息！
                    item.subitems = oomProcs[j];
                    oomMems.add(item);
                }
            }

            ... ... ... ...
            
            if (!isCompact) {
                pw.println("Total PSS by OOM adjustment:");
            }
            
            // 显示出最终的信息！！
            dumpMemItems(pw, "  ", "oom", oomMems, false, isCompact, dumpSwapPss);
 
            ... ... ... ... 
        }
    }

```
上面的逻辑是和 `Total PSS by OOM adjustment` 相关的！


```java
    ArrayList<ProcessRecord> collectProcesses(PrintWriter pw, int start, boolean allPkgs,
            String[] args) {
        ArrayList<ProcessRecord> procs;
        synchronized (this) {
            if (args != null && args.length > start
                    && args[start].charAt(0) != '-') {
                procs = new ArrayList<ProcessRecord>();
                int pid = -1;

                try {
                    pid = Integer.parseInt(args[start]);
                } catch (NumberFormatException e) {
                }

                for (int i=mLruProcesses.size()-1; i>=0; i--) {
                    ProcessRecord proc = mLruProcesses.get(i);
                    if (proc.pid == pid) { // 按照 pid 收集；
                        procs.add(proc);

                    } else if (allPkgs && proc.pkgList != null
                            && proc.pkgList.containsKey(args[start])) { // 按照 packageName 收集；
                        procs.add(proc);

                    } else if (proc.processName.equals(args[start])) { // 按照 processName 收集！
                        procs.add(proc);

                    }
                }

                if (procs.size() <= 0) {
                    return null;
                }
            } else {
                // 不指定 pid，processName，packageName 的话，就返回 mLruProcesses！
                procs = new ArrayList<ProcessRecord>(mLruProcesses);
            }
        }
        return procs;
    }
```
其实，我们可以看出 `adb shell dumpsys meminfo --oom` 没有传入任何参数，所以返回的就是 `mLruProcesses` 的拷贝！


# 3 总结

分析了 adb shell dumpsys meminfo --oom 方法的执行流程，我们来总结下，进程的分类：

|序号|进程 Label |起始 oom adj 值|
|-------|------|------|-----|
|`1`|`Native`            |`ProcessList.NATIVE_ADJ : -1000`           | 
|`2`|`System`            |`ProcessList.SYSTEM_ADJ : -900`            |
|`3`|`Persistent`        |`ProcessList.PERSISTENT_PROC_ADJ : -800`   |
|`4`|`Persistent Service`|`ProcessList.PERSISTENT_SERVICE_ADJ : -700`| 
|`5`|`Foreground`        |`ProcessList.FOREGROUND_APP_ADJ : 0`       |
|`6`|`Visible`           |`ProcessList.VISIBLE_APP_ADJ : 100`        |
|`7`|`Perceptible`       |`ProcessList.PERCEPTIBLE_APP_ADJ : 200`    | 
|`8`|`Backup`            |`ProcessList.BACKUP_APP_ADJ : 300`         | 
|`9`|`Heavy Weight`      |`ProcessList.HEAVY_WEIGHT_APP_ADJ : 400`   | 
|`10`|`A Services`       |`ProcessList.SERVICE_ADJ : 500`            | 
|`11`|`Home`             |`ProcessList.HOME_APP_ADJ : 600`           | 
|`12`|`Previous`         |`ProcessList.PREVIOUS_APP_ADJ : 700`       | 
|`13`|`B Services`       |`ProcessList.SERVICE_B_ADJ : 800`          | 
|`14`|`Cached`           |`ProcessList.CACHED_APP_MIN_ADJ : 900`     | 
