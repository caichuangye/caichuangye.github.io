---
title:  shareduserid、process和key
date: 2016-07-28 22:25:10
tags:
- Android
categories: Android
---

#### shareduserid
该属性有两个作用：
1. 共享数据
 若两个apk设置了相同的shareduserid，则表示这两个apk属于同一个用户，可以共享/data/data/【包名】/下的数据文件。

2. 提升应用权限
将应用的uid设置为某些系统进程的uid，这样应用就可以提示权限。
build/target/product/security目录中有四组默认签名供Android.mk在编译APK使用：
1、testkey：普通APK，默认情况下使用。
2、platform：该APK完成一些系统的核心功能。经过对系统中存在的文件夹的访问测试，这种方式编译出来的APK所在进程的UID为system。
3、shared：该APK需要和home/contacts进程共享数据。
4、media：该APK是media/download系统中的一环。
应用程序的Android.mk中有一个LOCAL_CERTIFICATE字段，由它指定用哪个key签名，未指定的默认用testkey.


#### key
key用来表示一个apk和它的开发者之间的关系

#### process
指定一个组件所在的进程名，默认为包名。注意，仅仅是指定进程名！

* 只设置process只能改变组件所在的进程名！！！若两个apk仅仅设置了process属性相同，只能说明它们的进程名是相同的，其uid、pid是不同的，并且此时系统中存在两个同名的进程，仅此而已！！！
![仅设置process](/assets/img/blogs/shareduid_process_key/sameprocessname.PNG)

* 只设置两个apk的shareduserid，则两个apk的数据目录下的文件可以互相共享
![仅设置shareduserid](/assets/img/blogs/shareduid_process_key/捕获.PNG)


* 若要两个apk运行在同一个进程中，则必须满足三个条件：
1. process属性相同
2. shareduserid相同
3. key相同

**`为什么只有满足以上三个条件两个apk才能运行在同一个进程？`**
`1. 首先，设置为同一个process属性是必须的，运行在用一个进程进程名肯定要相同。但这一步也仅仅设置了两个进程名相同而已。`
`  2. android中每个apk在安装时系统都会分配一个uid，即把每个apk都当做一个用户，两个apk只有uid才能互相共享数据。理论上，设置了process和shareduserid相同已经可以保证两个apk在同一个进程并共享数据了，实际上是不可以的。`
`3. 这两个属性都是在AndroidManifest.xml中配置的，很容易通过反编译的方式看到具体的配置，必须要通过其他的方法，把第三者apk排查再外。还剩最后一招：两个apk的签名必须一致！apk的签名需要签名文件和密码，这些是开发者私有的。`
