+++
title = "Android Looper Handler"
date = "2014-10-13"
slug = "2014/10/13/android-looper-handler"
Categories = ["dev", "android"]
+++
What can you do with ``Loopers`` and ``Handlers``? Basically, they implement a common concurrency pattern that I call the ``Pipeline Thread``. Here’s how it works:

+ The Pipeline Thread holds a queue of tasks which are just some units of work that can be executed or processed.
+ Other threads can safely push new tasks into the Pipeline Thread’s queue at any time.
+ The Pipeline Thread processes the queued tasks one after another. If there are no tasks queued, it blocks until a task appears in the queue.
+ Sometimes tasks can called messages and other names.

<!-- more -->

我们能用``Loopers``和``Handlers``来干什么？这两个类实现了一种通用的并发模型，我把它叫做：Pipeline 线程。它是这样工作的：

+ Pipeline 线程持有一个任务队列，这些任务就是一些可以执行的工作单元
+ 其他线程可以自由的将任务加到Pipeline线程的任务队列中去
+ Pipeline线程就按次序一个一个执行任务，如果任务队列中没有任务了，它就会自动阻塞直到有任务到来
+ 有些时候，任务可以叫做消息（messages）或者其他名字

``Looper`` is a class that turns a thread into a ``Pipeline Thread`` and ``Handler`` gives you a mechanism to push tasks into it from any other threads.The ``Looper`` is named so because it implements the loop – takes the next task, executes it, then takes the next one and so on. The ``Handler`` is called a handler because someone could not invent a better name.

Looper类可以将一个线程转换成Pipeline线程，而Handler提供了一种机制，你可以通过它将任务添加到Pipeline线程中。Looper之所以这么命名是因为它实现了循环——取一个task执行，然后再取下一个task执行，如此循环；Handler如此命名是因为他们无法想出一个更好的名字了~

Here’s what you should put into a Thread's ``run()`` method to turn it into a Pipeline Thread and to create a ``Handler`` so that other threads can assign tasks to it:

下面就是你需要添加到Thread类的run方法中的代码来创建一个你自己的Pipeline线程，并创建一个``Handler``以便其他线程可以将任务分发到此Pipeline线程中。

```java
@Override
public void run() {
  try {
    // preparing a looper on current thread
    // the current thread is being detected implicitly
    Looper.prepare();
 
    // now, the handler will automatically bind to the
    // Looper that is attached to the current thread
    // You don't need to specify the Looper explicitly
    handler = new Handler();
     
    // After the following line the thread will start
    // running the message loop and will not normally
    // exit the loop unless a problem happens or you
    // quit() the looper (see below)
    Looper.loop();
  } catch (Throwable t) {
    Log.e(TAG, "halted due to an error", t);
  } 
}
```

After that, you can just pass the handler to any other thread. It has a thread-safe interface that includes many operations, but the most straightforward ones are ``postMessage()`` and its relatives.

然后，你只要将这个handler对象传到其他任何线程中去，它有一个线程安全的接口，包括了很多操作，但是最主要的操作就是postMessage()以及相关的方法了。

For example, imagine another thread has a reference to the handler that was created in our Pipeline Thread. Here’s how that other thread can schedule an operation to be executed in the Pipeline Thread:

想象一下，一个线程A持有了handler对象的引用，此handler是在Pipeline线程中创建的，下面代码就可以让这个线程A在Pipeline线程中执行操作了：

```java
handler.post(new Runnable() {
  @Override
  public void run() {
    // this will be done in the Pipeline Thread
  }
});
```

By the way, the UI thread has a ``Looper`` created for it implicitly, so you can just create a ``Handler`` in activity's ``onCreate()`` and it will work fine:

UI线程拥有一个Looper（可以通过``Looper.getMainLooper()``方法获取，判断一个线程是否为主线程可以使用``Looper.getLooper() == Looper.getMainLooper()``来判断）。所以，你可以在``Activity``的``onCreate()``方法中直接新建一个handler对象：

```java
@Override
public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.main);
     
    // Create the Handler. It will implicitly bind to the Looper
    // that is internally created for this thread (since it is the UI thread)
    handler = new Handler();
}
```

什么时候使用多线程:

+ 耗时操作使用多线程, 耗时操作放在UI线程中会导致用户的操作无法得到响应.
+ 阻塞操作使用多线程, 理由同上.
+ 多核CPU的设备使用多线程, 可以有效提高CPU的利用率.
+ 并行操作使用多线程.

android中的多线程模型主要涉及的类有:Looper, Handler, MessageQueue, Message等. 

Looper类用来创建消息队列. 每个线程最多只能有一个消息队列, android中UI线程默认具有消息队列, 但非UI线程在默认情况下是不具备消息队列的. 如果需要在非UI线程中开启消息队列, 需要调用``Looper.prepare()``方法, 在该方法的执行过程中会创建一个``Looper``对象, 而``Looper``的构造函数中会创建一个``MessageQueue`` instance(Looper的构造函数是私有的, 在Looper类之外无法创建其对象).  此后再为该线程绑定一个Handler instance, 然后调用Looper.loop()方法, 就可以不断的从消息队列中取出消息和处理消息了. ``Looper.myLooper()``方法可以得到线程的Looper对象, 如果为null, 说明此时该线程尚未开启消息队列.

Handler类用于处理消息. 该类具有四个构造函数:

+ ``public Handler()``. 创建好的``Handler`` instance将绑定在代码所在的线程的消息队列上, 因此一定要确定该线程开启了消息队列, 否则程序将发生错误. 使用这个构造函数创建``Handler`` instance, 一般来说, 我们需要重写``Hanler``类的``handleMessage()``方法, 以便在之后的消息处理时调用.
+ ``public Handler(Callback callback)``. ``Callback``是``Handler``内部定义的一个接口, 因此想要使用这个构造函数创建``Handler``对象, 需要自定义一个类实现``Callback``接口, 并重写接口中定义的``handleMessage()``方法. 这个构造函数其实与无参的构造函数类似, 也要确保代码所在的线程开启了消息队列. 不同的是在之后处理消息时, 将调用``callback``的``handleMessage()``方法, 而不是``Handler``对象的``handleMssage()``方法.
+ ``public Handler(Looper looper)``. 这个构造函数表示创建一个``Handler`` instance, 并将其绑定在looper所在的线程上. 此时looper不能为null. 此时一般也需要重写``Hanler``类的``handleMessage()``方法
+ ``public Handler(Looper looper, Callback callback)``. 可以结合2和3理解.

``MessageQueue``类用于表示消息队列. 队列中的每一个Message都有一个when字段, 这个字段用来决定Message应该何时出对处理. 消息队列中的每一个Message根据when字段的大小由小到大排列, 排在最前面的消息会首先得到处理, 因此可以说消息队列并不是一个严格的先进先出的队列.

``Message``类用于表示消息. ``Message``对象可以通过arg1, arg2, obj字段和``setData()``携带数据, 此外还具有很多字段. when字段决定Message应该何时处理, target字段用来表示将由哪个Handler对象处理这个消息, next字段表示在消息队列中排在这个Message之后的下一个Message, callback字段如果不为null表示这个Message包装了一个runnable对象, what字段表示code, 即这个消息具体是什么类型的消息. 每个what都在其handler的namespace中, 我们只需要确保将由同一个handler处理的消息的what属性不重复就可以.

将消息压入消息队列: ``Message``对象的``target``字段关联了哪个线程的消息队列, 这个消息就会被压入哪个线程的消息队列中.

+ 调用``Handler``类中以``send``开头的方法可以将``Message``对象压入消息队列中, 调用Handler类中以post开头的方法可以将一个runnable对象包装在一个Message对象中, 然后再压入消息队列, 此时入队的Message其callback字段不为null, 值就是这个runnable对象. 调用``Handler``对象的这些方法入队的``Message``, 其target属性会被赋值为这个handler对象.
+ 调用``Message``对象的``sendToTarget()``方法可以将其本身压入与其target字段(即handler对象)所关联的消息队列中. 

将未来得及处理的消息从消息队列中删除:调用Handler对象中以remove开头的方法就可以.

从消息队列中取出消息并处理消息: 所有在消息队列中的消息, 都具有target字段. 消息是在target所关联的线程上被取出和处理的.

+ 如果取出的``Message``对象的callback字段不为null, 那么就调用``callback``字段的``run()``方法(callback字段的类型是runnable). 注意此时并不开启一个新的线程运行run()方法, 而是直接在handler对象(即``Message``的target字段)所关联的线程上运行.
+ 如果取出的Message对象的callback字段为null, 且``Handler``对象中的callback字段也为null, 那么这个消息将由``Handler``对象的``handleMessage(msg)``方法处理. 注意Message对象的callback字段是Runnable类型的而Handler对象的callback字段是Callback类型的, Handler对象的callback字段是在创建Handler instance的时候指定的, 如果没有指定则这个字段为null, 详见Handler类的四个构造方法.
+ 如果取出的``Message``对象的callback字段为null, 且Handler对象中的callback字段不为null, 那么这个消息将由``Handler``对象中的callback字段的handleMessage方法处理.

线程间通信: 有了以上的叙述, 线程间的通信也就好理解了. 假如一个handler关联了A线程上的消息队列, 那么我们可以在B线程上调用handler的相关方法向A线程上的消息队列压入一个Message, 这个Message将在A线程上得到处理.

REF:[Android Guts: Intro to Loopers and Handlers](http://mindtherobot.com/blog/159/android-guts-intro-to-loopers-and-handlers/)


