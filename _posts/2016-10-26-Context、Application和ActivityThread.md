---
title:  Context、Application和ActivityThread
date: 2016-10-26 21:28:15
tags:
- Android
categories: Android
---

#### 一. 关键类介绍
* Context
>应用程序环境的全局信息的接口，Context是抽象类，由Android系统实现。

* ContextImpl
> Context中API的一个通用实现，它为Activity和其他的应用组件提供了基本的Context对象

* ContextWrapper
> Context的一个代理类，内部由CotextImpl实现

* ContextThemeWrapper
> Activity的直接基类，用于处理Activity的主题

* Application
>  保存应用全局状态的一个基类，用户可以创建自己的实现（需要在manifest中指定）。Application或其子类是一个apk中第一个被实例化的类。

* IApplicationThread
> 与应用进行通信的系统私有API，在应用启动时提供给AMS，AMS利用此对象向应用发送请求。

* ApplicaitionThreadProxy
> IApplicationThread的客户端，用于向对应的服务端发送消息

* ApplicationThreadNative
 > IApplicationThread的服务端，响应对应客户端发来的消息

* ApplicationThread
> 继承自ApplicationThreadNative，用于接收AMS发来的请求，然后将请求转给ActivityThread处理

* ActivityThread
>一个应用不仅要向AMS发送请求，某些情况下AMS也要向应用发送请求，这时应用就是Binder中的服务端，AMS是客户端，比如执行广播的onReceive。ApplicationThread作为Binder架构中的app的服务端，在收到AMS的请求后，将消息转至ActivityThread中的Handler执行。

* IActivityManager
>与AMS进行通信的系统私有API，声明了AMS中的所有公开方法

* ActivityManagerProxy
> AMS的客户端实现，用于向AMS发送消息

* ActivityManagerNative
> AMS的服务端实现，用于接收客户端发来的消息

* ActivityManagerService
>AMS，继承自ActivityManagerNative

---
#### 二. Context、Application继承关系
![Context继承关系|center](/assets/img/blogs/context_activity/context.png)

从上图中可以，一个应用中Context的数量应该为：Activity的数量+Service的数量+1个Application。如果数量比理论值多，基本上可以断定发生了Activity或Service对象的内存泄漏。

从上图中可知，Context既是Activity和Service的基类，也是Application的基类，事实上，SystemServer中也存在Context（mSystemContext）。SystemServer在启动某些系统服务时，需要传入Context。Activity、Application和SystemServer分别是三个层次的组件，并且它们都需要Context，现在来分析下Context的具体实现ContextImpl的3种创建方式(以下3个方法都是在ContextImpl中实现的)。
`1. 创建System级别的Context`
可代码中可知，创建系统的Context仅需要ActivityThread这一个参数，并且更新了系统资源的配置选项。
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

`2. 创建应用基本的Context`
每个App都有一个Application对象，LoadedApk这个类描述用于描述每一个已经加载的Apk。从代码中可知，创建应用级别的Context，不仅需要ActivityThread，还需要一个对应的LoadedApk对象。
```java
static ContextImpl createAppContext(ActivityThread mainThread, LoadedApk packageInfo) {
        if (packageInfo == null) throw new IllegalArgumentException("packageInfo");
        return new ContextImpl(null, mainThread,
                packageInfo, null, null, false, null, null, Display.INVALID_DISPLAY);
    }
```

`3. 创建Activity级别的Context`
对应一个Activity来说，不仅有对应的ActivityThread和LoadedA对象，还有跟Activity相关的Configuration对象，因此，创建Activity级别的Context需要的参数更多，创建出的Context内容也更丰富。函数如下：
```java
 static ContextImpl createActivityContext(ActivityThread mainThread,
            LoadedApk packageInfo, int displayId, Configuration overrideConfiguration) {
        if (packageInfo == null) throw new IllegalArgumentException("packageInfo");
        return new ContextImpl(null, mainThread, packageInfo, null, null, false,
                null, overrideConfiguration, displayId);
    }
```

---
#### 三. ActivityThread继承关系

![ActivityThread和ApplicationThread](/assets/img/blogs/context_activity/Class Diagram1.png)


---
#### 四. App、AMS如何通信
![Context](/assets/img/blogs/context_activity/context.png)


app既可以向AMS发送请求，AMS也可以向app发送请求，二者直接的关联如上图所示。二者之间通信的基础和框架为binder机制，本文暂不详细讨论binder机制。

##### 1. app如何向AMS发送消息
Context中定义了大量的方法，这些方法中有些是调用AMS中的方法。以Activity中sendBroadcast为例来说明：
![sendBroadcast过程](/assets/img/blogs/context_activity/sendBroadcast.png)

ActivityManagerProxy中的broadcastIntent的方法为：
```java
 public int broadcastIntent(IApplicationThread caller,
            Intent intent, String resolvedType, IIntentReceiver resultTo,
            int resultCode, String resultData, Bundle map,
            String[] requiredPermissions, int appOp, Bundle options, boolean serialized,
            boolean sticky, int userId) throws RemoteException
```
第一个参数为IApplicationThread，它向AMS表明自己的身份，当AMS需要向应用发送请求时，就可以通过caller查找目标进程对应的ProcessRecord对象，ProcessRecord类中有成员thread，类型为IApplicationThread，用于向app发送请求。ActivityMangerNative作为对应的服务端，会收到app发来的消息，最终转由AMS处理（AMS继承自ActivityManagerNative）。

##### 2. AMS如何向app发生消息
在AMS当需要向app发送消息时，首先会查找目标进程的ProcessRecord对象，在ProcessRecord中有成员thread，类型为IApplicationThread，该对象是app的binder客户端，AMS就是利用ProcessRecord中的thread向app发送消息。对应的服务端在ActivityThread中的内部类ApplicationThread，ApplicationThread中有IApplicationThread中对应的方法，当ApplicationThread收到消息时，会转由ActivityThread中的Handler处理。至此，app就收到的AMS发来的请求。
