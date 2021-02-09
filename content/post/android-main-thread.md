+++
title = "Android Main Thread"
date = "2014-10-13"
slug = "2014/10/13/android-main-thread"
Categories = ["dev", "android "]
+++
When facing bugs that were related to how we interact with the main thread, I decided to get a closer look at what the main thread really is.

```java
public class BigBang {
  public static void main(String... args) {
    // The Java universe starts here.
  }
}
```

All Java programs start with a call to a ``public static void main()`` method. This is true for Java Desktop programs, JEE servlet containers, and Android applications.

When the Android system boots, it starts a Linux process called ``ZygoteInit``. This process is a Dalvik VM that loads the [most common classes](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/master/preloaded-classes) of the Android SDK on a thread, and then waits.

<!-- more -->

When starting a new Android application, the Android system forks the ``ZygoteInit`` process. The thread in the child fork stops waiting, and calls ``ActivityThread.main()``.

## Loopers
Before going any further, we need to look at the ``Looper`` class.

Using a looper is a good way to dedicate one thread to process messages serially.

Each looper has a queue of ``Message`` objects (a ``MessageQueue``).

A looper has a ``loop()`` method that will process each message in the queue, and block when the queue is empty.

The ``Looper.loop()`` method code is similar to this:

```java
void loop() {
  while(true) {
    Message message = queue.next(); // blocks if empty.
    dispatchMessage(message);
    message.recycle();
  }
}
```

Each looper is associated with one thread. To create a new looper and associate it to the current thread, you must call ``Looper.prepare()``. The loopers are stored in a static ``ThreadLocal`` in the ``Looper`` class. You can retrieve the ``Looper`` associated to the current thread by calling ``Looper.myLooper()``.

The ``HandlerThread`` class does everything for you:

```java
HandlerThread thread = new HandlerThread("SquareHandlerThread");
thread.start(); // starts the thread.
Looper looper = thread.getLooper();
```

Its code is similar to this:

```java
class HandlerThread extends Thread {
  Looper looper;
  public void run() {
    Looper.prepare(); // Create a Looper and store it in a ThreadLocal.
    looper = Looper.myLooper(); // Retrieve the looper instance from the ThreadLocal, for later use.
    Looper.loop(); // Loop forever.
  }
}
```

## Handlers
A handler is the natural companion to a looper.

A handler has two purposes:

+ Send messages to a looper message queue from any thread.
+ Hndle messages dequeued by a looper on the thread associated to that looper.

```java
// Each handler is associated to one looper.
Handler handler = new Handler(looper) {
  public void handleMessage(Message message) {
    // Handle the message on the thread associated to the given looper.
    if (message.what == DO_SOMETHING) {
      // do something
    }
  }
};


// Create a new message associated to that handler.
Message message = handler.obtainMessage(DO_SOMETHING);

// Add the message to the looper queue.
// Can be called from any thread.
handler.sendMessage(message);
```

You can associate multiple handlers to one looper. The looper delivers the message to ``message.target``.

A popular and simpler way to use a handler is to post a ``Runnable``:

```java
// Create a message containing a reference to the runnable and add it to the looper queue
handler.post(new Runnable() {
  public void run() {
    // Runs on the thread associated to the looper associated to that handler.
  }
});
```

A handler can also be created without providing any looper:

```java
// DON'T DO THIS
Handler handler = new Handler();
```

The handler no argument constructor calls ``Looper.myLooper()`` and retrieves the looper associated with the current thread. This may or may not be the thread you actually want the handler to be associated with.

Most of the time, you just want to create a handler to post on the main thread:

```java
Handler handler = new Handler(Looper.getMainLooper());
```

Back to PSVM
Let's look at ``ActivityThread.main()`` again. Here is what it is essentially doing:

```java
public class ActivityThread {
  public static void main(String... args) {
    Looper.prepare();

    // You can now retrieve the main looper at any time by calling Looper.getMainLooper().
    Looper.setMainLooper(Looper.myLooper());

    // Post the first messages to the looper.
    // { ... }

    Looper.loop();
  }
}
```

Now you know why this thread is called the main thread :) .

Note: As you would expect, one of the first things that the main thread will do is create the ``Application`` and call ``Application.onCreate()``.

## Activities love orientation changes

Let's start with the activity lifecycle and the magic behind the handling of configuration changes.

### Why it matters

This article was inspired by a real crash that occurred in Square Register.
A simplified version of the code is:

```java
public class MyActivity extends Activity {
  protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    Handler handler = new Handler(Looper.getMainLooper());
    handler.post(new Runnable() {
      public void run() {
        doSomething();
      }
    });
  }

  void doSomething() {
    // Uses the activity instance
  }
}
```

As we will see, ``doSomething()`` can be called after the activity ``onDestroy()`` method has been called due to a configuration change. At that point, you should not use the activity instance anymore.

### A refresher on orientation changes

The device orientation can change at any time. We will simulate an orientation change while the activity is being created using ``Activity#setRequestedOrientation(int)``.

Can you predict the log output when starting this activity in portrait?

```java
public class MyActivity extends Activity {
  protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    Log.d("Square", "onCreate()");
    if (savedInstanceState == null) {
      Log.d("Square", "Requesting orientation change");
      setRequestedOrientation(ActivityInfo.SCREEN_ORIENTATION_LANDSCAPE);
    }
  }

  protected void onResume() {
    super.onResume();
    Log.d("Square", "onResume()");
  }

  protected void onPause() {
    super.onPause();
    Log.d("Square", "onPause()");
  }

  protected void onDestroy() {
    super.onDestroy();
    Log.d("Square", "onDestroy()");
  }
}
```

If you know the Android lifecycle, you probably predicted this:

```java
onCreate()
Requesting orientation change
onResume()
onPause()
onDestroy()
onCreate()
onResume()
```

The Android Lifecycle goes on normally, the activity is created, resumed, and then the orientation change is taken into account and the activity is paused, destroyed, and a new activity is created and resumed.

### Orientation changes and the main thread

Here is an important detail to remember: an orientation change leads to recreating the activity via a simple post of a message to the main thread looper queue.

Let's look at that by writing a spy that will read the content of the looper queue via reflection:

```java
public class MainLooperSpy {
  private final Field messagesField;
  private final Field nextField;
  private final MessageQueue mainMessageQueue;

  public MainLooperSpy() {
    try {
      Field queueField = Looper.class.getDeclaredField("mQueue");
      queueField.setAccessible(true);
      messagesField = MessageQueue.class.getDeclaredField("mMessages");
      messagesField.setAccessible(true);
      nextField = Message.class.getDeclaredField("next");
      nextField.setAccessible(true);
      Looper mainLooper = Looper.getMainLooper();
      mainMessageQueue = (MessageQueue) queueField.get(mainLooper);
    } catch (Exception e) {
      throw new RuntimeException(e);
    }
  }

  public void dumpQueue() {
    try {
      Message nextMessage = (Message) messagesField.get(mainMessageQueue);
      Log.d("MainLooperSpy", "Begin dumping queue");
      dumpMessages(nextMessage);
      Log.d("MainLooperSpy", "End dumping queue");
    } catch (IllegalAccessException e) {
      throw new RuntimeException(e);
    }
  }

  public void dumpMessages(Message message) throws IllegalAccessException {
    if (message != null) {
      Log.d("MainLooperSpy", message.toString());
      Message next = (Message) nextField.get(message);
      dumpMessages(next);
    }
  }
}
```

As you can see, the message queue is merely a linked list where each message has a reference to the next message.

We log the content of the queue right after the orientation change:

```java
public class MyActivity extends Activity {
  private final MainLooperSpy mainLooperSpy = new MainLooperSpy();

  protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    Log.d("Square", "onCreate()");
    if (savedInstanceState == null) {
      Log.d("Square", "Requesting orientation change");
      setRequestedOrientation(ActivityInfo.SCREEN_ORIENTATION_LANDSCAPE);
      mainLooperSpy.dumpQueue();
    }
  }
}
```

Here is the output:

```java
onCreate()
Requesting orientation change
Begin dumping queue
{ what=118 when=-94ms obj={1.0 208mcc15mnc en_US ldltr sw360dp w598dp h335dp 320dpi nrml land finger -keyb/v/h -nav/h s.44?spn} }
{ what=126 when=-32ms obj=ActivityRecord{41fd2b48 token=android.os.BinderProxy@41fcce50 no component name} }
End dumping queue
```

A quick look at the ``ActivityThread`` class tells us what those 118 and 126 messages are:

```java
public final class ActivityThread {
  private class H extends Handler {
    public static final int CONFIGURATION_CHANGED   = 118;
    public static final int RELAUNCH_ACTIVITY       = 126;
  }
}
```

Requesting an orientation change added ``CONFIGURATION_CHANGED`` and a ``RELAUNCH_ACTIVITY`` message to the main thread looper queue.

Let's take a step back and think about what's going on:

When the activity starts for the first time, the queue is empty. The message currently being executed is ``LAUNCH_ACTIVITY``, which creates the activity instance, calls ``onCreate()`` and then ``onResume()`` in a row. Then only the main looper processes the next message in the queue.

When a device orientation change is detected, a ``RELAUNCH_ACTIVITY`` is posted to the queue.

When that message is processed, it:

- calls ``onSaveInstanceState()``, ``onPause()``, ``onDestroy()`` on the old activity instance,
- creates a new activity instance,
- calls ``onCreate()`` and ``onResume()`` on that new activity instance.

All that in one message handling. Any message you post in the meantime will be handled after ``onResume()`` has been called.

### Tying it all together

What could happen if you post to a handler in ``onCreate()`` during an orientation change? Let's look at the two cases, right before and right after the orientation change:

```java
public class MyActivity extends Activity {
  private final MainLooperSpy mainLooperSpy = new MainLooperSpy();

  protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    Log.d("Square", "onCreate()");
    if (savedInstanceState == null) {
      Handler handler = new Handler(Looper.getMainLooper());
      handler.post(new Runnable() {
        public void run() {
          Log.d("Square", "Posted before requesting orientation change");
        }
      });
      Log.d("Square", "Requesting orientation change");
      setRequestedOrientation(ActivityInfo.SCREEN_ORIENTATION_LANDSCAPE);
      handler.post(new Runnable() {
        public void run() {
          Log.d("Square", "Posted after requesting orientation change");
        }
      });
      mainLooperSpy.dumpQueue();
    }
  }

  protected void onResume() {
    super.onResume();
    Log.d("Square", "onResume()");
  }

  protected void onPause() {
    super.onPause();
    Log.d("Square", "onPause()");
  }

  protected void onDestroy() {
    super.onDestroy();
    Log.d("Square", "onDestroy()");
  }
}
```

Here is the output:

```java
onCreate()
Requesting orientation change
Begin dumping queue
{ what=0 when=-129ms }
{ what=118 when=-96ms obj={1.0 208mcc15mnc en_US ldltr sw360dp w598dp h335dp 320dpi nrml land finger -keyb/v/h -nav/h s.46?spn} }
{ what=126 when=-69ms obj=ActivityRecord{41fd6b68 token=android.os.BinderProxy@41fd0ae0 no component name} }
{ what=0 when=-6ms }
End dumping queue
onResume()
Posted before requesting orientation change
onPause()
onDestroy()
onCreate()
onResume()
Posted after requesting orientation change
```

To sum things up: at the end on ``onCreate()``, the queue contained four messages. The first was the post before the orientation change, then the two messages related to the orientation change, and then only the post after the orientation change. The logs show that these were executed in order.

Therefore, any message posted before the orientation change will be handled before ``onPause()`` of the leaving activity, and any message posted after the orientation change will be handled after ``onResume()`` of the incoming activity.

The practical implication is that when you post a message, you have no guarantee that the activity instance that existed at the time it was sent will still be running when the message is handled (even if you post from ``onCreate()`` or ``onResume()``). If your message holds a reference to a view or an activity, the activity won't be garbage collected until the message is handled.

### What could you do?
 
#### The real fix
Stop calling ``handler.post()`` when you are already on the main thread. In most cases, ``handler.post()`` is used as a quick fix to ordering problems. Fix your architecture instead of messing it up with random ``handler.post()`` calls.

#### If you have a good reason to post
Make sure your message does not hold a reference to an activity, as you would do for a background operation.

#### If you really need that activity reference
Remove the message from the queue with ``handler.removeCallbacks()`` in the activity ``onPause()``.

#### If you want to get fired
Use ``handler.postAtFrontOfQueue()`` to make sure a message posted before ``onPause()`` is always handled before ``onPause()``. Your code will become really hard to read and understand. Seriously, don't.

#### A word on ``runOnUiThread()``
Did you notice that we created a handler and used ``handler.post()`` instead of directly calling ``Activity.runOnUiThread()``?

Here is why:

```java
public class Activity {
  public final void runOnUiThread(Runnable action) {
    if (Thread.currentThread() != mUiThread) {
      mHandler.post(action);
    } else {
      action.run();
    }
  }
}
```

Unlike ``handler.post()``, ``runOnUiThread()`` does not post the runnable if the current thread is already the main thread. Instead, it calls ``run()`` synchronously.

## Services
There is a common misconception that needs to die: a service does not run on a background thread.

All service lifecycle methods (``onCreate()``, ``onStartCommand()``, etc) run on the main thread (the very same thread that's used to play funky animations in your activities).

Whether you are in a service or an activity, long tasks must be executed in a dedicated background thread. This background thread can live as long as the process of your app lives, even when your activities are long gone.

However, at any time the Android system can decide to kill the app process. A service is a way to ask the system to let us live if possible and be polite by letting the service know before killing the process.

Side note: When an ``IBinder`` returned from ``onBind()`` receives a call from another process, the method will be executed in a background thread.

Take the time to read the Service documentation -- it's pretty good.

## IntentService
IntentService provides a simple way to serially process a queue of intents on a background thread.

```java
public class MyService extends IntentService {
  public MyService() {
    super("MyService");
  }

  protected void onHandleIntent(Intent intent) {
    // This is called on a background thread.
  }
}
```

Internally, it uses a ``Looper`` to handle the intents on a dedicated ``HandlerThread``. When the service is destroyed, the looper lets you finish handling the current intent, and then the background thread terminates.

## Conclusion
Most Android lifecycle methods are called on the main thread. Think of these callbacks as simple messages sent to a looper queue.

This article wouldn't be complete without the reminder that goes into almost every Android dev article: Do not block the main thread.

REF:[A journey on the Android Main Thread](http://corner.squareup.com/2013/10/android-main-thread-1.html),[A journey on the Android Main Thread - Lifecycle bits](http://corner.squareup.com/2013/12/android-main-thread-2.html)
