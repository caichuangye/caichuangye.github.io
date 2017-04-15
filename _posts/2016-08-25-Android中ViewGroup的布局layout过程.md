---
title:  Android中ViewGroup的布局layout过程
date: 2016-08-25 20:26:14
tags:
- View
- Android
categories: Android
---

本文以FrameLayout的layout过程为例，来分析ViewGroup的layout过程

#### 1. FrameLayout的onLayout代码如下：

```java
    protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
        layoutChildren(left, top, right, bottom, false /* no force left gravity */);
    }
```
onLayout中直接调用了layoutChildren函数。

---

#### 2.layoutChildren函数：
```java

	@Override
    void layoutChildren(int left, int top, int right, int bottom, boolean forceLeftGravity) {
	    // 获取子View的数量
        final int count = getChildCount();

		// 获取上下左右的有效显示区域
        final int parentLeft = getPaddingLeftWithForeground();
        final int parentRight = right - left - getPaddingRightWithForeground();

        final int parentTop = getPaddingTopWithForeground();
        final int parentBottom = bottom - top - getPaddingBottomWithForeground();

        for (int i = 0; i < count; i++) {
            final View child = getChildAt(i);
            // 忽略可见状态为Gone的View
            if (child.getVisibility() != GONE) {
                final LayoutParams lp = (LayoutParams) child.getLayoutParams();
				
				// 获取宽、高
                final int width = child.getMeasuredWidth();
                final int height = child.getMeasuredHeight();
				
				// View要布局的相对于父布局的左边距
                int childLeft;
                // View要布局的相对于父布局的上边距
                int childTop;

                int gravity = lp.gravity;
                if (gravity == -1) {
	                // 默认值为 Gravity.TOP | Gravity.START
                    gravity = DEFAULT_CHILD_GRAVITY;
                }
				
				// 获取布局方向，RTL或LTR，希伯来语等少数语言文化中，视图是从右向左显示的（RTL）
                final int layoutDirection = getLayoutDirection();
                // 获取水平方向上的布局参数（水平方向上收到RTL和LTR的影响）
                final int absoluteGravity = Gravity.getAbsoluteGravity(gravity, layoutDirection);
                // 获取垂直方向上的布局参数
                final int verticalGravity = gravity & Gravity.VERTICAL_GRAVITY_MASK;

				// 计算View在父布局中的左边距
                switch (absoluteGravity & Gravity.HORIZONTAL_GRAVITY_MASK) {                  
                    case Gravity.CENTER_HORIZONTAL://水平居中，水平方向上View居中显示
                        childLeft = parentLeft + (parentRight - parentLeft - width) / 2 +
                        lp.leftMargin - lp.rightMargin;
                        break;
                    case Gravity.RIGHT://靠右显示
                        if (!forceLeftGravity) {
                            childLeft = parentRight - width - lp.rightMargin;
                            break;
                        }
                    case Gravity.LEFT://默认是靠左显示
                    default:
                        childLeft = parentLeft + lp.leftMargin;
                }
				
				// 计算View在父布局中的上边距
                switch (verticalGravity) {
                    case Gravity.TOP://靠上对齐
                        childTop = parentTop + lp.topMargin;
                        break;
                    case Gravity.CENTER_VERTICAL://垂直居中，垂直方向上View居中显示
                        childTop = parentTop + (parentBottom - parentTop - height) / 2 +
                        lp.topMargin - lp.bottomMargin;
                        break;
                    case Gravity.BOTTOM://靠底对齐
                        childTop = parentBottom - height - lp.bottomMargin;
                        break;
                    default:// 默认靠上对齐
                        childTop = parentTop + lp.topMargin;
                }
				
				//设置View的上、下、左、右的相对于父布局的坐标
				child.layout(childLeft, childTop, childLeft + width, childTop + height);
            }
        }
    }

```
在layoutChildren中，分别根据子View水平方向 和垂直方向上的布局布局参数，计算出相对于父布局的上下左右坐标，然后调用View的layout方法保存布局参数。

---
#### 3.View中的layout方法：
```java
 public void layout(int l, int t, int r, int b) {
		 // 在View的measure函数中，若使用缓存值，则会通过
		 //mPrivateFlags3 |= PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT设置该标志位，表示在接下来的layout中再进行测量
        if ((mPrivateFlags3 & PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT) != 0) {
            onMeasure(mOldWidthMeasureSpec, mOldHeightMeasureSpec);
            mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
        }

		// 保存上一次的上下左右坐标
        int oldL = mLeft;
        int oldT = mTop;
        int oldB = mBottom;
        int oldR = mRight;

		// 通过setFrame保存上下左右坐标，changed为true表示坐标有变化
        boolean changed = isLayoutModeOptical(mParent) ?
                setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);

		// 如果坐标有变化或者要求重新layout
        if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
	        // 对于View来说，一般不会重新onLayout；对于ViewGroup，需要重写onLayout安排子View的位置
            onLayout(changed, l, t, r, b);
            mPrivateFlags &= ~PFLAG_LAYOUT_REQUIRED;

			// 调用尺寸变化的回调函数
            ListenerInfo li = mListenerInfo;
            if (li != null && li.mOnLayoutChangeListeners != null) {
                ArrayList<OnLayoutChangeListener> listenersCopy =
                        (ArrayList<OnLayoutChangeListener>)li.mOnLayoutChangeListeners.clone();
                int numListeners = listenersCopy.size();
                for (int i = 0; i < numListeners; ++i) {
                    listenersCopy.get(i).onLayoutChange(this, l, t, r, b, oldL, oldT, oldR, oldB);
                }
            }
        }
		
		//取消强制layout的标志
        mPrivateFlags &= ~PFLAG_FORCE_LAYOUT;
        //设置布局完成的标志
        mPrivateFlags3 |= PFLAG3_IS_LAID_OUT;
    }

```

---

##### 3.1在layout中，通过调用setFrame保存上下左右的布局参数，现具体分析：

```java
 protected boolean setFrame(int left, int top, int right, int bottom) {
        boolean changed = false;

        if (DBG) {
            Log.d("View", this + " View.setFrame(" + left + "," + top + ","
                    + right + "," + bottom + ")");
        }
		
		// 只要上下左右有一个位置坐标发生了变化，我们就认为位置发生了变化
        if (mLeft != left || mRight != right || mTop != top || mBottom != bottom) {
            changed = true;

            // Remember our drawn bit
            int drawn = mPrivateFlags & PFLAG_DRAWN;

			// 计算上一次的宽高和当前的宽高
            int oldWidth = mRight - mLeft;
            int oldHeight = mBottom - mTop;
            int newWidth = right - left;
            int newHeight = bottom - top;
            boolean sizeChanged = (newWidth != oldWidth) || (newHeight != oldHeight);

            // 老的坐标位置无效
            invalidate(sizeChanged);
			
			// 保存当前的坐标值，View的getWidth方法返回的就是mRight - mLeft，实际的坐标值最终决定了View的宽和高
            mLeft = left;
            mTop = top;
            mRight = right;
            mBottom = bottom;
            //设置渲染区域
            mRenderNode.setLeftTopRightBottom(mLeft, mTop, mRight, mBottom);

            mPrivateFlags |= PFLAG_HAS_BOUNDS;

			//宽高发生变化
            if (sizeChanged) {
                sizeChange(newWidth, newHeight, oldWidth, oldHeight);
            }

            if ((mViewFlags & VISIBILITY_MASK) == VISIBLE || mGhostView != null) {
                // If we are visible, force the DRAWN bit to on so that
                // this invalidate will go through (at least to our parent).
                // This is because someone may have invalidated this view
                // before this call to setFrame came in, thereby clearing
                // the DRAWN bit.
                mPrivateFlags |= PFLAG_DRAWN;
                invalidate(sizeChanged);
                // parent display list may need to be recreated based on a change in the bounds
                // of any child
                invalidateParentCaches();
            }

            // Reset drawn bit to original value (invalidate turns it off)
            mPrivateFlags |= drawn;

            mBackgroundSizeChanged = true;
            if (mForegroundInfo != null) {
                mForegroundInfo.mBoundsChanged = true;
            }

            notifySubtreeAccessibilityStateChangedIfNeeded();
        }
        return changed;
    }

```

---

#### 4. 至此，ViewGroup的布局过程已完成，简单来说，layout决定了一个ViewGroup中子View的位置摆放以及自身的宽和高。
View的getWidth和getHeight代码如下：
```java

    public final int getWidth() {
        return mRight - mLeft;
    }
    
    public final int getHeight() {
        return mBottom - mTop;
    }
```