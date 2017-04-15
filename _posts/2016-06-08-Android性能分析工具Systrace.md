---
title:  Android性能分析工具Systrace
date: 2016-06-08 23:45:40
tags:
- Systrace
- Android
categories: Android
---
#### 一. 基本操作
##### 1.1 抓取和打开trace文件
可通过ADM或控制台脚本来抓取trace，利用ADM抓取trace文件的操作如下：
1. 在AndroidStudio中点击：Tools -> Android -> Android Device Monitor，打开ADM
2. 在ADM中先选择已连接的设备，然后点击捕获按钮
![选择设备](/assets/img/blogs/systrace/adm_open.PNG)
3. 设置参数
![设置参数](/assets/img/blogs/systrace/trace_opt.PNG)
设置完参数后点击确定，即进入抓取状态，此时在手机上进行要测试的操作。时间到后，在会目标文件夹生成格式为html的文件。
4. 打开chrome，在地址栏输入：chrome://tracing/，在出来的页面点击“load”，然后选择刚才生成的trace文件，此时文件会在chrome中打开

##### 1.2 基本操作
|操作(键盘按键)|作用|
|:--:|:--|
|w|放大当前视图|
|s|缩小当前视图|
|a|向左移动|
|d|向右移动|
|f|选中某一帧之后，放大当前区域|
|m|选中某一帧之后，标记该帧|
|v|高亮VSync线|
|g|是否显示60hz标记线|
|0|数字0，恢复当前视图到初始状态|
|h|是否显示详情|
|/|搜索关键字（title）|
|enter|显示搜索结果|
|`|显示、隐藏脚本控制台|
|?|显示帮助|
|1|切换到选择模式，双击已选定区能将所有相同的块高亮选中|
|2|切换到拖动模式，按下左键可拖动视图|
|3|切换到缩放模式，按下左键移动鼠标可缩放视图|
|4|切换到时间轴模式，可标记时间轴|
##### 1.3.分析trace文件
`整体情况：`
![@整体情况|center](./trace_level.PNG)
* 蓝色：整体较为流畅
* 黄色：中度掉帧
* 红色：有严重掉帧

`每一帧的情况：`
![每一帧的具体情况](/assets/img/blogs/systrace/frame.PNG)

* 蓝色：没有掉帧
* 黄色：中度掉帧
* 红色：有严重掉帧

`掉帧原因统计：`
点击页面最右侧的“Alerts”：
![掉帧原因统计](/assets/img/blogs/systrace/alert.PNG)
从该图中，可看出本次测试的整体掉帧原因统计，点击可现实详情。

`具体掉帧原因：`
点击目标线程，选择一帧（点击每一帧的圆形图标，内有字符“F”），在界面下方会现实具体的掉帧原因，点击可展开：
![@具体掉帧原因|center](./reason.PNG)
从截图中可知，当前帧出现掉帧的原因有4个。

#### 二. Systrace中的Title

下表为framework中常见的Trace title：
* Trace Title：Trace.traceBegin的第二个参数，用于表示当前监测的方法名，这个字符串不一定等于监测的函数名。
```java
Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "serviceStop");
```
* 方法名：Trace真正监测的函数名
* 类名：方法所在的类
*Trace Category：Trace.traceBegin的第一个参数，用于表示监测的类别，Systrace支持的类别如下：
![Trace Category](/assets/img/blogs/systrace/option.PNG)

Systrace中，常见的Ttitle如下（红色加粗的title比较常见）：
|序号|Trace Title|方法名|类名|Trace Category|
|:--:|:--|:--|:--|:--|
|1|`activityStart`|handleLaunchActivity|ActivityThread|ActivityManager||
|2|`activityRestart`|handleRelaunchActivity|ActivityThread|ActivityManager||
|3|`activityPause`|handlePauseActivity|ActivityThread|ActivityManager||
|4|`activityPause`|handlePauseActivity|ActivityThread|ActivityManager|pause后就finish|
|5|`activityStop`|handleStopActivity|ActivityThread|ActivityManager|stop_show|
|6|`activityStop`|handleStopActivity|ActivityThread|ActivityManager|stop_hide|
|7|`activityShowWindow`|handleWindowVisibility|ActivityThread|ActivityManager|show|
|8|`activityHideWindow`|handleWindowVisibility|ActivityThread|ActivityManager|hide|
|9|`activityResume`|handleResumeActivity|ActivityThread|ActivityManager||
|10|`activityDeliverResult`|handleSendResult|ActivityThread|ActivityManager||
|11|`activityDestroy`|handleDestroyActivity|ActivityThread|ActivityManager||
|12|`bindApplication`|handleBindApplication|ActivityThread|ActivityManager||
|13|`activityNewIntent`|handleNewIntent|ActivityThread|ActivityManager||
|14|`broadReceiverComp`|handleReceiver|ActivityThread|ActivityManager||
|15|`serviceCreate`|handleCreateService|ActivityThread|ActivityManager||
|16|`serviceBind`|handleBindService|ActivityThread|ActivityManager||
|17|`serviceUnbind`|handleUnbindService|ActivityThread|ActivityManager||
|18|`serviceStart`|handleServiceArgs|ActivityThread|ActivityManager||
|19|`serviceStop`|handleStopService|ActivityThread|ActivityManager||
|20|`configChanged`|handleConfigurationChanged|ActivityThread|ActivityManager||
|21|`lowMemory`|handleLowMemory|ActivityThread|ActivityManager||
|22|`activityConfigChanged`|handleActivityConfigurationChanged|ActivityThread|ActivityManager||
|23|`backupCreateAgent`|handleCreateBackupAgent|ActivityThread|ActivityManager||
|24|`backupDestroyAgent`|handleDestroyBackupAgent|ActivityThread|ActivityManager||
|25|`providerRemove`|completeRemoveProvider|ActivityThread|ActivityManager||
|26|`broadcastPackage`|handleDispatchPackageBroadcast|ActivityThread|ActivityManager||
|27|`sleeping`|handleSleeping|ActivityThread|ActivityManager||
|28|`setCoreSettings`|handleSetCoreSettings|ActivityThread|ActivityManager||
|29|`trimMemory`|handleTrimMemory|ActivityThread|ActivityManager||
|30|`activityThreadMain`|main|ActivityThread|ActivityManager|ActivityThread的main方法|
|31|`参数，不定`|new PathClassLoader|ApplicationLoaders|ActivityManager||
|32|`参数，不定`|new PathClassLoader|ApplicationLoaders|ActivityManager||
|33|`broadcastReceiveReg`|onReceive|LoadedAok|ActivityManager|BroadcastReceiver的onReceive|
|34|`参数，不定`|onPerformSync|AbstractThreaded|SyncManager||
|35|`参数，资源文件名`|loadDrawableForCookie|Resources|Resources|加载Drawable|
|36|`参数，资源文件名`|loadColorStateListForCookie|Resources|Resources|加载颜色|
|37|`Choreographer#doFrame`|doFrame|Choreographer|View||
|38|`参数，不定`|doCallbacks|Choreographer|View||
|39|**`inflate`**|inflate|LayoutInflater|View||
|40|`参数，view名称`|createView|LayoutInflater|View||
|41|**`Record View#draw()`**|updateRootDisplayList|ThreadedRenderer|View||
|42|`buildDrawingCache/SW Layer for+类名`|buildDrawingCache|View|View||
|43|**`measure`**|performMeasure|ViewRootImpl|View|调用View的measure方法|
|44|**`layout`**|performLayout|ViewRootImpl|View|调用View的layout方法|
|45|**`draw`**|performDraw|ViewRootImpl|View|调用View的draw方法|
|46|`getProvider()`|getProvider|WebViewFactory|WebView||
|47|`newInstance()`|getProvider|WebViewFactory|WebView||
|48|`loadNativeLibrary()`|loadNa`tiveLibrary|WebViewFactory|WebView||
|49|`getChromiumProvider()`|getChromiumProviderClass|WebViewFactory|WebView||
|50|`Class.forName(`)|getChromiumProviderClass|WebViewFactory|WebView||
|52|**`obtainView`**|obtainView|AbsListView|View||
|52|**`setupGridItem`**|setupChild|GridView|View||
|53|**`setupListItem`**|setupChild|ListView|View||
|54|`RuntimeInit`|zygoteInit|RuntimeInit|ActivityManager||
|55|`PostFork`|forkAndSpecialize|Zygote|ActivityManager||
|56|`Bitmap.compress`|compress|Bitmap|Resources||
|57|**`decodeBitmap`**|decodeByteArray|BitmapFactory|Graphics||
|58|`decodeBitmap`|decodeStream|BitmapFactory|Graphics||
|59|`decodeFileDescriptor`|decodeFileDescriptor|BitmapFactory|Graphics||
|60|`参数，文件名`|createFromStream|Drawable|Resources||
|61|`参数，文件名`|createFromResourceStream|Drawable|Resources||
|62|`参数，路径名`|createFromPath|Drawable|Resources||
|63|`onSurfaceCreated`|onSurfaceCreated|GLSurfaceView|View||
|64|`onSurfaceChanged`|onSurfaceChanged|GLSurfaceView|View||
|65|`onDrawFrame`|onDrawFrame|GLSurfaceView|View||
|66|`syncAll`|syncAll|Allocation|RenderScript||
|67|`ioSend`|ioSend|Allocation|RenderScript||
|68|`ioReceive`|ioReceive|Allocation|RenderScript||
|69|`copyFrom`|copyFrom|Allocation|RenderScript||
|70|`copyFromUnchecked`|copyFromUnchecked|Allocation|RenderScript||
|71|`copyFromUnchecked`|copyFromUnchecked|Allocation|RenderScript||
|72|`copyFrom`|copyFrom|Allocation|RenderScript||
|73|`copyFrom`|copyFrom|Allocation|RenderScript||
|74|`copyFrom`|copyFrom|Allocation|RenderScript||
|75|`copy1DRangeFromUnchecked`|copy1DRangeFromUnchecked|Allocation|RenderScript||
|76|`copy1DRangeFrom`|copy1DRangeFrom|Allocation|RenderScript||
|77|`copy2DRangeFromUnchecked`|copy2DRangeFromUnchecked|Allocation|RenderScript||
|78|`copy2DRangeFrom`|copy2DRangeFrom|Allocation|RenderScript||
|79|`copy2DRangeFrom`|copy2DRangeFrom|Allocation|RenderScript||
|80|`copy2DRangeFrom`|copy2DRangeFrom|Allocation|RenderScript||
|81|`copy3DRangeFromUnchecked`|copy3DRangeFromUnchecked|Allocation|RenderScript||
|82|`copy2DRangeFrom`|copy2DRangeFrom|Allocation|RenderScript||
|83|`copyTo`|copyTo|Allocation|RenderScript||
|84|`copyTo`|copyTo|Allocation|RenderScript||
|85|`copy1DRangeToUnchecked`|copy1DRangeToUnchecked|Allocation|RenderScript||
|86|`copy2DRangeToUnchecked`|copy2DRangeToUnchecked|Allocation|RenderScript||
|87|`copy3DRangeToUnchecked`|copy3DRangeToUnchecked|Allocation|RenderScript||
|88|`createTyped`|createTyped|Allocation|RenderScript||
|89|`createSized`|createSized|Allocation|RenderScript||
|90|`createFromBitmap`|createFromBitmap|Allocation|RenderScript||
|91|`killProcessGroup`|killProcessGroup|AMS|ActivityManager||
|92|`Start proc:+进程名`|Process.start|AMS|ActivityManager||
|93|`kill`|kill|ProcessRecord|ActivityManager||
|94|`requestGlobalDisplay`|requestGlobalDisplay|DisplayManagerService|Power||
|95|`setDisplayState`|requestDisplayStateLocked|LocalDisplayAdapter|Power||
|96|`setDisplayBrightness`|requestDisplayStateLocked|LocalDisplayAdapter|Power||
|97|`startDream`|startDream|DreamController|Power||
|98|`stopDream`|stopDream|DreamController|Power||
|99|`setLight`|setLightLocked|LightsService|Power||
|100|`userActivity`|userActivityNoUpdateLocked|PowerManagerService|Power||
|101|`wakeUp`|wakeUpNoUpdateLocked|PowerManagerService|Power||
|102|`goToSleep`|goToSleepNoUpdateLocked|PowerManagerService|Power||
|103|`nap`|napNoUpdateLocked|PowerManagerService|Power||
|104|`reallyGoToSleep`|reallyGoToSleep|PowerManagerService|Power||
|105|`updatePowerState`|updatePowerStateLocked|PowerManagerService|Power||
|106|`setHalAutoSuspend`|setHalAutoSuspend|PowerManagerService|Power||
|107|`setHalInteractive`|setHalInteractiveModeLocked|PowerManagerService|Power||
|108|`wmLayout`|performLayoutAndPlace|WMS|WindowManager||
|109|`wmUpdateFocus`|updateFocusedWindowLocked|WMS|WindowManager||
|110|`calculateError`|calcErrorRS|AutomaticActivity|Always||
|111|`loadBitmaps`|loadBitmaps|CompareActivity|Always||
|112|`softwareDraw`|loadBitmaps|CompareActivity|Always||
|113|`copyInto`|loadBitmaps|CompareActivity|Always||
|114|`pretendBusy`|performClick|CirclePropActivity|View|Thread.sleep|

`注意：由于显示问题，上表中部分内容有所精简，具体如下：`
* 34行，类名为：AbstractThreadedSyncAdapter
* 46行，title为：webViewFactory.getProvider()
* 47行，title为：providerClass.newInstance()
* 49行，title为：getChromiumProviderClass()
* 94行，title为：requestGlobalDisplayState
* 94行，方法为：requestGlobalDisplayStateInternal
* 104行，方法为：reallyGoToSleepNoUpdateLocked
* 106行，方法为：setHalAutoSuspendModeLocked
* 48行，title为：WebViewFactory.loadNativeLibrary()
* 49行，title为：WebViewFactory.getChromiumProviderClass()
* 108行，方法为：performLayoutAndPlaceSurfacesLockedLoop

#### 三. 常见卡顿原因
##### `3.1 Scheduling delay`
产生这一帧的任务延时了若干毫秒，导致出现了掉帧。要保证UI线程在工作时不会被其他线程阻塞，后台线程（比如网络请求、Bitmap加载等）应该运行在android.os.Process.THREAD_PRIORITY_BACKGROUD或更低的优先级，以确保它们不会中断UI线程。

![Scheduling delay](/assets/img/blogs/systrace/schedule_delay.PNG)


---
##### `3.2 Inefficient ListView recycling / rebinding`
ListView的回收在一帧中占用了太多的时间，确保在Adapter的getView方法中高效的绑定数据。
![Alt text](/assets/img/blogs/systrace/inefficient_recycling.PNG)




---
##### `3.3 Expensive measure/layout pass`
Measure/Layout占用了大量的时间导致掉帧，避免在执行动画时触发layout。

![Expensive measure/layout pass](/assets/img/blogs/systrace/listview_recycle.PNG)

这种场景非常常见：在首次进入一个有ListView的页面时经常出现这种情况，ListView在首次加载时，RecycleBin中并没有可以复用的数据，此时，每一个要显示的Item对于的View都需要从布局文件中inflate出来，而inflate又是比较耗时的，因此，当同时需要加载多个View时，就会导致较严重的掉帧。

---
##### `3.4 Inflating duraing ListView recycling`
ListView会回收已经加载过的View，确保在getView时复用已存在的view，而不是创新创建一个新的。
![Inflating duraing ListView recycling](/assets/img/blogs/systrace/listview_recycle.PNG)



---
##### `3.5 Long View.draw`
在View的绘制过程中要避免执行大量的耗时工作，尤其是分配和对象和绘制Bitmap
![Long View.draw](/assets/img/blogs/systrace/long_draw.PNG)



---
##### `3.6 Expensive Bitmap uploads`
被修改的Bitmap和新绘制的Bitmap都必需要上传到GPU中。如果上传的像素太多，这将是一个非常昂贵的操作，要尽量减少Bitmap的改动。
![Expensive Bitmap uploads](/assets/img/blogs/systrace/bitmapupload.PNG)

##### `3.7 gc导致线程暂停`
这种场景比较少见，而且GC导致的pause时间也较短，一般在10ms以内，很少会因为gc导致掉帧，如下面的两个截图所示：

* 如下图所示，full suspand check
![Alt text|center](/assets/img/blogs/systrace/gc1.JPG)

* Checkpoint function
![Alt text|center](/assets/img/blogs/systrace/gc2.JPG)








