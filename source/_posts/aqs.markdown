---
layout: post
title: "Java AQS解析"
date: 2019-07-28 16:59
comments: true
categories: 
- java
- concurrency
tags:
- java
---

``AbstractQueuedSynchronizer``是很多并发工具类如``ReentrantLock``的实现基础，本文对其进行分析。

<!-- more -->

TL;DR

以下是对[aqs.pdf](http://gee.cs.oswego.edu/dl/papers/aqs.pdf)部分内容零散的翻译，其实《Java并发编程实战》中有一章介绍了AQS，我也做了笔记，可以在[这里](https://solarex.github.io/reading-notes/jcip/ch14.html)看到，对于AQS模板方法的使用，可以在[这篇笔记](https://solarex.github.io/reading-notes/the-art-of-java-concurrency-programming/ch05.html)中看到。翻译很不专业，将就看下吧。如果想深入了解AQS，可以读下[aqs.pdf](http://gee.cs.oswego.edu/dl/papers/aqs.pdf)或者看下本文reference部分的几篇文章。

Synchronizers possess two kinds of methods : at least one ``acquire`` operation that blocks the calling thread unless/until the synchronization state allows it to proceed, and at least one ``release`` operation that changes synchronization state in a way that may allow one or more blocked threads to unblock.

同步器提供两种方法：一个``acquire``方法在同步状态不允许线程通过运行时阻塞线程，一个``release``方法改变同步状态来允许一个或多个被阻塞的线程继续运行。

The ``java.util.concurrent`` package does not define a single unified API for synchronizers. Some are defined via common interfaces (e.g., ``Lock``), but others contain only specialized versions. So, ``acquire`` and ``release`` operations take a range of names and forms across different classes. For example, methods ``Lock.lock``,``Semaphore.acquire``, ``CountDownLatch.await``, and ``FutureTask.get`` all map to ``acquire`` operations in the framework. However, the package does maintain consistent conventions across classes to support a range of common usage options. When meaningful, each synchronizer supports:
+ Nonblocking synchronization attempts (for example,``tryLock``) as well as blocking versions.
+ Optional timeouts, so applications can give up waiting.
+ Cancellability via interruption, usually separated into one version of acquire that is cancellable, and one that isn't.

JUC没有为同步器定义一个统一的API。有一些是在通用的接口中定义的，比如``Lock``，但是其他的包括一些特殊的版本。所以不同同步器的``acquire``和``release``方法在名字和形态上表现不同。比如，``Lock.lock``，``Semaphore.acquire``，``CountDownLatch.await``和``FutureTask.get``和AQS框架中的``acquire``方法相对应。但是JUC框架在通用操作上保持了一致性。对于同步器来说，一般都支持以下操作：

+ 非阻塞尝试和阻塞尝试，比如``Lock.tryLock``和``Lock.lock``
+ 可选的超时，超时后线程可以放弃等待尝试
+ 在等待获取尝试的时候，对线程中断的响应或不响应

Synchronizers may vary according to whether they manage only ``exclusive`` states – those in which only one thread at a time may continue past a possible blocking point – versus possible ``shared`` states in which multiple threads can at least sometimes proceed.Regular lock classes of course maintain only exclusive state, but counting semaphores, for example, may be acquired by as many threads as the count permits. To be widely useful, the framework must support both modes of operation.

同步器可能会因为共享状态是独占还是可共享的而不同。一般锁维护的是独占状态，同时只能有一个线程持有锁，但是Semaphore可能同时被多个线程获取。AQS对这两种模式都提供了支持。

The ``java.util.concurrent`` package also defines interface ``Condition``, supporting monitor-style ``await/signal`` operations that may be associated with ``exclusive Lock`` classes, and whose implementations are intrinsically intertwined with their associated Lock classes.

JUC框架还定义了``Condition``接口，来提供和内置锁``wait/notify``类似的操作，``Condition``的``await/signal``方法和独占锁绑定在一起。

The basic ideas behind a synchronizer are quite straightforward.
An acquire operation proceeds as:

```java
while (synchronization state does not allow acquire) {
    enqueue current thread if not already queued;
    possibly block current thread;
}
dequeue current thread if it was queued;
```

And a release operation is:

```java
update synchronization state;
if (state may permit a blocked thread to acquire)
unblock one or more queued threads;
```

Support for these operations requires the coordination of three basic components:
+ Atomically managing synchronization state
+ Blocking and unblocking threads
+ Maintaining queues

同步器背后的思想很简单，三个基本的部分构成了对同步器的支持：

+ 原子更新同步状态
+ 线程的阻塞和唤醒
+ 线程等待队列

Class ``AbstractQueuedSynchronizer`` maintains synchronization state using only a single (32bit) int, and exports ``getState``, ``setState``, and ``compareAndSetState`` operations to access and update this state. These methods in turn rely on ``java.util.concurrent.atomic`` support providing JSR133 (Java Memory Model) compliant ``volatile`` semantics on ``reads and writes``, and access to native ``compare-and-swap`` or ``loadlinked/store-conditional`` instructions to implement ``compareAndSetState``, that atomically sets state to a given new value only if it holds a given expected value.

``AbstractQueuedSynchronizer``用一个32位的int值来表示同步状态，暴露了``getState``, ``setState``和``compareAndSetState`` 方法来获取和更新同步状态。这些方法依赖JMM底层模型对``volatile``变量的语义支持，依赖CPU的``compare-and-swap`` 或 ``loadlinked/store-conditional`` 指令来完成``compareAndSetState``操作。CAS操作原子地设置同步状态为新的值，只有同步状态等于期待的原值时才会成功。

Restricting synchronization state to a 32bit ``int`` was a pragmatic decision. While JSR166 also provides atomic operations on 64bit ``long`` fields, these must still be emulated using internal locks on enough platforms that the resulting synchronizers would not perform well. In the future, it seems likely that a second base class, specialized for use with 64bit state (i.e., with long control arguments), will be added. However, there is not now a compelling reason to include it in the package. Currently, 32 bits suffice for most applications. Only one ``java.util.concurrent`` synchronizer class, ``CyclicBarrier``, would require more bits to maintain state, so instead uses locks (as do most higher-level utilities in the package).

JUC已经提供了``AbstractQueuedLongSynchronizer``，``AbstractQueuedLongSynchronizer``用``long``来表示同步状态。

Concrete classes based on ``AbstractQueuedSynchronizer`` must define methods ``tryAcquire`` and ``tryRelease`` in terms of these exported state methods in order to control the ``acquire`` and ``release`` operations. The ``tryAcquire`` method must return ``true`` if synchronization was acquired, and the ``tryRelease`` method must return ``true`` if the new synchronization state may allow future acquires. These methods accept a single ``int`` argument that can be used to communicate desired state; for example in a reentrant lock, to re-establish the recursion count when re-acquiring the lock after returning from a condition wait.Many synchronizers do not need such an argument, and so just ignore it.

AQS的具体实现类必须实现``tryAcquire`` 和 ``tryRelease``以便获得AQS提供的 ``acquire`` 和 ``release`` 操作。如果同步状态允许线程通过，``tryAcquire``方法必须返回``true``，``tryRelease``必须返回``true``如果新的同步状态允许阻塞的线程通过。``tryAcquire``和``tryRelease``接受一个``int``类型的参数，很多同步器不需要这个参数，直接忽略了它。

Until JSR166, there was no Java API available to block and unblock threads for purposes of creating synchronizers that are not based on built-in monitors. The only candidates were ``Thread.suspend`` and ``Thread.resume``, which are unusable because they encounter an unsolvable race problem: If an unblocking thread invokes ``resume`` before the blocking thread has executed ``suspend``, the ``resume`` operation will have no effect.

在JSR166之前，没有Java API来提供除了基于内置锁之外的线程阻塞唤醒同步操作。可选的有``Thread.suspend``和``Thread.resume``，但是如果一个线程在``suspend``之前先调用了``resume``，``resume``将无效。

The ``java.util.concurrent.locks`` package includes a ``LockSupport`` class with methods that address this problem. Method ``LockSupport.park`` blocks the current thread unless or until a ``LockSupport.unpark`` has been issued. (Spurious wakeups are also permitted.) Calls to ``unpark`` are not "counted", so multiple unparks before a park only unblock a single park.Additionally, this applies per-thread, not per-synchronizer. A thread invoking park on a new synchronizer might return immediately because of a "leftover" unpark from a previous usage. However, in the absence of an unpark, its next invocation will block. While it would be possible to explicitly clear this state, it is not worth doing so. It is more efficient to invoke park multiple times when it happens to be necessary.This simple mechanism is similar to those used, at some level, in the Solaris-9 thread library, in WIN32 "consumable events",and in the Linux NPTL thread library, and so maps efficiently to each of these on the most common platforms Java runs on.(However, the current Sun Hotspot JVM reference implementation on Solaris and Linux actually uses a pthread condvar in order to fit into the existing runtime design.) The park method also supports optional relative and absolute timeouts, and is integrated with JVM ``Thread.interrupt`` support — interrupting a thread unparks it.

JUC提供了``LockSupport``来解决这个问题。``LockSupport``还支持可选的超时，而且提供了对线程中断的支持。

The heart of the framework is maintenance of queues of blocked threads, which are restricted here to FIFO queues. Thus, the framework does not support priority-based synchronization.These days, there is little controversy that the most appropriate choices for synchronization queues are non-blocking data structures that do not themselves need to be constructed using lower-level locks. And of these, there are two main candidates: variants of Mellor-Crummey and Scott (MCS) locks, and variants of Craig, Landin, and Hagersten (CLH) locks.Historically, CLH locks have been used only in spinlocks.However, they appeared more amenable than MCS for use in the synchronizer framework because they are more easily adapted to handle cancellation and timeouts, so were chosen as a basis. The resulting design is far enough removed from the original CLH structure to require explanation.A CLH queue is not very queue-like, because its enqueuing and dequeuing operations are intimately tied to its uses as a lock. It is a linked queue accessed via two atomically updatable fields,head and tail, both initially pointing to a dummy node.

AQS的等待队列是CLH队列的变种。

A new node, node, is enqueued using an atomic operation:

```java
do { pred = tail;
} while(!tail.compareAndSet(pred, node));
```

The ``release`` status for each node is kept in its predecessor node.So, the "spin" of a spinlock looks like:

```java
while (pred.status != RELEASED) ; // spin
```

A dequeue operation after this spin simply entails setting the head field to the node that just got the lock:

```java
head = node;
```

新节点入队通过CAS设置``tail``来实现，当前节点的释放状态取决于前驱节点的状态。自旋结束后出队一个节点会把``head``设置为这个节点。

AQS的等待队列使用``next``指针保存了后继节点，用``status``保存了节点代表的线程的状态，比如可能线程已经取消了等待。

Omitting such details, the general form of the resulting implementation of the basic acquire operation (exclusive,noninterruptible, untimed case only) is:

```java
if (!tryAcquire(arg)) {
    node = create and enqueue new node;
    pred = node's effective predecessor;
    while (pred is not head node || !tryAcquire(arg)) {
        if (pred's signal bit is set)
            park();
        else
            compareAndSet pred's signal bit to true;
        pred = node's effective predecessor;
    }
    head = node;
}
```

And the release operation is:

```java
if (tryRelease(arg) && head node's signal bit is set) {
    compareAndSet head's signal bit to false;
    unpark head's successor, if one exists
}
```

The synchronizer framework provides a ``ConditionObject`` class for use by synchronizers that maintain exclusive synchronization and conform to the Lock interface. Any number of condition objects may be attached to a lock object, providing classic monitor-style ``await``, ``signal``, and ``signalAll`` operations, including those with timeouts, along with some inspection and monitoring methods.

AQS框架还提供了``ConditionObject``来提供和内置锁``wait/notify/notifyAll``类似的``await/signal/signalAll``功能。

The ``ConditionObject`` class enables conditions to be efficiently integrated with other synchronization operations,again by fixing some design decisions. This class supports only Java-style monitor access rules in which condition operations are legal only when the lock owning the condition is held by the current thread. Thus, a ``ConditionObject`` attached to a ReentrantLock acts in the same way as do built-in monitors (via ``Object.wait`` etc), differing only in method names, extra functionality, and the fact that users can declare multiple conditions per lock.A ``ConditionObject`` uses the same internal queue nodes as synchronizers, but maintains them on a separate condition queue.The signal operation is implemented as a queue transfer from the condition queue to the lock queue, without necessarily waking up the signalled thread before it has re-acquired its lock.

The basic await operation is:

```java
 create and add new node to condition queue;
 release lock;
 block until node is on lock queue;
 re-acquire lock;
```

And the signal operation is:

```java
 transfer the first node from condition queue to lock queue;
```

``ConditionObject``提供和内置锁类似的功能，在一个``Lock``上可以声明多个``ConditionObject``。``ConditionObject``使用condition queue来维护等待队列，condition queue上的节点和lock queue中的节点相同，``await``是入condition queue，``signal``是从condition queue中转移到lock queue中。

Because these operations are performed only when the lock is held, they can use sequential linked queue operations (using a ``nextWaiter`` field in nodes) to maintain the condition queue.The transfer operation simply unlinks the first node from the condition queue, and then uses CLH insertion to attach it to the lock queue. The main complication in implementing these operations is dealing with cancellation of condition waits due to timeouts or ``Thread.interrupt``. A cancellation and signal occuring at approximately the same time encounter a race whose outcome conforms to the specifications for built-in monitors. As revised in JSR133, these require that if an interrupt occurs before a signal, then the ``await`` method must, after re-acquiring the lock, throw ``InterruptedException``. But if it is interrupted after a signal, then the method must return without throwing an exception, but with its thread interrupt status set.

transfer操作是从将condition queue里的第一个节点从condition queue中移除，插入到lock queue中去。如果在``signal``之前发生了线程中断，那么``await``必须在重新获取锁之后抛出``InterruptedException``。如果在``signal``之后发生了中断，那么线程中断状态将被设置，但是不会抛出异常。

Here are sketches of how java.util.concurrent synchronizer classes are defined using this framework:

+ The ``ReentrantLock`` class uses synchronization state to hold the (recursive) lock count. When a lock is acquired, it also records the identity of the current thread to check recursions and detect illegal state exceptions when the wrong thread tries to unlock. The class also uses the provided ``ConditionObject``, and exports other monitoring and inspection methods. The class supports an optional "fair" mode by internally declaring two different ``AbstractQueuedSynchronizer`` subclasses (the fair one disabling barging) and setting each ``ReentrantLock`` instance to use the appropriate one upon construction.
+ The ``ReentrantReadWriteLock`` class uses 16 bits of the synchronization state to hold the write lock count, and the remaining 16 bits to hold the read lock count. The ``WriteLock`` is otherwise structured in the same way as ``ReentrantLock``.The ``ReadLock`` uses the ``acquireShared`` methods to enable multiple readers.
+ The ``Semaphore`` class (a counting semaphore) uses the synchronization state to hold the current count. It defines ``acquireShared`` to decrement the count or block if nonpositive, and ``tryRelease`` to increment the count, possibly unblocking threads if it is now positive.
+ The ``CountDownLatch`` class uses the synchronization state to represent the count. All acquires pass when it reaches zero.
+ The ``FutureTask`` class uses the synchronization state to represent the run-state of a future (initial, running, cancelled,done). Setting or cancelling a future invokes ``release``,unblocking threads waiting for its computed value via ``acquire``.
+ The ``SynchronousQueue`` class (a CSP-style handoff) uses internal wait-nodes that match up producers and consumers. It uses the synchronization state to allow a producer to proceed when a consumer takes the item, and vice-versa.

JUC框架中几个使用AQS的同步器类的内部实现。

### reference

+ [一行一行源码分析清楚 AbstractQueuedSynchronizer](https://javadoop.com/post/AbstractQueuedSynchronizer)
+ [一行一行源码分析清楚 AbstractQueuedSynchronizer（二）](https://javadoop.com/post/AbstractQueuedSynchronizer-2)
+ [一行一行源码分析清楚 AbstractQueuedSynchronizer（三）](https://javadoop.com/post/AbstractQueuedSynchronizer-3)
+ [AbstractQueuedSynchronizer的介绍和原理分析](http://ifeve.com/introduce-abstractqueuedsynchronizer/)
+ [深度解析 Java 8：JDK1.8 AbstractQueuedSynchronizer 的实现分析（上）](https://www.infoq.cn/article/jdk1.8-abstractqueuedsynchronizer)
+ [深度解析 Java 8：AbstractQueuedSynchronizer 的实现分析（下）](https://www.infoq.cn/article/java8-abstractqueuedsynchronizer)
+ [深入理解AbstractQueuedSynchronizer(AQS)](https://juejin.im/post/5aeb07ab6fb9a07ac36350c8)
+ [AbstractQueuedSynchronizer使用和源码分析](https://liuzhengyang.github.io/2017/05/12/aqs/)
+ [AbstractQueuedSynchronizer 源码分析 (基于Java 8)](https://www.jianshu.com/p/e7659436538b)
+ [源码分析JDK8之AbstractQueuedSynchronizer](https://segmentfault.com/a/1190000014221325)
+ [aqs.pdf](http://gee.cs.oswego.edu/dl/papers/aqs.pdf)
+ [CLH.pdf](http://www.cs.tau.ac.il/~shanir/nir-pubs-web/Papers/CLH.pdf)
+ [自旋锁、排队自旋锁、MCS锁、CLH锁](https://coderbee.net/index.php/concurrent/20131115/577)
