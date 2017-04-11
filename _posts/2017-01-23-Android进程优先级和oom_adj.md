---
title: Android进程优先级和oom_adj
date: 2017-01-23 20:26:18
tags:
- Process
- Android
categories: Android
---

#### 一. Linux进程优先级概念

`Linux采用了两种不同的优先级范围：`
第一种是nice值，它的范围是从-20到+19，默认值为0；越大的nice值意味着更低的优先级-nice似乎意味着你对系统中的其他进程更“有待”。相比高nice值（低优先级）的进程，低nice值（高优先级）的进程可以获得更多的处理器事件。nice值是所有Unix系统中标准化的概念，但不同的Unix系统由于调度算法不同，因此nice值得运用方式有所差异。Linux系统中，nice的值代表时间片的比例。你可以通过ps -el命令查看系统中的进程列表，其结果中标记NI的一列就是进程对于的nice值。

 第二种是实时优先级，其值是可配置的，默认情况下它的变化范围是从0到99（包括0和99）。与nice值意义相反，越高的实时优先级数值意味着进程优先级更高。任何实时进程的优先级都高于普通的进程，也就是说实时优先级和nice优先级处于互不相交的两个范畴。你可以通过如下命令：
```bash
ps -eo state,uid,pid,ppid,rtprio,time,comm  
```
查看系统中的进程列表，以及它们对应的实时优先级（位于RTPRIO列下），其中如果有进程对应列显示“-”，则说明它不是实时进程。

![Linux进程优先级](/assets/img/blogs/processandoomadj/进程优先级.PNG)

**`总结：`**
**`实时进程：优先级范围 0 - 99，数值越大优先级越高`**
**`普通进程：优先级范围 -20 - +19，数值越小优先级越高`**


3个进程优先级的概念：

`静态优先级： `
> 不会时间而改变，内核也不会修改，只能通过系统调用改变nice值的方法区修改。优先级映射公式： static_prio = MAX_RT_PRIO + nice + 20，其中MAX_RT_PRIO = 100，那么取值区间为[100, 139]；对应普通进程；

`实时优先级：`
> 只对实时进程有意义，取值区间为[0, MAX_RT_PRIO -1]，其中MAX_RT_PRIO = 100，那么取值区间为[0, 99]；对应实时进程；

`动态优先级：`
>  调度程序通过增加或减少进程静态优先级的值，来达到奖励IO消耗型或惩罚cpu消耗型的进程，调整后的进程称为动态优先级。区间范围[0, MX_PRIO-1]，其中MX_PRIO = 140，那么取值区间为[0,139]；


#### 二.Android进程优先级
     
 Android并没有创建新的进程的概念，而是沿用了Linux中的进程相关的概念和用法。本文第一节介绍了Linux进程优先级的概念，在Android中，进程有如下优先级：
 
 ` 1. public static final int THREAD_PRIORITY_LOWEST = 19;`
  > 线程最低的优先级
    
 `2. public static final int THREAD_PRIORITY_BACKGROUND = 10;`
   > 标准的后台线程优先级。比正常的优先级略低，所以不太可能影响到用户界面的交互。

 `3. public static final int THREAD_PRIORITY_LESS_FAVORABLE = +1;`
> 降低优先级的最小值

 `4. public static final int THREAD_PRIORITY_DEFAULT = 0;    `
 > 应用线程的默认优先级
 
`5. public static final int THREAD_PRIORITY_MORE_FAVORABLE = -1;`
> 增加优先级的最小值

`6. public static final int THREAD_PRIORITY_FOREGROUND = -2;`
> 当线程正在运行与用户交互的用户界面时默认的优先级。应用通常不能调整这个优先级；当用户在操作UI时系统会自动调整应用线程的优先级。
   
`7. public static final int THREAD_PRIORITY_DISPLAY = -4;`
 > 系统显示线程的标准优先级，涉及到更新用户界面，用户一般不能改变这个优先级。

`8. public static final int THREAD_PRIORITY_URGENT_DISPLAY = -8;`
> 最重要的显示线程的优先级，用户合成屏幕和检索输入。应用一般不能改变这个优先级。

`9. public static final int THREAD_PRIORITY_AUDIO = -16;`
> 音频线程的标准优先级，应用一般不能改变这个优先级。

`10. public static final int THREAD_PRIORITY_URGENT_AUDIO = -19;`
> 最重要的音频线程的优先级，应用一般不能改变这个优先级。
 
 **`从以上进程优先级的数值范围（-19 - +19）可知，Android中的进程都是普通进程。`**

对于Android中的进程，调整线程优先级的方法为：
```java
/*
** 修改线程的优先级
*/
public static final native void setThreadPriority(int tid, int priority)
            throws IllegalArgumentException, SecurityException;

/*
**修改调用该函数的线程的优先级
*/
public static final native void setThreadPriority(int priority)
            throws IllegalArgumentException, SecurityException;
```


#### 三.Android进程的oom_adj
Android根据oom_adj的值，将系统的进程分为以下16种（在ProcessList.java中定义），oom_adj越大，进程的重要性越低，内存不够用时越有可能被回收。可通过如下命令查看具体进程的oom_adj:
```bash
cat /proc/【进程id】/oom_adj
```

在Android中，一个正在运行的进程使用ProcessRecord对象来表示，在ProcessRecord中有表示oom_adj的变量。

`1. static final int UNKNOWN_ADJ = 16;`
> 一般来说将要缓存该进程，但不知道准确值
  
**`2. static final int CACHED_APP_MAX_ADJ = 15;`**
> 仅仅持有不可见Activity的进程，可在没有任何中断的情况下被杀死。
    
**`3.  static final int CACHED_APP_MIN_ADJ = 9;`**
> 缓存进程的最小adj值
    
`4. static final int SERVICE_B_ADJ = 8;`
> B list中的Service
  
`5. static final int PREVIOUS_APP_ADJ = 7;`
> 上一个应用，一般是通过按home键返回的

`6. static final int HOME_APP_ADJ = 6;`
>home应用所在的进程，要尽量避免被杀死

` 7. static final int SERVICE_ADJ = 5;`
> 服务进程，杀死该进程一般对用户不会有太大影响

`8. static final int HEAVY_WEIGHT_APP_ADJ = 4;`
> 重量级进程，运行在后台，我们期望尽量不要杀死它。启动时通过system/rootdir/init.rc设置

**`9. static final int BACKUP_APP_ADJ = 3;`**
>正在执行备份操作的进程，杀死它不会有致命的后果，但绝不是个好主意。

**`10. static final int PERCEPTIBLE_APP_ADJ = 2;`**
> 仅仅持有对用户来说可感知的组件的进程，我们真的要避免杀死它们，但它们不是立即可见的。比如音乐后台播放服务。
 
**`11. static final int VISIBLE_APP_ADJ = 1;`**
> 仅持有用户可见的Activity的进程，我们期望这些Activity不会消失。

**`12. static final int FOREGROUND_APP_ADJ = 0;`**
> 前台进程
   
`13.  static final int PERSISTENT_SERVICE_ADJ = -11;`
> 关联着系统进程或persistent进程的进程
  
`14. static final int PERSISTENT_PROC_ADJ = -12;`
> 系统进程或persistent进程

`15. static final int SYSTEM_ADJ = -16;`
> 系统进程

`16. static final int NATIVE_ADJ = -17;`
> native进程

除了以上oom_adj常量之外，ProcessList.java还定义了以下3个常量：
```java
 private final int[] mOomAdj = new int[] {
            FOREGROUND_APP_ADJ, VISIBLE_APP_ADJ, PERCEPTIBLE_APP_ADJ,
            BACKUP_APP_ADJ, CACHED_APP_MIN_ADJ, CACHED_APP_MAX_ADJ
 };
   
private final int[] mOomMinFreeLow = new int[] {
       12288, 18432, 24576,
       36864, 43008, 49152
 };
   
 private final int[] mOomMinFreeHigh = new int[] {
       73728, 92160, 110592,
       129024, 147456, 184320
 };
```
1. mOomAdj：`当系统内存不足时，LowMemoryKiller从mOomAdj数组中定义的6个级别的进程，根据oom_adj从大到小，依次杀死进程、回收内存，一直到内存满足阈值`
2. mOomMinFreeLow：`定义低配置设备内存回收的内存阈值`
3. mOomMinFreeHigh ：`定义低配置设备内存回收的内存阈值`

一般来说，手机厂商不会修改mOomAdj中指定的进程，mOomMinFreeLow和mOomMinFreeHigh 定义的内存阈值可根据手机的具体情况修改。


`需要注意：`
* `并不一定是oom_adj越大，越容易被LMK回收。home所在的进程的oom_adj为6，但不会被LMK回收，LMK只回收特定oom_adj级别的进程。`
* `内存紧张时，LMK最多只能杀死oom_adj为0的进程（前台进程），oom_adj小于0的进程一般为系统进程，不会被回收。`
* `如果一个进程中不包含任何组件，这个进程就认为是空进程，例如一个应用只有一个Activity，当这个Activity销毁时，该进程就变成了一个空进程；当Android结束一个进程时，并不会将它立即从系统中删除，而是将它标记为cache进程，当再次启动新进程时，会优先使用cache进程，这样可以加快应用启动速度。`
