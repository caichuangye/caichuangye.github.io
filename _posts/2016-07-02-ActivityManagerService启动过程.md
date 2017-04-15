---
title:  ActivityManagerService启动过程
date: 2016-07-02 21:25:53
tags:
- AMS
- Android
categories: Android
---


AMS在SystemServer中通过SystemServiceManager的startService方法启动，AMS的启动时机较早，属于系统引导服务。

#### 1. new ActivityManagerService
```java
public ActivityManagerService(Context systemContext) {
        mContext = systemContext;
        mFactoryTest = FactoryTest.getMode();
        //初始化ActivityThread
        mSystemThread = ActivityThread.currentActivityThread();

        Slog.i(TAG, "Memory class: " + ActivityManager.staticGetMemoryClass());

		//创建name为ActivityManagerService的HandlerThread
        mHandlerThread = new ServiceThread(TAG,
                android.os.Process.THREAD_PRIORITY_FOREGROUND, false /*allowIo*/);
        mHandlerThread.start();
        mHandler = new MainHandler(mHandlerThread.getLooper());
        
        mUiHandler = new UiHandler();

		//前台广播队列，超时时间为10s
        mFgBroadcastQueue = new BroadcastQueue(this, mHandler,
                "foreground", BROADCAST_FG_TIMEOUT, false);
        //后台广播队列，超时时间为60s
        mBgBroadcastQueue = new BroadcastQueue(this, mHandler,
                "background", BROADCAST_BG_TIMEOUT, true);
        mBroadcastQueues[0] = mFgBroadcastQueue;
        mBroadcastQueues[1] = mBgBroadcastQueue;

        mServices = new ActiveServices(this);
        mProviderMap = new ProviderMap(this);

        // TODO: Move creation of battery stats service outside of activity manager service.
        //创建/data/system目录
        File dataDir = Environment.getDataDirectory();
        File systemDir = new File(dataDir, "system");
        systemDir.mkdirs();
        //创建BatteryStatsService
        mBatteryStatsService = new BatteryStatsService(systemDir, mHandler);
        mBatteryStatsService.getActiveStatistics().readLocked();
        mBatteryStatsService.scheduleWriteToDisk();
        mOnBattery = DEBUG_POWER ? true
                : mBatteryStatsService.getActiveStatistics().getIsOnBattery();
        mBatteryStatsService.getActiveStatistics().setCallback(this);

        mProcessStats = new ProcessStatsService(this, new File(systemDir, "procstats"));

        mAppOpsService = new AppOpsService(new File(systemDir, "appops.xml"), mHandler);

        mGrantFile = new AtomicFile(new File(systemDir, "urigrants.xml"));

        // User 0 is the first and only user that runs at boot.
        mStartedUsers.put(UserHandle.USER_OWNER, new UserState(UserHandle.OWNER, true));
        mUserLru.add(UserHandle.USER_OWNER);
        updateStartedUserArrayLocked();

        GL_ES_VERSION = SystemProperties.getInt("ro.opengles.version",
            ConfigurationInfo.GL_ES_VERSION_UNDEFINED);

        mTrackingAssociations = "1".equals(SystemProperties.get("debug.track-associations"));

        mConfiguration.setToDefaults();
        mConfiguration.setLocale(Locale.getDefault());

        mConfigurationSeq = mConfiguration.seq = 1;
        mProcessCpuTracker.init();

        mCompatModePackages = new CompatModePackages(this, systemDir, mHandler);
        mIntentFirewall = new IntentFirewall(new IntentFirewallInterface(), mHandler);
        mRecentTasks = new RecentTasks(this);
        mStackSupervisor = new ActivityStackSupervisor(this, mRecentTasks);
        mTaskPersister = new TaskPersister(systemDir, mStackSupervisor, mRecentTasks);

		//创建name为“CpuTracker”的线程
        mProcessCpuThread = new Thread("CpuTracker") {
            @Override
            public void run() {
                while (true) {
                    try {
                        try {
                            synchronized(this) {
                                final long now = SystemClock.uptimeMillis();
                                long nextCpuDelay = (mLastCpuTime.get()+MONITOR_CPU_MAX_TIME)-now;
                                long nextWriteDelay = (mLastWriteTime+BATTERY_STATS_TIME)-now;
                                //Slog.i(TAG, "Cpu delay=" + nextCpuDelay
                                //        + ", write delay=" + nextWriteDelay);
                                if (nextWriteDelay < nextCpuDelay) {
                                    nextCpuDelay = nextWriteDelay;
                                }
                                if (nextCpuDelay > 0) {
                                    mProcessCpuMutexFree.set(true);
                                    this.wait(nextCpuDelay);
                                }
                            }
                        } catch (InterruptedException e) {
                        }
                        updateCpuStatsNow();
                    } catch (Exception e) {
                        Slog.e(TAG, "Unexpected exception collecting process stats", e);
                    }
                }
            }
        };

        Watchdog.getInstance().addMonitor(this);
        Watchdog.getInstance().addThread(mHandler);
    }
```
该过程共创建了3个线程：分别为”ActivityManagerService”，”android.ui”，”CpuTracker”。
* ActivityManagerService
>用来处理一些系统事件，比如时区更新、Service超时、进程启动超时等 

* android.ui
> 主要用于处理系统的一些需要界面的消息，比如应用的ANR、违反严格模式等

* CpuTracker
> `未知`

构造函数的最后，将AMS本身和mHandler加入到WatchDog的监控。

#### 2. start
```
 private void start() {
	    //移除所有进程组
        Process.removeAllProcessGroups();
        //启动Cpu统计
        mProcessCpuThread.start();
        //启动电池统计服务
        mBatteryStatsService.publish(mContext);
        mAppOpsService.publish(mContext);
        Slog.d("AppOps", "AppOpsService published");
        LocalServices.addService(ActivityManagerInternal.class, new LocalService());
    }
```

#### 3. setSystemServiceManager
```java
public void setSystemServiceManager(SystemServiceManager mgr) {
        mSystemServiceManager = mgr;
}
```
SystemServiceManager有3个作用：
* 启动系统服务
* 通知系统服务当前所处的启动阶段
* 通知系统服务当前的用户信息

在AMS中，mSystemServiceManager 用来实现第二和第三个功能：

`通知系统服务当前的用户信息`
> AMS收到启动用户或切换、停止用户时，mSystemServiceManager 要调用其对应的startUser或switchUser，通知系统服务当前的系统用户有变化。

`通知系统服务当前所处的启动阶段`
> AMS中mSystemServiceManager 主要用来通知当前启动阶段和通知当前用户信息。在AMS的finishBooting方法中，通知系统服务当前的启动阶段(最后一个阶段)：
> `mSystemServiceManager.startBootPhase(SystemService.PHASE_BOOT_COMPLETED);`



---
#### 4. setInstaller
```java
    public void setInstaller(Installer installer) {
        mInstaller = installer;
    }
```
mInstaller在AMS的finishBooting中用来通知启动完成。

---
#### 5. initPowerManagement
```java
public void initPowerManagement() {
        mStackSupervisor.initPowerManagement();
        mBatteryStatsService.initPowerManagement();
        mLocalPowerManager = LocalServices.getService(PowerManagerInternal.class);
        PowerManager pm = (PowerManager)mContext.getSystemService(Context.POWER_SERVICE);
        mVoiceWakeLock = pm.newWakeLock(PowerManager.PARTIAL_WAKE_LOCK, "*voice*");
        mVoiceWakeLock.setReferenceCounted(false);
    }

```
---
#### 6. setSystemProcess
```java
 public void setSystemProcess() {
        try {
            ServiceManager.addService(Context.ACTIVITY_SERVICE, this, true);
            ServiceManager.addService(ProcessStats.SERVICE_NAME, mProcessStats);
            ServiceManager.addService("meminfo", new MemBinder(this));
            ServiceManager.addService("gfxinfo", new GraphicsBinder(this));
            ServiceManager.addService("dbinfo", new DbBinder(this));
            if (MONITOR_CPU_USAGE) {
                ServiceManager.addService("cpuinfo", new CpuBinder(this));
            }
            ServiceManager.addService("permission", new PermissionController(this));
            ServiceManager.addService("processinfo", new ProcessInfoService(this));

            ApplicationInfo info = mContext.getPackageManager().getApplicationInfo(
                    "android", STOCK_PM_FLAGS);
            mSystemThread.installSystemApplicationInfo(info, getClass().getClassLoader());

            synchronized (this) {
                ProcessRecord app = newProcessRecordLocked(info, info.processName, false, 0);
                app.persistent = true;
                app.pid = MY_PID;
                app.maxAdj = ProcessList.SYSTEM_ADJ;
                app.makeActive(mSystemThread.getApplicationThread(), mProcessStats);
                synchronized (mPidsSelfLocked) {
                    mPidsSelfLocked.put(app.pid, app);
                }
                updateLruProcessLocked(app, false, null);
                updateOomAdjLocked();
            }
        } catch (PackageManager.NameNotFoundException e) {
            throw new RuntimeException(
                    "Unable to find android system package", e);
        }
    }
```
该过程创建了以下服务：

| 服务名      |    类名 | 功能  |
| :-------- |:--------| :--: |
| activity| ActivityManagerService |  AMS   |
| procstats|ProcessStatsService |  进程统计 |
| meminfo|MemBinder | 内存使用信息  |
| gfxinfo| GraphicsBinder |  图形信息   |
| dbinfo|   DbBinder |  数据库信息  |
| cpuinfo|   CpuBinder | CPU  |
| permission| PermissionController |  权限  |
| processinfo|   ProcessInfoService |  进程统计  |

以上这些服务可以通过adb命令查看：
```bash
adb shell dumpsys 服务名 [包名] > d:\log\record.txt
```
其中，服务名必须输入，包名为可选项。有包名时，输出的是指定包名的信息，不指定包名则输出所有应用的信息。
如果直接观察不方便，可将信息重定向到指定的文件中。


---


#### 7. setUsageStatsManager
```java
public void setUsageStatsManager(UsageStatsManagerInternal usageStatsManager) {
        mUsageStatsService = usageStatsManager;
    }
```

---
#### 8. installSystemProviders
```java
 public final void installSystemProviders() {
        List<ProviderInfo> providers;
        synchronized (this) {
            ProcessRecord app = mProcessNames.get("system", Process.SYSTEM_UID);
            providers = generateApplicationProvidersLocked(app);
            if (providers != null) {
                for (int i=providers.size()-1; i>=0; i--) {
                    ProviderInfo pi = (ProviderInfo)providers.get(i);
                    if ((pi.applicationInfo.flags&ApplicationInfo.FLAG_SYSTEM) == 0) {
                        Slog.w(TAG, "Not installing system proc provider " + pi.name
                                + ": not system .apk");
                        providers.remove(i);
                    }
                }
            }
        }
        if (providers != null) {
            mSystemThread.installSystemProviders(providers);
        }

        mCoreSettingsObserver = new CoreSettingsObserver(this);

        //mUsageStatsService.monitorPackages();
    }
```
* 在installSystemProviders中，为SystemServer的ActivityThread安装系统的Provider
* 创建CoreSettingsObserver对象，监控系统核心设置的变化。当有变化时，调用AMS的onCoreSettingsChange通知系统中的各个应用，最终调用了每个app对应的ActivityThread中的requestRelaunchActivity方法。


---
#### 9. setWindowManager
```java
 public void setWindowManager(WindowManagerService wm) {
        mWindowManager = wm;
        mStackSupervisor.setWindowManager(wm);
    }
```
---
#### 10. systemReady

systemReady代码较长，主要做了以下事情：
* 调用部分系统服务的systemReady或systemRunning方法
* 调用 mSystemServiceManager.startBootPhase的方法通知当前的启动阶段。在systemReady中，涉及的启动阶段有PHASE_ACTIVITY_MANAGER_READY、PHASE_THIRD_PARTY_APPS_CAN_START 和PHASE_BOOT_COMPLETED 
* 启动SystemUi
* 启动home应用

SystemUI通过如下方法实现：
```java
static final void startSystemUi(Context context) {
        Intent intent = new Intent();
        intent.setComponent(new ComponentName("com.android.systemui",
                    "com.android.systemui.SystemUIService"));
        //Slog.d(TAG, "Starting service: " + intent);
        context.startServiceAsUser(intent, UserHandle.OWNER);
    }
```

在systemReady中，调用startHomeActivityLocked启动home应用：
```java
boolean startHomeActivityLocked(int userId, String reason) {
        if (mFactoryTest == FactoryTest.FACTORY_TEST_LOW_LEVEL
                && mTopAction == null) {
            // We are running in factory test mode, but unable to find
            // the factory test app, so just sit around displaying the
            // error message and don't try to start anything.
            return false;
        }
        Intent intent = getHomeIntent();
        ActivityInfo aInfo =
            resolveActivityInfo(intent, STOCK_PM_FLAGS, userId);
        if (aInfo != null) {
            intent.setComponent(new ComponentName(
                    aInfo.applicationInfo.packageName, aInfo.name));
            // Don't do this if the home app is currently being
            // instrumented.
            aInfo = new ActivityInfo(aInfo);
            aInfo.applicationInfo = getAppInfoForUser(aInfo.applicationInfo, userId);
            ProcessRecord app = getProcessRecordLocked(aInfo.processName,
                    aInfo.applicationInfo.uid, true);
            if (app == null || app.instrumentationClass == null) {
                intent.setFlags(intent.getFlags() | Intent.FLAG_ACTIVITY_NEW_TASK);
                mStackSupervisor.startHomeActivity(intent, aInfo, reason);
            }
        }

        return true;
    }
```
在startHomeActivityLocked中通过getHomeIntent获取home应用对应的Intent：

```java
Intent getHomeIntent() {
        Intent intent = new Intent(mTopAction, mTopData != null ? Uri.parse(mTopData) : null);
        intent.setComponent(mTopComponent);
        if (mFactoryTest != FactoryTest.FACTORY_TEST_LOW_LEVEL) {
            intent.addCategory(Intent.CATEGORY_HOME);
        }
        return intent;
    }
```
`mTopAction为Intent.ACTION_MAIN，可见home应用对应 intent Action为Intent.ACTION_MAIN，Category为Intent.CATEGORY_HOME。当home启动后，Android系统启动完毕！`

`AMS中有方法finishBooting，该方法中通知了系统服务启动的最后一个阶段：PHASE_BOOT_COMPLETED ，并且发出了一个著名的广播：ACTION_BOOT_COMPLETED。该方法调用时机暂时未知。`