---
title: Zygote启动之native
date: 2016-09-23 00:05:53
tags:
- Zygote
- Android
categories: Android
---

Zygote进程是Android世界中的第一个进程，其父进程为init进程。Zygote是系统启动时吗，通过解析zygote.rc脚本来启动的，Zygote启动时会创建一个虚拟机，并注册了系统的JNI方法，考虑如下问题：
1. Zygote是由app_main.cpp负责启动的，很明显，这是在native层的逻辑，在Zygote的真正实现是在ZygoteInit这个类中，这个类是Java代码实现的，这就意味着ZygoteInit无法通过fork和exec的方式启动，ZygoteInit该如何启动呢？
2. Android中每一个进程都运行在一个虚拟机中，是不是说每次启动一个应用时都要创建一个新的虚拟机？
3. 每个应用在运行时都需要加载Java类和Android的核心类和资源，而这些过程又是非常耗时的，Android是如何解决这个问题的？

在分析完Zyogte的启动过程后，再回答上面的三个问题。

#### 一. 解析启动参数
Zygote启动参数如下：
```java
service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server
    class main
    socket zygote stream 660 root system
    onrestart write /sys/android_power/request_state wake
    onrestart write /sys/power/state on
    onrestart restart media
    onrestart restart netd
    writepid /dev/cpuset/foreground/tasks

```
Zygote启动执行的文件为app_main.cpp，入口为main方法，在main中，主要分为两步：
1. 解析启动参数
根据上文中的启动参数可知，可确定：启动的进程为Zygote，进程名为“zygote”，启动完毕后要启动SystemServer进程

2. 构造AppRuntime对象，根据解析出的启动参数，执行AppRuntime的start方法。AppRuntime继承AndroidRuntime，重写了启动生命过程中的几个函数，start方法还是在AndroidRuntime中定义的
```java
int main(int argc, char* const argv[])
{
    if (prctl(PR_SET_NO_NEW_PRIVS, 1, 0, 0, 0) < 0) {
        // Older kernels don't understand PR_SET_NO_NEW_PRIVS and return
        // EINVAL. Don't die on such kernels.
        if (errno != EINVAL) {
            LOG_ALWAYS_FATAL("PR_SET_NO_NEW_PRIVS failed: %s", strerror(errno));
            return 12;
        }
    }

    AppRuntime runtime(argv[0], computeArgBlockSize(argc, argv));
    // Process command line arguments
    // ignore argv[0]
    argc--;
    argv++;

    int i;
    for (i = 0; i < argc; i++) {
        if (argv[i][0] != '-') {
            break;
        }
        if (argv[i][1] == '-' && argv[i][2] == 0) {
            ++i; // Skip --.
            break;
        }
        runtime.addOption(strdup(argv[i]));
    }

    // Parse runtime arguments.  Stop at first unrecognized option.
    bool zygote = false;
    bool startSystemServer = false;
    bool application = false;
    String8 niceName;
    String8 className;

    ++i;  // Skip unused "parent dir" argument.
    while (i < argc) {
        const char* arg = argv[i++];
        if (strcmp(arg, "--zygote") == 0) {
            zygote = true;
            niceName = ZYGOTE_NICE_NAME;
        } else if (strcmp(arg, "--start-system-server") == 0) {
            startSystemServer = true;
        } else if (strcmp(arg, "--application") == 0) {
            application = true;
        } else if (strncmp(arg, "--nice-name=", 12) == 0) {
            niceName.setTo(arg + 12);
        } else if (strncmp(arg, "--", 2) != 0) {
            className.setTo(arg);
            break;
        } else {
            --i;
            break;
        }
    }

    Vector<String8> args;
    if (!className.isEmpty()) {
        args.add(application ? String8("application") : String8("tool"));
        runtime.setClassNameAndArgs(className, argc - i, argv + i);
    } else {
        // We're in zygote mode.
        maybeCreateDalvikCache();

        if (startSystemServer) {
            args.add(String8("start-system-server"));
        }

        char prop[PROP_VALUE_MAX];
        if (property_get(ABI_LIST_PROPERTY, prop, NULL) == 0) {
            LOG_ALWAYS_FATAL("app_process: Unable to determine ABI list from property %s.",
                ABI_LIST_PROPERTY);
            return 11;
        }

        String8 abiFlag("--abi-list=");
        abiFlag.append(prop);
        args.add(abiFlag);

        // In zygote mode, pass all remaining arguments to the zygote
        // main() method.
        for (; i < argc; ++i) {
            args.add(String8(argv[i]));
        }
    }

    if (!niceName.isEmpty()) {
        runtime.setArgv0(niceName.string());
        set_process_name(niceName.string());
    }

    if (zygote) {
        runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
    } else if (className) {
        runtime.start("com.android.internal.os.RuntimeInit", args, zygote);
    } else {
        fprintf(stderr, "Error: no class name or --zygote supplied.\n");
        app_usage();
        LOG_ALWAYS_FATAL("app_process: no class name or --zygote supplied.");
        return 10;
    }
}

```
AndroidRuntime的start方法如下，在start中，主要做了四件事：
1. 启动虚拟机
2. 注册JNI
3. 载入ZygoteInit的main方法
4. Zygote退出时，清理资源
```java
void AndroidRuntime::start(const char* className, const Vector<String8>& options, bool zygote)
{
    ALOGD(">>>>>> START %s uid %d <<<<<<\n",
            className != NULL ? className : "(unknown)", getuid());

    static const String8 startSystemServer("start-system-server");

    /*
     * 'startSystemServer == true' means runtime is obsolete and not run from
     * init.rc anymore, so we print out the boot start event here.
     */
    for (size_t i = 0; i < options.size(); ++i) {
        if (options[i] == startSystemServer) {
           /* track our progress through the boot sequence */
           const int LOG_BOOT_PROGRESS_START = 3000;
           LOG_EVENT_LONG(LOG_BOOT_PROGRESS_START,  ns2ms(systemTime(SYSTEM_TIME_MONOTONIC)));
        }
    }

    const char* rootDir = getenv("ANDROID_ROOT");
    if (rootDir == NULL) {
        rootDir = "/system";
        if (!hasDir("/system")) {
            LOG_FATAL("No root directory specified, and /android does not exist.");
            return;
        }
        setenv("ANDROID_ROOT", rootDir, 1);
    }

    //const char* kernelHack = getenv("LD_ASSUME_KERNEL");
    //ALOGD("Found LD_ASSUME_KERNEL='%s'\n", kernelHack);

    /* 启动虚拟机 */
    JniInvocation jni_invocation;
    jni_invocation.Init(NULL);
    JNIEnv* env;
    if (startVm(&mJavaVM, &env, zygote) != 0) {
        return;
    }
    onVmCreated(env);

    /*
     *注册Android中的JNI方法
     */
    if (startReg(env) < 0) {
        ALOGE("Unable to register all android natives\n");
        return;
    }

   
    jclass stringClass;
    jobjectArray strArray;
    jstring classNameStr;

    stringClass = env->FindClass("java/lang/String");
    assert(stringClass != NULL);
    strArray = env->NewObjectArray(options.size() + 1, stringClass, NULL);
    assert(strArray != NULL);
    classNameStr = env->NewStringUTF(className);
    assert(classNameStr != NULL);
    env->SetObjectArrayElement(strArray, 0, classNameStr);

    for (size_t i = 0; i < options.size(); ++i) {
        jstring optionsStr = env->NewStringUTF(options.itemAt(i).string());
        assert(optionsStr != NULL);
        env->SetObjectArrayElement(strArray, i + 1, optionsStr);
    }

    /*
     * 载入ZygoteInit的main方法
     */
    char* slashClassName = toSlashClassName(className);
    jclass startClass = env->FindClass(slashClassName);
    if (startClass == NULL) {
        ALOGE("JavaVM unable to locate class '%s'\n", slashClassName);
        /* keep going */
    } else {
        jmethodID startMeth = env->GetStaticMethodID(startClass, "main",
            "([Ljava/lang/String;)V");
        if (startMeth == NULL) {
            ALOGE("JavaVM unable to find main() in '%s'\n", className);
            /* keep going */
        } else {
            env->CallStaticVoidMethod(startClass, startMeth, strArray);

#if 0
            if (env->ExceptionCheck())
                threadExitUncaughtException(env);
#endif
        }
    }
    free(slashClassName);

	//zygote退出，清理资源
    ALOGD("Shutting down VM\n");
    if (mJavaVM->DetachCurrentThread() != JNI_OK)
        ALOGW("Warning: unable to detach main thread\n");
    if (mJavaVM->DestroyJavaVM() != 0)
        ALOGW("Warning: VM did not shut down cleanly\n");
}
```
#### 二. 启动虚拟机
```java
int AndroidRuntime::startVm(JavaVM** pJavaVM, JNIEnv** pEnv, bool zygote){
	...
	  if (JNI_CreateJavaVM(pJavaVM, pEnv, &initArgs) < 0) {
        ALOGE("JNI_CreateJavaVM failed\n");
        return -1;
    }
    return 0;
}
```
#### 三. 注册JNI
```java
int AndroidRuntime::startReg(JNIEnv* env)
{
    /*
     * This hook causes all future threads created in this process to be
     * attached to the JavaVM.  (This needs to go away in favor of JNI
     * Attach calls.)
     */
    androidSetCreateThreadFunc((android_create_thread_fn) javaCreateThreadEtc);

    ALOGV("--- registering native functions ---\n");

    /*
     * Every "register" function calls one or more things that return
     * a local reference (e.g. FindClass).  Because we haven't really
     * started the VM yet, they're all getting stored in the base frame
     * and never released.  Use Push/Pop to manage the storage.
     */
    env->PushLocalFrame(200);

    if (register_jni_procs(gRegJNI, NELEM(gRegJNI), env) < 0) {
        env->PopLocalFrame(NULL);
        return -1;
    }
    env->PopLocalFrame(NULL);

    //createJavaThread("fubar", quickTest, (void*) "hello");

    return 0;
}

```
#### 四. 启动ZygoteInit
在app_main.cpp的main方法中，启动AndroidRuntime的参数如下：
```java
    runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
```
可知在Zygote所在的虚拟机创建完毕后，将通过JNIEnv载入com.android.internal.os.ZygoteInit这个类：
1. 根据传入的类名构造ZygoteInit的jclass对象
2. 通过JNIEnv的GetStaticMethodID的方法，获取ZygoteInit的main方法
3. 通过JNI的CallStaticVoidMethod的的方法，执行ZygoteInit的main方法
```java
    char* slashClassName = toSlashClassName(className);
    jclass startClass = env->FindClass(slashClassName);
    if (startClass == NULL) {
        ALOGE("JavaVM unable to locate class '%s'\n", slashClassName);
        /* keep going */
    } else {
        jmethodID startMeth = env->GetStaticMethodID(startClass, "main",
            "([Ljava/lang/String;)V");
        if (startMeth == NULL) {
            ALOGE("JavaVM unable to find main() in '%s'\n", className);
            /* keep going */
        } else {
            env->CallStaticVoidMethod(startClass, startMeth, strArray);

#if 0
            if (env->ExceptionCheck())
                threadExitUncaughtException(env);
#endif
        }
    }
```
#### 五. 退出清理
在系统运行过程中，是不应该走到这一步的。Zygote启动后，会通过while(true)的方式循环监听系统创建进程的请求。
Zygote退出时，要执行两个清理函数：
* DetachCurrentThread：将当前进程中主线程的JNIEnv取消与虚拟机的关联
* DestroyJavaVM：销毁虚拟机
```
  ALOGD("Shutting down VM\n");
  if (mJavaVM->DetachCurrentThread() != JNI_OK)
        ALOGW("Warning: unable to detach main thread\n");
  if (mJavaVM->DestroyJavaVM() != 0)
        ALOGW("Warning: VM did not shut down cleanly\n");
```

**`现在来回答本文开头的三个问题：`**
`1. Zygote创建过程中会创建虚拟机，并创建虚拟机中主线程的JNIEnv对象，ZygoteInit是通过JNIEnv中的方法载入的`
`2. 每次启动应用时会创建新的虚拟机，但是是通过copy on wirte的技术实现的。每次创建应用时，zygote会通过fork自身进程，来    创建出一个新的进程，fork只是创建了一个zygote进程的副本，并没有完全从头的创建一个新的进程`
`3. zygote启动过程中提前加载的核心类和Android资源，存放在虚拟机中的zygote heap中，子进程会直接复用zygote进程的这个zygote heap，这样会大大加快应用的启动速度`