---
layout:     post
title:      判断应用被强杀
subtitle:   
header-img: img/post-bg-universe.jpg
author:     Allen Vork
catalog: true
tags:
    - android
    - android basic    
---

## 原理
应用在后台被强杀时，该应用的整个进程都被销毁了，但是 Activity 栈并没有被清掉。那么我们点击桌面图标进入应用时，会重新创建 Application, 然后创建栈顶的 Activity。 

## 如何判断
既然强杀后，进入应用只会初始化 Application 和栈顶 Activity，那么我们只需要在 Application 中创建一个静态变量，然后在闪屏页面去修改该值。后面的 Activity 判断如果值被修改了则没有强杀，如果没有被修改说明没有走闪屏页面的流程，说明之前被杀掉了。
```java
class MyApplication : Application() {
    companion object {
        var isForceKilled = true
    }
}
```

创建一个 BaseActivity，将应用是否被杀掉的判断放进去，避免所有的 Activity 都要自行判断
```java
open class BaseActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        if (MyApplication.isForceKilled)
            Toast.makeText(this, "Application was force killed", Toast.LENGTH_LONG).show()
    }
}
```

闪屏页（在每次应用正常启动的时候都会创建，而进程在后台被杀掉是不会创建的： 
```java
class SplashActivity : BaseActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        //走到闪屏页说明应用被正常启动，之前并没有被杀死
        MyApplication.isForceKilled = false
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        Handler().postDelayed({
            startActivity(Intent(this@SplashActivity, LoginActivity::class.java))
            finish()
        }, 2000)
    }
}

```

正常页面：
```java
class LoginActivity : BaseActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_login)
    }

    //-------- 应用被强杀后，下面的生命周期方法都不会被调用---------

    override fun onSaveInstanceState(outState: Bundle?) {
        super.onSaveInstanceState(outState)
    }

    override fun onPause() {
        super.onPause()
    }

    override fun onStop() {
        super.onStop()
    }

    override fun onDestroy() {
        super.onDestroy()
    }
}
```
操作：点击应用，启动后会初始化 Application 将 isForceKilled 设为 true，然后自动启动 SplashActivity ，SplashActivity 会将其设为 false。然后 2s 后 SplashActivity 会启动 LoginActivity。然后我们按 Home 键将应用放到后台，点击下面的“×”将应用杀掉:
![]({{site.url}}/img/android/basic/appforcekilled/dismiss.png)    
然后再点击桌面上应用的图标，它会初始化 Application 将 isForceKilled 设为 true，然后再创建栈顶的 LoginActivity。由于没有创建 SplashActivity，isForceKilled 为 true。那么就可以知道是被强杀了。

## 如何避免应用被强杀导致的问题
由于强杀后，它只会创建栈顶 Activity，那么如果该 Activity 使用了之前创建的 Activity 的一些参数就会出现空指针之类的问题。如果没有好的办法处理的话，可以在让它重新走一遍流程，即启动了栈顶 Activity 后，直接跳转到 MainActivity。

```java
open class BaseActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        if (MyApplication.isForceKilled) {
            startActivity(Intent(this, SplashActivity::class.java))
            Toast.makeText(this, "Application was force killed", Toast.LENGTH_LONG).show()
        }
    }
}
```