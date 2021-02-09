+++
title = "Java Thread Tutorial"
date = "2014-10-14"
slug = "2014/10/14/java-thread-tutorial"
Categories = ["dev", "java"]
+++
<center><p><img src="/images/java-thread-tutorial.png" ></p></center>

*	[Java Thread and Multithreading Tutorial](#overview)
	* [Java Thread Example](#example)
	* [Java Thread Sleep](#sleep)
	* [Java Thread Join](#join)
	* [Java Thread States](#states)
	* [Java Thread wait, notify and notifyAll](#wait)
	* [Java Thread Safety and Java Synchronization](#safety)
	* [Java Exception in thread main](#exception)
	* [Thread Safety in Singleton Class](#singleton)
	* [Java Daemon Thread](#daemon)
	* [Java Thread Local](#threadlocal)
	* [Java Thread Dump](#dump)
	* [How to Analyze Deadlock and avoid it in Java](#deadlock)
	* [Java Timer Thread](#timer)
	* [Java Producer Consumer Problem](#producer)
	* [Java Thread Pool](#pool)
	* [Java Callable Future](#future)
	* [Java FutureTask Example](#futuretask)
	
<h2 id="overview">Java Thread and Multithreading Tutorial</h2>
There are two types of threads in an application – ``user thread`` and ``daemon thread``. When we start an application, main is the first user thread created and we can create multiple user threads as well as daemon threads. When all the user threads are executed, JVM terminates the program.

We can set different priorities to different Threads but it doesn’t guarantee that higher priority thread will execute first than lower priority thread. Thread scheduler is the part of Operating System implementation and when a Thread is started, it’s execution is controlled by Thread Scheduler and JVM doesn’t have any control on it’s execution.

<!-- more -->

<h3 id="example">Java Thread Example</h3>
Every java application has at least one thread – main thread. Although there are so many other threads running in background like memory management, system management, signal processing etc. But from application point of view – main is the first thread and we can create multiple threads from it.

Multithreading refers to two or more threads executing concurrently in a single program. A computer single core processor can execute only one thread at a time and time slicing is the OS feature to share processor time between different processes and threads.

Benefits of Threads

+ Threads are lightweight compared to processes, it takes less time and resource to create a thread.
+ Threads share their parent process data and code
+ Context switching between threads is usually less expensive than between processes.
+ Thread intercommunication is relatively easy than process communication.

Java provides two ways to create a thread programmatically.

+ Implementing the ``java.lang.Runnable`` interface.
+ Extending the ``java.lang.Thread`` class.

**Once we start any thread, it’s execution depends on the OS implementation of time slicing and we can’t control their execution. However we can set threads priority but even then it doesn’t guarantee that higher priority thread will be executed first.**

As you have noticed that thread doesn’t return any value but what if we want our thread to do some processing and then return the result to our client program, check our [Java Callable Future](#future).

<h3 id="sleep">Java Thread Sleep</h3>
``java.lang.Thread sleep()`` method can be used to pause the execution of current thread for specified time in milliseconds. The argument value for milliseconds can’t be negative, else it throws ``IllegalArgumentException``.

There is another method ``sleep(long millis, int nanos)`` that can be used to pause the execution of current thread for specified milliseconds and nanoseconds. The allowed nano second value is between 0 and 999999.

Thread Sleep important points

+ It always pause the current thread execution.
+ The actual time thread sleeps before waking up and start execution depends on system timers and schedulers. For a quiet system, the actual time for sleep is near to the specified sleep time but for a busy system it will be little bit more.
+ Thread sleep doesn’t lose any monitors or locks current thread has acquired.
+ Any other thread can interrupt the current thread in sleep, in that case ``InterruptedException`` is thrown.

How Thread Sleep Works

``Thread.sleep()`` interacts with the thread scheduler to put the current thread in wait state for specified period of time. Once the wait time is over, thread state is changed to runnable state and wait for the CPU for further execution. So the actual time that current thread sleep depends on the thread scheduler that is part of operating system.

<h3 id="join">Java Thread Join</h3>

Java Thread ``join`` method can be used to pause the current thread execution until unless the specified thread is dead. There are three overloaded join functions.

+ ``public final void join()``: This method puts the current thread on wait until the thread on which it’s called is dead. If the thread is interrupted, it throws ``InterruptedException``.

+ ``public final synchronized void join(long millis)``: This method is used to wait for the thread on which it’s called to be dead or wait for specified milliseconds. Since thread execution depends on OS implementation, it doesn’t guarantee that the current thread will wait only for given time.

+ ``public final synchronized void join(long millis, int nanos)``: This method is used to wait for thread to die for given milliseconds plus nanoseconds.

```java
package org.solarex.threadtest;
 
public class ThreadJoinExample {
 
    public static void main(String[] args) {
        Thread t1 = new Thread(new MyRunnable(), "t1");
        Thread t2 = new Thread(new MyRunnable(), "t2");
        Thread t3 = new Thread(new MyRunnable(), "t3");
         
        t1.start();
         
        //start second thread after waiting for 2 seconds or if it's dead
        try {
            t1.join(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
         
        t2.start();
         
        //start third thread only when first thread is dead
        try {
            t1.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
         
        t3.start();
         
        //let all threads finish execution before finishing main thread
        try {
            t1.join();
            t2.join();
            t3.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
         
        System.out.println("All threads are dead, exiting main thread");
    }
 
}
 
class MyRunnable implements Runnable{
 
    @Override
    public void run() {
        System.out.println("Thread started:::"+Thread.currentThread().getName()+"@"+System.currentTimeMillis());
        try {
            Thread.sleep(4000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("Thread ended:::"+Thread.currentThread().getName()+"@"+System.currentTimeMillis());
    }
     
}

------------------------------------
// output begin
hrh@Solarex:Java$ java org.solarex.threadtest.ThreadJoinExample 
Thread started:::t1@1413344304212
Thread started:::t2@1413344306213
Thread ended:::t1@1413344308213
Thread started:::t3@1413344308214
Thread ended:::t2@1413344310213
Thread ended:::t3@1413344312214
All threads are dead, exiting main thread
// output end
```
<h3 id="states">Java Thread States</h3>
Thread States

Below diagram shows different states of thread in java, note that we can create a thread in java and start it but how the thread states change from Runnable to Running to Blocked depends on the OS implementation of thread scheduler and java doesn’t have full control on that.

<center><p><img src="/images/thread-lifecycle-states.png"></p></center>

+ ``New``:When we create a new Thread object using new operator, thread state is New Thread. At this point, thread is not alive and it’s a state internal to Java programming.

+ ``Runnable``:When we call ``start()`` function on ``Thread`` object, it’s state is changed to ``Runnable`` and the control is given to **Thread scheduler** to finish it’s execution. Whether to run this thread instantly or keep it in runnable thread pool before running it depends on the OS implementation of thread scheduler.

+ ``Running``:When thread is executing, it’s state is changed to ``Running``. Thread scheduler picks one of the thread from the runnable thread pool and change it’s state to Running and CPU starts executing this thread. A thread can change state to Runnable, Dead or Blocked from running state depends on time slicing, thread completion of run() method or waiting for some resources.

+ ``Blocked/Waiting``:A thread can be waiting for other thread to finish using thread ``join`` or it can be waiting for some resources to available, for example producer consumer problem or waiter notifier implementation or IO resources, then it’s state is changed to Waiting. Once the thread wait state is over, it’s state is changed to Runnable and it’s moved back to runnable thread pool.

+ ``Dead``:Once the thread finished executing, it’s state is changed to Dead and it’s considered to be not alive.

<h3 id="wait">Java Thread wait,notifyand notifyAll</h3>
+ **wait**:Object ``wait`` methods has three variance, one which waits indefinitely for any other thread to call notify or notifyAll method on the object to wake up the current thread. Other two variances puts the current thread in wait for specific amount of time before they wake up.

+ **notify**:``notify`` method wakes up only one thread waiting on the object and that thread starts execution. So if there are multiple threads waiting for an object, this method will wake up only one of them. The choice of the thread to wake depends on the OS implementation of thread management.

+ **notifyAll**:``notifyAll`` method wakes up all the threads waiting on the object, although which one will process first depends on the OS implementation.

```java
//Message.java
package org.solarex.threadtest;

public class Message{
    private String msg;

    public Message(String str){
        this.msg = str;
    }

    public String getMsg(){
        return msg;
    }

    public void setMsg(String str){
        this.msg = str;
    }
}
//Waiter.java
package org.solarex.threadtest;

public class Waiter implements Runnable{
    private Message msg;
    public Waiter(Message m){
        this.msg = m;
    }
    @Override
    public void run(){
        String name = Thread.currentThread().getName();
        synchronized(msg){
            try{
                System.out.println(name+" waiting to get notified @ " + System.currentTimeMillis());
                msg.wait();
            }catch(InterruptedException e){
                e.printStackTrace();
            }
            System.out.println(name+" waiter thread got notified @ " + System.currentTimeMillis());
            System.out.println(name+" proccessed: " + msg.getMsg());
        }
    }
}
//Notifier.java
package org.solarex.threadtest;
public class Notifier implements Runnable{
    private Message msg;
    public Notifier(Message msg){
        this.msg = msg;
    }

    @Override
    public void run(){
        String name = Thread.currentThread().getName();
        System.out.println(name+" started");
        try{
            Thread.sleep(1000);
            synchronized(msg){
                msg.setMsg(name+" notifier work done");
                // msg.notify();
                msg.notifyAll();
            }
        }catch (InterruptedException e){
            e.printStackTrace();
        }
    }
}
//WaitNotifierTest.java
package org.solarex.threadtest;
public class WaitNotifyTest {
    public static void main(String[] args){
        Message msg = new Message("Hi");
        Waiter waiter0 = new Waiter(msg);
        new Thread(waiter0, "waiter0").start();

        Waiter waiter1 = new Waiter(msg);
        new Thread(waiter1, "waiter1").start();

        Notifier notifier = new Notifier(msg);
        new Thread(notifier, "notifier").start();

        System.out.println("main thread exit");
    }
}

// javac -d . Message.java Waiter.java Notifier.java WaitNotifierTest.java
// java org.solarex.threadtest.WaitNotifierTest
// --------begin output------------
hrh@Solarex:Java$ java org.solarex.threadtest.WaitNotifyTest 
waiter0 waiting to get notified @ 1413423213359
waiter1 waiting to get notified @ 1413423213360
main thread exit
notifier started
waiter1 waiter thread got notified @ 1413423214362
waiter1 proccessed: notifier notifier work done
waiter0 waiter thread got notified @ 1413423214362
waiter0 proccessed: notifier notifier work done
// --------end output----------
```

<h3 id="safety">Java Thread Safety and Java Synchronization</h3>
Thread safety is the process to make our program safe to use in multithreaded environment, there are different ways through which we can make our program thread safe.

+ Synchronization is the easiest and most widely used tool for thread safety in java.
+ Use of Atomic Wrapper classes from ``java.util.concurrent.atomic`` package. For example ``AtomicInteger``
+ Use of locks from ``java.util.concurrent.locks`` package.
+ Using thread safe collection classes, check this post for usage of ``ConcurrentHashMap`` for thread safety.
+ Using volatile keyword with variables to make every thread read the data from memory, not read from thread cache.

Synchronization is the tool using which we can achieve thread safety, JVM guarantees that synchronized code will be executed by only one thread at a time. java keyword synchronized is used to create synchronized code and internally it uses locks on Object or Class to make sure only one thread is executing the synchronized code.

+ Java synchronization works on locking and unlocking of resource, before any thread enters into synchronized code, it has to acquire lock on the Object and when code execution ends, it unlocks the resource that can be locked by other threads. In the mean time other threads are in wait state to lock the synchronized resource.
+ We can use synchronized keyword in two ways, one is to make a complete method synchronized and other way is to create synchronized block.可以创建synchronized方法或者synchronized代码块
+ When a method is synchronized, it locks the Object, if method is static it locks the Class, so it’s always best practice to use synchronized block to lock the only sections of method that needs synchronization.
+ While creating synchronized block, we need to provide the resource on which lock will be acquired, it can be XYZ.class or any Object field of the class.
+ ``synchronized(this)`` will lock the Object before entering into the synchronized block.
+ You should use the lowest level of locking, for example if there are multiple synchronized block in a class and one of them is locking the Object, then other synchronized blocks will also be not available for execution by other threads. When we lock an Object, it acquires lock on all the fields of the Object.
+ Java Synchronization provides data integrity on the cost of performance, so it should be used only when it’s absolutely necessary.
+ Java Synchronization works only in the same JVM, so if you need to lock some resource in multiple JVM environment, it will not work and you might have to look after some global locking mechanism.
+ Java Synchronization could result in deadlocks, check this post about [deadlock in java and how to avoid them](#deadlock).
+ Java synchronized keyword cannot be used for constructors and variables.
+ It is preferable to create a dummy private Object to use for synchronized block, so that it’s reference can’t be changed by any other code. For example if you have a setter method for Object on which you are synchronizing, it’s reference can be changed by some other code leads to parallel execution of the synchronized block.
+ We should not use any object that is maintained in a constant pool, for example String should not be used for synchronization because if any other code is also locking on same String, it will try to acquire lock on the same reference object from String pool and even though both the codes are unrelated, they will lock each other.

```java
//dummy object variable for synchronization
private Object mutex=new Object();
...
//using synchronized block to read, increment and update count value synchronously
synchronized (mutex) {
        count++;
}
```

```java
public class MyObject {
  
  // Locks on the object's monitor
  public synchronized void doSomething() { 
    // ...
  }
}
  
// Hackers code
MyObject myObject = new MyObject();
synchronized (myObject) {
  while (true) {
    // Indefinitely delay myObject
    Thread.sleep(Integer.MAX_VALUE); 
  }
}
```

Notice that hacker’s code is trying to lock the myObject instance and once it gets the lock, it’s never releasing it causing ``doSomething()`` method to block on waiting for the lock, this will cause system to go on deadlock and cause Denial of Service (DoS).

```java
public class MyObject {
  public Object lock = new Object();
  
  public void doSomething() {
    synchronized (lock) {
      // ...
    }
  }
}
 
//untrusted code
 
MyObject myObject = new MyObject();
//change the lock Object reference
myObject.lock = new Object();
```

Notice that lock Object is public and by changing it’s reference, we can execute synchronized block parallel in multiple threads. Similar case is true if you have private Object but have setter method to change it’s reference.

```java
public class MyObject {
  //locks on the class object's monitor
  public static synchronized void doSomething() { 
    // ...
  }
}
  
// hackers code
synchronized (MyObject.class) {
  while (true) {
    Thread.sleep(Integer.MAX_VALUE); // Indefinitely delay MyObject
  }
}
```

Notice that hacker code is getting lock on class monitor and not releasing it, it will cause deadlock and DoS in the system.


<h3 id="exception">Java Exception in thread main</h3>

These are some of the common java exceptions in thread main, whenever you face any one of these check following:

+ Same JRE version is used to compile and run the java program
+ You are running java class from the classes directory and package is provided as directory.
+ Your java classpath is set properly to include all the dependency classes
+ You are using only file name without .class extension while running a java program
+ Java class main method syntax is correct
<h3 id="singleton">Thread Safety in Singleton Class</h3>
**Singleton** is one of the most widely used creational design pattern to restrict the object creation by applications. In real world applications, resources like Database connections or Enterprise Information Systems (EIS) are limited and should be used wisely to avoid any resource crunch. To achieve this, we can implement Singleton design pattern to create a wrapper class around the resource and limit the number of object created at runtime to one.

In general we follow below steps to create a singleton class:

+ Override the private constructor to avoid any new object creation with new operator.
+ Declare a private static instance of the same class
+ Provide a public static method that will return the singleton class instance variable. If the variable is not initialized then initialize it or else simply return the instance variable.

1. Create the instance variable at the time of class loading:
+ **Pros**:Thread safety without synchronization,Easy to implement
+ **Cons**:Early creation of resource that might not be used in the application,The client application can’t pass any argument, so we can’t reuse it. For example, having a generic singleton class for database connection where client application supplies database server properties.

2. Synchronize the ``getInstance()`` method:
+ **Pros**:Thread safety is guaranteed,Client application can pass parameters,Lazy initialization achieved
+ **Cons**:Slow performance because of locking overhead,Unnecessary synchronization that is not required once the instance variable is initialized.

3. Use synchronized block inside the if loop:
+ **Pros**:Thread safety is guaranteed,Client application can pass arguments,Lazy initialization achieved,Synchronization overhead is minimal and applicable only for first few threads when the variable is null.
+ **Cons**:Extra if condition

```java
public class ASingleton{
 
    private static ASingleton instance= null;
    private static Object mutex= new Object();
    private ASingleton(){
    }
 
    public static ASingleton getInstance(){
        if(instance==null){
            synchronized (mutex){
                if(instance==null) instance= new ASingleton();
            }
        }
        return instance;
    }
}
```
<h3 id="daemon">Java Daemon Thread</h3>
When we create a Thread in java, by default it’s a user thread and if it’s running JVM will not terminate the program. When a thread is marked as daemon thread, JVM doesn’t wait it to finish and as soon as all the user threads are finished, it terminates the program as well as all the associated daemon threads

```java
package org.solarex.threadtest;
public class JavaDaemonThread{
    public static void main(String[] args) throws InterruptedException{
        Thread dt = new Thread(new DaemonThread(), "dt");
        dt.setDaemon(true);
        dt.start();
        Thread.sleep(30000);
        System.out.println("main thread exit");
    }
}
class DaemonThread implements Runnable{
    @Override
    public void run(){
        while(true){
            processSth();
        }
    }

    private void processSth(){
        try{
            System.out.println("process @ " + System.currentTimeMillis());
            Thread.sleep(5000);
        }catch(Exception e){
            e.printStackTrace();
        }
    }
}

//----------begin output-----------
hrh@Solarex:Java$ java org.solarex.threadtest.JavaDaemonThread 
process @ 1413426032024
process @ 1413426037024
process @ 1413426042024
process @ 1413426047025
process @ 1413426052025
process @ 1413426057025
main thread exit
//----------end output-------------
```

If we don’t set the thread to be run as daemon thread, the program will never terminate even after main thread is finished it’s execution. Usually we create a daemon thread for functionalities that are not critical to system, for example logging thread or monitoring thread to capture the system resource details and their state.
<h3 id="threadlocal">Java Thread Local</h3>
Java ``ThreadLocal`` is used to create thread-local variables. We know that all threads of an Object share it’s variables, so if the variable is not thread safe, we can use synchronization but if we want to avoid synchronization, we can use ThreadLocal variables.Every thread has it’s own ``ThreadLocal`` variable and they can use it’s ``get()`` and ``set()`` methods to get the default value or change it’s value local to Thread. ``ThreadLocal`` instances are typically private static fields in classes that wish to associate state with a thread.

```java
package org.solarex.threadtest;

import java.text.SimpleDateFormat;
import java.util.Random;

public class ThreadLocalExample implements Runnable{
    // SimpleDateFormat is not thread-safe, so give one to each thread
    private static final ThreadLocal<SimpleDateFormat> formatter = new ThreadLocal<SimpleDateFormat>(){
        @Override
        protected SimpleDateFormat initialValue()
        {
            return new SimpleDateFormat("yyyyMMdd HHmm");
        }
    };

    public static void main(String[] args) throws InterruptedException {
        ThreadLocalExample obj = new ThreadLocalExample();
        for(int i=0 ; i<10; i++){
            Thread t = new Thread(obj, ""+i);
            Thread.sleep(new Random().nextInt(1000));
            t.start();
        }
    }

    @Override
    public void run() {
        System.out.println("Thread Name= "+Thread.currentThread().getName()+" default Formatter = "+formatter.get().toPattern());
        try {
            Thread.sleep(new Random().nextInt(1000));
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        formatter.set(new SimpleDateFormat());

        System.out.println("Thread Name= "+Thread.currentThread().getName()+" formatter = "+formatter.get().toPattern());
    }

}
//------begin output----------
hrh@Solarex:Java$ java org.solarex.threadtest.ThreadLocalExample 
Thread Name= 0 default Formatter = yyyyMMdd HHmm
Thread Name= 1 default Formatter = yyyyMMdd HHmm
Thread Name= 1 formatter = M/d/yy h:mm a
Thread Name= 0 formatter = M/d/yy h:mm a
Thread Name= 2 default Formatter = yyyyMMdd HHmm
Thread Name= 3 default Formatter = yyyyMMdd HHmm
Thread Name= 2 formatter = M/d/yy h:mm a
Thread Name= 4 default Formatter = yyyyMMdd HHmm
Thread Name= 3 formatter = M/d/yy h:mm a
Thread Name= 4 formatter = M/d/yy h:mm a
Thread Name= 5 default Formatter = yyyyMMdd HHmm
Thread Name= 6 default Formatter = yyyyMMdd HHmm
Thread Name= 5 formatter = M/d/yy h:mm a
Thread Name= 7 default Formatter = yyyyMMdd HHmm
Thread Name= 8 default Formatter = yyyyMMdd HHmm
Thread Name= 7 formatter = M/d/yy h:mm a
Thread Name= 6 formatter = M/d/yy h:mm a
Thread Name= 8 formatter = M/d/yy h:mm a
Thread Name= 9 default Formatter = yyyyMMdd HHmm
Thread Name= 9 formatter = M/d/yy h:mm a
//------end output------------
```

As you can see from the output that Thread-0 has changed the value of formatter but still thread-2 default formatter is same as the initialized value.
<h3 id="dump">Java Thread Dump</h3>
Java comes with ``jstack`` tool through which we can generate thread dump for a java process. This is a two step process.

+ Find out the PID of the java process using ``ps -eaf | grep java`` command
+ Run ``jstack`` tool as ``jstack PID`` to generate the thread dump output to console, you can append thread dump output to file using command ``jstack PID >> mydumps.tdump``
<h3 id="deadlock">How to Analytize Deadlock and avoid it in Java</h3>
Deadlock is a programming situation where two or more threads are blocked forever, this situation arises with at least two threads and two or more resources. 

```java
package org.solarex.threadtest;

public class ThreadDeadlock {

    public static void main(String[] args) throws InterruptedException {
        Object obj1 = new Object();
        Object obj2 = new Object();
        Object obj3 = new Object();
    
        Thread t1 = new Thread(new SyncThread(obj1, obj2), "t1");
        Thread t2 = new Thread(new SyncThread(obj2, obj3), "t2");
        Thread t3 = new Thread(new SyncThread(obj3, obj1), "t3");
        
        t1.start();
        Thread.sleep(5000);
        t2.start();
        Thread.sleep(5000);
        t3.start();
        
    }

}

class SyncThread implements Runnable{
    private Object obj1;
    private Object obj2;

    public SyncThread(Object o1, Object o2){
        this.obj1=o1;
        this.obj2=o2;
    }
    @Override
    public void run() {
        String name = Thread.currentThread().getName();
        System.out.println(name + " acquiring lock on "+obj1);
        synchronized (obj1) {
         System.out.println(name + " acquired lock on "+obj1);
         work();
         System.out.println(name + " acquiring lock on "+obj2);
         synchronized (obj2) {
            System.out.println(name + " acquired lock on "+obj2);
            work();
        }
         System.out.println(name + " released lock on "+obj2);
        }
        System.out.println(name + " released lock on "+obj1);
        System.out.println(name + " finished execution.");
    }
    private void work() {
        try {
            Thread.sleep(30000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

For analyzing deadlock, we need to look out for the threads with state as ``BLOCKED`` and then the resources it’s waiting to lock, every resource has a unique ID using which we can find which thread is already holding the lock on the object.

These are some of the guidelines using which we can avoid most of the deadlock situations.

+ **Avoid Nested Locks**: This is the most common reason for deadlocks, avoid locking another resource if you already hold one. It’s almost impossible to get deadlock situation if you are working with only one object lock. 
+ **Lock Only What is Required**: You should acquire lock only on the resources you have to work on, for example in above program I am locking the complete Object resource but if we are only interested in one of it’s fields, then we should lock only that specific field not complete object.
+ **Avoid waiting indefinitely**: You can get deadlock if two threads are waiting for each other to finish indefinitely using thread join. If your thread has to wait for another thread to finish, it’s always best to use join with maximum time you want to wait for thread to finish.
<h3 id="timer">Java Timer Thread</h3>

``java.util.Timer`` is a utility class that can be used to schedule a thread to be executed at certain time in future. Java ``Timer`` class can be used to schedule a task to be run one-time or to be run at regular intervals.``java.util.TimerTask`` is an abstract class that implements ``Runnable`` interface and we need to extend this class to create our own ``TimerTask`` that can be scheduled using java ``Timer`` class.Timer class is thread safe and multiple threads can share a single Timer object without need for external synchronization. Timer class uses ``java.util.TaskQueue`` to add tasks at given regular interval and at any time there can be only one thread running the ``TimerTask``, for example if you are creating a Timer to run every 10 seconds but single thread execution takes 20 seconds, then Timer object will keep adding tasks to the queue and as soon as one thread is finished, it will notify the queue and another thread will start executing.

Timer class uses Object **wait and notify** methods to schedule the tasks.

```java
package org.solarex.threadtest;

import java.util.Date;
import java.util.Timer;
import java.util.TimerTask;

public class MyTimerTask extends TimerTask {

    @Override
    public void run() {
        System.out.println("Timer task started at:"+new Date());
        completeTask();
        System.out.println("Timer task finished at:"+new Date());
    }

    private void completeTask() {
        try {
            //assuming it takes 20 secs to complete the task
            Thread.sleep(20000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
    
    public static void main(String args[]){
        TimerTask timerTask = new MyTimerTask();
        //running timer task as daemon thread
        Timer timer = new Timer(true);
        timer.scheduleAtFixedRate(timerTask, 0, 10*1000);
        System.out.println("TimerTask started");
        //cancel after sometime
        try {
            Thread.sleep(120000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        timer.cancel();
        System.out.println("TimerTask cancelled");
        try {
            Thread.sleep(30000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

}

//------begin output-----------
hrh@Solarex:Java$ java org.solarex.threadtest.MyTimerTask 
TimerTask started
Timer task started at:Thu Oct 16 13:55:16 CST 2014
Timer task finished at:Thu Oct 16 13:55:36 CST 2014
Timer task started at:Thu Oct 16 13:55:36 CST 2014
Timer task finished at:Thu Oct 16 13:55:56 CST 2014
Timer task started at:Thu Oct 16 13:55:56 CST 2014
Timer task finished at:Thu Oct 16 13:56:16 CST 2014
Timer task started at:Thu Oct 16 13:56:16 CST 2014
Timer task finished at:Thu Oct 16 13:56:36 CST 2014
Timer task started at:Thu Oct 16 13:56:36 CST 2014
Timer task finished at:Thu Oct 16 13:56:56 CST 2014
Timer task started at:Thu Oct 16 13:56:56 CST 2014
TimerTask cancelled
Timer task finished at:Thu Oct 16 13:57:16 CST 2014
//------end output--------------
```

The output confirms that if a task is already executing, Timer will **wait for it to finish and once finished**, it will start again the next task from the queue.

Timer object can be created to run the associated tasks as a daemon thread. Timer ``cancel()`` method is used to terminate the timer and discard any scheduled tasks, however it doesn’t interfere with the currently executing task and let it finish. If the timer is run as daemon thread, whether we cancel it or not, it will terminate as soon as all the user threads are finished executing.

Timer class contains several ``schedule()`` methods to schedule a task to run once at given date or after some delay. There are several ``scheduleAtFixedRate()`` methods to run a task periodically with certain interval.

While scheduling tasks using Timer, you should make sure that time interval is more than normal thread execution, otherwise tasks queue size will keep growing and eventually task will be executing always.
<h3 id="producer">Java Producer Consumer Problem</h3>

``java.util.concurrent.BlockingQueue`` is a Queue that supports operations that wait for the queue to become non-empty when retrieving and removing an element, and wait for space to become available in the queue when adding an element.

``BlockingQueue`` doesn’t accept null values and throw ``NullPointerException`` if you try to store null value in the queue.``BlockingQueue`` implementations are thread-safe. All queuing methods are atomic in nature and use internal locks or other forms of concurrency control.

``BlockingQueue`` interface is part of java collections framework and it’s primarily used for implementing producer consumer problem. We don’t need to worry about waiting for the space to be available for producer or object to be available for consumer in ``BlockingQueue`` as it’s handled by implementation classes of ``BlockingQueue``.Java provides several ``BlockingQueue`` implementations such as ``ArrayBlockingQueue``, ``LinkedBlockingQueue``, ``PriorityBlockingQueue``, ``SynchronousQueue`` etc.

While implementing producer consumer problem, we will use ``ArrayBlockingQueue`` implementation and following methods are important to know.

+ ``put(E e)``: This method is used to insert elements to the queue, if the queue is full it waits for the space to be available.
+ ``E take()``: This method retrieves and remove the element from the head of the queue, if queue is empty it waits for the element to be available.

```java
//Message.java
public class Message {
    private String msg;
     
    public Message(String str){
        this.msg=str;
    }
 
    public String getMsg() {
        return msg;
    }
 
}

//Producer.java
import java.util.concurrent.BlockingQueue;
 
public class Producer implements Runnable {
 
    private BlockingQueue<Message> queue;
     
    public Producer(BlockingQueue<Message> q){
        this.queue=q;
    }
    @Override
    public void run() {
        //produce messages
        for(int i=0; i<100; i++){
            Message msg = new Message(""+i);
            try {
                Thread.sleep(i);
                queue.put(msg);
                System.out.println("Produced "+msg.getMsg());
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        //adding exit message
        Message msg = new Message("exit");
        try {
            queue.put(msg);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
 
}

//Consumer.java
import java.util.concurrent.BlockingQueue;
 
public class Consumer implements Runnable{
 
private BlockingQueue<Message> queue;
     
    public Consumer(BlockingQueue<Message> q){
        this.queue=q;
    }
 
    @Override
    public void run() {
        try{
            Message msg;
            //consuming messages until exit message is received
            while((msg = queue.take()).getMsg() !="exit"){
            Thread.sleep(10);
            System.out.println("Consumed "+msg.getMsg());
            }
        }catch(InterruptedException e) {
            e.printStackTrace();
        }
    }
}

//ProducerConsumerService.java
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;
 
 
public class ProducerConsumerService {
 
    public static void main(String[] args) {
        //Creating BlockingQueue of size 10
        BlockingQueue<Message> queue = new ArrayBlockingQueue<>(10);
        Producer producer = new Producer(queue);
        Consumer consumer = new Consumer(queue);
        //starting producer to produce messages in queue
        new Thread(producer).start();
        //starting consumer to consume messages from queue
        new Thread(consumer).start();
        System.out.println("Producer and Consumer has been started");
    }
 
}
```
<h3 id="pool">Java Thread Pool</h3>

A thread pool manages the pool of worker threads, it contains a queue that keeps tasks waiting to get executed.

A thread pool manages the collection of ``Runnable`` threads and worker threads execute Runnable from the queue.

``java.util.concurrent.Executors`` provide implementation of ``java.util.concurrent.Executor`` interface to create the thread pool in java.

```java
// WorkerThread.java
public class WorkerThread implements Runnable {
    
    private String command;
    
    public WorkerThread(String s){
        this.command=s;
    }

    @Override
    public void run() {
        System.out.println(Thread.currentThread().getName()+" Start. Command = "+command);
        processCommand();
        System.out.println(Thread.currentThread().getName()+" End.");
    }

    private void processCommand() {
        try {
            Thread.sleep(5000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    @Override
    public String toString(){
        return this.command;
    }
}

//SimpleThreadPool.java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class SimpleThreadPool {

    public static void main(String[] args) {
        ExecutorService executor = Executors.newFixedThreadPool(5);
        for (int i = 0; i < 10; i++) {
            Runnable worker = new WorkerThread("" + i);
            executor.execute(worker);
          }
        executor.shutdown();
        while (!executor.isTerminated()) {
        }
        System.out.println("Finished all threads");
    }

}
```

Executors class provide simple implementation of ``ExecutorService`` using ``ThreadPoolExecutor`` but ``ThreadPoolExecutor`` provides much more feature than that. We can specify the number of threads that will be alive when we create ``ThreadPoolExecutor`` instance and we can limit the size of thread pool and create our own ``RejectedExecutionHandler`` implementation to handle the jobs that can’t fit in the worker queue.

```java
// RejectedExecutionHandlerImpl.java
import java.util.concurrent.RejectedExecutionHandler;
import java.util.concurrent.ThreadPoolExecutor;

public class RejectedExecutionHandlerImpl implements RejectedExecutionHandler {

    @Override
    public void rejectedExecution(Runnable r, ThreadPoolExecutor executor) {
        System.out.println(r.toString() + " is rejected");
    }

}

// MyMonitorThread.java
import java.util.concurrent.ThreadPoolExecutor;

public class MyMonitorThread implements Runnable
{
    private ThreadPoolExecutor executor;
    
    private int seconds;
    
    private boolean run=true;

    public MyMonitorThread(ThreadPoolExecutor executor, int delay)
    {
        this.executor = executor;
        this.seconds=delay;
    }
    
    public void shutdown(){
        this.run=false;
    }

    @Override
    public void run()
    {
        while(run){
                System.out.println(
                    String.format("[monitor] [%d/%d] Active: %d, Completed: %d, Task: %d, isShutdown: %s, isTerminated: %s",
                        this.executor.getPoolSize(),
                        this.executor.getCorePoolSize(),
                        this.executor.getActiveCount(),
                        this.executor.getCompletedTaskCount(),
                        this.executor.getTaskCount(),
                        this.executor.isShutdown(),
                        this.executor.isTerminated()));
                try {
                    Thread.sleep(seconds*1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
        }
            
    }
}

// WorkerPool.java
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.Executors;
import java.util.concurrent.ThreadFactory;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

public class WorkerPool {

    public static void main(String args[]) throws InterruptedException{
        //RejectedExecutionHandler implementation
        RejectedExecutionHandlerImpl rejectionHandler = new RejectedExecutionHandlerImpl();
        //Get the ThreadFactory implementation to use
        ThreadFactory threadFactory = Executors.defaultThreadFactory();
        //creating the ThreadPoolExecutor
        ThreadPoolExecutor executorPool = new ThreadPoolExecutor(2, 4, 10, TimeUnit.SECONDS, new ArrayBlockingQueue<Runnable>(2), threadFactory, rejectionHandler);
        //start the monitoring thread
        MyMonitorThread monitor = new MyMonitorThread(executorPool, 3);
        Thread monitorThread = new Thread(monitor);
        monitorThread.start();
        //submit work to the thread pool
        for(int i=0; i<10; i++){
            executorPool.execute(new WorkerThread("cmd"+i));
        }
        
        Thread.sleep(30000);
        //shut down the pool
        executorPool.shutdown();
        //shut down the monitor thread
        Thread.sleep(5000);
        monitor.shutdown();
        
    }
}

Notice that while initializing the ThreadPoolExecutor, we are keeping initial pool size as 2, maximum pool size to 4 and work queue size as 2. So if there are 4 running tasks and more tasks are submitted, the work queue will hold only 2 of them and rest of them will be handled by RejectedExecutionHandlerImpl.

Notice the change in active, completed and total completed task count of the executor. We can invoke ``shutdown()`` method to finish execution of all the submitted tasks and terminate the thread pool.
```
<h3 id="future">Java Callable Future</h3>

In last few posts, we learned a lot about java threads but sometimes we wish that **a thread could return some value that we can use**. Java 5 introduced ``java.util.concurrent.Callable`` interface in concurrency package that is similar to Runnable interface but it can return any Object and able to throw Exception.

Callable interface use Generic to define the return type of Object. ``Executors`` class provide useful methods to execute ``Callable`` in a thread pool. Since callable tasks run in parallel, we have to wait for the returned Object. Callable tasks return ``java.util.concurrent.Future`` object. Using Future we can find out the status of the Callable task and get the returned Object. It provides ``get()`` method that can wait for the Callable to finish and then return the result.

Future provides ``cancel()`` method to cancel the associated ``Callable`` task. There is an overloaded version of ``get()`` method where we can specify the time to wait for the result, it’s useful to avoid current thread getting blocked for longer time. There are ``isDone()`` and ``isCancelled()`` methods to find out the current status of associated ``Callable`` task.

```java
import java.util.ArrayList;
import java.util.Date;
import java.util.List;
import java.util.concurrent.Callable;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;
 
public class MyCallable implements Callable<String> {
 
    @Override
    public String call() throws Exception {
        Thread.sleep(1000);
        //return the thread name executing this callable task
        return Thread.currentThread().getName();
    }
     
    public static void main(String args[]){
        //Get ExecutorService from Executors utility class, thread pool size is 10
        ExecutorService executor = Executors.newFixedThreadPool(10);
        //create a list to hold the Future object associated with Callable
        List<Future<String>> list = new ArrayList<Future<String>>();
        //Create MyCallable instance
        Callable<String> callable = new MyCallable();
        for(int i=0; i< 100; i++){
            //submit Callable tasks to be executed by thread pool
            Future<String> future = executor.submit(callable);
            //add Future to the list, we can get return value using Future
            list.add(future);
        }
        for(Future<String> fut : list){
            try {
                //print the return value of Future, notice the output delay in console
                // because Future.get() waits for task to get completed
                System.out.println(new Date()+ "::"+fut.get());
            } catch (InterruptedException | ExecutionException e) {
                e.printStackTrace();
            }
        }
        //shut down the executor service now
        executor.shutdown();
    }
 
}
```

<h3 id="futuretask">Java FutureTask Example</h3>

``FutureTask`` is base concrete implementation of ``Future`` interface and provides asynchronous processing. It contains the methods to start and cancel a task and also methods that can return the state of the ``FutureTask`` as whether it’s completed or cancelled. We need a callable object to create a future task and then we can use Java Thread Pool Executor to process these asynchronously.

```java
// MyCallable.java
import java.util.concurrent.Callable;
 
public class MyCallable implements Callable<String> {
 
    private long waitTime;
     
    public MyCallable(int timeInMillis){
        this.waitTime=timeInMillis;
    }
    @Override
    public String call() throws Exception {
        Thread.sleep(waitTime);
        //return the thread name executing this callable task
        return Thread.currentThread().getName();
    }
 
}

// FutureTaskExample.java
import java.util.concurrent.ExecutionException;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.FutureTask;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.TimeoutException;
 
public class FutureTaskExample {
 
    public static void main(String[] args) {
        MyCallable callable1 = new MyCallable(1000);
        MyCallable callable2 = new MyCallable(2000);
 
        FutureTask<String> futureTask1 = new FutureTask<String>(callable1);
        FutureTask<String> futureTask2 = new FutureTask<String>(callable2);
 
        ExecutorService executor = Executors.newFixedThreadPool(2);
        executor.execute(futureTask1);
        executor.execute(futureTask2);
         
        while (true) {
            try {
                if(futureTask1.isDone() && futureTask2.isDone()){
                    System.out.println("Done");
                    //shut down executor service
                    executor.shutdown();
                    return;
                }
                 
                if(!futureTask1.isDone()){
                //wait indefinitely for future task to complete
                System.out.println("FutureTask1 output="+futureTask1.get());
                }
                 
                System.out.println("Waiting for FutureTask2 to complete");
                String s = futureTask2.get(200L, TimeUnit.MILLISECONDS);
                if(s !=null){
                    System.out.println("FutureTask2 output="+s);
                }
            } catch (InterruptedException | ExecutionException e) {
                e.printStackTrace();
            }catch(TimeoutException e){
                //do nothing
            }
        }
         
    }
 
}
```

When we run above program, you will notice that it doesn’t print anything for sometime because ``get()`` method of ``FutureTask`` waits for the task to get completed and then returns the output object. There is an overloaded method also to wait for only specified amount of time and we are using it for futureTask2. Also notice the use of ``isDone()`` method to make sure program gets terminated once all the tasks are executed.

Ref:

+ [Java Thread and Multithreading Tutorial](http://www.journaldev.com/1079/java-thread-tutorial)
+ [Concurrency](http://docs.oracle.com/javase/tutorial/essential/concurrency/)
