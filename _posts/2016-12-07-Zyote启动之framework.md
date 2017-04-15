---
title:  +Zyote启动之framework
date: 2016-12-07 21:26:56
tags:
- Zygote
- Android
categories: Android
---

#### 一. Zygote启动脚本
```bash
service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server
    class main
    socket zygote stream 660 root system
    onrestart write /sys/android_power/request_state wake
    onrestart write /sys/power/state on
    onrestart restart media
    onrestart restart netd
    writepid /dev/cpuset/foreground/tasks

```

#### 二. ZygoteInit.main
```java
public static void main(String argv[]) {
        try {
            RuntimeInit.enableDdms();
            // Start profiling the zygote initialization.
            SamplingProfilerIntegration.start();

            boolean startSystemServer = false;
            String socketName = "zygote";
            String abiList = null;
            //解析启动参数，从启动脚本中可知，启动时有“start-system-server”，表明Zygote启动时也要启动SystemServer
            for (int i = 1; i < argv.length; i++) {
                if ("start-system-server".equals(argv[i])) {
                    startSystemServer = true;
                } else if (argv[i].startsWith(ABI_LIST_ARG)) {
                    abiList = argv[i].substring(ABI_LIST_ARG.length());
                } else if (argv[i].startsWith(SOCKET_NAME_ARG)) {
                    socketName = argv[i].substring(SOCKET_NAME_ARG.length());
                } else {
                    throw new RuntimeException("Unknown command line argument: " + argv[i]);
                }
            }

            if (abiList == null) {
                throw new RuntimeException("No ABI list supplied.");
            }
			
			//注册本地socket，用于监听系统请求创建进程的请求
            registerZygoteSocket(socketName);
            EventLog.writeEvent(LOG_BOOT_PROGRESS_PRELOAD_START,
                SystemClock.uptimeMillis());
            preload();
            EventLog.writeEvent(LOG_BOOT_PROGRESS_PRELOAD_END,
                SystemClock.uptimeMillis());

            SamplingProfilerIntegration.writeZygoteSnapshot();

            // 启动之后做一次gc
            gcAndFinalize();

            // 禁用tracing
            Trace.setTracingEnabled(false);
			//启动SystemServer
            if (startSystemServer) {
                startSystemServer(abiList, socketName);
            }
			//循环监听创建进程请求，理论上来说该处不会返回
            Log.i(TAG, "Accepting command socket connections");
            runSelectLoop(abiList);

            closeServerSocket();
        } catch (MethodAndArgsCaller caller) {
            caller.run();
        } catch (RuntimeException ex) {
            Log.e(TAG, "Zygote died with exception", ex);
            closeServerSocket();
            throw ex;
        }
    }
```
main中主要做了以下：
`1. 解析启动参数`
> 解析的参数有是否创建SystemServer、支持的ABI、要创建的socket的名称

`2. 创建zygote中的socket`
> 创建socket用于监听系统请求

`3. 预加载资源和类`
> 预加载系统资源和某些类，这样可以加快应用的启动速度

`4. 启动SystemServer`
> 启动系统核心进程SystemServer

`5. 循环监听系统创建进程请求`
> 监听系统创建进程的请求，一直监听、不返回
 
以上内容中1较为简单，4已经在别处分析过，本文重点分析2、3、5.
#### 三. ZygoteInit.registerZygoteSocket
```java
    private static void registerZygoteSocket(String socketName) {
        if (sServerSocket == null) {
            int fileDesc;
            final String fullSocketName = ANDROID_SOCKET_PREFIX + socketName;
            try {
                String env = System.getenv(fullSocketName);
                fileDesc = Integer.parseInt(env);
            } catch (RuntimeException ex) {
                throw new RuntimeException(fullSocketName + " unset or invalid", ex);
            }

            try {
                FileDescriptor fd = new FileDescriptor();
                fd.setInt$(fileDesc);
                sServerSocket = new LocalServerSocket(fd);
            } catch (IOException ex) {
                throw new RuntimeException(
                        "Error binding to local socket '" + fileDesc + "'", ex);
            }
        }
    }
```
#### 四. ZygoteInit.preload
```java
static void preload() {
    //预加载位于/system/etc/preloaded-classes文件中的类
    preloadClasses();
    //预加载资源，包含drawable和color资源
    preloadResources();
    //预加载OpenGL
    preloadOpenGL();
    //通过System.loadLibrary()方法，
    //预加载"android","compiler_rt","jnigraphics"这3个共享库
    preloadSharedLibraries();
    //预加载文本连接符资源
    preloadTextResources();
    WebViewFactory.prepareWebViewInZygote();
    }
```
#### 五. ZygoteInit.runSelectLoop
```java
    private static void runSelectLoop(String abiList) throws MethodAndArgsCaller {
        ArrayList<FileDescriptor> fds = new ArrayList<FileDescriptor>();
        ArrayList<ZygoteConnection> peers = new ArrayList<ZygoteConnection>();

        fds.add(sServerSocket.getFileDescriptor());
        peers.add(null);

        while (true) {
            StructPollfd[] pollFds = new StructPollfd[fds.size()];
            for (int i = 0; i < pollFds.length; ++i) {
                pollFds[i] = new StructPollfd();
                pollFds[i].fd = fds.get(i);
                pollFds[i].events = (short) POLLIN;
            }
            //IO多路复用监听socket连接
            try {
                Os.poll(pollFds, -1);
            } catch (ErrnoException ex) {
                throw new RuntimeException("poll failed", ex);
            }
            for (int i = pollFds.length - 1; i >= 0; --i) {
                if ((pollFds[i].revents & POLLIN) == 0) {
                    continue;
                }
                if (i == 0) {
                   //接收连接数据，类似于c语言中socket的accpt
                    ZygoteConnection newPeer = acceptCommandPeer(abiList);
                    peers.add(newPeer);
                    fds.add(newPeer.getFileDesciptor());
                } else {
                    //执行fork，创建子进程
                    boolean done = peers.get(i).runOnce();
                    if (done) {
                        peers.remove(i);
                        fds.remove(i);
                    }
                }
            }
        }
    }
```

runSelectLoop中通过while(true)的方式循环监听socket连接。
