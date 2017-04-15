---
title: SystemServer启动过程（native）
date: 2016-11-03 23:42:51
tags:
- Android
categories: Android
---
### +++SystemServer启动过程（native）
[TOC]

SystemServer作为系统的一个核心的进程，在Zygote进程中被启动。SystemServer的启动过程可以分为两步，一是在Zygote中中通过fork产生子进程，然后调用SystemServer的main方法；二执行是framework层的SystemServer的构造函数。本文讨论SystemServer在native层的启动过程。

native层SystemServer的启动过程如下图所示：

![SystemServer启动过程](/assets/img/blogs/systemserver/SystemServer.png)


SystemServer进程是通过fork产生的，fork执行一次，会返回两次（分别返回到当前进程即Zygote进程和子进程）。本文将从根据fork的不同返回值，讨论其启动过程。

#### 一. 在Zygote进程中执行fork
##### 1.1 ZygoteInit.main
```cpp
 public static void main(String argv[]) {
        try {   
			...			
            if (startSystemServer) {
                startSystemServer(abiList, socketName);
            }          
            ...          
        } catch (MethodAndArgsCaller caller) {
            caller.run();
        } catch (RuntimeException ex) {        
            ...            
            throw ex;
        }
    }
```

在Zygote的main方法中，通过startSystemServer开始启动SystemServer。注意main内部抛出的异常，类型为MethodAndArgsCaller ：
```java
    public static class MethodAndArgsCaller extends Exception
            implements Runnable {
        /** method to call */
        private final Method mMethod;

        /** argument array */
        private final String[] mArgs;

        public MethodAndArgsCaller(Method method, String[] args) {
            mMethod = method;
            mArgs = args;
        }

        public void run() {
            try {
                mMethod.invoke(null, new Object[] { mArgs });
            } catch (IllegalAccessException ex) {
                throw new RuntimeException(ex);
            } catch (InvocationTargetException ex) {
                Throwable cause = ex.getCause();
                if (cause instanceof RuntimeException) {
                    throw (RuntimeException) cause;
                } else if (cause instanceof Error) {
                    throw (Error) cause;
                }
                throw new RuntimeException(ex);
            }
        }
    
```
MethodAndArgsCaller 的run方法通过反射的方式，调用传入的method方法。具体的调用时机本文稍后会具体分析。
`小结：main方法中通过startSystemServer启动SystemServer进程，并通过捕获异常的方式载入SystemServer的main方法。`

##### 1.2 ZygoteInit.startSystemServer
```java
    private static boolean startSystemServer(String abiList, String socketName)
            throws MethodAndArgsCaller, RuntimeException {
     ...
        String args[] = {
            "--setuid=1000",
            "--setgid=1000",
            "--setgroups=1001,1002,1003,1004,1005,1006,1007,1008,1009,1010,1018,1021,1032,3001,3002,3003,3006,3007",
            "--capabilities=" + capabilities + "," + capabilities,
            "--nice-name=system_server",
            "--runtime-args",
            "com.android.server.SystemServer",
        };
        ZygoteConnection.Arguments parsedArgs = null;
		...
        int pid;
		...
        try {
         
            pid = Zygote.forkSystemServer(
                    parsedArgs.uid, parsedArgs.gid,
                    parsedArgs.gids,
                    parsedArgs.debugFlags,
                    null,
                    parsedArgs.permittedCapabilities,
                    parsedArgs.effectiveCapabilities);
        } catch (IllegalArgumentException ex) {
            throw new RuntimeException(ex);
        }

        /* For child process */
        if (pid == 0) {
            if (hasSecondZygote(abiList)) {
                waitForSecondaryZygote(socketName);
            }

            handleSystemServerProcess(parsedArgs);
        }

        return true;
    
```
startSystemServer主要做了三件事：
>1. 构造SystemServer的启动参数，比如uid、gid等
>2. 调用Zygote的静态方法forkSystemServer
>3. forkSystemServer返回值其实就是fork的返回值，fork会返回两次，当pid == 0，表示当前是在子进程的环境中。在子进程（即创建的SystemServer进程）中继续处理启动过程


##### 1.3 Zygote.forkSystemServer
```java
 public static int forkSystemServer(int uid, int gid, int[] gids, int debugFlags,
            int[][] rlimits, long permittedCapabilities, long effectiveCapabilities) {
        VM_HOOKS.preFork();
        int pid = nativeForkSystemServer(
                uid, gid, gids, debugFlags, rlimits, permittedCapabilities, effectiveCapabilities);
        // Enable tracing as soon as we enter the system_server.
        if (pid == 0) {
            Trace.setTracingEnabled(true);
        }
        VM_HOOKS.postForkCommon();
        return pid;
    }
```
调用native方法nativeForkSystemServer继续启动过程，`该方法在com_android_internal_os_Zygote.cpp中实现。`
##### 1.4 com_android_internal_os_Zygote.nativeForkSystemServer
```cpp
static jint com_android_internal_os_Zygote_nativeForkSystemServer(
        JNIEnv* env, jclass, uid_t uid, gid_t gid, jintArray gids,
        jint debug_flags, jobjectArray rlimits, jlong permittedCapabilities,
        jlong effectiveCapabilities) {
  pid_t pid = ForkAndSpecializeCommon(env, uid, gid, gids,
                                      debug_flags, rlimits,
                                      permittedCapabilities, effectiveCapabilities,
                                      MOUNT_EXTERNAL_DEFAULT, NULL, NULL, true, NULL,
                                      NULL, NULL);
  if (pid > 0) {
      // The zygote process checks whether the child process has died or not.
      ALOGI("System server process %d has been created", pid);
      gSystemServerPid = pid;
      // There is a slight window that the system server process has crashed
      // but it went unnoticed because we haven't published its pid yet. So
      // we recheck here just to make sure that all is well.
      int status;
      if (waitpid(pid, &status, WNOHANG) == pid) {
          ALOGE("System server process %d has died. Restarting Zygote!", pid);
          RuntimeAbort(env);
      }
  }
  return pid;
}
```
通过ForkAndSpecializeCommon继续启动过程。fork之后，通过waitpid监控SystemServer创建失败的场景。当SystemServer创建失败时，将重启Zygote。

##### 1.5 com_android_internal_os_Zygote.ForkAndSpecializeCommon
```cpp
static pid_t ForkAndSpecializeCommon(JNIEnv* env, uid_t uid, gid_t gid, jintArray javaGids,
                                     jint debug_flags, jobjectArray javaRlimits,
                                     jlong permittedCapabilities, jlong effectiveCapabilities,
                                     jint mount_external,
                                     jstring java_se_info, jstring java_se_name,
                                     bool is_system_server, jintArray fdsToClose,
                                     jstring instructionSet, jstring dataDir) {
  SetSigChldHandler();

#ifdef ENABLE_SCHED_BOOST
  SetForkLoad(true);
#endif
//通过fork创建子进程
  pid_t pid = fork();

//pid == 0,表示在子进程中返回
  if (pid == 0) {
    // The child process.
    gMallocLeakZygoteChild = 1;

    // Clean up any descriptors which must be closed immediately
    DetachDescriptors(env, fdsToClose);

    // Keep capabilities across UID change, unless we're staying root.
    if (uid != 0) {
      EnableKeepCapabilities(env);
    }

    DropCapabilitiesBoundingSet(env);

    bool use_native_bridge = !is_system_server && (instructionSet != NULL)
        && android::NativeBridgeAvailable();
    if (use_native_bridge) {
      ScopedUtfChars isa_string(env, instructionSet);
      use_native_bridge = android::NeedsNativeBridge(isa_string.c_str());
    }
    if (use_native_bridge && dataDir == NULL) {
      use_native_bridge = false;
      ALOGW("Native bridge will not be used because dataDir == NULL.");
    }

    if (!MountEmulatedStorage(uid, mount_external, use_native_bridge)) {
      ALOGW("Failed to mount emulated storage: %s", strerror(errno));
      if (errno == ENOTCONN || errno == EROFS) {   
      } else {
        ALOGE("Cannot continue without emulated storage");
        RuntimeAbort(env);
      }
    }

    if (!is_system_server) {
        int rc = createProcessGroup(uid, getpid());
        if (rc != 0) {
            if (rc == -EROFS) {
                ALOGW("createProcessGroup failed, kernel missing CONFIG_CGROUP_CPUACCT?");
            } else {
                ALOGE("createProcessGroup(%d, %d) failed: %s", uid, pid, strerror(-rc));
            }
        }
    }

    SetGids(env, javaGids);

    SetRLimits(env, javaRlimits);

    if (use_native_bridge) {
      ScopedUtfChars isa_string(env, instructionSet);
      ScopedUtfChars data_dir(env, dataDir);
      android::PreInitializeNativeBridge(data_dir.c_str(), isa_string.c_str());
    }

    int rc = setresgid(gid, gid, gid);
    if (rc == -1) {
      ALOGE("setresgid(%d) failed: %s", gid, strerror(errno));
      RuntimeAbort(env);
    }

    rc = setresuid(uid, uid, uid);
    if (rc == -1) {
      ALOGE("setresuid(%d) failed: %s", uid, strerror(errno));
      RuntimeAbort(env);
    }

    if (NeedsNoRandomizeWorkaround()) {
        // Work around ARM kernel ASLR lossage (http://b/5817320).
        int old_personality = personality(0xffffffff);
        int new_personality = personality(old_personality | ADDR_NO_RANDOMIZE);
        if (new_personality == -1) {
            ALOGW("personality(%d) failed: %s", new_personality, strerror(errno));
        }
    }

    SetCapabilities(env, permittedCapabilities, effectiveCapabilities);

    SetSchedulerPolicy(env);

    const char* se_info_c_str = NULL;
    ScopedUtfChars* se_info = NULL;
    if (java_se_info != NULL) {
        se_info = new ScopedUtfChars(env, java_se_info);
        se_info_c_str = se_info->c_str();
        if (se_info_c_str == NULL) {
          ALOGE("se_info_c_str == NULL");
          RuntimeAbort(env);
        }
    }
    const char* se_name_c_str = NULL;
    ScopedUtfChars* se_name = NULL;
    if (java_se_name != NULL) {
        se_name = new ScopedUtfChars(env, java_se_name);
        se_name_c_str = se_name->c_str();
        if (se_name_c_str == NULL) {
          ALOGE("se_name_c_str == NULL");
          RuntimeAbort(env);
        }
    }
    rc = selinux_android_setcontext(uid, is_system_server, se_info_c_str, se_name_c_str);
    if (rc == -1) {
      ALOGE("selinux_android_setcontext(%d, %d, \"%s\", \"%s\") failed", uid,
            is_system_server, se_info_c_str, se_name_c_str);
      RuntimeAbort(env);
    }

    if (se_info_c_str == NULL && is_system_server) {
      se_name_c_str = "system_server";
    }
    if (se_info_c_str != NULL) {
      SetThreadName(se_name_c_str);
    }

    delete se_info;
    delete se_name;

    UnsetSigChldHandler();

    env->CallStaticVoidMethod(gZygoteClass, gCallPostForkChildHooks, debug_flags,
                              is_system_server ? NULL : instructionSet);
    if (env->ExceptionCheck()) {
      ALOGE("Error calling post fork hooks.");
      RuntimeAbort(env);
    }
  } else if (pid > 0) {
    // the parent process

#ifdef ENABLE_SCHED_BOOST
    // unset scheduler knob
    SetForkLoad(false);
#endif

  }
  return pid;
}
}  
```
在这里，终于看到了关键的fork函数！fork之后，设置了uid、gid、子进程信号处理函数等。至此，Zygote通过fork成功创建了一个子进程，需要注意的是：此时只执行了fork，没有执行exec，即此时创建的子进程只是父进程Zygote的一个副本，并没有真正载入SystemServer的代码。SystemServer代码的调用在ZygoteInit中通过handleSystemServerProcess载入。


#### 二. 在SystemServer进程中载入SystemServer
##### 2.1 ZygoteInit.handleSystemServerProcess
```java
private static void handleSystemServerProcess(
            ZygoteConnection.Arguments parsedArgs)
            throws ZygoteInit.MethodAndArgsCaller {
		//关闭从父进程继承而来的socket，这是fork的常见用法
        closeServerSocket();

        // set umask to 0077 so new files and directories will default to owner-only permissions.
        Os.umask(S_IRWXG | S_IRWXO);
		//设置进程名为system_server
        if (parsedArgs.niceName != null) {
            Process.setArgV0(parsedArgs.niceName);
        }
		//优化dex文件
        final String systemServerClasspath = Os.getenv("SYSTEMSERVERCLASSPATH");
        if (systemServerClasspath != null) {
            performSystemServerDexOpt(systemServerClasspath);
        }

        if (parsedArgs.invokeWith != null) {
            String[] args = parsedArgs.remainingArgs;
            // If we have a non-null system server class path, we'll have to duplicate the
            // existing arguments and append the classpath to it. ART will handle the classpath
            // correctly when we exec a new process.
            if (systemServerClasspath != null) {
                String[] amendedArgs = new String[args.length + 2];
                amendedArgs[0] = "-cp";
                amendedArgs[1] = systemServerClasspath;
                System.arraycopy(parsedArgs.remainingArgs, 0, amendedArgs, 2, parsedArgs.remainingArgs.length);
            }
			//启动应用
            WrapperInit.execApplication(parsedArgs.invokeWith,
                    parsedArgs.niceName, parsedArgs.targetSdkVersion,
                    VMRuntime.getCurrentInstructionSet(), null, args);
        } else {
            ClassLoader cl = null;
            if (systemServerClasspath != null) {
                cl = new PathClassLoader(systemServerClasspath, ClassLoader.getSystemClassLoader());
                Thread.currentThread().setContextClassLoader(cl);
            }

            /*
             * Pass the remaining arguments to SystemServer.
             */
             //继续初始化
            RuntimeInit.zygoteInit(parsedArgs.targetSdkVersion, parsedArgs.remainingArgs, cl);
        }

        /* should never reach here */
    }

```
##### 2.2 RuntimeInit.zygoteInit
```java
public static final void zygoteInit(int targetSdkVersion, String[] argv, ClassLoader classLoader)
            throws ZygoteInit.MethodAndArgsCaller {
        if (DEBUG) Slog.d(TAG, "RuntimeInit: Starting application from zygote");

        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "RuntimeInit");
        redirectLogStreams();

        commonInit();
        nativeZygoteInit();
        applicationInit(targetSdkVersion, argv, classLoader);
    }
```
zygoteInit中主要调用了三个函数：
* commonInit：通用初始化
* nativeZygoteInit：初始化zygote
* applicationInit：初始化应用
下面逐个分析：
`1. commonInit`
```java
private static final void commonInit() {
        if (DEBUG) Slog.d(TAG, "Entered RuntimeInit!");
        //设置默认的未捕获的异常处理方法
        Thread.setDefaultUncaughtExceptionHandler(new UncaughtHandler());

		//设置时区
        TimezoneGetter.setInstance(new TimezoneGetter() {
            @Override
            public String getId() {
                return SystemProperties.get("persist.sys.timezone");
            }
        });
        TimeZone.setDefault(null);

		//重新配置log
        LogManager.getLogManager().reset();
        new AndroidConfig();
          
        //设置默认的HTTP User-agent格式,用于HttpURLConnection  
        String userAgent = getDefaultUserAgent();
        System.setProperty("http.agent", userAgent);
			
		//设置socket的tag，用于网络流量统计
        NetworkManagementSocketTagger.install();

        String trace = SystemProperties.get("ro.kernel.android.tracing");
        if (trace.equals("1")) {
            Slog.i(TAG, "NOTE: emulator trace profiling enabled");
            Debug.enableEmulatorTraceOutput();
        }

        initialized = true;
    }
```
`2. nativeZygoteInit`
```
static void com_android_internal_os_RuntimeInit_nativeZygoteInit(JNIEnv* env, jobject clazz)
{
    gCurRuntime->onZygoteInit();
}
```
nativeZygoteInit最终调用的上app_main.cpp中的类AppRuntime的onZygoteInit方法：
```cpp
virtual void onZygoteInit()
    {
        sp<ProcessState> proc = ProcessState::self();
        ALOGV("App process: starting thread pool.\n");
        proc->startThreadPool();
    }
```
在onZygoteInit中打开binder驱动，然后通过startThreadPool循环监听binder上收到的消息。该处涉及到binder机制，在此不做深入分析。

`3. applicationInit`
```java
private static void applicationInit(int targetSdkVersion, String[] argv, ClassLoader classLoader)
            throws ZygoteInit.MethodAndArgsCaller {
        nativeSetExitWithoutCleanup(true);
        //设置heap的利用率为0.75     
        VMRuntime.getRuntime().setTargetHeapUtilization(0.75f);
        //设置targetVersion
        VMRuntime.getRuntime().setTargetSdkVersion(targetSdkVersion);

        final Arguments args;
        try {
            args = new Arguments(argv);
        } catch (IllegalArgumentException ex) {
            Slog.e(TAG, ex.getMessage());
            // let the process exit
            return;
        }
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
		//调用invokeStaticMain，载入SystemServer的main方法
        invokeStaticMain(args.startClass, args.startArgs, classLoader);
    }
```
##### 2.4RuntimeInit.invokeStaticMain
```java
    private static void invokeStaticMain(String className, String[] argv, ClassLoader classLoader)
            throws ZygoteInit.MethodAndArgsCaller {
        Class<?> cl;

        try {
            cl = Class.forName(className, true, classLoader);
        } catch (ClassNotFoundException ex) {
            throw new RuntimeException(
                    "Missing class when invoking static main " + className,
                    ex);
        }

        Method m;
   
            m = cl.getMethod("main", new Class[] { String[].class });
       

        int modifiers = m.getModifiers();
        if (! (Modifier.isStatic(modifiers) && Modifier.isPublic(modifiers))) {
            throw new RuntimeException(
                    "Main method is not public and static on " + className);
        }

    
        throw new ZygoteInit.MethodAndArgsCaller(m, argv);
    }
```
1. 根据SystemServer类的路径，创建对应的Class对象
2. 获取SystemServer的main方法，并且对main的修饰符进行有效性判断：必须满足static 和 public
3. 将main方法包装到Method中，构造MethodAndArgsCaller，然后抛出异常。本文的1.1节分析了类MethodAndArgsCaller，在ZygoteInit的main方法中将捕获到这个异常，然后执行其run方法中执行SystemServer的main方法。至此，SystemServer的子进程创建完毕、并且SystemServer的代码也载入到了SystemServer的进程中。

**`总结：一般来说，fork之后都会立即执行exec，但在SystemServer的启动过程中，只有fork，exec的工作是通过线程载入目标进程的main方法来实现的。`**