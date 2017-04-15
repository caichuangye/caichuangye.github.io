---
title: Activity launchMode和taskaffinity
date: 2016-05-25 22:45:50
tags:
- Activity
- Android
categories: Android
---

#### 一. Activity的四种launchMode
启动Activity时，可设置launchmode为以下四种方式之一，默认为standard模式。`以下说明都有一个前提：所有的Activity的taskAffinity属性相同。`
`1.1 standard`
标准模式，每次startActivity时都会在当前task中新建一个Activity，这是最常见的方式。这种方式在逻辑上符合用户的操作，即每次startActivity时都会新建，每次按下back键时会返回上一个创建的Activity。

---
`1.2 singleTop`
如果当前Task中有该Activity并且存在于栈顶，则不会新建Activity，此时已存在的Activity的onNewIntent会被调用:
```java
protected void onNewIntent(Intent intent) {
	...
}
```
如果目标Activity不在栈顶，则会新建目标Activity，该种情况等同于standard模式。
**`注意：`**
* `Activity在收到onNewIntent之前会先暂停，事件流程为：onPause -> onNewIntent -> onResume。`
* `在onNewIntent之后，通过getIntent方法获得的intent还是原始的Intent，可通过setIntent方法更新当前的Intent。`
* `该模式等同于在startActivity设置Intent的FLAG_ACTIVITY_SINGLE_TOP标志`。

`应用场景：`
singleTop适合接收通知启动的内容显示页面。
在应用的信息详情页，当在通知栏收到消息推送后，点击通知栏还是进入到详情页，如果详情页的Activity的launchMode为standard模式，则意味着每次点击通知栏都会创建一个新的Activity，而且需要按多次back才能返回上上一级页面。此时，详情页的Activity应该设置为singleTop模式，这样从通知栏进入时显示的还是当前页面的Activity，页面具体信息通过onNewIntent更新。

---
`1.3 singleTask`
* 若要创建的目标Activity在当前Task中不存在，则会新建一个目标Activity，该种情况等同于standard模式；
* 若要创建的目标Activity在当前Task中存在，则会将该Activity之上所有的Activity弹出，将目标Activity置于栈顶，目标Activity的onNewIntent会被调用。

**`注意：`**
`singleTask模式的Activity只允许在系统中有一个实例.`

`应用场景：`
singleTask适合作为程序入口点。
例如浏览器的主界面，不管从多少个应用启动浏览器，只会启动主界面一次，其余情况都会走onNewIntent，并且会清空主界面上面的其他页面。

---
`1.4 singleInstance`
在这种模式下Activity将独占一个Task，Activity会在一个新的Task中被创建。

假设以下场景：
前提：有三个Activity，分别为A、B、C，在A中点击启动B，在B中点击启动C（A -> B ->C），启动到C界面：
* 场景一.  A为singleInstance模式，B、C为standard模式
`此时A在一个单独的Task中，B、C在同一个Task中，按下两次back键，依次显示的界面为C、B、A。`
` `

* 场景二.  B为singleInstance模式，A、C为standard模式
` 此时B在一个单独的Task中，A、C在同一个Task中，按下两次back键，依次显示的界面为C、A、B。即使B在A之后创建，但回退时，由于A和C在同一个栈中，必须将当前Task中的Activity全部弹出来之后，才能处理另外的Task。`
 ` `
* 场景三.  C为singleInstance模式，A、B为standard模式
 `此时C在一个单独的Task中，A、B在同一个Task中，按下两次back键，依次显示的界面为C、B、A。`


`应用场景：`
闹铃的响铃界面。 你以前设置了一个闹铃：上午6点。在上午5点58分，你启动了闹铃设置界面，并按 Home 键回桌面；在上午5点59分时，你在微信和朋友聊天；在6点时，闹铃响了，并且弹出了一个对话框形式的 Activity(名为 AlarmAlertActivity) 提示你到6点了(这个 Activity 就是以 SingleInstance 加载模式打开的)，你按返回键，回到的是微信的聊天界面，这是因为 AlarmAlertActivity 所在的 Task 的栈只有他一个元素， 因此退出之后这个 Task 的栈空了。如果是以 SingleTask 打开 AlarmAlertActivity，那么当闹铃响了的时候，按返回键应该进入闹铃设置界面。
` `
` `

#### 二. < activity>中跟Task相关的属性
`2.1 android:allowTaskReparenting=["true" | "false"]`
>当启动 Activity 的任务接下来转至前台时，Activity 是否能从该任务转移至与其有亲和关系的任务 —“true”表示它可以转移，“false”表示它仍须留在启动它的任务处。
>如果未设置该属性，则对 Activity 应用由 <application> 元素的相应 allowTaskReparenting 属性设置的值。 默认值为“false”。

>正常情况下，当 Activity 启动时，会与启动它的任务关联，并在其整个生命周期中一直留在该任务处。您可以利用该属性强制 Activity 在其当前任务不再显示时将其父项更改为与其有亲和关系的任务。该属性通常用于使应用的 Activity 转移至与该应用关联的主任务。

>例如，如果电子邮件包含网页链接，则点击链接会调出可显示网页的 Activity。 该 Activity 由浏览器应用定义，但作为电子邮件任务的一部分启动。 如果将其父项更改为浏览器任务，它会在浏览器下一次转至前台时显示，当电子邮件任务再次转至前台时则会消失。

>Activity 的亲和关系由 taskAffinity 属性定义。 任务的亲和关系通过读取其根 Activity 的亲和关系来确定。因此，按照定义，根 Activity 始终位于具有相同亲和关系的任务之中。 由于具有“singleTask”或“singleInstance”启动模式的 Activity 只能位于任务的根，因此更改父项仅限于“standard”和“singleTop”模式。 （另请参阅 launchMode 属性。）

假设以下场景：
AppA：A -> B -> C
AppB:  D -> B（D页面点击唤起B页面）
* 当B的allowTaskReparenting为false时，在AppB中唤起的B与D在同一个Task中，与AppA没有关联
* 当B的allowTaskReparenting为true时，在AppB中唤起的B与D在同一个Task中，在AppB中唤起B后，按下home键，此时再启动AppA，AppA中出现的第一个Activity为B，即在AppB中唤起的B被重新宿主到了AppA的Task中，按下back键，此时出现的为A！在桌面再次点击AppB的图标，此时出现的是界面D，因为此时B已经被重新宿主到了AppA的Task中了。

---
`2.2 android:alwaysRetainTaskState=["true" | "false"]`
> 系统是否始终保持 Activity 所在任务的状态 —“true”表示保持，“false”表示允许系统在特定情况下将任务重置到其初始状态。 默认值为“false”。该属性只对任务的根 Activity 有意义；对于所有其他 Activity，均忽略该属性。

>正常情况下，当用户从主屏幕重新选择某个任务时，系统会在特定情况下清除该任务（从根 Activity 之上的堆栈中移除所有 Activity）。 系统通常会在用户一段时间（如 30 分钟）内未访问任务时执行此操作。

>不过，如果该属性的值是“true”，则无论用户如何到达任务，将始终返回到最后状态的任务。 例如，在网络浏览器这类存在大量用户不愿失去的状态（如多个打开的标签）的应用中，该属性会很有用。

`2.3 android:clearTaskOnLaunch=["true" | "false"]`
>是否每当从主屏幕重新启动任务时都从中移除根 Activity 之外的所有 Activity —“true”表示始终将任务清除到只剩其根 Activity；“false”表示不做清除。 默认值为“false”。`该属性只对启动新任务的 Activity（根 Activity）有意义；对于任务中的所有其他 Activity，均忽略该属性。`

>当值为“true”时，每次用户再次启动任务时，无论用户最后在任务中正在执行哪个 Activity，也无论用户是使用返回还是主屏幕按钮离开，都会将用户转至任务的根 Activity。 当值为“false”时，可在某些情况下清除任务中的 Activity（请参阅 alwaysRetainTaskState 属性），但并非一律可以。

>例如，假定有人从主屏幕启动了 Activity P，然后从那里转到 Activity Q。该用户接着按了主屏幕按钮，然后返回到 Activity P。正常情况下，用户将看到 Activity Q，因为那是其最后在 P 的任务中执行的 Activity。 不过，如果 P 将此标志设置为“true”，则当用户按下主屏幕将任务转入后台时，其上的所有 Activity（在本例中为 Q）都会被移除。 因此用户返回任务时只会看到 P。

>如果该属性和 allowTaskReparenting 的值均为“true”，则如上所述，任何可以更改父项的 Activity 都将转移到与其有亲和关系的任务；其余 Activity 随即被移除。


---
`2.4 android:finishOnTaskLaunch=["true" | "false"]`
>每当用户再次启动其任务（在主屏幕上选择任务）时，是否应关闭（完成）现有 Activity 实例 —“true”表示应关闭，“false”表示不应关闭。 默认值为“false”。
>
> 如果该属性和 allowTaskReparenting 均为“true”，则优先使用该属性。 Activity 的亲和关系会被忽略。 系统不是更改 Activity 的父项，而是将其销毁。

---
`2.5 android:noHistory=["true" | "false"]  `
>当用户离开 Activity 并且其在屏幕上不再可见时，是否应从 Activity 堆栈中将其移除并完成（调用其 finish() 方法）—“true”表示应将其完成，“false”表示不应将其完成。 默认值为“false”。

>`“true”一值表示 Activity 不会留下历史轨迹。 它不会留在任务的 Activity 堆栈内，因此用户将无法返回 Activity。 在此情况下，如果您启动另一个 Activity 来获取该 Activity 的结果，系统永远不会调用 onActivityResult()。`

>该属性是在 API 级别 3 引入的。

`考虑以下场景：应用在A界面需要调用系统的图库（界面B）来选择图片，在选择完图片后进入到下一个Activity C。此时，系统的图片选择Activity B就应该设置noHistorytrue，否则在Activity C中按下back键时，将返回到系统图片选择页面B。`

---
`2.6  android:taskAffinity="string"`
>与 Activity 有着亲和关系的任务。从概念上讲，具有相同亲和关系的 Activity 归属同一任务（从用户的角度来看，则是归属同一“应用”）。 任务的亲和关系由其根 Activity 的亲和关系确定。
>亲和关系确定两件事 - Activity 更改到的父项任务（请参阅 allowTaskReparenting 属性）和通过 FLAG_ACTIVITY_NEW_TASK 标志启动 Activity 时将用来容纳它的任务。

>默认情况下，应用中的所有 Activity 都具有相同的亲和关系。您可以设置该属性来以不同方式组合它们，甚至可以将在不同应用中定义的 Activity 置于同一任务内。 要指定 Activity 与任何任务均无亲和关系，请将其设置为空字符串。

>如果未设置该属性，则 Activity 继承为应用设置的亲和关系（请参阅 <application> 元素的 taskAffinity 属性）。 应用默认亲和关系的名称是 <manifest> 元素设置的软件包名称。

manifest文件中application节点也可设置该属性，当设置application节点中的该属性时，当前应用所有的Activity的taskAffinity的默认值都是application中的这个属性值。每个Activity都可以重新该属性来覆盖application中的这个属性。
当application中不设置该属性时，默认为应用的包名。

---
#### 三. Intent中跟Activity和Task相关的flag
`3.1 public static final int FLAG_ACTIVITY_NO_HISTORY = 0x40000000;`
通过设置该flag启动的Activity，不会在当前的Task中留下记录。比如A -> B -> C，假设A启动B时设置了该flag，则从B启动完C之后，B会销毁，当前的Task中只存在A和C。`等同于activity的android:noHistory="true"。`


---
`3.2  public static final int FLAG_ACTIVITY_SINGLE_TOP = 0x20000000;`
  等同于launchMode中的singleTop

---
`3.3   public static final int FLAG_ACTIVITY_NEW_TASK = 0x10000000;`
设置此状态，记住以下原则，首先会查找是否存在和被启动的Activity具有相同的亲和性的任务栈，如果有，刚直接把这个栈整体移动到前台，并保持栈中的状态不变，即栈中的activity顺序不变，如果没有，则新建一个栈来存放被启动的activity
考虑以下场景：
AppA: A -> B -> C
AppB: M - > B（AppA的B）
在AppB的M中启动B时，intent设置了FLAG_ACTIVITY_NEW_TASK 属性：
* 当AppA已经启动（假设当前在A界面），此时在AppB中在M界面启动的B会在AppA的Task中创建（AppA Task的affinity与B的affinity相同），在界面B按下back，此时出现的是界面A，而不是界面M。
* 当AppA未启动时，当前不存在与B的taskAffinity相同的Task，因此此时会新建一个Task在存放新建的B。
`使用这个标志并不意味着一定会在一个新的Task中创建Activity，只有在当前的taskAffinity与要启动的Activity的taskAffinity不同时，才会在一个新的Task中启动Activity。`

---
`3.4 public static final int FLAG_ACTIVITY_MULTIPLE_TASK = 0x08000000;`

---
`3.5 public static final int FLAG_ACTIVITY_CLEAR_TOP = 0x04000000;`
等同于launchMode中的singleTask

 ---
`3.6 public static final int FLAG_ACTIVITY_FORWARD_RESULT = 0x02000000;`

---

`3.7 public static final int FLAG_ACTIVITY_PREVIOUS_IS_TOP = 0x01000000;`

---
`3.8 public static final int FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS = 0x00800000;`

---
`3.9 public static final int FLAG_ACTIVITY_BROUGHT_TO_FRONT = 0x00400000;`

---
`3.10  public static final int FLAG_ACTIVITY_RESET_TASK_IF_NEEDED = 0x00200000;`

---
`3.11 public static final int FLAG_ACTIVITY_LAUNCHED_FROM_HISTORY = 0x00100000;`
  
  ---
`3.12  public static final int FLAG_ACTIVITY_CLEAR_WHEN_TASK_RESET = 0x00080000;`
 
 ---
`3.13  public static final int FLAG_ACTIVITY_NEW_DOCUMENT = FLAG_ACTIVITY_CLEAR_WHEN_TASK_RESET;`

---
`3.14  public static final int FLAG_ACTIVITY_REORDER_TO_FRONT = 0X00020000;`
如果设置了这个标志，而且被启动的Activity如果已经在运行，那这个Activity会被调到栈顶。
比如，一个任务中有4个Activity：A，B，C，D。如果D调用了startActivity() 来启动B时使用了这个标志，那B就会被调到历史栈的栈顶，结果顺序：A，C，D，B，否则顺序会是：A，B，C，D，B。 如果使用了标志 FLAG_ACTIVITY_CLEAR_TOP，那这个FLAG_ACTIVITY_REORDER_TO_FRONT标志会被忽略。
 
 ---
`3.15  public static final int FLAG_ACTIVITY_NO_ANIMATION = 0X00010000;`
切换Activity时禁用动画。

---
`3.16  public static final int FLAG_ACTIVITY_CLEAR_TASK = 0X00008000;`
  
  --- 
`3.17  public static final int FLAG_ACTIVITY_TASK_ON_HOME = 0X00004000;`
比如，A->B->C->D，如果在C启动D的时候设置了这个标志，那在D中按Back键则是直接回到桌面，而不是C。
`注意：
只有D是在新的task中被创建时（也就是D的launchMode是singleInstance时，或者是给D指定了与C不同的taskAffinity并且加了FLAG_ACTIVITY_NEW_TASK标志时），使用 FLAG_ACTIVITY_TASK_ON_HOME标志才会生效。`
  
  ---
`3.18  public static final int FLAG_ACTIVITY_RETAIN_IN_RECENTS = 0x00002000;`

