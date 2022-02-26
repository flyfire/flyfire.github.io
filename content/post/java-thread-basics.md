---
title: "Java Thread Basics"
date: 2022-02-13T12:22:29+08:00
draft: false
---

进程：资源分配的最小单位
线程：CPU调度的最小单位

时间片轮转，线程上下文切换，并发

并发：任务交替执行，时间片轮转
并行：任务同时执行

Thread java对线程的抽象
Runnable 对任务的抽象。Callable 对任务/业务逻辑的抽象，有返回值。

stop可能导致线程占用的资源不能得到释放。

interrupt()方法设置线程的中断标志位。

isInterrupted()方法返回中断标志位，不清除中断标志位。保留为true

Thread.interrupted()方法返回中断标志位，清除中断标志位。由true改为false

不要使用自定义cancel来设置值来进行线程的中断，因为线程的sleep和wait这些方法不会去检测自定义的cancel字段，但是会去响应中断，会抛出InterruptedException。
sleep方法检测到中断调用interrupt()调用时，会抛出InterruptedException同时把中断标志位由true改为false，如果这时仍然需要中断线程，可以再次调用interrupt()方法。

死锁状态的线程是不会理会中断的。

Thread的start方法开始时会检查线程状态，不能多次调用。在start方法里面会真正启动一个系统线程。

线程状态：新建，就绪，运行，阻塞，死亡

<p><center><img src="/images/thread-state.jpg"/></center></p>

``Lock``底层实现是``LockSupport``，所以使用显式锁Lock，线程会进入等待或者等待超时状态。只有在``synchronized``方法或者方法块中线程才会进入阻塞状态。

``sleep``线程会进入等待/等待超时状态。

阻塞是被动进入，等待时主动进入。

死锁是指两个或两个以上的线程在执行过程中，由于竞争资源或者由于彼此通信而造成的一种阻塞的现象，若无外力作用，它们都将无法推进下去。此时称系统处于死锁状态或系统产生了死锁。

死锁的发生条件

+ 多个操作者(M个M >= 2)争夺N个资源(N>=2),N <= M
+ 争夺资源的顺序不对
+ 拿到资源不放手

打破``争夺资源的顺序不对``，规定必须先拿A锁再拿B锁。
打破``拿到资源不放手``,``Lock.tryLock``

```java
while(true) {
    if(lockA.tryLock()) {
        try {
            if(lockB.tryLock()) {
                try {
                    // do sth
                } finally {
                    lockB.unlock()
                }
            }
        } finally {
            lockA.unlock()
        }
    }
    Thread.sleep(random)
}
```

+ 互斥条件
+ 请求保持
+ 不剥夺
+ 环路等待

活锁

yield方法将线程从运行转到可运行状态，让出CPU执行权，不会释放锁。
threadA中调用threadB.join()等待threadB执行完之后继续执行操作。

join让线程执行变成串行。

setDeamon让线程变为守护线程。

线程优先级只有参考意义，不能决定谁先得到执行。

用户线程结束之后，所有的守护线程跟着结束。
守护线程run方法中的finally语句不一定会得到执行。
synchronized拿到对象锁才能继续执行。
synchronized在static方法上锁住的是class对象。
对象头中标志位
对Integer i加锁，i++之后i的地址发生了变化
Integer i = Integer.valueOf(this.intvalue + 1)

volatile保证变量的可见性，值被修改后能立即被其他线程可见，不保证原子性，适合一写多读的情况。

ThreadLocal 每个线程有自己的变量副本。

执行方法的时候在栈上执行
new对象的时候在堆上new
Object o = Object()
栈上的o指向堆上的对象，这称之为强引用，只要栈上面有一个引用指向这个对象，这个对象就不会被回收。
SoftReference 要发生OOM，内存不足的时候会被回收
WeakReference 发生内存回收的时候，GC的时候会被回收
虚引用 发生内存回收的时候，通知一下

ThreadLocalMap Entry是WeakReference<ThreadLocal>
ThreadLocalMap 有 Entry 数组
发生GC的时候ThreadLocal会被回收,ThreadLocal指向的value无法访问
ThreadLocal内存泄漏：发生GC的时候ThreadLocal会被回收，ThreadLocal指向的value无法访问，导致内存泄漏，用完ThreadLocal之后要调用threadLocal.remove()将value置为null进行内存回收。
get()/set()的时候会清理key为null的Entry。

共享引用，可能导致ThreadLocal的value指向同一个引用。

线程共享synchronized,volatile
线程协作wait/notify/notifyAll，在某个对象上等待通知

```
synchronized(对象) {
    while(条件不满足){
        对象.wait()
    }
    //业务逻辑
}

synchronized(对象) {
    // 业务逻辑，改变条件
    对象.notify()/对象.notifyAll();
}
```

线程调用wait方法，会进入休眠，进入休眠之前会释放锁。notify/notifyAll不会释放锁，只有synchronized方法块中的方法执行完之后才会释放锁。
被唤醒，while条件不满足，继续等待。
有两个条件满足都会notify，就会出现伪唤醒的情况。

```java
// 等待超时模式

class DBPool(private val initCap: Int) {
    companion object {
        @Volatile private var dbPoolInstance:DBPool? = null
        fun instance(initCap: Int): DBPool {
            if(dbPoolInstance == null) {
                synchronized(DBPool::class) {
                    if(dbPoolInstance == null) {
                        dbPoolInstance = DBPool(initCap);
                    }
                }
            }
            return dbPoolInstance;
        }
    }
    val list = mutableListOf<Connection>()
    init {
        for(i in 0..initCap) {
            list.add(Connection())
        }
    }

    fun release(connection: Connection) {
        synchronized(list) {
            list.add(connection)
            list.notifyAll()
        }
    }

    fun acquire(timeout: Long): Connection? {
        synchronized(list) {
            if(timeout < 0L) {
                while(list.isEmpty()) {
                    list.wait()
                }
                return list.removeFirst()
            } else {
                val future = System.currentTimeMillis() + timeout;
                var remain = timeout;
                while(list.isEmpty() && remain > 0L) {
                    list.wait(remain)
                    remain = future - System.currentTimeMills()
                }
                var connection: Connection? = null
                if(!list.isEmpty()) {
                    connection = list.removeFirst()
                }
                return connection
            }
        }
    }
}
```

[等待超时机制](https://github.com/flyfire/JavaWithKotlin/blob/master/src/main/kotlin/com/solarexsoft/coroutinelearning/Hi.kt)

``yield()``方法让出CPU使用权，不让出内存资源，不释放锁

``sleep()``方法不会释放锁

``wiat()``方法会释放锁，被唤醒了之后去竞争锁

``notify()``/``notifyAll()``不会释放锁，只有等同步代码块执行完了之后才会释放锁。

分而治之：大问题拆分成小问题，小问题之间无关联。
动态规划：大问题拆分成小问题，小问题之间有关联。

快速排序，归并排序，二分查找：分而治之。

ForkJoin

CountDownLatch 发令枪，闭锁

CAS Compare and Swap

synchronized悲观锁，认为总有人要改我的值，先锁住再说
CAS乐观锁，认为没人会改我的值。

synchronized竞争锁会导致上下文切换。

CAS自旋，死循环

CAS的问题
+ ABA问题，加一个版本戳解决，AtomicMarkableReference/AtomicStampedReference
+ 开销问题，自旋导致CPU很忙
+ 只能保证一个共享变量的原子操作。AtomicReference

AtomicMarkableReference 只关心有没有人改过
AtomicStampedReference 关心改了多少次

ThreadLocal 一个变量需要在线程内跨方法访问。

BlockingQueue

+ ``add``,``remove``方法不阻塞，但会抛出异常。
+ ``offer``往里添加元素，如果添加不进去，会返回false，``poll``从里面取元素，如果没有会返回null
+ ``put``,``take``阻塞操作

生产者、消费者性能不一致

+ ``ArrayBlockingQueue``,一个由数组结构组成的有界阻塞队列
+ ``LinkedBlockingQueue``，一个由链表结构组成的有界阻塞队列
+ ``PriorityBlockingQueue``,一个支持优先级排序的无界阻塞队列
+ ``DelayQueue``，一个使用优先级队列实现的无界阻塞队列
+ ``SynchronousQueue``，一个不存储元素的阻塞队列
+ ``LinkedTransferQueue``，一个由链表结构组成的无界阻塞队列
+ ``LinkedBlockingDeque``，一个由链表结构组成的双向阻塞队列

有界，规定了最大容量

无界，没有最大容量

无界，插入不会阻塞，拿会阻塞。

单机的缓存系统，``DelayQueue``

``SynchronousQueue``数据的直接传递，生产者往里添加元素，必须有消费者把元素拿走

``LinkedTransferQueue``，try尝试传输。生产者在往队列放元素的时候，如果正好有消费者在等待拿元素，直接把元素给消费者，不往里面放。

``ThreadPoolExecutor`` 

先启动corePoolSize的线程处理任务，corePoolSize的线程全都启动了，再来了任务就放入BlockingQueue里，BlockingQueue里放满了，继续来任务的话启动非核心线程，达到maxPoolSize之后，拒绝策略起作用。

``RejectedExecutionHandler``

+ ``AbortPolicy`` 抛出异常
+ ``CallerRunsPolicy`` 调用线程运行任务
+ ``DiscardPolicy``,``DiscardOldestPolicy`` 丢弃提交的任务

+ ``shutdown``中断未执行的任务
+ ``shutdownNow``不管是正在运行的还是未运行的，都发出中断信号

CPU密集型 最大线程数 CPU核心数+1  虚拟内存调度到内存，页缺失 避免线程上下文切换

IO密集型  最大线程数 CPU核心数*2

混合型 

IO操作 DMA CPU向磁盘控制器提交任务，磁盘控制器完成任务后发起CPU中断。

零拷贝 应用程序向内核空间申请空间，mmap，免去应用空间和内核空间来回拷贝 mmap direct memory

AQS 模板方法

+ ``tryAcquire`` 独占

+ ``tryAcquireShared`` 共享

Java Memory Model

CPU L1Cache L2Cache L3CPU共享Cache

Java线程 工作内存线程独享 主内存

变量 可见性，原子性

volatile只能保证可见性，不能保证原子性。适合一个线程写，其他线程读的情况。

synchronized既能保证可见性又能保证原子性。

指令流水线和重排序，CPU可以同时执行多条指令。

重排序缓存

volatile 变量抑制指令重排序。

volatile+CAS操作替换synchronized

有volatile修饰的共享变量进行写操作的时候会使用CPU提供的Lock前缀指令

+ 将当前处理器缓存行的数据写回到系统内存

+ 这个写回内存的操作会使在其他CPU缓存里面缓存了该内存地址的数据无效//CPU缓存行失效