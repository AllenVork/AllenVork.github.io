---
layout:     post
title:      java 并发
subtitle:   解析 Selector 源码
header-img: img/android3.png
author:     Allen Vork
catalog: true
tags:
    - java
---

## 原子操作实现原理和 CAS
原子操作可以通过2种方式实现：    
1. 锁：保证只有获得锁的线程才能操作锁定的内存区域
2. 循环 CAS

### 有了锁机制为什么还需要 CAS 机制？
1. 多线程竞争下，加锁和释放锁会导致比较多的上下文切换和调度延时 ，引起性能问题
2. 一个线程持有锁会导致其它所有需要此锁的线程挂起
3. 若优先级高的线程等待优先级低的线程释放锁，会导致优先级导致，引起性能风险。

volatile是不错的机制，但是volatile不能保证原子性。因此对于同步最终还是要回到锁机制上来。    

锁有两种：乐观锁与悲观锁。独占锁是一种悲观锁，而 synchronized 就是一种独占锁，synchronized 会导致其它所有未持有锁的线程阻塞，而等待持有锁的线程释放锁。乐观锁就是每次不加锁，而是假设没有冲突而去完成某项操作。如果有冲突则失败重试，直到成功为止。乐观锁的机制就是 CAS。CAS操作包含三个操作数——内存位置（V）、预期原值（A）、新值(B)。如果内存位置的值与预期原值相匹配，那么处理器会自动将该位置值更新为新值。否则说明其它线程修改了值，那么 CAS 尝试失败，继续尝试。    
CAS有效地说明了“我认为位置V应该包含值A；如果包含该值，则将B放到这个位置；否则只告诉我这个位置现在的值即可。

### 原子操作(CAS)
Compare And Set（或Compare And Swap），CAS是解决多线程并行情况下使用锁造成性能损耗的一种机制，使用处理器提供的的高效机器级别的原子指令。下面通过 AtomicInteger 来讲解。    
通常 i++ 操作是不安全的，它是由3个独立操作构成的：获取变量当前值，+1，写会新值。通常都是通过加锁来保证读-改-写的线程安全。但加锁会让代码变复杂难以维护。    
如果利用 atomic 包中相关类型就可以很简单实现此操作，下面是一个计数程序实例：    
```java
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.atomic.AtomicInteger;
 
public class Counter {
	private AtomicInteger ai = new AtomicInteger();
	private int i = 0;
 
	public static void main(String[] args) {
		final Counter cas = new Counter();
		List<Thread> ts = new ArrayList<Thread>();
		// 添加100个线程
		for (int j = 0; j < 100; j++) {
			ts.add(new Thread(new Runnable() {
				public void run() {
					// 执行100次计算，预期结果应该是10000
					for (int i = 0; i < 100; i++) {
						cas.count();
						cas.safeCount();
					}
				}
			}));
		}
		//开始执行
		for (Thread t : ts) {
			t.start();
		}
		// 等待所有线程执行完成
		for (Thread t : ts) {
			try {
				t.join();
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		}
		System.out.println("非线程安全计数结果："+cas.i);
		System.out.println("线程安全计数结果："+cas.ai.get());
	}
 
	/** 使用CAS实现线程安全计数器 */
	private void safeCount() {
		for (;;) {
			int i = ai.get();
			// 如果当前值 == 预期值，则以原子方式将该值设置为给定的更新值
			boolean suc = ai.compareAndSet(i, ++i);
			if (suc) {
				break;
			}
		}
	}
 
	/** 非线程安全计数器 */
	private void count() {
		i++;
	}
}
//结果：
非线程安全计数结果：9867
线程安全计数结果：10000
```
来看看 compareAndSet 的实现：    
```java
public final boolean compareAndSet(int expect, int update) {
	return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
}
```
Unsafe类是用于执行低级别、不安全操作的方法集合。    

CAS 虽然高效的解决了原子操作，但它存在3大问题： ABA、循环时间长开销大、只能保证一个共享变量的原子操作。
+ ABA 问题：因为CAS需要在操作值的时候检查下值有没有发生变化，如果没有发生变化则更新，但是如果一个值原来是A，变成了B，又变成了A，那么使用CAS进行检查时会发现它的值没有发生变化，但是实际上却变化了。ABA问题的解决思路就是使用版本号。在变量前面追加上版本号，每次变量更新的时候把版本号加一，那么A－B－A 就会变成1A-2B－3A。 从Java1.5开始JDK的 atomic包里提供了一个类AtomicStampedReference 来解决ABA问题。这个类的 compareAndSet方法作用是首先检查当前引用是否等于预期引用，并且当前标志是否等于预期标志，如果全部相等，则以原子方式将该引用和该标志的值设置为给定的更新值。
```
compareAndSet(V expectedReference, V newReference, int expectedStamp, int newStamp) 
```
+ 循环时间长开销大：自旋CAS如果长时间不成功，会给CPU带来非常大的执行开销。
+ 只能保证一个共享变量的原子操作：对多个共享变量操作时，循环 CAS 无法保证操作的原子性。这个时候可以用锁。也可以把多个变量合并成一个来操作。比如有两个共享变量i＝2,j=a，合并一下ij=2a，然后用CAS来操作ij。




## 参考文献
+ [原子操作(CAS)](https://blog.csdn.net/jack86312031/article/details/84795825)  
+ jdk 1.8 源码
+ [NIO源码阅读(3)-ServerSocketChannel](https://www.jianshu.com/p/5cadad72a2ec) 