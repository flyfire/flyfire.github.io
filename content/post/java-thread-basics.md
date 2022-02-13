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