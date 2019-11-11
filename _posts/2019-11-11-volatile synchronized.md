---
title: volatile 和 synchronize
categories: [java]
tags: [多线程]
date: 2019-11-11 21:13:02
---
**注** 摘自《Java 高并发编程详解: 多线程与架构设计》 机械工业出版社 汪文君著
# 使用上的区别
1. volatile 关键字只能用于修饰实例变量或类变量，不能用于修饰方法以及方法参数和局部变量、常量等；

2. synchronize 关键字不能用于对变量的修饰，只能用于修饰方法或语句块。

3. volatile 修饰的变量可以为null，synchronize 关键字同步语句块的monitor 对象不能为null

# 对原子性的保证
1. volatile 无法保证原子性

2. 由于synchronize 是一种排他的机制，因此被synchronize 关键字修饰的同步代码是无法被中途打断的，因此其能够保证代码的原子性。
# 对可见性的保证
1. 两者均可以保证共享资源在多线程间的可见性，但是实现机制完全不同。

2. synchronize 借助于 JVM 指令 monitor enter 和 monitor exit 对通过排他的方式使得同步代码块串行化，在monitor exit 时所有共享资源都将会被刷新到主内存中。

3. 相比较于 synchronize 关键字 volatile 使用机器指令（偏硬件）"lock ;" 的方式迫使其他线程工作内存中的数据失效，不得到主内存中进行再次加载。
# 对有序性的保证
1. volatile 关键字禁止 JVM 编译器以及处理器对其进行重排序，所以它能够保证有序性。

2. 虽然 synchronize 关键字所修饰的同步方法也可以保证顺序性，但是这种顺序性是以程序的串行化执行换来的，在 synchronize 关键字所修饰的代码块中代码指令也会发生指令重排序的情况，比如：
    ```java
    synchronized(this) {
        int x = 10;
        int y = 20;
        x++;
        y = y + 1;
    }
    ```
# 其他
1. volatile 不会使其他线程陷入阻塞

2. synchronized 关键字会使线程进入阻塞状态