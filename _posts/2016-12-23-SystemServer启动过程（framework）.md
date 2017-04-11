---
title: SystemServer启动过程（framework）
date: 2016-12-23 22:43:13
tags:
- SystemServer
- Android
categories: Android
---

#### 一. 启动过程
SystemServer通过main方法启动：
```java
public static void main(String[] args) {
    new SystemServer().run();
}
```
main方法中直接调用了run函数。整个run函数的过程中，做个5件特别重要的事（当然还有一些其他的action，在此不讨论）：
##### 1.1  创建SystemServer对应的Context
```java
 private void createSystemContext() {
        ActivityThread activityThread = ActivityThread.systemMain();
        mSystemContext = activityThread.getSystemContext();
        mSystemContext.setTheme(android.R.style.Theme_DeviceDefault_Light_DarkActionBar);
    }
```
从代码可知，在createSystemContext中做了三件事：创建ActivityThread 、创建mSystemContext 、设置SystemContext的主题。
###### 1.1.1创建ActivityThread
ActivityThread的SystemMain方法如下：
```java
 public static ActivityThread systemMain() {
        ....................
        ActivityThread thread = new ActivityThread();
        thread.attach(true);
        return thread;
    }
```
先new出一个ActivityThread 对象，然后调用attach方法，注意传入的参数为true，表明这是一个系统的ActivityThread。
```java
 private void attach(boolean system) {
 ...............
      try {
            mInstrumentation = new Instrumentation();
            ContextImpl context = ContextImpl.createAppContext(
                        this, getSystemContext().mPackageInfo);
            mInitialApplication = context.mPackageInfo.makeApplication(true, null);
            mInitialApplication.onCreate();
       } catch (Exception e) {
            ..........................
      }    
    }
.................
```
调用ContextImpl.createAppContext时，第二个参数为getSystemContext().mPackageInfo，ContextImpl有3中创建方式，其中getSystemContext对应的是ContextImpl中的createSystemContext方法，代码如下：
```java
 static ContextImpl createSystemContext(ActivityThread mainThread) {
        LoadedApk packageInfo = new LoadedApk(mainThread);
        ContextImpl context = new ContextImpl(null, mainThread,
                packageInfo, null, null, false, null, null, Display.INVALID_DISPLAY);
        context.mResources.updateConfiguration(context.mResourcesManager.getConfiguration(),
                context.mResourcesManager.getDisplayMetricsLocked());
        return context;
    }
```
创建ContextImpl对象时居然传入了LoadedApk 作为参数！！！一般来说，我们认为LoadedApk代表一个已经加载的apk，在这里我们看到，对应SystemServer来说，也可以认为是一个app，一个特殊的apk。SystemServer是个系统级的进程，apk也是存在于进程之中，区别在于apk一般来说都有界面而已。
再回到ActivityThread的attach方法中，接下来做的事情为创建Application对象。
mInitialApplication = context.mPackageInfo.makeApplication(true, null);
mInitialApplication.onCreate();
mInitialApplication类型为Application，makeApplication最终调用的是Instrumentation的newApplication方法：
```java
  static public Application newApplication(Class<?> clazz, Context context)
            throws InstantiationException, IllegalAccessException, 
            ClassNotFoundException {
        Application app = (Application)clazz.newInstance();
        app.attach(context);
        return app;
    }
```
`注意，此时传入的Context类型为App类型Context，其实很好理解，因为一个Application对应一个app。
Applicaiton的attach方法就是把Context穿进去，设置其基类ContextWrapper中的mBase。`至此，ActivityThread中的Application创建完毕，ActivityThread对象也创建完毕。

###### 1.1.2创建mSystemContext

创建mSystemContext对象的方法为：
```java
public ContextImpl getSystemContext() {
        synchronized (this) {
            if (mSystemContext == null) {
                mSystemContext = ContextImpl.createSystemContext(this);
            }
            return mSystemContext;
        }
    }
```
ContextImpl中的createSystemContext方法在2.1.1小节中已经讲过，在此略过。
###### 1.1.3设置SystemContext的主题
给系统的Context设置了一个Theme_DeviceDefault_Light_DarkActionBar的主题。

`至此，SystemServer中的Context上下文创建完毕，通过分析创建过程，可以一个ActivityThread不仅可以在Apk中用来执行AMS发来的请求，在SystemServer也可以充当类似的角色，同理，LoadedApk对象也是如此。也就是说，我们可以把SystemServer这个进程理解为一个特殊的Apk，Apk和SystemServer的某些信息都可以用ActivityThread和LoadedApk来表示。`
##### 1.2 启动引导服务：startBootstrapServices（7个）
* ActivityManagerService
* PowerManagerService
*  LightsService
*  DisplayManagerService 
*  PackageManagerService 
*  UserManagerService
*  sensor
PHASE_WAIT_FOR_DEFAULT_DISPLA阶段发生在该过程中。
##### 1.3. 启动核心服务：startCoreServices（3个）
* BatteryService
* UsageStatsService
* WebViewUpdateService
##### 1.4. 启动其他的系统服务：startOtherServices
该过程中创建的系统服务较多，在此不分析。
##### 1.5. 系统SystemServer进程的Looper循环
#### 二. 系统服务的基类和创建方式
在SystemServer中要启动众多的服务，大部分的Service都是通过SystemServiceManager来启动和监控启动过程的。SystemServiceManager内部有变量：
```java
 private final ArrayList<SystemService> mServices = new ArrayList<SystemService>();
```
对于每一个要通过SystemServiceManager启动的Serice，都要加入到mServices 中。`注意：凡是通过该中方式启动的Service，都必需有onBootPhase方法，下文会解释原因。`

SystemServiceManager的两个主要功能：
* 启动系统Service
> SystemServiceManager中有方法startService，用来启动系统服务。首先根据Service的类名字符串获取对应的Class，然后根据Class获取构造函数 Constructor<T>，最终通过Constructor的newInstance的方法创建具体的Service对象，最后调用Service的onStart方法来启动系统服务。`在调用onStart之前，把创建的Service对象加入到了mServices 中。`

* 监控启动过程
>每一个通过SystemServiceManager启动的Service，都加入到了mServices 中，而且这些Service都实现了onBootPhase。在启动的不同阶段，SystemServiceManager会调用其startBootPhase，在startBootPhase中，会遍历mServices 中的每个Service，然后调用其实现的onBootPhase，这样，每个Service就可以灵活的监控自己的启动过程了。




系统服务的命名基本上都是XXXXService，其基类主要有以下三种：
##### 1. 继承自SystemService，比如PowerManagerService
     SystemService主要用于系统服务启动阶段的动态控制，在SystemService中定义了系统服务启动的6个不同的阶段，Serivice可通过重写SystemService中的 onBootPhase(int phase)方法来监控启动过程，该方法在SystemService由SystemServiceManager统一调用。该类型的Service通过SystemServiceManager的startService方式启动和监控启动过程。
##### 2. 继承自XXXXXManagerNative，比如ActivityManagerService，继承自ActivityManagerNative
     系统服务通过binder架构给app提供服务，Service作为服务端，app作为客户端。ActivityManagerNative是binder架构中framework层的服务端。在ActivityManagerService中有个内部类LifeCycle，继承自SystemService。由于Java只支持单继承，ActivityManagerService要继承自ActivityManagerNative，又想通过SystemService实现启动过程的控制，因此通过LifeCycle对ActivityManagerService进行了一层封装，从而利用SystemServiceManager来启动和监控启动过程。

##### 3. 继承自XXXXManager.Stub，比如PackageManagerService，继承自IPackageManager.Stub
    IPackageManager.Stub也是binder架构的服务端，该种类型的Service不受SystemServiceManager控制。

`总结：以上第一种和第二种Service由SystemServiceManager启动和管理启动过程，第三种用过ServiceManager的addService方法或调用自身的main方法启动。`
#### 三. 系统服务的不同启动阶段
系统服务的启动阶段分为6个，由SystemServiceManager统一管理：
##### 1.   `PHASE_WAIT_FOR_DEFAULT_DISPLAY = 100`
> 等待默认显示，在SystemServer的startBootstrapServices中调用
##### 2. ` PHASE_LOCK_SETTINGS_READY = 480`
> 进入该阶段，服务能获取到锁屏设置的数据，在SystemServer的startOtherServices的后期中调用
##### 3. `PHASE_SYSTEM_SERVICES_READY = 500`
> 进入该阶段，服务够安全的调用系统核心服务，比如 PowerManager 或者 PackageManager，在SystemServer的startOtherServices的后期中调用，紧挨着PHASE_LOCK_SETTINGS_READY 
##### 4. ` PHASE_ACTIVITY_MANAGER_READY = 550`
> 进入该阶段，服务可以广播Intent，在AMS的systemReady方法中调用
##### 5. `PHASE_THIRD_PARTY_APPS_CAN_START = 600`
> 进入该阶段，服务可以start、bind第三方app，这些app可以通过binder机制调用系统服务，在SystemServer的在AMS的systemReady方法中调用
##### 6. `PHASE_BOOT_COMPLETED = 1000`
> 进入该阶段，服务允许用户与设备进行交互，当系统启动完成并且home应用启动完毕时，进入该阶段。系统服务可能倾向于监听这个阶段而不是注册广播去监听ACTION_BOOT_COMPLETED ，从而减少延迟。该阶段发生在AMS的finishBooting中，**`finishBooting中通知完该启动阶段后，还发出了一个著名的广播：ACTION_BOOT_COMPLETED`**。