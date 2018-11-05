---
layout:     post
title:      主要介绍 VSYNC、三重缓存
subtitle:   
header-img: img/android3.png
author:     Allen Vork
catalog: true
tags:
    - performance
---

## Synopsis
Google 在 Android4.1 提出了 Project Butter 用于提升系统流畅度。Project Butter 对 Android Display 系统进行了重构，引入了 VSYNC（垂直同步）、Triple Buffer（三重缓存） 和 Choreographer。
![]({{site.url}}/img/android/basic/performance/ProfileGpuRendering/gpu.jpg)

## Problems
在一个典型的显示系统中，一般包括 CPU、GPU、display 这3个部分。CPU 用于计算数据然后交给 GPU 进行渲染，渲染好后放到 buffer 中存起来，然后 display 会将 buffer 里的数据显示到屏幕上。但显示后会出现2种问题：

### tearing
即撕裂。当 CPU/GPU 将数据准备好存入 buffer 中，但 display 还没来得及显示，这时 CPU/GPU 把下一帧的数据往 buffer 中写，还没写完的时候，display 开始读取 buffer 来显示（也就是绘图速度大于显示速度）。这时就会出现显示的上半部分是下一帧的数据，下半部分为上一帧的数据，就是所说的撕裂。

### jank
绘图速度过慢的时候，同一帧在屏幕上至少出现2次。

## Solutions
### tearing
撕裂的原因是 display 还没来得及读 buffer 就被重写了，那么就可以准备2个 buffer 即双缓冲。back buffer 用于 CPU/GPU 后台绘制，frame buffer 用于显示。back buffer 准备好后才可以交换，这样就可以避免撕裂问题。但是此时屏幕还没有完整显示上一帧的内容时是不能交换的。那么只有等屏幕处理完成当前帧才能进行交换操作。当扫描完一屏后，会回到第一行进入下一次的循环，中间会有一段空隙（VBI），这个空隙为缓冲区交换的最佳时间。VSYNC 就是利用这个空隙出现的垂直刷新脉冲来保证双缓冲的最佳时间点。

### jank
我们来看下在双缓冲下，没有 VSYNC 的情况：
![]({{site.url}}/img/android/basic/performance/ProfileGpuRendering/1.webp)
Display 为显示屏， VSYNC 仅仅指双缓冲的交换。我们来看下将会发生的异常：
+ Step1：Display 显示第0帧，此时 CPU/GPU 渲染第1帧画面，并且在 Display 显示下一帧前完成。
+ Step2：Display 正常渲染第一帧。
+ Step3：出于某种原因，如 CPU 资源被占用，系统没有及时处理第2帧数据，当 Display 显示下一帧时，由于数据没处理完，所以依然显示第1帧，即发生“Jank”。    
上图出现的情况就是第2帧没有在显示前及时处理，导致屏幕多显示第一帧一次，导致后面的帧都延时了。那么如何让第2帧及时绘制呢？    
![]({{site.url}}/img/android/basic/performance/ProfileGpuRendering/2.webp)
可以看出，当且仅当 VSYNC 出现时，CPU 就会立即处理下一帧数据，大大降低了 Jank 的概率。而且也杜绝了 CPU/GPU 不停的绘制，导致帧生成速度高于屏幕刷新速度，生成的帧不能显示而被丢弃，这样导致的丢帧情况。引入 VSYNC 后，绘制速度和屏幕刷新速度保持一致了。现在 Android 设备的屏幕刷新频率为 60HZ，那么 CPU/GPU 渲染的时间需要在 16ms 内。当 CPU/GPU 的 FPS 高于 60 HZ 显示效果会很完美，如果设备硬件性能较差，无法达到这个要求会出现什么情况呢？
![]({{site.url}}/img/android/basic/performance/ProfileGpuRendering/3.webp)
我们先来看下正常情况，A 和 B 分别代表2个缓冲区。整个过程很顺滑。现在来看下FPS低于屏幕刷新率的情况：
![]({{site.url}}/img/android/basic/performance/ProfileGpuRendering/4.webp)
可以看出当第1个 VSYNC 到来时 GPU 还在处理数据，这时 B 缓冲区被占用了，那么就无法进行交换，屏幕依然显示 A 缓冲区的数据。下一个信号到来时，此时 GPU 已经处理完了，那么就可以交换缓冲区，此时屏幕显示 B 缓冲区，CPU/GPU 开始操作 A。下一个信号到来时，A 被占用，那么屏幕依然显示 B 的数据。这种情况就是因为 GPU/CPU 无法在 16ms 内处理完数据而导致缓冲区交换延迟。    
那么有没有办法避免呢？    
因为设备不能升级硬件，我们无法改变 CPU/GPU 渲染的时间，那么第一次 Jank 是无法避免的。我们重点关注 CPU 第一次和第二次执行中间浪费的时间。当第1次信号到来时，由于 GPU 占用了 B，导致屏幕会一直占用 A。两个缓冲区都被占用了，即使此时 CPU 是空闲的，它也没有办法处理下一帧的数据。如果增加一个 buffer,会不会有所改善？
![]({{site.url}}/img/android/basic/performance/ProfileGpuRendering/5.webp)
当第一个信号到来时，A、B 都被占用，此时 CPU 开始使用 C 缓冲区来处理下一帧数据。之前第二次发生的 Jank 就避免了。有效的降低了显示错误的几率。可以看出双缓冲和三重缓冲都会有 lag（延时）问题。C 缓冲区延时了16ms才显示。

## skip frames
前面讲的其实都是解决撕裂和同一帧出现多次的情况。但是所有帧都能得到绘制，只不过是延时。那么什么情况会造成丢帧呢，查过


## References
+ [Inspect GPU rendering speed and overdraw](https://developer.android.com/studio/profile/inspect-gpu-rendering)   
+ [Analyze with Profile GPU Rendering](https://developer.android.com/topic/performance/rendering/profile-gpu) 