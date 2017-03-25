---
title: 使用gradle减少apk大小
date: 2016-12-12 22:46:49
tags:
- Gradle
- Android
---
### 一. proguard

>ProGuard是一个Java工具，不仅可以减少APK文件大小，还可以在编译期间优化、混淆和预校验代码。通过应用的所有的代码路径，找到未被使用到的代码，并将其删除。ProGuard还会重命名类和方法。

`官方链接：`[https://www.guardsquare.com/en/proguard](https://www.guardsquare.com/en/proguard)
```
android {
    buildTypes {
        release {
            minifyEnabled true
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }
}
```
当minifyEnabled被设置为true，在构建过程中，proguardRelease Task会被执行，调用ProguardggetDefaultProguardFile('proguard-android.txt')方法从Android SDK的tools/proguard文件夹下的proguard-android.txt中获取默认的Proguard设置， 也可以在当前module中的proguard-rules.pro指定一些针对当前module的规则。

---
### 二. 缩减资源
>当给app打包时，Gradle和Gradle的Android插件可以在构建期间删除所有未使用的资源。如果有旧的资源忘记删除，那么这个功能可能非常有用。另一个使用案例就是当导入一个拥有很多资源的依赖库，但实际上只使用了其中的一小部分，可以通过激活缩减资源来解决这个问题。缩减资源有两种方式：自动和手动。
####1. 自动缩减shrinkResources
最简单的方式是在构建中设置shrinkResources属性，如果设置该属性为true，则Android构建工具将自动判断哪些资源没有被使用，并将它们排除在APK外。使用该功能时，必须开启ProGuard。这是因为缩减资源的工作方式是：直到代码引用这些资源被删除之前，Android构建工具不能指出哪些资源没有被用到。
```
 buildTypes {
        release {
            minifyEnabled true
            shrinkResources true
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }
```
如果想看看在激活自动缩减资源后，APK缩减了多少，可以运行shrinkReleaseResources任务，这个任务会打印出包的大小缩减了多少。也可以通过在构建命令后添加-info标志，来获取APK缩减资源的概览：
```
gradlew clean assembleRelease -info
```

自动缩减资源有一个问题：它可能移除了过多的资源，特别是那些动态使用的资源肯能被意外删除。为了防止这种情况，可以在res/raw/下的一个叫keep.xml的文件中定义这些例外，一个简单的keep.xml的文件如下：
```xml
<?xml version="1.0" encoding="utf-8"?>
<resources xmlns:tools="http://schemas.android.com/tools"
	tools:keep="@layout/keep_me,@layout/main_*"/>
```
keep.xml文件本身也将从最终的结果中被剥离出来。
#### 2. 手动缩减resConfigs
去除某种语言文件或某个密度的图片，是删减资源的一种比较好的方式，可以通过resConfigs熟悉配置想要保留的资源，其余部分将被删除。

Case 1. 只保留指定语言的字符串
```
android{
	defaultConfigs{
		resConfigs "en","da","nl"
	}
}
```
case2. 只保留指定密度的图片
```
android{
	defaultConfigs{
		resConfigs "hdpi","xhdpi","xxhdpi","xxxhdpi"
	}
}
```
### 三. APK分割
可以通过在android配置代码中定义一个splits代码块来配置分割，目前支持density分割和ABI分割。
splits中支持以下属性：
>* enable：boolean型，表示打开或关闭APK分割功能
>* reset()：复位，若要使用include功能，则使用前需调用reset()
>* include：创建白名单，仅构建出白名单中指定的格式
>* exclude：黑名单，不会构建出黑名单中指定的格式
>* compatibleScreens(仅限density)：未知
>* universalApk(仅限ABI)：默认为true，即除了指定的格式外，还会构建出一个通用的APK
#### 1. density splits
```
android{}
	splits {
	        density {
	            enable true
	            exclude 'ldpi','mdpi'
	            compatibleScreens 'normal', 'large', 'xlarge'
	        }
	    }
}
```
仅构建release，输出为：
* app-hdpi-release.apk
* app-universal-release.apk
* app-xhdpi-release.apk
* app-xxhdpi-release.apk
* app-xxxhdpi-release.apk
#### 2.  ABI splits
```
android{
	 splits {
	        abi {
	            enable true
	            reset()
	            include 'x86', 'armeabi-v7a', 'mips'
	            universalApk false
	        }
	    }
}
```
仅构建debug，输出为：
* app-armeabi-v7a-debug.apk
* app-mips-debug.apk
* app-x86-debug.apk