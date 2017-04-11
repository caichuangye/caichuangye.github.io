---
title: Android WatchDog原理分析
date: 2017-02-01 21:34:43
tags:
- WatchDog
- Android
categories: Android
---

### 一 . WatchDog作用

 *SytemServer进程中运行将近一百种服务，是最有可能出现问题的进程，而且这些进程为系统提供核心的服务，一旦出现问题，将影响系统的正常运行。因此，有必要对SystemServer中的各种服务进行监控。Android开发了WatchDog类来监控SystemServer中的服务，一旦发现有服务被死锁或消息队列被阻塞，Watchdog将会杀死SystemServer进程。SytemServer的父进程Zygote进程收到SystemServer的死亡信号后，会杀死自己。Init进程收到Zygote进程的死亡消息后，会杀死Zygote进程的所有子进程并重新Zygote，相当于系统进行了一次软重启。*


#### 1. 监控服务中的方法死锁
>服务中的方法若使用的全局的资源，考虑到多线程的场景，必须使用synchronized关键字来同步，若某个方法一直持有锁，该服务中的其他方法将无法获取到锁。下面的这个函数来自ActivityManagerService，若该函数一直持有this锁，则AMS中其他需要该锁的方法将不能正常执行。
```java
public void showWaitingForDebugger(IApplicationThread who, boolean waiting) {
        synchronized (this) {
            ProcessRecord app =
                who != null ? getRecordForAppLocked(who) : null;
            if (app == null) return;

            Message msg = Message.obtain();
            msg.what = WAIT_FOR_DEBUGGER_MSG;
            msg.obj = app;
            msg.arg1 = waiting ? 1 : 0;
            mUiHandler.sendMessage(msg);
        }
    }
```
#### 2. 监控消息队列循环
>Android中的每个线程都关联到一个Looper，Looper管理了一个MessageQueue，用于管理线程的消息队>列。如果一个线程的MessageQueue不能正常的进行轮训处理，则消息队里中的靠后的Message将不能得到及时有效的处理。
#### 3. 接收系统重启广播
>WatchDog内部注册了一个BroadcastReceiver，监听Intent.ACTION_REBOOT广播，当收到Intent.ACTION_REBOOT时，通过调用PowerManagerService中的reboot方法来重启系统。

### 二 . 用法和启动时机
#### 1. 启动时机
SystemServer将所有的服务分为三类，分别启动，启动方法分别为startBootstrapServices、startCoreServices、startOtherServices，在startOtherServices中，会创建WatchDog的单例对象并初始化，代码如下：
```java
Slog.i(TAG, "Init Watchdog");
final Watchdog watchdog = Watchdog.getInstance();
watchdog.init(context, mActivityManagerService);
```
#### 2. 用法
Watchdog在初始化时，监听了5个公共线程，分别为：
* MainThread
* FgThread
* UiThread
* IoThread
* DisplayThread
WatchDog在init方法中注册了对重启广播的监听：
```java
public void init(Context context, ActivityManagerService activity) {
    mResolver = context.getContentResolver();
    mActivity = activity;
    context.registerReceiver(new RebootRequestReceiver(),
        new IntentFilter(Intent.ACTION_REBOOT),
        android.Manifest.permission.REBOOT, null);
}
```

WatchDog中有public方法：
```java
/**
* 添加要监控的服务
*/
public void addMonitor(Monitor monitor) {
        synchronized (this) {
            if (isAlive()) {
                throw new RuntimeException("Monitors can't be added once the Watchdog is running");
            }
            mMonitorChecker.addMonitor(monitor);
        }
    }

/**
* 添加要监控的线程
*/
public void addThread(Handler thread) {
       addThread(thread, DEFAULT_TIMEOUT);
}
```
一个服务要想被WatchDog监控，必须实现Monitor接口，定义如下：
```java
public interface Monitor {
    void monitor();
}
```

若要监控线程的消息队列，通过。调用WatchDog的addThread即可。

### 三. 原理分析

1. WatchDog内部类
```java
  public final class HandlerChecker implements Runnable {
        private final Handler mHandler;
        private final String mName;
        private final long mWaitMax;
        private final ArrayList<Monitor> mMonitors = new ArrayList<Monitor>();
        private boolean mCompleted;
        private Monitor mCurrentMonitor;
        private long mStartTime;

        HandlerChecker(Handler handler, String name, long waitMaxMillis) {
            mHandler = handler;
            mName = name;
            mWaitMax = waitMaxMillis;
            mCompleted = true;
        }

        public void addMonitor(Monitor monitor) {
            mMonitors.add(monitor);
        }
        ...
 }
```
线程死锁最终通过WatchDog的内部类HandlerChecker 实现的，具体分析其中重要的变量：
* Handler mHandler：表示当前要监控的线程Handler
* long mWaitMax：要等待的时间，等待mWaitMax之后要是还没返回则视为超时
* ArrayList<Monitor> mMonitors：一个HandlerChecker对象可监控多个线程的死锁，需要注意的是，一个HandlerChecker内部所监控所有的线程都运行在同一个Handler上，即mHandler。

一个线程要想通过HandlerChecker来监控死锁，有两种方式：
` 方法一 . 实现interface Monitor接口，通过尝试获取当前线程的锁来监控线程运行状态，用这种方法监控的线程必须运行在指定的线程上。`
` 方法二 . 该方法可监控任意线程的，通过监控线程Handler的MessageQueue的状态来判断当前消息循环是否正常运行。`
以上两个方法的区别在于添加监控和判断死锁的方法不同，大体上是一致的，下面详细分析两种方法。

**`方法1：`**
Monitor的接口如下：
```java
public interface Monitor {
        void monitor();
    }
```
一般的实现方式为：
```java
 public void monitor() {
        synchronized (this) { }
 }
```
WatchDog内部已经实现了一个HandlerChecker：
```
       mMonitorChecker = new HandlerChecker(FgThread.getHandler(),
                "foreground thread", DEFAULT_TIMEOUT);
```
该HandlerChecker会加入到WatchDog的final ArrayList<HandlerChecker> mHandlerCheckers = new ArrayList<>()中。

通过Monitor监控的线程，必须运行在FgThread中。

线程先实现Monitor接口，然后通过WatchDongle中的addMonitor将自身加入到mMonitorChecker 中。AMS就是通过这种方式监控线程死锁的。
```java
  public void addMonitor(Monitor monitor) {
        synchronized (this) {
            if (isAlive()) {
                throw new RuntimeException("Monitors can't be added once the Watchdog is running");
            }
            mMonitorChecker.addMonitor(monitor);
        }
    }
```
**`方法2：`**
该方法通过addThread的方式将目标线程加入到WatchDog监控，方法如下：
```java
 public void addThread(Handler thread, long timeoutMillis) {
        synchronized (this) {
            if (isAlive()) {
                throw new RuntimeException("Threads can't be added once the Watchdog is running");
            }
            final String name = thread.getLooper().getThread().getName();
            mHandlerCheckers.add(new HandlerChecker(thread, name, timeoutMillis));
        }
    }
```
可见，该方法新建了一个HandlerChecker对象，然后加入到WatchDog的mHandlerCheckers中，WatchDog会对所有的HandlerChecker做统一死锁检查。

以上分析了两种通过WatchDog监控死锁的方法，现在来分析了WatchDog的构造函数。WatchDog为单例模式，构造函数如下：
```java
 private Watchdog() {
        super("watchdog");    
        mMonitorChecker = new HandlerChecker(FgThread.getHandler(),
                "foreground thread", DEFAULT_TIMEOUT);
        mHandlerCheckers.add(mMonitorChecker);     
        mHandlerCheckers.add(new HandlerChecker(new Handler(Looper.getMainLooper()),
                "main thread", DEFAULT_TIMEOUT));
        // Add checker for shared UI thread.
        mHandlerCheckers.add(new HandlerChecker(UiThread.getHandler(),
                "ui thread", DEFAULT_TIMEOUT));
        // And also check IO thread.
        mHandlerCheckers.add(new HandlerChecker(IoThread.getHandler(),
                "i/o thread", DEFAULT_TIMEOUT));
        // And the display thread.
        mHandlerCheckers.add(new HandlerChecker(DisplayThread.getHandler(),
                "display thread", DEFAULT_TIMEOUT));

        // Initialize monitor for Binder threads.
        addMonitor(new BinderThreadMonitor());
    }
```
* 在构造函数中首先创建了mMonitorChecker 对象，该对象用来监控实现Monitor接口的线程死锁
* 创建main、ui、i\o、display的线程监控对象
* 将Binder线程加入到mMonitorChecker 监控中

核心检测流程如下：
```java
  @Override
    public void run() {
        boolean waitedHalf = false;
        //循环监听所有的线程
        while (true) {
            final ArrayList<HandlerChecker> blockedCheckers;
            final String subject;
            final boolean allowRestart;
            int debuggerWasConnected = 0;
            synchronized (this) {
                long timeout = CHECK_INTERVAL;
                //对每一个HandlerChecker，执行超时监测
                for (int i=0; i<mHandlerCheckers.size(); i++) {
                    HandlerChecker hc = mHandlerCheckers.get(i);
                    //向线程的Handler发送Runnable任务
                    hc.scheduleCheckLocked();
                }
               //记录开始时间
                long start = SystemClock.uptimeMillis();
                //睡眠30s
                while (timeout > 0) {
                    if (Debug.isDebuggerConnected()) {
                        debuggerWasConnected = 2;
                    }
                    try {
                        wait(timeout);
                    } catch (InterruptedException e) {
                        Log.wtf(TAG, e);
                    }
                    if (Debug.isDebuggerConnected()) {
                        debuggerWasConnected = 2;
                    }
                    timeout = CHECK_INTERVAL - (SystemClock.uptimeMillis() - start);
                }
				
				//获取当前的状态
                final int waitState = evaluateCheckerCompletionLocked();
                //已经完成，说明线程状态正常
                if (waitState == COMPLETED) {
                    // The monitors have returned; reset
                    waitedHalf = false;
                    continue;
                } else if (waitState == WAITING) {
                    // 正在等待线程返回
                    continue;
                } else if (waitState == WAITED_HALF) {
                    if (!waitedHalf) {                
                        ArrayList<Integer> pids = new ArrayList<Integer>();
                        pids.add(Process.myPid());
                        ActivityManagerService.dumpStackTraces(true, pids, null, null,
                                NATIVE_STACKS_OF_INTEREST);
                        waitedHalf = true;
                    }
                    continue;
                }

                // 走到这说明当前有线程发生了死锁             
                blockedCheckers = getBlockedCheckersLocked();
                subject = describeCheckersLocked(blockedCheckers);
                allowRestart = mAllowRestart;
            }

            //收集系统所有进程的堆栈信息
            EventLog.writeEvent(EventLogTags.WATCHDOG, subject);
            ArrayList<Integer> pids = new ArrayList<Integer>();
            //WatchDog是在AMS中运行的，而AMS又是在SystemServer进程中启动的，因此Process.myPid()获取的就是SystemServer的进程id
            pids.add(Process.myPid());
            if (mPhonePid > 0) pids.add(mPhonePid);
            // Pass !waitedHalf so that just in case we somehow wind up here without having
            // dumped the halfway stacks, we properly re-initialize the trace file.
            final File stack = ActivityManagerService.dumpStackTraces(
                    !waitedHalf, pids, null, null, NATIVE_STACKS_OF_INTEREST);

            //睡眠2秒，保证堆栈信息能完全写入
            SystemClock.sleep(2000);

            // Pull our own kernel thread stacks as well if we're configured for that
            if (RECORD_KERNEL_THREADS) {
                dumpKernelStackTraces();
            }
       
            doSysRq('w');
            doSysRq('l');

            // Try to add the error to the dropbox, but assuming that the ActivityManager
            // itself may be deadlocked.  (which has happened, causing this statement to
            // deadlock and the watchdog as a whole to be ineffective)
            Thread dropboxThread = new Thread("watchdogWriteToDropbox") {
                    public void run() {
                        mActivity.addErrorToDropBox(
                                "watchdog", null, "system_server", null, null,
                                subject, null, stack, null);
                    }
                };
            dropboxThread.start();
            try {
                dropboxThread.join(2000);  // wait up to 2 seconds for it to return.
            } catch (InterruptedException ignored) {}

            IActivityController controller;
            synchronized (this) {
                controller = mController;
            }
            if (controller != null) {
                Slog.i(TAG, "Reporting stuck state to activity controller");
                try {
                    Binder.setDumpDisabled("Service dumps disabled due to hung system process.");
                    // 1 = keep waiting, -1 = kill system
                    int res = controller.systemNotResponding(subject);
                    if (res >= 0) {
                        Slog.i(TAG, "Activity controller requested to coninue to wait");
                        waitedHalf = false;
                        continue;
                    }
                } catch (RemoteException e) {
                }
            }

            // Only kill the process if the debugger is not attached.
            if (Debug.isDebuggerConnected()) {
                debuggerWasConnected = 2;
            }
            if (debuggerWasConnected >= 2) {
                Slog.w(TAG, "Debugger connected: Watchdog is *not* killing the system process");
            } else if (debuggerWasConnected > 0) {
                Slog.w(TAG, "Debugger was connected: Watchdog is *not* killing the system process");
            } else if (!allowRestart) {
                Slog.w(TAG, "Restart not allowed: Watchdog is *not* killing the system process");
            } else {
                Slog.w(TAG, "*** WATCHDOG KILLING SYSTEM PROCESS: " + subject);
                for (int i=0; i<blockedCheckers.size(); i++) {
                    Slog.w(TAG, blockedCheckers.get(i).getName() + " stack trace:");
                    StackTraceElement[] stackTrace
                            = blockedCheckers.get(i).getThread().getStackTrace();
                    for (StackTraceElement element: stackTrace) {
                        Slog.w(TAG, "    at " + element);
                    }
                }
                Slog.w(TAG, "*** GOODBYE!");
                //杀掉SystemServer
                Process.killProcess(Process.myPid());
                System.exit(10);
            }

            waitedHalf = false;
        }
    } 
```
每次循环都要检测所有的Handler有没有发生死锁，即只要有一个发生了死锁，SystemServer都会被WatchDog杀死。因此在判断死锁检测，需要获取所有的Handler中最坏的情况：
```java
 private int evaluateCheckerCompletionLocked() {
        int state = COMPLETED;
        for (int i=0; i<mHandlerCheckers.size(); i++) {
            HandlerChecker hc = mHandlerCheckers.get(i);
            state = Math.max(state, hc.getCompletionStateLocked());
        }
        return state;
    }
```
线程运行状态由以下四种：
* `static final int COMPLETED = 0; 30秒内检测函数返回，说明线程运行良好`
* `static final int WAITING = 1; 30秒内检测函数为返回，正在等待`
* ` static final int WAITED_HALF = 2; 等待时间已过半，正在等待`
* `static final int OVERDUE = 3; 已出现超时`

可以，数值越大，情况越糟糕！

run中有一个非常核心的函数：
```java
 public void scheduleCheckLocked() {
            if (mMonitors.size() == 0 && mHandler.getLooper().getQueue().isPolling()) {            
                mCompleted = true;
                return;
            }

            if (!mCompleted) {
                // we already have a check in flight, so no need
                return;
            }

            mCompleted = false;
            mCurrentMonitor = null;
            mStartTime = SystemClock.uptimeMillis();
            mHandler.postAtFrontOfQueue(this);
        }
```
`if (mMonitors.size() == 0 && mHandler.getLooper().getQueue().isPolling())`:这个if分支是针对上文中第二种通过WatchDog监控的死锁判断方法，第二种方法是通过addThread的方式来创建一个HandlerChecker，因此HandlerChecker内部的mMonitor的size等于0.若Handler中的MessageQueue的isPolling返回true，说明当前MessageQueue的消息循环处于正常状态，没有发生阻塞。

` mHandler.postAtFrontOfQueue(this);`:这句是针对上文中的第一个方法来检测死锁的，HandlerChecker实现了Runnable接口，如下：

```java
   @Override
        public void run() {
            final int size = mMonitors.size();
            for (int i = 0 ; i < size ; i++) {
                synchronized (Watchdog.this) {
                    mCurrentMonitor = mMonitors.get(i);
                }
                mCurrentMonitor.monitor();
            }

            synchronized (Watchdog.this) {
                mCompleted = true;
                mCurrentMonitor = null;
            }
        }
```
在上文中介绍了，第一种方法中，线程需要实现Monitor接口，线程一般在monitor中获取当前的锁。若目标线程运行正常，monitor会立刻返回，否则monitor可能在60s内都没有返回，即发生了死锁。