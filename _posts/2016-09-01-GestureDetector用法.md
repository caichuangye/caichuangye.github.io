---
title: GestureDetector用法
date: 2016-09-01 10:40:19
tags:
- Event
- Android
categories: Android
---

#### 一. 基本用法
##### 1. 创建GestureDetector对象
GestureDetector有三个构造函数，如下：
 1. public GestureDetector(Context context, OnGestureListener listener)
 2. public GestureDetector(Context context, OnGestureListener listener, Handler handler)
 3. public GestureDetector(Context context, OnGestureListener listener, Handler handler,
            boolean unused)
其中，第一个和第三个本质上调用的是第二个构造函数，第二个构造函数有3个参数，分别为：
* Context：用来获取ViewConfiguration对象
* OnGestureListener ：事件回调，除了传入OnGestureListener 的实例，还可以传入SimpleOnGestureListener的实例
* Handler：运行事件回调所在的线程

##### 2. 调用GestureDetector的onTouchEvent函数
```java
mButton.setOnTouchListener(new View.OnTouchListener() {
      @Override
       public boolean onTouch(View v, MotionEvent event) {
            mGestureDetector.onTouchEvent(event);
            return true;
       }
});
```

#### 二. 监听回调介绍

##### 1. OnGestureListener

 * `boolean onDown(MotionEvent e)`
> 当MotionEvent的Down事件发生时，触发这个回调。只要用户触碰到屏幕，首先触发的就是该回调。
 
---
 * `void onShowPress(MotionEvent e)`
>  当MotionEvent的Down事件发生但Move和Up还没发生时会触发该回调。这个回调一般用于为用户提供视觉反馈，好让
>   用户知道他们的行为已经被识别了，比如高亮一个元素。单击、双击时不会触发，长按、移动时触发，`即Down事件持续100ms后，才会触发该回调。`
 
---
 * ` boolean onSingleTapUp(MotionEvent e)`
> 当Up事件发生时，触发这个回调，触发该事件表明一个Click事件已经发生。
 
---
 * ` boolean onScroll(MotionEvent e1, MotionEvent e2, float distanceX, float distanceY)`
 > 滑动事件发生时将触发该回调。
    @param e1： 开始滑动时的第一个Down事件
    @param e2： 触发当前事件的MotionEvent事件
    @param distanceX：  最后一次调用onScroll时在X轴上划过的距离，`不是e1和e2之间的距离`
    @param distanceY ： 最后一次调用onScroll时在Y轴上划过的距离，`不是e1和e2之间的距离`
    @如果事件被消费，返回true，否则返回false
   
---
 * ` void onLongPress(MotionEvent e)`
> 当长按事件发生时，将立即触发这个回调。`Down事件发生后500ms，并且在这段时间内没有其他的事件，`
> `则视为长按。参数e为Click的Down事件，不是Up！`
 
---
 * `boolean onFling(MotionEvent e1, MotionEvent e2, float velocityX, float velocityY)`
> 当fling动作发生时，将触发该回调   
>  @param e1：开始fling动作时的第一个Down事件
> @param e2 触发当前onFling事件的MotionEvent事件
> @param velocityX ：当前fling动作在X轴方向上每秒移动的像素数
> @param velocityY：当前fling动作在X轴方向上每秒移动的像素数
> @如果该事件被消费，返回true，否则返回false
   
---

##### 2. OnDoubleTapListener
* `boolean onSingleTapConfirmed(MotionEvent e)`
> 当单击事件发生时，触发该回调。当调用该回调时，监测者可以非常自信的确保接下来不会再有一个tap事件，如果两个tap事件间隔太近（小于300ms）就会变成双击事件。`参数e为单击事件的Down事件！`
 
---

* `boolean onDoubleTap(MotionEvent e)`
> 当双击事件发生时，触发这个回调。`第一个Click事件的Up和第二个Click事件的Down之间的间隔小于300则视为双击事件，双击事件的触发时机为第二次Down事件！`
 
---
* `boolean onDoubleTapEvent(MotionEvent e)`
> 当双击事件发生时，触发这个回调。当onDoubleTap发生后，会触发该回调，`从双击事件的第二次Down开始，直到双击事件结束，所有的事件都可以通过该回调监听，包括（down、move、up）。`
 
---
##### 3. OnContextClickListener
* ` boolean onContextClick(MotionEvent e)`
> 当上下文点击事件发生时，触发这个回调
 
---

#### 三. 典型事件
##### 1. 单击
情景1：Down与UP之间的间隔小于100ms
`onDown(down) -> onSingleTapUp(up) -> onSingleTapConfirmed(down)`

情景2：Down与UP之间的间隔大于100ms、小于500ms
`onDown(down) -> onShowPress(down) -> onSingleTapUp(up) -> onSingleTapConfirmed(down)`

情景3：Down与UP之间的间隔大于500ms，即长按
`onDown(down) -> onShowPress(down) -> onLongPress(down)`

当Up发生时，表明一个Click事件已经发生，接下来的onSingleTapConfirmed是用来确认当前发生的是单击事件，而不是双击事件，onSingleTapConfirmed发生时其参数为Down事件，该事件是单击事件的down事件，与onDown时的事件相同。

##### 2. 双击
`onDown(down) -> onSingleTapUp(up) -> onDoubleTap(down) -> onDoubleTapEvent(down) -> onDown(down)-> onDoubleTapEvent(move)-> onDoubleTapEvent(up)`
如果第一个Click事件的Up和第二个Click的Down之间的间隔小于300ms，则视为双击，否则就是两次单击。
在onSingleTapUp后，下个事件是onDoubleTap，可见双击事件的通知是在第二次Down事件发生时。接下来通过onDoubleTapEvent检测从双击事件确认发生（第二次Down）到结束后的所有动作，包括down、move和up。

##### 3. 长按
`onDown(down) -> onShowPress(down) -> onLongPress(down)`
Down事件持续500ms并且在此期间没有其他事件，则视为长按

##### 4. 滑动
`onDown(down) -> onShowPress(down) -> onScroll(move) -> ... -> onScroll(move)`


##### 5. 抛(fling)
情形1：Down持续事件小于100ms，然后执行move和up
`onDown(down) -> onScroll(move)-> ... -> onScroll(move) -> onFling(up)`

情形2：Down持续事件大于100ms，然后执行move和up
`onDown(down) -> onShowPress(down) ->onScroll(move)-> ... -> onScroll(move) -> onFling(up)`

**`总结：`**
**`1. 单击事件首先由Down触发，当Down的时间持续100ms时，会触发onShowPress，当Down持续的时间500ms时，单击事件会变为长按事件。`**
**`2. 第一次Click的Up事件和第二次Click的Down事件之间的间隔如果小于300ms，则视为双击，否则视为两次单击。`**
**`2. 长按之后，即使快速执行第二次Click，也不会被视为双击。`**




       