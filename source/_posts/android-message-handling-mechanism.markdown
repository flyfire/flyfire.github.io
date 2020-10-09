---
layout: post
title: "Android Message Handling Mechanism"
date: 2015-09-22 17:28
comments: true
categories: 
- dev
- android
tags:
- android
---

<p><center><img src="/images/android_robot.png" width="255" height="300"></center>

Android is a message driven, message driven several elements:

+ The message says: Message
+ Message queue: MessageQueue
+ The news cycle, remove the message processing for circulation: Looper
+ Message processing, message loop out messages from the message queue should be carried out after the processing of messages: Handler

Usually we use most often is ``Message`` and ``Handler``, if you use ``HandlerThread`` or similar ``HandlerThread`` things may come into contact with ``Looper``, while ``MessageQueue`` is used inside ``Looper``, for the standard SDK, we are unable to instantiate and use (constructor is package visibility).
　　
We usually come into contact with ``Looper``, ``Message``, ``Handler`` are realized by JAVA, Android for Linux based systems, the bottom with a C, C++, and NDK exist, how could the message driven model only exists in the JAVA layer, in fact, there are corresponding to the Java layer such as ``Looper``, ``MessageQueue`` in the Native layer.

Android是消息驱动的，实现消息驱动有几个要素：

+ 消息的表示：Message
+ 消息队列：MessageQueue
+ 消息循环，用于循环取出消息进行处理：Looper
+ 消息处理，消息循环从消息队列中取出消息后要对消息进行处理：Handler

平时我们最常使用的就是Message与Handler了，如果使用过HandlerThread或者自己实现类似HandlerThread的东西可能还会接触到Looper，而MessageQueue是Looper内部使用的，对于标准的SDK，我们是无法实例化并使用的（构造函数是包可见性）。

我们平时接触到的Looper、Message、Handler都是用JAVA实现的，Android做为基于Linux的系统，底层用C、C++实现的，而且还有NDK的存在，消息驱动的模型怎么可能只存在于JAVA层，实际上，在Native层存在与Java层对应的类如Looper、MessageQueue等。

<!-- more -->

<h2 id="init-message-queue">Initialization message queue</h2>

First to see if a thread to achieve the message loop should do, taking HandlerThread as an example:

首先来看一下如果一个线程想实现消息循环应该怎么做，以HandlerThread为例：

```java
public void run() {
    mTid = Process.myTid();
    Looper.prepare(); //red mark
    synchronized (this) {
        mLooper = Looper.myLooper();
        notifyAll();
    }
    Process.setThreadPriority(mPriority);
    onLooperPrepared();
    Looper.loop(); //red mark
    mTid = -1;
} 
```

In the red mark of the two, first call prepare to initialize the ``MessageQueue`` and ``Looper``, and then call the loop enter the message loop. First look at the ``Looper.prepare``.

主要是红色标明的两句，首先调用prepare初始化MessageQueue与Looper，然后调用loop进入消息循环。先看一下Looper.prepare。

```java
public static void prepare() {
    prepare(true);
}

private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper(quitAllowed));
}
```

Overloaded functions, ``quitAllowed`` default to true, from the name can be seen is whether the message loop can exit, the default is to exit, the Main thread (UI thread) initialization message loop is called ``prepareMainLooper``, pass is false. The use of ``ThreadLocal``, each thread can initialize a ``Looper``.

Look at the ``Looper`` in the initialization did:

重载函数，quitAllowed默认为true，从名字可以看出来就是消息循环是否可以退出，默认是可退出的，Main线程（UI线程）初始化消息循环时会调用prepareMainLooper，传进去的是false。使用了ThreadLocal，每个线程可以初始化一个Looper。

再来看一下Looper在初始化时都做了什么：

```java
private Looper(boolean quitAllowed) {
    mQueue = new MessageQueue(quitAllowed);
    mRun = true;
    mThread = Thread.currentThread();
}

MessageQueue(boolean quitAllowed) {
    mQuitAllowed = quitAllowed;
    nativeInit();
} 
```

In the ``Looper`` initialization, a ``MessageQueue`` object created saved in the mQueue member. The ``MessageQueue`` constructor is package visibility, so we cannot be used directly, while the ``MessageQueue`` initialization is called the ``nativeInit``, which is a Native method:

在Looper初始化时，新建了一个MessageQueue的对象保存了在成员mQueue中。MessageQueue的构造函数是包可见性，所以我们是无法直接使用的，在MessageQueue初始化的时候调用了nativeInit，这是一个Native方法：

```c
static void android_os_MessageQueue_nativeInit(JNIEnv* env, jobject obj) {
    NativeMessageQueue* nativeMessageQueue = new NativeMessageQueue();
    if (!nativeMessageQueue) {
        jniThrowRuntimeException(env, "Unable to allocate native queue");
        return;
    }

    nativeMessageQueue->incStrong(env);
    android_os_MessageQueue_setNativeMessageQueue(env, obj, nativeMessageQueue);
}

static void android_os_MessageQueue_setNativeMessageQueue(JNIEnv* env, jobject messageQueueObj,
        NativeMessageQueue* nativeMessageQueue) {
    env->SetIntField(messageQueueObj, gMessageQueueClassInfo.mPtr,
             reinterpret_cast<jint>(nativeMessageQueue));
}
```

In ``nativeInit``, the new object of a Native layer ``MessageQueue``, and the address stored in the Java layer of the ``MessageQueue`` member of the ``mPtr``, there are a lot of implementation of Android, a class has been implemented in Java layer and Native layer, the ``GetFieldID`` and ``SetIntField`` JNI saved Native layer class an address to the Java layer for an instance of the class ``mPtr`` members, such as ``Parcel``.

Then the realization of ``NativeMessageQueue``:

在nativeInit中，new了一个Native层的MessageQueue的对象，并将其地址保存在了Java层MessageQueue的成员mPtr中，Android中有好多这样的实现，一个类在Java层与Native层都有实现，通过JNI的GetFieldID与SetIntField把Native层的类的实例地址保存到Java层类的实例的mPtr成员中，比如Parcel。

再看NativeMessageQueue的实现：

```c
NativeMessageQueue::NativeMessageQueue() : mInCallback(false), mExceptionObj(NULL) {
    mLooper = Looper::getForThread();
    if (mLooper == NULL) {
        mLooper = new Looper(false);
        Looper::setForThread(mLooper);
    }
}
```

Gets a Native layer of the ``Looper`` object in the constructor of the ``NativeMessageQueue``, Native layer Looper also use thread local storage, note that new ``Looper`` introduced parameter false.

在NativeMessageQueue的构造函数中获得了一个Native层的Looper对象，Native层的Looper也使用了线程本地存储，注意new Looper时传入了参数false。

```c
Looper::Looper(bool allowNonCallbacks) :
        mAllowNonCallbacks(allowNonCallbacks), mSendingMessage(false),
        mResponseIndex(0), mNextMessageUptime(LLONG_MAX) {
    int wakeFds[2];
    int result = pipe(wakeFds);
    LOG_ALWAYS_FATAL_IF(result != 0, "Could not create wake pipe.  errno=%d", errno);

    mWakeReadPipeFd = wakeFds[0];
    mWakeWritePipeFd = wakeFds[1];

    result = fcntl(mWakeReadPipeFd, F_SETFL, O_NONBLOCK);
    LOG_ALWAYS_FATAL_IF(result != 0, "Could not make wake read pipe non-blocking.  errno=%d",
            errno);

    result = fcntl(mWakeWritePipeFd, F_SETFL, O_NONBLOCK);
    LOG_ALWAYS_FATAL_IF(result != 0, "Could not make wake write pipe non-blocking.  errno=%d",
            errno);

    // Allocate the epoll instance and register the wake pipe.
    mEpollFd = epoll_create(EPOLL_SIZE_HINT);
    LOG_ALWAYS_FATAL_IF(mEpollFd <0, "Could not create epoll instance.  errno=%d", errno);

    struct epoll_event eventItem;
    memset(& eventItem, 0, sizeof(epoll_event)); // zero out unused members of data field union
    eventItem.events = EPOLLIN;
    eventItem.data.fd = mWakeReadPipeFd;
    result = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, mWakeReadPipeFd, & eventItem);
    LOG_ALWAYS_FATAL_IF(result != 0, "Could not add wake read pipe to epoll instance.  errno=%d",
            errno);
}
```

Native layer in ``Looper`` using ``epoll``. Initializes a pipeline, ``mWakeWritePipeFd`` and ``mWakeReadPipeFd`` were preserved to write and read end of pipe end, and monitor the read end of the EPOLLIN event. Note the initializer list of values, the value of ``mAllowNonCallbacks`` for ``false``.

What is ``mAllowNonCallback``? The use of ``epoll`` to monitor ``mWakeReadPipeFd`` events? In fact, Native ``Looper`` can not only monitor the descriptors, ``Looper`` also provides the ``addFd`` method:

Native层的Looper使用了epoll。初始化了一个管道，用mWakeWritePipeFd与mWakeReadPipeFd分别保存了管道的写端与读端，并监听了读端的EPOLLIN事件。注意下初始化列表的值，mAllowNonCallbacks的值为false。

mAllowNonCallback是做什么的？使用epoll仅为了监听mWakeReadPipeFd的事件？其实Native Looper不仅可以监听这一个描述符，Looper还提供了addFd方法：

```c
int addFd(int fd, int ident, int events, ALooper_callbackFunc callback, void* data);
int addFd(int fd, int ident, int events, const sp<LooperCallback>& callback, void* data);
```

``fd`` said to monitor descriptor. ``ident`` said to monitor event identification, Value must be >=0 or be ALOOPER_POLL_BACK(-2), ``event`` said to monitor events, ``callback`` is a callback function when the event occurs, This is the role of ``mAllowNonCallbacks``, When the ``mAllowNonCallbacks`` is true allows callback to NULL, ``ident`` in ``pollOnce`` as a result of return, Otherwise, do not allow the callback is empty, When the callback is not NULL, The value of ident will be ignored. Or just look at the code easy to understand:

fd表示要监听的描述符。ident表示要监听的事件的标识，值必须>=0或者为ALOOPER_POLL_CALLBACK(-2)，event表示要监听的事件，callback是事件发生时的回调函数，mAllowNonCallbacks的作用就在于此，当mAllowNonCallbacks为true时允许callback为NULL，在pollOnce中ident作为结果返回，否则不允许callback为空，当callback不为NULL时，ident的值会被忽略。还是直接看代码方便理解：

```c
int Looper::addFd(int fd, int ident, int events, const sp<LooperCallback>& callback, void* data) {
#if DEBUG_CALLBACKS
    ALOGD("%p ~ addFd - fd=%d, ident=%d, events=0x%x, callback=%p, data=%p", this, fd, ident,
            events, callback.get(), data);
#endif
    if (!callback.get()) {
        if (! mAllowNonCallbacks) {
            ALOGE("Invalid attempt to set NULL callback but not allowed for this looper.");
            return -1;
        }
        if (ident <0) {
            ALOGE("Invalid attempt to set NULL callback with ident <0.");
            return -1;
        }
    } else {
        ident = ALOOPER_POLL_CALLBACK;
    }

    int epollEvents = 0;
    if (events & ALOOPER_EVENT_INPUT) epollEvents |= EPOLLIN;
    if (events & ALOOPER_EVENT_OUTPUT) epollEvents |= EPOLLOUT;

    { // acquire lock
        AutoMutex _l(mLock);

        Request request;
        request.fd = fd;
        request.ident = ident;
        request.callback = callback;
        request.data = data;

        struct epoll_event eventItem;
        memset(& eventItem, 0, sizeof(epoll_event)); // zero out unused members of data field union
        eventItem.events = epollEvents;
        eventItem.data.fd = fd;

        ssize_t requestIndex = mRequests.indexOfKey(fd);
        if (requestIndex <0) {
            int epollResult = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, fd, & eventItem);
            if (epollResult <0) {
                ALOGE("Error adding epoll events for fd %d, errno=%d", fd, errno);
                return -1;
            }
            mRequests.add(fd, request);
        } else {
            int epollResult = epoll_ctl(mEpollFd, EPOLL_CTL_MOD, fd, & eventItem);
            if (epollResult <0) {
                ALOGE("Error modifying epoll events for fd %d, errno=%d", fd, errno);
                return -1;
            }
            mRequests.replaceValueAt(requestIndex, request);
        }
    } // release lock
    return 1;
}
```

If the ``callback`` is empty will check to see whether to allow ``mAllowNonCallbacks`` callback is empty, if the callback is empty will determine whether the ident > =0. If the callback is not empty the ident value of ``ALOOPER_POLL_CALLBACK``, no matter in what value is.

Then the parameters of incoming values encapsulated into a ``Request`` structure, and the descriptor for the key to save to a ``KeyedVector`` mRequests, and then through the ``epoll_ctl`` to add or replace (if the descriptor before calling the ``addFD`` add monitoring) on this descriptor wiretapping.

The class diagram:

如果callback为空会检查mAllowNonCallbacks看是否允许callback为空，如果允许callback为空还会检测ident是否>=0。如果callback不为空会把ident的值赋值为ALOOPER_POLL_CALLBACK，不管传进来的是什么值。

接下来把传进来的参数值封装到一个Request结构体中，并以描述符为键保存到一个KeyedVector mRequests中，然后通过epoll_ctl添加或替换（如果这个描述符之前有调用addFD添加监听）对这个描述符事件的监听。

类图：

<p><center><img src="/images/looper_epoll.jpg" width="430" height="225"></center>

<h2 id="send-message">Send a message</h2>

Through the ``Looper.prepare`` initialized the message queue can be called after the ``Looper.loop`` enter the message loop, then we can send message to the message queue, message loop will remove messages are processed, before seeing the message processing, look at the news is how to be added to the message queue.

In the Java layer, the ``Message`` class represents a message object, to send a message must first obtain a message object, the constructor of the ``Message`` class is public, but does not recommend direct new Message, ``Message`` stored in a cache of the message pool, we can get a message from the buffer pool to use ``obtain``, ``Message`` after using the system calls the recycle recovery, if they new a lot of ``Message``, after each use system into the buffer pool, will occupy a lot of memory, as shown below:

通过Looper.prepare初始化好消息队列后就可以调用Looper.loop进入消息循环了，然后我们就可以向消息队列发送消息，消息循环就会取出消息进行处理，在看消息处理之前，先看一下消息是怎么被添加到消息队列的。

在Java层，Message类表示一个消息对象，要发送消息首先就要先获得一个消息对象，Message类的构造函数是public的，但是不建议直接new Message，Message内部保存了一个缓存的消息池，我们可以用obtain从缓存池获得一个消息，Message使用完后系统会调用recycle回收，如果自己new很多Message，每次使用完后系统放入缓存池，会占用很多内存的，如下所示：

```java
public static Message obtain() {
    synchronized (sPoolSync) {
        if (sPool != null) {
            Message m = sPool;
            sPool = m.next;
            m.next = null;
            sPoolSize--;
            return m;
        }
    }
    return new Message();
}

public void recycle() {
    clearForRecycle();

    synchronized (sPoolSync) {
        if (sPoolSize < MAX_POOL_SIZE) {
            next = sPool;
            sPool = this;
            sPoolSize++;
        }
    }
}
```


The internal ``Message`` achieve a list by ``next`` members, such as the ``sPool`` to cache a list of ``Messages``.The message object get how to send it, as we all know is by ``Handler`` post, ``sendMessage`` and other methods, but these methods are ultimately ``sendMessageAtTime`` with a method call:

Message内部通过next成员实现了一个链表，这样sPool就了为了一个Messages的缓存链表。

消息对象获取到了怎么发送呢，大家都知道是通过Handler的post、sendMessage等方法，其实这些方法最终都是调用的同一个方法sendMessageAtTime:

```java
    public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, uptimeMillis);
    } 
```

``sendMessageAtTime`` access to the message queue and then call the ``enqueueMessage`` method, the ``mQueue`` is obtained from the message queue associated with the ``Handler`` ``Looper``.

sendMessageAtTime获取到消息队列然后调用enqueueMessage方法，消息队列mQueue是从与Handler关联的Looper获得的。

```java
    private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }
```

``enqueueMessage`` message target is set to the current handler, and then call the ``MessageQueue`` ``enqueueMessage``, before calling ``queue.enqueueMessage`` judge ``mAsynchronous``, from the name is asynchronous message means, to understand the role of Asynchronous, need to understand a concept ``Barrier``.

enqueueMessage将message的target设置为当前的handler，然后调用MessageQueue的enqueueMessage，在调用queue.enqueueMessage之前判断了mAsynchronous，从名字看是异步消息的意思，要明白Asynchronous的作用，需要先了解一个概念Barrier。

<h2 id="barrier-and-async-message">Barrier and Asynchronous Message</h2>

``Barrier`` is what meaning, from the name is an interceptor, the interceptor behind the news is temporarily unable to perform, until this interceptor is removed, the ``MessageQueue`` has a function called ``enqueueSyncBarier`` can add a ``Barrier``.

Barrier是什么意思呢，从名字看是一个拦截器，在这个拦截器后面的消息都暂时无法执行，直到这个拦截器被移除了，MessageQueue有一个函数叫enqueueSyncBarier可以添加一个Barrier。

```java
    int enqueueSyncBarrier(long when) {
        // Enqueue a new sync barrier token.
        // We don't need to wake the queue because the purpose of a barrier is to stall it.
        synchronized (this) {
            final int token = mNextBarrierToken++;
            final Message msg = Message.obtain();
            msg.arg1 = token;

            Message prev = null;
            Message p = mMessages;
            if (when != 0) {
                while (p != null && p.when <= when) {
                    prev = p;
                    p = p.next;
                }
            }
            if (prev != null) { // invariant: p == prev.next
                msg.next = p;
                prev.next = msg;
            } else {
                msg.next = p;
                mMessages = msg;
            }
            return token;
        }
    }
```

In ``enqueueSyncBarrier``, Obtain a ``Message``, And set ``msg.arg1=token``, Token is only one of each call to ``enqueueSyncBarrier`` since the increase of int value, Objective is to return a token only each call to enqueueSyncBarrier, The ``Message`` also need to set the execution time, And then inserted into the message queue, Special is that the ``Message`` is not set to target, Msg.target null.

Enter the message loop will keep the news from the ``MessageQueue`` in the implementation, call the next function ``MessageQueue``, in which there was a:

在enqueueSyncBarrier中，obtain了一个Message，并设置msg.arg1=token，token仅是一个每次调用enqueueSyncBarrier时自增的int值，目的是每次调用enqueueSyncBarrier时返回唯一的一个token，这个Message同样需要设置执行时间，然后插入到消息队列，特殊的是这个Message没有设置target，即msg.target为null。

进入消息循环后会不停地从MessageQueue中取消息执行，调用的是MessageQueue的next函数，其中有这么一段：

```java
Message msg = mMessages;
if (msg != null && msg.target == null) {
    // Stalled by a barrier.  Find the next asynchronous message in the queue.
    do {
        prevMsg = msg;
        msg = msg.next;
    } while (msg != null && !msg.isAsynchronous());
}
```

If the head of the queue for the message target null says it is a ``Barrier``, because there are only two ways to add ``mMessages`` news, one is ``enqueueMessage``, another is ``enqueueBarrier``, and ``enqueueMessage`` if ``msg.target`` null is a direct throw an exception, will see the back.

Asynchronous message call is actually the case, we can through the ``enqueueBarrier`` to the message queue insert a ``Barrier``, then the implementation of synchronization message queue in the ``Barrier`` time will be the ``Barrier`` intercept cannot execute until we call ``removeBarrier``, remove the ``Barrier``, and asynchronous message has no effect, the default is the synchronous message the message, unless we call the ``Message`` ``setAsynchronous``, this method is hidden. Only in the Handler initialization parameter to the Handler specified by the message sent is asynchronous, so that it will be in the ``Handler`` ``enqueueMessage`` called ``Message`` ``setAsynchronous`` setup messages are asynchronous, from the above ``Handler.enqueueMessage`` code can be seen in the.

The so-called asynchronous message, there is only one, is in setting the ``Barrier`` still can be not affected by ``Barrier`` is normal, if not set ``Barrier``, ``asynchronous`` message is no different from the ``synchronous`` message, can pass the ``removeSyncBarrier`` to remove the ``Barrier``:

如果队列头部的消息的target为null就表示它是个Barrier，因为只有两种方法往mMessages中添加消息，一种是enqueueMessage，另一种是enqueueBarrier，而enqueueMessage中如果mst.target为null是直接抛异常的，后面会看到。

所谓的异步消息其实就是这样的，我们可以通过enqueueBarrier往消息队列中插入一个Barrier，那么队列中执行时间在这个Barrier以后的同步消息都会被这个Barrier拦截住无法执行，直到我们调用removeBarrier移除了这个Barrier，而异步消息则没有影响，消息默认就是同步消息，除非我们调用了Message的setAsynchronous，这个方法是隐藏的。只有在初始化Handler时通过参数指定往这个Handler发送的消息都是异步的，这样在Handler的enqueueMessage中就会调用Message的setAsynchronous设置消息是异步的，从上面Handler.enqueueMessage的代码中可以看到。

 所谓异步消息，其实只有一个作用，就是在设置Barrier时仍可以不受Barrier的影响被正常处理，如果没有设置Barrier，异步消息就与同步消息没有区别，可以通过removeSyncBarrier移除Barrier：

```java
void removeSyncBarrier(int token) {
    // Remove a sync barrier token from the queue.
    // If the queue is no longer stalled by a barrier then wake it.
    final boolean needWake;
    synchronized (this) {
        Message prev = null;
        Message p = mMessages;
        while (p != null && (p.target != null || p.arg1 != token)) {
            prev = p;
            p = p.next;
        }
        if (p == null) {
            throw new IllegalStateException("The specified message queue synchronization "
                    + " barrier token has not been posted or has already been removed.");
        }
        if (prev != null) {
            prev.next = p.next;
            needWake = false;
        } else {
            mMessages = p.next;
            needWake = mMessages == null || mMessages.target != null;
        }
        p.recycle();
    }
    if (needWake) {
        nativeWake(mPtr);
    }
}
```

The return value of parameter token is ``enqueueSyncBarrier``, if not non-existent exception will be thrown calls the specified token.

参数token就是enqueueSyncBarrier的返回值，如果没有调用指定的token不存在是会抛异常的。

<h2 id="enqueue-message">enqueueMessage</h2>

Then look at how ``MessageQueue`` ``enqueueMessage``.

接下来看一下是怎么MessageQueue的enqueueMessage。

```java
    final boolean enqueueMessage(Message msg, long when) {
        if (msg.isInUse()) {
            throw new AndroidRuntimeException(msg + " This message is already in use."); //red mark
        }
        if (msg.target == null) {
            throw new AndroidRuntimeException("Message must have a target.");
        }

        boolean needWake;
        synchronized (this) {
            if (mQuiting) {
                RuntimeException e = new RuntimeException(
                        msg.target + " sending message to a Handler on a dead thread");
                Log.w("MessageQueue", e.getMessage(), e);
                return false;
            }

            msg.when = when;
            Message p = mMessages;
            if (p == null || when == 0 || when < p.when) {
                // New head, wake up the event queue if blocked.
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
                // Inserted within the middle of the queue.  Usually we don't have to wake
                // up the event queue unless there is a barrier at the head of the queue
                // and the message is the earliest asynchronous message in the queue.
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                for (;;) {
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {
                        break;
                    }
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                msg.next = p; // invariant: p == prev.next
                prev.next = msg;
            }
        }
        if (needWake) {
            nativeWake(mPtr);
        }
        return true;
    }
```

When the ``msg.target`` null is directly thrown exception.The first judgment in ``enqueueMessage``, if the current message queue is empty, the execution time of when or add a new message is 0, or add a new message than the execution time of the message queue head message execution time is early, the message is added to the message queue head (message queue according to the time sequence), or to find a suitable location to the current message is added to the message queue.

注意上面代码红色的部分，当msg.target为null时是直接抛异常的。

在enqueueMessage中首先判断，如果当前的消息队列为空，或者新添加的消息的执行时间when是0，或者新添加的消息的执行时间比消息队列头的消息的执行时间还早，就把消息添加到消息队列头（消息队列按时间排序），否则就要找到合适的位置将当前消息添加到消息队列。

<h2 id="native-send-message>Native send a message</h2>

``Message`` model not only the Java layer, Native layer can also be used, the front also saw the message queue initialization also initializes the ``Looper`` and the ``NativeMessageQueue`` Native layer, so the Native layer should also can send a message. Unlike the Java layer, Native layer is the message ``Looper``, also sending method all end is called ``sendMessageAtTime``:

消息模型不只是Java层用的，Native层也可以用，前面也看到了消息队列初始化时也同时初始化了Native层的Looper与NativeMessageQueue，所以Native层应该也是可以发送消息的。与Java层不同的是，Native层是通过Looper发消息的，同样所有的发送方法最终是调用sendMessageAtTime：

```c
void Looper::sendMessageAtTime(nsecs_t uptime, const sp<MessageHandler>& handler,
        const Message& message) {
#if DEBUG_CALLBACKS
    ALOGD("%p ~ sendMessageAtTime - uptime=%lld, handler=%p, what=%d",
            this, uptime, handler.get(), message.what);
#endif

    size_t i = 0;
    { // acquire lock
        AutoMutex _l(mLock);

        size_t messageCount = mMessageEnvelopes.size();
        while (i <messageCount && uptime >= mMessageEnvelopes.itemAt(i).uptime) {
            i += 1;
        }

        MessageEnvelope messageEnvelope(uptime, handler, message);
        mMessageEnvelopes.insertAt(messageEnvelope, i, 1);

        // Optimization: If the Looper is currently sending a message, then we can skip
        // the call to wake() because the next thing the Looper will do after processing
        // messages is to decide when the next wakeup time should be.  In fact, it does
        // not even matter whether this code is running on the Looper thread.
        if (mSendingMessage) {
            return;
        }
    } // release lock

    // Wake the poll loop only when we enqueue a new message at the head.
    if (i == 0) {
        wake();
    }
}
```

Native Message only one int type what field is used to distinguish between different message, ``SendMessageAtTime`` ``Message`` is specified, ``Message`` to the execution time of when, And processing the message ``Handler: MessageHandler``, Then use the ``MessageEnvelope`` package of time, ``MessageHandler`` and ``Message``, Native messages are saved to the ``mMessageEnvelopes``, ``MMessageEnvelopes`` is a ``Vector<MessageEnvelope>``. The Native message is also according to the time sequence, and the Java layer information is stored separately in the two cohort.

Native Message只有一个int型的what字段用来区分不同的消息，sendMessageAtTime指定了Message，Message要执行的时间when，与处理这个消息的Handler：MessageHandler，然后用MessageEnvelope封装了time, MessageHandler与Message，Native层发的消息都保存到了mMessageEnvelopes中，mMessageEnvelopes是一个Vector<MessageEnvelope>。Native层消息同样是按时间排序，与Java层的消息分别保存在两个队列里。

<h2 id="message-loop">The message loop</h2>

Message queue initialization is good, also know how to send a message, the following is how to process the message, see the ``Handler.loop`` function:

消息队列初始化好了，也知道怎么发消息了，下面就是怎么处理消息了，看Handler.loop函数：

```java
    public static void loop() {
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;

        // Make sure the identity of this thread is that of the local process,
        // and keep track of what that identity token actually is.
        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();

        for (;;) {
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }

            // This must be in a local variable, in case a UI event sets the logger
            Printer logging = me.mLogging;
            if (logging != null) {
                logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);
            }

            msg.target.dispatchMessage(msg);

            if (logging != null) {
                logging.println("<<<<<Finished to " + msg.target + " " + msg.callback);
            }

            // Make sure that during the course of dispatching the
            // identity of the thread wasn't corrupted.
            final long newIdent = Binder.clearCallingIdentity();
            if (ident != newIdent) {
                Log.wtf(TAG, "Thread identity changed from 0x"
                        + Long.toHexString(ident) + " to 0x"
                        + Long.toHexString(newIdent) + " while dispatching to "
                        + msg.target.getClass().getName() + " "
                        + msg.callback + " what=" + msg.what);
            }

            msg.recycle();
        }
    }
```

When loop took a ``Message`` from the ``MessageQueue``, call ``msg.target.dispatchMessage (MSG)``, handler and message associated target is to send message, so that the familiar ``dispatchMessage`` calls, ``Message`` was treated by recycle. When ``queue.next`` returns null will exit the news cycle, then look at ``MessageQueue.next`` is how to remove the message, and returns null when.

loop每次从MessageQueue取出一个Message，调用msg.target.dispatchMessage(msg)，target就是发送message时跟message关联的handler，这样就调用到了熟悉的dispatchMessage，Message被处理后会被recycle。当queue.next返回null时会退出消息循环，接下来就看一下MessageQueue.next是怎么取出消息的，又会在什么时候返回null。

```java
		final Message next() {
        int pendingIdleHandlerCount = -1; // -1 only during first iteration
        int nextPollTimeoutMillis = 0;

        for (;;) {
            if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
            }
            nativePollOnce(mPtr, nextPollTimeoutMillis);

            synchronized (this) {
                if (mQuiting) {
                    return null;
                }

                // Try to retrieve the next message.  Return if found.
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;
                if (msg != null && msg.target == null) {
                    // Stalled by a barrier.  Find the next asynchronous message in the queue.
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                if (msg != null) {
                    if (now < msg.when) {
                        // Next message is not ready.  Set a timeout to wake up when it is ready.
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        // Got a message.
                        mBlocked = false;
                        if (prevMsg != null) {
                            prevMsg.next = msg.next;
                        } else {
                            mMessages = msg.next;
                        }
                        msg.next = null;
                        if (false) Log.v("MessageQueue", "Returning message: " + msg);
                        msg.markInUse();
                        return msg;
                    }
                } else {
                    // No more messages.
                    nextPollTimeoutMillis = -1;
                }

                // If first time idle, then get the number of idlers to run.
                // Idle handles only run if the queue is empty or if the first message
                // in the queue (possibly a barrier) is due to be handled in the future.
                if (pendingIdleHandlerCount <0
                        && (mMessages == null || now < mMessages.when)) {
                    pendingIdleHandlerCount = mIdleHandlers.size();
                }
                if (pendingIdleHandlerCount <= 0) {
                    // No idle handlers to run.  Loop and wait some more.
                    mBlocked = true;
                    continue;
                }

                if (mPendingIdleHandlers == null) {
                    mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
                }
                mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
            }

            // Run the idle handlers.
            // We only ever reach this code block during the first iteration.
            for (int i = 0; i <pendingIdleHandlerCount; i++) {
                final IdleHandler idler = mPendingIdleHandlers[i];
                mPendingIdleHandlers[i] = null; // release the reference to the handler

                boolean keep = false;
                try {
                    keep = idler.queueIdle();
                } catch (Throwable t) {
                    Log.wtf("MessageQueue", "IdleHandler threw exception", t);
                }

                if (!keep) {
                    synchronized (this) {
                        mIdleHandlers.remove(idler);
                    }
                }
            }

            // Reset the idle handler count to 0 so we do not run them again.
            pendingIdleHandlerCount = 0;

            // While calling an idle handler, a new message could have been delivered
            // so go back and look again for a pending message without waiting.
            nextPollTimeoutMillis = 0;
        }
    }
```

``MessageQueue.next`` will be the first to call ``nativePollOnce``, then if ``mQuiting`` true returns null, ``Looper`` will exit the message loop.
　　
The next head of the news from the message queue, If the head message is ``Barrier`` (target==null) on the next traversal to find the first asynchronous message, The next test access to the message (message queue message or the first asynchronous message), If NULL said no message to perform, Set ``nextPollTimeoutMillis = -1``; otherwise detect this news to execution time., If the execution time of the markInUse and the message from the message queue to remove, And then returns from next to loop; otherwise, set ``nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE)``, That is the nearest to perform message also need how long, Both the current message queue no news can perform (set ``Barrier`` and no asynchronous message or the message queue is empty) or the head of the queue of the message not to execution time, Will execute the code behind the, To see if there is no set ``IdleHandler``, If there is to run ``IdleHandler``, When the ``IdleHandler`` is executing will set ``nextPollTimeoutMillis ＝ 0``.

First look at the ``nativePollOnce``, native method, called JNI, and finally transferred to the Native ``Looper:: pollOnce``, and from Java layer transfer in ``nextPollTimeMillis``, namely the execution time of Java layer in the message queue to recent news how long execution time.

MessageQueue.next首先会调用nativePollOnce，然后如果mQuiting为true就返回null，Looper就会退出消息循环。

接下来取消息队列头部的消息，如果头部消息是Barrier（target==null）就往后遍历找到第一个异步消息，接下来检测获取到的消息（消息队列头部的消息或者第一个异步消息），如果为null表示没有消息要执行，设置nextPollTimeoutMillis = -1；否则检测这个消息要执行的时间，如果到执行时间了就将这个消息markInUse并从消息队列移除，然后从next返回到loop；否则设置nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE)，即距离最近要执行的消息还需要多久，无论是当前消息队列没有消息可以执行（设置了Barrier并且没有异步消息或消息队列为空）还是队列头部的消息未到执行时间，都会执行后面的代码，看有没有设置IdleHandler，如果有就运行IdleHandler，当IdleHandler被执行之后会设置nextPollTimeoutMillis ＝ 0。

首先看一下nativePollOnce，native方法，调用JNI，最后调到了Native Looper::pollOnce，并从Java层传进去了nextPollTimeMillis，即Java层的消息队列中执行时间最近的消息还要多久到执行时间。

```c
int Looper::pollOnce(int timeoutMillis, int* outFd, int* outEvents, void** outData) {
    int result = 0;
    for (;;) {
        while (mResponseIndex < mResponses.size()) {
            const Response& response = mResponses.itemAt(mResponseIndex++);
            int ident = response.request.ident;
            if (ident >= 0) {
                int fd = response.request.fd;
                int events = response.events;
                void* data = response.request.data;
#if DEBUG_POLL_AND_WAKE
                ALOGD("%p ~ pollOnce - returning signalled identifier %d: "
                        "fd=%d, events=0x%x, data=%p",
                        this, ident, fd, events, data);
#endif
                if (outFd != NULL) *outFd = fd;
                if (outEvents != NULL) *outEvents = events;
                if (outData != NULL) *outData = data;
                return ident;
            }
        }

        if (result != 0) {
#if DEBUG_POLL_AND_WAKE
            ALOGD("%p ~ pollOnce - returning result %d", this, result);
#endif
            if (outFd != NULL) *outFd = 0;
            if (outEvents != NULL) *outEvents = 0;
            if (outData != NULL) *outData = NULL;
            return result;
        }

        result = pollInner(timeoutMillis);
    }
}
```

Don't see a bunch of code first, look at the ``pollInner``:

先不看开始的一大串代码，先看一下pollInner：

```c
int Looper::pollInner(int timeoutMillis) {
#if DEBUG_POLL_AND_WAKE
    ALOGD("%p ~ pollOnce - waiting: timeoutMillis=%d", this, timeoutMillis);
#endif

    // Adjust the timeout based on when the next message is due.
    if (timeoutMillis != 0 && mNextMessageUptime != LLONG_MAX) {
        nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
        int messageTimeoutMillis = toMillisecondTimeoutDelay(now, mNextMessageUptime);
        if (messageTimeoutMillis >= 0
                && (timeoutMillis <0 || messageTimeoutMillis < timeoutMillis)) {
            timeoutMillis = messageTimeoutMillis;
        }
#if DEBUG_POLL_AND_WAKE
        ALOGD("%p ~ pollOnce - next message in %lldns, adjusted timeout: timeoutMillis=%d",
                this, mNextMessageUptime - now, timeoutMillis);
#endif
    }

    // Poll.
    int result = ALOOPER_POLL_WAKE;
    mResponses.clear();
    mResponseIndex = 0;

    struct epoll_event eventItems[EPOLL_MAX_EVENTS];
    int eventCount = epoll_wait(mEpollFd, eventItems, EPOLL_MAX_EVENTS, timeoutMillis);

    // Acquire lock.
    mLock.lock();

    // Check for poll error.
    if (eventCount <0) {
        if (errno == EINTR) {
            goto Done;
        }
        ALOGW("Poll failed with an unexpected error, errno=%d", errno);
        result = ALOOPER_POLL_ERROR;
        goto Done;
    }

    // Check for poll timeout.
    if (eventCount == 0) {
#if DEBUG_POLL_AND_WAKE
        ALOGD("%p ~ pollOnce - timeout", this);
#endif
        result = ALOOPER_POLL_TIMEOUT;
        goto Done;
    }

    // Handle all events.
#if DEBUG_POLL_AND_WAKE
    ALOGD("%p ~ pollOnce - handling events from %d fds", this, eventCount);
#endif

    for (int i = 0; i <eventCount; i++) {
        int fd = eventItems[i].data.fd;
        uint32_t epollEvents = eventItems[i].events;
        if (fd == mWakeReadPipeFd) {
            if (epollEvents & EPOLLIN) {
                awoken();
            } else {
                ALOGW("Ignoring unexpected epoll events 0x%x on wake read pipe.", epollEvents);
            }
        } else {
            ssize_t requestIndex = mRequests.indexOfKey(fd);
            if (requestIndex >= 0) {
                int events = 0;
                if (epollEvents & EPOLLIN) events |= ALOOPER_EVENT_INPUT;
                if (epollEvents & EPOLLOUT) events |= ALOOPER_EVENT_OUTPUT;
                if (epollEvents & EPOLLERR) events |= ALOOPER_EVENT_ERROR;
                if (epollEvents & EPOLLHUP) events |= ALOOPER_EVENT_HANGUP;
                pushResponse(events, mRequests.valueAt(requestIndex));
            } else {
                ALOGW("Ignoring unexpected epoll events 0x%x on fd %d that is "
                        "no longer registered.", epollEvents, fd);
            }
        }
    }
Done: ;

    // Invoke pending message callbacks.
    mNextMessageUptime = LLONG_MAX;
    while (mMessageEnvelopes.size() != 0) {
        nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
        const MessageEnvelope& messageEnvelope = mMessageEnvelopes.itemAt(0);
        if (messageEnvelope.uptime <= now) {
            // Remove the envelope from the list.
            // We keep a strong reference to the handler until the call to handleMessage
            // finishes.  Then we drop it so that the handler can be deleted *before*
            // we reacquire our lock.
            { // obtain handler
                sp<MessageHandler> handler = messageEnvelope.handler;
                Message message = messageEnvelope.message;
                mMessageEnvelopes.removeAt(0);
                mSendingMessage = true;
                mLock.unlock();

#if DEBUG_POLL_AND_WAKE || DEBUG_CALLBACKS
                ALOGD("%p ~ pollOnce - sending message: handler=%p, what=%d",
                        this, handler.get(), message.what);
#endif
                handler->handleMessage(message);
            } // release handler

            mLock.lock();
            mSendingMessage = false;
            result = ALOOPER_POLL_CALLBACK;
        } else {
            // The last message left at the head of the queue determines the next wakeup time.
            mNextMessageUptime = messageEnvelope.uptime;
            break;
        }
    }

    // Release lock.
    mLock.unlock();

    // Invoke all response callbacks.
    for (size_t i = 0; i <mResponses.size(); i++) {
        Response& response = mResponses.editItemAt(i);
        if (response.request.ident == ALOOPER_POLL_CALLBACK) {
            int fd = response.request.fd;
            int events = response.events;
            void* data = response.request.data;
#if DEBUG_POLL_AND_WAKE || DEBUG_CALLBACKS
            ALOGD("%p ~ pollOnce - invoking fd event callback %p: fd=%d, events=0x%x, data=%p",
                    this, response.request.callback.get(), fd, events, data);
#endif
            int callbackResult = response.request.callback->handleEvent(fd, events, data);
            if (callbackResult == 0) {
                removeFd(fd);
            }
            // Clear the callback reference in the response structure promptly because we
            // will not clear the response vector itself until the next poll.
            response.request.callback.clear();
            result = ALOOPER_POLL_CALLBACK;
        }
    }
    return result;
}
```

Java messages are stored in the Java layer of the ``MessageQueue`` member of the ``mMessages``, Native messages are stored in the Native ``Looper`` ``mMessageEnvelopes``, it can be said that there are two message queue, and is arranged by time. TimeOutMillis said Java layer next to perform the news how long execution, ``mNextMessageUpdate`` said Native layer next to perform the news how long, if timeOutMillis is 0, ``epoll_wait`` does not set the TimeOut returned directly; if -1 Java layer without message directly with the Native time out; minimum or ``pollInner`` take the two one as the timeOut call ``epoll_wait``. When epoll_wait returns may have the following conditions:

+ Error return
+ Time out
+ The normal return, there are events generated descriptors.

If the former is directly goto DONE two. 

Otherwise, FD has happened, If the ``mWakeReadPipeFd`` EPOLLIN event awoken is called, If not ``mWakeReadPipeFd``, It is through the ``addFD`` add FD, In addFD, to FD and events., callback,The data package into the ``Request`` object, Taking FD as the key stored in the KeyedVector mRequests, So here in FD to obtain association in addFD Request, And along with the events by adding ``pushResonse`` ``mResonse queue(Vector)``, ``Resonse`` is only for events and ``Request`` package. If an error of ``epoll_wait`` or timeout, there is no event descriptors, not the implementation of this code, so the direct goto DONE.

Java层的消息都保存在了Java层MessageQueue的成员mMessages中，Native层的消息都保存在了Native Looper的mMessageEnvelopes中，这就可以说有两个消息队列，而且都是按时间排列的。timeOutMillis表示Java层下个要执行的消息还要多久执行，mNextMessageUpdate表示Native层下个要执行的消息还要多久执行，如果timeOutMillis为0，epoll_wait不设置TimeOut直接返回；如果为-1说明Java层无消息直接用Native的time out；否则pollInner取这两个中的最小值作为timeOut调用epoll_wait。当epoll_wait返回时就可能有以下几种情况：

+ 出错返回。

+ Time Out

+ 正常返回，描述符上有事件产生。

+ 如果是前两种情况直接goto DONE。

否则就说明FD上有事件发生了，如果是mWakeReadPipeFd的EPOLLIN事件就调用awoken，如果不是mWakeReadPipeFd，那就是通过addFD添加的fd，在addFD中将要监听的fd及其events，callback,data封装成了Request对象，并以fd为键保存到了KeyedVector mRequests中，所以在这里就以fd为键获得在addFD时关联的Request，并连同events通过pushResonse加入mResonse队列（Vector），Resonse仅是对events与Request的封装。如果是epoll_wait出错或timeout，就没有描述符上有事件，就不用执行这一段代码，所以直接goto DONE了。

```c
void Looper::pushResponse(int events, const Request& request) {
    Response response;
    response.events = events;
    response.request = request;
    mResponses.push(response);
}
```

Enter DONE next, remove Native message headers from ``mMessageEnvelopes``, if arrived at execution time ``handleMessage`` is called to handle it internally stored ``MessageeHandler`` and removed from the Native message queue, set result to ``ALOOPER_POLL_CALLBACK``, otherwise the calculation of ``mNextMessageUptime`` said Native message queue a message to execution time. If not to the head message execution time may be Java layer message queuing message execution time is less than Native layer message queue message execution time, arrived at the execution time of ``epoll_wait`` TimeOut Java message, or add by addFd descriptor has occurred in ``epoll_wait`` returns, or ``epoll_wait`` error return. The Native message is not ``Barrier`` and ``Asynchronous``.

Last, Through the ``mResponses`` (just by ``pushResponse`` put into it), If ``response.request.ident = = ALOOPER_POLL_CALLBACK``, We call the registration of callback ``handleEvent (FD, events, data)`` processing, And then removed from the ``mResonses`` queue, After the traversal., MResponses retention and are ident> =0 and callback NULL. In the NativeMessageQueue initialization ``Looper`` into ``mAllowNonCallbacks`` false, so the treatment after ``mResponses`` must be empty. 

Then return to ``pollOnce``. PollOnce is a for cycle, The ``pollInner`` handles all ``response.request.ident==ALOOPER_POLL_CALLBACK`` Response, In second into the for cycle if the ``mResponses`` is not empty to find ident> 0 Response, The ident as the value returned by the function call pollOnce's own processing, Here we are called in NativeMessageQueue Loope ``pollOnce``, No return processing, But ``mAllowNonCallbacks`` false is not likely to enter the cycle. The return value of ``pollInner`` may not be 0, or can only be negative, so ``pollOnce`` for cycle will only be executed two times, then returned in second times.Native Looper can be used alone, also has a prepare function, then the ``mAllowNonCallbakcs`` value may be true, ``pollOnce`` in ``mResponses`` makes sense.

接下来进入DONE部分，从mMessageEnvelopes取出头部的Native消息，如果到达了执行时间就调用它内部保存的MessageeHandler的handleMessage处理并从Native 消息队列移除，设置result为ALOOPER_POLL_CALLBACK，否则计算mNextMessageUptime表示Native消息队列下一次消息要执行的时间。如果未到头部消息的执行时间有可能是Java层消息队列消息的执行时间小于Native层消息队列头部消息的执行时间，到达了Java层消息的执行时间epoll_wait TimeOut返回了，或都通过addFd添加的描述符上有事件发生导致epoll_wait返回，或者epoll_wait是出错返回。Native消息是没有Barrier与Asynchronous的。

最后，遍历mResponses（前面刚通过pushResponse存进去的），如果response.request.ident == ALOOPER_POLL_CALLBACK，就调用注册的callback的handleEvent(fd, events, data)进行处理，然后从mResonses队列中移除，这次遍历完之后，mResponses中保留来来的就都是ident>=0并且callback为NULL的了。在NativeMessageQueue初始化Looper时传入了mAllowNonCallbacks为false，所以这次处理完后mResponses一定为空。

接下来返回到pollOnce。pollOnce是一个for循环，pollInner中处理了所有response.request.ident==ALOOPER_POLL_CALLBACK的Response，在第二次进入for循环后如果mResponses不为空就可以找到ident>0的Response，将其ident作为返回值返回由调用pollOnce的函数自己处理，在这里我们是在NativeMessageQueue中调用的Loope的pollOnce，没对返回值进行处理，而且mAllowNonCallbacks为false也就不可能进入这个循环。pollInner返回值不可能是0，或者说只可能是负数，所以pollOnce中的for循环只会执行两次，在第二次就返回了。

Native Looper可以单独使用，也有一个prepare函数，这时mAllowNonCallbakcs值可能为true，pollOnce中对mResponses的处理就有意义了。

<h2 id="wake-and-awoken">Wake and awoken</h2>

In the Native Looper constructor, The pipe opened a pipeline, Combined use of ``mWakeReadPipeFd`` and ``mWakeWritePipeFd`` were preserved the read and write end of pipe end, Then use the ``epoll_ctl (mEpollFd, EPOLL_CTL_ADD, mWakeReadPipeFd, & eventItem)`` monitored the read end of EPOLLIN events, In the pollInner by ``epoll_wait (mEpollFd, eventItems, EPOLL_MAX_EVENTS, timeoutMillis)`` read event, When writing to ``mWakeWritePipeFd``, And at what time to read ``mWakeReadPipeFd``?

In ``Looper.cpp`` we can be found below two functions:

在Native Looper的构造函数中，通过pipe打开了一个管道，并用mWakeReadPipeFd与mWakeWritePipeFd分别保存了管道的读端与写端，然后用epoll_ctl(mEpollFd, EPOLL_CTL_ADD, mWakeReadPipeFd,& eventItem)监听了读端的EPOLLIN事件，在pollInner中通过epoll_wait(mEpollFd, eventItems, EPOLL_MAX_EVENTS, timeoutMillis)读取事件，那是在什么时候往mWakeWritePipeFd写，又是在什么时候读的mWakeReadPipeFd呢？

在Looper.cpp中我们可以发现如下两个函数：

```c
void Looper::wake() {
#if DEBUG_POLL_AND_WAKE
    ALOGD("%p ~ wake", this);
#endif

    ssize_t nWrite;
    do {
        nWrite = write(mWakeWritePipeFd, "W", 1);
    } while (nWrite == -1 && errno == EINTR);

    if (nWrite != 1) {
        if (errno != EAGAIN) {
            ALOGW("Could not write wake signal, errno=%d", errno);
        }
    }
}

void Looper::awoken() {
#if DEBUG_POLL_AND_WAKE
    ALOGD("%p ~ awoken", this);
#endif

    char buffer[16];
    ssize_t nRead;
    do {
        nRead = read(mWakeReadPipeFd, buffer, sizeof(buffer));
    } while ((nRead == -1 && errno == EINTR) || nRead == sizeof(buffer));
}
```

The wake function to write a &ldquo to ``mWakeWritePipeFd`` ``W`` character, awoken read from ``mWakeReadPipeFd``, to ``mWakeWritePipeFd`` to write data to ``pollInner`` in the ``epoll_wait`` you can listen to the event returns. The ``pollInner`` can also see if the ``mWakeReadPipeFd`` EPOLLIN event is called the awoken consumed character written back processing.

When calling wake? Just find a place of call on the line, Look at the ``Looper.cpp``, In the ``sendMessageAtTime`` when sending Native Message, The calculation should be inserted according to the execution time for ``mMessageEnvelopes`` to send Message, If it is in the head is inserted into the, We call the wake wake up ``epoll_wait``, Because in the ``pollInner`` according to the execution time and the Native layer message queue head message execution time Java layer message queue header calculated a timeout, If the new message is inserted in the head, The execution time of at least the two messages in a before, So you should wake up ``epoll_wait``, Epoll_wait returns, Check the Native message queue, To see whether the head message is just inserted message to the execution time, To execute, Otherwise you may need to set the new timeout. Also in the Java layer in the MessageQueue, there is a function nativeWake also can be called by JNI wake, call the ``nativeWake`` timing and call in Native wake time similar, insert a message in the message queue, there is also a case, the message queue head is a ``Barrier``, and insert the message was the first message.

wake函数向mWakeWritePipeFd写入了一个“W”字符，awoken从mWakeReadPipeFd读，往mWakeWritePipeFd写数据只是为了在pollInner中的epoll_wait可以监听到事件返回。在pollInner也可以看到如果是mWakeReadPipeFd的EPOLLIN事件只是调用了awoken消耗掉了写入的字符就往后处理了。

那什么时候调用wake呢？这个只要找到调用的地方分析一下就行了，先看Looper.cpp，在sendMessageAtTime即发送Native Message的时候，根据发送的Message的执行时间查找mMessageEnvelopes计算应该插入的位置，如果是在头部插入，就调用wake唤醒epoll_wait，因为在进入pollInner时根据Java层消息队列头部消息的执行时间与Native层消息队列头部消息的执行时间计算出了一个timeout，如果这个新消息是在头部插入，说明执行时间至少在上述两个消息中的一个之前，所以应该唤醒epoll_wait，epoll_wait返回后，检查Native消息队列，看头部消息即刚插入的消息是否到执行时间了，到了就执行，否则就可能需要设置新的timeout。同样在Java层的MessageQueue中，有一个函数nativeWake也同样可以通过JNI调用wake，调用nativeWake的时机与在Native调用wake的时机类似，在消息队列头部插入消息，还有一种情况就是，消息队列头部是一个Barrier，而且插入的消息是第一个异步消息。

```c
if (p == null || when == 0 || when < p.when) {
    // New head, wake up the event queue if blocked.
    msg.next = p;
    mMessages = msg;
    needWake = mBlocked;
} else {
    // Inserted within the middle of the queue.  Usually we don't have to wake
    // up the event queue unless there is a barrier at the head of the queue
    // and the message is the earliest asynchronous message in the queue.
    needWake = mBlocked && p.target == null && msg.isAsynchronous();//If the head is Barrier and the new messages are asynchronous message “ &rdquo may need to wake up;
    Message prev;
    for (;;) {
        prev = p;
        p = p.next;
        if (p == null || when < p.when) {
            break;
        }
        if (needWake && p.isAsynchronous()) { // The message queue is asynchronous message and the execution time in the news before, so there is no need to wake up. 
            needWake = false;
        }
    }
    msg.next = p; // invariant: p == prev.next
    prev.next = msg;
}
```

In the head is inserted into the news does not have to call nativeWake, Because before may be performing ``IdleHandler``, If the implementation of the ``IdleHandler``, In the ``IdleHandler`` implementation of the ``nextPollTimeoutMillis`` is set to 0, Next time you start for cycle with the 0 call to ``nativePollOnce``, Don't need wake, Only when no message can perform (the message queue is empty or not to the execution time) and not set ``IdleHandler`` ``mBlocked`` to true.

If the Java layer of the message queue is ``Barrier`` Block and the insertion is an asynchronous message may need to wake up ``Looper``, because the asynchronous message can be implemented in ``Barrier``, but the asynchronous message must be the first time the implementation of asynchronous message.

Exit the ``Looper`` also need wake, ``removeSyncBarrier`` may also require. 

在头部插入消息不一定调用nativeWake，因为之前可能正在执行IdleHandler，如果执行了IdleHandler，就在IdleHandler执行后把nextPollTimeoutMillis设置为0，下次进入for循环就用0调用nativePollOnce，不需要wake，只有在没有消息可以执行（消息队列为空或没到执行时间）并且没有设置IdleHandler时mBlocked才会为true。

如果Java层的消息队列被Barrier Block住了并且当前插入的是一个异步消息有可能需要唤醒Looper，因为异步消息可以在Barrier下执行，但是这个异步消息一定要是执行时间最早的异步消息。

退出Looper也需要wake，removeSyncBarrier时也可能需要。

+ [eng](2015-09-22-android-message-handling-mechanism.markdown)
+ [zh_cn](http://www.cnblogs.com/angeldevil/p/3340644.html)