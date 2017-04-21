---
title: VelocityTracker用法
date: 2017-03-25 21:44:02
tags:
- VelocityTracker
- Android
categories: Android
---

#### VelocityTracker是用来计算触摸事件速度的帮助类，主要用法如下：

* 获取对象：

```
if(mTracker == null){
     mTracker = VelocityTracker.obtain();
}else{
    mTracker.clear();
}
```


* 跟踪事件

```
mTracker.addMovement(event);
```

* 计算速度

```
mTracker.computeCurrentVelocity(1000);
float xVel = mTracker.getXVelocity();
float yVel = mTracker.getYVelocity();
```
* 释放资源：
```
mTracker.recycle();
mTracker = null;
```
--------------------------------------------------------------------
VelocityTraker有两个方法，需要注意：
1. recycle：把一个VelocityTracker对象恢复到可被重复使用的状态，执行完该函数后不应再使用VelocityTracker对象
```
public void recycle() {
    if (mStrategy == null) {
       clear();
       sPool.release(this);
    }
}
```

2. release：把VelocityTracker复位到初始状态
```
public void clear() {
    nativeClear(mPtr);
}
```

-------------------------------------------------------------------
完整实例代码：
```
 private VelocityTracker mTracker;
 @Override
    public boolean onTouchEvent(MotionEvent event){
        if(mTracker == null){
            mTracker = VelocityTracker.obtain();
        }else{
            mTracker.clear();
        }
        mTracker.addMovement(event);
        switch (event.getAction()){
            case MotionEvent.ACTION_DOWN:

                break;
            case MotionEvent.ACTION_MOVE:
                mTracker.computeCurrentVelocity(1000);
                float xVel = mTracker.getXVelocity();
                float yVel = mTracker.getYVelocity();
                Log.d("test","x = "+xVel+"; y = "+yVel);
                break;
            case MotionEvent.ACTION_CANCEL:
            case MotionEvent.ACTION_UP:
                mTracker.recycle();
                mTracker = null;
                break;
        }
        return  super.onTouchEvent(event);
    }
```