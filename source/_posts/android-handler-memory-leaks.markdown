---
layout: post
title: "Android Handler Memory Leaks"
date: 2016-02-25 19:58
comments: true
categories: 
- dev
- android
tags:
- android
---
Android uses Java as a platform for development. This helps us with many low level issues including memory management, platform type dependencies, and so on. However we still sometimes get crashes with OutOfMemory. So where’s the garbage collector?

I’m going to focus on one of the cases where big objects in memory can’t be cleared for a lengthy period of time. This case is not ultimately a memory leak - objects will be collected at some point - so we sometimes ignore it. This is not advisable as it can sometimes lead to OOM errors.

The case I’m describing is the Handler leak, which is usually detected as a warning by Lint.

<!-- more -->

##Basic Example

<center><img src="/images/anonymous_runnable_code.png"></center>

This is a very basic activity. Notice that this anonymous ``Runnable`` has been posted to the ``Handler`` with a very long delay. We’ll run it and rotate the phone couple of times, then dump memory and analyze it.

<center><img src="/images/anonymous_runnable_memory_analyze.png"></center>

We have seven activities in memory now. This is definitely not good. Let’s find out why GC is not able to clear them.ps.The query I made to get a list of all Activities remaining in memory was created in OQL (Object Query Language), which is very simple, yet powerful.

<center><img src="/images/anonymous_runnable_memory_explained.png"></center>

As you can see, one of the activities is referenced by this$0. This is an indirect reference from the anonymous class to the owner class. This$0 is referenced by callback, which is then referenced by a chain of next’s of Message back to the main thread.Any time you create a non-static class inside the owner class, Java creates an indirect reference to the owner.

Once you post ``Runnable`` or ``Message`` into Handler, it’s then stored in list of ``Message`` commands referenced from ``LooperThread`` until the message is executed. Posting delayed messages is a clear leak for at least the time of the delay value. Posting without delay may cause a temporary leak as well if the queue of messages is large.

##Static Runnable Solution

Let’s try to overcome a memory leak by getting rid of ``this$0``, by converting the anonymous class to static.

<center><img src="/images/static_class.png"></center>

Run, rotate and get the memory dump.

<center><img src="/images/static_class_memory_analyze.png"></center>

What, again? Let’s see who keeps referring to Activities.

<center><img src="/images/static_class_memory_analyze_explained.png"></center>

Take a look at the bottom of the tree - activity is kept as a reference to mContext inside mTextView of our DoneRunnable class. Using static inner classes is not enough to overcome memory leaks, however. We need to do more.

##Static Runnable With WeakReference

Let’s continue using iterative fixes and get rid of the reference to TextView, which keeps activity from being destroyed.

<center><img src="/images/static_class_with_WeakRef.png"></center>

Note that we are keeping WeakReference to TextView, and let’s run, rotate and dump memory.Be careful with WeakReferences. They can be null at any moment, so resolve them first to a local variable (hard reference) and then check to null before use.

<center><img src="/images/static_class_with_WeakRef_memory_analyze.png"></center>

Hooray! Only one activity instance. This solves our memory problem.

So for this approach we should:

+ Use static inner classes (or outer classes)
+ Use ``WeakReference`` to all objects manipulated from ``Handler/Runnable``

If you compare this code to the initial code, you might find a big difference in readability and code clearance. The initial code is much shorter and much clearer, and you’ll see that eventually, text in textView will be changed to ‘Done’. No need to browse the code to realise that.

Writing this much boilerplate code is very tedious, especially if postDelayed is set to a short time, such as 50ms. There are better and clearer solutions.

##Cleanup All Messages onDestroy

``Handler`` class has an interesting feature - ``removeCallbacksAndMessages`` - which can accept ``null`` as argument. It will remove all ``Runnables`` and ``Messages`` posted to a particular handler. Let’s use it in ``onDestroy``.

<center><img src="/images/removeCallbacks.png"></center>

Let’s run, rotate and dump memory.

<center><img src="/images/removeCallbacks_memory_analyze.png"></center>

Good! Only one instance.

This approach is way better than the previous one, as it keeps code clear and readable. The only overhead is to remember to clear all messages on activity/fragment destroy.

I have one more solution which, if you’re lazy like me, you might like even more. :)

##Use WeakHandler

The Badoo team came up with the interesting idea of introducing ``WeakHandler`` - a class that behaves as ``Handler``, but is way safer.

It takes advantage of hard and weak references to get rid of memory leaks. I will describe the idea in detail a bit later, but let’s look at the code first:

<center><img src="/images/WeakHandler.png"></center>

Very similar to the original code apart from one small difference - instead of using ``android.os.Handler``, I’ve used ``WeakHandler``. Let’s run, rotate and dump memory:

<center><img src="/images/WeakHandler_memory_analyze.png"></center>

Nice, isn’t it? The code is cleaner than ever, and memory is clean as well! :)

To use it, just add dependency to your ``build.gradle``:

```groovy
repositories {
    maven {
        repositories {
            url 'https://oss.sonatype.org/content/repositories/releases/'
        }
    }
}

dependencies {
    compile 'com.badoo.mobile:android-weak-handler:1.0'
}
```

And import it in your java class:

```java
import com.badoo.mobile.util.WeakHandler;
```

Visit Badoo’s github page, where you can fork it, or study it’s [source code](https://github.com/badoo/android-weak-handler).

##WeakHandler. How it works

The main aim of ``WeakHandler`` is to keep ``Runnables/Messages`` hard-referenced while ``WeakHandler`` is also hard-referenced. Once it can be GC-ed, all messages should go away as well.

Here is a simple diagram that demonstrates differences between using normal ``Handler`` and ``WeakHandler`` to post anonymous runnables:

<center><img src="/images/WeakHandler_how_it_works.png"></center>

Looking at the top diagram, ``Activity`` keeps a reference to ``Handler``, which posts ``Runnable`` (puts it into queue of ``Messages`` referenced from ``Thread``). Everything is fine except the indirect reference from ``Runnable`` to ``Activity``. While ``Message`` is in the queue, all graphs can’t be garbage-collected.

By comparison, in the bottom diagram ``Activity`` holds ``WeakHandler``, which keeps ``Handler`` inside. When we ask it to post ``Runnable``, it is wrapped into ``WeakRunnable`` and posted. So the ``Message`` queue keeps reference only to ``WeakRunnable``. ``WeakRunnable`` keeps weak reference to the desired ``Runnable``, so the ``Runnable`` can be garbage-collected.

Another little trick is that ``WeakHandler`` still keeps a hard reference to the desired ``Runnable``, to prevent it from being garbage-collected while ``WeakRunnable`` is active.

The side-effect of using ``WeakHandler`` is that all messages and runnables may not be executed if ``WeakHandler`` has been garbage-collected. To prevent that, just keep a reference to it from ``Activity``. Once ``Activity`` is ready to be collected, all graphs with ``WeakHandler`` will collected as well.

##Conclusions

Using ``postDelayed`` in Android requires additional effort. To achieve it we came up with three different methods:

+ Use a static inner ``Runnable/Handler`` with ``WeakReference`` to owner class
+ Clear all messages from ``Handler`` in ``onDestroy`` of ``Activity/Fragment``
+ Use [WeakHandler](https://github.com/badoo/android-weak-handler) from Badoo as a silver bullet

It’s up to you to choose your preferred technique. The second seems very reasonable, but needs some extra work. The third is my favourite, obviously, but it require some attention as well - ``WeakHandler`` should not be used without hard reference from outside.


```java
package com.badoo.mobile.util;

import android.os.Handler;
import android.os.Looper;
import android.os.Message;
import android.support.annotation.NonNull;
import android.support.annotation.Nullable;
import android.support.annotation.VisibleForTesting;

import java.lang.ref.WeakReference;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

/**
 * Memory safer implementation of android.os.Handler
 * <p/>
 * Original implementation of Handlers always keeps hard reference to handler in queue of execution.
 * If you create anonymous handler and post delayed message into it, it will keep all parent class
 * for that time in memory even if it could be cleaned.
 * <p/>
 * This implementation is trickier, it will keep WeakReferences to runnables and messages,
 * and GC could collect them once WeakHandler instance is not referenced any more
 * <p/>
 *
 * @see android.os.Handler
 *
 * Created by Dmytro Voronkevych on 17/06/2014.
 */
@SuppressWarnings("unused")
public class WeakHandler {
    private final Handler.Callback mCallback; // hard reference to Callback. We need to keep callback in memory
    private final ExecHandler mExec;
    private Lock mLock = new ReentrantLock();
    @SuppressWarnings("ConstantConditions")
    @VisibleForTesting
    final ChainedRef mRunnables = new ChainedRef(mLock, null);

    /**
     * Default constructor associates this handler with the {@link Looper} for the
     * current thread.
     *
     * If this thread does not have a looper, this handler won't be able to receive messages
     * so an exception is thrown.
     */
    public WeakHandler() {
        mCallback = null;
        mExec = new ExecHandler();
    }

    /**
     * Constructor associates this handler with the {@link Looper} for the
     * current thread and takes a callback interface in which you can handle
     * messages.
     *
     * If this thread does not have a looper, this handler won't be able to receive messages
     * so an exception is thrown.
     *
     * @param callback The callback interface in which to handle messages, or null.
     */
    public WeakHandler(@Nullable Handler.Callback callback) {
        mCallback = callback; // Hard referencing body
        mExec = new ExecHandler(new WeakReference<>(callback)); // Weak referencing inside ExecHandler
    }

    /**
     * Use the provided {@link Looper} instead of the default one.
     *
     * @param looper The looper, must not be null.
     */
    public WeakHandler(@NonNull Looper looper) {
        mCallback = null;
        mExec = new ExecHandler(looper);
    }

    /**
     * Use the provided {@link Looper} instead of the default one and take a callback
     * interface in which to handle messages.
     *
     * @param looper The looper, must not be null.
     * @param callback The callback interface in which to handle messages, or null.
     */
    public WeakHandler(@NonNull Looper looper, @NonNull Handler.Callback callback) {
        mCallback = callback;
        mExec = new ExecHandler(looper, new WeakReference<>(callback));
    }

    /**
     * Causes the Runnable r to be added to the message queue.
     * The runnable will be run on the thread to which this handler is
     * attached.
     *
     * @param r The Runnable that will be executed.
     *
     * @return Returns true if the Runnable was successfully placed in to the
     *         message queue.  Returns false on failure, usually because the
     *         looper processing the message queue is exiting.
     */
    public final boolean post(@NonNull Runnable r) {
        return mExec.post(wrapRunnable(r));
    }

    /**
     * Causes the Runnable r to be added to the message queue, to be run
     * at a specific time given by <var>uptimeMillis</var>.
     * <b>The time-base is {@link android.os.SystemClock#uptimeMillis}.</b>
     * The runnable will be run on the thread to which this handler is attached.
     *
     * @param r The Runnable that will be executed.
     * @param uptimeMillis The absolute time at which the callback should run,
     *         using the {@link android.os.SystemClock#uptimeMillis} time-base.
     *
     * @return Returns true if the Runnable was successfully placed in to the
     *         message queue.  Returns false on failure, usually because the
     *         looper processing the message queue is exiting.  Note that a
     *         result of true does not mean the Runnable will be processed -- if
     *         the looper is quit before the delivery time of the message
     *         occurs then the message will be dropped.
     */
    public final boolean postAtTime(@NonNull Runnable r, long uptimeMillis) {
        return mExec.postAtTime(wrapRunnable(r), uptimeMillis);
    }

    /**
     * Causes the Runnable r to be added to the message queue, to be run
     * at a specific time given by <var>uptimeMillis</var>.
     * <b>The time-base is {@link android.os.SystemClock#uptimeMillis}.</b>
     * The runnable will be run on the thread to which this handler is attached.
     *
     * @param r The Runnable that will be executed.
     * @param uptimeMillis The absolute time at which the callback should run,
     *         using the {@link android.os.SystemClock#uptimeMillis} time-base.
     *
     * @return Returns true if the Runnable was successfully placed in to the
     *         message queue.  Returns false on failure, usually because the
     *         looper processing the message queue is exiting.  Note that a
     *         result of true does not mean the Runnable will be processed -- if
     *         the looper is quit before the delivery time of the message
     *         occurs then the message will be dropped.
     *
     * @see android.os.SystemClock#uptimeMillis
     */
    public final boolean postAtTime(Runnable r, Object token, long uptimeMillis) {
        return mExec.postAtTime(wrapRunnable(r), token, uptimeMillis);
    }

    /**
     * Causes the Runnable r to be added to the message queue, to be run
     * after the specified amount of time elapses.
     * The runnable will be run on the thread to which this handler
     * is attached.
     *
     * @param r The Runnable that will be executed.
     * @param delayMillis The delay (in milliseconds) until the Runnable
     *        will be executed.
     *
     * @return Returns true if the Runnable was successfully placed in to the
     *         message queue.  Returns false on failure, usually because the
     *         looper processing the message queue is exiting.  Note that a
     *         result of true does not mean the Runnable will be processed --
     *         if the looper is quit before the delivery time of the message
     *         occurs then the message will be dropped.
     */
    public final boolean postDelayed(Runnable r, long delayMillis) {
        return mExec.postDelayed(wrapRunnable(r), delayMillis);
    }

    /**
     * Posts a message to an object that implements Runnable.
     * Causes the Runnable r to executed on the next iteration through the
     * message queue. The runnable will be run on the thread to which this
     * handler is attached.
     * <b>This method is only for use in very special circumstances -- it
     * can easily starve the message queue, cause ordering problems, or have
     * other unexpected side-effects.</b>
     *
     * @param r The Runnable that will be executed.
     *
     * @return Returns true if the message was successfully placed in to the
     *         message queue.  Returns false on failure, usually because the
     *         looper processing the message queue is exiting.
     */
    public final boolean postAtFrontOfQueue(Runnable r) {
        return mExec.postAtFrontOfQueue(wrapRunnable(r));
    }

    /**
     * Remove any pending posts of Runnable r that are in the message queue.
     */
    public final void removeCallbacks(Runnable r) {
        final WeakRunnable runnable = mRunnables.remove(r);
        if (runnable != null) {
            mExec.removeCallbacks(runnable);
        }
    }

    /**
     * Remove any pending posts of Runnable <var>r</var> with Object
     * <var>token</var> that are in the message queue.  If <var>token</var> is null,
     * all callbacks will be removed.
     */
    public final void removeCallbacks(Runnable r, Object token) {
        final WeakRunnable runnable = mRunnables.remove(r);
        if (runnable != null) {
            mExec.removeCallbacks(runnable, token);
        }
    }

    /**
     * Pushes a message onto the end of the message queue after all pending messages
     * before the current time. It will be received in callback,
     * in the thread attached to this handler.
     *
     * @return Returns true if the message was successfully placed in to the
     *         message queue.  Returns false on failure, usually because the
     *         looper processing the message queue is exiting.
     */
    public final boolean sendMessage(Message msg) {
        return mExec.sendMessage(msg);
    }

    /**
     * Sends a Message containing only the what value.
     *
     * @return Returns true if the message was successfully placed in to the
     *         message queue.  Returns false on failure, usually because the
     *         looper processing the message queue is exiting.
     */
    public final boolean sendEmptyMessage(int what) {
        return mExec.sendEmptyMessage(what);
    }

    /**
     * Sends a Message containing only the what value, to be delivered
     * after the specified amount of time elapses.
     * @see #sendMessageDelayed(android.os.Message, long)
     *
     * @return Returns true if the message was successfully placed in to the
     *         message queue.  Returns false on failure, usually because the
     *         looper processing the message queue is exiting.
     */
    public final boolean sendEmptyMessageDelayed(int what, long delayMillis) {
        return mExec.sendEmptyMessageDelayed(what, delayMillis);
    }

    /**
     * Sends a Message containing only the what value, to be delivered
     * at a specific time.
     * @see #sendMessageAtTime(android.os.Message, long)
     *
     * @return Returns true if the message was successfully placed in to the
     *         message queue.  Returns false on failure, usually because the
     *         looper processing the message queue is exiting.
     */
    public final boolean sendEmptyMessageAtTime(int what, long uptimeMillis) {
        return mExec.sendEmptyMessageAtTime(what, uptimeMillis);
    }

    /**
     * Enqueue a message into the message queue after all pending messages
     * before (current time + delayMillis). You will receive it in
     * callback, in the thread attached to this handler.
     *
     * @return Returns true if the message was successfully placed in to the
     *         message queue.  Returns false on failure, usually because the
     *         looper processing the message queue is exiting.  Note that a
     *         result of true does not mean the message will be processed -- if
     *         the looper is quit before the delivery time of the message
     *         occurs then the message will be dropped.
     */
    public final boolean sendMessageDelayed(Message msg, long delayMillis) {
        return mExec.sendMessageDelayed(msg, delayMillis);
    }

    /**
     * Enqueue a message into the message queue after all pending messages
     * before the absolute time (in milliseconds) <var>uptimeMillis</var>.
     * <b>The time-base is {@link android.os.SystemClock#uptimeMillis}.</b>
     * You will receive it in callback, in the thread attached
     * to this handler.
     *
     * @param uptimeMillis The absolute time at which the message should be
     *         delivered, using the
     *         {@link android.os.SystemClock#uptimeMillis} time-base.
     *
     * @return Returns true if the message was successfully placed in to the
     *         message queue.  Returns false on failure, usually because the
     *         looper processing the message queue is exiting.  Note that a
     *         result of true does not mean the message will be processed -- if
     *         the looper is quit before the delivery time of the message
     *         occurs then the message will be dropped.
     */
    public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
        return mExec.sendMessageAtTime(msg, uptimeMillis);
    }

    /**
     * Enqueue a message at the front of the message queue, to be processed on
     * the next iteration of the message loop.  You will receive it in
     * callback, in the thread attached to this handler.
     * <b>This method is only for use in very special circumstances -- it
     * can easily starve the message queue, cause ordering problems, or have
     * other unexpected side-effects.</b>
     *
     * @return Returns true if the message was successfully placed in to the
     *         message queue.  Returns false on failure, usually because the
     *         looper processing the message queue is exiting.
     */
    public final boolean sendMessageAtFrontOfQueue(Message msg) {
        return mExec.sendMessageAtFrontOfQueue(msg);
    }

    /**
     * Remove any pending posts of messages with code 'what' that are in the
     * message queue.
     */
    public final void removeMessages(int what) {
        mExec.removeMessages(what);
    }

    /**
     * Remove any pending posts of messages with code 'what' and whose obj is
     * 'object' that are in the message queue.  If <var>object</var> is null,
     * all messages will be removed.
     */
    public final void removeMessages(int what, Object object) {
        mExec.removeMessages(what, object);
    }

    /**
     * Remove any pending posts of callbacks and sent messages whose
     * <var>obj</var> is <var>token</var>.  If <var>token</var> is null,
     * all callbacks and messages will be removed.
     */
    public final void removeCallbacksAndMessages(Object token) {
        mExec.removeCallbacksAndMessages(token);
    }

    /**
     * Check if there are any pending posts of messages with code 'what' in
     * the message queue.
     */
    public final boolean hasMessages(int what) {
        return mExec.hasMessages(what);
    }

    /**
     * Check if there are any pending posts of messages with code 'what' and
     * whose obj is 'object' in the message queue.
     */
    public final boolean hasMessages(int what, Object object) {
        return mExec.hasMessages(what, object);
    }

    public final Looper getLooper() {
        return mExec.getLooper();
    }

    private WeakRunnable wrapRunnable(@NonNull Runnable r) {
        //noinspection ConstantConditions
        if (r == null) {
            throw new NullPointerException("Runnable can't be null");
        }
        final ChainedRef hardRef = new ChainedRef(mLock, r);
        mRunnables.insertAfter(hardRef);
        return hardRef.wrapper;
    }

    private static class ExecHandler extends Handler {
        private final WeakReference<Handler.Callback> mCallback;

        ExecHandler() {
            mCallback = null;
        }

        ExecHandler(WeakReference<Handler.Callback> callback) {
            mCallback = callback;
        }

        ExecHandler(Looper looper) {
            super(looper);
            mCallback = null;
        }

        ExecHandler(Looper looper, WeakReference<Handler.Callback> callback) {
            super(looper);
            mCallback = callback;
        }

        @Override
        public void handleMessage(@NonNull Message msg) {
            if (mCallback == null) {
                return;
            }
            final Handler.Callback callback = mCallback.get();
            if (callback == null) { // Already disposed
                return;
            }
            callback.handleMessage(msg);
        }
    }

    static class WeakRunnable implements Runnable {
        private final WeakReference<Runnable> mDelegate;
        private final WeakReference<ChainedRef> mReference;

        WeakRunnable(WeakReference<Runnable> delegate, WeakReference<ChainedRef> reference) {
            mDelegate = delegate;
            mReference = reference;
        }

        @Override
        public void run() {
            final Runnable delegate = mDelegate.get();
            final ChainedRef reference = mReference.get();
            if (reference != null) {
                reference.remove();
            }
            if (delegate != null) {
                delegate.run();
            }
        }
    }

    static class ChainedRef {
        @Nullable
        ChainedRef next;
        @Nullable
        ChainedRef prev;
        @NonNull
        final Runnable runnable;
        @NonNull
        final WeakRunnable wrapper;

        @NonNull
        Lock lock;

        public ChainedRef(@NonNull Lock lock, @NonNull Runnable r) {
            this.runnable = r;
            this.lock = lock;
            this.wrapper = new WeakRunnable(new WeakReference<>(r), new WeakReference<>(this));
        }

        public WeakRunnable remove() {
            lock.lock();
            try {
                if (prev != null) {
                    prev.next = next;
                }
                if (next != null) {
                    next.prev = prev;
                }
                prev = null;
                next = null;
            } finally {
                lock.unlock();
            }
            return wrapper;
        }

        public void insertAfter(@NonNull ChainedRef candidate) {
            lock.lock();
            try {
                if (this.next != null) {
                    this.next.prev = candidate;
                }

                candidate.next = this.next;
                this.next = candidate;
                candidate.prev = this;
            } finally {
                lock.unlock();
            }
        }

        @Nullable
        public WeakRunnable remove(Runnable obj) {
            lock.lock();
            try {
                ChainedRef curr = this.next; // Skipping head
                while (curr != null) {
                    if (curr.runnable == obj) { // We do comparison exactly how Handler does inside
                        return curr.remove();
                    }
                    curr = curr.next;
                }
            } finally {
                lock.unlock();
            }
            return null;
        }
    }
}
```

##reference
+ [Android Handler Memory Leaks](https://techblog.badoo.com/blog/2014/08/28/android-handler-memory-leaks)

