---
layout:     post
title:      进程保活
subtitle:   进程保活
header-img: img/android3.png
author:     Allen Vork
catalog: true
tags:
    - android basics  
---

由于现在的 android 系统越来越“智能”了，几乎市面上所有的进程保活都渐渐失效，像微信之类的通过手机管家清理后就再也醒不过来了。我们可以学习一下这些进程保活机制来了解一下进程的相关知识。

## 为啥需要进程保活
其实进程保活不仅占用内存还耗电，感觉有点流氓。但是有些存在感不高的应用可能用户安装一次就不会打开了，但是如果进程被保活的话，就可以接受推送，引导用户使用，同时也可以收集用户的一些位置等信息来进行分析。像微信这种需要接受消息的应用就很需要保活了。

## 有哪些进程保活的方式
+ 进程间相互唤醒
	+ 广播：通过监听系统广播（如网络切换，拍照，拍视频等），android 7.0 上已经将这些广播给取消了，所以无效
	+ 接入第三方 sdk 唤醒：譬如我们接入了 qq 的登陆 sdk，那么我们进程启动了的话，该 sdk 就会去唤醒 qq
	+ 串通唤醒：譬如说腾讯有多个应用，那么他们就可以串通好启动了任何一个我就把其他的都拉起来

+ 启动前台 Service：
	+ 在通知栏显示通知：像 qq 音乐等创建了一个有通知栏的服务优先级会很高 ，那么即使应用在后台也不会被轻易杀死
	+ 不显示通知：它是利用系统漏洞来启动一个前台服务，但是会隐藏掉通知。他的实现方式有2种
		+ API < 18时，启动前台 Service 时直接传一个 new Notification()
		+ API >= 18时，像上面启动了一个前台 Service 后，再启动另一个前台 Service，两个 Service 的 id 要相同，然后停掉 

## 进程回收机制
要知道怎样去保活，那就要先去了解进程的回收机制了。我们知道，除非手动杀掉进程，当我们按返回键返回到桌面的话，进程没有被杀死，而是被缓存起来方便热启动。当开启的进程越来越多的时候，系统就要根据它的回收机制（Low Memory Killer）开杀了。它是根据进程的优先级来决定杀的先后顺序，而这个优先级是由 oom_adj 决定的。它是由 linux 内核分配给每个进程的一个值，值越大则优先级越低越容易被杀死。那么怎么查看进程的 oom_adj 呢？      
```vi
$ adb shell ps | grep com.a.b.c
u0_a121   12212 508   2475320 344332 SyS_epoll_ 7c9bcfa924 S com.a.b.c
```
可以得到进程的 id 如上面的12212。然后我们来获取该进程的 oom_adj。    
```vi
$ adb shell cat /proc/12212/oom_adj
11
```
不同手机厂商该值可能会不同。

## 那么如何减小 oom_adj 来增加进程优先级呢

+ **类似 QQ，在屏幕上保留一个像素，这样就变成了前台进程**    
实现过程是接收系统锁屏和开屏广播，在锁屏时启动一个只有一个像素大小的页面，然后在开屏时将该 Activity 销毁（该 Activity 虽然只有一个像素大小，但是用户如果不将其关闭的话，是无法进行其他操作的，所以只能在开屏和锁屏的情况下做），通过 meizu flyme7 测试，开屏和锁屏是不会发送广播的，所以该方式已经失效了。下面来看下具体实现：
```kotlin
class KeepAliveActivity : AppCompatActivity() {

    companion object {
        fun start(ctx: Context) {
            ctx.startActivity(Intent(ctx, KeepAliveActivity::class.java))
        }
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        initWindow()
    }
    //设置 activity 只有一个像素的大小
    fun initWindow() {
        window.setGravity(Gravity.LEFT or Gravity.TOP)
        val attrs = window.attributes
        attrs.width = 1
        attrs.height = 1

        attrs.x = 0
        attrs.y = 0
    }

    override fun onDestroy() {
        super.onDestroy()
    }
}
```
为了让 activity 的启动和关闭无痕化，可以给它设置 style 为透明，让用户无法感知。
```xml
    <style name="OnePixelActivity" parent="Theme.AppCompat.Light.NoActionBar">
        <item name="android:windowIsTranslucent">true</item>
        <item name="android:windowBackground">@android:color/transparent</item>
        <item name="android:windowFrame">@null</item>
        <item name="android:windowNoTitle">true</item>
        <item name="android:windowIsFloating">true</item>
        <item name="android:windowContentOverlay">@null</item>
        <item name="android:windowCloseOnTouchOutside">true</item>
        <item name="android:windowAnimationStyle">@null</item>
        <item name="android:windowDisablePreview">true</item>
        <item name="android:windowNoDisplay">false</item>
        <item name="android:backgroundDimEnabled">false</item><!--activity不变暗-->
    </style>
```
下面开始监听广播，接收到广播后，回调给需要的监听器:
```kotlin
class WindowBroadcastListenter {
    private var screenStateListener: ScreenStateListener? = null
    private var ctx: Context? = null
    private var broadcastReceiver: SystemBroadcastReceiver? = null

    constructor(context: Context?) {
        ctx = context
        broadcastReceiver = SystemBroadcastReceiver()
    }

    interface ScreenStateListener {
        fun onScreenOn()
        fun onScreenOff()
    }

    fun registerListener(listener: ScreenStateListener) {
        screenStateListener = listener
        registerListener()
    }

    private fun registerListener() {
        val filter = IntentFilter()
        filter.addAction(Intent.ACTION_SCREEN_ON)
        filter.addAction(Intent.ACTION_SCREEN_OFF)
        ctx?.registerReceiver(broadcastReceiver, filter)
    }

    inner class SystemBroadcastReceiver : BroadcastReceiver() {

        override fun onReceive(context: Context?, intent: Intent?) {
            val action = intent?.action

            if (TextUtils.equals(action, Intent.ACTION_SCREEN_ON)) {
                screenStateListener?.onScreenOn()
            } else if (TextUtils.equals(action, Intent.ACTION_SCREEN_OFF)) {
                screenStateListener?.onScreenOff()
            }
        }
    }
}
```
下面创建一个类用于管理该一个像素的 Activity:
```kotlin
class KeepAliveActivityManager private constructor(val ctx: Context) {
    private var activity: WeakReference<AppCompatActivity>? = null

    companion object {
        @Volatile private var INSTANCE: KeepAliveActivityManager? = null

        fun getInstance(ctx: Context) = INSTANCE ?: synchronized(this) {
            INSTANCE ?: buildInstance(ctx).also { INSTANCE = it }
        }

        private fun buildInstance(ctx: Context) = KeepAliveActivityManager(ctx.applicationContext)
    }

    fun setActivity(activity: AppCompatActivity) {
        this.activity = WeakReference(activity)
    }

    fun startActivity() {
        KeepAliveActivity.start(ctx)
    }

    fun finishActivity() {
        activity?.get()?.finish()
    }
}
```
下面就开始启动监听了：
```kotlin
class MainActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        
        registerBroadcast()
    }

    fun registerBroadcast() {
        val windowBroadcastListener = WindowBroadcastListenter(this)
        windowBroadcastListener.registerListener(object : WindowBroadcastListenter.ScreenStateListener {
            override fun onScreenOn() {
                KeepAliveActivityManager.getInstance(this@MainActivity).finishActivity()
            }

            override fun onScreenOff() {
                KeepAliveActivityManager.getInstance(this@MainActivity).startActivity()
            }
        })
    }
}
```

+ **前台服务**    
通过将服务升级为前台服务来提升进程的优先级：
```kotlin
class KeepAliveService : Service() {

    val NOTIFICATION_ID = 0x11

    companion object {
        fun start(ctx: Context) {
            ctx.startService(Intent(ctx, KeepAliveService::class.java))
        }
    }

    override fun onCreate() {
        super.onCreate()
        //API 18以下，直接发送Notification并将其置为前台
        if (Build.VERSION.SDK_INT < Build.VERSION_CODES.JELLY_BEAN_MR2) {
            startForeground(NOTIFICATION_ID, Notification())
        } else {
            //API 18以上，发送Notification并将其置为前台后，启动InnerService
            val builder = Notification.Builder(this)
            builder.setSmallIcon(R.mipmap.ic_launcher)
            startForeground(NOTIFICATION_ID, builder.build())
            startService(Intent(this, InnerService::class.java))
        }
    }

    override fun onBind(intent: Intent): IBinder? {
        throw UnsupportedOperationException("Not yet implemented")
    }

    inner class InnerService : Service() {
        override fun onBind(intent: Intent): IBinder? {
            return null
        }

        override fun onCreate() {
            super.onCreate()
            //发送与KeepLiveService中ID相同的Notification，然后将其取消并取消自己的前台显示
            val builder = Notification.Builder(this)
            builder.setSmallIcon(R.mipmap.ic_launcher)
            startForeground(NOTIFICATION_ID, builder.build())
            Handler().postDelayed(Runnable {
                stopForeground(true)
                val manager = getSystemService(Context.NOTIFICATION_SERVICE) as NotificationManager
                manager.cancel(NOTIFICATION_ID)
                stopSelf()
            }, 100)

        }
    }
}
```
然后在首页销毁时启动该 Activity:
```kotlin
    override fun onDestroy() {
        super.onDestroy()
        KeepAliveService.start(this)
    }
```
发现进程优先级确实提高了。正常情况下，返回至桌面时，进程优先级从0升到15，启动了前台 service 后，优先级从0升到3。但是通过 meizu 手机(android 7.0) 测试，通知栏上的通知并没有像传说中的那样消失掉。

+ **JobService 进行保活**    
JobService 是目前测试发现的唯一一种在进程被杀死后依然能拉起的方式。下面来看下实现：
```kotlin
class MyJobService : JobService() {

    companion object {
        fun start(ctx: Context) {
            ctx.startService(Intent(ctx, MyJobService::class.java))
        }
    }

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        val builder = JobInfo.Builder(1, ComponentName(this, MyJobService::class.java))
        builder.setOverrideDeadline(0) //设置执行延时
        builder.setPersisted(true) //设置持续运行

        val jobScheduler = getSystemService(Context.JOB_SCHEDULER_SERVICE) as JobScheduler
        jobScheduler.schedule(builder.build())
        return START_STICKY
    }

    override fun onStopJob(params: JobParameters?): Boolean {
        Log.e("MyJobService", "onStopJob:")
        return false
    }

    override fun onStartJob(params: JobParameters?): Boolean {
        Log.e("MyJobService", "onStartJob:")
        return false
    }

    override fun onDestroy() {
        super.onDestroy()
        Log.e("MyJobService", "onDestroy:")
    }
}
```
然后添加 Service 和相应权限到 AndroidManifest.xml 中：
```xml
    <uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED" />

        <service
            android:name=".service.MyJobService"
            android:permission="android.permission.BIND_JOB_SERVICE" />
```
然后我们直接调用 MyJobService.start(this) 即可。通过命令查看 oom_adj 值为 3。这种方法使用 android studio 上的 `terminate application` 才有效，如果是直接在手机上调起多任务等杀掉是无效的。
![]({{site.url}}/img/android/basic/appforcekilled/dismiss.png)   

## 总结
综上，android 5.0开始进程保活的方式基本都阵亡了。只有 JobService 在被杀死后可能会被唤醒。    
> 本文参考文献：    
+ https://developer.android.com/guide/components/activities/process-lifecycle
+ https://blog.csdn.net/superxlcr/article/details/70244803?ref=myread
+ https://blog.csdn.net/u013263323/article/details/56285475