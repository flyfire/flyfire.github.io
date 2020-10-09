---
layout: post
title: "The Hidden Pitfalls of AsyncTask"
date: 2014-10-19 15:56
comments: true
categories: 
- dev
- android
tags:
- asynctask
---
<p><center><img src="/images/android_logo.jpg"/></center></p>

I originally wrote this article when I was (foolishly) still using AsyncTasks. Nowadays I simply consider it a mistake in all cases. As you'll see from the original article, there are a lot of problems with it - and there are much better solutions.

My preferred alternative these days are combining [RxJava](https://github.com/Netflix/RxJava) with schedulers. You get the same effect as an ``AsyncTask`` with none of the problems, plus you get an awesome framework in addition. I know, recommending a library to solve a problem is irritating, but RxJava is worth looking at for many reasons.

When ``AsyncTask`` was introduced to Android, it was labeled as “[Painless Threading](http://android-developers.blogspot.com/2009/05/painless-threading.html).” Its goal was to make background Threads which could interact with the UI thread easier. It was successful on that count, but it’s not exactly painless – there are a number of cases where ``AsyncTask`` is not a silver bullet. It is easy to blindly use ``AsyncTask`` without realizing what can go wrong if not handled with care. Below are some of the problems that can arise when using ``AsyncTask`` without fully understanding it.

<!-- more -->

##AsyncTask and Rotation

AsyncTask’s primary goal is to make it easy to run a Thread in the background that can later interact with the UI thread. Therefore the most common use case is to have an ``AsyncTask`` run a time-consuming operation that updates a portion of the UI when it’s completed (in ``AsyncTask.onPostExecute()``).

This works great… until you rotate the screen. **When an app is rotated, the entire ``Activity`` is destroyed and recreated. When the Activity is restarted, your AsyncTask’s reference to the ``Activity`` is invalid, so ``onPostExecute()`` will have no effect on the new Activity.** This can be confusing if you are implicitly referencing the current Activity by having AsyncTask as an inner class of the Activity.

The usual solution to this problem is to hold onto a reference to AsyncTask that lasts between configuration changes, which updates the target Activity as it restarts. There are a variety of ways to do this, though they either boil down to using a global holder (such as in the ``Application`` object) or passing it through ``Activity.onRetainNonConfigurationInstance()``. For a Fragment-based system, you could use a retained Fragment (via ``Fragment.setRetainedInstance(true)``) to store running AsyncTasks.

##AsyncTasks and the Lifecycle

Along the same lines as above, it is a misconception to think that just because the Activity that originally spawned the ``AsyncTask`` is dead, the ``AsyncTask`` is as well. It will continue running on its merry way even if you exit the entire application. **The only way that an ``AsyncTask`` finishes early is if it is canceled via ``AsyncTask.cancel()``.**

This means that you have to manage the cancellation of AsyncTasks yourself; otherwise you run the risk of bogging down your app with unnecessary background tasks, or of leaking memory. When you know you will no longer need an ``AsyncTask``, be sure to cancel it so that it doesn’t cause any headaches later in the execution of your app.

##Cancelling AsyncTasks

Suppose you’ve got a search query that runs in an ``AsyncTask``. The user may be able to change the search parameters while the ``AsyncTask`` is running, so you call ``AsyncTask.cancel()`` and then fire up a new ``AsyncTask`` for the next query. **This seems to work… until you check the logs and realize that your ``AsyncTask``s all ran till completion, regardless of whether you called ``cancel()`` or not!** This even happens if you pass mayInterruptIfRunning as true – what’s going on?

The problem is that there’s a misconception about what ``AsyncTask.cancel()`` actually does. It does not kill the Thread with no regard for the consequences! All it does is set the ``AsyncTask`` to a “cancelled” state. **It’s up to you to check whether the AsyncTask has been canceled so that you can halt your operation**. As for mayInterruptIfRunning – all it does is send an ``interrupt()`` to the running Thread. In the case that your Thread is uninterruptible, then it won’t stop the Thread at all.There are two simple solutions that cover most situations: Either check ``AsyncTask.isCancelled()`` on a regular basis during your long-running operation, or keep your Thread interruptible. Either way, when you call ``AsyncTask.cancel()`` these methods should prevent your operation from running longer than necessary.

This advice doesn’t always work, though – what if you’re calling a long-running method that is uninterruptible (such as ``BitmapFactory.decodeStream()``)? The only success I’ve had in this situation is to create a situation which causes an Exception to be thrown (in this case, prematurely closing the stream that BitmapFactory was using). This meant that ``cancel()`` alone wouldn’t solve the problem – outside intervention was required.

##Limitations on Concurrent AsyncTasks

I’m not encouraging people to start hundreds of threads in the background; however, it is worth noting that there are some limitations on the number of ``AsyncTask``s that you can start. The modern ``AsyncTask`` is limited to 128 concurrent tasks, with an additional queue of 10 tasks (if supporting Android 1.5, it’s a limit of ten tasks at a time, with a maximum queue of 10 tasks). That means that if you queue up more than 138 tasks before they can complete, your app will crash. Most often I see this problem when people use ``AsyncTask``s to load Bitmaps from the net.

If you are finding yourself running up against these limits, you should start by rethinking your design that calls for so many background threads. Alternatively, you could setup a more intelligent queue for your tasks so that you’re not starting them all at once. If you’re desperate, you can grab a copy of ``AsyncTask`` and adjust the pool sizes in the code itself.

+ [The Hidden Pitfalls of AsyncTask](http://blog.danlew.net/2014/06/21/the-hidden-pitfalls-of-asynctask/)
