---
layout: post
title: "Java并发编程的艺术笔记"
date: 2018-06-15 23:04
comments: true
categories: 
- dev
- java
- notes
tags:
- multi-thread
---

已迁移到gitbook，请访问[Java并发编程的艺术笔记](https://solarex.github.io/reading-notes/the-art-of-java-concurrency-programming/readme.html)。

<!-- more -->

* [并发编程的挑战](#ch01)
* [Java并发机制的底层实现原理](#ch02)
* [Java内存模型](#ch03)
* [Java并发编程基础](#ch04)
* [Java中的锁](#ch05)
* [Java并发容器和框架](#ch06)
* [Java中的13个原子操作类](#ch07)
* [Java中的并发工具类](#ch08)
* [Java中的线程池](#ch09)
* [Executor框架](#ch10)
* [Java并发编程实践](#ch11)

<!-- more -->

<h2 id="ch01">并发编程的挑战</h2>
利用``vmstat``测量上下文切换次数。CS(Context Switch)

减少上下文切换的方法有无锁并发编程、CAS算法、使用最少线程和使用协程。

+ 无锁并发编程。多线程竞争锁时，会引起上下文切换，所以多线程处理数据时，可以用一些办法来避免使用锁，如将数据的ID按照Hash算法取模分段，不同的线程处理不同段的数据。
+ CAS算法。Java的Atomic包使用CAS算法来更新数据，而不需要加锁。
+ 使用最少线程。避免创建不需要的线程，比如任务很少，但是创建了很多线程来处理，这样会使大量线程处于等待状态。
+ 协程：在单线程里实现多任务的调度，并在单线程里维持多个任务间的切换。

``sudo -u admin jstack {pid}``

避免死锁的几种常见方法：

+ 避免一个线程同时获取多个锁
+ 避免一个线程在锁内同时占用多个资源，尽量保证每个锁只占用一个资源
+ 尝试使用定时锁，使用``lock.tryLock(timeout)``来替代使用内置锁机制
+ 对于数据库锁，加锁和解锁必须在一个数据库连接里，否则会出现解锁失败的情况

<h2 id="ch02">Java并发机制的底层实现原理</h2>
如果一个字段被声明为``volatile``，Java线程内存模型确保所有线程看到这个变量的值是一致的。有``volatile``变量修饰的共享变量进行写操作的时候会多出``lock``指令的汇编代码。``lock``前缀的指令在多核处理器下会引发两件事情。



+ 将当前处理器缓存行的数据写回到系统内存
+ 这个写回内存的操作会使在其他CPU里缓存了改内存地址的数据无效

``synchronized``实现同步的基础：Java中每一个对象都可以作为锁。具体表现为3中形式：



+ 对于普通同步方法，锁是当前实例对象
+ 对于静态同步方法，锁时当前类的Class对象
+ 对于同步方法块，锁是``synchronized``括号里配置的对象。

当一个线程试图访问同步代码块时，它首先必须得到锁，退出或抛出异常时必须释放锁。

代码块的同步是使用``monitorenter``和``monitorexit``指令实现的。

``synchronized``用的锁是存在Java对象头里的。如果对象是数组类型，则虚拟机用3个字宽Word存储对象头，如果对象是非数组类型，则用2字宽存储对象头。在32位虚拟机中，1字宽等于4字节，即32bit。

Java对象头里的Mark Word里默认存储对象的hashcode、分代年龄、锁标记位。

JVM中CAS操作正是利用了处理器提供的``CMPXCHG``指令实现的。自旋CAS实现的基本思路就是循环进行CAS操作直到成功为止。

使用CAS实现原子操作的三大问题：



+ ABA问题。``AtomicStampedReference``可以用来解决ABA问题。
+ 循环时间开销大
+ 只能保证一个共享变量的原子操作。从Java 1.5开始，JDK提供了``AtomicReference``类来保证引用对象之间的原子性，就可以把多个变量放在一个对象里来进行CAS操作。

<h2 id="ch03">Java内存模型</h2>

在Java中，所有实例域、静态域和数组元素都存储在堆内存中，堆内存在线程之间共享。局部变量，方法定义参数和异常处理器参数不会再线程之间共享，他们不会有内存可见性问题，也不受内存模型的影响。Java线程之间的通信由Java内存模型JMM控制，JMM决定一个线程对共享变量的写入何时对另一个线程可见。从抽象的角度来看，JMM定义了线程和主内存之间的抽象关系：线程之间的共享变量存储在主内存（Main Memory）中，每个线程都有一个私有的本地内存（Local Memory），本地内存中存储了该线程以读/写共享变量的副本。本地内存是JMM的一个抽象概念，并不真实存在。它涵盖了缓存、写缓冲区、寄存器以及其他的硬件和编译器优化。

在执行程序时，为了提高性能，编译器和处理器常常会对指令做重排序。重排序分3中类型：



+ 编译器优化的重排序。编译器在不改变单线程程序语义的前提下，可以重新安排语句的执行顺序。
+ 指令级并行的重排序。
+ 内存系统的重排序。

为了保证内存可见性，Java编译器在生成指令序列的适当位置会插入内存屏障指令来禁止特定类型的处理器重排序。JMM把内存屏障指令分为4类：



+ LoadLoad Barriers
+ StoreStore Barriers
+ LoadStore Barriers
+ StoreLoad Barriers



JSR-133 使用happens-before的概念来阐述操作之间的内存可见性。在JMM中，如果一个操作执行的结果需要对另一个操作可见，那么这两个操作之间必须要存在happens-before关系。



+ 程序顺序规则：一个线程中的每个操作，happens-before于该线程中的任意后续操作
+ 监视器锁规则：对一个锁的解锁，happens-before于随后对这个所得加锁
+ volatile变量规则：对于一个volatile域的写，happens-before于任意后续对这个volatile域的读
+ 传递性：如果A happens-before B，且 B happens-before C，那么A happens-before C

对于final域，编译器和处理器要遵守两个重排序规则。



+ 在构造函数内对一个final域的写入，与随后把这个被构造对象的引用赋值给一个引用变量，这两个操作之间不能重排序。
+ 初次读一个包含final域的对象的引用，与随后初次读这个final域，这两个操作之间不能重排序。

<h2 id="ch04">Java并发编程基础</h2>

![](https://ws2.sinaimg.cn/large/006tNbRwgy1fv3mv5q2gsj31400u00tz.jpg)

中断可以理解为线程的一个标识位属性，它表示一个运行中的线程是否被其他线程进行了中断操作。中断好比其他线程对该线程打了个招呼，其他线程通过调用该线程的``interrupt``方法对其进行中断操作。线程通过检查自身是否被中断来进行响应，线程通过方法``isInterrupted``来进行判断是否被中断，也可以调用静态方法``Thread.interrupted()``对当前线程的中断标识位进行复位。



等待通知的经典范式，该范式分为两部分，分别针对等待方（消费者）和通知方（生产者）。

等待方遵循如下原则：



+ 获取对象的锁
+ 如果条件不满足，那么调用对象的wait()方法，被通知后仍要检查条件
+ 条件满足时则执行对应的逻辑

```java
synchronized(对象) {
    while(条件不满足) {
        对象.wait();
    }
    对应的处理逻辑
}
```

通知方遵循如下原则：



+ 获得对象的锁
+ 改变条件
+ 通知所有等待在对象上的线程

```java
synchronized(对象) {
    改变条件
    对象.notifyAll();
}
```

``ThreadLocal``，即线程变量，是一个以``ThreadLocal``对象为键、任意对象为值的存储结构。这个结构被附带在线程上，也就是说一个线程可以根据一个``ThreadLocal``对象查询到绑定在这个线程上的一个值。

等待超时模式

```java
// 对当前对象加锁
public synchronized Object get(long mills) throws InterruptedException {
    long future = System.currentTimeMillis() + mills;
    long remaining = mills;
    while ((result == null) && remaining > 0) {
        wait(remaining);
        remaining = future - System.currentTimeMillis();
    }
    return result;
}
```



<h2 id="ch05">Java中的锁</h2>

Java 1.5之后，并发包中新增了Lock接口以及相关实现类用来实现锁功能，它提供了与``synchronized``关键字类似的同步功能，只是在使用时需要显式地获取锁和释放锁。虽然他缺少了通过synchronized块或者方法提供的隐式获取释放锁的便捷性，但是却拥有了锁获取与释放的可操作性、可中断的获取锁以及超时获取锁等多种synchronized关键字所不具备的同步特性。

AQS的主要使用方式是继承，子类通过继承AQS并实现它的抽象方法来管理同步状态，在抽象方法的实现过程中免不了要对同步状态进行更改，这时就需要使用AQS提供的3个方法——``getState()``,``setState(int newState)``,``compareAndSetState(int expect, int update)``来进行操作，因为他们能够保证状态的改变是安全的。

当需要阻塞或唤醒一个线程的时候，都会使用``LockSupport``工具类来完成相应工作。``LockSupport``定义了一组公共静态方法，这些方法提供了最基本的线程阻塞和唤醒功能，而LockSupport也成为构建同步组件的基础工具。

任意一个Java对象，都拥有一组监视器方法（定义在java.lang.Object上），主要包括``wait()``,``wait(long timeout)``,``notify()``和``notifyAll()``方法，这些方法和``synchronized``关键字配合，可以实现等待、通知模式。

![](https://ws3.sinaimg.cn/large/006tNbRwgy1fv3ogb2axgj31kw16okjm.jpg)



<h2 id="ch06">Java并发容器和框架</h2>

ConcurrentHashMap 分段锁

ConcurrentHashMap的size操作，先尝试2次不锁住Segment的方式来统计各个Segment的大小，如果统计的过程中，容器的count发生了变化，则再采用加锁的方式来统计所有Segment的大小。

ConcurrentLinkedQueue

阻塞队列

<h2 id="ch07">Java中的13个原子操作类</h2>

原子更新基本类型类：``AtomicBoolean``，``AtomicInteger``，``AtomicLong``

原子更新数组：``AtomicIntegerArray``,``AtomicLongArray``,``AtomicReferenceArray``

原子更新引用类型：``AtomicReference``,``AtomicReferenceFieldUpdater``,``AtomicMarkableReference``

原子更新字段类：``AtomicIntegerFieldUpdater``，``AtomicLongFieldUpdater``,``AtomicStampedReference``

<h2 id="ch08">Java中的并发工具类</h2>

等待多线程完成的``CountDownLatch``

同步屏障``CyclicBarrier``

``CountDownLatch``的计数器只能使用一次，而``CyclicBarrier``的计数器可以使用``reset()``方法重置。所以``CyclicBarrier``能处理更为复杂的业务场景。例如，如果计算发生错误，可以重置计数器，并让线程重新执行一次。``CyclicBarrier``还提供其他有用的方法，比如``getNumberWaiting``方法可以获得``CyclicBarrier``阻塞的线程数量，``isBroken``方法用来了解阻塞的线程是否被中断。

控制并发线程数的``Semaphore``

线程间交换数据的``Exchanger``

<h2 id="ch09">Java中的线程池</h2>

当提交一个新任务到线程池时，线程池的处理流程如下：



+ 线程池判断核心线程池里的线程是否都在执行任务，如果不是，则创建一个新的工作线程来执行任务。如果核心线程池里的线程都在执行任务，则进入下个流程。
+ 线程池判断工作队列是否已满，如果工作队列没有满，则将新提交的任务存储在这个工作队列里。如果工作队列满了，则进入下个流程。
+ 线程池判断线程池的线程数是否小于maxPoolSize，如果没有，则创建一个新的工作线程来执行任务。如果已经满了，则交给饱和策略来处理这个任务。

``ThreadPoolExecutor(corePoolSize, maxPoolSize, keepAliveTime, milliseconds, runnableTaskQueue, handler)``



+ corePoolSize 核心线程数
+ runnableTaskQueue任务队列，用于保存等待执行的任务的阻塞队列，可以选择``ArrayBlockingQueue``,``LinkedBlockingQueue``,``SynchronousQueue``,``PriorityBlockingQueue``
+ maxPoolSize 线程池允许创建的最大线程数
+ ThreadFactory 创建线程的工厂
+ RejectExecutionHandler 饱和策略 ``AbortPolicy``,``CallerRunsPolicy``,``DiscardOldestPolicy``,``DiscardPolicy``

<h2 id="ch10">Executor框架</h2>

Executor框架主要由3大部分组成如下：



+ 任务。包括被执行任务需要实现的接口：Runnable或Callable
+ 任务的执行。包括任务执行机制的核心接口Executor，以及继承自Executor的ExecutorService接口。Executor框架有两个

<h2 id="ch11">Java并发编程实践</h2>

