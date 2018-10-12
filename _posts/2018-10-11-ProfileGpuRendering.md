---
layout:     post
title:      GPU 呈现模式分析
subtitle:   
header-img: img/android3.png
author:     Allen Vork
catalog: true
tags:
    - performance
---

## Synopsis
GPU 呈现模式分析工具展示了一个滚动的直方图来展示它渲染每一帧的时间。如下图所示，它展示了一个直方图，中间的横线就是这个屏幕刷新频率16fps，我们最好将柱子都控制在它之下。
![]({{site.url}}/img/android/basic/performance/ProfileGpuRendering/gpu.jpg)


## Enable the profiler
按如下步骤来启用设备的 GPU 渲染分析工具：
+ 进入**设置** -> **开发者选项**
+ 在**监控**区域点击**GPU 呈现模式分析**
+ 选择**在屏幕上显示为条形图**
+ 打开你要分析的 app

## Inspect the output
分析柱状图前，我们首先需要注意以下几点：
+ 每一根柱子代表的是一帧，他的高度则代表渲染这一帧的时间。
+ 绿色的横线代表16ms，为了达到60fps的屏幕刷新频率，绘制每一帧的时间都应控制在绿线之下。
+ 每一根柱子都由不同的颜色，这些颜色都对应着渲染中不同的阶段

我们来看下这些颜色代表的含义：    
![]({{site.url}}/img/android/basic/performance/ProfileGpuRendering/2.jpg)

+ Swap Buffers：代表 CPU 等待 GPU 完成它的工作的时间。如果它比较高，则代表 GPU 执行了太多的工作。
+ Command Issue：代表2D渲染器向 OpenGL 发起的绘制和重新绘制显示列表的命令的时间。它的高度与执行每个显示列表的时间的总和成正比。
+ Sync & Upload：代表它上传 bitmap 信息给 GPU 的时间。 加载的 bitmap 越大越多，那么它占用的时间就会越长。
+ Draw：代表它创建和更新显示列表的时间（即所有 onDraw() 执行的时间）。如果它比较高，则可能由许多自定义视图绘制，或者 onDraw() 占用的时间比较多。
+ Measure / Layout：代表 onLayout() 和 onMeasure() 所执行的时间。
+ Animation：表示评估该帧的所有动画所花费的时间。如果耗时较长则代表可能使用了低性能的自定义动画，或者是因为更新了属性而导致了额外的工作。
+ Input Handling：代表应用执行输入回调的时间。它较长的话，代表应用耗费很多时间处理用户输入，可以考虑将这些处理放到异步线程。
+ Misc Time / VSync Delay：表示应用执行两帧之间的操作所花费的时间。它较长的话，可能是 UI 线程执行了本可以分流到子线程中的耗时的工作。


## References
+ [Inspect GPU rendering speed and overdraw](https://developer.android.com/studio/profile/inspect-gpu-rendering)   
+ [Analyze with Profile GPU Rendering](https://developer.android.com/topic/performance/rendering/profile-gpu) 