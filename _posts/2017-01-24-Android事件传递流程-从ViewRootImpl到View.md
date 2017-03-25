---
title: Android事件传递流程-从ViewRootImpl到View
date: 2017-01-24 22:46:49
tags:
- View
- Android
categories: Android
---

#### Step1. 从ViewRootImpl到DecorView
![从驱动到DecorView](img/author.jpg)
                            
##### 1.1 ViewRootImpl.dispatchInputEvent
```java
public void dispatchInputEvent(InputEvent event, InputEventReceiver receiver) {
        SomeArgs args = SomeArgs.obtain();
        args.arg1 = event;
        args.arg2 = receiver;
        Message msg = mHandler.obtainMessage(MSG_DISPATCH_INPUT_EVENT, args);
        msg.setAsynchronous(true);
        mHandler.sendMessage(msg);
    }
```
ViewRootImpl中有一个内部类ViewRootHandler，继承自Handler，此时的消息会交由ViewRootHandler处理，核心代码如下：
```java
  case MSG_DISPATCH_INPUT_EVENT: {
                SomeArgs args = (SomeArgs)msg.obj;
                InputEvent event = (InputEvent)args.arg1;
                InputEventReceiver receiver = (InputEventReceiver)args.arg2;
                enqueueInputEvent(event, receiver, 0, true);
                args.recycle();
            } break;
```
注意：此时调用enqueueInputEvent，最后一个参数为true，表明要立即处理当前事件。

##### 1.2 ViewRootImpl.enqueueInputEvent
```java
void enqueueInputEvent(InputEvent event,
            InputEventReceiver receiver, int flags, boolean processImmediately) {
        adjustInputEventForCompatibility(event);
        QueuedInputEvent q = obtainQueuedInputEvent(event, receiver, flags);   
        QueuedInputEvent last = mPendingInputEventTail;
        if (last == null) {
            mPendingInputEventHead = q;
            mPendingInputEventTail = q;
        } else {
            last.mNext = q;
            mPendingInputEventTail = q;
        }
        mPendingInputEventCount += 1;
        Trace.traceCounter(Trace.TRACE_TAG_INPUT, mPendingInputEventQueueLengthCounterName,
                mPendingInputEventCount);

        if (processImmediately) {
            doProcessInputEvents();
        } else {
            scheduleProcessInputEvents();
        }
    }
```
按照先后顺序将QueuedInputEvent 加入到当前队列中，然后调用doProcessInputEvents。

##### 1.3 ViewRootImpl.doProcessInputEvents
```java
void doProcessInputEvents() {
        // Deliver all pending input events in the queue.
        while (mPendingInputEventHead != null) {
            QueuedInputEvent q = mPendingInputEventHead;
            mPendingInputEventHead = q.mNext;
            if (mPendingInputEventHead == null) {
                mPendingInputEventTail = null;
            }
            q.mNext = null;

            mPendingInputEventCount -= 1;
            Trace.traceCounter(Trace.TRACE_TAG_INPUT, mPendingInputEventQueueLengthCounterName,
                    mPendingInputEventCount);

            long eventTime = q.mEvent.getEventTimeNano();
            long oldestEventTime = eventTime;
            if (q.mEvent instanceof MotionEvent) {
                MotionEvent me = (MotionEvent)q.mEvent;
                if (me.getHistorySize() > 0) {
                    oldestEventTime = me.getHistoricalEventTimeNano(0);
                }
            }
            mChoreographer.mFrameInfo.updateInputEventTime(eventTime, oldestEventTime);

            deliverInputEvent(q);
        }

        // We are done processing all input events that we can process right now
        // so we can clear the pending flag immediately.
        if (mProcessInputEventsScheduled) {
            mProcessInputEventsScheduled = false;
            mHandler.removeMessages(MSG_PROCESS_INPUT_EVENTS);
        }
    }
```
循环从pending队列中取出未处理的事件，然后调用deliverInputEvent处理。

##### 1.4 ViewRootImpl.deliverInputEvent
```java
private void deliverInputEvent(QueuedInputEvent q) {
        Trace.asyncTraceBegin(Trace.TRACE_TAG_VIEW, "deliverInputEvent",
                q.mEvent.getSequenceNumber());
        if (mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onInputEvent(q.mEvent, 0);
        }

        InputStage stage;
        if (q.shouldSendToSynthesizer()) {
            stage = mSyntheticInputStage;
        } else {
            stage = q.shouldSkipIme() ? mFirstPostImeInputStage : mFirstInputStage;
        }

        if (stage != null) {
            stage.deliver(q);
        } else {
            finishInputEvent(q);
        }
    }

  private void finishInputEvent(QueuedInputEvent q) {
        Trace.asyncTraceEnd(Trace.TRACE_TAG_VIEW, "deliverInputEvent",
                q.mEvent.getSequenceNumber());

        if (q.mReceiver != null) {
            boolean handled = (q.mFlags & QueuedInputEvent.FLAG_FINISHED_HANDLED) != 0;
            q.mReceiver.finishInputEvent(q.mEvent, handled);
        } else {
            q.mEvent.recycleIfNeededAfterDispatch();
        }

        recycleQueuedInputEvent(q);
    }
```
在deliverInputEvent中，调用了InputStage的deliver方法继续处理事件。
在deliverInputEvent函数开始是，开始跟踪事件：
```java
Trace.asyncTraceBegin(Trace.TRACE_TAG_VIEW, "deliverInputEvent",
                q.mEvent.getSequenceNumber());
```
在finishInputEvent中，结束跟踪事件：
```java
Trace.asyncTraceEnd(Trace.TRACE_TAG_VIEW, "deliverInputEvent",
                q.mEvent.getSequenceNumber());

```
此处的“deliverInputEvent”，有什么含义呢？很明显，Trace在此处跟踪的是一个Pointer事件的处理过程。通过systrace抓取数据时，可以看到“deliverInputEvent”的具体耗时。


##### 1.5 InputStage.deliver
```java
 public final void deliver(QueuedInputEvent q) {
            if ((q.mFlags & QueuedInputEvent.FLAG_FINISHED) != 0) {
                forward(q);
            } else if (shouldDropInputEvent(q)) {
                finish(q, false);
            } else {
                apply(q, onProcess(q));
            }
        }
```
InputStage采用职责链的模式处理事件，正常情况下，事件处理会执行第7行的分支。InputStage有多个子类，此处onProcess在InputStage的子类ViewPostImeInputStage中实现。
##### 1.6 ViewPostImeInputStage.onProcess
```java
 @Override
        protected int onProcess(QueuedInputEvent q) {
            if (q.mEvent instanceof KeyEvent) {
                return processKeyEvent(q);
            } else {
                // If delivering a new non-key event, make sure the window is
                // now allowed to start updating.
                handleDispatchWindowAnimationStopped();
                final int source = q.mEvent.getSource();
                if ((source & InputDevice.SOURCE_CLASS_POINTER) != 0) {
                    return processPointerEvent(q);
                } else if ((source & InputDevice.SOURCE_CLASS_TRACKBALL) != 0) {
                    return processTrackballEvent(q);
                } else {
                    return processGenericMotionEvent(q);
                }
            }
        }
```
首先检查是不是一个按键事件，如果是，则执行第4行；若不不是，需要继续判断事件类型。如果是屏幕点击，执行第11行，轨迹球事件则执行第13行。接下来分析第11行的函数。

##### 1.7 ViewPostImeInputStage.processPointerEvent
```java
private int processPointerEvent(QueuedInputEvent q) {
            final MotionEvent event = (MotionEvent)q.mEvent;

            mAttachInfo.mUnbufferedDispatchRequested = false;
            boolean handled = mView.dispatchPointerEvent(event);
            if (mAttachInfo.mUnbufferedDispatchRequested && !mUnbufferedInputDispatch) {
                mUnbufferedInputDispatch = true;
                if (mConsumeBatchedInputScheduled) {
                    scheduleConsumeBatchedInputImmediately();
                }
            }
            return handled ? FINISH_HANDLED : FORWARD;
        }
```
ViewPostImeInputStage是ViewRootImpl的内部类，在第5行，mView的真实类型是DecorView。DecorView继承自FrameLayout，第5行调用的就是View中定义的dispatchPointerEvent方法。
##### 1.8 View.dispatchPointerEvent
```java
 public final boolean dispatchPointerEvent(MotionEvent event) {
        if (event.isTouchEvent()) {
            return dispatchTouchEvent(event);
        } else {
            return dispatchGenericMotionEvent(event);
        }
    }
```
很明显，此时是Touch事件，进入第3行的分支。由于DecorView重写了dispatchTouchEvent方法，接下来分支DecorView中该函数的实现。

##### 1.9 DecorView.dispatchTouchEvent
```java
 @Override
        public boolean dispatchTouchEvent(MotionEvent ev) {
            final Callback cb = getCallback();
            return cb != null && !isDestroyed() && mFeatureId < 0 ? cb.dispatchTouchEvent(ev)
                    : super.dispatchTouchEvent(ev);
        }
```
DecorView是PhoneWindow的内部类，PhoneWindow的是Window的子类。Activity实现了Window中的接口Callback,在Activity的attach函数中，给Window对象设置了Callback：mWindow.setCallback(this)，因此，第4行的cb.dispatchTouchEvent就跳转到了Activity中，事件传递到了Activity。

#### Step2. 从DecorView到每一个View
在Step1中，我们看到了一个Touch事件是如何由ViewRootImpl传递到Activity中的，现在来分析事件是如何从Activity传递到Activity中的每个View中的。
![从DecorView到每一个View](img/blogs/viewroot/v2.png)

#####2.1 Activity.dispatchTouchEvent
```java
 public boolean dispatchTouchEvent(MotionEvent ev) {
        if (ev.getAction() == MotionEvent.ACTION_DOWN) {
            onUserInteraction();
        }
        if (getWindow().superDispatchTouchEvent(ev)) {
            return true;
        }
        return onTouchEvent(ev);
    }
```
Activity收到事件后，调用了PhoneWindow中的superDispatchTouchEvent方法。

##### 2.2 PhonWindow.superDispatchTouchEvent
```java
   @Override
    public boolean superDispatchTouchEvent(MotionEvent event) {
        return mDecor.superDispatchTouchEvent(event);
    }
```
调用DecorView的superDispatchTouchEvent方法。

#####2.3 DecorView.superDispatchTouchEvent
```java
   public boolean superDispatchTouchEvent(MotionEvent event) {
            return super.dispatchTouchEvent(event);
        }
```
DecorView继承自FrameLayout，是Activity中所有界面元素的根布局，此时，事件传递到了所有布局元素中最外层的FrameLayout中，接下来通过FrameLayout的dispatchTouchEvent方法，将事件一层层的传递下去。

**`总结：`**
事件传递的整体流程如下图：
![事件传递整体流程](img/blogs/viewroot/v3.png)


1. ViewRootImpl首先接收到事件，然后交给DecorView处理
2. DecorView接收到事件后，没有直接处理，而是调用了Activity实现的Callback。为什么要这么设计？理论上来说，DecorView可以直接处理事件。如果DecorView直接处理了事件，这样用户就无法拦截到初始事件。某些情况下，可能会重新Activity中的dispatchTouchEvent事件来实现某些功能。
3. 事件传递到Activity，是为了给用户一个拦截、处理事件的机会。在Activity的dispatchTouchEvent中，事件又传递到了PhoneWindow中，PhoneWindow又将事件传递到DecorView中，即最终交由DecorView处理。