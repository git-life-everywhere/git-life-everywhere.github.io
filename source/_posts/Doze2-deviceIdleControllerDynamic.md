# Doze模式第 2 篇 - DeviceIdleController 动态机制
title: Doze模式第 2 篇 - DeviceIdleController 动态机制
date: 2017/11/25 20:46:25 
categories: 
- AndroidFramework源码分析
- Doze假寐模式
tags: Doze假寐模式
---


[toc]

基于 Android 7.1.1 源码，分析 doze 模式的原理！

# 0 综述

在 doze 模式下，应用会受到以下限制：

- 暂停访问 **network**。
- 系统将忽略 **wake locks**。
- 标准 **AlarmManager** 闹铃（包括 setExact() 和 setWindow()）推迟到 doze 模式的下一个 maintenance window 时间窗。
    - 如果您需要设置在低电耗模式下触发的闹铃，请使用 setAndAllowWhileIdle() 或 setExactAndAllowWhileIdle()。
    - 一般情况下，使用 setAlarmClock() 设置的闹铃将继续触发，但系统会在这些闹铃触发之前不久退出低电耗模式。
- 系统不执行 **Wi-Fi 扫描**。
- 系统不允许运行 **同步适配器**。
- 系统不允许运行 **JobScheduler**。

下面先分析下 doze 模式内部机制是如何调控的：

# 1 动态监听机制

## 1.1 monitor display

### 1.1.1 updateDisplayLocked

监控屏幕状态：

```java
    private final DisplayManager.DisplayListener mDisplayListener
            = new DisplayManager.DisplayListener() {
        @Override public void onDisplayAdded(int displayId) {
        }

        @Override public void onDisplayRemoved(int displayId) {
        }

        @Override public void onDisplayChanged(int displayId) {
            if (displayId == Display.DEFAULT_DISPLAY) {
                synchronized (DeviceIdleController.this) {
                    //【1】当屏幕状态发生变化后，会触发该方法！
                    updateDisplayLocked();
                }
            }
        }
    };
```
我们去看看 DeviceIdleController.updateDisplayLocked 方法！

updateDisplayLocked 用于更新屏幕的状态！

```java
    void updateDisplayLocked() {
        //【1】获取屏幕当前的状态，并判断是否是亮屏状态！
        mCurDisplay = mDisplayManager.getDisplay(Display.DEFAULT_DISPLAY);
        boolean screenOn = mCurDisplay.getState() == Display.STATE_ON;
        if (DEBUG) Slog.d(TAG, "updateDisplayLocked: screenOn=" + screenOn);
        if (!screenOn && mScreenOn) {
            //【2.1】如果是从亮屏转为熄屏，设置 mScreenOn 为 false！
            // 同时，尝试进入 Doze 模式；
            mScreenOn = false;
            if (!mForceIdle) {
                becomeInactiveIfAppropriateLocked(); 
            }
        } else if (screenOn) {
            //【2.2】如果是从熄屏转亮屏，设置 mScreenOn 为 true！
            // 同时，退出 Doze 模式；
            mScreenOn = true;
            if (!mForceIdle) {
                becomeActiveLocked("screen", Process.myUid());
            }
        }
    }
```
mForceIdle 表示是否强制进入 idle 状态，默认为 false 的，目前唯一的开启方式是通过 adb shell，执行 dumpsys 命令，触发 force-idle，force-inactive 相关指令，强制进入 idle 状态！！

这里看到：

- 当熄屏后，设置 mScreenOn 为 false，会调用 becomeInactiveIfAppropriateLocked 方法，进入 doze 模式；

- 当亮屏后，设置 mScreenOn 为 true，会调用 becomeActiveLocked 方法，退出 doze 模式；

## 1.2 monitor battey

### 1.2.1 updateChargingLocked

监控电池状态：
```java
    private final BroadcastReceiver mReceiver = new BroadcastReceiver() {
        @Override public void onReceive(Context context, Intent intent) {
            switch (intent.getAction()) {
                ... ... ...
                case Intent.ACTION_BATTERY_CHANGED: {
                    synchronized (DeviceIdleController.this) {
                        int plugged = intent.getIntExtra("plugged", 0);
                        updateChargingLocked(plugged != 0); //【1】电量变化
                    }
                } break;
                ... ... ...
            }
        }
    };
```
我们继续看 DeviceIdleController.updateChargingLocked 方法！
```java
    void updateChargingLocked(boolean charging) {
        if (DEBUG) Slog.i(TAG, "updateChargingLocked: charging=" + charging);
        if (!charging && mCharging) {
            //【1】结束充电！
            mCharging = false;
            if (!mForceIdle) {
                //【2.1】尝试进入 doze 模式！
                becomeInactiveIfAppropriateLocked();
            }
        } else if (charging) {
            //【2】开始充电！
            mCharging = charging;
            if (!mForceIdle) {
                //【2.2】退出 doze 模式！
                becomeActiveLocked("charging", Process.myUid());
            }
        }
    }
```
这里看到：

- 当结束充电后，设置 mCharging 为 false，会调用 becomeInactiveIfAppropriateLocked 方法，进入 doze 模式；

- 当开始充电后，设置 mCharging 为 true，会调用 becomeActiveLocked 方法，退出 doze 模式；


## 1.3 monitor connectivity

### 1.3.1 updateConnectivityState

监控网络连接状态：
```java
    private final BroadcastReceiver mReceiver = new BroadcastReceiver() {
        @Override public void onReceive(Context context, Intent intent) {
            switch (intent.getAction()) {
                case ConnectivityManager.CONNECTIVITY_ACTION: {
                    //【1】更新网络连接状态！
                    updateConnectivityState(intent);
                } break;
                ... ... ...
            }
        }
    };
```

我们继续看 DeviceIdleController.updateConnectivityState 方法！
```java
    void updateConnectivityState(Intent connIntent) {
        ConnectivityService cm;
        synchronized (this) {
            cm = mConnectivityService;
        }
        if (cm == null) {
            return;
        }
        NetworkInfo ni = cm.getActiveNetworkInfo();
        synchronized (this) {
            boolean conn;
            if (ni == null) {
                conn = false; //【1】网络断开，conn 为 false；
            } else {
                //【2】获得网络的连接状态；
                if (connIntent == null) {
                    conn = ni.isConnected();
                } else {
                    final int networkType =
                            connIntent.getIntExtra(ConnectivityManager.EXTRA_NETWORK_TYPE,
                                    ConnectivityManager.TYPE_NONE);
                    if (ni.getType() != networkType) {
                        return;
                    }
                    conn = !connIntent.getBooleanExtra(ConnectivityManager.EXTRA_NO_CONNECTIVITY,
                            false);
                }
            }
            //【3】处理连接状态，如果当前状态和之前的状态发生了变化，更新 mNetworkConnected 的值！
            if (conn != mNetworkConnected) {
                mNetworkConnected = conn;
                if (conn && mLightState == LIGHT_STATE_WAITING_FOR_NETWORK) {
                    //【2.3】如果本次状态是处于连接中，并且 mLightState 值为 LIGHT_STATE_WAITING_FOR_NETWORK！
                    // 调用 stepLightIdleStateLocked 方法！
                    stepLightIdleStateLocked("network");
                }
            }
        }
    }
```
mNetworkConnected 表示之前的网络连接！

## 1.4 monitor Motion

### 1.4.1 startMonitoringMotionLocked

监听设备的运动：

```java
    void startMonitoringMotionLocked() {
        if (DEBUG) Slog.d(TAG, "startMonitoringMotionLocked()");
        if (mMotionSensor != null && !mMotionListener.active) {
            mMotionListener.registerLocked();
        }
    }
```
DeviceIdleController 内部有一个 mMotionListener 对象，用于监听传感器的变化！

```java
private final MotionListener mMotionListener = new MotionListener();
```
我们来看下 MotionListener 中的方法！

#### 1.4.1.1 mMotionListener.registerLocked
```java
    private final class MotionListener extends TriggerEventListener
            implements SensorEventListener {
        boolean active = false; // 表示 MotionListener 是否被注册！
        
        //【1】当传感器的被触发后，会调用 onTrigger()
        @Override
        public void onTrigger(TriggerEvent event) {
            synchronized (DeviceIdleController.this) {
                active = false;
                //【1.4.1.2】调用 motionLocked 方法！
                motionLocked();
            }
        }
        
        //【2】当传感器的值发生变化时，会调用 OnSensorChanged()
        @Override
        public void onSensorChanged(SensorEvent event) {
            synchronized (DeviceIdleController.this) {
                mSensorManager.unregisterListener(this, mMotionSensor);
                active = false;
                //【1.4.1.2】调用 motionLocked 方法！
                motionLocked();
            }
        }

        // 当传感器的精度发生变化时会调用 OnAccuracyChanged() 方法!
        @Override
        public void onAccuracyChanged(Sensor sensor, int accuracy) {}

        //【3】注册 MotionListener！
        public boolean registerLocked() {
            boolean success;
            //【1.1】调用了 mSensorManager 的相关方法注册监听器！
            if (mMotionSensor.getReportingMode() == Sensor.REPORTING_MODE_ONE_SHOT) {
                success = mSensorManager.requestTriggerSensor(mMotionListener, mMotionSensor);
            } else {
                success = mSensorManager.registerListener(
                        mMotionListener, mMotionSensor, SensorManager.SENSOR_DELAY_NORMAL);
            }
            if (success) {
                active = true; //【1.2】设置监听器为活跃状态！
            } else {
                Slog.e(TAG, "Unable to register for " + mMotionSensor);
            }
            return success;
        }

        //【4】解除注册 MotionListener！
        public void unregisterLocked() {
            if (mMotionSensor.getReportingMode() == Sensor.REPORTING_MODE_ONE_SHOT) {
                mSensorManager.cancelTriggerSensor(mMotionListener, mMotionSensor);
            } else {
                mSensorManager.unregisterListener(mMotionListener);
            }
            active = false;
        }
    }
```
继续看：

#### 1.4.1.2 DeviceIdleC.motionLocked

```java
    void motionLocked() {
        if (DEBUG) Slog.d(TAG, "motionLocked()");
        //【1.4.1.3】调用 handleMotionDetectedLocked 处理传感器变化！
        handleMotionDetectedLocked(mConstants.MOTION_INACTIVE_TIMEOUT, "motion");
    }
```
#### 1.4.1.3 DeviceIdleC.handleMotionDetectedLocked

由于此时传感器被触发或者值发生了变化，那么此时我们不能进入 doze 模式，需要回退到 active 状态！！

```java
    void handleMotionDetectedLocked(long timeout, String type) {
        boolean becomeInactive = false;
        //【1】处理 deep idle 模式的退出！！
        if (mState != STATE_ACTIVE) {
            //【2.2.1】调用了 scheduleReportActiveLocked，退出 doze 模式，恢复 active 状态！
            scheduleReportActiveLocked(type, Process.myUid());
            mState = STATE_ACTIVE;
            mInactiveTimeout = timeout;
            mCurIdleBudget = 0;
            mMaintenanceStartTime = 0;
            
            EventLogTags.writeDeviceIdle(mState, type);
            addEvent(EVENT_NORMAL);

            becomeInactive = true;
        }
        //【1】处理 light idle 模式的退出，我们只有在退出 deep idle 时才会同时退出 light idle！！
        if (mLightState == LIGHT_STATE_OVERRIDE) {
            mLightState = STATE_ACTIVE;
            EventLogTags.writeDeviceIdleLight(mLightState, type);
            becomeInactive = true;
        }

        //【2.1】这里 becomeInactive 为 true 时，会重新调用 becomeInactiveIfAppropriateLocked 方法
        // 重新开始了阶段条件的判断！
        if (becomeInactive) {
            becomeInactiveIfAppropriateLocked();
        }
    }
```
这里就先分析到这里！

## 1.5 monitor Angle

### 1.5.1 AnyMotionDetector.checkForAnyMotion

监听设备的角度变化，我们在 onStart 方法中有创建一个 AnyMotionDetector 实例，用于监听设备的角度变化是否超过阈值！

```java
    //【1】获得角度阈值；
    float angleThreshold = getContext().getResources().getInteger(
            com.android.internal.R.integer.config_autoPowerModeThresholdAngle) / 100f;
    //【2】创建角度变化监听器；
    mAnyMotionDetector = new AnyMotionDetector(
            (PowerManager) getContext().getSystemService(Context.POWER_SERVICE),
            mHandler, mSensorManager, this, angleThreshold);
```

这里面最关键的一个参数是 AnyMotionDetector 的倒数第二个参数：DeviceIdleCallback callback，这里传入的是 DeviceIdleController，因为 DeviceIdleController 实现了 AnyMotionDetector.DeviceIdleCallback！

```java
public class AnyMotionDetector {
    interface DeviceIdleCallback {
        //【1.5.2】onAnyMotionResult 处理角度的变化；
        public void onAnyMotionResult(int result);
    }
    ... ... ...
}
```
当角度发生变化后，会触发 onAnyMotionResult 回调！

### 1.5.2 DeviceIdleC.onAnyMotionResult

我们来看看 onAnyMotionResult 方法！
```java
    @Override
    public void onAnyMotionResult(int result) {
        if (DEBUG) Slog.d(TAG, "onAnyMotionResult(" + result + ")");
        //【1】正常情况，首先取消 mSensingTimeoutAlarmListener；
        if (result != AnyMotionDetector.RESULT_UNKNOWN) {
            synchronized (this) {
                //【1.1.1.1.2】调用 cancelSensingTimeoutAlarmLocked 取消；
                cancelSensingTimeoutAlarmLocked();
            }
        }

        //【2】当 result 为 RESULT_MOVED，说明角度变化超过了阈值；
        if ((result == AnyMotionDetector.RESULT_MOVED) ||
            (result == AnyMotionDetector.RESULT_UNKNOWN)) {
            synchronized (this) {
                //【1.4.3】不处于静止状态，又调用了 handleMotionDetectedLocked 方法，恢复 active 状态；
                handleMotionDetectedLocked(mConstants.INACTIVE_TIMEOUT, "non_stationary");
            }

        } else if (result == AnyMotionDetector.RESULT_STATIONARY) {
            //【3】当 result 为 RESULT_STATIONARY，说明角度并没有发生超过阈值的变化；
            if (mState == STATE_SENSING) {
                //【3.1】当 mState 是 STATE_SENSING，那么我们会进入下一阶段；
                synchronized (this) {
                    // 设置 STATE_SENSING 为 true！
                    mNotMoving = true; 
                    stepIdleStateLocked("s:stationary");
                }

            } else if (mState == STATE_LOCATING) {
                //【3.1】当 mState 是 STATE_LOCATING，那么我们同样会进入下一阶段；
                synchronized (this) {
                    mNotMoving = true;
                    if (mLocated) {
                        stepIdleStateLocked("s:stationary");
                    }
                }

            }
        }
    }
```


## 1.6 monitor Location - 监听位置

监控定位：

```java
    //【1】监听网络定位 network location provider！
    if (mLocationManager != null
            && mLocationManager.getProvider(LocationManager.NETWORK_PROVIDER) != null) {
        mLocationManager.requestLocationUpdates(mLocationRequest,
                mGenericLocationListener, mHandler.getLooper()); //【1.6.1】
        mLocating = true;
    } else {
        mHasNetworkLocation = false;
    }

    //【2】监听 GPS location provider！
    if (mLocationManager != null
            && mLocationManager.getProvider(LocationManager.GPS_PROVIDER) != null) {
        mHasGps = true;
        mLocationManager.requestLocationUpdates(LocationManager.GPS_PROVIDER, 1000, 5,
                mGpsLocationListener, mHandler.getLooper());  //【1.6.2】
        mLocating = true;
    } else {
        mHasGps = false;
    }
```
我们看到，这里向 LocationManager 中注册了两个监听器：mGenericLocationListener 和 mGpsLocationListener！

### 1.6.1 mGenericLocationListener.onLocationChanged
```java
    private final LocationListener mGenericLocationListener = new LocationListener() {
        @Override
        public void onLocationChanged(Location location) {
            synchronized (DeviceIdleController.this) {
                //【1.6.1.1】处理定位的变化！
                receivedGenericLocationLocked(location);
            }
        }
        ... ... ... ... // 省去无需关注！
    };
```
当定位发生变化后，onLocationChanged 触发！

#### 1.6.1.1 DeviceIdleC.receivedGenericLocationLocked
```java
    void receivedGenericLocationLocked(Location location) {
        //【1】判断下 mState，如果不是 STATE_LOCATING，就调用 cancelLocatingLocked，取消定位监听！
        if (mState != STATE_LOCATING) {
            cancelLocatingLocked();
            return;
        }
        if (DEBUG) Slog.d(TAG, "Generic location: " + location);
        //【2】将定位信息保存到 mLastGenericLocation 中，用于 dumpsys！
        mLastGenericLocation = new Location(location);
        // 如果定位精度超过 mConstants.LOCATION_ACCURACY(20meter)，直接返回，不进入下一阶段；
        if (location.getAccuracy() > mConstants.LOCATION_ACCURACY && mHasGps) {
            return;
        }
        //【3】设置 mLocated 为 true，表示定位完成！
        mLocated = true;
        if (mNotMoving) {
            //【3.1】如果此时手机没有发生移动，调用 stepIdleStateLocked 处理 STATE_LOCATING 状态！
            stepIdleStateLocked("s:location");
        }
    }
```
不多说！

### 1.6.2 mGpsLocationListener.onLocationChanged
```
    private final LocationListener mGpsLocationListener = new LocationListener() {
        @Override
        public void onLocationChanged(Location location) {
            synchronized (DeviceIdleController.this) {
                //【1.6.2.1】处理定位的变化！
                receivedGpsLocationLocked(location);
            }
        }
        ... ... ... ... ... // 省去无需关注！
    };
```
当定位发生变化后，onLocationChanged 触发！

#### 1.6.2.1 DeviceIdleC.receivedGpsLocationLocked

```java
    void receivedGpsLocationLocked(Location location) {
        //【1】判断下 mState，如果不是 STATE_LOCATING，就调用 cancelLocatingLocked，取消定位监听！
        if (mState != STATE_LOCATING) {
            cancelLocatingLocked();
            return;
        }
        if (DEBUG) Slog.d(TAG, "GPS location: " + location);
        //【2】将定位信息保存到 mLastGpsLocation 中，用于 dumpsys！
        mLastGpsLocation = new Location(location);
        // 如果定位精度超过 mConstants.LOCATION_ACCURACY(20meter)，直接返回，不进入下一阶段；
        if (location.getAccuracy() > mConstants.LOCATION_ACCURACY) {
            return;
        }
        //【3】设置 mLocated 为 true，表示定位完成！
        mLocated = true;
        if (mNotMoving) {
            //【3.1】如果此时手机没有发生移动，调用 stepIdleStateLocked 处理 STATE_LOCATING 状态！
            stepIdleStateLocked("s:gps");
        }
    }
```
不多说！

# 2 核心逻辑分析

## 2.1 DeviceIdleC.becomeInactiveIfAppropriateLocked

尝试进入 doze 模式，这里会对 doze 模式第一阶段的条件做一个处理！！
```java
    void becomeInactiveIfAppropriateLocked() {
        if (DEBUG) Slog.d(TAG, "becomeInactiveIfAppropriateLocked()");
        //【1】如果是熄屏并且没有充电的充电的情况下，或者是强制进入 idle 状态下！
        if ((!mScreenOn && !mCharging) || mForceIdle) {
            //【1.1】处理 deep idle 模式逻辑！！
            if (mState == STATE_ACTIVE && mDeepEnabled) {
                mState = STATE_INACTIVE;
                if (DEBUG) Slog.d(TAG, "Moved from STATE_ACTIVE to STATE_INACTIVE");
                //【5.1】重置相关变量！！
                resetIdleManagementLocked();
                //【2.1.1.1】设置 alarm，用于进入下一阶段！！
                scheduleAlarmLocked(mInactiveTimeout, false);

                EventLogTags.writeDeviceIdle(mState, "no activity");
            }
            //【1.2】处理 light idle 模式逻辑！！
            if (mLightState == LIGHT_STATE_ACTIVE && mLightEnabled) {
                mLightState = LIGHT_STATE_INACTIVE;
                if (DEBUG) Slog.d(TAG, "Moved from LIGHT_STATE_ACTIVE to LIGHT_STATE_INACTIVE");
                //【5.2】重置相关变量！！
                resetLightIdleManagementLocked();
                //【2.1.2.1】设置 alarm！！
                scheduleLightAlarmLocked(mConstants.LIGHT_IDLE_AFTER_INACTIVE_TIMEOUT);

                EventLogTags.writeDeviceIdleLight(mLightState, "no activity");
            }
        }
    }
```
首先判断了条件：

 - 如果是熄屏并且没有充电的充电的情况下，或者是强制进入 idle 状态下！

然后针对 deep idle 和 light idle 两种模式，做了不同的处理！


### 2.1.1 deep idle 逻辑

首先，设置 mState 从 STATE_ACTIVE 为 STATE_INACTIVE；

然后，重置 deep idle 逻辑相关的变量，取消 Alarm 和监控器！！

最后，设置 deep idle 的 Alarm：

#### 2.1.1.1 scheduleAlarmLocked

设置 deep idle 的 Alarm：

参数传递：

- long delay 传入的是 mInactiveTimeout，mInactiveTimeout 在 onStat 方法中被初始化为 mConstants.INACTIVE_TIMEOUT 为 30 mins;

- boolean idleUntil 传入的是 false，因为还没有进入 idle 状态！

```java
    void scheduleAlarmLocked(long delay, boolean idleUntil) {
        if (DEBUG) Slog.d(TAG, "scheduleAlarmLocked(" + delay + ", " + idleUntil + ")");
        if (mMotionSensor == null) {
            //【1】如果当前设备没有运动传感器，那就不会设置 alarm，因为无法判断设备是否移动！
            return;
        }
        //【2】计算 alarm 的触发时间！
        mNextAlarmTime = SystemClock.elapsedRealtime() + delay;
        if (idleUntil) {
            mAlarmManager.setIdleUntil(AlarmManager.ELAPSED_REALTIME_WAKEUP,
                    mNextAlarmTime, "DeviceIdleController.deep", mDeepAlarmListener, mHandler);
        } else {
            //【2.1.1.2】idleUntil 为 false，设置一个 delay 时间间隔后的 Alarm！
            mAlarmManager.set(AlarmManager.ELAPSED_REALTIME_WAKEUP,
                    mNextAlarmTime, "DeviceIdleController.deep", mDeepAlarmListener, mHandler);
        }
    }
```
在 delay 时间间隔后会触发 mDeepAlarmListener，在 mHandler 所在的线程执行！

#### 2.1.1.2 mDeepAlarmListener.onAlarm
```java
    private final AlarmManager.OnAlarmListener mDeepAlarmListener
            = new AlarmManager.OnAlarmListener() {
        @Override
        public void onAlarm() {
            synchronized (DeviceIdleController.this) {

                //【2.3】第一阶段条件满足！
                stepIdleStateLocked("s:alarm");
            }
        }
    };
```
当设备处于熄屏并且未充电 30 mins 以后，DeepAlarmListener 会被触发，这时 deep idle 模式的第一阶段条件满足，执行 stepIdleStateLocked 方法！

### 2.1.2 light idle 逻辑

回顾下流程：

- 首先，设置 mLightState 从 LIGHT_STATE_ACTIVE 变为 LIGHT_STATE_INACTIVE；
- 然后，同样的重置相关变量；
- 最后，设置 light idle 的 alarm；

#### 2.1.2.1 scheduleLightAlarmLocked

设置 light idle 的 Alarm，参数 long delay 传入：mConstants.LIGHT_IDLE_AFTER_INACTIVE_TIMEOUT 为 5 mins！
```java
    void scheduleLightAlarmLocked(long delay) {
        if (DEBUG) Slog.d(TAG, "scheduleLightAlarmLocked(" + delay + ")");
        mNextLightAlarmTime = SystemClock.elapsedRealtime() + delay;

        //【2.1.2.2】设置一个时间间隔为 delay 后的 alarm，用于触发 mLightAlarmListener！
        mAlarmManager.set(AlarmManager.ELAPSED_REALTIME_WAKEUP,
                mNextLightAlarmTime, "DeviceIdleController.light", mLightAlarmListener, mHandler);
    }
```
设置一个时间间隔为 delay 后的 alarm，用于触发 mLightAlarmListener！

#### 2.1.2.2 mLightAlarmListener.onAlarm
```java
    private final AlarmManager.OnAlarmListener mLightAlarmListener
            = new AlarmManager.OnAlarmListener() {
        @Override
        public void onAlarm() {
            synchronized (DeviceIdleController.this) {
                //【3.2】进入 light idle 模式的第一阶段条件满足！
                stepLightIdleStateLocked("s:alarm");
            }
        }
    };
```

当设备处于熄屏并且未充电 5 mins 以后，mLightAlarmListener.onAlarm 会被触发，这时 light idle 模式的第一阶段条件满足，执行 stepIdleStateLocked 方法！

## 2.2 DeviceIdleC.becomeActiveLocked - 退出 doze 模式

退出 doze 模式！

```java
    void becomeActiveLocked(String activeReason, int activeUid) {
        if (DEBUG) Slog.i(TAG, "becomeActiveLocked, reason = " + activeReason);
        //【1】如果 mState 不等于 STATE_ACTIVE 或者 mLightState 不等于 STATE_ACTIVE！
        if (mState != STATE_ACTIVE || mLightState != STATE_ACTIVE) {
            EventLogTags.writeDeviceIdle(STATE_ACTIVE, activeReason);
            EventLogTags.writeDeviceIdleLight(LIGHT_STATE_ACTIVE, activeReason);
            
            //【2.2.1】调用了 scheduleReportActiveLocked，退出 doze 模式，恢复 active 状态！
            scheduleReportActiveLocked(activeReason, activeUid);
            
            // 重置关键变量！
            mState = STATE_ACTIVE;
            mLightState = LIGHT_STATE_ACTIVE;
            mInactiveTimeout = mConstants.INACTIVE_TIMEOUT;
            mCurIdleBudget = 0;
            mMaintenanceStartTime = 0;
            
            //【5.1】重置 deep idle 模式
            resetIdleManagementLocked();
            //【5.2】重置 light idle 模式
            resetLightIdleManagementLocked();
            addEvent(EVENT_NORMAL);
        }
    }
```

首先判断了下状态：

- 如果 mState 不等于 STATE_ACTIVE 或者 mLightState 不等于 STATE_ACTIVE，执行退出操作！
- 设置 mState 为 STATE_ACTIVE；
- 设置 mLightState 为 LIGHT_STATE_ACTIVE；


### 2.2.1 scheduleReportActiveLocked
```java
    void scheduleReportActiveLocked(String activeReason, int activeUid) {
        //【4.4】发送 MSG_REPORT_ACTIVE 给 MyHandler，退出 doze 模式！
        Message msg = mHandler.obtainMessage(MSG_REPORT_ACTIVE, activeUid, 0, activeReason);
        mHandler.sendMessage(msg);
    }
```

# 3 Doze 模式状态处理

## 3.1 DeviceIdleC.stepIdleStateLocked

对于 deep idle 模式来说，是通过 stepIdleStateLocked 方法处理每个阶段的状态的！

```java
    void stepIdleStateLocked(String reason) {
        if (DEBUG) Slog.d(TAG, "stepIdleStateLocked: mState=" + mState);
        EventLogTags.writeDeviceIdleStep();

        final long now = SystemClock.elapsedRealtime();
        //【1】如果在 1 小时之内有会在 idle 状态下唤醒设备的 Alarm，那么我们不会进入 idle 状态！
        if ((now + mConstants.MIN_TIME_TO_ALARM) > mAlarmManager.getNextWakeFromIdleTime()) {
            if (mState != STATE_ACTIVE) {
                //【1.1】退出 doze 模式！
                becomeActiveLocked("alarm", Process.myUid());
                //【2.1】再次调用 becomeInactiveIfAppropriateLocked 重新从第一阶段开始处理！
                becomeInactiveIfAppropriateLocked();
            }
            return;
        }

        //【2】下面进入状态的判断！
        switch (mState) {
            case STATE_INACTIVE:
                break;
                
            case STATE_IDLE_PENDING:
                break;
                
            case STATE_SENSING:

            case STATE_LOCATING:

            case STATE_IDLE_MAINTENANCE:
                break;

            case STATE_IDLE:
                break;
        }
    }

```

我们来重点看看，deep idle 状态处理如下：

### 3.1.1 状态：STATE_INACTIVE

此时，设备已经在熄屏且为充电的状态下 30mins 了，这个时候，准备进入第二阶段：

```java
                //【1.4】启动传感器监听，判断设备是否有移动！
                startMonitoringMotionLocked();
                //【2.1.1.2】启动下一个阶段的 alarm 触发！
                scheduleAlarmLocked(mConstants.IDLE_AFTER_INACTIVE_TIMEOUT, false);
                
                // Reset the upcoming idle delays.
                mNextIdlePendingDelay = mConstants.IDLE_PENDING_TIMEOUT;
                mNextIdleDelay = mConstants.IDLE_TIMEOUT;
                
                // 设置状态为 STATE_IDLE_PENDING
                mState = STATE_IDLE_PENDING; 
```

- 流程分析：

通过 startMonitoringMotionLocked 启动了传感器监听，监听设备是否运动！

又**设置了一个时间间隔为 mConstants.IDLE_AFTER_INACTIVE_TIMEOUT 30mins 的 Alarm**，触发的仍然是 mDeepAlarmListener！

**mState 状态被设置为了 STATE_IDLE_PENDING**！

- 第二阶段条件：

**这里我们看到了第二阶段条件：熄屏且不充电，同时设备不移动 30mins 后**！

条件满足后，依然会调用 stepIdleStateLocked 方法！


### 3.1.2 状态：STATE_IDLE_PENDING


此时，设备熄屏且不充电 60mins，同时设备不移动 30mins 了，这个时候，准备进入第三阶段：

```java
            case STATE_IDLE_PENDING:
                //【2.2】熄屏且不充电 60mins，设备不移动 30mins 后，第二阶段条件满足，mState 为 STATE_IDLE_PENDING，
                // 那么我们会进入第三阶段！
                mState = STATE_SENSING;
                if (DEBUG) Slog.d(TAG, "Moved from STATE_IDLE_PENDING to STATE_SENSING.");
                EventLogTags.writeDeviceIdle(mState, reason);
                //【3.1.2.1】启动角度监听！
                scheduleSensingTimeoutAlarmLocked(mConstants.SENSING_TIMEOUT);
                //【1.1.1.1.3】取消定位监听，因为定位是下一个阶段的！
                cancelLocatingLocked();
                
                // 初始化一些变量！
                mNotMoving = false;
                mLocated = false;
                mLastGenericLocation = null;
                mLastGpsLocation = null;

                //【1.5】启动角度变化监听！
                mAnyMotionDetector.checkForAnyMotion();
                break;
```
- 流程分析：

通过 startMonitoringMotionLocked 启动了传感器监听，监听设备是否运动！

又**设置了一个时间间隔为 mConstants.IDLE_AFTER_INACTIVE_TIMEOUT 30mins 的 Alarm，触发的仍然是 mDeepAlarmListener**！

**mState 状态被设置为了 STATE_SENSING**！

- 第三阶段条件：

**熄屏且不充电，设备不移动，设备没发生角度变化 4 mins**，以后！

#### 3.1.2.1 DeviceIdleC.scheduleSensingTimeoutAlarmLocked

接着设置第三阶段的 Alarm，参数传入的是 mConstants.SENSING_TIMEOUT 4mins：
```java
    void scheduleSensingTimeoutAlarmLocked(long delay) {
        if (DEBUG) Slog.d(TAG, "scheduleSensingAlarmLocked(" + delay + ")");

        mNextSensingTimeoutAlarmTime = SystemClock.elapsedRealtime() + delay;
        //【3.1.2.2】4mins 后触发 mSensingTimeoutAlarmListener！
        mAlarmManager.set(AlarmManager.ELAPSED_REALTIME_WAKEUP, mNextSensingTimeoutAlarmTime,
            "DeviceIdleController.sensing", mSensingTimeoutAlarmListener, mHandler);
    }
```
接着，设置了一个新的 Alarm，Alarm 触发后，会执行 mSensingTimeoutAlarmListener.onAlarm 方法！

#### 3.1.2.2 mSensingTimeoutAlarmListener.onAlarm
```java
    private final AlarmManager.OnAlarmListener mSensingTimeoutAlarmListener
            = new AlarmManager.OnAlarmListener() {
        @Override
        public void onAlarm() {
            if (mState == STATE_SENSING) {
                synchronized (DeviceIdleController.this) {
                    //【2.1】进入下一个阶段！
                    becomeInactiveIfAppropriateLocked();
                }
            }
        }
    };
```
当 mSensingTimeoutAlarmListener 触发后，继续进入了 becomeInactiveIfAppropriateLocked 方法！


### 3.1.3 状态：STATE_SENSING and STATE_LOCATING and STATE_IDLE_MAINTENANCE

接下来，我们把 STATE_SENSING 和 STATE_LOCATING 和 STATE_IDLE_MAINTENANCE 放在一起看，因为这几个有关联！

```java
            case STATE_SENSING:
                //【2.3】熄屏且不充电，设备不移动，没发生角度变化 4 mins 后，第三阶段条件满足，
                // 此时 mState 为 STATE_SENSING，
                // 那么我们会进入第四阶段！
                //【1.1.1.1.2】取消 mSensingTimeoutAlarmListener！
                cancelSensingTimeoutAlarmLocked();
                
                // 设置 mState 为 STATE_LOCATING！
                mState = STATE_LOCATING;
                
                if (DEBUG) Slog.d(TAG, "Moved from STATE_SENSING to STATE_LOCATING.");
                EventLogTags.writeDeviceIdle(mState, reason);
                
                //【2.1.1.2】设置该阶段的 Alarm，时间间隔为 mConstants.LOCATING_TIMEOUT 30s！
                scheduleAlarmLocked(mConstants.LOCATING_TIMEOUT, false);
                
                // 注册定位监听器，如果注册成功，mLocating 为 true！
                if (mLocationManager != null
                        && mLocationManager.getProvider(LocationManager.NETWORK_PROVIDER) != null) {
                    mLocationManager.requestLocationUpdates(mLocationRequest,
                            mGenericLocationListener, mHandler.getLooper());
                    mLocating = true;
                } else {
                    mHasNetworkLocation = false;
                }
                if (mLocationManager != null
                        && mLocationManager.getProvider(LocationManager.GPS_PROVIDER) != null) {
                    mHasGps = true;
                    mLocationManager.requestLocationUpdates(LocationManager.GPS_PROVIDER, 1000, 5,
                            mGpsLocationListener, mHandler.getLooper());
                    mLocating = true;
                } else {
                    mHasGps = false;
                }


                // 如果注册成功， mLocating 为 true，那么就等定位的结果返回
                // 如果注册失败，系统中没有 location provider，那就直接进入 STATE_LOCATING 状态！
                if (mLocating) {
                    break;
                }

            case STATE_LOCATING:
                //【2.4】处理状态 STATE_LOCATING，这里会取消 Alarm，然后会直接进入 STATE_IDLE_MAINTENANCE
                //【5.1.1】取消 mDeepAlarmListener；
                cancelAlarmLocked();
                //【5.1.3】取消 mGenericLocationListener，mGpsLocationListener；
                cancelLocatingLocked();
                // 停止监听角度变化！
                mAnyMotionDetector.stop();

            case STATE_IDLE_MAINTENANCE:
                //【2.5】处理状态 STATE_IDLE_MAINTENANCE，这里设置新的 Alarm，用于进入下一个 STATE_IDLE_MAINTENANCE；
                // 时间检测是 mNextIdleDelay，注意这里第二个参数是 true，会触发 mAlarmManager.setIdleUntil 方法！
                scheduleAlarmLocked(mNextIdleDelay, true);
                if (DEBUG) Slog.d(TAG, "Moved to STATE_IDLE. Next alarm in " + mNextIdleDelay +
                        " ms.");
                
                // 计算下一个 mNextIdleDelay 时间间隔，在当前的 mNextIdleDelay 基础上，乘以时间因子！
                // 然后在 mNextIdleDelay 和 mConstants.MAX_IDLE_TIMEOUT(6h) 中选择最小的值
                // 最后，在 mNextIdleDelay 和 mConstants.IDLE_TIMEOUT(60min) 中选择最大值
                mNextIdleDelay = (long)(mNextIdleDelay * mConstants.IDLE_FACTOR);
                if (DEBUG) Slog.d(TAG, "Setting mNextIdleDelay = " + mNextIdleDelay);
                mNextIdleDelay = Math.min(mNextIdleDelay, mConstants.MAX_IDLE_TIMEOUT);
                if (mNextIdleDelay < mConstants.IDLE_TIMEOUT) {
                    mNextIdleDelay = mConstants.IDLE_TIMEOUT;
                }
                
                // 同时，设置状态为 STATE_IDLE
                mState = STATE_IDLE;
                
                // 如果此时 light idle 的状态，不为 LIGHT_STATE_OVERRIDE，由于现在要进入 deep idle 状态了
                // 所以 light idle 会失效，这里会取消 light idle 的 Alarm！
                if (mLightState != LIGHT_STATE_OVERRIDE) {
                    mLightState = LIGHT_STATE_OVERRIDE;
                    cancelLightAlarmLocked();
                }
                EventLogTags.writeDeviceIdle(mState, reason);
                addEvent(EVENT_DEEP_IDLE);
                
                //【4.1】进入 idle 状态！
                mHandler.sendEmptyMessage(MSG_REPORT_IDLE_ON);
                break;
```

- **1. STATE_SENSING**

当熄屏且不充电，设备不移动，没发生角度变化 4 mins 后，就会进入 STATE_SENSING 阶段！

接着，会**设置 mState 为 STATE_LOCATING**，然后设置一个 30s 的 Alarm，如果在 30s 内，条件没有发生变化，那么会进入 STATE_LOCATING 阶段！

接着会注册定位监听器，如果注册成功，那就等待 30s 的 Alarm 触发；如果注册失败，那就直接进入 STATE_LOCATING；

所以该阶段设置的条件是：

- 熄屏且不充电，设备不移动，没发生角度变化后，开始定位，保持 30s 以后；
- 熄屏且不充电，设备不移动，没发生角度变化后，直接进入下一个条件；

- **2. STATE_LOCATING**

该状态会直接取消 mDeepAlarmListener，mGenericLocationListener，mGpsLocationListener，同时停止监听角度变化！

然后会立刻进入 STATE_IDLE_MAINTENANCE 状态！

- **3. STATE_IDLE_MAINTENANCE**

进入该阶段后，我们就即将进入 deep idle 状态了，接下来，如果所有条件都满足，那么，设备的状态将会在 STATE_IDLE_MAINTENANCE 和 STATE_IDLE 直接切换！

在 STATE_IDLE_MAINTENANCE 阶段，我们设置了一个 Alarm，时间间隔是 mNextIdleDelay, 在 STATE_INACTIVE 阶段 mNextIdleDelay 初始化为了

mConstants.IDLE_TIMEOUT(60mins)

第一次进入 STATE_IDLE_MAINTENANCE 阶段，我们的 Alarm 的时间间隔为 mConstants.IDLE_TIMEOUT(60mins)，接着对时间间隔 mNextIdleDelay 进行了调整，用于下一次设置 Alarm！

计算下一个 mNextIdleDelay 时间间隔，在当前的 mNextIdleDelay 基础上，乘以时间因子；然后在 mNextIdleDelay 和 mConstants.MAX_IDLE_TIMEOUT(6h) 中选择最小的值；在 mNextIdleDelay 和 mConstants.IDLE_TIMEOUT(60min) 中选择最大值
```java
        mNextIdleDelay = (long)(mNextIdleDelay * mConstants.IDLE_FACTOR);
        mNextIdleDelay = Math.min(mNextIdleDelay, mConstants.MAX_IDLE_TIMEOUT);
        if (mNextIdleDelay < mConstants.IDLE_TIMEOUT) {
            mNextIdleDelay = mConstants.IDLE_TIMEOUT;
        }
```

那么，这个时间间隔为 mNextIdleDelay 的 Alarm 作用是什么呢？我们知道，当设备进入 doze 模式后，隔一段时间后，会进入一个 maintenance window 时间窗，这时，设备会从 idle 状态暂时恢复，集中处理任务！

所以这个时间间隔 mNextIdleDelay，既是 doze 模式的时间间隔，又是进入 maintenance window 的时间间隔！

**设置 mState 为 STATE_IDLE**！

此时，设备已经进入了 idle 状态！

如果此时 light idle 的状态，不为 LIGHT_STATE_OVERRIDE，由于现在要进入 deep idle 状态了，所以 light idle 会失效，这里会取消 light idle 的 Alarm！

最后发送 MSG_REPORT_IDLE_ON 给 MyHandler，通知其他服务，设备进入了 idle 状态！

注意这里会调用 scheduleAlarmLocked 方法，但是第二个参数传入的是 true：

```java
    void scheduleAlarmLocked(long delay, boolean idleUntil) {
        if (idleUntil) {
            //【1】执行 setIdleUntil 方法！
            mAlarmManager.setIdleUntil(AlarmManager.ELAPSED_REALTIME_WAKEUP,
                    mNextAlarmTime, "DeviceIdleController.deep", mDeepAlarmListener, mHandler);
        }
    }
```
分析 AlarmManagerService 的时候，我们又说过 setIdleUntil，该方法会让 AlarmManagerService 进入 doze 模式，直到该 Alarm 触发！该 Alarm 触发后，状态会变为 STATE_IDLE_MAINTENANCE，也就是进入时间窗，同时 Alarm 恢复了！

### 3.1.4 状态：STATE_IDLE

当 mNextIdleDelay 时间间隔的 Alarm 触发后，设备会暂时退出 doze 模式，进入 maintenance window，集中处理任务！

```java
            case STATE_IDLE:
                //【2.5】处理状态 STATE_IDLE，这里会设置新的 Alarm，用于进入下一次 STATE_IDLE 状态！
                // 同时更新状态为 STATE_IDLE_MAINTENANCE，进入 maintenance window！
                mActiveIdleOpCount = 1; // 计数设置为 1；
                mActiveIdleWakeLock.acquire();
                scheduleAlarmLocked(mNextIdlePendingDelay, false);
                if (DEBUG) Slog.d(TAG, "Moved from STATE_IDLE to STATE_IDLE_MAINTENANCE. " +
                        "Next alarm in " + mNextIdlePendingDelay + " ms.");
                        
                //【1】计算进入 maintenance window 的时间点！
                mMaintenanceStartTime = SystemClock.elapsedRealtime();
                
                //【2】计算下一次进入 deep idle 状态的 Alarm 时间间隔！
                mNextIdlePendingDelay = Math.min(mConstants.MAX_IDLE_PENDING_TIMEOUT,
                        (long)(mNextIdlePendingDelay * mConstants.IDLE_PENDING_FACTOR));
                if (mNextIdlePendingDelay < mConstants.IDLE_PENDING_TIMEOUT) {
                    mNextIdlePendingDelay = mConstants.IDLE_PENDING_TIMEOUT;
                }
                
                //【3】设置状态为 STATE_IDLE_MAINTENANCE！
                mState = STATE_IDLE_MAINTENANCE;
                EventLogTags.writeDeviceIdle(mState, reason);
                addEvent(EVENT_DEEP_MAINTENANCE);
                
                //【4.1】退出 idle 状态！
                mHandler.sendEmptyMessage(MSG_REPORT_IDLE_OFF);
                break;
```
进入该阶段后，我们就即将进入 maintenance window 了！

mActiveIdleOpCount 用于统计进入 maintenance window 的次数！

在 STATE_IDLE 阶段，我们再次设置了一个 Alarm，时间间隔是 mNextIdlePendingDelay, 在 STATE_INACTIVE 阶段 mNextIdleDelay 初始化为了

mConstants.IDLE_TIMEOUT(60mins)

第一次进入 STATE_IDLE_MAINTENANCE 阶段，我们的 Alarm 的时间间隔为 mConstants.IDLE_PENDING_TIMEOUT(5mins)，接着对时间间隔 mNextIdlePendingDelay 进行了调整，用于下一次设置 Alarm！

- 计算下一个 mNextIdlePendingDelay 时间间隔：

在当前的 mNextIdlePendingDelay 基础上，乘以时间因子；然后在 mNextIdlePendingDelay 和 mConstants.MAX_IDLE_PENDING_TIMEOUT(10mins) 中选择最小的值；在 mNextIdlePendingDelay 和 mConstants.IDLE_PENDING_TIMEOUT(5mins) 中选择最大值：
```java
        mNextIdlePendingDelay = Math.min(mConstants.MAX_IDLE_PENDING_TIMEOUT,
                (long)(mNextIdlePendingDelay * mConstants.IDLE_PENDING_FACTOR));
        if (mNextIdlePendingDelay < mConstants.IDLE_PENDING_TIMEOUT) {
            mNextIdlePendingDelay = mConstants.IDLE_PENDING_TIMEOUT;
        }
```

那么，这个时间间隔为 mNextIdlePendingDelay 的 Alarm 作用是什么呢？

- 我们知道，当设备进入 maintenance window 时间窗后，会退出 idle 状态，这时，设备会集中处理任务，但是设备在 maintenance window 时间窗结束后，又要进入 deep idle 状态的！
- 所以这个时间间隔 mNextIdlePendingDelay，既是 deep idle 模式的maintenance window 时间窗长度，又是再次进入 idle 的时间间隔！

最后发送 MSG_REPORT_IDLE_OFF 给 MyHandler，同时其他服务，设备退出了 idle 状态！


## 3.2 DeviceIdleC.stepLightIdleStateLocked

看完了 deep idle 的状态调度，我们来看看 light idle 的状态调度！

```java
    void stepLightIdleStateLocked(String reason) {
        //【1】如果 mLightState 为 LIGHT_STATE_OVERRIDE，说明我们已经处于 deep idle 状态了！
        // 那么忽略 light idle！
        if (mLightState == LIGHT_STATE_OVERRIDE) {
            return;
        }

        if (DEBUG) Slog.d(TAG, "stepLightIdleStateLocked: mLightState=" + mLightState);
        EventLogTags.writeDeviceIdleLightStep();
        //【2】处理 mLightState 的状态！
        switch (mLightState) {
            case LIGHT_STATE_INACTIVE:
                ... ... ...
            case LIGHT_STATE_PRE_IDLE:
            case LIGHT_STATE_IDLE_MAINTENANCE:
                ... ... ...
                break;
                
            case LIGHT_STATE_IDLE:
            case LIGHT_STATE_WAITING_FOR_NETWORK:
                ... ... ...
                break;
        }
    }
```
继续看！

### 3.1.1 状态：LIGHT_STATE_INACTIVE

如果是熄屏并且没有充电的充电的情况下，或者是强制进入 idle 状态下，mLightState 会从 LIGHT_STATE_ACTIVE 变为 LIGHT_STATE_INACTIVE

同时会设置一个 Alarm，时间间隔为 mConstants.LIGHT_IDLE_AFTER_INACTIVE_TIMEOUT 为(5 mins)，Alarm 触发后 mLightAlarmListener.onAlarm 方法会被回调，进入 stepLightIdleStateLocked！

我们来看看 LIGHT_STATE_INACTIVE 状态的处理！
```java
            case LIGHT_STATE_INACTIVE:
                mCurIdleBudget = mConstants.LIGHT_IDLE_MAINTENANCE_MIN_BUDGET;

                //【1】重置了 mNextLightIdleDelay 和 mMaintenanceStartTime
                // 后面设置时间窗会用到！
                mNextLightIdleDelay = mConstants.LIGHT_IDLE_TIMEOUT;
                mMaintenanceStartTime = 0;
                
                //【2】判断当前设备是否还有任务和操作要执行！
                if (!isOpsInactiveLocked()) {
                    // 如果还有任务要执行，那就不能直接进入 LIGHT_STATE_PRE_IDLE 状态
                    // 而是设置了一个 Alarm，等待任务执行完！
                    //【2.1】设置 mLightState 为 LIGHT_STATE_PRE_IDLE！
                    mLightState = LIGHT_STATE_PRE_IDLE;
                    EventLogTags.writeDeviceIdleLight(mLightState, reason);
                    //【2.1.2.1】设置 Alarm！
                    scheduleLightAlarmLocked(mConstants.LIGHT_PRE_IDLE_TIMEOUT);
                    break;
                }
                //【3】没有任务在做，直接进入下一阶段！
            case LIGHT_STATE_PRE_IDLE:
```
这里调用了 isOpsInactiveLocked 方法，用于判断当前设备是否还有任务和操作要执行！

- 当其返回 false 时，说明当前设备还有任务和操作要执行！

   - 那么就不能直接进入 LIGHT_STATE_IDLE 状态，这里先设置为 LIGHT_STATE_PRE_IDLE，可以看作是 idle 前的一个临时状态！
   - 同时 scheduleLightAlarmLocked 设置了一个时间间隔为 mConstants.LIGHT_PRE_IDLE_TIMEOUT(10mins) 的 Alarm，等待那些要执行的工作执行完成，然后进入下一个阶段的处理！


- 当其返回 true 时，说明当前设备没有任务和操作要执行了！

   - 那么这种情况，直接进入 idle 状态，具体的逻辑处理是在 3.1.2 LIGHT_STATE_PRE_IDLE/LIGHT_STATE_IDLE_MAINTENANCE 里！


### 3.1.2 状态：LIGHT_STATE_PRE_IDLE and LIGHT_STATE_IDLE_MAINTENANCE

进入该阶段有两种情况：

- 设备没有任务和操作要执行，从 LIGHT_STATE_INACTIVE 直接进入；
- 设备没有任务和操作要执行，Alarm 触发后，从 LIGHT_STATE_PRE_IDLE 进入；

```java
            // Nothing active, fall through to immediately idle.
            case LIGHT_STATE_PRE_IDLE:
            case LIGHT_STATE_IDLE_MAINTENANCE:
                //【1】根据 mMaintenanceStartTime 调整 mCurIdleBudget！
                if (mMaintenanceStartTime != 0) {
                    long duration = SystemClock.elapsedRealtime() - mMaintenanceStartTime;
                    if (duration < mConstants.LIGHT_IDLE_MAINTENANCE_MIN_BUDGET) {
                        // We didn't use up all of our minimum budget; add this to the reserve.
                        mCurIdleBudget += (mConstants.LIGHT_IDLE_MAINTENANCE_MIN_BUDGET-duration);
                    } else {
                        // We used more than our minimum budget; this comes out of the reserve.
                        mCurIdleBudget -= (duration-mConstants.LIGHT_IDLE_MAINTENANCE_MIN_BUDGET);
                    }
                }
                mMaintenanceStartTime = 0;

                //【2】设置一个新的 Alarm，时间间隔为 mNextLightIdleDelay，mNextLightIdleDelay 默认为
                // mConstants.LIGHT_IDLE_TIMEOUT(5mins)
                scheduleLightAlarmLocked(mNextLightIdleDelay);

                //【3】计算下一次 Alarm 的时间间隔！
                mNextLightIdleDelay = Math.min(mConstants.LIGHT_MAX_IDLE_TIMEOUT,
                        (long)(mNextLightIdleDelay * mConstants.LIGHT_IDLE_FACTOR));
                if (mNextLightIdleDelay < mConstants.LIGHT_IDLE_TIMEOUT) {
                    mNextLightIdleDelay = mConstants.LIGHT_IDLE_TIMEOUT;
                }
                if (DEBUG) Slog.d(TAG, "Moved to LIGHT_STATE_IDLE.");
                
                //【4】设置 mLightState 为 LIGHT_STATE_IDLE，进入 light idle 状态！
                mLightState = LIGHT_STATE_IDLE;
                EventLogTags.writeDeviceIdleLight(mLightState, reason);
                addEvent(EVENT_LIGHT_IDLE);
                
                //【4.2】发送 MSG_REPORT_IDLE_ON_LIGHT 给 MyHandler，进入 light idle 状态！
                mHandler.sendEmptyMessage(MSG_REPORT_IDLE_ON_LIGHT);
                break;
```

该阶段，我们将 mLightState 设置为了 LIGHT_STATE_IDLE，表示已经进入了 light idle 状态！

接着，设置一个 Alarm，时间间隔为 mNextLightIdleDelay，第一次设置的时候 mNextLightIdleDelay 使用的默认值 mConstants.LIGHT_IDLE_TIMEOUT(5mins)！

然后计算下一个 mNextLightIdleDelay 时间间隔：

- 在当前的 mNextLightIdleDelay 基础上，乘以时间因子；
- 然后在 mNextLightIdleDelay 和 mConstants.LIGHT_MAX_IDLE_TIMEOUT(15mins) 中选择最小的值；
- 在 mNextIdleDelay 和 mConstants.LIGHT_IDLE_TIMEOUT(5mins) 中选择最大值：

```java
        mNextLightIdleDelay = Math.min(mConstants.LIGHT_MAX_IDLE_TIMEOUT,
                (long)(mNextLightIdleDelay * mConstants.LIGHT_IDLE_FACTOR));
        if (mNextLightIdleDelay < mConstants.LIGHT_IDLE_TIMEOUT) {
            mNextLightIdleDelay = mConstants.LIGHT_IDLE_TIMEOUT;
        }
```
这个时间间隔 mNextLightIdleDelay，既是 light idle 模式的 idle 状态时间间隔，又是进入 maintenance window 的时间间隔！


等待 alarm 触发，进入下一阶段！

### 3.1.3 状态：LIGHT_STATE_IDLE and LIGHT_STATE_WAITING_FOR_NETWORK

接着我们来看下对于 LIGHT_STATE_IDLE 和 LIGHT_STATE_WAITING_FOR_NETWORK 的处理，当 light idle 时间间隔过去后，会进入 maintenance window，执行任务！
```java
        case LIGHT_STATE_IDLE:
        case LIGHT_STATE_WAITING_FOR_NETWORK:
            if (mNetworkConnected || mLightState == LIGHT_STATE_WAITING_FOR_NETWORK) {
                //【1】如果此时网络连接，或者 mLightState 的状态为 LIGHT_STATE_WAITING_FOR_NETWORK
                // 此时 Alarm 已经触发，进入 maintenance window 时间窗，执行任务！
                mActiveIdleOpCount = 1; // mActiveIdleOpCount 置为 1；
                mActiveIdleWakeLock.acquire();
                
                //【2】计算当前的 maintenance window 的开始时间
                mMaintenanceStartTime = SystemClock.elapsedRealtime();
                if (mCurIdleBudget < mConstants.LIGHT_IDLE_MAINTENANCE_MIN_BUDGET) {
                    mCurIdleBudget = mConstants.LIGHT_IDLE_MAINTENANCE_MIN_BUDGET;
                } else if (mCurIdleBudget > mConstants.LIGHT_IDLE_MAINTENANCE_MAX_BUDGET) {
                    mCurIdleBudget = mConstants.LIGHT_IDLE_MAINTENANCE_MAX_BUDGET;
                }
                
                //【3】再次设置 Alarm，时间间隔为 mCurIdleBudget，该 Alarm 用于再次进入 idle 状态！
                scheduleLightAlarmLocked(mCurIdleBudget);
                
                if (DEBUG) Slog.d(TAG,
                        "Moved from LIGHT_STATE_IDLE to LIGHT_STATE_IDLE_MAINTENANCE.");
                        
                // 设置 mLightState 为 LIGHT_STATE_IDLE_MAINTENANCE，表示此时处于时间窗中！
                mLightState = LIGHT_STATE_IDLE_MAINTENANCE;

                EventLogTags.writeDeviceIdleLight(mLightState, reason);
                addEvent(EVENT_LIGHT_MAINTENANCE);
                
                //【4.3】发送 MSG_REPORT_IDLE_OFF 给 MyHandler，通知其他服务，退出 idle 状态！
                mHandler.sendEmptyMessage(MSG_REPORT_IDLE_OFF);

            } else {
                //【2】此时本应该进入 maintenance window，执行任务，但是由于没有网络连接，那么需要等待网络连接
                // 设置一个 Alarm，时间间隔为 mNextLightIdleDelay，等待网络连接！
                // 如果在 Alarm 触发之前，网络连接了，那么会再次触发 stepLightIdleStateLocked 方法！
                // 如果在 Alarm 触发时，网络仍然没有连接，那就会直接进入 maintenance window 时间窗，执行任务！
                scheduleLightAlarmLocked(mNextLightIdleDelay);
                
                if (DEBUG) Slog.d(TAG, "Moved to LIGHT_WAITING_FOR_NETWORK.");
                
                // 设置 mLightState 为 LIGHT_STATE_WAITING_FOR_NETWORK
                mLightState = LIGHT_STATE_WAITING_FOR_NETWORK;
                EventLogTags.writeDeviceIdleLight(mLightState, reason);
            }
            break;
```
我们分析下流程：

此时 idle 状态时间到了，需要进入时间窗执行任务，这里会对 network state 和 mLightState 进行一个判断：

- 如果 mNetworkConnected 为 true 或者 mLightState == LIGHT_STATE_WAITING_FOR_NETWOR：
    - 表示当前网络已经连接，或者当前网络没有连接但是 light idle 正在等待网络，那么就进入 maintenance window 时间窗，执行任务！
    - 同时设置 mLightState 为 LIGHT_STATE_IDLE_MAINTENANCE；
    - 接着设置了一个 再次设置 Alarm，时间间隔为 mCurIdleBudget，该 Alarm 用于再次进入 idle 状态！时间间隔 mCurIdleBudget 取值在 [mConstants.LIGHT_IDLE_MAINTENANCE_MIN_BUDGET(1mins), mConstants.LIGHT_IDLE_MAINTENANCE_MAX_BUDGET(5mins)] 之间！
    - 最后发送 MSG_REPORT_IDLE_OFF 给 MyHandler，通知其他服务，退出 idle 状态！！

</br>


- 如果 mNetworkConnected 为 false 且 mLightState != LIGHT_STATE_WAITING_FOR_NETWOR:
    - 那么这个时候，light idle 状态不会立刻进入 maintenance window 时间窗；
    - 会设置一个时间间隔 mNextLightIdleDelay 的 Alarm，等待网络连接；
    - 同时设置 mLightState 为 LIGHT_STATE_WAITING_FOR_NETWORK；
       - 如果在 Alarm 触发之前，网络连接了，那么会再次触发 stepLightIdleStateLocked 方法，进入 if 分支！
       - 如果在 Alarm 触发时，网络仍然没有连接，那就会直接进入 if 分支！ 
    

# 4 MyHandler 逻辑分析

我们来看下 MyHandler 逻辑分析：
```java
    final class MyHandler extends Handler {
        MyHandler(Looper looper) {
            super(looper);
        }

        @Override public void handleMessage(Message msg) {
            if (DEBUG) Slog.d(TAG, "handleMessage(" + msg.what + ")");
            switch (msg.what) {
                case MSG_WRITE_CONFIG: {
                } break;
                
                case MSG_REPORT_IDLE_ON:
                case MSG_REPORT_IDLE_ON_LIGHT: {

                } break;
                
                case MSG_REPORT_IDLE_OFF: {
                } break;
                
                case MSG_REPORT_ACTIVE: {
                } break;
                
                case MSG_TEMP_APP_WHITELIST_TIMEOUT: {
                } break;
                
                case MSG_REPORT_MAINTENANCE_ACTIVITY: {
                } break;
                
                case MSG_FINISH_IDLE_OP: {
                } break;
            }
        }
    }
```
下面分析下每个消息的具体处理！
## 4.1 消息：MSG_WRITE_CONFIG

更新本地持久化文件：
```java
        case MSG_WRITE_CONFIG: {
            //【4.1.1】持久化文件！
            handleWriteConfigFile();
        } break;
```

### 4.1.1 handleWriteConfigFile
更新本地持久化文件：

```java
    void handleWriteConfigFile() {
        final ByteArrayOutputStream memStream = new ByteArrayOutputStream();

        try {
            synchronized (this) {
                XmlSerializer out = new FastXmlSerializer();
                out.setOutput(memStream, StandardCharsets.UTF_8.name());
                writeConfigFileLocked(out);
            }
        } catch (IOException e) {
        }

        synchronized (mConfigFile) {
            FileOutputStream stream = null;
            try {
                stream = mConfigFile.startWrite();
                memStream.writeTo(stream);
                stream.flush();
                FileUtils.sync(stream);
                stream.close();
                mConfigFile.finishWrite(stream);
            } catch (IOException e) {
                Slog.w(TAG, "Error writing config file", e);
                mConfigFile.failWrite(stream);
            }
        }
    }
```
最后调用 writeConfigFileLocked 方法
```java
    void writeConfigFileLocked(XmlSerializer out) throws IOException {
        out.startDocument(null, true);
        out.startTag(null, "config");
        for (int i=0; i<mPowerSaveWhitelistUserApps.size(); i++) {
            String name = mPowerSaveWhitelistUserApps.keyAt(i);
            out.startTag(null, "wl");
            out.attribute(null, "n", name);
            out.endTag(null, "wl");
        }
        out.endTag(null, "config");
      
```
每一个白名单都是以 wl 标签开始和结束，n 属性为包名！

## 4.2 消息：MSG_REPORT_IDLE_ON and MSG_REPORT_IDLE_ON_LIGHT

进入 doze 模式，MSG_REPORT_IDLE_ON 表示的是 deep idle，而 MSG_REPORT_IDLE_ON_LIGHT 而是 light idle！

```java
        case MSG_REPORT_IDLE_ON:
        case MSG_REPORT_IDLE_ON_LIGHT: { // 进入 idle 状态！
            EventLogTags.writeDeviceIdleOnStart();
            final boolean deepChanged;
            final boolean lightChanged;
            //【1】针对于 deep 和 light，PowerManager 做不同的处理！
            if (msg.what == MSG_REPORT_IDLE_ON) {
                //【1.1】进入 deep idle！
                deepChanged = mLocalPowerManager.setDeviceIdleMode(true);
                lightChanged = mLocalPowerManager.setLightDeviceIdleMode(false);
            } else {
                //【1.2】进入 light idle！
                deepChanged = mLocalPowerManager.setDeviceIdleMode(false);
                lightChanged = mLocalPowerManager.setLightDeviceIdleMode(true);
            }
            //【2】NetworkPolicy 和 BatteryStats 设置不同的状态；
            try {
                mNetworkPolicyManager.setDeviceIdleMode(true);
                mBatteryStats.noteDeviceIdleMode(msg.what == MSG_REPORT_IDLE_ON
                        ? BatteryStats.DEVICE_IDLE_MODE_DEEP
                        : BatteryStats.DEVICE_IDLE_MODE_LIGHT, null, Process.myUid());
            } catch (RemoteException e) {
            }
            //【3】发送广播，给监听 doze 模式的进程！
            if (deepChanged) {
                getContext().sendBroadcastAsUser(mIdleIntent, UserHandle.ALL);
            }
            if (lightChanged) {
                getContext().sendBroadcastAsUser(mLightIdleIntent, UserHandle.ALL);
            }
            EventLogTags.writeDeviceIdleOnComplete();
        } break;
```
在进入 idle 状态的时候，我们会发送相应的广播！！

## 4.3 消息：MSG_REPORT_IDLE_OFF

MSG_REPORT_IDLE_OFF 表示暂时退出 idle 状态，进入了时间窗，执行任务！

```java
        case MSG_REPORT_IDLE_OFF: {
            EventLogTags.writeDeviceIdleOffStart("unknown");
            //【1】退出 deep idle 和 light idle 模式，更新 PowerManager 状态！
            final boolean deepChanged = mLocalPowerManager.setDeviceIdleMode(false);
            final boolean lightChanged = mLocalPowerManager.setLightDeviceIdleMode(false);
            try {
                mNetworkPolicyManager.setDeviceIdleMode(false);
                mBatteryStats.noteDeviceIdleMode(BatteryStats.DEVICE_IDLE_MODE_OFF,
                        null, Process.myUid());
            } catch (RemoteException e) {
            }

            //【2】如果 deep idle(light idle) 状态发生了变化，那就更新 mActiveIdleOpCount 计数；
            // 同时有序发送通知广播，最后自身也会接受广播！
            if (deepChanged) {
                //【4.3.1】增加引用计数 mActiveIdleOpCount！
                incActiveIdleOps();
                //【4.3.2】自身也会监听 doze 模式变化的广播！
                getContext().sendOrderedBroadcastAsUser(mIdleIntent, UserHandle.ALL,
                        null, mIdleStartedDoneReceiver, null, 0, null, null);
            }
            if (lightChanged) {
                incActiveIdleOps(); // 同上！
                getContext().sendOrderedBroadcastAsUser(mLightIdleIntent, UserHandle.ALL,
                        null, mIdleStartedDoneReceiver, null, 0, null, null);
            }

            //【4.3.3】减少 mActiveIdleOpCount 计数，尝试提前退出时间窗！
            decActiveIdleOps();

            EventLogTags.writeDeviceIdleOffComplete();
        } break;
```
处理 MSG_REPORT_IDLE_OFF 消息比较特殊，这里我们是有序的发送广播，同时在发送 mIdleIntent 或者 mLightIdleIntent 广播的时候，我们额外传入了 mIdleStartedDoneReceiver，是最后有序队列中的最后一个接收者：

### 4.3.1 IdleStartedDoneReceiver

```java
    private final BroadcastReceiver mIdleStartedDoneReceiver = new BroadcastReceiver() {
        @Override public void onReceive(Context context, Intent intent) {
            if (PowerManager.ACTION_DEVICE_IDLE_MODE_CHANGED.equals(intent.getAction())) {
                //【4.7】如果是 deep idle 模式发生了变化，延迟 mConstants.MIN_DEEP_MAINTENANCE_TIME(30s)
                // 发送 MSG_FINISH_IDLE_OP 消息！
                mHandler.sendEmptyMessageDelayed(MSG_FINISH_IDLE_OP,
                        mConstants.MIN_DEEP_MAINTENANCE_TIME);
            } else {
                //【4.7】如果是 light idle 模式发生了变化，延迟 mConstants.MIN_LIGHT_MAINTENANCE_TIME(5s) 
                // 发送 MSG_FINISH_IDLE_OP 消息！
                mHandler.sendEmptyMessageDelayed(MSG_FINISH_IDLE_OP,
                        mConstants.MIN_LIGHT_MAINTENANCE_TIME);
            }
        }
    };
```
对于暂时退出 idle 状态来说，mIdleIntent 或者 mLightIdleIntent 广播的时候，除了要发送给其他服务，DeviceIdleController 自身作为最后一个接收者，也会接受该广播！

当 DeviceIdleController 接收到广播后，会判断下是退出 deep idle 还是 light idle 状态！

- 如果退出 deep idle，延迟延迟 mConstants.MIN_DEEP_MAINTENANCE_TIME (30s)，发送 MSG_FINISH_IDLE_OP 消息给 MyHandler；
- 如果退出 light idle，延迟延迟  mConstants.MIN_LIGHT_MAINTENANCE_TIME (5s)，发送 MSG_FINISH_IDLE_OP 消息给 MyHandler；

MyHandler 在接收到 MSG_FINISH_IDLE_OP 消息后，会减少 mActiveIdleOpCount 计数，同时会尝试提前退出时间窗！

### 4.3.2 incActiveIdleOps
```java
    void incActiveIdleOps() {
        synchronized (this) {
            mActiveIdleOpCount++;
        }
    }
```
incActiveIdleOps 主要是将 mActiveIdleOpCount 加 1，表示此时我们进入时间窗，执行相关的操作；

### 4.3.3 decActiveIdleOps
```java
    void decActiveIdleOps() {
        synchronized (this) {
            mActiveIdleOpCount--;
            if (mActiveIdleOpCount <= 0) {
                //【4.3.2.1】尝试提前退出时间窗！
                exitMaintenanceEarlyIfNeededLocked();
                mActiveIdleWakeLock.release();
            }
        }
    }
```
decActiveIdleOps 方法会先将 mActiveIdleOpCount 减 1；如果此时 mActiveIdleOpCount <= 0，那么会尝试提前退出时间窗：

#### 4.3.2.1 exitMaintenanceEarlyIfNeededLocked - 提前退出时间窗

我们来看尝试提前退出时间窗的逻辑！
```java
    void exitMaintenanceEarlyIfNeededLocked() {
        //【1】首先，对状态做一个判断！
        if (mState == STATE_IDLE_MAINTENANCE || mLightState == LIGHT_STATE_IDLE_MAINTENANCE
                || mLightState == LIGHT_STATE_PRE_IDLE) {
            //【4.3.2.2】判断时间窗内是否还有任务执行，如果返回 true，表明可以提前退出时间窗！
            if (isOpsInactiveLocked()) {
                final long now = SystemClock.elapsedRealtime();
                if (DEBUG) {
                    StringBuilder sb = new StringBuilder();
                    sb.append("Exit: start=");
                    TimeUtils.formatDuration(mMaintenanceStartTime, sb);
                    sb.append(" now=");
                    TimeUtils.formatDuration(now, sb);
                    Slog.d(TAG, sb.toString());
                }

                //【2】如果是 deep idle 模式，且状态为 STATE_IDLE_MAINTENANCE，调用 stepIdleStateLocked
                // 提前退出时间窗，如果是 light idle 模式，调用 stepLightIdleStateLocked 退出时间窗！
                if (mState == STATE_IDLE_MAINTENANCE) {
                    stepIdleStateLocked("s:early"); //【3.1】
                } else if (mLightState == LIGHT_STATE_PRE_IDLE) {
                    stepLightIdleStateLocked("s:predone");//【3.2】
                } else {
                    stepLightIdleStateLocked("s:early");
                }
            }
        }
    }
```
提前退出时间窗的意义在于，很可能在很短的时间内，时间窗内的工作已经完成了，我们就没有必要等到时间窗结束了！

首先对状态做了判断：
- deep idle 为 STATE_IDLE_MAINTENANCE；light idle 为 LIGHT_STATE_IDLE_MAINTENANCE/LIGHT_STATE_PRE_IDLE


然后调用 isOpsInactiveLocked 方法，判断此时是否没有 active op，avtive job 和 active alarm！


#### 4.3.2.2 isOpsInactiveLocked
```java
    boolean isOpsInactiveLocked() {
        return mActiveIdleOpCount <= 0 && !mJobsActive && !mAlarmsActive;
    }
```
该方法用于判断时间窗内是否还有任务执行：

如果 mActiveIdleOpCount <= 0，同时没有 JobService，Alarms 处于活跃状态，那么 isOpsInactiveLocked 就返回 true！

## 4.4 消息：MSG_REPORT_ACTIVE

当 idle 状态的条件不满足后，会退出 doze 模式，恢复 STATE_ACTIVE 状态，发送 MSG_REPORT_ACTIVE 给 MyHandler！

```java
                case MSG_REPORT_ACTIVE: {
                    String activeReason = (String)msg.obj;
                    int activeUid = msg.arg1;
                    EventLogTags.writeDeviceIdleOffStart(
                            activeReason != null ? activeReason : "unknown");
                    //【1】通知 PowerManager 退出 deep idle 和 light idle 状态！
                    final boolean deepChanged = mLocalPowerManager.setDeviceIdleMode(false);
                    final boolean lightChanged = mLocalPowerManager.setLightDeviceIdleMode(false);
                    try {
                        //【2】通知 NetworkPolicy 和 BatteryStats 退出 idle 状态！
                        mNetworkPolicyManager.setDeviceIdleMode(false);
                        mBatteryStats.noteDeviceIdleMode(BatteryStats.DEVICE_IDLE_MODE_OFF,
                                activeReason, activeUid);
                    } catch (RemoteException e) {
                    }

                    //【3】如果是退出 deep idle，发送广播通知其他服务！ 
                    if (deepChanged) {
                        getContext().sendBroadcastAsUser(mIdleIntent, UserHandle.ALL);
                    }
                    //【4】如果是退出 light idle，发送广播通知其他服务！ 
                    if (lightChanged) {
                        getContext().sendBroadcastAsUser(mLightIdleIntent, UserHandle.ALL);
                    }
                    EventLogTags.writeDeviceIdleOffComplete();
                } break;
```
方法很简单，不多说了！

## 4.5 消息：MSG_TEMP_APP_WHITELIST_TIMEOUT

我们可以将用户应用程序动态添加到 doze 模式的用户白名单中，使得应用可以在 doze 模式下运行，这种方式一旦添加，就一直生效，因为他会持久化到本地文件；

同时，我们还可以动态添加应用到临时缓存白名单中，使得该应用能够临时获得 doze 模式下的 network 和 wakelocks 访问，在添加时要指定临时的访问截至时间。时间过后，名单失效！！

```java
    // 临时缓存白名单，在该名单中的 uid 被临时标记为在 doze 模式下允许访问网络并获取唤醒锁；
    // MutableLong 用于指定临时时间！
    private final SparseArray<Pair<MutableLong, String>> mTempWhitelistAppIdEndTimes
            = new SparseArray<>();

     // 当 mTempWhitelistAppIdEndTimes 发生变化后，用于通知 NetworkPolicyManagerService 执行任务！
    Runnable mNetworkPolicyTempWhitelistCallback;

    // 用于记录缓存 mTempWhitelistAppIdEndTimes 中的所有应用的 uid；
    private int[] mTempWhitelistAppIdArray = new int[0];
```
那么 MSG_TEMP_APP_WHITELIST_TIMEOUT 消息的所用是什么呢？就是不断的检查 mTempWhitelistAppIdEndTimes，mTempWhitelistAppIdArray，如果有过期名单，移除！

```java
                case MSG_TEMP_APP_WHITELIST_TIMEOUT: {
                    int uid = msg.arg1;
                    //【4.5.1】调用 checkTempAppWhitelistTimeout 方法！
                    checkTempAppWhitelistTimeout(uid);
                } break;
```
核心方法在 checkTempAppWhitelistTimeout 中！

### 4.5.1 checkTempAppWhitelistTimeout

检查缓存白名单中是否有应用过期！
```java
    void checkTempAppWhitelistTimeout(int uid) {
        final long timeNow = SystemClock.elapsedRealtime();
        if (DEBUG) {
            Slog.d(TAG, "checkTempAppWhitelistTimeout: uid=" + uid + ", timeNow=" + timeNow);
        }
        synchronized (this) {
            //【1】判断临时缓存白名单中是否有该 uid！
            Pair<MutableLong, String> entry = mTempWhitelistAppIdEndTimes.get(uid);
            if (entry == null) {
                return;
            }

            if (timeNow >= entry.first.value) {
                //【2】当前时间 timeNow 超过了白名单中应用的有效期时间；
                // 将该应用从白名单中删除！
                mTempWhitelistAppIdEndTimes.delete(uid);
                if (DEBUG) {
                    Slog.d(TAG, "Removing UID " + uid + " from temp whitelist");
                }
                //【4.5.2.1】更新 PowerManager 中缓存白名单！
                updateTempWhitelistAppIdsLocked();
                if (mNetworkPolicyTempWhitelistCallback != null) {
                    mHandler.post(mNetworkPolicyTempWhitelistCallback); // 更新 NetworkPolicy 内部名单！
                }
                //【4.5.2.2】通知其他服务，缓存白名单发生了变化！
                reportTempWhitelistChangedLocked();
                try {
                    mBatteryStats.noteEvent(BatteryStats.HistoryItem.EVENT_TEMP_WHITELIST_FINISH,
                            entry.second, uid);
                } catch (RemoteException e) {
                }
            } else {
                //【3】如果当前时间 timeNow 还没有超过白名单中应用的有效期时间；
                // 延迟 entry.first.value - timeNow 时间间隔再次发送 MSG_TEMP_APP_WHITELIST_TIMEOUT 消息！
                if (DEBUG) {
                    Slog.d(TAG, "Time to remove UID " + uid + ": " + entry.first.value);
                }
                postTempActiveTimeoutMessage(uid, entry.first.value - timeNow);
            }
        }
    }
```

#### 4.5.2.1 updateTempWhitelistAppIdsLocked

更新缓存白名单 mTempWhitelistAppIdEndTimes！
```java
    private void updateTempWhitelistAppIdsLocked() {
        //【1】获得最新的 mTempWhitelistAppIdEndTimes 记录，更新 mTempWhitelistAppIdArray！
        final int size = mTempWhitelistAppIdEndTimes.size();
        if (mTempWhitelistAppIdArray.length != size) {
            mTempWhitelistAppIdArray = new int[size];
        }
        for (int i = 0; i < size; i++) {
            mTempWhitelistAppIdArray[i] = mTempWhitelistAppIdEndTimes.keyAt(i);
        }
        //【2】将最新的 mTempWhitelistAppIdArray 添加到 PowerManager 中！
        if (mLocalPowerManager != null) {
            if (DEBUG) {
                Slog.d(TAG, "Setting wakelock temp whitelist to "
                        + Arrays.toString(mTempWhitelistAppIdArray));
            }
            mLocalPowerManager.setDeviceIdleTempWhitelist(mTempWhitelistAppIdArray);
        }
    }
```
mTempWhitelistAppIdArray 数组中用来保存缓存白名单的应用的 uid，和 mTempWhitelistAppIdEndTimes 是不同的描述角度！

#### 4.5.2.2 reportTempWhitelistChangedLocked

```java
    private void reportTempWhitelistChangedLocked() {
        Intent intent = new Intent(PowerManager.ACTION_POWER_SAVE_TEMP_WHITELIST_CHANGED);
        intent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY);
        getContext().sendBroadcastAsUser(intent, UserHandle.SYSTEM);
    }
```
其实就是发送了 PowerManager.ACTION_POWER_SAVE_TEMP_WHITELIST_CHANGED 广播！

## 4.6 消息：MSG_REPORT_MAINTENANCE_ACTIVITY

当设备退出了 idle 状态后，系统中的 JobService 会开始执行，JobSchedulerService 会调用 DeviceIdleC.setJobsActive(true) 方法，将 JobService 的状态保存到 DeviceIdleController 中去！

### 4.6.1 DeviceIdleC.setJobsActive

setJobsActive 用于设置 JobService 的活跃状态：

```java
    void setJobsActive(boolean active) {
        synchronized (this) {
            //【1】将 mJobsActive 为 active 的值！
            mJobsActive = active;
            //【4.6.1.1】JobService 的活跃状态发生了变化！
            reportMaintenanceActivityIfNeededLocked();

            //【2】如果 active 为 false，说明此时没有 JobService 活跃了，
            // 那就尝试提前退出时间窗；
            if (!active) {
                exitMaintenanceEarlyIfNeededLocked();
            }
        }
    }
```

#### 4.6.1.1 DeviceIdleC.reportMaintenanceActivityIfNeededLocked

```java
    void reportMaintenanceActivityIfNeededLocked() {
        boolean active = mJobsActive;
        if (active == mReportedMaintenanceActivity) {
            return;
        }
        mReportedMaintenanceActivity = active;

        //【2】发送 MSG_REPORT_MAINTENANCE_ACTIVITY 给 MyHandler！
        Message msg = mHandler.obtainMessage(MSG_REPORT_MAINTENANCE_ACTIVITY,
                mReportedMaintenanceActivity ? 1 : 0, 0);
        mHandler.sendMessage(msg);
    }
```

我们来看下 MyHandler 对于 MSG_REPORT_MAINTENANCE_ACTIVITY 消息的处理：
```java
                case MSG_REPORT_MAINTENANCE_ACTIVITY: {
                    boolean active = (msg.arg1 == 1);
                    final int size = mMaintenanceActivityListeners.beginBroadcast();
                    try {
                        for (int i = 0; i < size; i++) {
                            try {
                                mMaintenanceActivityListeners.getBroadcastItem(i)
                                        .onMaintenanceActivityChanged(active);
                            } catch (RemoteException ignored) {
                            }
                        }
                    } finally {
                        mMaintenanceActivityListeners.finishBroadcast();
                    }
                } break;
```

## 4.7 消息：MSG_FINISH_IDLE_OP

对于 MSG_FINISH_IDLE_OP 消息的处理很简单，就是调用 decActiveIdleOps 减少 mActiveIdleOpCount 引用计数！

```java
                case MSG_FINISH_IDLE_OP: {
                    //【4.7.1】减少 
                    decActiveIdleOps();
                } break;
```

### 4.7.1 decActiveIdleOps

我们来看看 decActiveIdleOps 方法：
```java
    void decActiveIdleOps() {
        synchronized (this) {
            //【1】减少 mActiveIdleOpCount 计数；
            mActiveIdleOpCount--;
            if (mActiveIdleOpCount <= 0) {
                //【4.3.2.1】尝试提前退出时间窗！
                exitMaintenanceEarlyIfNeededLocked();
                mActiveIdleWakeLock.release();
            }
        }
    }
```
这里的逻辑我们之前见过！

进入时间窗的时候 mActiveIdleOpCount 会被加 1，退出时间窗的时候，mActiveIdleOpCount 会被减 1；

# 5 重置操作

对于重置操作，我们放到这里来一起分析下；

然后，重置 deep idle 逻辑相关的变量，取消 Alarm 和监控器！！

## 5.1 resetIdleManagementLocked
```java
    void resetIdleManagementLocked() {
        mNextIdlePendingDelay = 0;
        mNextIdleDelay = 0;
        mNextLightIdleDelay = 0;
        //【1.1.1.1.1】取消 mDeepAlarmListener
        cancelAlarmLocked();
        //【1.1.1.1.2】取消 mSensingTimeoutAlarmListener
        cancelSensingTimeoutAlarmLocked();
        //【1.1.1.1.3】取消 mGenericLocationListener，mGpsLocationListener
        cancelLocatingLocked();
        //【1.1.1.1.4】取消 mMotionListener
        stopMonitoringMotionLocked();
        mAnyMotionDetector.stop();
    }
```

### 5.1.1 cancelAlarmLocked

```java
    void cancelAlarmLocked() {
        if (mNextAlarmTime != 0) {
            mNextAlarmTime = 0;
            mAlarmManager.cancel(mDeepAlarmListener);
        }
    }
```
取消 mDeepAlarmListener！

该 mDeepAlarmListener 触发后会执行 stepIdleStateLocked 方法！

### 5.1.2 cancelSensingTimeoutAlarmLocked

```java
    void cancelSensingTimeoutAlarmLocked() {
        if (mNextSensingTimeoutAlarmTime != 0) {
            mNextSensingTimeoutAlarmTime = 0;
            mAlarmManager.cancel(mSensingTimeoutAlarmListener);
        }
    }
```
取消 mSensingTimeoutAlarmListener！

mSensingTimeoutAlarmListener 触发后，如果 mState 为 STATE_SENSING，这时会执行 1.1：becomeInactiveIfAppropriateLocked 方法！

### 5.1.3 cancelLocatingLocked

```java
    void cancelLocatingLocked() {
        if (mLocating) {
            mLocationManager.removeUpdates(mGenericLocationListener);
            mLocationManager.removeUpdates(mGpsLocationListener);
            mLocating = false;
        }
    }
```
取消 mGenericLocationListener，mGpsLocationListener！

- mGenericLocationListener 触发后，会执行 receivedGenericLocationLocked 方法；

- mGpsLocationListener 触发后，会执行 receivedGpsLocationLocked 方法；

### 5.1.4 stopMonitoringMotionLocked
```java
    void stopMonitoringMotionLocked() {
        if (DEBUG) Slog.d(TAG, "stopMonitoringMotionLocked()");
        if (mMotionSensor != null && mMotionListener.active) {
            mMotionListener.unregisterLocked();
        }
    }
```
取消 mMotionListener！


## 5.2 resetLightIdleManagementLocked

```java
    void resetLightIdleManagementLocked() {
        //【1.1.2.1.1】取消之前设置的 light idle alarm！
        cancelLightAlarmLocked();
    }
```
light idle 模式的重置相关变量过程很简单，只是取消 mLightAlarmListener！


### 5.2.1 cancelLightAlarmLocked
```java
    void cancelLightAlarmLocked() {
        if (mNextLightAlarmTime != 0) {
            mNextLightAlarmTime = 0;
            //【1】取消 mLightAlarmListener！
            mAlarmManager.cancel(mLightAlarmListener);
        }
    }
```
取消 mLightAlarmListener！

# 6 提前退出时间窗


# 7 总结

我们来看看 doze 模式下的监听器有哪些，下图是 doze 模式的所有监听器，监听不同的条件是否满足：

## 7.1 状态监听器

下面的几张图是监听手机状态的监听器的逻辑：

## 7.2 deep idle

我们来总结下 deep idle 状态的流程：

![deep idle 流程分析](https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/23/deep idle 流程分析.png)

## 7.3 light idle

我们来总结下 light idle 状态的流程：

![light idle 流程.png-281.7kB](https://coolqifiles.oss-cn-hangzhou.aliyuncs.com/uPic2/2022/10/24/23/light idle 流程.png)
