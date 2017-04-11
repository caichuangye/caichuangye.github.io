---
title:  DecorView显示过程
date: 2017-02-25 22:06:13
tags:
- Memory Leak
- Android
categories: Android
---
当Activity执行完setContentView后，界面元素已被添加到DecorView中。从ActivityThread的handleResumeActivity方法开始，执行DecorView的显示过程。具体流程如下：

![DecorView添加到窗口过程](/assets/img/blogs/decorview/时序图.png)


#### `1. ActivityThread.handleResumeActivity`
```java
final void handleResumeActivity(IBinder token,
            boolean clearHide, boolean isForward, boolean reallyResume) {
     ...
     ActivityClientRecord r = performResumeActivity(token, clearHide);
     ...       
	 if (r.window == null && !a.mFinished && willBeVisible) {
                r.window = r.activity.getWindow();
                View decor = r.window.getDecorView();
                decor.setVisibility(View.INVISIBLE);
                ViewManager wm = a.getWindowManager();//①
                WindowManager.LayoutParams l = r.window.getAttributes();
                a.mDecor = decor;
                l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
                l.softInputMode |= forwardBit;
                if (a.mVisibleFromClient) {
                    a.mWindowAdded = true;
                    wm.addView(decor, l);//②
                }
	  }
	  ...
}
```
取出PhoneWindow中的DecorView对象和布局参数，然后调用ViewManager的addView方法将DecorView加入到窗口中。



#### `2. ViewManager.addView`

在代码中的①，获取了Activity中的WindowManager对象，类型为ViewManager ，ViewManager的定义如下：
```java
public interface ViewManager{
    public void addView(View view, ViewGroup.LayoutParams params);
    public void updateViewLayout(View view, ViewGroup.LayoutParams params);
    public void removeView(View view);
}
```
ViewManger主要用于在Activity上添加、删除或更新View的布局，Activity的getWindowManager()方法返回的其实是WindowManagerImpl，ViewManager、WindowManager、WindowManagerImpl三者之间的关系为：
![ViewManager、WindowManager、WindowManagerImpl关系](/assets/img/blogs/decorview/viewmanager.PNG)

#### `3. WindowManagerImpl.addView`
ViewManger的addView真正在WindowManagerImpl中现，具体如下：
```java
	...
    private final WindowManagerGlobal mGlobal = WindowManagerGlobal.getInstance();
    ...
    @Override
    public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
        applyDefaultToken(params);
        mGlobal.addView(view, params, mDisplay, mParentWindow);
    }

```
可知，WinowManagerImpl的addView方法仅仅是调用了WindowManagerGlobal的addView方法。

#### `4. WindowManagerGlobal.addView`
```java
public void addView(View view, ViewGroup.LayoutParams params,
            Display display, Window parentWindow) {
        ...
        ViewRootImpl root;
        View panelParentView = null;
        synchronized (mLock) {
            ...
            root = new ViewRootImpl(view.getContext(), display);
            view.setLayoutParams(wparams);
            mViews.add(view);
            mRoots.add(root);
            mParams.add(wparams);
        }
        try {
            root.setView(view, wparams, panelParentView);
        } catch (RuntimeException e) {
           ...
        }
    }
```
首先创建ViewRootImpl对象root，然后设置DecorView（即参数view）的布局参数，最后调用root的setView方法添加view，接下来分析setView的实现过程。
每次执行addView都会创建一个与之对应的ViewRootImpl对象，对象存放到mRoots中。一个view既然可以被add到window上，自然也就可以被remove，当一个view被remove时，需要从mRoots中找到与该view对应的ViewRootImpl对象，然后执行具体的remove操作。ViewRootImpl中有一个类型为View的变量mView，在addView时，mView会被赋值为addView的第一个参数view，可通过ViewRootImpl的getView方法返回mView对象。

#### `5. ViewRootImpl.setView`
至此，Activity中的的布局文件已经被添加到DecorView中，但界面中的每个View元素大小、布局位置等信息还没有计算出来，在setView中调用了requestLayout方法，来递归计算每个view的位置、大小和绘制。

#### `6. ViewRootImpl.requestLayout`
```java
  @Override
    public void requestLayout() {
        if (!mHandlingLayoutInLayoutRequest) {
            checkThread();
            mLayoutRequested = true;
            scheduleTraversals();
        }
    }
```
在真正执行遍历布局元素之前，需要检查当前线程是否为主线程。只有主线程才允许操作view。

#### `7. ViewRootImpl.scheduleTraversals`
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

```
在scheduleTraversals中，向mChoreographer post了一个mTraversalRunnable，定义如下
```java
 final class TraversalRunnable implements Runnable {
        @Override
        public void run() {
            doTraversal();
        }
    } 
 final TraversalRunnable mTraversalRunnable = new TraversalRunnable();
```
可知，scheduleTraversals主要调用了doTraversal方法

#### `8. ViewRootImpl.doTraversal`
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
在doTraversal中，通过调用performTraversals实现真正的处理界面元素。

#### `9. ViewRootImpl.performTraversals`
perfromTraversals过程如下：

![performTraversals过程](/assets/img/blogs/decorview/performTraversals.png)


