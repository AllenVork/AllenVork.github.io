---
layout:     post
title:      android 四种启动模式
subtitle:   主要介绍不同的 task 的 activity 的启动返回问题
author:     Allen Vork
header-img: img/pen1.jpg
catalog: true
tags:
    - android basics    
---

## standard
该模式为 activity 的标准启动模式，每次启动都会新创建一个 activity。新启动的 activity　会被放在启动该 activity 所在的栈的栈顶。

## singleTop
如果**调用者**的 Task 栈顶已经有相同类型的 activity 实例的话，则不会创建新的实例，而是通过 onNewIntent 方法将 intent 传入这个已经存在的 activity 中。

## singleTask 
该模式的 activity 在**系统中**只允许存在一个实例，类似单例模式。framework 在启动该模式的 activity 时只会将它标志为可以在一个新的 task 中启动。默认情况下是在启动它的 activity 所在的 task 中创建，当给该 activity 指定 android:taskAffinity = "新 task 名" 时才会在这个指定的任务中。    
如果是在 **相同的应用程序** 中启动的话，如果该 activity 已经存在系统中，则会将该 activity 上面的 activity 都 pop 出栈，让该 activity 成为栈顶，并调用 onNewIntent。如果不存在该实例的话，则会创建一个新的 Activity 放到栈顶；    
如果是在 **不同的应用程序** 中启动的话，如果该 activity 已经被创建，则将该 activity 所在的栈上面的 activity 都销毁让其位于栈顶。如果该 activity 不存在，则有两种情况：该 activity 所在的进程没有 task 则会创建一个新的 task 并将之放入；如果它的进程存在 task 则会放到该 task 的栈顶。点击返回的逻辑都一样：点击返回不是回到启动该 activity 的那个进程页面，而是将该 activity 所在栈的 activity 全部返回后才会回到启动它的应用的页面。

## singleInstance
这种模式很接近 singleTask，只允许系统中存在一个 Activity 实例。不同之处在于拥有这个 Activity 的 Task 只能包含一个 Activity 实例。点击返回是返回到启动该 activity 的页面。

## 不同应用间调用 activity 示例
> singleTop 和 standard

![]({{site.url}}/img/android/basic/launchmode/singletop.png) 
无论该 activity 是否（以任何形式）存在，都会创建一个新的 activity 压入调用者所在的 task 的栈顶。singleTop 仅仅作用于同一个栈中，即使该 activity 在栈顶，如果是从另一个栈启动该 activity 的话，依然会重新创建一个新的。

> singleTask

![]({{site.url}}/img/android/basic/launchmode/singletask.png) 
第一个例子：启动前应用 B 已经有了 3 个 Activity，启动后会将 activity 3 移除，然后 activity 2 执行相应的生命周期事件并调用 onNewIntent。    
第二个例子：当 activity 2 没有创建，但是 activity 2 所在的栈存在（一个 app 默认会创建一个与包名同名的栈，上面第二个例子中应用 B 已经创建了 MainActivity，那么就已经拥有了一个默认栈），如果没有通过 android:taskAffinity="taskName" 指定一个新的栈的话，所启动的 singleTask 的 activity 就会直接放到栈顶。

## 使用 adb shell dumpsys 检测 Android 的 Activity 任务栈
`adb shell dumpsys` 可以获得系统的所有服务信息。我们可以通过 `adb shell dumpsys -l` 查看有哪些正在运行的服务。    
我们要查看 activity 的信息，则使用 `adb shell dumpsys activity `可以得到关于设备非常长的一段讯息。它有很多类别，每一个类别都有一个括号内容，给出了更加详细的指令来查看该类别更加详细的内容。我们想得到 activity 的详细信息，于是可以运行 `db shell dumpsys activity activities` 得到当前所有在运行的任务栈。关于栈的内容就在”Running activities (most recent first)”这部分。当然，我们也可以直接使用`adb shell dumpsys activity activities | sed -En -e '/Running activities/,/Run #0/p'`直接打印出”Running activities”列表。如：

![]({{site.url}}/img/android/basic/launchmode/activitytask.png) 
可以看到 taskid 为206的栈里有2个 Activity，Main2Activity 在栈顶，MainActivity在栈底。    

> 常用命令：    

> + adb shell dumpsys activity top---------------查看当前显示的 activity 的名字，还可以显示包含的 Fragment 的信息。
> + adb shell dumpsys activity-------------------查看ActvityManagerService 所有信息
> + adb shell dumpsys activity activities--------查看Activity组件信息
> + adb shell dumpsys activity services----------查看Service组件信息
> + adb shell dumpsys activity providers---------查看ContentProvider组件信息
> + adb shell dumpsys activity broadcasts--------查看BraodcastReceiver信息
> + adb shell dumpsys activity intents-----------查看Intent信息
> + adb shell dumpsys activity processes---------查看进程信息

## 上面提的到 task 到底是什么？
task 是一组相关联的 activity 的集合，用于管理界面的跳转和返回。它是存在于 back stack（后压栈）中。它是可以跨应用的，即不同的应用的 activity 可以在同一个 task 中。

## taskAffinity 相关属性
**taskAffinity** 指的是 activity 的归属，即 activity 属于哪个 task。如果 activity 没有显式地指明 taskAffinity，那么它的这个属性就等于 Application 指明的 taskAffinity。如果 Application 也没有指明，则 taskAffinity 的值等于应用的包名。     
taskAffinity 可以与 **allowTaskReparenting** 结合使用。allowTaskReparenting 用于标记 activity 是否从启动的 Task 移动到 taskAffinity 所指定的 task 中。true 表示会移动，false 表示不会。该属性的特性在于当你启动该模式的 activity 的时候，会在启动者的 task 栈顶（并不是 taskAffinity 所指定的 task 中）。但是一旦有与被启动的 activity 的 taskAffinity 相同的 task 切换的前台的时候，这个被启动的 activity 会被移动到 taskAffinity 所指定的栈中。    
**allowTaskReparenting 只有在发生 resettask 的时候才会生效**。而我们从桌面上启动的程序都带了`FLAG_ACTIVITY_RESET_TASK_IF_NEEDED`。所以从home上启动程序会进行task reset.也就会使用到这个allowTaskReparenting。    
下面看一个例子：    
![]({{site.url}}/img/android/basic/launchmode/allowtaskreparenting.png) 