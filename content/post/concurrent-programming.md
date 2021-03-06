---
title: "多线程学习笔记"
date: Thu Aug 29 2019
lastmod: 2019-08-29T21:40:23+08:00
keywords: []
categories: [Notes]
tags: [多线程, 并发]
mathjax: false

---

开一篇多线程学习笔记，记录下在实习过程中遇到的一些简单问题。

> 注意：这是一篇以学习笔记，难免有误，主要写给自己参考。请酌情判别，如有错误，也欢迎指正！

## 互斥

互斥的本质就是等待！

互斥的特性

1. 无死锁
2. 无饥饿
3. 等待

并发系统中存在两种类型的通信：

- 瞬时通信：要求通信双方在同一时刻都参与通信。
- 持续通信：允许发送者和接收者在不同时间参与通信。

互斥需要的是持续通信。并发系统中常用的一种通信协议：**中断**。现在操作系统中，一个线程要引起另一个线程注意的常用方法就是发送中断信号。更准确的说，线程A通过设置一个位向线程B发出一个中断信号，线程B周期地检查这个位。一旦B检测到该位置被设置，则做出相应的响应。响应结束后，通常由B进行复位（A不能复位）。虽然中断不能解决互斥问题。但它仍是非常有用的。例如，Java中的`wait()`调用和`notifyall()`调用的本质就是中断。

## 编写多线程需要注意的点

1. 在脑中先大致想好每个线程的工作是什么，什么时候开始，什么时候结束。
2. 捋清楚了之后再开始动手写。

调用`t.join()`的作用类似于，如果线程结束，主线程执行到join就可以立即返回，如果线程为结束，主线程执行到join会阻塞，直到线程结束。然后主线程继续执行。
```
            main  thread1
              +      +
              |      |
              |      |
              |      |
              |      |
thread1.join()+------+
              |
              |
              |
              v
```

如果某线程申请占有互斥量时，该互斥量被其他线程占有，则会引起该线程阻塞。

一般来说线程安全性很难保证，但只有两种操作可以保证线程安全性，

1. 基本的原子操作
2. CAS（compare and swap) 操作

## 使用`std::conditional_variable`注意事项

- 调用wait的线程必须占有mutex，否则undefined
- 所有并发线程（如果使用同一个条件变量交互）必须使用同一个mutex，否则undefined

## 使用`std::thread`注意事项

- thread对象构造完成即开始执行
- 使用detach之后，程序失去该线程的控制权，线程结束之后资源全部释放
- 使用detach之后，主线程结束时，所有资源都被释放，即便该线程还未停止
- 线程之间没有父/子关系。如果线程A创建线程B，然后线程A终止，则线程B将继续执行。但如果主线程终止，则整个进程终止，自然进程下的所有线程都终止，资源释放
- Any thread can potentially access any object in the program (objects with automatic and thread-local storage duration may still be accessed by another thread through a pointer or by reference).

## 使用`std::atomic`注意事项

- atomic is neither copyable nor movable
- mutex is neither copyable nor movable

加一次锁耗时大概25ns，使用lock-free的话能够提高到十几纳秒，事实上提升不大。加锁并没有想象中那么耗时，提高效率的关键是减少锁的碰撞。即一个线程占有锁的时候，其他线程不会去申请锁，因为在锁被占用的情况下去申请锁比较耗时，会先去loop一段时间，拿不到锁才会进入内核陷入睡眠等待锁，这样的耗时是比较浪费的。所以关键要减少锁的碰撞。

有原子的函数吗，就是要么执行成功，要么失败？
不存在，一个函数内部多少指令，在多线程的情况下，很难保证可以全部的顺序的原子的执行完成。

## From Shuo's blog

依据《Java 并发编程实践》/《Java Concurrency in Practice》一书，一个线程安全的 class 应当满足三个条件：

- 从多个线程访问时，其表现出正确的行为
- 无论操作系统如何调度这些线程，无论这些线程的执行顺序如何交织
- 调用端代码无需额外的同步或其他协调动作

对象构造要做到线程安全，惟一的要求是在构造期间不要泄露 this 指针，即

- 不要在构造函数中注册任何回调
- 也不要在构造函数中把 this 传给跨线程的对象
- 即便在构造函数的最后一行也不行

作为 class 数据成员的 Mutex 只能用于同步本 class 的其他数据成员的读和写，它不能保护安全地析构。因为成员 mutex 的生命期最多与对象一样长，而析构动作可说是发生在对象身故之后（或者身亡之时）。另外，对于基类对象，那么调用到基类析构函数的时候，派生类对象的那部分已经析构了，那么基类对象拥有的 mutex 不能保护整个析构过程。
