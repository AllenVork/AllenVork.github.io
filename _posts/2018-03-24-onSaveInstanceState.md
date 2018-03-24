---
layout:     post
title:      onSavedInstanceState 和 onRestoreInstanceState
subtitle:   
author:     Allen Vork
header-img: img/post-bg-kuaidi.jpg
catalog: true
tags:
    - android
    - android basics    
---

## 作用
当 activity **变得容易销毁**的时候，会调用 onSavedInstanceState 方法来存储数据。如果 Activity 销毁了，则会使用 onRestoreInstanceState 来恢复数据（onSaveInstanceState 中保存的数据在恢复时也会传给 onCreate)。

## 调用场景
> onSaveInstanceState

+ 按下 Home 键，Activity 进入后台时
+ 按下电源键，屏幕关闭，Activity 进入后台
+ 启动其他 Activity
+ 横竖屏切换

如果是用户点击 back，或者主动调用 finish 方法来关闭 Activity，而不是 Activity 进入后台后被杀掉的话，是不会调用的。

> onRestoreInstanceState
在 Activity 被系统销毁后，恢复 Activity 时会调用。

可以看出 onSaveInstanceState 方法 和 onRestoreInstanceState 方法并不是成对出现的。    

## 调用时机
onSaveInstanceState 出现在 onStop 或 onPause 之前，onRestoreInstanceState 出现在 onstart 与 onResume 之间。    
下面是旋转屏幕时，Activity 的生命周期的变化图：
![]({{site.url}}/img/android/basic/onsaveinstancestate/lifecycle.png) 
注意，onSavedInstanceState 并不一定在 onPause 之后调用。

### onCreate(Bundle savedInstanceState) 中既然能恢复数据，为什么还需要 onRestoreInstanceState?
如果是在 onCreate 中进行恢复数据的话，会导致每次创建（而非恢复）Activity 的时候也要有处理这个 bundle （判断是否为空）的逻辑。而且用 onRestoreInstanceState 方法，我们可以将恢复的逻辑和创建的逻辑解耦。

### 什么时候用到 onSaveInstanceState?
由于 View 中都有默认的 onSaveInstanceState 方法，只要这个 View 有 id,当 Activity 被系统杀死时，它会自动保存 UI 的一些状态。譬如往 EditText 中输入数据后，切换横屏，这时候数据是自动被保存的，不需要我们去重写 onSaveInstanceState 进行处理。

-----------------------
## 源码解析
### 我们首先来看看 Activity 在 pause 的时候会不会调用 onSaveInstanceState 方法
```java
    final Bundle performPauseActivity(ActivityClientRecord r, boolean finished,
            boolean saveState, String reason) {
        if (r.paused) {
            if (r.activity.mFinished) {
                // If we are finishing, we won't call onResume() in certain cases.
                // So here we likewise don't want to call onPause() if the activity
                // isn't resumed.
                return null;
            }
            RuntimeException e = new RuntimeException(
                    "Performing pause of activity that is not resumed: "
                    + r.intent.getComponent().toShortString());
            Slog.e(TAG, e.getMessage(), e);
        }
        if (finished) {
            r.activity.mFinished = true;
        }

        // Next have the activity save its current state and managed dialogs...
        if (!r.activity.mFinished && saveState) {
            callCallActivityOnSaveInstanceState(r);
        }

        performPauseActivityIfNeeded(r, reason);

        // Notify any outstanding on paused listeners
        ArrayList<OnActivityPausedListener> listeners;
        synchronized (mOnPauseListeners) {
            listeners = mOnPauseListeners.remove(r.activity);
        }
        int size = (listeners != null ? listeners.size() : 0);
        for (int i = 0; i < size; i++) {
            listeners.get(i).onPaused(r.activity);
        }

        return !r.activity.mFinished && saveState ? r.state : null;
    }
```
在 PerformPauseActivity 的时候，会调用 callCallActivityOnSaveInstanceState()。但是有一个条件 `r.activity.mFinished && saveState`。mFinished 值只有当调用了 finishi() 方法才会为 true。而 saveState 是由下面这个函数来决定的：    
```java
        public boolean isPreHoneycomb() {
            if (activity != null) {
                return activity.getApplicationInfo().targetSdkVersion
                        < android.os.Build.VERSION_CODES.HONEYCOMB;
            }
            return false;
        }
```
只要 targetSdk < 11 就返回 true。那么意思就是只要没有调用 finish() 方法，则会调用 callCallActivityOnSaveInstanceState 方法。    
再来看 callCallActivityOnSaveInstanceState() 方法:    

``` java
    private void callCallActivityOnSaveInstanceState(ActivityClientRecord r) {
        r.state = new Bundle();
        r.state.setAllowFds(false);
        if (r.isPersistable()) {
            r.persistentState = new PersistableBundle();
            mInstrumentation.callActivityOnSaveInstanceState(r.activity, r.state,
                    r.persistentState);
        } else {
            mInstrumentation.callActivityOnSaveInstanceState(r.activity, r.state);
        }
    }
```
Instrumentation 里的该函数：    
```java
public void callActivityOnSaveInstanceState(Activity activity, Bundle outState,  
            PersistableBundle outPersistentState) {  
        activity.performSaveInstanceState(outState, outPersistentState);  
    }  
```
再看看 activity 的 performSaveInstanceState:
```java
final void performSaveInstanceState(Bundle outState) {  
        onSaveInstanceState(outState);  
        saveManagedDialogs(outState);  
        mActivityTransitionState.saveState(outState);  
    }  
```
我们看到这样就调用到了 activity 的 onSaveInstanceState 方法。即如果没有调用 finish() 方法的话，onSavedInstanceState() 方法会在 onPause() 之前调用。

### 下面来看看 Activity 在 stop 的时候会不会调用 onSavedInstanceState 方法
在 Activity 执行 onStop 时会调用到 ActivityThread 中的 handleStopActivity() 方法，最终会调用到：
```java
    private void performStopActivityInner(ActivityClientRecord r,
            StopInfo info, boolean keepShown, boolean saveState, String reason) {
        if (localLOGV) Slog.v(TAG, "Performing stop of " + r);
        if (r != null) {
            if (!keepShown && r.stopped) {
                if (r.activity.mFinished) {
                    // If we are finishing, we won't call onResume() in certain
                    // cases.  So here we likewise don't want to call onStop()
                    // if the activity isn't resumed.
                    return;
                }
                RuntimeException e = new RuntimeException(
                        "Performing stop of activity that is already stopped: "
                        + r.intent.getComponent().toShortString());
            }

            // One must first be paused before stopped...
            performPauseActivityIfNeeded(r, reason);

            // Next have the activity save its current state and managed dialogs...
            if (!r.activity.mFinished && saveState) {
                if (r.state == null) {
                    callCallActivityOnSaveInstanceState(r);
                }
            }
		}
    }
```