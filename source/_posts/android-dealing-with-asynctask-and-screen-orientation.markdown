---
layout: post
title: "Dealing with AsyncTask and Screen Orientation"
date: 2014-10-26 19:31
comments: true
categories: 
- dev
- android
tags:
- android
---
<p><center><img src="/images/android_logo.jpg"/></center></p>

A common task in Android is to perform some background activity in another thread, meanwhile displaying a ProgressDialog to the user. Example of these tasks include downloading some data from internet, logging into an application, etc. Implementing an AsyncTask is fairly a simple job, the big challenge is how to handle it properly when an orientation change occurs.

<!-- more -->

In this article I will walk though a series of potential solutions to address the screen orientation issues when using an AsyncTask.

So, lets create a proof of concept application that makes use of an AsyncTask which does not handle configuration changes yet, and then present a few solutions.

Here’s the AsyncTask implementation that we will be using during the tutorial:

```java
public class AsyncTaskExample extends AsyncTask<String, Integer, String> {
 
    private final TaskListener listener;
 
    public AsyncTaskExample(TaskListener listener) {
        this.listener = listener;
    }
 
    @Override
    protected void onPreExecute() {
        listener.onTaskStarted();
    }
 
    @Override
    protected String doInBackground(String... params) {
        for (int i = 1; i <= 10; i++) {
            Log.d("GREC", "AsyncTask is working: " + i);
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        return "All Done!";
    }
 
    @Override
    protected void onPostExecute(String result) {
        listener.onTaskFinished(result);
    }
}
```

+ ``doInBackground()`` – this will be called by the AsyncTask on a background thread, and performs all the heavy work. For the sake of this example, I just wrote a simple loop  with a delay of 1 sec between iterations to simulate a task that takes some time.
+ The constructor of the class takes a listener as a parameter. The listener will be used to delegate the work of ``onPreExecute()/onPostExecute()`` to the calling Activity.

This is the interface definition used by AsyncTaskExample:

```java
public interface TaskListener {
    void onTaskStarted();
 
    void onTaskFinished(String result);
}
```

And here’s the usage of AsyncTaskExample (the problematic case):

```java
public class MainActivity extends Activity implements TaskListener, OnClickListener {
 
    private ProgressDialog progressDialog;
 
    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.main);
        findViewById(R.id.start).setOnClickListener(this);
    }
 
    @Override
    public void onClick(View v) {
        if (v.getId() == R.id.start) {
            new AsyncTaskExample(this).execute();
        }
    }
 
    @Override
    public void onTaskStarted() {
        progressDialog = ProgressDialog.show(CopyOfMainActivity.this, "Loading", "Please wait a moment!");
    }
 
    @Override
    public void onTaskFinished(String result) {
        if (progressDialog != null) {
            progressDialog.dismiss();
        }
    }
}
```

The Activity implements the TaskListener interface and provides appropriate implementation for its methods,  displaying the ProgressDialog when the task is started, and dismissing it when the task is finished. The AsyncTask is fired when clicking on a button.

Now, if you run this example without changing the screen orientation, the AsyncTask will start and finish its work normally. Problems begin to appear when the device orientation is changed while the AsyncTask is in the middle of the work. The application will crash, and an exception similar to these ones will be thrown: java.lang.IllegalArgumentException: View not attached to window manager, or Activity has leaked window com.android.internal.policy….

The cause relies in the Activity life cycle. A change in device orientation is interpreted as a configuration change which causes the current activity to be destroyed and then recreated. Android calls onPause(), onStop(), and onDestroy() on currently instance of activity, then a new instance of the same activity is recreated calling onCreate(), onStart(), and onResume(). The reason why Android have to do this, is because depending of screen orientation, portrait or landscape, we may need to load and display different resources, and only through re-creation Android can guarantee that all our resources will be re-requested.

But don’t panic, you are not alone, there are several solutions that will help us to deal with this situation.

###Solution 1 – Think twice if you really need an AsyncTask.

AsyncTasks are good for performing background work, but they are bound to the Activity which adds some complexity. For things like making HTTP requests to a server perhaps you should consider an IntentService. IntentService used in conjunction with a BroadcastReceiver or ResultReceiver to deliver results, could do a better job than an AsyncTask in certain situations.

###Solution 2 – Put the AsyncTask in a Fragment.

Using fragments probably is the cleanest way to handle configuration changes. By default, Android destroys and recreates the fragments just like activities, however, fragments have the ability to retain their instances, simply by calling: ``setRetainInstance(true)``, in one of its callback methods, for example in the onCreate().

There’s however one aspect that should be taken in consideration in order to achieve the desired result. Our AsyncTask uses a ProgressDialog to signal when the AsyncTask is started, and dismisses it when the task is done. This complicates a bit the things because even if we are using ``setRetainInstance(true)``, we should close all windows and dialogs when the Activity is destroyed, otherwise we will get an: ``Activity has leaked window com.android.internal.policy…``  exception. This happens when you try to show a dialog after you have exited the Activity.

In order to address this issue, we will add some logic to keep track of AsyncTask status (running/not running). We will dismiss the ProgressDialog when the fragment is detached from activity, and check in ``onActivityCreated()`` the status of AsyncTask. If the status is “running”, this means we are returning from a screen orientation and we will just re-create and display the ProgressDialog to show that the AsyncTask is still working.

```java
public class ExampleFragment extends Fragment implements TaskListener, OnClickListener {
 
    private ProgressDialog progressDialog;
    private boolean isTaskRunning = false;
    private AsyncTaskExample asyncTask;
 
    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setRetainInstance(true);
    }
 
    @Override
    public void onActivityCreated(Bundle savedInstanceState) {
        super.onActivityCreated(savedInstanceState);
        // If we are returning here from a screen orientation
        // and the AsyncTask is still working, re-create and display the
        // progress dialog.
        if (isTaskRunning) {
            progressDialog = ProgressDialog.show(getActivity(), "Loading", "Please wait a moment!");
        }
    }
 
    @Override
    public View onCreateView(LayoutInflater inflater, ViewGroup container, Bundle savedInstanceState) {
        View view = inflater.inflate(R.layout.fragment_layout, container, false);
        Button button = (Button) view.findViewById(R.id.start);
        button.setOnClickListener(this);
        return view;
    }
 
    @Override
    public void onClick(View v) {
        if (!isTaskRunning) {
            asyncTask = new AsyncTaskExample(this);
            asyncTask.execute();
        }
    }
 
    @Override
    public void onTaskStarted() {
        isTaskRunning = true;
        progressDialog = ProgressDialog.show(getActivity(), "Loading", "Please wait a moment!");
    }
 
    @Override
    public void onTaskFinished(String result) {
        if (progressDialog != null) {
            progressDialog.dismiss();
        }
        isTaskRunning = false;
    }
 
    @Override
    public void onDetach() {
        // All dialogs should be closed before leaving the activity in order to avoid
        // the: Activity has leaked window com.android.internal.policy... exception
        if (progressDialog != null && progressDialog.isShowing()) {
            progressDialog.dismiss();
        }
        super.onDetach();
    }
}
```

###Solution 3 – Lock the screen orientation

You could do this in 2 ways:

a) permanently locking the screen orientation of the activity, specifying the screenOrientation attribute in the AndroidManifest with “portrait” or “landscape” values:

```xml
<activity
   android:screenOrientation="portrait"
   ...  />
```

b) or, temporarily locking the screen in ``onPreExecute()``, and unlocking it in ``onPostExecute()``, thus preventing any orientation change while the AsyncTask is working:

```java
@Override
public void onTaskStarted() {
    lockScreenOrientation();
    progressDialog = ProgressDialog.show(CopyOfCopyOfMainActivity.this, "Loading", "Please wait a moment!");
}
 
@Override
public void onTaskFinished(String result) {
    if (progressDialog != null) {
        progressDialog.dismiss();
    }
    unlockScreenOrientation();
}
 
private void lockScreenOrientation() {
    int currentOrientation = getResources().getConfiguration().orientation;
    if (currentOrientation == Configuration.ORIENTATION_PORTRAIT) {
        setRequestedOrientation(ActivityInfo.SCREEN_ORIENTATION_PORTRAIT);
    } else {
        setRequestedOrientation(ActivityInfo.SCREEN_ORIENTATION_LANDSCAPE);
    }
}
 
private void unlockScreenOrientation() {
    setRequestedOrientation(ActivityInfo.SCREEN_ORIENTATION_SENSOR);
}
```

####Solution 4 – Prevent the Activity from being recreated.

This is the easiest way to handle configuration changes, but the less advised. The only thing you need to do is to specify the configChanges attribute followed by a list of values that specifies when the activity should prevent itself from restarting.

```xml
<activity
   android:configChanges="orientation|keyboardHidden"
   android:name=".MainActivity"
   .... />
```

Using this approach however, is not recommended, and this is clearly stated in the Android documentation: Using this attribute should be avoided and used only as a last-resort.

You may ask what’s wrong with this approach. Well, if you build the above example against Android 2.2 it will work fine, but if you build it against Android 3.0 and higher, you may notice that the application still crashes on orientation change. This is because starting  with Android 3.0 you need also to handle the ``screenSize``, and ``smallestScreenSize``:

```xml
<activity
   android:configChanges="orientation|keyboardHidden|screenSize|smallestScreenSize"
   android:name=".MainActivity"
   .... />
```

As it turns out, not only a screen orientation causes the Activity to recreate, there are also other events which may produce configuration changes and restart the Activity, and there’s a good chance that we won’t handle them all. This is why the use of this technique is discouraged.

<embed src="http://player.youku.com/player.php/sid/XODExOTY3MzY0/v.swf" allowFullScreen="true" quality="high" width="480" height="400" align="middle" allowScriptAccess="always" type="application/x-shockwave-flash"></embed>

<embed src="http://player.youku.com/player.php/sid/XODExOTg2MTgw/v.swf" allowFullScreen="true" quality="high" width="480" height="400" align="middle" allowScriptAccess="always" type="application/x-shockwave-flash"></embed>

REF:

+ [Dealing with AsyncTask and Screen Orientation](http://androidresearch.wordpress.com/2013/05/10/dealing-with-asynctask-and-screen-orientation/)
