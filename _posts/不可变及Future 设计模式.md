---
title: 不可变及Future 设计模式
categories: [java]
tags: [多线程]
date: 2019-11-17 20:32:29
---
**注** 摘自《Java 高并发编程详解: 多线程与架构设计》 机械工业出版社 汪文君著
# 不可变对象设计模式
所谓共享的资源，是指在多个线程同时对其进行访问的情况下，各线程都会使其发生变化，而线程安全性的主要目的就在于在受控的并发访问中防止数据发生变化，除了使用 synchronize 关键字同步对资源的写操作外，还可以在线程之间不共享资源状态，甚至将状态设置为不可变。

设计一个不可变的类共享资源需要具备不可破坏性，比如用final 修饰，另外针对共享资源操作的方法是不允许被重写的，以防止由于继承而带来的安全性问题，但是，单凭这两点补不足于保证一个类是不可变的。

```java
public final class IntegerAccumulator {

	private final int init;
	
	public IntegerAccumulator(int init) {
		this.init = init;
	}

	public IntegerAccumulator(IntegerAccumulator accumulator, int init) {
		this.init = accumulator.getVal() + init;
	}
	
	public IntegerAccumulator add(int i) {
		return new IntegerAccumulator(this, i);
	}
	
	public int getVal() {
		return this.init;
	}
	
}
```
# Future 设计模式
当摸个任务运行需要较长的时间时，调用线程在提交任务之后的徒劳等待对CPU资源来说是一种浪费，在等待的这段时间里，完全可以进行其他任务的执行。
