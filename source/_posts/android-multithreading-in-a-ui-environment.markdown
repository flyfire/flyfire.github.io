---
layout: post
title: "Multithreading in a UI Environment"
date: 2014-10-31 09:14
comments: true
categories: 
- android
- dev
tags:
- android
---
<p><center><img src="/images/android_threads.png"></center>

##Why do we need multithreading in Android applications?

Let’s say you want to do a very long operation when the user pushes a button.If you are not using another thread, it will look something like this:

```java
((Button)findViewById(R.id.Button01)).setOnClickListener(
             new OnClickListener() {
      
          @Override
          public void onClick(View v) {
            int result = doLongOperation();
            updateUI(result);
          }
      });
```

What will happen?The UI freezes. This is a really bad UI experience. The program may even crash.

##The problem in using threads in a UI environment

So what will happen if we use a Thread for a long running operation.Let’s try a simple example:

```java
((Button)findViewById(R.id.Button01)).setOnClickListener(
new OnClickListener() {
          
          @Override
          public void onClick(View v) {
              
              (new Thread(new Runnable() {
                  
                  @Override
                  public void run() {
                    int result = doLongOperation();
                    updateUI(result);
                  }
              })).start();
              
          }
```

The result in this case is that the application crashes.

```bash
12-07 16:24:29.089: ERROR/AndroidRuntime(315): FATAL EXCEPTION: Thread-8
12-07 16:24:29.089: ERROR/AndroidRuntime(315): android.view.ViewRoot$CalledFromWrongThreadException: Only the original thread that created a view hierarchy can touch its views.
12-07 16:24:29.089: ERROR/AndroidRuntime(315): at ...Clearly the Android OS wont let threads other than the main thread change UI elements.
```

But why?Android UI toolkit, like many other UI environments, is not thread-safe.

<!-- more -->

The solution

+ A queue of messages. Each message is a job to be handled.
+ Threads can add messages.
+ Only a single thread pulls messages one by one from the queue.

The same solution was implemented in swing ([Event dispatching thread](http://en.wikipedia.org/wiki/Event_dispatching_thread) and [SwingUtilities.invokeLater()](http://download.oracle.com/javase/1.4.2/docs/api/javax/swing/SwingUtilities.html#invokeLater%28java.lang.Runnable%29) )

##Handler

The Handler is the middleman between a new thread and the message queue.

+ Option 1 – Run the new thread and use the handler to send messages for ui changes

```java
final Handler myHandler = new Handler(){
  @Override
public void handleMessage(Message msg) {
      updateUI((String)msg.obj);
}
  
};

(new Thread(new Runnable() {
  
  @Override
  public void run() {
      Message msg = myHandler.obtainMessage();
      
      msg.obj = doLongOperation();
      
      myHandler.sendMessage(msg);
  }
})).start();
```

keep in mind that updating the UI should still be a short operation, since the UI freezes during the updating process.

Other possibilities:

+ ``handler.obtainMessage`` with parameters
+ ``handler.sendMessageAtFrontOfQueue()``
+ ``handler.sendMessageAtTime()``
+ ``handler.sendMessageDelayed()``
+ ``handler.sendEmptyMessage()``

+ Option 2 – run the new thread and use the handler to post a runnable which updates the GUI.

```java
final Handler myHandler = new Handler();
              
              (new Thread(new Runnable() {
                  
                  @Override
                  public void run() {
                      final String res = doLongOperation();
                      myHandler.post(new Runnable() {
                          
                          @Override
                          public void run() {
                              updateUI(res);
                          }
                      });
                  }
              })).start();
              
          }
```

<p><center><img src="/images/android_looper.png"></center>

##Looper

If we want to dive a bit deeper into the android mechanism we have to understand what is a Looper.We have talked about the message queue that the main thread pulls messages and runnables from it and executes them.We also said that each handler you create has a reference to this queue.What we haven’t said yet is that the main thread has a reference to an object named Looper.

The Looper gives the Thread the access to the message queue.Only the main thread has executes to the Looper by default.

Lets say you would like to create a new thread and you also want to take advantage of the message queue functionality in that thread.

```java
(new Thread(new Runnable() {

                      @Override
                      public void run() {

                          innerHandler = new Handler();
                          
                          Message message = innerHandler.obtainMessage();
                          innerHandler.dispatchMessage(message);
                      }
                  })).start();
```

Here we created a new thread which uses the handler to put a message in the messages queue.

This will be the result:

```bash
12-10 20:41:51.807: ERROR/AndroidRuntime(254): Uncaught handler: thread Thread-8 exiting due to uncaught exception
12-10 20:41:51.817: ERROR/AndroidRuntime(254): java.lang.RuntimeException: Can't create handler inside thread that has not called Looper.prepare()
12-10 20:41:51.817: ERROR/AndroidRuntime(254): at android.os.Handler.(Handler.java:121)
12-10 20:41:51.817: ERROR/AndroidRuntime(254): at ...
```

The new created thread does not have a ``Looper`` with a queue attached to it. Only the UI thread has a ``Looper``.We can however create a ``Looper`` for the new thread.In order to do that we need to use 2 functions: ``Looper.prepare()`` and ``Looper.loop()``.

```java
(new Thread(new Runnable() {

                  @Override
                  public void run() {

                      Looper.prepare();
                      innerHandler = new Handler();
                              
                      Message message = innerHandler.obtainMessage();
                      innerHandler.dispatchMessage(message);
                      Looper.loop();
                  }
              })).start();
```

If you use this option, don’t forget to use also the ``quit()`` function so the Looper will not loop for ever.

```java
@Override
  protected void onDestroy() {
      innerHandler.getLooper().quit();
      super.onDestroy();
  }
```

##AsyncTask

I have explained to you that a Handler is the new thread’s way to communicate with the UI thread.If while reading this you were thinking to yourself, isn’t there an easier way to do all of that… well, you know what?! There is.

Android team has created a class called AsyncTask which is in short a thread that can handle UI.

Just like in java you extend the class Thread and a SwingWorker in Swing, in Android you extend the class AsyncTask.There is no interface here like Runnable to implement I’m afraid.

```java
class MyAsyncTask extends AsyncTask<Integer, String, Long> {
      
      @Override
      protected Long doInBackground(Integer... params) {
          
          long start = System.currentTimeMillis();
          for (Integer integer : params) {
              publishProgress("start processing "+integer);
              doLongOperation();
              publishProgress("done processing "+integer);
          }
          
          return start - System.currentTimeMillis();
      }

      
      
      @Override
      protected void onProgressUpdate(String... values) {
          updateUI(values[0]);
      }
      
      @Override
      protected void onPostExecute(Long time) {
          updateUI("Done with all the operations, it took:" +
                                     time + " millisecondes");
      }

      @Override
      protected void onPreExecute() {
          updateUI("Starting process");
      }
      
      
      public void doLongOperation() {
          
          try {
              Thread.sleep(1000);
          } catch (InterruptedException e) {
              e.printStackTrace();
          }
          
      }
      
    }
```

This is how you start this thread:

```java
MyAsyncTask aTask = new MyAsyncTask();
aTask.execute(1, 2, 3, 4, 5);
```

AsyncTask defines 3 generic types:``AsyncTask<{type of the input}, {type of the update unit}, {type of the result}>``,You don’t have to use all of them – simply use ‘Void’ for any of them.

Notice that ``AsyncTask`` has 4 operations, which are executed by order.

+ ``onPreExecute()`` – is invoked before the execution.
+ ``onPostExecute()`` – is invoked after the execution.
+ ``doInBackground()`` – the main operation. Write your heavy operation here.
+ ``onProgressUpdate()`` – Indication to the user on progress. It is invoked every time publishProgress() is called.

**Notice: doInBackground() is invoked on a background thread where onPreExecute(), onPostExecute() and onProgressUpdate() are invoked on the UI thread since their purpose is to update the UI.**

Android developer website also mentions these 4 rules regarding the ``AsyncTask``:

+ The task instance must be created on the UI thread.
+ ``execute(Params…)`` must be invoked on the UI thread.
+ Do not call ``onPreExecute()``, ``onPostExecute(Result)``, ``doInBackground(Params…)``, ``onProgressUpdate(Progress…)`` manually.
+ The task can be executed only once (an exception will be thrown if a second execution is attempted.)

##Timer + TimerTask

Another option is to use a Timer.Timer is a comfortable way to dispatch a thread in the future, be it once or more.

Instead of doing this:

```java
Runnable threadTask = new Runnable() {
  
  @Override
  public void run() {
      
      while(true){
          try {
              Thread.sleep(2000);
          } catch (InterruptedException e) {
              e.printStackTrace();
          }
          
          doSomething();
      }
  }
};
  
(new Thread(threadTask)).start();
```

do this:


```java
TimerTask timerTask = new TimerTask() {     
  @Override
  public void run() {
      doSomething();            
  }
};

Timer timer = new Timer();
timer.schedule(timerTask, 2000,2000);
```

which is a bit more elegant.

Bare in mind, you still have to use Handler if you need to do UI operations.

