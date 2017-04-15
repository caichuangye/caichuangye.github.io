---
title:  Activity setContentView过程分析
date: 2017-02-25 19:32:12
tags:
- Android
categories: Android
---

#### `1.Activity.setContentView `
```java
	public void setContentView(int layoutResID) {  
	       getWindow().setContentView(layoutResID);  
	       initActionBar();  
	} 
```

Activity、Window、PhoneWindow、DecorView关系如下图：

![Activity、Window、DecorView关系](/assets/img/blogs/setcontentview/window.png)


#### `2.PhoneWindow.setContentView `
```java
  @Override
    public void setContentView(View view, ViewGroup.LayoutParams params) {
        if (mContentParent == null) {
            installDecor();
        } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            mContentParent.removeAllViews();
        }
        if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            view.setLayoutParams(params);
            final Scene newScene = new Scene(mContentParent, view);
            transitionTo(newScene);
        } else {
            mContentParent.addView(view, params);
        }
        mContentParent.requestApplyInsets();
        final Callback cb = getCallback();
        if (cb != null && !isDestroyed()) {
            cb.onContentChanged();
        }
    }

```

1. 判断mContentParent 是否为空，为空则执行installDecor函数来完成初始化，mContentParent 就是承载用户定义的布局的父布局；若mContentParent 不为空，则要判断是否设置了FEATURE_CONTENT_TRANSITIONS，没有设置的话就把mContentParent 中中之前存在的view全部移除。
2. 如果设置了FEATURE_CONTENT_TRANSITIONS，则新的view不是不是直接加到mContentParent 中，而是才采用过渡效果，该过程是通过transitionTo函数实现的；否则直接将view add 到mContentParent 中。
3. 执行onContentChanged的回调，onContentChanged是在Activity中实现的，默认为空实现。
以上过程逻辑比较清晰，最重要的一步就是installDecor的过程，下面重点分析。 

#### `3.PhoneWindow.installDecor`
installDecor中代码较多，现在主要分析初始化mDecor和mContentParent的过程：
* mDecor的初始化通过函数generateDecor完成，
* mContentParent 的初始化通过generateLayout完成。
```java
 private void installDecor() {
        if (mDecor == null) {
            mDecor = generateDecor();
            mDecor.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
            mDecor.setIsRootNamespace(true);
            if (!mInvalidatePanelMenuPosted && mInvalidatePanelMenuFeatures != 0) {
                mDecor.postOnAnimation(mInvalidatePanelMenuRunnable);
            }
        }
        if (mContentParent == null) {
            mContentParent = generateLayout(mDecor);
            ...
        }
        ...
 }
```

#### `4.PhoneWindow.generateDecor`
mDecor的初始化过程如下：
```java
  protected DecorView generateDecor() {
        return new DecorView(getContext(), -1);
    }
```


#### `5.PhoneWindow.generateLayout`
```java
  protected ViewGroup generateLayout(DecorView decor) {
		...
        int layoutResource;
        int features = getLocalFeatures();
        ...
        View in = mLayoutInflater.inflate(layoutResource, null);
        decor.addView(in, new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));
        mContentRoot = (ViewGroup) in;

        ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
        ...
        return contentParent;
    }
```
在generateLayout中，layoutResource代表DecorView中的根布局的资源id，具体的根布局样式取决于主题样式，这些布局文件所在的目录为**`frameworks\base\core\res\res\layout`**，下面以screen_title.xml为例来分析：
```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="vertical"
    android:fitsSystemWindows="true">
    <!-- Popout bar for action modes -->
    <ViewStub android:id="@+id/action_mode_bar_stub"
              android:inflatedId="@+id/action_mode_bar"
              android:layout="@layout/action_mode_bar"
              android:layout_width="match_parent"
              android:layout_height="wrap_content"
              android:theme="?attr/actionBarTheme" />
    <FrameLayout
        android:layout_width="match_parent" 
        android:layout_height="?android:attr/windowTitleSize"
        style="?android:attr/windowTitleBackgroundStyle">
        <TextView android:id="@android:id/title" 
            style="?android:attr/windowTitleStyle"
            android:background="@null"
            android:fadingEdge="horizontal"
            android:gravity="center_vertical"
            android:layout_width="match_parent"
            android:layout_height="match_parent" />
    </FrameLayout>
    <FrameLayout android:id="@android:id/content"
        android:layout_width="match_parent" 
        android:layout_height="0dip"
        android:layout_weight="1"
        android:foregroundGravity="fill_horizontal|top"
        android:foreground="?android:attr/windowContentOverlay" />
</LinearLayout>
```
从布局文件中可知，其根布局为LinearLayout，其中ViewStub先不管，screen_title.xml中主要有两个FrameLayout，第一个用来显示标题，第二个用来显示界面内容（id为"@android:id/content"），mContentParent在screen_title.xml中对应的就是id为“content”的FrameLayout。

下面这个根布局名为screen_simple.xml
```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:fitsSystemWindows="true"
    android:orientation="vertical">
    <ViewStub android:id="@+id/action_mode_bar_stub"
              android:inflatedId="@+id/action_mode_bar"
              android:layout="@layout/action_mode_bar"
              android:layout_width="match_parent"
              android:layout_height="wrap_content"
              android:theme="?attr/actionBarTheme" />
    <FrameLayout
         android:id="@android:id/content"
         android:layout_width="match_parent"
         android:layout_height="match_parent"
         android:foregroundInsidePadding="false"
         android:foregroundGravity="fill_horizontal|top"
         android:foreground="?android:attr/windowContentOverlay" />
</LinearLayout>
```
实际上DecorView可能的根布局有多个，但基本上根布局的文件的根布局都是是LinearLayout，根布局文件有以下特点：
* 根布局文件的根节点一般都是LinearLayout
* 根布局文件中具体的布局样式不固定，但一定有一个id为“content”的FrameLayout，该FrameLayout用于承载用户添加的view


在generateLayout的第6、7、8行：
```java
 View in = mLayoutInflater.inflate(layoutResource, null);
 decor.addView(in, new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));
 mContentRoot = (ViewGroup) in;
```
首先加载DecorView的根布局，然后添加到DecorView中，其布局参数中的宽和高都是MATCH_PARENT。由于是根布局，宽和高应设置为MATCH_PARENT。最后，把根布局保存到变量mContentRoot 中。

第10行，找出id为android.R.id.content的ViewGroup。PhoneWindow中没有定义findViewById方法，在其基类Window中：
```java
 @Nullable
 public View findViewById(@IdRes int id) {
    return getDecorView().findViewById(id);
 }
```
可知，findViewById是在PhoneWindow的内部类DecorView中定义的。查看DecorView的代码，没有发现定义函数findViewById。但DecorView的基类为FrameLayout，由此可知，第10行就是在DecorView中查找资源id为android.R.id.content的ViewGroup。

回到本文step2中第13行：
```java
	mContentParent.addView(view, params);
```
**`可知，在Activity的onCreate中通过setContentView添加布局，最终添加到了DecorView中id为android.R.id.content的ViewGroup中。`**

**`总结：`**
1. DecorView继承自FrameLayout
2. DecorView的直接子View为LinearLayout，具体的样式取决了当前设置的主题样式
3. DecorView的直接子View中一定存在一个id为‘content’的FrameLayout，也可能存在其他的FrameLayout（比如用来显示标题）
4. 通过setContentView方法中添加的View，最终被添加到了DecorView的子View（LinearLayout）的子View（id为‘content’的FrameLayout）中
![DecorView布局层次](/assets/img/blogs/setcontentview/DecorView.png)
