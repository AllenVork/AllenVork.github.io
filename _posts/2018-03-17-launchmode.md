# android 四种启动模式

## standard
该模式为 activity 的标准启动模式，每次启动都会新创建一个 activity。新启动的 activity　会被放在启动该 activity 所在的栈的栈顶。

## singleTop
如果**调用者**的 Task 栈顶已经有相同类型的 activity 实例的话，则不会创建新的实例，而是通过 onNewIntent 方法将 intent 传入这个已经存在的 activity 中。

## singleTask 
该模式的 activity 在**系统中**只允许存在一个实例，类似单例模式。    
如果是在 **相同的应用程序** 中启动的话，如果该 activity 已经存在系统中，则会将该 activity 上面的 activity 都 pop 出栈，让该 activity 成为栈顶，并调用 onNewIntent。如果不存在该实例的话，则会创建一个新的 Activity 放到栈顶；    
如果是在 **不同的应用程序** 中启动的话，如果该 activity 已经被创建，则将该 activity 所在的栈上面的 activity 都销毁让其位于栈顶。如果该 activity 不存在，则有两种情况：该 activity 所在的进程没有 task 则会创建一个新的 task 并将之放入；如果它的进程存在 task 则会放到该 task 的栈顶。点击返回的逻辑都一样：点击返回不是回到启动该 activity 的那个进程页面，而是将该 activity 所在栈的 activity 全部返回后才会回到启动它的应用的页面。

## singleInstance
这种模式很接近 singleTask，只允许系统中存在一个 Activity 实例。不同之处在于拥有这个 Activity 的 Task 只能包含一个 Activity 实例。点击返回是返回到启动该 activity 的页面。

## 不同应用间调用 activity 示例
> singleTop 和 standard

![]({{site.url}}/img/android/basic/launchmode/singletop.png) 
无论该 activity 是否（以任何形式）存在，都会创建一个新的 activity 压入调用者所在的 task 的栈顶。

> singleTask

![]({{site.url}}/img/android/basic/launchmode/singletask.png) 

## 使用 adb shell dumpsys 检测 Android 的 Activity 任务栈
使用 `adb shell dumpsys activity `可以得到关于设备非常长的一段讯息。它有很多类别，每一个类别都有一个括号内容，给出了更加详细的指令来查看该类别更加详细的内容。我们想得到 activity 的详细信息，于是可以运行 `db shell dumpsys activity activities` 得到当前所有在运行的任务栈。关于栈的内容就在”Running activities (most recent first)”这部分。当然，我们也可以直接使用`adb shell dumpsys activity activities | sed -En -e '/Running activities/,/Run #0/p'`直接打印出”Running activities”列表。如：

```
    Running activities (most recent first):
      TaskRecord{5c74055 #101 A=com.example.allen.demo U=0 StackId=1 sz=1}
        Run #2: ActivityRecord{d30501e u0 com.example.allen.demo/.MainActivity t101}
      TaskRecord{f129b5b #99 A=com.example.allen.launchmode U=0 StackId=1 sz=1}
        Run #1: ActivityRecord{caaed3f u0 com.example.allen.launchmode/.Main2Act                                 ivity t99}
      TaskRecord{fd59237 #98 A=com.example.allen.launchmode U=0 StackId=1 sz=1}
        Run #0: ActivityRecord{909cddc u0 com.example.allen.launchmode/.MainActi                                 vity t98}
```
可以看出有2个进程，3个 task stack 分别为 101 99 98。 每个 task stack 中都只有一个 activity。