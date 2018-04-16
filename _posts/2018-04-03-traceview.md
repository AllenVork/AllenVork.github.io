---
layout:     post
title:      TraceView 解决界面卡顿
subtitle:   TraceView 找到耗时方法
header-img: img/pen1.jpg
author:     Allen Vork
catalog: true
tags:
    - android
    - android performance    
---
## TraceView 是什么
它是一个数据采集和分析工具，当我们界面出现卡顿的时候，可以通过它找到出现问题的方法。一般我们会关注2个问题： **哪些方法很耗时**，**哪些方法调用的次数很多**。方法很耗时的话，会造成界面卡顿的现象，而方法不耗时但调用的次数过多会导致 CPU 频繁调用导致手机发热问题。

## 怎样判断一个方法是否耗时
我们知道，人眼与大脑之间的协作无法感知超过 60fps 的画面更新，所以应用程序是否达到 60fps（GPU 1s 要绘制60帧，也就是每隔 16ms 要绘制一次）是衡量 APP 性能的标准。Android 系统通过每隔16ms发出 VSYNC 信号来触发对 UI 的渲染，那么程序中的大多数操作都要在 16ms 内完成。如果你的操作需要 30ms,那么系统在得到信号时无法进行渲染导致丢帧，则用户在32ms内看到的是同一个画面，这就导致了卡顿。 布局复杂（多层嵌套，绘制的元素过多），无法在 16ms 内完成绘制都会引起卡顿的。那么我们就看耗时超过16ms的方法是否能优化。

## 使用 TraceView
> 方式1：直接通过工具找出卡顿时所有执行的方法的耗时    

在 Android Studio 的菜单上点击 `Tools` -> `Android` -> `Android Device Monitor` 选中需要调试的应用，点击下图中的`Start Method Profiling`然后开始复现卡顿，然后再次点击该按钮停止抓取。这里在实际操作的时候最好快准狠，也就是在最短的时间内复现出卡顿，这样才容易分析，如果觉得确实找不到，不妨多抓几次。
![]({{site.url}}/img/android/basic/traceview/adm.png)
完成后会看到 traceview 分析面板。它有2个部分：    
+ 时间面板部分：每个线程为一行，每行是该线程每个方法执行的时间，不同的方法颜色不同
+ 分析面板部分：展示所有线程的方法的各项指标

我们来看看分析面板部分：
![]({{site.url}}/img/android/basic/traceview/traceview1.png)
我们看到每一个方法的 Incl Cpu Time% 等信息，下面来看看它的意义：
+ InCl Cpu Time: Cpu 执行该方法（包括方法内执行的子方法）的时间
+ Incl Cpu Time%：上述时间占 Cpu 总执行时间的百分比
+ Excl Cpu Time: Cpu 执行该方法（不包括方法内执行的子方法）的时间
+ Excl Cpu Time%：上述时间占 Cpu 总执行时间的百分比
+ Incl Real Time: 从执行该方法（包括子方法）到执行完所花费的总时间
+ Incl Real Time%: 上述时间占总运行时间的百分比
+ Excl Real Time: 从执行该方法（不包括子方法）到执行完所花费的总时间
+ Excl Real Time%: 上述时间占总运行时间的百分比
+ Calls+RecurCalls/Total: 方法被调用的次数 + 递归次数
+ Calls/Total: 调用次数和总次数的占比
+ Cpu Time/Call： 该方法平均占用 Cpu 的时间
+ Real Time/Call: 该方法平均执行时间

我们首要关心的是 `Cpu Time/Call` 即单个方法的耗时。其次是 `Calls + Recur Calls/Total` 即调用次数过多的方法。这里我们点击 `Cpu Time/Call` 让其按照方法执行的时间从高到低排序，然后找到我们代码中写的比较耗时的方法。

> 方式2：直接在可能出现耗时的代码片段头部和尾部注入代码，缩小范围    

```java
Debug.startMethodTracing(“trace”); //参数为生成的文件名
//可能出现耗时的代码
Debug.stopMethodTracing();
```
加入代码后，可以直接操作手机，让其执行这段代码，然后就会生成 trace 文件。然后将其导入到 pc 上:`adb pull sdcard/trace.trace    C:\Users\wangjing\Desktop`。然后打开 Android Device Monitor -> `File` -> `Open File` 打开 traceview 文件即可。

**Caution:**实际操作过程中，特别是第一次用 Trace View 可能会出现分析不出耗时的方法（抓到的耗时方法其实是在子线程中执行的，或者是 onMeasure 之类的方法里的耗时但是该方法确实没做什么操作），这时要多抓几次，我抓了好几次才真正抓到耗时的方法（消耗了140ms)。