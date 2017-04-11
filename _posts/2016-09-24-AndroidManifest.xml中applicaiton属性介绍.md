---
title: AndroidManifest.xml中applicaiton属性介绍
date: 2016-09-24 21:43:10
tags:
- Application
- Android
categories: Android
---

application支持的属性如下：

#### `*1. android:allowTaskReparenting=["true" | "false"]`
>当该Task下一次被带到前面时，应用程序定义的Activity是否可以从启动它们的Task移动到有相同affinity 的Task - 如果它们可以移动，则为true，false表示它们仍要停留在启动它们的task中。 默认值为“false”。Activity可以设置自己的allowTaskReparenting属性来覆盖Application节点下的这个属性。

---

#### `*2.  android:allowBackup=["true" | "false"]`
>是否允许应用程序参与备份和恢复基础架构。 如果此属性设置为false，则不会执行应用程序的备份或恢复，即使是全系统备份，否则将导致所有应用程序数据通过adb保存。 此属性的默认值为true。

`注意：慎用该属性！！！`当该属性为true时，可通过adb backup命令将应用中的数据备份到pc上：
```bash
adb backup -nosystem -noshared -apk -f d:\com.test.bak.ab com.test.bak
```
各参数意义如下：
* -nosystem：不备份系统应用
* -noshared：不备份应用存储在SD中的数据
* -apk：备份应用APK安装包 
* -f d:\com.test.bak.ab： 备份文件在pc上的的路径
* com.test.bak： 要备份的包名
将数据备份到pc上后，将另一台手机连接该pc，然后执行：
```bash
adb restore d:\com.test.bak.ab
```
此时，第一部手机上被备份的应用会在第二部手机上被恢复，从而可能出现泄漏隐私。如果处于业务需求需要将该属性设置为true，则要增加额外的身份验证以及其他安全验证手段。

---
#### `*3. android:backupAgent="string"`
>实现应用backup代理的类名，它是BackupAgent的一个子类。这个属性应当是一个类的全路径名（比如com.example.project.MyBackupAgent）。如果类名的第一个字母是“.”（ 比如".MyBackupAgent"）,它会附加到 < manifest> 元素中指定的包名之后。没有默认值。必须指定该名称。

---
#### `4. android:backupInForeground=["true" | "false"]`
>一个应用即使在前台状态下也可以执行自动备份。当应用正在自动备份时，系统可能会停止应用，因此要谨慎的使用该属性。该属性设置为true会影响到处于激活状态下的应用的行为。
>默认值为false，意味者系统会避免在应用处于前台状态时执行备份操作（比如一个处于激活状态的音乐应用，它通过startForeground的方式启动了一个前台服务）。


---
#### `5.  android:banner="drawable resource"`
>一个drawable资源，用来提供一个扩展的图形banner。使用 < application>标签为应用所有的Activity提供一个默认的banner，或者使用 < activity>标签为一个指定的Activity设置banner。
>在Aandroid TV的home页，系统使用banner来表示一个应用。由于banner仅仅在home页显示，当一个应用中有处理CATEGORY_LEANBACK_LAUNCHER这个intent的Activity时，才能指定该属性。  
>这个属性必须设置为一个包含图片的drawable资源的引用（例如 "@drawable/banner"）。没有默认值。

---
#### `6. android:debuggable=["true" | "false"]`
>它表示当一个设备即使运行在user模式，能否被调试-true表示能被调试，false表示不能被调试。默认值是false。

---
#### `7. android:description="string resource"`
>关于应用的用户可读的文字，比label属性更长、更具描述性。这个属性必须设置为一个字符串资源的引用。不像label属性，它不能是一个raw字符串。没有默认值。           

---
#### `8.  android:enabled=["true" | "false"]`
>Android系统能否实例化应用的组件-true表示可以，false表示不可以。如果设置为true，每个组件的enabled 属性决定了这个组件是否可用。如果设置为false，它会覆盖组件指定的值，即所有的组件都被禁用。默认值为true。

---
#### `*9.  android:extractNativeLibs=["true" | "false"]`
>包安装器能否从apk中抽取本地库到文件系统。如果设置为false，你的本地库必须页对其并且在apk中以未压缩的方式存储。链接器在运行时从apk中直接加载库不会导致代码改动。默认值为true。

---
#### `10. android:fullBackupContent="string"`
>该属性指向一个xml文件，这个文件包含了自动备份时全部的备份规则。这些规则决定了哪些文件会被备份。该属性为可选的。如果没设置，默认情况下，自动备份会备份应用的大部分的文件。

---
#### `11.  android:fullBackupOnly=["true" | "false"]`
>当设备可用时是否使用自动备份。如果设置为true，当应用安装到运行Android 6.0及其更高版本的的设备上时，会运行自动备份，在低版本的设备上，应用将忽略这个属性并执行key/value方式的备份。默认值是false。

---
#### `12.  android:hasCode=["true" | "false"]`
>应用是否包含代码-true表示有代码，false表示没有代码。当设置为false，当加载组件时，系统不会尝试加载应用的任何代码。默认值是true。
>当一个应用仅仅使用了内置的组件时，比如使用了AliasActivity 的Activity，才不会有任何代码，这种场景很少出现。

---
#### `13. android:hardwareAccelerated=["true" | "false"]`
>应用中所有的Activity和View是否启用硬件加速渲染-true表示启用，false表示不启用。如果你设置了minSdkVersion o或targetSdkVersion 为14或更高，默认是启用；否则默认不启用。
>从 Android 3.0 (API level 11)开始，应用可使用支持硬件加速的OpenGL 渲染，用来提升许多通用的2D图形操作性能。当硬件加速渲染打开时，Canvas, Paint, Xfermode, ColorFilter, Shader, 和 Camera中的大部分操作都会被加速。这会使得动画更流畅、滑动更流畅，提升整体的响应性，甚至应用不必显式的使用framework的 OpenGL库。
>需要注意的是，不是全部的OpenGL 2D操作都会被加速。如果你打开了硬件加速渲染，请测试你的应用以确保使用渲染的过程中不会出错。

---
#### `14.  android:icon="drawable resource"`
>一个作为整体应用的图标，并且是应用的每个组件的默认图标。 < activity>, < activity-alias>, < service>, < receiver>和 < provider>都可以有各自的图标。这个属性必须被设置为一个包含图片的drawable资源文件的引用（比如"@drawable/icon"）。没有默认图标。

---
#### `15.   android:isGame=["true" | "false"]`
>应用是否为游戏。系统可能把应用根据游戏分组或把它们与其他的应用分开来显示。默认为false。

---
#### `16. android:killAfterRestore=["true" | "false"]`
>当应用在全系统层面的恢复操作时，应用的设置被恢复后，应用应当终止运行。单个包的恢复操作不会导致应用关闭。全系统的恢复操作一般来说只会发生一次，即在手机第一次启动时。第三方的应用一般没必要使用这个属性。默认值为true，意味着当应用在全系统的恢复操作中处理完自己的数据后，将会停止运行。

---
#### `*17. android:largeHeap=["true" | "false"]`
>你的应用进程是否通过一个large  Dalvik heap 被创建。这个属性适用于所有的为应用创建的进程。它仅适用于被加载到进程的第一个应用；如果你使用shared user id来允许多个应用在同一个进程中，它们都必需一致的使用这个选项，否则可能出现不可预知的结果。
>大部分app都不应使用这个属性，而应该把焦点放在如何减少整体的内存使用来提示性能。启用此功能也不能保证可用内存的固定增加，因为一些设备被全部可用内存限制了。
>可使用ActivityManager.getMemoryClass() or ActivityManager.getLargeMemoryClass()在运行时获取可用的内存大小。

---
#### `18. android:label="string resource"`
>把应用作为整体的用户可读标签，也是应用中各个组件的默认标签。 < activity>, < activity-alias>, < service>, < receiver>, < provider> 都可以设置各自的标签。标签应设置为一个字符串资源的引用，以便可以像用户界面中的其他字符串一样进行本地化。为了方便，在开发应用时可以设置为原始字符串。

---
#### `19. android:logo="drawable resource"`
>应用作为一个整体的logo，也是所有Activity的默认logo。这个属性必须被设置为一个包含图片的drawable资源的引用（比如@drawable/logo）。没有默认logo。


`默认情况下，使用在ActionBar中使用icon，如果同时也设置了logo属性，ActionBar上将使用logo。logo应比icon远，但不应包括不必要的文本。 只有在以用户认可的传统格式表示您的品牌时才应使用logo。 logo代表预期的使用者品牌，而应用程序的的icon是符合启动器图示方形要求的修改版本。`

---
#### `20. android:manageSpaceActivity="string"`
>一个Activity子类的全限定名，系统可以启动它来让用户管理设备上应用的内存使用情况。这个Activity也必须使用< activity> 元素来声明。

---
#### `21.  android:name="string"`
>Application类的子类名。当应用进程启动时，这个类会在应用所有的组件之前被实例化。这个属性是可选的；大部分的应用都没有必要使用，在没有子类的情况下，Android使用Application的实例。

---
#### `22.  android:permission="string"`
>为了跟应用交互，客户端必须拥有的权限。这个属性是适用于应用中所有组件设置权限的一个方便的方法。每个组件可设置各自的permission属性来覆盖application中的这个属性。

---
#### `*23.  android:persistent=["true" | "false"]`
>应用是否应始终保持运行状态-true表示是，false表示否。默认值是false。应用通常不应设置此标志；persistence 模式仅适用于某些系统应用。

---
#### `24.  android:process="string"`
>应用程序所有的组件运行时所在的进程名。每个组件都可以设置自己的process 属性来覆盖整个属性。默认情况下，当应用的第一个组件需要运行时，Android会为应用创建一个进程。默认的进程名为manifest文件中设置的包名。

>通过将此属性设置为与另一个应用程序共享的进程名称，您可以安排两个应用程序的组件在同一进程中运行，但前提是这两个应用程序也共享一个用户ID并使用相同的证书签名。


>如果分配给此属性的名称以冒号（'：'）开头，则在需要时，将创建对应用程序为专用的新进程。 如果进程名称以小写字符开头，则会创建该名称的全局进程。 全局进程可以与其他应用程序共享进程，从而减少资源使用。

---
#### `25.  android:restoreAnyVersion=["true" | "false"]`
>表示应用程序准备尝试还原任何备份的数据集，即使该备份是由比当前在设备上安装的应用程序的更新版本存储的。 将此属性设置为true将允许备份管理器尝试恢复，即使版本不匹配表明数据不兼容。 使用时要小心！此属性的默认值为false。


---
#### `26.  android:requiredAccountType="string"`
>指定应用程序为了运行所需的帐户类型。 如果您的应用需要帐户，则此属性的值必须与您的应用使用的帐户验证器类型（由AuthenticatorDescription定义）（如“com.google”）相对应。默认值为null，表示应用程序可以在没有任何帐户的情况下工作。

>由于受限个人资料目前无法添加帐户，因此指定此属性会使您的应用无法从受限个人资料中获取，除非您也声明了android：restrictedAccountType具有相同的值。

>警告：如果帐户数据可能显示个人身份信息，请务必声明此属性并将android：restrictedAccountType设置为null，以便受限配置文件无法使用您的应用访问属于所有者用户的个人信息。

>此属性在API级别18中添加。

---
#### `27.android:resizeableActivity=["true" | "false"]`
>指定应用是否支持多窗口显示。你可以在 < activity> 或 < application>元素中设置该属性。
>如果设置为true，用户可在分屏和freeform 模式下启动Activity。如果设置为false，Activity将不支持多窗口模式。如果这个值为false，并且用户尝试在多窗口模式下启动Activity，Activity将会全屏显示。
>如果你的应用的 targets API 是24或者更高，如果你不指定这个属性，默认值是true。这个属性是在 API level 24新增的。

---
#### `28. android:restrictedAccountType="string"`
>指定此应用程序所需的帐户类型，并指示允许受限制的配置文件访问属于所有者用户的此类帐户。 如果您的应用需要帐户，且受限个人资料可以访问主要用户的帐户，则此属性的值必须与您的应用所使用的帐户验证器类型（由AuthenticatorDescription定义）（如“com.google”）相对应。默认值为null，表示应用程序可以在没有任何帐户的情况下工作。

>警告：指定此属性允许受限配置文件使用属于所有者用户的帐户使用您的应用程序，这可能会泄露个人身份信息。 如果帐户可能显示个人详细信息，您不应使用此属性，而应声明android：requiredAccountType属性，使您的应用程序不能使用限制的配置文件。

>此属性在API级别18中添加。

---
#### `29. android:supportsRtl=["true" | "false"]`
>声明你的应用是否支持从右向左布局。
>如果设置为true并且targetSdkVersion 大于等于17，多个RTL相关的API将被激活，被系统使用，从而使得你的app能够显示从右向左的布局。
>如果设置为false或targetSdkVersion小于17，RTL相关的API将无效或者被忽略，你的app将无视布局方向，始终是从左向右的方向。
>默认值是false，在API level 17中引入的这个属性。

---
#### `*30. android:taskAffinity="string"`
>一个应用中所有的Activity的affinity名称，除了那些设置了不同affinity属性的Activity。默认情况下，一个应用中所有的Activity的affinity都相同，为manifest中通过 < manifest>元素设置的包名。

---
#### `31.  android:testOnly=["true" | "false"]`
>表示一个应用是否仅仅用于测试。例如，它可能暴露它自己的功能或外部数据，这些可能导致一个安全漏洞，但对调试来说有用。这类应用仅仅可以通过adb命令安装。

---
#### `*32. android:theme="resource or theme"`
>为应用中所有的Activity定义默认主题的样式资源的引用。每个Activity都可以通过设置它们自己的这个属性。      

---
#### `33.  android:uiOptions=["none" | "splitActionBarWhenNarrow"]`
>Activity UI的额外选项，必须是下列值中的一个。
>* "none"： 没有额外的UI选项，这是默认值。
>* "splitActionBarWhenNarrow"：在屏幕底部添加一个条形，以在水平空间受限时（例如在手机上的纵向模式下）在应用栏（也称为操作栏）中显示操作项目。 而不是显示在屏幕顶部的应用栏中的少量操作项，应用栏分割为顶部导航部分和操作项的底部栏。 这确保了不仅为操作项目而且在顶部的导航和标题元素提供合理的空间量。 菜单项不分割在两个条上; 他们总是出现在一起。
>该属性是在 API level 14中新增的。

---
#### `34.  android:usesCleartextTraffic=["true" | "false"]`
>指示应用程序是否要使用明文网络流量，例如明文HTTP。默认值为“true”。当属性设置为“false”时，平台组件（例如，HTTP和FTP堆栈，DownloadManager，MediaPlayer）将拒绝应用程序使用明文流量的请求。强烈鼓励第三方图书馆也遵守此设置。避免明文流量的关键原因是缺乏保密性，真实性和防止篡改的保护：网络攻击者可以窃听传输的数据，并且还可以在不检测的情况下对其进行修改。

>这个标志是最好的努力的基础上，因为它是不可能防止来自Android应用程序的所有明文流量给定的访问级别提供给他们。例如，没有期望Socket API将尊重此标志，因为它不能确定其流量是否为明文。然而，来自应用程序的大多数网络流量由更高级网络堆栈/组件处理，可以通过从ApplicationInfo.flags或NetworkSecurityPolicy.isCleartextTrafficPermitted（）读取该标志来处理该标志。

>注意：WebView不遵守此标志。

>在应用程序开发期间，StrictMode可用于标识来自应用程序的任何明文流量：请参阅StrictMode.VmPolicy.Builder.detectCleartextNetwork（）。

>此属性是在API级别23中添加的。

>如果存在Android网络安全配置，此标记在Android 7.0（API级别24）及以上版本中将被忽略。

---
#### `35. android:vmSafeMode=["true" | "false"]`
>表示应用让虚拟机运行在安全模式。默认值为false。这个属性在 API level 8 中引入，true表示禁用Dalvik  JIT编译器。在API level 22 中有改动，true表示禁用ART AOT编译器。