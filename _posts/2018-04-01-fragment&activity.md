---
layout:     post
title:      Fragment
subtitle:   
author:     Allen Vork
header-img: img/post-bg-universe.jpg
catalog: true
tags:
    - android
    - android basics    
---

## Fragment 和 Activity 生命周期的对应关系
![]({{site.url}}/img/android/basic/fragment/fna.jpg) 
我们将它们的对应关系分开来看下，黄色为 Fragment 的生命周期，绿色为 Activity 的生命周期：
![]({{site.url}}/img/android/basic/fragment/fragment&activity.png)     
我们可以看出：    
+ Fragment 的 onAttach -> onViewCreated 对应着 Activity 的 onCreate
+ 初始化时，Activity 要先于 Fragment 调用，销毁时相反。如  Activity 的 onStart, onResume 要 “先于” Fragment（当然这得看 activity 在什么时候去添加 Fragment)。而 onPause -> onDestroy 则是 Fragment 要先于 activity 调用。这很好理解，Fragment 是 Activity 的子元素，所以当然要先创建 Activity 才能添加 Fragment。而销毁时，要将 Activity 里的东西销毁才能销毁自己（否则 Activity 销毁了，但 Fragment 还没销毁，导致 Fragment 里使用 activity 的方法就会 crash)。

## Adding a fragment to an activity
+ Declare the fragment inside the activity's layout file

```xml    
<LinearLayout>
    <fragment android:name="com.example.news.ArticleListFragment"
    android:id="@+id/viewer"/>
    <fragment android:name="com.example.news.ArticleReaderFragment"
    android:id="@+id/list"/>
</LinearLayout>
```    
当系统创建 Activity 的 layout 时，它初始化 layout 中的 Fragment 并且调用 onCreateView() 来获取每一个 Fragment 的 layout。系统会将 Fragment 返回的 View 直接替换 <Fragment> 标签。    
**Note:**每个 Fragment 都要设置一个唯一标志（通过 android:id 或 android:tag 指定)，这样当 activity 重新启动时系统个就能恢复 fragment。    

+ Or, programmatically add the fragment to an existing ViewGroup    

```java
//创建 Fragment
ExampleFragment fragment = new ExampleFragment();

//创建 Fragment 的操纵类 FragmentTransaction
FragmentManager fragmentManager = getSupportFragmentManager();
FragmentTransaction fragmentTransaction = fragmentManager.beginTransaction();

//将 activity 中的 ViewGroup 替换为 Fragment
//R.id.fragment_container 一般为 FrameLayout（最简单的 ViewGroup)。
fragmentTransaction.add(R.id.fragment_container, fragment);
fragmentTransaction.commit();
```

## FragmentManager
Activity 提供 getSupportFragmentManager() 获得 FragmentManager 来管理 Fragment。    
我们可以用它来：    
+ 获得 Activity 中的 Fragment： findFragmentById() (获得在 Activity 的 layout 中提供了 UI 的 Fragment) or findFragmentByTag() (获得提供或没提供 UI 的 Fragment)。
+ 通过 popBackStack() 将 Fragment pop 出栈。
+ 通过 addOnBackStackChangedListener() 注册监听器来监听 back stack 变化。
详情可以查阅 [FragmentManager](https://developer.android.com/reference/android/support/v4/app/FragmentManager.html) 文档。       

上面的示例中，我们看到 FragmentManager 还可以打开 FragmentTransaction 来添加和移除 Fragment。    

## FragmentTransaction
在 Activity 中，你可以对 Fragment 进行添加，移除，替换等操作。这些在 activity 中对 Fragment 进行的操作叫事务（transaction)，可以通过 [FragmentTransaction](https://developer.android.com/reference/android/support/v4/app/FragmentTransaction.html) 来执行事务。你可以通过FragmentManager 获得 FragmentTransaction。
```java
FragmentManager fragmentManager = getSupportFragmentManager();
FragmentTransaction fragmentTransaction = fragmentManager.beginTransaction();
```
每一个事务是同一时间内你想执行的变化的集合。这些变化包括 add(), remove() 和 replace() 方法来改变 Fragment，然后通过 commit() 方法将变化提交给 Activity。    
```java
// Create new fragment and transaction
Fragment newFragment = new ExampleFragment();
FragmentTransaction transaction = getSupportFragmentManager().beginTransaction();

// Replace whatever is in the fragment_container view with this fragment,
// and add the transaction to the back stack
transaction.replace(R.id.fragment_container, newFragment);
transaction.addToBackStack(null);

// Commit the transaction
transaction.commit();
```
通过调用 addToBackStack 方法，replace 事务会被保存到 back stack 中，当户点击返回时可以反转事务，回到之前的 fragment。 FragmentActivity 会自动通过 onBackPressed() 方法从 back stack 中拿到 fragments。如果不使用 addToStack() 方法，当你执行一个移除 Fragment 的事务，该 Fragment 会被销毁，用户无法点击返回到该 Fragment。    
**tips:**你可以在 commit() 前调用 `setTrasaction` 给 fragment 事务添加动画。    
调用 commit() 并不会立马执行事务，而是安排它运行在 activity 的 UI 线程中。你也可以在 UI 线程调用 executePendingTransactions() 来立马执行通过 commit() 提交的事务，一般这样做没必要，除非其他线程中的任务依赖了这个事务。    
**Caution：**commit() 方法只能在 activity 保存其状态（用户离开 activity)前,否则就会抛异常。因为保存状态后的事务在 activity 恢复时都会丢失。如果你允许状态丢失的话，可以调用 commitAllowingStateLoss()。 

## show() hide() replace() add()
+ replace() 方法会将被替换的 Fragment 销毁，即调用 onDestroy() 方法。该方法不便重用 Fragment
+ show() hide() 方法是为了弥补上面的不足，从 A Fragment 通过 show() 切换到 B Fragment 并没有调用 A 的任何生命周期函数，只是会走 B 的生命周期来创建 B，然后再切换到 A 时，两者的生命周期函数都没有被调用。    
总结：当 Fragment 需要重用或不断切换的时候，可以使用 show(),hide() 方法来提高效率。但是该方法会有一个问题就是它只会给子控件所需要的高度而不是使用 layout 中设置的高度。可以通过嵌套一个子 layout，给该 layout 设置好宽高即可。

## 判断 Fragment 是否对用户可见
+ 使用 add() 或 replace() 加载或切换 Fragment    
直接在 Activity 的 layout 中引入 Fragment 或通过上述方式进行动态载入的话，onResume 和 onPause 就可以判断 Fragment　的显隐。　　　　
+ 使用 show() 和 hide() 加载或切换 Fragment    
hide(Fragment) 并没有调用 onPause() 方法，需要通过 onHiddenChanged 方法来判断。
+ ViewPager 中嵌套 Fragment    
通过 setUserVisibleHint() 方法来判断

## Android Fragment＋ViewPager 组合
ViewPager 提供了2种 adapter 来管理 Fragment 的切换：FragmentPagerAdapter 和 FragmentStatePagerAdapter。我们来看下源码    
```java
@Override
  public Object instantiateItem(ViewGroup container, int position) {
      if (mFragments.size() > position) {
          Fragment f = mFragments.get(position);//fragment被释放后这里得到的null值
          if (f != null) {
              return f;
          }
      }

      if (mCurTransaction == null) {
          mCurTransaction = mFragmentManager.beginTransaction();
      }

	  //拿到新的Fragment实例
      Fragment fragment = getItem(position);

      if (mSavedState.size() > position) {
          Fragment.SavedState fss = mSavedState.get(position);
          if (fss != null) {
              fragment.setInitialSavedState(fss);
          }
      }
      while (mFragments.size() <= position) {
          mFragments.add(null);
      }
      fragment.setMenuVisibility(false);
      fragment.setUserVisibleHint(false);
      mFragments.set(position, fragment);
      mCurTransaction.add(container.getId(), fragment);//新的Fragment实例 是add上去的

      return fragment;
  }

 @Override
  public void destroyItem(ViewGroup container, int position, Object object) {
      Fragment fragment = (Fragment) object;

      if (mCurTransaction == null) {
          mCurTransaction = mFragmentManager.beginTransaction();
      }

      while (mSavedState.size() <= position) {
          mSavedState.add(null);
      }
      mSavedState.set(position, fragment.isAdded()
              ? mFragmentManager.saveFragmentInstanceState(fragment) : null);
	  //真正释放了fragment实例
      mFragments.set(position, null);

      mCurTransaction.remove(fragment);
  }
```
我们可以看到 FragmentStatePagerAdapter 在 destroyItem 的时候，调用了 `mFragments.set(position, null)` 将 fragment 的引用设为 null了，将之释放了。

```java
@Override
  public Object instantiateItem(ViewGroup container, int position) {
      if (mCurTransaction == null) {
          mCurTransaction = mFragmentManager.beginTransaction();
      }

      final long itemId = getItemId(position);

      // Do we already have this fragment?
      String name = makeFragmentName(container.getId(), itemId);
      Fragment fragment = mFragmentManager.findFragmentByTag(name);
      if (fragment != null) {
		  //因为fragment实例没有被真正释放，所以可以直接attach,不用重新创建
          mCurTransaction.attach(fragment);
      } else {
          fragment = getItem(position);
          mCurTransaction.add(container.getId(), fragment,
                  makeFragmentName(container.getId(), itemId));
      }
      if (fragment != mCurrentPrimaryItem) {
          fragment.setMenuVisibility(false);
          fragment.setUserVisibleHint(false);
      }
      return fragment;
  }

  @Override
  public void destroyItem(ViewGroup container, int position, Object object) {
      if (mCurTransaction == null) {
          mCurTransaction = mFragmentManager.beginTransaction();
      }
      //并没有真正释放fragment对象只是detach
      mCurTransaction.detach((Fragment)object);
  }
```

**总结：**    
当视图不可见时，FragmentPagerAdapter 只会调用 onDestroyView 方法，适合少量页面的场景；而 FragmentStatePagerAdapter 会销毁实例，仅仅保存 Fragment 状态，适用于有大量的动态页面的应用。

## 懒加载
在 Fragment 可见时才会加载数据，可以减轻页面数据请求的带宽压力。我们知道 ViewPager 的 setOffscreenPageLimit(int limit) 方法里，参数不能小于1,意味着它至少会预加载一个页面。为了防止 ViewPager 去预加载，我们可以结合 setUserVisibleHint() 方法来判断 Fragment 是否可见，可见才会加载数据。
```java
public abstract class LazyLoadFragment extends BaseFragment {
    protected boolean isViewInitiated;
    protected boolean isDataLoaded;

    @Override
    public void onActivityCreated(Bundle savedInstanceState) {
        super.onActivityCreated(savedInstanceState);
        isViewInitiated = true;
        prepareRequestData();
    }
    @Override
    public void setUserVisibleHint(boolean isVisibleToUser) {
        super.setUserVisibleHint(isVisibleToUser);
        prepareRequestData();
    }
    public abstract void requestData();
    public boolean prepareRequestData() {
        return prepareRequestData(false);
    }
    public boolean prepareRequestData(boolean forceUpdate) {
        if (getUserVisibleHint() && isViewInitiated && (!isDataLoaded || forceUpdate)) {
            requestData();
            isDataLoaded = true;
            return true;
        }
        return false;
    }
}
```
子类只需要实现 requestData 方法即可。