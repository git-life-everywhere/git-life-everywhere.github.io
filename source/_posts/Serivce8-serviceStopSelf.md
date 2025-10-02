# Serivce 篇 8 - Service stopSelf 流程分析
title: Serivce 篇 8 - Service stopSelf 流程分析
date: 2016/10/29 20:46:25
catalog: true
categories:
- AndroidFramework源码分析
- Service服务
tags: Service服务
---

本文基于 `Android 7.1.1` 源码分析，转载请说明出处！

# 0 综述

对于 `Service`，我们除了可以调用 `stopService` 来停止服务，服务自身也可以调用 `stopSelf` 方法停止，下面我们来看看这几个方法！
```
    public final void stopSelf() {
        stopSelf(-1);
    }

    public final void stopSelf(int startId) {
        if (mActivityManager == null) {
            return;
        }
        try {
            mActivityManager.stopServiceToken(
                    new ComponentName(this, mClassName), mToken, startId);
        } catch (RemoteException ex) {
        }
    }
    
    public final boolean stopSelfResult(int startId) {
        if (mActivityManager == null) {
            return false;
        }
        try {
            return mActivityManager.stopServiceToken(
                    new ComponentName(this, mClassName), mToken, startId);
        } catch (RemoteException ex) {
        }
        return false;
    }
```
这几个方法有什么区别呢？这里先简单的说一下：

- `stopSelf()` 方法，会调用第二个 `stopSelf(int startId)` 方法，如果服务之前通过 `startService` 方法启动过了，那该方法就等价于调用 `stopService` 方法！
- `stopSelf(int startId)` 方法，是 `stopSelfResult` 方法的旧版本，不会返回任何结果！
- `stopSelfResult(int startId)` 方法，通过指定启动项的 `startId` 来停止服务，该方法也等价于 `stopService` 方法，但是它能避免在停止服务时，服务仍没有接收到启动项！

下面，我们来看看该方法的调用过程！

# 1 系统进程

这里省略了 `Binder` 调用的过程，`Binder` 客户端发送 `STOP_SERVICE_TOKEN_TRANSACTION` 到 `Binder` 服务端，进入系统进程中！！

## 1.1 ActivityManagerS.stopServiceToken
```
    @Override
    public boolean stopServiceToken(ComponentName className, IBinder token,
            int startId) {
            
        synchronized(this) {
            return mServices.stopServiceTokenLocked(className, token, startId);
        }
    }
```

这里我们来解释一下参数：

- **ComponentName className**：服务对应的 `ComponentName` 对象。封装了组件信息，包括其所在的运行环境，对应的 `.class` 文件名！
- **IBinder token**：`ServiceRecord` 对象，在启动服务时，会将该对象保存到 `Service` 的
- **int startId**：启动项的 `id` 值；

## 1.2 ActiveServices.stopServiceToken
```java
    boolean stopServiceTokenLocked(ComponentName className, IBinder token,
            int startId) {
        if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "stopServiceToken: " + className
                + " " + token + " startId=" + startId);
        
        //【1】尝试根据服务的信息，匹配到对应的 ServiceRecord 对象！
        ServiceRecord r = findServiceLocked(className, token, UserHandle.getCallingUserId());
        if (r != null) {
            
            // 如果指定了启动项目的 Id，进入该分支！
            if (startId >= 0) {

                //【2】从已经分发的启动项列表 deliveredStarts 中移除并返回找到 startId 对应的启动项 StartItem；
                ServiceRecord.StartItem si = r.findDeliveredStart(startId, false);
                if (si != null) {
                    while (r.deliveredStarts.size() > 0) {
                        ServiceRecord.StartItem cur = r.deliveredStarts.remove(0);
                        cur.removeUriPermissionsLocked();
                        if (cur == si) {
                            break;
                        }
                    }
                }
                
                // 如果指定的启动项 id 不是最后自动服务的启动项目，那就不能停止服务！
                if (r.getLastStartId() != startId) {
                    return false;
                }

                if (r.deliveredStarts.size() > 0) {
                    Slog.w(TAG, "stopServiceToken startId " + startId
                            + " is last, but have " + r.deliveredStarts.size()
                            + " remaining args");
                }
            }

            synchronized (r.stats.getBatteryStats()) {
                r.stats.stopRunningLocked();
            }

            // 将服务的 startRequested 置为 false，表示未请求启动；
            r.startRequested = false;
            if (r.tracker != null) {
                r.tracker.setStarted(false, mAm.mProcessStats.getMemFactorLocked(),
                        SystemClock.uptimeMillis());
            }
            // 将服务的 callStart 置为false，表示未执行启动；
            r.callStart = false;
            final long origId = Binder.clearCallingIdentity();
            
            //【3】停止服务！
            bringDownServiceIfNeededLocked(r, false, false);
            Binder.restoreCallingIdentity(origId);
            return true;
        }
        return false;
    }
```
如果没指定启动项的 `id`，即：`startId >= 0`，那就要判断下该 `id` 是否对应了最后一个启动项，如果是，才会立刻停止！！
如果指定了启动项的 `id`，即：`startId < 0`，那就立刻停止服务；

最后调用了 `bringDownServiceIfNeededLocked` 来停止服务，这里的逻辑就和 `stopService` 中的逻辑一样了！

下面我们来看看其他的几个方法：

- 获得对应的 `ServiceRecord` 对象：
```
    private final ServiceRecord findServiceLocked(ComponentName name,
            IBinder token, int userId) {
        ServiceRecord r = getServiceByName(name, userId);
        return r == token ? r : null;
    }
```
- 找到 `id` 对应的启动项 `StartItem`： 
```
    public StartItem findDeliveredStart(int id, boolean remove) {
        final int N = deliveredStarts.size();
        for (int i=0; i<N; i++) {
            StartItem si = deliveredStarts.get(i);
            if (si.id == id) {
                if (remove) deliveredStarts.remove(i);
                return si;
            }
        }
        
        return null;
    }
```

这里就不在多书了，过程很清晰哦！