---
title: Android平台开源log工具-Logger
date: 2016-12-27 23:26:32
tags:
- Log
- Android
categories: Android
---
#### 一. Logger介绍
`Logger是一个Android平台下简单、优雅、功能强大的日志工具。github:` [Logger](https://github.com/orhanobut/logger)

Logger提供：

* 线程信息
* 类信息
* 函数信息
* 格式化的json格式输出
* 优雅的新行"\n"输出
* 清除输出
* 跳转到源码

#### 二. 常见用法
##### 2.1 将Logger加入到Android Studio工程
```
compile 'com.orhanobut:logger:1.15'
```
##### 2.2 初始化
方式1： 默认初始化，使用默认的tag：PRETTYLOGGER
```java
Logger.init();
```
方式2：默认初始化，使用自定义的tag
```java
Logger.init("test_logger");
```
方式三：自定义设置选项：
```java
Logger
  .init(YOUR_TAG)                 // default PRETTYLOGGER or use just init()
  .methodCount(3)                 // default 2
  .hideThreadInfo()               // default shown
  .logLevel(LogLevel.NONE)        // default LogLevel.FULL
  .methodOffset(2)                // default 0
  .logAdapter(new AndroidLogAdapter()); //default AndroidLogAdapter
}
```


##### 2.3 输出日志


#### 三. 原理
##### 3.1 StackTraceElement
Java中，错误信息是通过StackTraceElement对象表示的：
```java
public final class StackTraceElement implements java.io.Serializable {
    private String declaringClass;
    private String methodName;
    private String fileName;
    private int    lineNumber;
    ...
}
```
该类中存储了一条栈帧的基本信息：
* String declaringClass：本次函数调用的所在的类
* String methodName：本次函数调用的所在函数名
* String fileName：类所在的文件名
* int    lineNumber：当前函数调用在文件中的行号

不管是Logger还是Android中的Log类，在输出错误堆栈时，都是通过StackTraceElement来表示调用堆栈的。


##### 3.2 Logger中的设置选项
1.  private int methodCount = 2
  在输出log调用栈时，输出的方法数，默认值为2。是从调用栈的顶部开始计算的。当输出错误日志时，可将该值设置的大些，这样可以输出较多的调用栈信息，有助于定位问题的源头。
  
  ---
2.  private boolean showThreadInfo = true
  是否显示线程信息，即当前输出log的是哪个线程。true表示显示线程信息，false表示不显示线程信息，默认值为true。在多线程的情况下，该选项可以帮助快速定位输出log的线程。
  
 --- 
3. private int methodOffset = 0
  输出日志的调用栈时，输入栈的偏移量，默认值为0，即从输出log的函数开始，逐个函数输出完成的调用栈。某些情况下可能需要屏蔽当前log的直接或间接调用函数，可将该变量设置为一个大于0的整数，输出时，将不输出从栈顶部开的methodOffset 个函数信息。
  ---
4. private LogAdapter logAdapter
log输出的接口，Logger默认实现了一个，即AndroidLogAdapter，在AndroidLogAdapter中，log是通过Android的Log中的方法输出的。如果需要将log写入到文件或者上传到服务器，可从新实现一个实现LogAdapter 接口的类。

---
5. private LogLevel logLevel = LogLevel.FULL
log输出级别，Logger支持两个级别：FULL和NONE，FULL表示输出任何级别的log，NONE表示不输出任何级别的log。
##### 3.3 LogAdapter设计
log输出有多种方式，比如输出到控制台、写入到文件、上传到服务器等。在Logger中，在log的输出方式上采用面向接口的方式设计。即定义了一个接口，在接口中声明了所有的log输出方法，如下所示：
```java
public interface LogAdapter {
  void d(String tag, String message);
  void e(String tag, String message);
  void w(String tag, String message);
  void i(String tag, String message);
  void v(String tag, String message);
  void wtf(String tag, String message);
}
```

Logger实现了一个实现了LogAdapter 接口的类AndroidLogAdapter ，如下。在该类的实现中，调用Android的Log类实现LogAdapter中定义的接口。开发者可根据自己的需求，自定义一个实现LogAdapter 的类，将log输出到指定的地方。
```java
class AndroidLogAdapter implements LogAdapter {
  @Override public void d(String tag, String message) {
    Log.d(tag, message);
  }

  @Override public void e(String tag, String message) {
    Log.e(tag, message);
  }

  @Override public void w(String tag, String message) {
    Log.w(tag, message);
  }

  @Override public void i(String tag, String message) {
    Log.i(tag, message);
  }

  @Override public void v(String tag, String message) {
    Log.v(tag, message);
  }

  @Override public void wtf(String tag, String message) {
    Log.wtf(tag, message);
  }
}
```
##### 3.4 图形化输出//to-do