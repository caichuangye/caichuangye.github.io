---
title:  Android中View的测量measure过程
date: 2017-02-12 20:46:26
tags:
- View
- Android
categories: Android
---

本文中，以自定义ViewGroup中重写onMeasure入口，来分析View的测量过程：


#### 1.自定义ViewGroup中重写onMeasure
```java

    /**
	 * 计算所有ChildView的宽度和高度 然后根据ChildView的计算结果，设置自己的宽和高
	 */
	@Override
	protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec)
	{
		/**
		 * 获得此ViewGroup上级容器为其推荐的宽和高，以及计算模式
		 */
		int widthMode = MeasureSpec.getMode(widthMeasureSpec);
		int heightMode = MeasureSpec.getMode(heightMeasureSpec);
		int sizeWidth = MeasureSpec.getSize(widthMeasureSpec);
		int sizeHeight = MeasureSpec.getSize(heightMeasureSpec);


		// 计算出所有的childView的宽和高
		measureChildren(widthMeasureSpec, heightMeasureSpec);
		/**
		 * 记录如果是wrap_content是设置的宽和高
		 */
		int width = 0;
		int height = 0;

		...
		//计算在wrap_content模式下的宽和高
		...
		
		/**
		 * 如果是wrap_content设置为我们计算的值
		 * 否则：直接设置为父容器计算的值
		 */
		setMeasuredDimension((widthMode == MeasureSpec.EXACTLY) ? sizeWidth
				: width, (heightMode == MeasureSpec.EXACTLY) ? sizeHeight
				: height);
	}
```
一个ViewGroup的典型onMeasure的过程如上所示。如果其测量模式为wrap_content，则需要重新计算其测量尺寸，否则直接使用父布局提供的测量尺寸。在onMeasure的最后，需要调用setMeasuredDimension保存测量结果，该处保存的测量结果也就是getMeasuredWidth、getMeasuredHeight获取到的值（严格来说，以上描述是不严谨的，因为在setMeasuredDimension中可能会修改传入的参数，测量值最终通过setMeasuredDimensionRaw来保存）。

在上面的示例代码中，ViewGroup需要根据其子View的大小来确定自身的大小，子View是通过measureChildren来测量的，接下来以此为入口，分析ViewGroup是如何测量其子View大小并如何计算自身测量大小的。


---

#### 2. measureChildren
```java
  protected void measureChildren(int widthMeasureSpec, int heightMeasureSpec) {
        final int size = mChildrenCount;
        final View[] children = mChildren;
        for (int i = 0; i < size; ++i) {
            final View child = children[i];
            if ((child.mViewFlags & VISIBILITY_MASK) != GONE) {
                measureChild(child, widthMeasureSpec, heightMeasureSpec);
            }
        }
    }
```
获取ViewGroup中所有的子View，依次调用measureChild测量每一个子View。注意，状态为GONE的View不会被测量。

---

#### 3. measureChild
```java
  protected void measureChild(View child, int parentWidthMeasureSpec,
            int parentHeightMeasureSpec) {
        final LayoutParams lp = child.getLayoutParams();

        final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
                mPaddingLeft + mPaddingRight, lp.width);
        final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
                mPaddingTop + mPaddingBottom, lp.height);

        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    }
```
measureChild中的第二个和第三个参数是ViewGroup的测量参数，我们需要通过ViewGroup的测量参数结合View自身的测量参数，来最终决定View的测量参数。
首先调用getChildMeasureSpec方法获取到View的WidthMeasureSpec和HeightMeasureSpec，即获取到View测量到的尺寸和显示Mode，然后再调用View的measure方法。 
getChildMeasureSpec方法非常关键！正是通过这个函数，View才根据其自身参数和父View的参数，计算出了自己的测量值。

---

3.1 getChildMeasureSpec
```java
/**
* spec: ViewGroup的测量参数
* padding：ViewGroup的内边距，是同一方向上的两个内边距之和
* childDimension： View的宽或者高
*/
public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
		// 计算出ViewGroup的测量模式和测量值
        int specMode = MeasureSpec.getMode(spec);
        int specSize = MeasureSpec.getSize(spec);

		// ViewGroup除去padding之外的尺寸
        int size = Math.max(0, specSize - padding);
		
		// view的尺寸
        int resultSize = 0;

		//view的测量模式
        int resultMode = 0;

        switch (specMode) {
        // ViewGroup的测量模式为EXACTLY，即尺寸为具体的指定的数值
        case MeasureSpec.EXACTLY:
            if (childDimension >= 0) {//View在布局参数中指定了尺寸，则使用布局参数中的尺寸，测量模式为EXACTLY
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                //view的尺寸为MATCH_PARENT，即跟其所在的ViewGroup一样大，则设置View的size为ViewGroup的size
                resultSize = size;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                //view的尺寸为WRAP_CONTENT，但此时还未测量View内容具体由多大，则暂时设置View的size为ViewGroup的size
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;

        // ViewGroup的sizeWRAP_CONTENT，即取决于子View的大小
        case MeasureSpec.AT_MOST:
            if (childDimension >= 0) {
                // Child wants a specific size... so be it
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size, but our size is not fixed.
                // Constrain child to not be bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size. It can't be
                // bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;

        // Parent asked to see how big we want to be
        case MeasureSpec.UNSPECIFIED:
            if (childDimension >= 0) {
                // Child wants a specific size... let him have it
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size... find out how big it should
                // be
                resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                resultMode = MeasureSpec.UNSPECIFIED;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size.... find out how
                // big it should be
                resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                resultMode = MeasureSpec.UNSPECIFIED;
            }
            break;
        }
        //noinspection ResourceType
        return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
    }
```
在getChildMeasureSpec中，会根据ViewGroup的剩余空间、测量模式以及子View的尺寸或者测量模式来最终决定子View的measureSpec。
1.  父布局的测量模式为EXACTLY
	*   view的宽或者高设置了具体的尺寸，则宽或者高为自身设置的值，测量模式为EXACTLY
	*   view的宽或者高设置为MATCH_PARENT，则宽或者高为父布局的尺寸，测量模式为EXACTLY
	*   `view的宽或者高设置为WRAP_CONTENT，则宽或者高为父布局的尺寸，测量模式为AT_MOST`
	
2.  父布局的测量模式为AT_MOST
    *   view的宽或者高设置了具体的尺寸，则宽或者高为自身设置的值，测量模式为EXACTLY
	*   view的宽或者高设置为MATCH_PARENT，则宽或者高为父布局的尺寸，测量模式为AT_MOST
	*   view的宽或者高设置为WRAP_CONTENT，则宽或者高为父布局的尺寸，测量模式为AT_MOST
	
3.  父布局的测量模式为UNSPECIFIED
*   view的宽或者高设置了具体的尺寸，则宽或者高为自身设置的值，测量模式为EXACTLY
	*   view的宽或者高设置为MATCH_PARENT，则宽或者高为父布局的尺寸，测量模式为UNSPECIFIED
	*   view的宽或者高设置为WRAP_CONTENT，则宽或者高为父布局的尺寸，测量模式为UNSPECIFIED

---

#### 4. View.measure

```java
 public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
        boolean optical = isLayoutModeOptical(this);
        //处理可能的边框
        if (optical != isLayoutModeOptical(mParent)) {
            Insets insets = getOpticalInsets();
            int oWidth  = insets.left + insets.right;
            int oHeight = insets.top  + insets.bottom;
            widthMeasureSpec  = MeasureSpec.adjust(widthMeasureSpec,  optical ? -oWidth  : oWidth);
            heightMeasureSpec = MeasureSpec.adjust(heightMeasureSpec, optical ? -oHeight : oHeight);
        }

        // Suppress sign extension for the low bytes
        long key = (long) widthMeasureSpec << 32 | (long) heightMeasureSpec & 0xffffffffL;
        if (mMeasureCache == null) mMeasureCache = new LongSparseLongArray(2);

		//判断是否要求强制重新布局
        final boolean forceLayout = (mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT;

		// 宽和高在的测试模式为EXACTLY并且尺寸没有变化的情况下，不需要再次测量，可以直接父布局提供的参数
        // Optimize layout by avoiding an extra EXACTLY pass when the view is
        // already measured as the correct size. In API 23 and below, this
        // extra pass is required to make LinearLayout re-distribute weight.
        // specsize是否有变化
        final boolean specChanged = widthMeasureSpec != mOldWidthMeasureSpec
                || heightMeasureSpec != mOldHeightMeasureSpec;
        //宽和高是的测量模式是否都是EXACTLY
        final boolean isSpecExactly = MeasureSpec.getMode(widthMeasureSpec) == MeasureSpec.EXACTLY
                && MeasureSpec.getMode(heightMeasureSpec) == MeasureSpec.EXACTLY;
        //测量后的宽和高的尺寸是否都没有变化
        final boolean matchesSpecSize = getMeasuredWidth() == MeasureSpec.getSize(widthMeasureSpec)
                && getMeasuredHeight() == MeasureSpec.getSize(heightMeasureSpec);

		/**
		* 只有满足以下条件才需要重新测量
		* 1. spec变化了，即本次父布局提供的布局参数与上次保存的布局参数不相同
		* 2. 系统要求Exactly每次都需要重新 or 宽或者高的测量模式不都是EXACTLY or 测量后的宽和高与本次不同
		*/
        final boolean needsLayout = specChanged
                && (sAlwaysRemeasureExactly || !isSpecExactly || !matchesSpecSize);

        if (forceLayout || needsLayout) {
            // 清除已测量标志
            mPrivateFlags &= ~PFLAG_MEASURED_DIMENSION_SET;

            resolveRtlPropertiesIfNeeded();
			
			// 获取缓存索引
            int cacheIndex = forceLayout ? -1 : mMeasureCache.indexOfKey(key);
            // 没有缓存 或者需要忽略缓存，执行onMeaure进行真正的测量
            if (cacheIndex < 0 || sIgnoreMeasureCache) {
                // measure ourselves, this should set the measured dimension flag back
                onMeasure(widthMeasureSpec, heightMeasureSpec);
                mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
            } else {//使用缓存的测量结果，并调用setMeasuredDimensionRaw保存测量结果
                long value = mMeasureCache.valueAt(cacheIndex);
                // Casting a long to int drops the high 32 bits, no mask needed
                setMeasuredDimensionRaw((int) (value >> 32), (int) value);
                //设置PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT，在layout函数中会再进行measure，即延迟测量
                mPrivateFlags3 |= PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
            }

            // onMeausre执行完，必须调用setMeasuredDimension保存测量结果，否则会抛出异常
            if ((mPrivateFlags & PFLAG_MEASURED_DIMENSION_SET) != PFLAG_MEASURED_DIMENSION_SET) {
                throw new IllegalStateException("View with id " + getId() + ": "
                        + getClass().getName() + "#onMeasure() did not set the"
                        + " measured dimension by calling"
                        + " setMeasuredDimension()");
            }
		
            mPrivateFlags |= PFLAG_LAYOUT_REQUIRED;
        }

		/**
		* 保存父布局提供的测量参数
		* mOldWidthMeasureSpec和mOldHeightMeasureSpec存在有两个用处：
		*   1. 执行measure时判断当前的测量参数是否与上一次的相同，可用来避免不必要的测量
		*   2. layout时，如果需要measure，可使用这两个参数执行onMeasure
		*/
        mOldWidthMeasureSpec = widthMeasureSpec;
        mOldHeightMeasureSpec = heightMeasureSpec;

		// 缓存测量结果
        mMeasureCache.put(key, ((long) mMeasuredWidth) << 32 |
                (long) mMeasuredHeight & 0xffffffffL); // suppress sign extension
    }
```

View的measure中主要做了以下逻辑：
1. 判断当前是否需要重新测量
2. 如果需要重新测量，则调用onMeasure进行测量；如果不需要测量，则使用之前的缓存值
3. 保存传入的测量参数，并且把测量结果缓存起来

---

#### 5. View中默认的onMeausure实现
```java
 protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
                getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
    }
```
先调用getDefaultSize获取View的默认尺寸，再调用setMeasuredDimension保存测量结果。接下来分析getDefaultSize的逻辑，代码如下：

```java
 public static int getDefaultSize(int size, int measureSpec) {
        int result = size;
        int specMode = MeasureSpec.getMode(measureSpec);
        int specSize = MeasureSpec.getSize(measureSpec);

        switch (specMode) {
        case MeasureSpec.UNSPECIFIED:
            result = size;
            break;
        case MeasureSpec.AT_MOST:
        case MeasureSpec.EXACTLY:
            result = specSize;
            break;
        }
        return result;
    }
```
从以上代码可知，getDefaultSize主要就是为了处理测量模式为UNSPECIFIED情景，在这种情况下，View的size为设置为建议的最小尺寸。

---

#### 6. setMeasuredDimension
一般来说，当我们自定义View时，如果View的宽或者高设置为wrap_content，我们就需要重写onMeasure来计算一个View或ViewGroup到底应该为多大，如果不重写，则默认为父布局的大小。重新onMeasure时，需要调用setMeasuredDimension来保存测量结果，代码如下：

```java
protected final void setMeasuredDimension(int measuredWidth, int measuredHeight) {
        boolean optical = isLayoutModeOptical(this);
        if (optical != isLayoutModeOptical(mParent)) {
            Insets insets = getOpticalInsets();
            int opticalWidth  = insets.left + insets.right;
            int opticalHeight = insets.top  + insets.bottom;

            measuredWidth  += optical ? opticalWidth  : -opticalWidth;
            measuredHeight += optical ? opticalHeight : -opticalHeight;
        }
        setMeasuredDimensionRaw(measuredWidth, measuredHeight);
    }
```
setMeasuredDimension中需要处理可能出现的边框，最终调用setMeasuredDimensionRaw保存最终的测量结果，并且设置了已测量的标志。

```java
 private void setMeasuredDimensionRaw(int measuredWidth, int measuredHeight) {
        mMeasuredWidth = measuredWidth;
        mMeasuredHeight = measuredHeight;

        mPrivateFlags |= PFLAG_MEASURED_DIMENSION_SET;
    }
```

---

#### 7. 总结

现在，我们回到本文最开始的自定义ViewGroup中的onMeasure方法，在onMeasure中，通过measureChildren计算了所有的子View的大小（假设子View也重新了onMeasure并正确计算了大小）。若我们自定义的ViewGroup的宽或者高为WRAP_CONTENT，我们就可以根据其子View的大小来计算当前自定义的ViewGroup的宽或者高的大小，并且通过setMeasuredDimension保存测量好的结果。

上面的示例是自定义ViewGroup，我们根据其子View的大小来确定ViewGroup的大小；若我们自定义的是View，比如定义一个View，用来显示字符串，此时又应该如果重新onMeasure呢？
如果我们自定的View的宽或者高设置的为WRAP_CONTENT，我们可以通过Paint和Canvas根据要显示的字符串的字体，计算出字符串的大小，然后将View的宽或者高设置为字符串的大小，此时字符串就可以填充整个View了。

---

`本文最后，讨论一个问题：getWidth与getMeasuredWidth有什么区别呢？`

```java
public final int getWidth() {
    return mRight - mLeft;
}

public final int getMeasuredWidth() {
    return mMeasuredWidth & MEASURED_SIZE_MASK;
}
```


* getWidth：获取的是View的真实的显示宽度
* getMeasuredWidth：获取的是测量宽度。从上文的分析中可知，当View的测量模式为EXACTLY时，其测量宽度和真实宽度是相同的；当测量模式为ATMOST（即wrap_content）时，若不重写onMeasure方法，则getMeasuredWidth获取到的就是其父布局的宽度，若重写了onMeasure，在onMeasure的最后，需要调用setMeasuredDimension保存测量结果，此时保存的测量结果就是我们再onMeasure中重新计算的宽度，也就是真实的宽度。