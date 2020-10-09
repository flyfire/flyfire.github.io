---
layout: post
title: "Java并发编程实战笔记"
date: 2018-06-03 21:43
comments: true
categories: 
- dev
- java
- notes
tags:
- java
---

已迁移到gitbook，请访问[Java并发编程实战笔记](https://solarex.github.io/reading-notes/jcip/readme.html)。

<!-- more -->

* [简介](#ch01)
* [线程安全性](#ch02)
* [对象的共享](#ch03)
* [对象的组合](#ch04)
* [基础构建模块](#ch05)
* [任务执行](#ch06)
* [取消与关闭](#ch07)
* [线程池的使用](#ch08)
* [图形用户界面应用程序](#ch09)
* [避免活跃性危险](#ch10)
* [性能和可伸缩性](#ch11)
* [并发程序的测试](#ch12)
* [显式锁](#ch13)
* [构建自定义的同步工具](#ch14)
* [原子变量与非阻塞同步机制](#ch15)
* [Java内存模型](#ch16)

<h2 id="ch01">简介</h2>
操作系统的出现使得计算机每次能运行多个程序，并且不同的程序都在单独的进程中运行：操作系统为每个独立执行的进程分配各种资源，包括内存，文件句柄以及安全证书等。如果需要的话，在不同的进程之间可以通过一些粗粒度的通信机制来交换数据，包括：套接字、信号处理器、共享内存、信号量以及文件等。

进程会共享进程范围内的资源，例如内存句柄和文件句柄，但每个线程都有各自的程序计数器、栈以及局部变量等。线程还提供了一种直观的分解模式来充分利用多处理器系统中的硬件并行性，而在同一个程序中的多个线程也可以被同时调度到多个CPU上运行。线程也被称为轻量级进程。在大多数操作系统上，都是以线程为基本的调度单位。

在设计良好的并发应用程序中，并发能提升程序的性能，但无论如何，线程总会带来某种程度的运行时开销。在多线程程序中，当线程调度器临时挂起活跃线程并转而运行另一个线程时，就会频繁地出现上下文切换操作，这种操作将带来极大的开销：保存和恢复执行上下文，丢失局部性，并且CPU时间将更多地话再线程调度而不是线程运行上。当线程共享数据时，必须使用同步机制，而这些同步机制往往会抑制某些编译器优化，使内存缓存区的数据无效，以及增加共享内存总线的同步流量。

<h2 id="ch02">线程安全性</h2>
要编写线程安全的代码，其核心在于要对状态访问操作进行管理，特别是对共享（shared）的和可变的（mutable）状态的访问。

如果当多个线程访问同一个可变的状态变量时没有使用合适的同步，那么程序就会出现错误。有三种方式可以修复这个问题：

+ 不在线程之间共享该状态变量
+ 将状态变量修改为不可变的变量
+ 在访问状态变量时使用同步

在并发编程中，这种由于不恰当的执行时序而出现不正确的结果是一种非常重要的情况，他有一个正式的名字：竞态条件（race condition）。

要避免竞态条件问题，就必须在某个线程修改该变量时，通过某种方式防止其他线程使用这个变量，从而确保其他线程只能在修改操作完成之前或之后读取和修改状态，而不是在修改状态的过程中。

Java提供了一种内置的锁机制来支持原子性：同步代码块（synchronized block）。同步代码块包括两部分：一个作为锁的对象引用，一个作为由这个锁保护的代码块。以关键字``synchronized``来修饰的方法就是一种横跨整个方法体的同步代码块，其中该同步代码块的锁就是方法调用所在的对象。静态的synchronized方法以class对象作为锁。

每个Java对象都可以用做一个可以实现同步的锁，这些锁被称为内置锁（intrinsic lock）或监视器锁（monitor lock）。线程在进入同步代码块之前会自动获取锁，并且在退出代码块时自动释放锁，而无论是通过正常的控制路径退出还是通过从代码块中抛出异常退出。获取内置锁的唯一途径是进入由这个锁保护的同步代码块或方法。Java内置锁相当于一种互斥体（或互斥锁），这意味着最多只有一个线程能持有这种锁。当线程A尝试获取一个由线程B持有的锁时，线程A必须等待或者阻塞，直到线程B释放这个锁。如果B永远不释放锁，那么A也将永远地等下去。

当某个线程请求一个由其他线程持有的锁时，发出请求的线程就会阻塞。然而，由于内置锁是可重入的，因此如果某个线程师徒获取一个已经由他自己持有的锁，那么这个请求就会成功。重入意味着获取锁的操作的粒度是线程而不是调用。重入的一种实现方法是，为每个锁关联一个获取计数值和一个所有者线程。当计数值为0时，这个锁就被认为是没有被任何线程持有。当线程请求一个未被持有的锁时，JVM将记下锁的持有者，并且将获取计数值置为1。如果同一个线程再次获取这个锁，计数值将递增，而当线程退出同步代码块时，计数值会相应地递减。当计数值为0时，这个锁将被释放。Java对象头的mark word会保存这些信息。

对于可能被多个线程同时访问的可变状态变量，在访问它的时候都需要持有同一个锁，在这种情况下，我们称状态变量是由这个锁保护的。一种常见的加锁约定是，将所有的可变状态都封装在对象内部，并通过对象的内置锁对所有访问可变状态的代码路径进行同步，是的在该对象上不会发生并发访问。在许多线程安全类中都使用了这种模式，例如Vector、Hashtable等其他同步集合类。

当执行时间较长的计算或者可能无法快速完成的操作时（例如，网络I/O或者控制台I/O），一定不要持有锁。

<h2 id="ch03">对象的共享</h2>
在没有同步的情况下，编译器、处理器以及运行时等都可能对操作的执行顺序进行一些意想不到的调整（重排序）。在缺乏足够同步的多线程程序中，要相对内存操作的执行顺序进行判断，几乎无法得出正确的结论。

Java内存模型要求，变量的读取操作和写入操作都必须是原子操作，但对于非volatile类型的long和double变量，JVM允许将64位的读操作或写操作分解为两个32位的操作。当读取一个非volatile类型的long变量时，如果对该变量的读操作和写操作是在不同的线程中执行，那么很可能读取到某个值的高32位和另一个值的低32位，因此，即使不考虑失效数据问题，在多线程程序中使用共享且可变的long和double等类型的变量也是不安全的，除非用volatile关键字来声明他们或者用锁保护起来。

Java提供了一种稍弱的同步机制，即volatile变量，用来确保将变量的更新操作通知到其他线程。当把变量声明为volatile类型后，编译器和运行时都会注意到这个变量是共享的，因此不会将该变量上的操作与其他内存操作一起重排序。volatile变量不会被缓存在寄存器或者对其他处理器不可见的地方，因此在读取volatile变量时总是返回最新写入的值。对volatile变量的写总是happens before对volatile变量的读。对volatile变量的写会导致其他处理器缓存行失效，会从内存中重新读取值。volatile变量通常用作某个操作完成、发生中断或者状态的标志。

加锁机制既可以保证可见性又可以保证原子性，而volatile变量只能保证可见性。volatile语义不足以确保递增操作的原子性。

当且仅当满足以下所有条件时，才应该使用volatile变量：

+ 对变量的写入操作不依赖变量的当前值，或者你能保证只有单个线程更新变量的值。
+ 该变量不会与其他状态变量一起纳入不可变条件中。
+ 在访问变量时不需要加锁。

当且仅当对象的构造函数返回时，对象才处于可预测的和一致的状态。因此，当从对象的构造函数中发布对象时，只是发布了一个尚未构造完成的对象。如果this引用在构造过程中逸出，那么这种对象就被认为是不正确构造。``不要在构造过程中使this引用逸出``

不可变对象一定是线程安全的。

当满足以下条件时，对象才是不可变的：

+ 对象创建之后其状态就不能修改
+ 对象的所有域都是final类型
+ 对象是正确创建的（在对象的创建期间，this引用没有逸出）

final类型的域是不能修改的（但如果final域引用的对象是可变的，那么这些被引用的对象是可以修改的）。Java内存模型中，final域还有着特殊的语义。final域能确保初始化过程的安全性，从而可以不受限制地访问不可变对象，并在共享这些对象时无需使用同步。

要安全地发布一个对象，对象的引用以及对象的状态必须同时对其他线程可见。一个正确构造的对象可以通过以下方式来安全地发布：

+ 在静态初始化函数中初始化一个对象引用
+ 将对象的引用保存到volatile类型的域或者AtomicReference对象中
+ 将对象的引用保存到某个正确构造对象的final域中
+ 将对象的引用保存到一个由锁保护的域中

在并发程序中使用和共享对象时，可以使用一些实用的策略，包括：

+ 线程封闭。线程封闭的对象只能由一个线程拥有，对象被封闭在该线程中，并且只能由这个线程修改，如ThreadLocal对象
+ 只读共享。在没有额外同步的情况下，共享的只读对象可以由多个线程并发访问，但任何线程都不能修改它。共享的只读对象包括不可变对象和事实不可变对象。final对象
+ 线程安全的共享。线程安全的对象在其内部实现同步，因此多个线程可以通过对象的公有接口来进行访问而不需要进一步的同步。如使用线程安全的容器
+ 保护对象。被保护的对象只能通过持有特定的锁来访问。保护对象包括封装在其他线程安全对象中的对象，以及已发布的并且由某个特定锁保护的对象。

<h2 id="ch04">对象的组合</h2>
在设计线程安全类的过程中，需要包含以下三个基本要素：

+ 找出构成对象状态的所有变量
+ 找出约束状态变量的不变性条件
+ 建立对象状态的并发访问管理策略

如果不了解对象的不变性条件与后验条件，那么就不能确保线程安全性。要满足在状态变量的有效值或状态转换上的各种约束条件，就需要借助于原子性与封装性。

<h2 id="ch05">基础构建模块</h2>
同步容器类包括Vector和Hashtable，二者是早期JDK的一部分，此外还包括在JDK1.2中添加的一些功能相似的类，这些同步的封装器类是由``Collections.synchronizedXXX``等工厂方法创建的。这些类实现线程安全的方式是：将他们的状态封装起来，并对每个公有方法都进行同步，使得每次只有一个线程能访问容器的状态。

同步容器类都是线程安全的，但在某些情况下可能需要额外的客户端加锁来保护复合操作。容器上常见的复合操作包括：迭代（反复访问元素，直到遍历完容器中所有元素）、跳转（根据指定顺序找到当前元素的下一个元素）以及条件运算（如“若没有就添加”，检查Map中是否存在键值K，如果没有，就加入二元组(K,V)）。在同步容器类中，这些复合操作在没有客户端加锁的情况下仍然是线程安全的，但当其他线程并发地修改容器时，它们可能会表现出意料之外的行为。

许多现代的容器类也并没有消除复合操作中的问题。无论在直接迭代还是在Java5.0引入的for-each循环语句中，对容器类进行迭代的标准方式都是使用Iterator。然而，如果有其他线程并发地修改容器，那么即使是使用迭代器也无法避免在迭代期间对容器加锁。在设计同步容器类的迭代器时并没有考虑到并发修改的问题，并且他们表现出的行为是及时失败fail-fast的。这意味着，当他们发现容器在迭代过程中被修改时，就会抛出一个ConcurrentModificationException异常。

容器的toString、hashCode、equals等方法也会间接地执行迭代操作，当容器作为另一个容器的元素或键值时，就会出现这种情况。同样，containsAll、removeAll、retainAll等方法，以及把容器作为参数的构造函数，都会对容器进行迭代。所有这些间接的迭代操作都可能抛出ConcurrentModificationException。

通过并发容器来代替同步容器，可以极大地提高伸缩性并降低风险。

Java5.0增加了两种新的容器类型：Queue和BlockingQueue。Queue用来临时保存一组等待处理的元素。它提供了几种实现，包括ConcurrentLinkedQueue，这是一个传统的先进先出队列，以及PriorityQueue，这是一个非并发优先队列。Queue上的操作不会阻塞，如果队列为空，那么获取元素的操作将返回空值。虽然可以用List来模拟Queue的行为——事实上，正是通过LinkedList来实现Queue的，但还需要一个Queue的类，因为它能去掉List的随机访问需求，从而实现更高效的并发。

BlockingQueue扩展了Queue，增加了可阻塞的插入和获取等操作。如果队列为空，那么获取元素的操作将一直阻塞，知道队列中出现一个可用的元素。如果队列已满（对于有界队列来说），那么插入操作将一直阻塞，只到队列中出现ke'yong的空间。

正如ConcurrentHashMap用于代替基于散列的同步Map，Java6也引入了ConcurrentSkipListMap和ConcurrentSkipListSet，分别作为同步的SortedMap和SortedSet的并发替代品（例如用synchronizedMap包装起来的TreeMap或TreeSet）。

ConcurrentHashMap使用分段锁来实现更大程度的共享，任意数量的读线程可以并发地访问Map，执行读取操作的线程和执行写入操作的线程可以并发的访问Map，并且一定数量的写入线程可以并发地修改Map。

```java
public interface ConcurrentMap<K,V> extends Map<K,V> {
    //仅当K没有相应的映射值时才插入
    V putIfAbsent(K key,V value);
    //仅当K被映射到V时才移除
    boolean remove(K key,V value);
    //仅当K被映射到oldValue时才替换为newValue
    boolean replace(K key,V oldValue,V newValue);
    //仅当K被映射到某个值时才替换为newValue
    V replace(K key, V newValue);
}
```

阻塞队列提供了可阻塞的put和take方法，以及支持定时的offer和poll方法。

Java6增加了两种容器类型，Deque和BlockingDeque，他们分别对Queue和BlockingQueue进行了扩展。Deque是一个双端队列，实现了在队列头和队列尾的高效插入和移除。具体实现包括ArrayDeque和LinkedBlockingDeque。正如阻塞队列适用于生产者--消费者模式，双端队列同样适用于另一种相关模式，即工作密取。在生产者——消费者设计中，所有消费者有一个共享的工作队列，而在工作密取中，每个消费者都有各自的双端队列。如果一个消费者完成了自己双端队列中的全部工作，那么它可以从其他消费者双端队列末尾秘密地获取工作。密取工作模式比传统的生产者——消费者模式具有更高的可伸缩性，这是因为工作者线程不会在单个共享的任务队列上发生竞争。大多数时候，它们都只是访问自己的双端队列，从而极大地减少了竞争。当工作者线程需要访问另一个队列时，它会从队列的尾部而不是头部获取工作，因此进一步降低了队列上的竞争。工作密取非常适用于既是消费者也是生产者问题——当执行某个工作时可能导致出现更多的工作。例如，在网页爬虫程序中处理一个页面时，通常会发现有更多的页面需要处理。

线程可能会阻塞或暂停执行，原因有多种：等待I/O操作结束，等待获取一个锁，等待从Thread.sleep方法中醒来，或是等待另一个线程的计算结果。当线程阻塞时，它通常被挂起，并处于某种阻塞状态(Blocked、Waiting、Timed_Waiting)。阻塞操作与执行时间很长的普通操作的差别在于，被阻塞的线程必须等待某个不受它控制的事件发生后才能继续执行，例如等待I/O操作完成，等待某个锁变成可用，或者等待外部计算的结束。当某个外部事件发生时，线程被置回Runnable，并可以再次被调度执行。

当某方法抛出InterruptException时，表示该方法是一个阻塞方法，如果这个方法被中断，那么它将努力提前结束阻塞状态。Thread提供了interrupt方法，用于中断线程或查询线程是否已经被中断。每个线程都有一个boolean类型的属性，表示线程的中断状态，当中断线程时将设置这个状态。

当在代码中调用了一个将抛出InterruptException异常的方法时，你自己的方法也就变成了一个阻塞方法，并且必须要处理对中断的响应。有两种基本选择：

+ 传递InterruptException
+ 恢复中断，有时候不能抛出InterruptException,例如当代码是Runnable的一部分时，在这些情况下，必须捕获InterruptException，并通过调用当前线程上的interrupt方法恢复中断状态，这样在调用栈中更高层的代码将看到引发了一个中断。

阻塞队列可以作为同步工具类，其他类型的同步工具类还包括信号量(Semaphore)、栅栏（Barrier）、以及闭锁（Latch）。

所有的同步工具类都包含一些特定的结构化属性：它们封装了一些状态，这些状态将决定执行同步工具类的线程是继续执行还是等待，此外还提供了一些方法对状态进行操作，以及另一些方法用于高效地等待同步工具类进入到预期状态。

闭锁是一种同步工具类，可以延迟线程的进度直到其到达终止状态。闭锁的作用相当于一扇门：在闭锁到达结束状态之前，这扇门一直是关闭的，并且没有任何线程能通过，当到达结束状态时，这扇门会打开并允许所有线程通过。当闭锁到达结束状态后，将不会再改变状态，隐藏这扇门将永远保持打开状态。闭锁可以用来确保某些活动指导其他活动都完成后才继续执行。

CountDownLatch是一种灵活的闭锁实现。

```java
public static long timeTasks(int num, final Runnable task) throws InterruptedException{
        final CountDownLatch startGate = new CountDownLatch(1);
        final CountDownLatch endGate = new CountDownLatch(num);

        for (int i = 0; i < num; i++) {
            Thread t = new Thread(new Runnable() {
                @Override
                public void run() {
                    try {
                        startGate.await();
                        try {
                            task.run();
                        } finally {
                            endGate.countDown();
                        }
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            });
            t.start();
        }
        long start = System.nanoTime();
        startGate.countDown();
        endGate.await();
        long end = System.nanoTime();
        return end - start;
    }
```

FutureTask也可以用作闭锁。FutureTask实现了Future语义，表示一种抽象的可生成结果的计算。FutureTask表示的计算是通过Callable来实现的，相当于一种可生成结果的Runnable，并且可以处于以下3种状态：等待运行Waiting to run，正在运行Running，运行完成Completed。运行完成表示计算的所有可能结束方式，包括正常结束、由于取消而结束和由于异常而结束等。FutureTask进入完成状态后，它会永远停留在这个状态上。

计数信号量（Counting Semaphore）用来控制同时访问某个资源的操作数量，或者同时执行某个特定操作的数量。计数信号量还可以用来实现某种资源池，或者对容器施加边界。Semaphore中管理着一组虚拟的许可，许可的初始数量可通过构造函数来指定。在执行操作时可以首先获取许可（只要还有剩余的许可），并在使用以后释放许可。如果没有许可，那么acquire将阻塞直到有许可（或者直到被中断或者操作超时）。release方法将返回一个许可给Semaphore。计数信号量的一种简化形式是二值信号量，即初始值为1的Semaphore，二值信号量可以用作互斥体Mutex，并具备不可重入的加锁语义：谁拥有这个唯一的许可，谁就拥有了互斥锁。

```java
public class BoundedHashSet<T> {
    private final Set<T> set;
    private final Semaphore semaphore;

    public BoundedHashSet(int bound) {
        this.set = Collections.synchronizedSet(new HashSet<T>());
        this.semaphore = new Semaphore(bound);
    }

    public boolean add(T o) throws InterruptedException {
        semaphore.acquire();
        boolean wasAdded = false;
        try {
            wasAdded = set.add(o);
            return wasAdded;
        } finally {
            if (!wasAdded) {
                semaphore.release();
            }
        }
    }

    public boolean remove(T o) {
        boolean wasRemoved = set.remove(o);
        if (wasRemoved) {
            semaphore.release();
        }
        return wasRemoved;
    }
}
```

闭锁是一次性对象，一旦进入终止状态，就不能被重置。栅栏（Barrier）类似于闭锁，它能一直阻塞一组线程直到某个事件发生。栅栏与闭锁的关键区别在于，所有线程必须同时到达栅栏位置，才能继续执行。闭锁用于等待事件，而栅栏用于等待其他线程。

CyclicBarrier可以使一定数量的参与方反复地在栅栏位置汇集，它在并行迭代算法中非常有用：这种算法通常将一个问题拆分成一系列相互独立的子问题。当线程到达栅栏位置时将调用await方法，这个方法将阻塞直到所有线程都到达栅栏位置。如果所有线程都到达了栅栏位置，那么栅栏将打开，此时所有线程都被释放，而栅栏将被重置以便下次使用。如果对await的调用超时，或者await阻塞的线程被中断，那么栅栏就被认为是打破了，所有阻塞的await调用都将终止并抛出BrokenBarrierException。如果成功地通过栅栏，那么await将为每个线程返回一个唯一的到达索引号，我们可以利用这些索引来选举产生一个领导线程，并在下一次迭代中由该领导线程执行一些特殊的工作。CyclicBarrier还可以使你将一个栅栏操作传递给构造函数，这是一个Runnable，当成功通过栅栏时会在一个子任务线程中执行它，但在阻塞线程被释放之前是不能执行的。

另一种形式的栅栏是Exchanger，它是一种两方栅栏，各方在栅栏位置上交换数据。当两方执行不对称的操作时，Exchanger会非常有用，例如当一个线程向缓冲区写入数据，而另一个线程从缓冲区读取数据。这些线程可以使用Exchanger来汇合，并将满的缓冲区与空的缓冲区交换。当两个线程通过Exchanger交换对象时，这种交换就把这两个对象安全地发布给另一方。

```java
public class Memoizer<A, V> implements Computable<A, V> {
    private final ConcurrentHashMap<A, Future<V>> cache = new ConcurrentHashMap<>();
    private final Computable<A, V> delegate;

    public Memoizer(Computable<A, V> delegate) {
        this.delegate = delegate;
    }

    @Override
    public V compute(A arg) throws InterruptedException {
        while (true) {
            Future<V> f = cache.get(arg);
            if (f == null) {
                Callable<V> eval = new Callable<V>() {
                    @Override
                    public V call() throws Exception {
                        return delegate.compute(arg);
                    }
                };
                FutureTask<V> ft = new FutureTask<V>(eval);
                f = cache.putIfAbsent(arg, ft);
                if (f == null) {
                    f = ft;
                    ft.run();
                }
            }
            try {
                return f.get();
            } catch (CancellationException e) {
                cache.remove(arg, f);
            } catch (ExecutionException e) {
                e.printStackTrace();
            }
        }
    }
}

interface Computable<A, V> {
    V compute(A arg) throws InterruptedException;
}
```

并发技巧清单：

+ 可变状态是至关重要的。所有的并发问题都可以归结为如何协调对并发状态的访问。可变状态越少，就越容易确保线程安全性。
+ 尽量将域声明为final类型，除非需要他们是可变的。
+ 不可变对象一定是线程安全的。不可变对象能极大地降低并发编程的复杂性。它们更为简单而且安全，可以任意共享而无需使用加锁或保护性复制等机制。
+ 封装有助于管理复杂性。在编写线程安全的程序时，虽然可以将所有数据都保存在全局变量中，但为什么要这么做？将数据封装在对象中，更易于维持不变性条件：将同步机制封装在对象中，更易于遵循同步策略。
+ 用锁来保护每个可变变量。
+ 当保护同一个不变性条件中的所有变量时，要使用同一把锁。
+ 在执行复合操作期间，要持有锁。
+ 如果从多个线程中访问同一个可变变量时没有使用同步机制，那么线程会出现问题。

<h2 id="ch06">任务执行</h2>

+ 线程生命周期的开销非常高。线程的创建和销毁并不是没有代价的，根据平台的不同，实际的开销也有所不同，单线程的创建过程都会需要时间，延迟处理的请求，并且需要JVM和操作系统提供一些辅助操作。
+ 资源消耗，活跃的线程会消耗系统资源，尤其是内存。如果可运行的线程数量多于可用处理器的数量，那么有些线程将被闲置。大量空闲的线程会占用许多内存，给垃圾回收器带来压力，而且大量线程在竞争CPU资源还将产生其他的性能开销。
+ 稳定性，在创建线程的数量上存在一个限制。这个限制值将随着平台的不同而不同，并且受多个因素制约，包括JVM的启动参数，Thread构造函数中请求的栈大小，以及底层操作系统对线程的限制等。在32位机器上，其中一个主要的限制因素是线程栈的地址空间。每个线程都维护两个执行栈，一个用于Java代码，一个用于Native代码。通常，JVM在默认情况下会生成一个复合的栈，大小约为0.5MB。可以通过JVM标志-Xss或者通过Thread的构造函数来修改这个值。如果将2^32处以每个线程的栈大小，那么线程数量将被限制为几千到几万。如果破坏了这些限制，那么很可能抛出OutOfMemoryError异常。

Timer类负责管理延迟任务以及周期任务。然而，Timer存在一些缺陷，应该考虑使用ScheduledThreadPoolExecutor来代替它。

+ Timer在执行所有的定时任务时只会创建一个线程。如果某个任务的执行时间过长，那么将破坏其他TimerTask的定时准确性。
+ 如果TimerTask抛出了一个未检查的异常，那么Timer将表现出糟糕的行为。Timer线程并不捕获异常，因此当TimerTask抛出未检查的异常时将终止定时线程。这种情况下，Timer也不会恢复线程的执行，而是会错误地认为整个Timer都被取消了。因此，已经被调度但尚未执行的TimerTask将不会再执行，新的任务也不能被调度。

如果要构建自己的调度服务，那么可以使用DelayQueue，它实现了BlockingQueue，并为ScheduledThreadPoolExecutor提供调度功能。DelayQueue管理着一组Delayed对象。每个Delayed对象都有一个相应的延迟时间：在DelayQueue中，只有某个元素逾期后，才能从DelayQueue中执行take操作。从DelayQueue中返回的对象将根据他们的延迟时间进行排序。

如果向Executor提交了一组计算任务，并且希望在计算完成后获得结果，那么可以保留与每个任务关联的Future，然后反复使用get方法，同时将参数timeout指定为0，从而通过轮询来判断任务是否完成。这种方法虽然可行，但却有些繁琐。幸运的是，还有一种更好的方法：CompletionService。CompletionService将Executor和BlockingQueue的功能融合在一起。可以将Callable任务提交给他来执行，然后使用类似于队列操作的take和poll等方法来获得已完成的结果，而这些结果会在完成时将被封装为Future。ExecutorCompletionService实现了CompletionService，并将计算部分委托给一个Executor。

ExecutorCompletionService的实现非常简单。在构造函数中创建一个BlockingQueue来保存计算完成的结果。当计算完成时，调用FutureTask中的done方法。当提交任务时，任务将首先被包装为一个QueueingFuture，这是FutureTask的一个子类，然后再改写子类的done方法，并将结果放入BlockingQueue中。

<h2 id="ch07">取消与关闭</h2>
每个线程都有一个boolean类型的中断状态。当中断线程时，这个线程的中断状态将被设置为true。在Thread中包含了中断线程以及查询线程中断状态的方法。``interrupt``方法能中断目标线程，而``isInterrupted``方法能返回目标线程的中断状态。静态的``interrupted``方法将
清除当前线程的中断状态，并返回它之前的状态，这也是清除中断状态的唯一方法。

```java
public class Thread {
    public void interrupt() {...}
    public boolean isInterrupted() {...}
    public static boolean interrupted() {...}
}
```

阻塞库方法，例如``Thread.sleep``和``Object.wait``等，都会检查线程何时中断，并且在发现中断时提前返回。它们在响应中断时执行的操作包括：清除中断状态，抛出InterruptedException，表示阻塞操作由于中断而提前结束。

对中断操作的正确理解是：它并不会真正地中断一个正在运行的线程，而只是发出中断请求，然后由线程在下一个合适的时刻中断自己。有些方法，例如wait、sleep、join等，都严格地处理这种请求，当它们收到中断请求或者在开始执行时发现某个已被设置好的中断状态时，将抛出一个异常。设计良好的方法可以完全忽略这种请求，只要它们能使调用代码对中断请求进行某种处理。设计糟糕的方法可能会屏蔽中断请求，从而导致调用栈中的其他代码无法对中断请求做出响应。

在使用静态的interrupted时应该小心，因为它会清除当前线程的中断状态。如果在调用interrupted时返回了true，那么除非你想屏蔽这个中断，否则必须对它进行处理——可以抛出InterruptedException，或者通过再次调用interrupt来恢复中断状态。

ExecutorService.submit将返回一个Future来描述任务。Future拥有一个cancel方法，该方法带有一个boolean类型的参数mayInterruptIfRunning，表示取消操作是否成功（这只是表示任务是否能够接收中断，而不是表示任务是否能检测并处理中断）。如果这个参数为false，那么意味着“如果任务还没有启动，就不要运行它”，这种方式应该用于那些不处理中断的任务中。

ExecutorService提供了两种关闭方法，使用shutdown正常关闭，以及使用shutdownNow强行关闭。在进行强行关闭时，shutdownNow首先关闭当前正在执行的任务，然后返回所有尚未启动的任务清单。

<h2 id="ch08">线程池的使用</h2>
线程池的基本大小Core Pool Size、最大大小Max Pool Size以及存活时间等因素共同负责线程的创建和销毁。基本大小就是线程池的目标大小，即在没有任务执行时线程池的大小，并且只有在工作队列满了的情况下才会创建超出这个数量的线程。线程池的最大大小表示可同时活动的线程数量的上限。如果某个线程的空闲时间超过了存活时间，那么将被标记为可回收的，并且当线程池的当前大小超过了基本大小时，这个线程将被终止。

ThreadPoolExecutor允许提供一个BlockingQueue来保存等待执行的任务。基本的任务排队方法有3种：无界队列、有界队列和同步移交（Synchronous Handoff）。

newFixedThreadPool和newSingleThreadPoolExecutor在默认情况下将使用一个无界的LinkedBlockingQueue。如果所有工作者线程都处于忙碌状态，那么任务将在队列中等待。如果任务持续快速的到达，并且超过了线程池处理他们的速度，那么队列将无限制的增加。

一种更稳妥的资源管理策略是使用有界队列，例如ArrayBlockingQueue、有界的LinkedBlockingQueue、PriorityBlockingQueue。

对于非常大的或者无界的线程池，可以通过使用SynchronousQueue来避免任务排队，以及直接将任务从生产者移交给工作者线程。SynchronousQueue不是一个真正的队列，而是一种在线程之间进行移交的机制。要将一个元素放入SynchronousQueue中，必须有另一个线程正在等待接收这个元素，如果没有线程正在等待，并且线程池的当先大小小于最大值，那么ThreadPoolExecutor将创建一个新的线程，否则根据饱和策略，这个任务将被拒绝。使用直接一脚将更高效，因为任务会直接移交给执行它的线程，而不是被首先放到队列中。只有当线程池是无界的或者可以拒绝任务时，SynchronousQueue才有实际价值。在newCachedThreadPool工厂方法中就使用了SynchronousQueue。

当有界队列被填满后，饱和策略开始发挥作用。JDK提供了几种不同的RejectedExecutionHandler实现，有AbortPolicy、CallerRunsPolicy、DiscardPolicy、DiscardOldestPolicy。

<h2 id="ch09">图形用户界面应用程序</h2>
<h2 id="ch10">避免活跃性危险</h2>
我们使用加锁机制来确保线程安全，单如果过度地使用加锁，则可能导致锁顺序死锁。同样，使用线程池和信号量来限制对资源的使用，单这些被限制的行为可能会导致资源死锁。

在制定锁的顺序时，可以使用System.identityHashCode方法，该方法将返回由Object.hashCode返回的值。在极少数的情况下，两个对象可能拥有相同的散列值，此时必须通过某种任意的方法来决定锁的顺序，而这可能又会重新引入死锁。为了避免这种情况，可以使用“加时赛”锁。在获得两个Account锁之前，首先获得这个“加时赛”锁，从而保证每次只有一个线程以未知的顺序获得这两个锁，从而消除了死锁发生的可能性。

当使用内置锁的时候，只要没有获取锁，就会永远等待下去，而显式锁则可以指定一个超时时限，在等待超过该时间后tryLock会返回一个失败信息。

当线程由于无法访问它所需要的资源而不能继续执行时，就发生了饥饿。

活锁时另一种形式的活跃性问题，该问题尽管不会阻塞线程，但也不能继续执行，因为线程将不断重复执行相同的操作，而且总会失败。活锁通常发生在处理事务消息的应用程序中：如果不能成功地处理某个信息，那么消息处理机制将回滚整个事务，并将它重新放到队列的开头。如果消息处理器在处理某种特定类型的消息时存在错误并导致它失败，那么每当这个消息从队列中取出并传递到存在错误的处理器时，都会发生事务回滚。由于这条消息又被放回到队列开头，隐藏处理器将被反复调用，并返回相同的结果（有时候也被称为毒丸消息）。虽然处理消息的线程并没有阻塞，但也无法继续执行下去。这种形式的活锁通常是由过度的错误恢复代码造成的，因为它错误地将不可修复的错误作为可修复的错误。要解决这种活锁问题，需要在重试机制中引入随机性。在并发应用程序中，通过等待随机长度的时间和回退可以有效地避免活锁的发生。

<h2 id="ch11">性能和可伸缩性</h2>
同步操作的性能开销包括多个方面。在synchronized和volatile提供的可见性保证中可能会使用一些特殊指令，即内存栅栏。内存栅栏可以刷新缓存，使缓存无效，刷新硬件的写缓冲，以及停止执行管道。内存栅栏可能同样会对性能带来间接地影响，因为他们将抑制一些编译器优化操作。在内存栅栏中，大多数操作都是不能被重排序的。

缩小所得范围，快进快出。

减小锁的粒度，锁分段。
<h2 id="ch12">并发程序的测试</h2>
<h2 id="ch13">显式锁</h2>

```java
public interface Lock {
    void lock();
    void lockInterruptibly() throws InterruptedException;
    boolean tryLock();
    boolean tryLock(long timeout, TimeUnit unit) throws InterruptedException;
    void unlock();
    Condition newCondition();
}
```

在大多数情况下：内置锁都能很好地工作，但在功能上存在一些局限性，例如，无法中断一个正在等待获得锁的线程，或者无法在请求获取一个锁时无限地等待下去。内置锁必须在等待获取所得代码块中释放，这就简化了编码工作，并且与异常处理操作实现了很好地交互，但却无法实现非阻塞结构的加锁规则。

```java
Lock lock = new ReentrantLock();
...
lock.lock();
try {
    // 更新对象状态
    // 捕获异常，并在必要时恢复不变性条件
} finally {
    lock.unlock();
}
```

ReentrantLock的构造函数中提供了两种公平性选择：创建一个非公平的锁（默认）或者一个公平的锁。在公平的锁上，线程将按照他们发出请求的顺序来获得锁，但在非公平的锁上，则允许插队：当一个线程请求非公平的锁时，如果在发出请求的同时该锁的状态变为可用，那么这个线程将跳过队列中所有的等待线程并获得这个锁。（在Semaphore中同样可以选择采用公平的或者非公平的获取顺序）非公平的ReentrantLock并不提倡插队行为，但无法防止某个线程在合适的时候进行插队。在公平的锁中，如果有另一个线程持有这个锁或者有其他线程在队列中等待这个锁，那么发出请求的线程将被放入队列中。在非公平的锁中，只有当锁被某个线程持有时，新发出请求的线程才会被放入队列中。

在大多数情况下，非公平锁的性能要高于公平锁的性能。

当持有锁的时间相对较长，或者请求锁的平均时间间隔较长，那么应该使用公平锁。在这些情况下，插队带来的吞吐量提升（当锁处于可用状态时，线程还处于被唤醒的过程中）则可能不会出现。

在一些内置锁无法满足需求的情况下，ReentrantLock可以作为一种高级工具。当需要一些高级功能时才应该使用ReentrantLock，这些功能包括：可定时的、可轮询的与可中断的锁获取操作，公平队列，以及非块结构的锁。否则，还是应该优先使用synchronized。

读写锁：一个资源可以被多个读操作访问，或者被一个写操作访问，但两者不能同时进行。

```java
public interface ReadWriterLock {
    Lock readLock();
    Lock writeLock();
}
```

ReentrantReadWriteLock为这两种锁都提供了可重入的加锁语义。与ReentrantLock泪水，ReentrantReadWriteLock在构造时也可以选择是一个非公平的锁（默认）还是一个公平的锁。在公平锁上，等待时间最长的线程将优先获得锁。如果这个锁由读线程持有，而另一个线程请求写入锁，那么其他读线程都不能获得读取锁，直到写线程使用完并释放了写入锁。在非公平的锁中，线程获得访问许可的顺序是不确定的。写线程可以降级为读线程，但从读线程升级为写线程是不可以的（这样做会导致死锁）。与ReentrantLock类似的是，ReentrantReadWriteLock中的写入锁只能有唯一的所有者，并且只能由获得该锁的线程来释放。在Java5.0中，读取锁的行为更类似于一个Semaphore而不是锁，它只维护活跃的读线程的数量，而不考虑它们的标识。在Java6.0中修改了这个行为：记录那些线程已经获得了读者锁。做出这种修改的原因是：在Java5.0的锁实现中，无法区别一个线程是首次请求读取锁，还是可重入锁请求，从而可能使公平的读——写锁发生死锁。

当锁的持有时间较长并且大部分操作都不会修改被守护的资源时，那么读——写锁能提高并发性。
<h2 id="ch14">构建自定义的同步工具</h2>
要想正确地使用条件队列，关键是找出对象在哪个条件谓词上等待。条件谓词是使某个操作成为状态依赖操作的前提条件。在有界缓存中，只有当缓存不为空时，take方法才能执行，否则必须等待。对take方法来说，它的条件谓词就是“缓存不为空”，take方法在执行之前必须首先测试该条件谓词。同样，put方法的条件谓词是“缓存不满”。条件谓词是由类中各个状态变量构成的表达式。

当使用条件等待时（例如Object.wait或Condition.await）:

+ 通常都有一个条件谓词——包括一些对象状态的测试，线程在执行前必须首先通过这些测试。
+ 在调用wait之前测试条件谓词，并且从wait中返回时再次进行测试
+ 在一个循环中调用wait
+ 确保使用与条件队列相关的锁来保护构成条件谓词的各个状态变量
+ 当调用wait、notify、notifyAll等方法时，一定要持有与条件队列相关的锁
+ 在检查条件谓词之后以及开始执行相应的操作之前，不要释放锁。

在等待一个条件时，一定要确保在条件谓词变为真时通过某种方式发出通知。

在条件队列API中有两个发出通知的方法，即notify和notifyAll。无论调用哪一个，都必须持有与条件队列对象相关联的锁。在调用notify时，JVM会从这个条件队列上等待的多个线程中选择一个唤醒，而调用notifyAll则会唤醒所有在这个条件队列上等待的线程。由于在调用notify或notifyAll时必须持有条件队列对象的锁，而如果这些等待中线程此时不能重新获得锁，那么无法从wait返回，因此发出通知的线程应该尽快地释放锁，从而确保正在等待的线程尽可能快地解除阻塞。

```java
public class ConditionBoundedBuffer<T> {
    private final int BUFFER_SIZE = 50;
    protected final Lock lock = new ReentrantLock();
    // 条件谓词 notFull (count < items.length)
    private final Condition notFull = lock.newCondition();
    // 条件谓词 notEmpty (count > 0)
    private final Condition notEmpty = lock.newCondition();
    private final T[] items = (T[])new Object[BUFFER_SIZE];
    private int tail,head,count;

    // 阻塞并直到notFull
    public void put(T x) throws InterruptedException {
        lock.lock();
        try {
            while (count == items.length) {
                notFull.await();
            }
            items[tail] = x;
            if (++tail == items.length) {
                tail = 0;
            }
            ++count;
            notEmpty.signal();
        } finally {
            lock.unlock();
        }
    }

    // 阻塞直到notEmpty
    public T take() throws InterruptedException {
        lock.lock();
        try {
            while (count == 0) {
                notEmpty.await();
            }
            T x = items[head];
            items[head] = null;
            if (++head == items.length) {
                head = 0;
            }
            --count;
            notFull.signal();
            return x;
        } finally {
            lock.unlock();
        }
    }
}
```

在基于AQS构建的同步器类中，最基本的操作包括各种形式的获取操作和释放操作。获取操作时一种依赖状态的操作，通常会阻塞。当使用锁或信号量时，获取操作的含义就很直观，即获取的是锁或者许可，并且调用者可能会一直等待直到同步器类处于可被获取的状态。在使用CountDownLatch时，获取操作意味着“等待并直到闭锁到达结束状态”，而在使用FutureTask时，则意味着“等待并直到任务已经完成”。“释放”并不是一个可阻塞的操作，当执行“释放”操作时，所有在请求时被阻塞的线程都会开始执行。

如果一个类想成为状态依赖的类，那么它必须拥有一些状态。AQS负责管理同步器类中的状态，它管理了一个整数状态信息，可以通过getState，setState以及compareAndSetState等protected类型方法来进行操作。这个整数可以用于表示任意状态。例如，ReentrantLock用它来表示所有者线程已经重复获取该锁的次数，Semaphore用它来表示剩余的许可数量，FutureTask用它来表示任务的状态（尚未开始、正在运行、已完成以及已取消）。在同步器类中还可以自行管理一些额外的状态变量，例如，ReentrantLock保存了锁的当前所有者的信息，这样就能区分某个获取操作时重入的还是竞争的。

```java
boolean acquire() throws InterruptedException {
    while (当前状态不允许获取操作) {
        if(需要阻塞获取请求){
            如果当前线程不在队列中，则将其插入队列
            阻塞当前线程
        } else {
            返回失败
        }
    }
    可能更新同步器的状态
    如果线程位于队列中，则将其移出队列
    返回成功
}

void release() {
    更新同步器的状态
    if(新的状态允许某个被阻塞的线程获取成功) {
        解除队列中一个或多个线程的阻塞状态
    }
}
```

ReadWriteLock接口表示存在两个锁：一个读取锁和一个写入锁，但在基于AQS实现的ReentrantReadWriteLock中，单个AQS子类将同时管理读取加锁和写入加锁。ReentrantReadWriteLock使用了一个16位的状态来表示写入锁的计数，并且使用了另一个16位的状态来表示读取锁的计数。在读取锁上的操作将使用共享的获取方式与释放方法，在写入锁上的操作将使用独占的获取方法与释放方法。

在AQS内部维护一个等待线程队列，其中记录了某个线程请求的事独占访问还是共享访问。在ReentrantReadWriteLock中，当锁可用时，如果位于队列头部的线程执行写入操作，那么线程会得到这个锁，如果位于队列头部的线程执行读取访问，那么队列中在第一个写入线程之前的所有都将获取这个锁。

<h2 id="ch15">原子变量与非阻塞同步机制</h2>
原子变量提供了与volatile类型变量相同的内存语义，此外还支持原子的更新操作，从而使他们更适用于实现计数器、序列发生器和统计数据收集等，同时还能比基于锁的方法提供更高的可伸缩性。

CAS包含了3个操作数——需要读写的内存位置V、进行比较的值A和拟写入的新值B，当且仅当V的值等于A时，CAS才会通过原子方式用新值B来更新V的值，否则不会执行任何操作。无论位置V的值是否等于A，都将返回V原有的值。

当多个线程尝试使用CAS同时更新同一个变量时，只有一个线程能更新变量的值，而其他的线程都将失败。然而，失败的线程并不会被挂起（这与获取锁的情况不同：当获取锁失败时，线程将被挂起），而是被告知在这次竞争中失败，并可以再次尝试。由于一个线程在竞争CAS时失败不会阻塞，因此它可以决定是否重新尝试，或者执行一些恢复操作，也或者不执行任何操作。

CAS的主要缺点是，它将使调用者处理竞争问题（通过重试、回退、放弃），而在锁中能自动处理竞争问题（线程获取锁之前将一直阻塞）。

```java
public class ConcurrentStack<E> {
    private static class Node<E> {
        public final E item;
        public Node<E> next;

        public Node(E item) {
            this.item = item;
        }
    }

    AtomicReference<Node<E>> top = new AtomicReference<>();

    public void push(E item) {
        Node<E> newHead = new Node<>(item);
        Node<E> oldHead;
        do {
            oldHead = top.get();
            newHead.next = oldHead;
        } while (!top.compareAndSet(oldHead, newHead));
    }

    public E pop() {
        Node<E> oldHead;
        Node<E> newHead;
        do {
            oldHead = top.get();
            if (oldHead == null) {
                return null;
            }
            newHead = oldHead.next;
        } while (!top.compareAndSet(oldHead, newHead));
        return oldHead.item;
    }
}
```

```java
public class LinkedQueue<E> {
    private static class Node<E> {
        final E item;
        final AtomicReference<Node<E>> next;

        public Node(E item, Node<E> next) {
            this.item = item;
            this.next = new AtomicReference<>(next);
        }
    }

    private final Node<E> dummy = new Node<E>(null, null);
    private final AtomicReference<Node<E>> head = new AtomicReference<>(dummy);
    private final AtomicReference<Node<E>> tail = new AtomicReference<>(dummy);

    public boolean put(E item) {
        Node<E> newNode = new Node<>(item, null);
        while (true) {
            Node<E> curTail = tail.get();
            Node<E> curTailNext = curTail.next.get();
            if (curTail == tail.get()) {
                if (curTailNext != null) {
                    tail.compareAndSet(curTail, curTailNext);
                } else {
                    if (curTail.next.compareAndSet(null, newNode)) {
                        tail.compareAndSet(curTail, newNode);
                        return true;
                    }
                }
            }
        }

    }
}
```

解决ABA问题，不是更新某个引用的值，而是更新两个值，包括一个引用和一个版本号。

在基于锁的算法中可能会发生活跃性故障。如果线程在持有锁时由于阻塞I/O，内存也缺失或其他延迟而导致推迟执行，那么很可能所有线程都不能继续执行下去。如果在某种算法中，一个线程的失败或挂起不会导致其他线程也失败或挂起，那么这种算法就被称为非阻塞算法。如果在算法的每个步骤中都存在某个线程能够执行下去，那么这种算法也被称为无锁算法。如果在算法中仅将CAS用于协调线程之间的操作，并且能正确地实现，那么它既是一种无阻塞算法又是一种无锁算法。
<h2 id="ch16">Java内存模型</h2>
Java内存模型是通过各种操作来定义的，包括对变量的读、写操作，监视器的加锁和释放操作，以及线程的启动和合并操作。JMM为程序中所有的操作定义了一个偏序关系，称之为Happens-Before。

Happens-Before的规则包括：

+ 程序顺序规则。如果程序中操作A在操作B之前，那么在线程中A操作将在B操作之前执行。
+ 监视器锁规则。在监视器上的解锁操作必须在同一个监视器锁上的加锁操作之前执行。显式锁和内置锁在加锁和解锁等操作上有着相同的内存语义
+ volatile变量规则。对volatile变量的写入操作必须在对该变量的读操作之前执行。
+ 线程启动规则。在线程上对Thread.start的调用必须在该线程中执行任何操作之前执行。
+ 线程结束规则。线程中的任何操作都必须在其他线程检测到该线程已经结束之前执行，或者从Thread.join中成功返回，或者在调用Thread.isAlive时返回false
+ 中断规则。当一个线程在另一个线程上调用interrupt时，必须在被中断线程检测到interrupt调用之前执行（通过抛出InterruptedException，或者调用isInterrupted和interrupted）。
+ 终结器规则。对象的构造函数必须在启动该对象的终结器之前执行完成。
+ 传递性。如果操作A在操作B之前执行，并且操作B在操作C之前执行，那么操作A必须在操作C之前执行。

在类库中提供的其他Happens-Before排序包括：

+ 将一个元素放入一个线程安全容器的操作将在另一个线程从该容器中获取这个元素的操作之前执行
+ 在CountDownLatch上的倒数操作将在线程从闭锁上的await方法中返回之前执行
+ 释放Semaphore许可的操作将在从该Semaphore上获取一个许可之前执行。
+ Future表示的任务的所有操作将在从Future.get中返回之前执行。
+ 向Executor提交一个Runnable或Callable的操作将在任务开始执行之前执行
+ 一个线程到达CyclicBarrier或Exchanger的操作将在其他到达该栅栏或交换点的线程被释放之前执行。如果CyclicBarrier使用一个栅栏操作，那么到达栅栏的操作将在栅栏操作之前执行，而栅栏操作又会在线程从栅栏中释放之前执行。

延迟初始化占位。JVM将推迟ResourceHolder的初始化操作，知道开始使用这个类时才初始化，并且由于通过一个静态初始化来初始化Resource，因此不需要额外的同步。当任何一个线程第一次调用getResource时，都会使ResourceHolder被加载和被初始化，此时静态初始化器将执行Resource的初始化操作。

```java
public class ResourceFactory {
    private static class ResourceHolder {
        public static Resource resource = new Resource();
    }

    public static Resource getResource() {
        return ResourceHolder.resource;
    }
}
```

```java
public class DoubleCheckedLocking {
    private static volatile Resource resource;

    public static Resource getInstance() {
        if(resource == null) {
            synchronized(DoubleCheckedLocking.class) {
                if(resource == null) {
                    resource = new Resource();
                }
            }
        }
        return resource;
    }
}
```

DCL的真正问题在于：当在没有同步的情况下读取一个共享对象时，可能发生的最糟糕事情只是看到一个失效值（在这种情况下是一个空值），此时DCL方法将通过在持有锁的情况下再次尝试来避免这种风险。然而，实际情况远比这种情况糟糕——线程可能看到引用的当前值，但对象的状态值却是失效的，这意味着线程可以看到对象处于无效或错误的状态。在JMM的后续版本（Java5.0以及更高的版本）中，如果把resource声明为volatile类型，那么就能启用DCL，并且这种方式对性能的影响很小。
