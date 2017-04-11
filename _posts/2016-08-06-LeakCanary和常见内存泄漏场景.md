---
title: LeakCanary和常见内存泄漏场景
date: 2016-08-06 23:56:32
tags:
- Memory Leak
- Android
categories: Android
---
#### 一. LeakCanary介绍
##### 1. 介绍
LeakCanary是一个检测内存泄露的开源类库，以可视化的方式 轻松检测内存泄露，并且在出现内存泄漏时及时通知开发者，省去手工分析hprof的过程。

`Github：`[LeakCanary](https://github.com/square/leakcanary)

##### 2. 用法
**Step1**：在app的build.gradle的dependencies节点中添加以下两行
```java
debugCompile 'com.squareup.leakcanary:leakcanary-android:1.3'
releaseCompile 'com.squareup.leakcanary:leakcanary-android-no-op:1.3'
```

**Step2**：在Application中的onCreate方法中注册，并将该Application加入到manifest文件中
```java
public class LeakApplication extends Application {

    @Override
    public void onCreate(){
        super.onCreate();
        LeakCanary.install(this);
    }

}
```

**Step3**：启动应用，等到memory leak发生，当内存泄漏发生时，launcher上生成一个图标（名称为Leaks），点击进去，即可看到完整的内存泄漏的引用路径。`实际测试发现，点击Leaks后，页面上可能出现一段时间的空白，等待一段时间才出现内存泄漏的引用路径。`
![内存泄漏后桌面上产生的图标](/assets/img/blogs/leakcanary/leaks.PNG)


#### 二. 常见泄漏方式
##### 1. 不合理的单例模式、静态Activity、Context等
示例代码：
```java
public class SingleInstance {
    private static SingleInstance sInstance;
    private Context mContext;
    private SingleInstance(Context context){
        mContext = context;
    }
    public static SingleInstance getInstance(Context context){
        if(sInstance == null){
            sInstance = new SingleInstance(context);
        }
        return sInstance;
    }
}
```
```java
public class SecondActivity extends AppCompatActivity {
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_second);
        SingleInstance.getInstance(this);
    }
    
}
```
LeakCanary提示：

![不合理的SingleInstance导致的内存泄漏](/assets/img/blogs/leakcanary/singleInstance1.PNG)


原因分析：
>这种场景非常常见！`当SecondActivity销毁时，SingleInstance仍持有SecondActivity的对象的引用（mContext），导致SecondActivity对象不能释放。`

解决方法：
>`可使用Application的Context代替Activity。`

---
##### 2. 持有Activity内的静态View
示例代码：
```java
public class TestDataModel {
    private static TestDataModel sInstance;
    private TextView mRetainedTextView;
    public static TestDataModel getInstance() {
        if (sInstance == null) {
            sInstance = new TestDataModel();
        }
        return sInstance;
    }
    public void setRetainedTextView(TextView textView) {
        mRetainedTextView = textView;
    }
}
```
```java
public class SecondActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_second);
        TextView textView = (TextView) findViewById(R.id.test_text_view);
        TestDataModel.getInstance().setRetainedTextView(textView);
    }

}
```
LeakCanary提示：

![持有静态View导致的内存泄漏](/assets/img/blogs/leakcanary/textview1.PNG)

原因分析：
>`TestDataModel 持有Activity中的TextView的静态引用，而TextView又持有SecondActivity的引用，从而导致SecondActity在onDestory之后不能释放`

解决方法：
>`尽量避免这种用法`

---
##### 3.较长生命周期的匿名内部类

示例代码：
```java
public class SecondActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_second);
        testLeakMemory();
    }
    
    private void testLeakMemory(){
        new Thread(new Runnable() {
            @Override
            public void run() {
                SystemClock.sleep(200000);
            }
        }).start();
    }

}
```
LeakCanary提示：

![匿名内部类导致的内存泄漏](/assets/img/blogs/leakcanary/匿名内部类1.PNG)


原因分析：
>`SecondActivity中的匿名内部类持有SecondActivity的引用，并且该匿名内部类的生命周期比Activity要长`

解决方法：
>`SecondActivity在销毁时，应取消内部类对其的引用`

---
##### 4.Handler中有生命周期较长的匿名内部类

示例代码：
```java
public class SecondActivity extends AppCompatActivity {

    private Handler mHandler;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_second);
        testLeakMemory();
    }

    private void testLeakMemory(){
        HandlerThread thread = new HandlerThread("test");
        thread.start();
        mHandler = new Handler(thread.getLooper());
        mHandler.post(new Runnable() {
            @Override
            public void run() {
                SystemClock.sleep(200000);
            }
        });
    }
}
```
LeakCanary提示：

![Handler中有匿名内部类Runnable](/assets/img/blogs/leakcanary/handler1.PNG)

原因分析：
>`mHandler在post时，其参数为匿名内部类，持有SecondActivity的引用，并且该Runnable的生命周期比SecondActivity长`

解决方法：
>`SecondActivity在onDestory时，应移除mHandler中未完成的任务`

---
##### 5. 资源未关闭造成的内存泄漏
Cursor、BroadcastReceiver、 TypedArray、File 、Stream 、Bitmap等使用完毕后应及时释放或关闭或反注册。
示例代码（Cursor）：
```java
public class SecondActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_second);
        testLeakMemory();
    }

    private void testLeakMemory(){
        Cursor cursor = getContentResolver().query(MediaStore.Audio.Media.EXTERNAL_CONTENT_URI,null,null,null,null);
        if(cursor != null){
            while (cursor.moveToNext()){
                Log.d("test",cursor.getString(0));
            }
        }
    }

}
```

示例代码（BroadcastReceiver）：
```java
public class SecondActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_second);
        testLeakMemory();
    }

    private void testLeakMemory(){
        IntentFilter filter = new IntentFilter(Intent.ACTION_MEDIA_BUTTON);
        TestReceiver receiver = new TestReceiver();
        registerReceiver(receiver,filter);
    }

    private class TestReceiver extends BroadcastReceiver{
        @Override
        public void onReceive(Context context, Intent intent){

        }
    }
}
```
解决方法：
>`Cursor、BroadcastReceiver、 TypedArray、File 、Stream 、Bitmap等使用完毕后应及时关闭或释放`





