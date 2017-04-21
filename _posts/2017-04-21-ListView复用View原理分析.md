---
title: ListView复用View原理分析
date: 2017-04-21 23:03:56
tags:
- ListView
- Android
categories: Android
---

ListView继承自ViewGroup，在onLayout时，需要获取View并且将View放置到制定位置，本文以ListView的onLayout方法为入口，来分析ListView显示和复用View过程。

#### 1. onLayout中调用layoutChildren显示子View
onLayout是在AbsListView中实现的，在onLayout中调用了layoutChildren来显示adapter中的数据。
```java
Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        super.onLayout(changed, l, t, r, b);

        mInLayout = true;

        final int childCount = getChildCount();
        if (changed) {
            for (int i = 0; i < childCount; i++) {
                getChildAt(i).forceLayout();
            }
            mRecycler.markChildrenDirty();
        }

        layoutChildren();
        mInLayout = false;

        mOverscrollMax = (b - t) / OVERSCROLL_LIMIT_DIVISOR;

        // TODO: Move somewhere sane. This doesn't belong in onLayout().
        if (mFastScroll != null) {
            mFastScroll.onItemCountChanged(getChildCount(), mItemCount);
        }
    }
```

#### 2. layoutChildren
```java
	 @Override
    protected void layoutChildren() {
	    .................
            // Pull all children into the RecycleBin.
            // These views will be reused if possible
            final int firstPosition = mFirstPosition;
            final RecycleBin recycleBin = mRecycler;
            if (dataChanged) {
                for (int i = 0; i < childCount; i++) {
                    recycleBin.addScrapView(getChildAt(i), firstPosition+i);
                }
            } else {
                recycleBin.fillActiveViews(childCount, firstPosition);
            }

            // Clear out old views
            detachAllViewsFromParent();
            recycleBin.removeSkippedScrap();

            switch (mLayoutMode) {
            ......
            case LAYOUT_FORCE_TOP:
                mFirstPosition = 0;
                sel = fillFromTop(childrenTop);
                adjustViewsUpOrDown();
                break;
           ......
            }
      ....................      
      }
```
layoutChildren中代码较长，现分析layoutChildren中最主要的逻辑：
1. 如果Adapter中的数据集有变化，则将ListView的所有的View都放到RecycleBin的废弃View集合中；若数据无变化，则将ListView的所有的View放入RecycleBin的激活的View集合中。
2. 调用detachAllViewsFromParent解除View与ListView之间的关联。
3. 重新将View显示到ListView中。

以上的第一和第二步比较简单，现在分析如何将RecycleBin中的View重新绑定到ListView中,下面以fillFromTop为例来展开分析：

#### 3. fillFromTop
```java
private View fillFromTop(int nextTop) {
        mFirstPosition = Math.min(mFirstPosition, mSelectedPosition);
        mFirstPosition = Math.min(mFirstPosition, mItemCount - 1);
        if (mFirstPosition < 0) {
            mFirstPosition = 0;
        }
        return fillDown(mFirstPosition, nextTop);
    }
```
在fillFromTop中计算了要绘制的第一个View的索引，然后调用了fillDown。fillDown有两个参数：
* pos 第一个要绘制的view在adapter中的索引
* nextTop 第一个要绘制的view顶部相对于ListView顶部的距离

#### 4. fillDown
```java
 private View fillDown(int pos, int nextTop) {
		 // 被选中的view
        View selectedView = null;
		
		//end就是ListVew的高度
        int end = (mBottom - mTop);
        if ((mGroupFlags & CLIP_TO_PADDING_MASK) == CLIP_TO_PADDING_MASK) {
            end -= mListPadding.bottom;
        }
		
		//nextTop < end 的目的是保证view铺满ListView的可见高度就可以
		//pos < mItemCount的目的是保证显示的view数量不大于数据的数目
        while (nextTop < end && pos < mItemCount) {
            // is this the selected item?
            boolean selected = pos == mSelectedPosition;
            // 根据pos生成view，并且加入到listview中
            View child = makeAndAddView(pos, nextTop, true, mListPadding.left, selected);
			// 计算下一个要显示的view距离Listview的top距离
            nextTop = child.getBottom() + mDividerHeight;
            if (selected) {
                selectedView = child;
            }
            pos++;
        }

        setVisibleRangeHint(mFirstPosition, mFirstPosition + getChildCount() - 1);
        //返回被选中的view
        return selectedView;
    }
```
fillDown的作用是从上往下，将View铺满当前ListView的可见区域，并且返回被选中的View。fillDown中最核心的函数就是makeAndAddView，
从函数名知道该函数产生一个view，并且添加到了listview。

#### 5. makeAndAddView
```java
private View makeAndAddView(int position, int y, boolean flow, int childrenLeft,
            boolean selected) {
        View child;

		// 当调用BaseAdapter的notifyDataSetChanged方法时，mDataChanged会被置为true
        if (!mDataChanged) {
            // 数据没有变化，直接从RecycleBin的Active的集合的指定位置取出View
            child = mRecycler.getActiveView(position);
            if (child != null) {
                // Found it -- we're using an existing child
                // 将view放置到指定位置 
                setupChild(child, position, y, flow, childrenLeft, selected, true);
                return child;
            }
        }

        // Make a new view for this position, or convert an unused view if possible
        child = obtainView(position, mIsScrap);

        // This needs to be positioned and measured
        setupChild(child, position, y, flow, childrenLeft, selected, mIsScrap[0]);

        return child;
    }
```

在分析makeAndAddView之前，先简要介绍下ABSListView中的一个非常重要的内部类**`RecycleBin`**：
>RecycleBin用于复用ABSListView中的View，RecycleBin存储了两个不同层次的View：活动的View和废弃的View。活动的View是指那些当前正在显示在屏幕上的View，
废弃的view是指已经不再显示的View，它们可以被重用，以避免不必要的inflate。



在makeAndAddView中，有两个分支：
1. 若数据没有变化并且RecycleBin中可直接使用的View，则直接使用该View，然后调用setupChild将View放到指定位置。
`需要注意到：此时调用setupChild的最后一个参数为true，表明这个View时被循环利用的，在显示时不要再次measure和layout。`
2. 否则，先调用obtainView获取一个View，再调用setupChild。`需要注意到：此时调用setupChild的最后一个参数为取决于obtainView执行的结果：
如果Adapter的getView使用了传入的convertView，则此时为true，否则为false。`

setupChild中title为“obtainView” Trace标记，setupChild中有title为setupListItem的Trace标记，这两张情况的systrace分析分别如下所示：

1. 在layout中直接复用RecycleBin中的Active View
![直接复用RecycleBin中的Active View](/assets/img/blogs/listview/onlysetuplistitem.JPG)

2. 先调用obtainView，再调用setupListView
![先调用obtainView，再调用setupListView](/assets/img/blogs/listview/layout.JPG)
一般来说，这种情况出现在ListView第一次加载数据或数据源发生变化时，此时一般都会导致较为严重的掉帧问题：obtainView中会调用Adapter的getView方法，
若getView中调用了inflate方法，就可能会出现掉帧问题。

接下来分析上面涉及到的两个非常重要的函数：obtainView和setupChild。

#### 6. obtainView
obtainView源码如下，obtainView的调用时机：当RecycleBin中没有可以直接使用的View，这时需要调用obtainView来convert一个已废弃的View或者inflate一个新的View。

```java
 View obtainView(int position, boolean[] isScrap) {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "obtainView");

        isScrap[0] = false;

        ...
        
        final View scrapView = mRecycler.getScrapView(position);
        final View child = mAdapter.getView(position, scrapView, this);
        if (scrapView != null) {
            if (child != scrapView) {
                // Failed to re-bind the data, return scrap to the heap.
                mRecycler.addScrapView(scrapView, position);
            } else {
                isScrap[0] = true;

                // Finish the temporary detach started in addScrapView().
                child.dispatchFinishTemporaryDetach();
            }
        }

        ....
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);

        return child;
    }
```

obtainView的核心代码如上所示，obatinView中添加了类型为View，title为“obtainView”的Trace标记，在systrace中可观察到。


需要注意到obtainView的第二个参数isScrap：这个参数用于表示从obtainView中获取的view是否是一个被复用的View：
* 首先，不管isScap的初始值为什么，isScrap[0] 都会被置为false
* 根据要显示的位置从RecycleBin的废弃View集合中取出一个View，作为BaseAdapter的getView的参数传入
* 判断getView返回的View是否就是getView中传入的convertView，如果是，isScap[0]被置为true，此时获取到的View就是一个被复用的View，
显示时需要被measure和layout；否则需要measure和layout。



由上面的分析可见，obtainView会调用Adpater的getView方法，systrace中obtainView这个标签显示的事件主要就是耗费在getView方法中。

RecycleBin中getScrapView方法如下所示：

```java
View getScrapView(int position) {
            final int whichScrap = mAdapter.getItemViewType(position);
            if (whichScrap < 0) {
                return null;
            }
            if (mViewTypeCount == 1) {
                return retrieveFromScrap(mCurrentScrap, position);
            } else if (whichScrap < mScrapViews.length) {
                return retrieveFromScrap(mScrapViews[whichScrap], position);
            }
            return null;
        }
```
`一般来说，ListView中View的样式只用一种，即ListView的每一行的View的布局样式都相同，假设：如要要求ListView的单数行与偶数行的View样式不同，
我们又该如何自定义BaseAdapter？`
BaseAdapter中有如下两个方法：
```java
    public int getItemViewType(int position) {
        return 0;
    }

    public int getViewTypeCount() {
        return 1;
    }
```
* getViewTypeCount用于表示当前ListView中有多少不同样式的View，返回值默认是1，即默认情况下，ListView的每一行的View的样式都相同
* getItemViewType用于获取制定位置上View的样式的类型，默认返回值为0

通过以上分析可知，当我们要定义一个有多个Item样式的ListView，需要实现getViewTypeCount和getItemViewType方法。如果不实现，
会导致从RecycleBin获取的复用View与目标View的样式不匹配。


#### 7. setupChild
以上分析了View的获取和数据绑定方法，下面分析最后一个方法：setupChild。setupChild用于将一个View放到合适的位置，并确保该View是被测量过的。
```java
 private void setupChild(View child, int position, int y, boolean flowDown, int childrenLeft,
            boolean selected, boolean recycled) {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "setupListItem");

        final boolean isSelected = selected && shouldShowSelector();
        final boolean updateChildSelected = isSelected != child.isSelected();
        final int mode = mTouchMode;
        final boolean isPressed = mode > TOUCH_MODE_DOWN && mode < TOUCH_MODE_SCROLL &&
                mMotionPosition == position;
        final boolean updateChildPressed = isPressed != child.isPressed();
        final boolean needToMeasure = !recycled || updateChildSelected || child.isLayoutRequested();

        // Respect layout params that are already in the view. Otherwise make some up...
        // noinspection unchecked
        AbsListView.LayoutParams p = (AbsListView.LayoutParams) child.getLayoutParams();
        if (p == null) {
            p = (AbsListView.LayoutParams) generateDefaultLayoutParams();
        }
        p.viewType = mAdapter.getItemViewType(position);

        if ((recycled && !p.forceAdd) || (p.recycledHeaderFooter
                && p.viewType == AdapterView.ITEM_VIEW_TYPE_HEADER_OR_FOOTER)) {
            attachViewToParent(child, flowDown ? -1 : 0, p);
        } else {
            p.forceAdd = false;
            if (p.viewType == AdapterView.ITEM_VIEW_TYPE_HEADER_OR_FOOTER) {
                p.recycledHeaderFooter = true;
            }
            addViewInLayout(child, flowDown ? -1 : 0, p, true);
        }

        if (updateChildSelected) {
            child.setSelected(isSelected);
        }

        if (updateChildPressed) {
            child.setPressed(isPressed);
        }

        if (mChoiceMode != CHOICE_MODE_NONE && mCheckStates != null) {
            if (child instanceof Checkable) {
                ((Checkable) child).setChecked(mCheckStates.get(position));
            } else if (getContext().getApplicationInfo().targetSdkVersion
                    >= android.os.Build.VERSION_CODES.HONEYCOMB) {
                child.setActivated(mCheckStates.get(position));
            }
        }

        if (needToMeasure) {
            final int childWidthSpec = ViewGroup.getChildMeasureSpec(mWidthMeasureSpec,
                    mListPadding.left + mListPadding.right, p.width);
            final int lpHeight = p.height;
            final int childHeightSpec;
            if (lpHeight > 0) {
                childHeightSpec = MeasureSpec.makeMeasureSpec(lpHeight, MeasureSpec.EXACTLY);
            } else {
                childHeightSpec = MeasureSpec.makeSafeMeasureSpec(getMeasuredHeight(),
                        MeasureSpec.UNSPECIFIED);
            }
            child.measure(childWidthSpec, childHeightSpec);
        } else {
            cleanupLayoutState(child);
        }

        final int w = child.getMeasuredWidth();
        final int h = child.getMeasuredHeight();
        final int childTop = flowDown ? y : y - h;

        if (needToMeasure) {
            final int childRight = childrenLeft + w;
            final int childBottom = childTop + h;
            child.layout(childrenLeft, childTop, childRight, childBottom);
        } else {
            child.offsetLeftAndRight(childrenLeft - child.getLeft());
            child.offsetTopAndBottom(childTop - child.getTop());
        }

        if (mCachingStarted && !child.isDrawingCacheEnabled()) {
            child.setDrawingCacheEnabled(true);
        }

        if (recycled && (((AbsListView.LayoutParams)child.getLayoutParams()).scrappedFromPosition)
                != position) {
            child.jumpDrawablesToCurrentState();
        }

        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
```
从obtainView的过程可知，当obtainView返回的是一个被复用的View时，setupChild的最后一个参数recycled为true，表示当前的View是一个被复用的View，
否则就是为false。一个View是否需要被测量由以下表达式决定：

```java
final boolean needToMeasure = !recycled || updateChildSelected || child.isLayoutRequested();
```
可见，若setupChild传入的是一个新创建的View（通过inflate方法，recycled为false），needToMeasure一定为true。

setupChild中主要做了以下：
1. 添加了类型为View、title为“setupListItem”的Trace标记
2. 调用attachViewToParent或addViewInLayout将View放到指定的位置
3. 如果needToMeasure为true，执行View的measure方法
4.  如果needToMeasure为true，执行View的layout方法

**`简单来说，若ListView在显示数据时，没有使用复用的View，会发生以下耗时操作：`**
**`1. 通过inflate方法从布局文件中加载View`**
**`2. 在setupChild时需要对View进行measure`**
**`3.  在setupChild时需要对View进行layout`**
