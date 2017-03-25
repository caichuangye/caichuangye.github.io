---
title: ViewManager接口分析
date: 2017-02-04 22:46:49
tags:
- View
- Android
categories: Android
---

#### `一. ViewManger接口定义`
```java
public interface ViewManager{
    /**
    * 将view添加到一个Activity的window上
    */
    public void addView(View view, ViewGroup.LayoutParams params);

	/**
    * 更新view布局参数
    */
    public void updateViewLayout(View view, ViewGroup.LayoutParams params);

	/**
    * 从window中删除view
    */
    public void removeView(View view);
}
```
ViewManger中定义的接口是在WindowManageImpl中实现的，但仅仅是个壳，WindowManagerImpl有类型为WindowManagerGlobal的变量mGlobal，以上3个接口真正是在mGlobal中实现的，具体的关系为：
![ViewManager接口实现](/assets/img/blogs/viewmanager/viewmanager.PNG)
接下来分析WindowManagerGlobal中addView、updateViewLayout和removeView的实现过程。

#### `二. addView代码分析`
```java
public void addView(View view, ViewGroup.LayoutParams params,
            Display display, Window parentWindow) {
        ...
        ViewRootImpl root;
        root = new ViewRootImpl(view.getContext(), display);
        view.setLayoutParams(wparams);
        ...
        mViews.add(view);
        mRoots.add(root);
        mParams.add(wparams);
        ...
        try {
            root.setView(view, wparams, panelParentView);
        } catch (RuntimeException e) {
            ...
            throw e;
        }
    }    
```
addView的代码较长，其核心逻辑主要分为以下：
1. 为目标View创建ViewRootImpl对象，由此可见，每次addView都会创建一个ViewRootImpl对象
2. 保存目标View、ViewRootImpl和布局参数
3. 将目标view添加到ViewRootImpl中（调用setView方法）

ViewRootImpl的setView方法较长，在此重点分析setView中调用的requestLayout函数：
```java
 @Override
    public void requestLayout() {
        if (!mHandlingLayoutInLayoutRequest) {
            //检查当前线程是否为ui线程
            checkThread();
            mLayoutRequested = true;
            //开始遍历
            scheduleTraversals();
        }
    }
```
scheduleTraversals代码如下：
```java
 void scheduleTraversals() {
        if (!mTraversalScheduled) {
            mTraversalScheduled = true;
            mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
            mChoreographer.postCallback(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
            if (!mUnbufferedInputDispatch) {
                scheduleConsumeBatchedInput();
            }
            notifyRendererOfFramePending();
            pokeDrawLockIfNeeded();
        }
    }

   final class TraversalRunnable implements Runnable {
        @Override
        public void run() {
            doTraversal();
        }
    }
    final TraversalRunnable mTraversalRunnable = new TraversalRunnable();

```
在scheduleTraversals中，向mChoreographer提交一个TraversalRunnable 类型的Runnable，接下来分析doTraversal：
```java
 void doTraversal() {
        if (mTraversalScheduled) {
            mTraversalScheduled = false;
            mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);

            if (mProfile) {
                Debug.startMethodTracing("ViewAncestor");
            }

            performTraversals();

            if (mProfile) {
                Debug.stopMethodTracing();
                mProfile = false;
            }
        }
    }
```
在doTraversal中调用了performTraversals，代码较长，主要做了的measure、layout和draw操作：
```java
 private void performTraversals() {
	...
	performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
 	...
	performLayout(lp, desiredWindowWidth, desiredWindowHeight);
    ...
	performDraw();
	...
 }
```
至此，进入了每个View的measure、layout和draw流程！！！
![performTraversals](/assets/img/blogs/viewmanager/performTraversals.png)


#### `三. updateViewLayout代码分析`
```java
 public void updateViewLayout(View view, ViewGroup.LayoutParams params) {
		 //view不可为空
        if (view == null) {
            throw new IllegalArgumentException("view must not be null");
        }
        //判断布局参数类型，updateViewLayout的参数要求第二个参数的类型为ViewGroup.LayoutParams，但实际上必须传入
        //WindowManager.LayoutParams，它是ViewGroup.LayoutParams的子类
        if (!(params instanceof WindowManager.LayoutParams)) {
            throw new IllegalArgumentException("Params must be WindowManager.LayoutParams");
        }

        final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams)params;

        view.setLayoutParams(wparams);

        synchronized (mLock) {
            int index = findViewLocked(view, true);
            ViewRootImpl root = mRoots.get(index);
            //删除之前的布局参数
            mParams.remove(index);
            //重新添加
            mParams.add(index, wparams);
            //重设布局参数
            root.setLayoutParams(wparams, false);
        }
    }
```

#### `四. removeView代码分析`
```java
public void removeView(View view, boolean immediate) {
        if (view == null) {
            throw new IllegalArgumentException("view must not be null");
        }

        synchronized (mLock) {
            int index = findViewLocked(view, true);
            View curView = mRoots.get(index).getView();
            removeViewLocked(index, immediate);
            if (curView == view) {
                return;
            }

            throw new IllegalStateException("Calling with view " + view
                    + " but the ViewAncestor is attached to " + curView);
        }
    }
```
1. 判断要移除的View是否为空，为空则抛出异常
2. 从mViews查找出当前view对应的索引，mViews的类型为ArrayList，在addView时，View和ViewRootImpl是同步添加到两个不同的list中，因此它们的索引是对应的，即从mViews中获取的索引index，在mRoots中也是正确的。
3. 根据step2中获取的索引获取之前存储的view
4. 通过removeViewLocked移除查找出的view
5. 若查找出的view与传入的view不同，表示传入的view之前没有被add，抛出异常

removeViewLocked源码如下，注意参数immediate为true，意思是立即销毁view
```java
private void removeViewLocked(int index, boolean immediate) {
		//查找要删除的view对应的ViewRootImpl
        ViewRootImpl root = mRoots.get(index);
        View view = root.getView();
        //dismiss输入法窗口
        if (view != null) {
            InputMethodManager imm = InputMethodManager.getInstance();
            if (imm != null) {
                imm.windowDismissed(mViews.get(index).getWindowToken());
            }
        }
        //销毁view，immediate为true
        //die方法返回true表示要延迟销毁，在该处，die返回false，表示立即销毁
        //ViewRootImpl的die方法在此不分析了
        boolean deferred = root.die(immediate);
        if (view != null) {
            view.assignParent(null);
            if (deferred) {
                mDyingViews.add(view);
            }
        }
    }
```


#### `五. ViewManager示例用法`
```java
 @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        ViewManager vm = getWindowManager();
        View view = LayoutInflater.from(MainActivity.this).inflate(R.layout.test_layout, null);
        WindowManager.LayoutParams params = new WindowManager.LayoutParams();
        params.gravity = Gravity.TOP | Gravity.LEFT;
        params.format = PixelFormat.TRANSLUCENT;
        params.windowAnimations = 0;
        params.alpha = 0.6f;
        params.flags = WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE
                | WindowManager.LayoutParams.FLAG_NOT_TOUCHABLE
                | WindowManager.LayoutParams.FLAG_KEEP_SCREEN_ON
                | WindowManager.LayoutParams.FLAG_LAYOUT_IN_SCREEN
                | WindowManager.LayoutParams.FLAG_FULLSCREEN;

        params.width = 1080;
        params.height = 1920;
        params.x = 0;
        params.y = 0;
        vm.addView(view, params);
    }
```
#### `六. 问题讨论`
**`有三个问题：`**
`1. 在另一篇博客中讨论了Activity的onCreate的setContentView的实现过程，在此过程中，初始化了PhoneWindow的DecorView。在该示例中，并没有调用setContentView方法，这是否意味着此时PhoneWindow中就没有DecorView呢？如果有，是在何时被创建的呢？`
该示例中有DecorView。具体创建时机，暂时还未发现在哪！如何判断有DecorView：查看Activity的布局层次，可以看到其中有一个id为'content'的FrameLayout，该FrameLayout存在于DecorView之中。

`2. 问题1中的确有DecorView，那么DecorView与通过ViewManager的addView方法创建的View之间有什么关系呢？`
DecorView与通过addView添加的view之间没有直接联系，二者不过都显示在同一个PhoneWindow中而已。

`3. DecorView与通过addView方法添加的view，有root view吗？`
DecorView与用户添加的View都是直接显示到PhoneWindow上的，没有父布局。但二者的getRootView都返回不为空的View，为其自身，即DecorView和用户添加的view的root view都是其自身。

DecorView可能的初始化时机：
1. public void setContentView(View view, ViewGroup.LayoutParams params)
2. public void addContentView(View view, ViewGroup.LayoutParams params)
3. `public final View getDecorView()`
4.  private ImageView getLeftIconView()
5.  private ImageView getRightIconView()
6.  private ProgressBar getCircularProgressBar(boolean shouldInstallDecor) 
7.  private ProgressBar getHorizontalProgressBar(boolean shouldInstallDecor) 

