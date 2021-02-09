+++
title = "Futuretask"
date = "2019-06-28"
slug = "2019/06/28/futuretask"
Categories = ["java", "concurrency"]
+++

本文主要对FutureTask源码进行分析。

<!-- more -->

Java中一般通过继承Thread类或实现Runnable接口来创建线程，但是这两种方式都有个缺陷，就是不能在线程执行完成后获取执行的结果，因此Java1.5之后提供了``Callable``和``Future``接口，通过他们就可以在任务执行完成之后获取到任务的执行结果。

### Callable接口

```java
/**
 * A task that returns a result and may throw an exception.
 * Implementors define a single method with no arguments called
 * {@code call}.
 *
 * <p>The {@code Callable} interface is similar to {@link
 * java.lang.Runnable}, in that both are designed for classes whose
 * instances are potentially executed by another thread.  A
 * {@code Runnable}, however, does not return a result and cannot
 * throw a checked exception.
 *
 * <p>The {@link Executors} class contains utility methods to
 * convert from other common forms to {@code Callable} classes.
 *
 * @see Executor
 * @since 1.5
 * @author Doug Lea
 * @param <V> the result type of method {@code call}
 */
@FunctionalInterface
public interface Callable<V> {
    /**
     * Computes a result, or throws an exception if unable to do so.
     *
     * @return computed result
     * @throws Exception if unable to compute a result
     */
    V call() throws Exception;
}
```

可以看到``Callable``是个泛型接口，泛型V代表返回值的类型，执行任务过程中如果有异常会抛出异常。

### Future接口

```java
/**
 * A {@code Future} represents the result of an asynchronous
 * computation.  Methods are provided to check if the computation is
 * complete, to wait for its completion, and to retrieve the result of
 * the computation.  The result can only be retrieved using method
 * {@code get} when the computation has completed, blocking if
 * necessary until it is ready.  Cancellation is performed by the
 * {@code cancel} method.  Additional methods are provided to
 * determine if the task completed normally or was cancelled. Once a
 * computation has completed, the computation cannot be cancelled.
 * If you would like to use a {@code Future} for the sake
 * of cancellability but not provide a usable result, you can
 * declare types of the form {@code Future<?>} and
 * return {@code null} as a result of the underlying task.
 *
 * <p>
 * <b>Sample Usage</b> (Note that the following classes are all
 * made-up.)
 *
 * <pre> {@code
 * interface ArchiveSearcher { String search(String target); }
 * class App {
 *   ExecutorService executor = ...
 *   ArchiveSearcher searcher = ...
 *   void showSearch(final String target)
 *       throws InterruptedException {
 *     Future<String> future
 *       = executor.submit(new Callable<String>() {
 *         public String call() {
 *             return searcher.search(target);
 *         }});
 *     displayOtherThings(); // do other things while searching
 *     try {
 *       displayText(future.get()); // use future
 *     } catch (ExecutionException ex) { cleanup(); return; }
 *   }
 * }}</pre>
 *
 * The {@link FutureTask} class is an implementation of {@code Future} that
 * implements {@code Runnable}, and so may be executed by an {@code Executor}.
 * For example, the above construction with {@code submit} could be replaced by:
 * <pre> {@code
 * FutureTask<String> future =
 *   new FutureTask<>(new Callable<String>() {
 *     public String call() {
 *       return searcher.search(target);
 *   }});
 * executor.execute(future);}</pre>
 *
 * <p>Memory consistency effects: Actions taken by the asynchronous computation
 * <a href="package-summary.html#MemoryVisibility"> <i>happen-before</i></a>
 * actions following the corresponding {@code Future.get()} in another thread.
 *
 * @see FutureTask
 * @see Executor
 * @since 1.5
 * @author Doug Lea
 * @param <V> The result type returned by this Future's {@code get} method
 */
public interface Future<V> {

    /**
     * Attempts to cancel execution of this task.  This attempt will
     * fail if the task has already completed, has already been cancelled,
     * or could not be cancelled for some other reason. If successful,
     * and this task has not started when {@code cancel} is called,
     * this task should never run.  If the task has already started,
     * then the {@code mayInterruptIfRunning} parameter determines
     * whether the thread executing this task should be interrupted in
     * an attempt to stop the task.
     *
     * <p>After this method returns, subsequent calls to {@link #isDone} will
     * always return {@code true}.  Subsequent calls to {@link #isCancelled}
     * will always return {@code true} if this method returned {@code true}.
     *
     * @param mayInterruptIfRunning {@code true} if the thread executing this
     * task should be interrupted; otherwise, in-progress tasks are allowed
     * to complete
     * @return {@code false} if the task could not be cancelled,
     * typically because it has already completed normally;
     * {@code true} otherwise
     */
    boolean cancel(boolean mayInterruptIfRunning);

    /**
     * Returns {@code true} if this task was cancelled before it completed
     * normally.
     *
     * @return {@code true} if this task was cancelled before it completed
     */
    boolean isCancelled();

    /**
     * Returns {@code true} if this task completed.
     *
     * Completion may be due to normal termination, an exception, or
     * cancellation -- in all of these cases, this method will return
     * {@code true}.
     *
     * @return {@code true} if this task completed
     */
    boolean isDone();

    /**
     * Waits if necessary for the computation to complete, and then
     * retrieves its result.
     *
     * @return the computed result
     * @throws CancellationException if the computation was cancelled
     * @throws ExecutionException if the computation threw an
     * exception
     * @throws InterruptedException if the current thread was interrupted
     * while waiting
     */
    V get() throws InterruptedException, ExecutionException;

    /**
     * Waits if necessary for at most the given time for the computation
     * to complete, and then retrieves its result, if available.
     *
     * @param timeout the maximum time to wait
     * @param unit the time unit of the timeout argument
     * @return the computed result
     * @throws CancellationException if the computation was cancelled
     * @throws ExecutionException if the computation threw an
     * exception
     * @throws InterruptedException if the current thread was interrupted
     * while waiting
     * @throws TimeoutException if the wait timed out
     */
    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```

``Future``代表任务异步执行的结果，通过``Future``接口可以查询任务执行的状态，取消任务执行，获取任务执行的结果。

+ cancel方法尝试去取消任务的执行，如果任务已经完成，或者已经取消，或者由于其他原因无法被取消，cancel方法将返回false。如果cancel调用的时候，任务还没开始执行，任务将不被执行，cancel返回true。如果cancel调用的时候，任务已经开始执行，``mayInterruptIfRunning``决定执行任务的线程是否应该被中断来执行任务的执行。cancel方法调用返回true后，后续的``isDone``方法总是返回true，后续的``isCancelled``方法总是返回true。
+ 如果在任务完成之前被取消了，``isCancelled``方法会返回true。
+ 任务完成后，``isDone``方法会返回true。无论是正常的结束，抛出异常结束，被取消，``isDone``都会返回true。
+ get方法会等待任务执行结束来获取任务执行的结果，如果任务已经执行结束，直接返回结果。可能抛出``CancellationException``如果任务被取消，``ExecutionException``如果任务执行过程中抛出了异常，``InterruptedException``如果当前线程在等待执行任务的线程的执行结果的过程中被中断了。
+ get(long,TimeUnit)会等待最多设定的时间来获取结果。可能抛出``CancellationException``如果任务被取消，``ExecutionException``如果任务执行过程中抛出了异常，``InterruptedException``如果当前线程在等待执行任务的线程的执行结果的过程中被中断了，``TimeoutException``如果等待超时了。

### FutureTask

``Future``只是一个接口，``FutureTask``是``Future``的实现类。确切的说``FutureTask``实现了``RunnableFuture``接口，``RunnableFuture``接口扩展了``Runnable``和``Future``接口。

#### FutureTask任务执行的状态

```java
	/**
     * The run state of this task, initially NEW.  The run state
     * transitions to a terminal state only in methods set,
     * setException, and cancel.  During completion, state may take on
     * transient values of COMPLETING (while outcome is being set) or
     * INTERRUPTING (only while interrupting the runner to satisfy a
     * cancel(true)). Transitions from these intermediate to final
     * states use cheaper ordered/lazy writes because values are unique
     * and cannot be further modified.
     *
     * Possible state transitions:
     * NEW -> COMPLETING -> NORMAL
     * NEW -> COMPLETING -> EXCEPTIONAL
     * NEW -> CANCELLED
     * NEW -> INTERRUPTING -> INTERRUPTED
     */
    private volatile int state;
    private static final int NEW          = 0;
    private static final int COMPLETING   = 1;
    private static final int NORMAL       = 2;
    private static final int EXCEPTIONAL  = 3;
    private static final int CANCELLED    = 4;
    private static final int INTERRUPTING = 5;
    private static final int INTERRUPTED  = 6;
```

可以看到``FutureTask``使用``volatile``变量``state``来表示任务执行的状态，初始时是``NEW``，在``set``，``setException``和``cancel``方法中会对``state``进行赋值。可能的状态转换有``NEW -> COMPLETING -> NORMAL``，``NEW -> COMPLETING -> EXCEPTIONAL``，``NEW -> CANCELLED``，``NEW -> INTERRUPTING -> INTERRUPTED``。

#### FutureTask内部变量

```java
	/** The underlying callable; nulled out after running */
    private Callable<V> callable;
    /** The result to return or exception to throw from get() */
    private Object outcome; // non-volatile, protected by state reads/writes
    /** The thread running the callable; CASed during run() */
    private volatile Thread runner;
    /** Treiber stack of waiting threads */
    private volatile WaitNode waiters;
```

``callable``表示将要执行的任务，``volatile``类型的变量``runner``表示执行任务的线程，``waiters``表示等待任务执行结果的线程队列。任务执行的结果用Object类型的``outcome``表示，可以看到并没有用``volatile``关键字来修饰，那不会有可见性问题吗？这个问题后面我们会提到。

#### FutureTask任务执行run

```java
public void run() {
        if (state != NEW ||
            !U.compareAndSwapObject(this, RUNNER, null, Thread.currentThread()))
            return;
        try {
            Callable<V> c = callable;
            if (c != null && state == NEW) {
                V result;
                boolean ran;
                try {
                    result = c.call();
                    ran = true;
                } catch (Throwable ex) {
                    result = null;
                    ran = false;
                    setException(ex); // 任务执行抛出异常走setException
                }
                if (ran)
                    set(result); // 任务正常执行完成走set
            }
        } finally {
            // runner must be non-null until state is settled to
            // prevent concurrent calls to run()
            runner = null;
            // state must be re-read after nulling runner to prevent
            // leaked interrupts
            int s = state;
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
    }
```

``run``方法首先对状态进行判断，并尝试CAS替换``runner``为当前线程，如果失败，表明已经有线程在执行任务了，直接返回，否则执行任务，如果任务执行过程中抛出了异常，走``setException``分支，如果任务正常结束，走``set``分支，下面看下这两个方法。

#### FutureTask的set和setException方法

```java
	/**
     * Sets the result of this future to the given value unless
     * this future has already been set or has been cancelled.
     *
     * <p>This method is invoked internally by the {@link #run} method
     * upon successful completion of the computation.
     *
     * @param v the value
     */
    protected void set(V v) {
        if (U.compareAndSwapInt(this, STATE, NEW, COMPLETING)) {
            outcome = v;
            U.putOrderedInt(this, STATE, NORMAL); // final state
            finishCompletion();
        }
    }
	/**
     * Removes and signals all waiting threads, invokes done(), and
     * nulls out callable.
     */
    private void finishCompletion() {
        // assert state > COMPLETING;
        for (WaitNode q; (q = waiters) != null;) {
            if (U.compareAndSwapObject(this, WAITERS, q, null)) {
                for (;;) {
                    Thread t = q.thread;
                    if (t != null) {
                        q.thread = null;
                        LockSupport.unpark(t);
                    }
                    WaitNode next = q.next;
                    if (next == null)
                        break;
                    q.next = null; // unlink to help gc
                    q = next;
                }
                break;
            }
        }

        done();

        callable = null;        // to reduce footprint
    }
	/**
     * Causes this future to report an {@link ExecutionException}
     * with the given throwable as its cause, unless this future has
     * already been set or has been cancelled.
     *
     * <p>This method is invoked internally by the {@link #run} method
     * upon failure of the computation.
     *
     * @param t the cause of failure
     */
    protected void setException(Throwable t) {
        if (U.compareAndSwapInt(this, STATE, NEW, COMPLETING)) {
            outcome = t;
            U.putOrderedInt(this, STATE, EXCEPTIONAL); // final state
            finishCompletion();
        }
    }
```

可以看到``set``和``setException``方法都是CAS对``state``进行赋值，并对``outcome``进行赋值，同时调用``finishCompletion``方法唤醒等待队列中的线程去获取任务执行的结果。在这里可以看到对``outcome``的写发生在对``volatile``变量``state``写之前，因此保证了``state``为``NORMAL``或``EXCEPTIONAL``时``outcome``变量的可见性。在``finishCompletion``中可以看到是对等待队列上的线程进行唤醒操作，那么这些线程是什么时候进行等待队列并阻塞的呢，接下来看``get``方法。

#### FutureTask的get方法

```java
	/**
     * @throws CancellationException {@inheritDoc}
     */
    public V get() throws InterruptedException, ExecutionException {
        int s = state;
        if (s <= COMPLETING)
            s = awaitDone(false, 0L);
        return report(s);
    }
	/**
     * @throws CancellationException {@inheritDoc}
     */
    public V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException {
        if (unit == null)
            throw new NullPointerException();
        int s = state;
        if (s <= COMPLETING &&
            (s = awaitDone(true, unit.toNanos(timeout))) <= COMPLETING)
            throw new TimeoutException();
        return report(s);
    }
	/**
     * Awaits completion or aborts on interrupt or timeout.
     *
     * @param timed true if use timed waits
     * @param nanos time to wait, if timed
     * @return state upon completion or at timeout
     */
    private int awaitDone(boolean timed, long nanos)
        throws InterruptedException {
        // The code below is very delicate, to achieve these goals:
        // - call nanoTime exactly once for each call to park
        // - if nanos <= 0L, return promptly without allocation or nanoTime
        // - if nanos == Long.MIN_VALUE, don't underflow
        // - if nanos == Long.MAX_VALUE, and nanoTime is non-monotonic
        //   and we suffer a spurious wakeup, we will do no worse than
        //   to park-spin for a while
        long startTime = 0L;    // Special value 0L means not yet parked
        WaitNode q = null;
        boolean queued = false;
        for (;;) {
            int s = state;
            if (s > COMPLETING) {
                if (q != null)
                    q.thread = null;
                return s;
            }
            else if (s == COMPLETING)
                // We may have already promised (via isDone) that we are done
                // so never return empty-handed or throw InterruptedException
                Thread.yield();
            else if (Thread.interrupted()) {
                removeWaiter(q);
                throw new InterruptedException();
            }
            else if (q == null) {
                if (timed && nanos <= 0L)
                    return s;
                q = new WaitNode();
            }
            else if (!queued)
                queued = U.compareAndSwapObject(this, WAITERS,
                                                q.next = waiters, q);
            else if (timed) {
                final long parkNanos;
                if (startTime == 0L) { // first time
                    startTime = System.nanoTime();
                    if (startTime == 0L)
                        startTime = 1L;
                    parkNanos = nanos;
                } else {
                    long elapsed = System.nanoTime() - startTime;
                    if (elapsed >= nanos) {
                        removeWaiter(q);
                        return state;
                    }
                    parkNanos = nanos - elapsed;
                }
                // nanoTime may be slow; recheck before parking
                if (state < COMPLETING)
                    LockSupport.parkNanos(this, parkNanos);
            }
            else
                LockSupport.park(this);
        }
	/**
     * Returns result or throws exception for completed task.
     *
     * @param s completed state value
     */
    @SuppressWarnings("unchecked")
    private V report(int s) throws ExecutionException {
        Object x = outcome;
        if (s == NORMAL)
            return (V)x;
        if (s >= CANCELLED)
            throw new CancellationException();
        throw new ExecutionException((Throwable)x);
    }
```

可以看到``get``的两个方法都是调用了``awaitDone``方法，下面重点看下``awaitDone``方法。

``awaitDone``方法里面有个for死循环，退出循环的只有``state > COMPLETING``时``return state``，或者是在``Thread.interrupted()``返回true表示线程被中断时将线程从等待队列中移除并抛出``InterruptedException``。

进入for循环的时候如果任务的状态已经完成或者任务执行的时候抛出了异常，也即``state > COMPLETING``时，直接返回``state``，在``set``或者``setException``中会根据``state``进行``report``调用返回不同的状态。

在for循环中如果当前线程被中断，则将当前线程从等待队列中移除并抛出``InterruptedException``异常。

如果任务的状态``state < COMPLETING``也即任务正在执行，当前线程也没有被中断，第一次进入for循环的时候会进入``q == null``分支，创建``WaitNode``节点。

```java
/**
     * Simple linked list nodes to record waiting threads in a Treiber
     * stack.  See other classes such as Phaser and SynchronousQueue
     * for more detailed explanation.
     */
    static final class WaitNode {
        volatile Thread thread;
        volatile WaitNode next;
        WaitNode() { thread = Thread.currentThread(); }
    }
```

下次再进入for循环会进入``!queued``分支，尝试将刚创建的``WaitNode``节点的next指针指向``FutureTask``的``waiters``，并CAS替换``FutureTask``的``waiters``为刚创建的``WaitNode``节点，如果CAS失败，说明有其他线程也在进行CAS替换``FutureTask``的``waiters``的操作，并且成功了，下次再进for循环继续进行这个CAS操作，直到返回true，``queued``为true为止。到此，线程进入到了等待队列中，下次再进入for循环会根据是否``timed``来进行``LockSupport.parkNanos``或``LockSupport.park``阻塞线程操作，等待其他线程``unpark``来唤醒当前线程。那什么时候唤醒呢，其实在分析``run``方法的时候我们已经看到了在执行完后会进行``finishCompletion``操作，在``finishCompletion``方法中会唤醒等待队列中的线程。

#### FutureTask的cancel方法

```java
public boolean cancel(boolean mayInterruptIfRunning) {
        if (!(state == NEW &&
              U.compareAndSwapInt(this, STATE, NEW,
                  mayInterruptIfRunning ? INTERRUPTING : CANCELLED)))
            return false;
        try {    // in case call to interrupt throws exception
            if (mayInterruptIfRunning) {
                try {
                    Thread t = runner;
                    if (t != null)
                        t.interrupt();
                } finally { // final state
                    U.putOrderedInt(this, STATE, INTERRUPTED);
                }
            }
        } finally {
            finishCompletion();
        }
        return true;
    }
```

如果``state == NEW && U.compareAndSwapInt(this, STATE, NEW,mayInterruptIfRunning ? INTERRUPTING : CANCELLED)``返回false说明任务的``state``已经不是``NEW``了，直接返回false。否则根据``mayInterruptIfRunning``来对执行任务的线程``runner``进行中断操作。最后在``finally``块中进行了``finishCompletion``操作，来唤醒等待队列中的线程。

#### FutureTask的runAndReset方法

```java
	/**
     * Executes the computation without setting its result, and then
     * resets this future to initial state, failing to do so if the
     * computation encounters an exception or is cancelled.  This is
     * designed for use with tasks that intrinsically execute more
     * than once.
     *
     * @return {@code true} if successfully run and reset
     */
    protected boolean runAndReset() {
        if (state != NEW ||
            !U.compareAndSwapObject(this, RUNNER, null, Thread.currentThread()))
            return false;
        boolean ran = false;
        int s = state;
        try {
            Callable<V> c = callable;
            if (c != null && s == NEW) {
                try {
                    c.call(); // don't set result
                    ran = true;
                } catch (Throwable ex) {
                    setException(ex);
                }
            }
        } finally {
            // runner must be non-null until state is settled to
            // prevent concurrent calls to run()
            runner = null;
            // state must be re-read after nulling runner to prevent
            // leaked interrupts
            s = state;
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
        return ran && s == NEW;
    }
```

``runAndReset``方法相比``run``方法是在调用完``callable``的``call``方法后没有调用``set(result)``，没有对``state``任务状态进行转换，没有对``outcome``进行赋值，如果任务正常执行结束，``state``应该还是``NEW``，因此可以被重复调用。

#### FutureTask的isCancelled和isDone方法

```java
	public boolean isCancelled() {
        return state >= CANCELLED;
    }

    public boolean isDone() {
        return state != NEW;
    }
```

都是对``state``的判断

### reference

+ [why outcome object in FutureTask is non-volatile?](https://stackoverflow.com/questions/14432400/why-outcome-object-in-futuretask-is-non-volatile)
