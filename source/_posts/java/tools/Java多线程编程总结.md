---
title: Java多线程编程总结
categories: 
    - Java
date: 2019-08-26 20:44:39
tags:
    - Java多线程
---

# 多线程编程

## 线程基本概念

### 主线程与子线程

每个Java应用程序都有一个执行Main()函数的默认线程，这就是主线程（main thread）。当Java程序启动时，主线程立刻运行，因为它是程序开始时就执行的。主线程的重要性体现在两方面：

1. 它是产生其他子线程的线程

2. 通常它必须最后完成执行，因为它执行各种关闭动作

由主线程创建的线程即被称为子线程。Java主要通过`jaava.lang.Thread`类以及`java.lang.Runnable`接口来实现线程机制。

**实现`Runnable`接口：**

```java
public class NewRunnable implements Runnable {
    public void run() {
        System.out.println("NewRunnable ... ");
    }
    public static void main(String[] args) {
        Thread thread = new Thread(new NewRunnable());
        thread.start();
    }
}

```

从该例可以看出，实现`Runnable`接口只需要实现其`run`方法，仅仅声明了方法，要执行该方法还需要使用`Thread`类调用`start`方法执行。

**可以继承`Thread`类：**

```java
public class NewThread extends Thread {
    @Override
    public void run() {
        System.out.println("NewThread ...");
    }
    public static void main(String[] args) {
        Thread thread = new NewThread();
        thread.start();
    }
}
```

对于创建线程，一般都使用实现`Runnable`接口方式来实现；如果需要重载`Thread`的方法，则使用继承`Thread`的方式实现。

### 线程状态

线程的状态分为6种，分别为：

1. 创建(NEW)：新创建了一个线程对象，但还没有调用start()方法。
2. 就绪(RUNNABLE)：Java线程中将就绪（ready）和运行中（running）两种状态笼统的称为“运行”。
3. 阻塞(BLOCKED)：线程被锁阻塞。
4. 等待(WAITING)：进入该状态的线程需要等待其他线程做出一些特定动作（通知或中断）。
5. 超时等待(TIMED_WAITING)：该状态不同于WAITING，它可以在指定的时间后自行返回。
6. 死亡(TERMINATED)：表示该线程已经执行完毕。

**状态切换：**

![线程状态切换](https://static.jiangliuhong.top/blogimg/tools/%E7%BA%BF%E7%A8%8B%E5%88%87%E6%8D%A2.png)

### 线程优先级

线程优先级被线程调度用来判定何时每个线程允许运行。理论上，优先级高的线程比优先级低的线程获得更多的`CPU`时间。实际上，线程获得的`CPU`时间通常由包括优先级在内的多个因素决定（例如，一个实行多任务处理的操作系统如何更有效的利用`CPU`时间）。

一个优先级高的线程自然比优先级低的线程优先。举例来说，当低优先级线程正在运行，而一个高优先级的线程被恢复（例如从沉睡中或等待I/O中），它将抢占低优先级线程所使用的`CPU`。

在创建线程的时候，可以通过`Thread.setPriority(int newPriority)`方法指定线程优先级；改方法是否生效与操作系统及虚拟机版本相关。

其中`newPriority`参数使用1-10表示：

- 最低优先级 1：`Thread.MIN_PRIORITY`
- 普通优先级 5：`Thread.NORM_PRIORITY`
- 最高优先级 10：`Thread.MAX_PRIORITY`

在Java程序中，线程的默认优先级为其父线程的优先级，而非普通优先级(`NORM_PRIORITY`)。

### 线程同步

Java程序运行多线程之间并发控制，当两个或两个以上的线程需要共享资源，它们需要某种方法来确定资源在某一刻仅被一个线程占用，避免相互之间产生冲突，保证变量的唯一性和准确性，达到此目的的过程叫做同步（synchronization）。

**线程同步方法：**

1.使用`synchronized`修饰方法、代码块。如果修饰的是静态方法，则锁的是类。

```java
public synchronized void save(){}
synchronized(object){ 
}
public synchronized static void method() {
}
```

2.使用`volatile`变量实现线程同步

`volatile`通常被比喻成"轻量级的`synchronized`"，不同的是`volatile`只能修饰变量。

`volatile`主要作用是保证线程之间的可见性、有序性：

- 可见性：Java中的`volatile`关键字提供了一个功能，那就是被其修饰的变量在被修改后可以立即同步到主内存，被其修饰的变量在每次是用之前都从主内存刷新。
- 有序性：`volatile`可以禁止指令重排，被它修饰的变量，会严格按照代码顺序执行。

- 原子性：一个操作或者多个操作要么全部执行，要么全不执行。

## 线程池

线程池的优势：

- 降低资源消耗：通过重复利用已创建的线程降低线程创建的销毁造成的消耗，需要注意的是，如果线程过多，线程上下文切换会是一种消耗。
- 提高响应速度：当任务到达时，任务可以不需要等待线程创建就能立即执行。服务端拿到用户请求，可以先向用户返回结果，然后利用线程异步的执行任务。
- 提高线程的可管理性：线程是稀缺资源，如果无限制的创建，不仅会消耗系统资源，还会降低系统的稳定性，使用线程池可以进行统一的分配，调优和监控。

### Excutor的线程池

- `newSingleThreadExecutor`：一个单线程的线程池，可以用于需要保证顺序执行的场景，并且只有一个线程在执行。
- `newFixedThreadPool`：一个固定大小的线程池，可以用于已知并发压力的情况下，对线程数做限制。
- `newCachedThreadPool`：一个可以无限扩大的线程池，比较适合处理执行时间比较小的任务。
- `newScheduledThreadPool`：可以延时启动，定时启动的线程池，适用于需要多个后台线程执行周期任务的场景。
- `newWorkStealingPool`：一个拥有多个任务队列的线程池，可以减少连接数，创建当前可用cpu数量的线程来并行执行。
- 自定义线程池

## 四种并发模型

### 多线程编程模型

- 多个相互独立的执行流
- 共享内存状态
- 抢占式调度
- 依赖锁、信号量等同步机制

### Callback编程模型

回调。某个函数(A)可以接受另一个函数(B)作为参数，在执行流程到某个点时作为参数的函数B就会被函数A调用执行，这个行为就被称为回调。

作用：快速响应。

应用：在javascript中应用较多

### Actor编程模型

- 万物皆是Actor
- Actor之间完全独立，只允许消息传递，不允许其他”任何”共享
-  每个Actor最多同时只能进行一样工作
-  每个Actor都有一个专属的命名Mailbox(非匿名)
-  消息的传递是完全异步的
- 消息是不可变的

### CSP编程模型

CSP（Communicating Sequential Processes）是由Tony Hoare在1978的论文上首次提出的。 它是处理并发编程的一种设计模式或者模型，指导并发程序的设计，提供了一种并发程序可实践的组织方法或者设计范式。通过此方法，可以减少并发程序引入的其它缺点，减少和规避并发程序的常见缺点和bug，并且可以被数学理论所论证。