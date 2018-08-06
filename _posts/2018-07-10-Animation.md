---
layout:     post
title:      Android 动画基础
subtitle:   主要介绍动画的几个种类以及基本使用
header-img: img/android3.png
author:     Allen Vork
catalog: true
tags:
    - Data Structures
---

## Synopsis
Android 动画分为3种：    
+ View 动画：    
	+ 补间动画：通过对图像进行平移、旋转等产生动画效果。   
	+ 帧动画：通过顺序播放一系列图像从而产生动画效果。    
+ 属性动画：动态地改变对象的属性从而达到动画效果。

> 问题来了，为什么有 View 动画，还要推出属性动画？    
View 动画（主要是补间动画）机制本身就比较全面，提供了一系列的动画效果如淡入淡出，旋转等。并且可以借助 AnimationSet 来将四种动画组合起来使用，还可以结合 Interpolator 来控制动画的播放速度等。但是它有很大的**局限性**:补间动画只能作用在 View 上（如果你只是想改变一个变量，然后 invalidate() 让 onDraw() 去绘制从而产生动画效果的话就不行了）；它仅仅改变的是 View 的显示效果，而不会真正去改变 View 的属性（例如将一个按钮平移到另一个位置的时候，点击该按钮是无效的，因为补间动画只是将按钮绘制到那个位置而已）；只能实现移动、缩放、旋转和淡入淡出这四种动画操作(像动态改变 View 的背景颜色，这四种操作就无能为力了)。    
出于这些原因，Android 3.0 引入了属性动画。它不再针对 View 设计，而是一种对值进行操作的机制，并将获得到的值赋值到指定对象的属性上。这样就打破了补间动画的限制了。

## 补间动画
支持4种动画效果：平移动画（TranslateAnimation）、旋转动画（RotateAnimation）、缩放动画（ScaleAnimation）、透明度动画（AlphaAnimation）。它们既可以用代码动态创建，也可以通过 xml 文件来定义动画（这种方式可读性更好）。

### 通过 XML 文件定义动画
在 res/anim/ 中创建动画文件，然后定义一个或者一组动画。    
定义一个旋转动画：
```xml
<rotate xmlns:android="http://schemas.android.com/apk/res/android"
    android:fromDegrees="0"
    android:toDegrees="180"
    android:duration="3000"
    android:interpolator="@android:anim/overshoot_interpolator"
    android:fillAfter="true"
    android:repeatCount="2"
    android:repeatMode="reverse"
    android:pivotX="50%"
    android:pivotY="50%"/>
```
定义一个动画组合：
```xml
<set xmlns:android="http://schemas.android.com/apk/res/android"
    android:duration="3000"
    android:interpolator="@android:anim/overshoot_interpolator"
    android:shareInterpolator="true">

    <!--从完全透明到完全不透明-->
    <alpha
        android:fromAlpha="1.0"
        android:toAlpha="0" />

    <!--从控件大小的50%，以控件的中心点为原点，放到到控件大小的三倍-->
    <scale
        android:fromXScale="50%"
        android:fromYScale="50%"
        android:pivotX="0.5"
        android:pivotY="0.5"
        android:toXScale="3"
        android:toYScale="3" />

    <!--从控件左边的100dp位置平移到控件右边100dp位置-->
    <translate
        android:fromXDelta="-100"
        android:fromYDelta="0"
        android:toXDelta="100"
        android:toYDelta="0" />

    <!--以控件的中心为原点旋转180°-->
    <rotate
        android:fromDegrees="0"
        android:pivotX="0.1"
        android:pivotY="0.5"
        android:toDegrees="180" />

</set>
```
根节点可以是 `<set>`,`<alpha>`,`<scale>`,`<translate>`,`<rotate>` 这5种。其中 `<set>` 是放置一组动画的。然后将动画文件设置给 View:
```kotlin
        val anim = AnimationUtils.loadAnimation(this, R.anim.filename)
        animView.startAnimation(anim)
```
### 使用代码方式写动画
```kotlin
class MainActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
//        alpha(animView)
//        translate(animView)
          scale(animView)
    }

    //透明度动画
    fun alpha(view: View) {
        val anim = AlphaAnimation(0f, 1f) //从透明到完全不透明
        setAnimParams(anim)
        view.startAnimation(anim)
    }

    //平移动画
    fun translate(view: View) {
        //构造函数里4个参数的意义是：如果view在A(x,y)点 那么动画就是从B(x+fromXDelta, y+fromYDelta)点移动到C(x+toXDelta,y+toYDelta)点
        val anim = TranslateAnimation(0f, 200f, 0f, 200f)
        setAnimParams(anim)

        view.startAnimation(anim)
    }

    //缩放动画
    fun scale(view: View) {
        //构造函数里参数的意义是：fromX/Y(**方向缩放的起始比例），toX/Y（**方向的最终比例），pivotX/Y:以该点作为参照物进行缩放
        val anim = ScaleAnimation(0f, 2f, 0f, 2f, 15f, 0.5f)
        setAnimParams(anim)
        view.startAnimation(anim)
    }

    private fun setAnimParams(anim: Animation) {
        anim.duration = 2000
        anim.fillAfter = true //动画最后是否停留在终止状态
        anim.repeatCount = 2 //重复次数
        anim.repeatMode = Animation.REVERSE //反转模式
        anim.interpolator = BounceInterpolator() //设置弹簧特效

        //监听动画状态
        anim.setAnimationListener(object : Animation.AnimationListener {
            override fun onAnimationRepeat(animation: Animation?) {
                Log.e("MainActivity", "重复播放动画")
            }

            override fun onAnimationEnd(animation: Animation?) {
                Log.e("MainActivity", "结束动画")
            }

            override fun onAnimationStart(animation: Animation?) {
                Log.e("MainActivity", "开始动画")
            }
        })
    }
}
```

## 帧动画
类似幻灯片，定义好一系列的图片后，挨个播放。
### 通过 XML 文件定义动画
在 res/drawable/ 中创建一个动画文件：
```xml
<animation-list xmlns:android="http://schemas.android.com/apk/res/android"
    android:oneshot="false">
    <!--oneshot: false 代表循环播放，true 表示只播放一次-->

    <item android:drawable="@drawable/1" android:duration="150" />
    <item android:drawable="@drawable/2" android:duration="150" />
    <item android:drawable="@drawable/3" android:duration="150" />
    <item android:drawable="@drawable/4" android:duration="150" />
    <item android:drawable="@drawable/5" android:duration="150" />
    <item android:drawable="@drawable/6" android:duration="150" />
</animation-list>
```
然后加载到 ImageView 中去，android:src="@drawable/filename"。 我们可以通过代码控制动画：
```kotlin
        val drawable = imageView.drawable as AnimationDrawable

        button.setOnClickListener {
            if (drawable.isRunning) {
                drawable.stop();
            } else {
                drawable.start();
            }
        }
```   

### 使用代码方式写动画
```kotlin
        val animDrawable = AnimationDrawable()
        animDrawable.isOneShot = false
        animDrawable.addFrame(resources.getDrawable(R.drawable.a), 350)
        animDrawable.addFrame(resources.getDrawable(R.drawable.b), 350)
        animDrawable.addFrame(resources.getDrawable(R.drawable.c), 350)
        animDrawable.addFrame(resources.getDrawable(R.drawable.d), 350)
        animDrawable.addFrame(resources.getDrawable(R.drawable.e), 350)
        animDrawable.addFrame(resources.getDrawable(R.drawable.f), 350)
        imageView.setImageDrawable(animDrawable)
        animDrawable.start()
```

## 属性动画
属性动画是一种不断对值进行操作的机制，并将值赋值到指定对象的指定属性上。我们只需要告诉它动画的时长、动画的类型、动画的初始和结束值即可。

### ValueAnimator
ValueAnimator 是属性动画最重要的类，属性动画是通过不断对值进行操作实现的，而这个初始值和结束值的过渡就是通过 ValueAnimator 计算的。它是使用时间循环的机制来计算值之间的动画过渡的。

#### 通过 XML 文件定义动画
首先在 res 中创建一个新目录 animator，然后创建一个动画文件如 valueanimator:
```xml
<animator xmlns:android="http://schemas.android.com/apk/res/android"
    android:valueFrom="0"
    android:valueTo="1"
    android:valueType="floatType"
    android:duration="2000"
    android:repeatCount="10"
    android:repeatMode="reverse" />
```
这样就创建了一个值从0变化到1的 float 类型的动画，动画耗时2s,使用反转模式重复10次。
```kotlin
        val animator = AnimatorInflater.loadAnimator(this, R.animator.valueanimator) as ValueAnimator
        animator.setTarget(button)
        animator.addUpdateListener {
            // ValueAnimator 只会改变值，我们需要将计算出来的值手动设置到对应的属性上去
            button.alpha = it.animatedValue as Float
        }
        animator.start()
```

#### 使用代码方式写动画
```kotlin
        val animator = ValueAnimator.ofFloat(0f, 1f)
        animator.duration = 2000
        animator.setTarget(button)
        animator.addUpdateListener {
            button.alpha = it.animatedValue as Float
        }
```

### ObjectAnimator
它是 ValueAnimator 的子类，相对于 ValueAnimator 这个更常用。 ValueAnimator 是对值进行平滑的动画过渡，而 ObjectAnimator 是对指定的对象的指定属性（譬如 ImageView 的 background)进行操作。

#### 通过 XML 文件定义动画
创建方式和 ValueAnimator 相同：
```xml
<objectAnimator xmlns:android="http://schemas.android.com/apk/res/android"
    android:valueFrom="0"
    android:valueTo="1"
    android:propertyName="alpha" //直接指定动画所作用到的属性，而不需要像 ValueAnimator 那样手动赋值
    android:valueType="floatType"
    android:duration="2000"
    android:repeatCount="10"
    android:repeatMode="reverse" />
```
可以看到 ObjectAnimator 的根节点为 objectAnimator，而 ValueAnimator 的根节点为 animator。
```kotlin
        val animator = AnimatorInflater.loadAnimator(this, R.animator.objectanimator) as ObjectAnimator
        animator.setTarget(button)
        animator.start()
```
ObjectAnimator 不需要像 ValueAnimator 那样手动将变化的值设置到相应对象的属性上，直接在动画中指定属性就行了。

#### 使用代码方式写动画
```kotlin
        val objectAnimator = ObjectAnimator.ofFloat(imageView, "alpha", 0f)
        objectAnimator.duration = 1000
        objectAnimator.start()
```
ObjectAnimator 在构造的时候就可以指定动画作用的对象的属性。

### AnimatorSet
用于组合属性动画：

#### 通过 XML 文件定义动画
```xml
<set xmlns:android="http://schemas.android.com/apk/res/android"
    android:ordering="sequentially"><!--ordering 指定播放次序，sequentially 表示从上往下依次播放-->

    <!--先从水平方向的-500的位置平移到原点-->
    <objectAnimator
        android:duration="2000"
        android:propertyName="translationX"
        android:valueFrom="-500"
        android:valueTo="0"
        android:valueType="floatType" />

    <set android:ordering="together">
        <!--然后旋转360°-->
        <objectAnimator
            android:duration="3000"
            android:propertyName="rotation"
            android:valueFrom="0"
            android:valueTo="360"
            android:valueType="floatType" />

        <!--旋转的同时，先从透明变化到不透明，再从不透明变化到透明-->
        <set android:ordering="sequentially">
            <objectAnimator
                android:duration="1500"
                android:propertyName="alpha"
                android:valueFrom="1"
                android:valueTo="0"
                android:valueType="floatType" />
            <objectAnimator
                android:duration="1500"
                android:propertyName="alpha"
                android:valueFrom="0"
                android:valueTo="1"
                android:valueType="floatType" />
        </set>
    </set>
</set>
```
然后像上面一样，通过 AnimatorInflator 去加载就行了:
```kotlin
        val animator = AnimatorInflater.loadAnimator(this, R.animator.objectanimator)
        animator.setTarget(button)
        animator.start()
```

#### 使用代码方式写动画
```kotlin
        val animatorA = ObjectAnimator.ofFloat(button, "TranslationX", -300f, 300f, 0f).setDuration(1000)
        val animatorB = ObjectAnimator.ofFloat(button, "scaleY", 0.5f, 1.5f, 1f).setDuration(1000)
        val animatorC = ObjectAnimator.ofFloat(button, "rotation", 0f, 270f, 90f, 180f, 0f).setDuration(1000)

        val animatorSet = AnimatorSet()
        animatorSet.play(animatorA).after(animatorB).with(animatorC)
        animatorSet.start()
```
animatorA 在 animatorB 之后播放，animatorA 播放的同时播放 animatorC

### ViewPropertyAnimator 
我们知道，属性动画不再针对 View 进行设计，但是大多数情况下，我们都是对 View 做动画，为了提供更友好的 API，Android 3.1 系统中新增了 ViewPropertyAnimator——专门为 View 设计的属性动画。    
如果我们想让一个 textView 变透明，一般会使用属性动画这样写：    
```java
ObjectAnimator animator = ObjectAnimator.ofFloat(textview, "alpha", 0f);  
animator.start();  
```
如果使用 ViewPropertyAnimator 的话，就可以简化为：    
```java
textview.animate().alpha(0f);
```
用法比较简单，调用 View 的 animate() 方法就会返回 ViewPropertyAnimator 对象，然后调用里面的 translate 等方法。动画都是采用连缀的方式，当动画定义完成后才会启动动画。

## 参考文献
+ [Android 动画](https://www.jianshu.com/p/d2912e5402bc)    