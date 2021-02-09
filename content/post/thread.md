+++
title = "Thread"
date = "2019-05-28"
slug = "2019/05/28/thread"
Categories = ["java", "concurrency"]
+++

本文主要分析Android平台上的Thread类源码，分为Java部分和native部分。

<!-- more -->

Java部分比较简单，大致过下各个方法吧。

### 线程创建

```java
	/**
     * Initializes a Thread.
     *
     * @param g the Thread group
     * @param target the object whose run() method gets called
     * @param name the name of the new Thread
     * @param stackSize the desired stack size for the new thread, or
     *        zero to indicate that this parameter is to be ignored.
     */
    private void init(ThreadGroup g, Runnable target, String name, long stackSize) {
        Thread parent = currentThread();
        if (g == null) {
            g = parent.getThreadGroup();
        }

        g.addUnstarted();
        this.group = g;

        this.target = target;
        this.priority = parent.getPriority();
        this.daemon = parent.isDaemon();
        setName(name);

        init2(parent);

        /* Stash the specified stack size in case the VM cares */
        this.stackSize = stackSize;
        tid = nextThreadID();
    }
	private void init2(Thread parent) {
        this.contextClassLoader = parent.getContextClassLoader();
        this.inheritedAccessControlContext = AccessController.getContext();
        if (parent.inheritableThreadLocals != null) {
            this.inheritableThreadLocals = ThreadLocal.createInheritedMap(
                    parent.inheritableThreadLocals);
        }
    }
```

线程``NEW``出来的时候只是从创建它的线程那里继承一些属性，比如threadgroup，daemon状态，priority，stacksize，inheritableThreadLocals之类的。真正启动线程是在``start``方法里。

### 线程启动

```java
/**
     * Causes this thread to begin execution; the Java Virtual Machine
     * calls the <code>run</code> method of this thread.
     * <p>
     * The result is that two threads are running concurrently: the
     * current thread (which returns from the call to the
     * <code>start</code> method) and the other thread (which executes its
     * <code>run</code> method).
     * <p>
     * It is never legal to start a thread more than once.
     * In particular, a thread may not be restarted once it has completed
     * execution.
     *
     * @exception  IllegalThreadStateException  if the thread was already
     *               started.
     * @see        #run()
     * @see        #stop()
     */
    public synchronized void start() {
        /**
         * This method is not invoked for the main method thread or "system"
         * group threads created/set up by the VM. Any new functionality added
         * to this method in the future may have to also be added to the VM.
         *
         * A zero status value corresponds to state "NEW".
         */
        // Android-changed: throw if 'started' is true
        if (threadStatus != 0 || started)
            throw new IllegalThreadStateException();

        /* Notify the group that this thread is about to be started
         * so that it can be added to the group's list of threads
         * and the group's unstarted count can be decremented. */
        group.add(this);

        started = false;
        try {
            nativeCreate(this, stackSize, daemon);
            started = true;
        } finally {
            try {
                if (!started) {
                    group.threadStartFailed(this);
                }
            } catch (Throwable ignore) {
                /* do nothing. If start0 threw a Throwable then
                  it will be passed up the call stack */
            }
        }
    }

    private native static void nativeCreate(Thread t, long stackSize, boolean daemon);
```

```c++
// http://androidxref.com/9.0.0_r3/xref/art/runtime/thread.cc#623
void Thread::CreateNativeThread(JNIEnv* env, jobject java_peer, size_t stack_size, bool is_daemon) {
  CHECK(java_peer != nullptr);
  Thread* self = static_cast<JNIEnvExt*>(env)->GetSelf();

  if (VLOG_IS_ON(threads)) {
    ScopedObjectAccess soa(env);

    ArtField* f = jni::DecodeArtField(WellKnownClasses::java_lang_Thread_name);
    ObjPtr<mirror::String> java_name =
        f->GetObject(soa.Decode<mirror::Object>(java_peer))->AsString();
    std::string thread_name;
    if (java_name != nullptr) {
      thread_name = java_name->ToModifiedUtf8();
    } else {
      thread_name = "(Unnamed)";
    }

    VLOG(threads) << "Creating native thread for " << thread_name;
    self->Dump(LOG_STREAM(INFO));
  }

  Runtime* runtime = Runtime::Current();

  // Atomically start the birth of the thread ensuring the runtime isn't shutting down.
  bool thread_start_during_shutdown = false;
  {
    MutexLock mu(self, *Locks::runtime_shutdown_lock_);
    if (runtime->IsShuttingDownLocked()) {
      thread_start_during_shutdown = true;
    } else {
      runtime->StartThreadBirth();
    }
  }
  if (thread_start_during_shutdown) {
    ScopedLocalRef<jclass> error_class(env, env->FindClass("java/lang/InternalError"));
    env->ThrowNew(error_class.get(), "Thread starting during runtime shutdown");
    return;
  }

  Thread* child_thread = new Thread(is_daemon);
  // Use global JNI ref to hold peer live while child thread starts.
  child_thread->tlsPtr_.jpeer = env->NewGlobalRef(java_peer);
  stack_size = FixStackSize(stack_size);

  // Thread.start is synchronized, so we know that nativePeer is 0, and know that we're not racing
  // to assign it.
  env->SetLongField(java_peer, WellKnownClasses::java_lang_Thread_nativePeer,
                    reinterpret_cast<jlong>(child_thread));

  // Try to allocate a JNIEnvExt for the thread. We do this here as we might be out of memory and
  // do not have a good way to report this on the child's side.
  std::string error_msg;
  std::unique_ptr<JNIEnvExt> child_jni_env_ext(
      JNIEnvExt::Create(child_thread, Runtime::Current()->GetJavaVM(), &error_msg));

  int pthread_create_result = 0;
  if (child_jni_env_ext.get() != nullptr) {
    pthread_t new_pthread;
    pthread_attr_t attr;
    child_thread->tlsPtr_.tmp_jni_env = child_jni_env_ext.get();
    CHECK_PTHREAD_CALL(pthread_attr_init, (&attr), "new thread");
    CHECK_PTHREAD_CALL(pthread_attr_setdetachstate, (&attr, PTHREAD_CREATE_DETACHED),
                       "PTHREAD_CREATE_DETACHED");
    CHECK_PTHREAD_CALL(pthread_attr_setstacksize, (&attr, stack_size), stack_size);
    pthread_create_result = pthread_create(&new_pthread,
                                           &attr,
                                           Thread::CreateCallback,
                                           child_thread);
    CHECK_PTHREAD_CALL(pthread_attr_destroy, (&attr), "new thread");

    if (pthread_create_result == 0) {
      // pthread_create started the new thread. The child is now responsible for managing the
      // JNIEnvExt we created.
      // Note: we can't check for tmp_jni_env == nullptr, as that would require synchronization
      //       between the threads.
      child_jni_env_ext.release();
      return;
    }
  }

  // Either JNIEnvExt::Create or pthread_create(3) failed, so clean up.
  {
    MutexLock mu(self, *Locks::runtime_shutdown_lock_);
    runtime->EndThreadBirth();
  }
  // Manually delete the global reference since Thread::Init will not have been run.
  env->DeleteGlobalRef(child_thread->tlsPtr_.jpeer);
  child_thread->tlsPtr_.jpeer = nullptr;
  delete child_thread;
  child_thread = nullptr;
  // TODO: remove from thread group?
  env->SetLongField(java_peer, WellKnownClasses::java_lang_Thread_nativePeer, 0);
  {
    std::string msg(child_jni_env_ext.get() == nullptr ?
        StringPrintf("Could not allocate JNI Env: %s", error_msg.c_str()) :
        StringPrintf("pthread_create (%s stack) failed: %s",
                                 PrettySize(stack_size).c_str(), strerror(pthread_create_result)));
    ScopedObjectAccess soa(env);
    soa.Self()->ThrowOutOfMemoryError(msg.c_str());
  }
}
```

一大堆代码只需要记住进行了``pthread_create``系统调用即可。

### 线程状态

```java
/**
     * A thread state.  A thread can be in one of the following states:
     * <ul>
     * <li>{@link #NEW}<br>
     *     A thread that has not yet started is in this state.
     *     </li>
     * <li>{@link #RUNNABLE}<br>
     *     A thread executing in the Java virtual machine is in this state.
     *     </li>
     * <li>{@link #BLOCKED}<br>
     *     A thread that is blocked waiting for a monitor lock
     *     is in this state.
     *     </li>
     * <li>{@link #WAITING}<br>
     *     A thread that is waiting indefinitely for another thread to
     *     perform a particular action is in this state.
     *     </li>
     * <li>{@link #TIMED_WAITING}<br>
     *     A thread that is waiting for another thread to perform an action
     *     for up to a specified waiting time is in this state.
     *     </li>
     * <li>{@link #TERMINATED}<br>
     *     A thread that has exited is in this state.
     *     </li>
     * </ul>
     *
     * <p>
     * A thread can be in only one state at a given point in time.
     * These states are virtual machine states which do not reflect
     * any operating system thread states.
     *
     * @since   1.5
     * @see #getState
     */
    public enum State {
        /**
         * Thread state for a thread which has not yet started.
         */
        NEW,

        /**
         * Thread state for a runnable thread.  A thread in the runnable
         * state is executing in the Java virtual machine but it may
         * be waiting for other resources from the operating system
         * such as processor.
         */
        RUNNABLE,

        /**
         * Thread state for a thread blocked waiting for a monitor lock.
         * A thread in the blocked state is waiting for a monitor lock
         * to enter a synchronized block/method or
         * reenter a synchronized block/method after calling
         * {@link Object#wait() Object.wait}.
         */
        BLOCKED,

        /**
         * Thread state for a waiting thread.
         * A thread is in the waiting state due to calling one of the
         * following methods:
         * <ul>
         *   <li>{@link Object#wait() Object.wait} with no timeout</li>
         *   <li>{@link #join() Thread.join} with no timeout</li>
         *   <li>{@link LockSupport#park() LockSupport.park}</li>
         * </ul>
         *
         * <p>A thread in the waiting state is waiting for another thread to
         * perform a particular action.
         *
         * For example, a thread that has called <tt>Object.wait()</tt>
         * on an object is waiting for another thread to call
         * <tt>Object.notify()</tt> or <tt>Object.notifyAll()</tt> on
         * that object. A thread that has called <tt>Thread.join()</tt>
         * is waiting for a specified thread to terminate.
         */
        WAITING,

        /**
         * Thread state for a waiting thread with a specified waiting time.
         * A thread is in the timed waiting state due to calling one of
         * the following methods with a specified positive waiting time:
         * <ul>
         *   <li>{@link #sleep Thread.sleep}</li>
         *   <li>{@link Object#wait(long) Object.wait} with timeout</li>
         *   <li>{@link #join(long) Thread.join} with timeout</li>
         *   <li>{@link LockSupport#parkNanos LockSupport.parkNanos}</li>
         *   <li>{@link LockSupport#parkUntil LockSupport.parkUntil}</li>
         * </ul>
         */
        TIMED_WAITING,

        /**
         * Thread state for a terminated thread.
         * The thread has completed execution.
         */
        TERMINATED;
    }
```

线程有6种状态，``NEW``表示线程新建，还没调用``start()``。``RUNNABLE``表示线程调用了``start``处于可运行状态，等待CPU调度。``BLOCKED``表示线程在等待获取monitor lock。当在线程中调用``Object.wait()``或``Thread.join()``或``LockSupport.park()``方法时线程进入``WAITING``状态，在线程中调用``Thread.sleep()``或``Object.wait(long)``或``Thread.join(long)``或``LockSupport.parkNanos(long)``或``LockSupport.parkUntil(long)``线程进入``TIMED_WAITING``状态。线程执行完操作后会进入``TERMINATED``状态。

### 线程中断

线程中一共有3个和中断有关的方法，分别是``interrupt()``，``isInterrupted()``和静态方法``interrupted``。我们分别看一下。

#### interrupt

```java
/**
     * Interrupts this thread.
     *
     * <p> Unless the current thread is interrupting itself, which is
     * always permitted, the {@link #checkAccess() checkAccess} method
     * of this thread is invoked, which may cause a {@link
     * SecurityException} to be thrown.
     *
     * <p> If this thread is blocked in an invocation of the {@link
     * Object#wait() wait()}, {@link Object#wait(long) wait(long)}, or {@link
     * Object#wait(long, int) wait(long, int)} methods of the {@link Object}
     * class, or of the {@link #join()}, {@link #join(long)}, {@link
     * #join(long, int)}, {@link #sleep(long)}, or {@link #sleep(long, int)},
     * methods of this class, then its interrupt status will be cleared and it
     * will receive an {@link InterruptedException}.
     *
     * <p> If this thread is blocked in an I/O operation upon an {@link
     * java.nio.channels.InterruptibleChannel InterruptibleChannel}
     * then the channel will be closed, the thread's interrupt
     * status will be set, and the thread will receive a {@link
     * java.nio.channels.ClosedByInterruptException}.
     *
     * <p> If this thread is blocked in a {@link java.nio.channels.Selector}
     * then the thread's interrupt status will be set and it will return
     * immediately from the selection operation, possibly with a non-zero
     * value, just as if the selector's {@link
     * java.nio.channels.Selector#wakeup wakeup} method were invoked.
     *
     * <p> If none of the previous conditions hold then this thread's interrupt
     * status will be set. </p>
     *
     * <p> Interrupting a thread that is not alive need not have any effect.
     *
     * @throws  SecurityException
     *          if the current thread cannot modify this thread
     *
     * @revised 6.0
     * @spec JSR-51
     */
    public void interrupt() {
        if (this != Thread.currentThread())
            checkAccess();

        synchronized (blockerLock) {
            Interruptible b = blocker;
            if (b != null) {
                nativeInterrupt();
                b.interrupt(this);
                return;
            }
        }
        nativeInterrupt();
    }
	@FastNative
    private native void nativeInterrupt();
```

从注释我们可以看出调用Thread的``interrupt``方法，分为几种情况

+ 如果线程阻塞在``Object.wait()``、``Object.wait(long)``、``Object.wait(long,int)``、``Thread.join()``、``Thread.join(long)``、``Thread.join(long,int)``、``Thread.sleep(long)``、``Thread.sleep(long,int)``这些方法上，线程的中断状态将被清除，并抛出``InterruptedException``。
+ 如果阻塞在``java.nio.channels.InterruptibleChannel``IO操作操作上，线程的中断状态将被设置（这里有个疑惑，这里设置是指设置为true还是false，如果设置为false就表示线程状态被清除了，由于对NIO了解不多，这里暂时无法确定，从注释上下文理解应该是设置为true了）并抛出``java.nio.channels.ClosedByInterruptException``
+ 如果阻塞在``java.nio.channels.Selector``上，线程的中断状态将被设置，并从``Selector``操作中返回。
+ 如果是其他情况，比如正在执行不会响应``interrupt``方法的方法（如``Socket``的读写或``ServerSocket``的``accept``），那么线程的中断状态将被设置。

继续看``nativeInterrupt``方法，从[java_lang_thread.cc](<http://androidxref.com/9.0.0_r3/xref/art/runtime/native/java_lang_Thread.cc>)中[Thread_nativeInterupt](<http://androidxref.com/9.0.0_r3/xref/art/runtime/native/java_lang_Thread.cc#126>)可以看到是调用了[Thread::Interrupt](<http://androidxref.com/9.0.0_r3/xref/art/runtime/thread.cc#2416>)。

```c++
// http://androidxref.com/9.0.0_r3/xref/art/runtime/thread.cc#2416
void Thread::Interrupt(Thread* self) {
  MutexLock mu(self, *wait_mutex_);
  if (tls32_.interrupted.LoadSequentiallyConsistent()) {
    return;
  }
  tls32_.interrupted.StoreSequentiallyConsistent(true);
  NotifyLocked(self);
}
```

可以看到只是设置了一下中断状态。

#### isInterrupted 和 interrupted

```java
	/**
     * Tests whether this thread has been interrupted.  The <i>interrupted
     * status</i> of the thread is unaffected by this method.
     *
     * <p>A thread interruption ignored because a thread was not alive
     * at the time of the interrupt will be reflected by this method
     * returning false.
     *
     * @return  <code>true</code> if this thread has been interrupted;
     *          <code>false</code> otherwise.
     * @see     #interrupted()
     * @revised 6.0
     */
    @FastNative
    public native boolean isInterrupted(); // 实例方法，返回中断状态
	
	/**
     * Tests whether the current thread has been interrupted.  The
     * <i>interrupted status</i> of the thread is cleared by this method.  In
     * other words, if this method were to be called twice in succession, the
     * second call would return false (unless the current thread were
     * interrupted again, after the first call had cleared its interrupted
     * status and before the second call had examined it).
     *
     * <p>A thread interruption ignored because a thread was not alive
     * at the time of the interrupt will be reflected by this method
     * returning false.
     *
     * @return  <code>true</code> if the current thread has been interrupted;
     *          <code>false</code> otherwise.
     * @see #isInterrupted()
     * @revised 6.0
     */
    @FastNative
    public static native boolean interrupted();
	
```

实例方法``isInterrupted``会返回线程的中断状态，静态方法``interrupted``会返回调用这个方法的线程的中断状态，并清除线程的中断状态，这也是清除中断状态的唯一方法，设置中断状态当然调用``interrupt``方法就好了。

不妨看下native层实现：

```c++
// http://androidxref.com/9.0.0_r3/xref/art/runtime/thread.cc
// Implements java.lang.Thread.interrupted.
bool Thread::Interrupted() {
  DCHECK_EQ(Thread::Current(), this);
  // No other thread can concurrently reset the interrupted flag.
  bool interrupted = tls32_.interrupted.LoadSequentiallyConsistent();
  if (interrupted) {
    tls32_.interrupted.StoreSequentiallyConsistent(false);
  }
  return interrupted;
}

// Implements java.lang.Thread.isInterrupted.
bool Thread::IsInterrupted() {
  return tls32_.interrupted.LoadSequentiallyConsistent();
}
```

### 守护线程

线程可分为两种，普通线程和守护线程（其实叫服务线程好理解些，这些线程一般是服务其他线程的）。在JVM启动时创建的所有线程中，除了主线程以外，其他的线程都是守护线程（例如垃圾回收器以及执行其他辅助工作的线程，signal dispatcher之类的）。

当创建一个新线程时，新线程将继承创建它的线程的守护状态。

普通线程和守护线程之间的差异仅在于当线程退出时发生的操作。当一个线程退出时，JVM会检查其他正在运行的线程，如果这些线程都是守护线程，那么JVM会正常退出操作。当JVM停止时，所有仍然存在的守护线程都将被抛弃——既不会执行finally代码块，也不会执行回卷栈，而JVM只是直接退出。

### reference

+ 《Java并发编程实战》
+ [Android平台上除了Java线程还有native线程，可以参考这篇](<http://gityuan.com/2016/09/24/android-thread/>)
