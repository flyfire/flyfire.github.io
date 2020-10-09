---
layout: post
title: "[Training]Best Practices for Background Jobs"
date: 2014-10-31 10:12
comments: true
categories: 
- dev
- android
tags:
- background
- android
---
<p><center><img src="/images/android_training.jpg"></center>

*   [Background jobs best practices](#overview)
    *   [Running in a background service](#running)
    *   [Loading data in the background](#loading)
    *   [Managing the device awake state](#managing)

<h2 id="overview">Best Practices for Background Jobs</h2>

These classes show you how to run jobs in the background to boost your application’s performance and minimize its drain on the battery.

<h3 id="running">Running in a background service</h3>

This class describes how to implement an IntentService, send it work requests, and report its results to other components.

####Creating a background service

The IntentService class provides a straightforward structure for running an operation on a single background thread. This allows it to handle long-running operations without affecting your user interface’s responsiveness. Also, an IntentService isn’t affected by most user interface lifecycle events, so it continues to run in circumstances that would shut down an AsyncTask.

An IntentService has a few limitations:

+ It can’t interact directly with your user interface. To put its results in the UI, you have to send them to an Activity.不能直接与UI交互
+ Work requests run sequentially. If an operation is running in an IntentService, and you send it another request, the request waits until the first operation is finished.任务顺序执行
+ An operation running on an IntentService can’t be interrupted.任务不可被打断

However, in most cases an IntentService is the preferred way to simple background operations.

<!-- more -->

To create an IntentService component for your app, define a class that extends IntentService, and within it, define a method that overrides ``onHandleIntent()``. For example:

```java
public class RSSPullService extends IntentService {
    @Override
    protected void onHandleIntent(Intent workIntent) {
        // Gets data from the incoming Intent
        String dataString = workIntent.getDataString();
        ...
        // Do work here, based on the contents of dataString
        ...
    }
}
```

Notice that the other callbacks of a regular Service component, such as ``onStartCommand()`` are automatically invoked by IntentService. In an IntentService, you should avoid overriding these callbacks.

####Sending work requests to the background service

This lesson shows you how to trigger the IntentService to run an operation by sending it an Intent. This Intent can optionally contain data for the IntentService to process. You can send an Intent to an IntentService from any point in an Activity or Fragment.

To create a work request and send it to an IntentService, create an explicit Intent, add work request data to it, and send it to IntentService by calling ``startService()``.

```java
/*
 * Creates a new Intent to start the RSSPullService
 * IntentService. Passes a URI in the
 * Intent's "data" field.
 */
mServiceIntent = new Intent(getActivity(), RSSPullService.class);
mServiceIntent.setData(Uri.parse(dataUrl));

// Starts the IntentService
getActivity().startService(mServiceIntent);
```

Once you call ``startService()``, the IntentService does the work defined in its ``onHandleIntent()`` method, and then stops itself.

####Reporting work status

This lesson shows you how to report the status of a work request run in a background service to the component that sent the request. This allows you, for example, to report the status of the request in an Activity object’s UI. The recommended way to send and receive status is to use a ``LocalBroadcastManager``, which limits broadcast Intent objects to components in your own app.

To send the status of a work request in an IntentService to other components, first create an Intent that contains the status in its extended data. As an option, you can add an action and data URI to this Intent.

Next, send the Intent by calling ``LocalBroadcastManager.sendBroadcast()``. This sends the Intent to any component in your application that has registered to receive it. To get an instance of ``LocalBroadcastManager``, call ``getInstance()``.

```java
public final class Constants {
    ...
    // Defines a custom Intent action
    public static final String BROADCAST_ACTION =
        "com.example.android.threadsample.BROADCAST";
    ...
    // Defines the key for the status "extra" in an Intent
    public static final String EXTENDED_DATA_STATUS =
        "com.example.android.threadsample.STATUS";
    ...
}
public class RSSPullService extends IntentService {
...
    /*
     * Creates a new Intent containing a Uri object
     * BROADCAST_ACTION is a custom Intent action
     */
    Intent localIntent =
            new Intent(Constants.BROADCAST_ACTION)
            // Puts the status into the Intent
            .putExtra(Constants.EXTENDED_DATA_STATUS, status);
    // Broadcasts the Intent to receivers in this app.
    LocalBroadcastManager.getInstance(this).sendBroadcast(localIntent);
...
}
```

To receive broadcast Intent objects, use a subclass of ``BroadcastReceiver``. In the subclass, implement the ``BroadcastReceiver.onReceive()`` callback method, which ``LocalBroadcastManager`` invokes when it receives an Intent. ``LocalBroadcastManager`` passes the incoming Intent to ``BroadcastReceiver.onReceive()``.

```java
// Broadcast receiver for receiving status updates from the IntentService
private class ResponseReceiver extends BroadcastReceiver
{
    // Prevents instantiation
    private DownloadStateReceiver() {
    }
    // Called when the BroadcastReceiver gets an Intent it's registered to receive
    @
    public void onReceive(Context context, Intent intent) {
        ...
        /*
         * Handle Intents here.
         */
        ...
    }
}

// Class that displays photos
public class DisplayActivity extends FragmentActivity {
    ...
    public void onCreate(Bundle stateBundle) {
        ...
        super.onCreate(stateBundle);
        ...
        // The filter's action is BROADCAST_ACTION
        IntentFilter mStatusIntentFilter = new IntentFilter(
                Constants.BROADCAST_ACTION);

        // Adds a data filter for the HTTP scheme
        mStatusIntentFilter.addDataScheme("http");
        ...

        // Instantiates a new DownloadStateReceiver
        DownloadStateReceiver mDownloadStateReceiver =
                new DownloadStateReceiver();
        // Registers the DownloadStateReceiver and its intent filters
        LocalBroadcastManager.getInstance(this).registerReceiver(
                mDownloadStateReceiver,
                mStatusIntentFilter);
        ...


        /*
         * Instantiates a new action filter.
         * No data filter is needed.
         */
        statusIntentFilter = new IntentFilter(Constants.ACTION_ZOOM_IMAGE);
        ...
        // Registers the receiver with the new filter
        LocalBroadcastManager.getInstance(getActivity()).registerReceiver(
                mDownloadStateReceiver,
                mIntentFilter);
```

Sending an broadcast Intent doesn’t start or resume an Activity. The BroadcastReceiver for an Activity receives and processes Intent objects even when your app is in the background, but doesn’t force your app to the foreground. If you want to notify the user about an event that happened in the background while your app was not visible, use a Notification.

<h3 id="loading">Loading data in the background</h3>

Besides doing the initial background query, a CursorLoader automatically re-runs the query when data associated with the query changes.This class describes how to use a CursorLoader to run a background query.

####Running a query with the CursorLoader

A CursorLoader runs an asynchronous query in the background against a ContentProvider, and returns the results to the Activity or FragmentActivity from which it was called. This allows the Activity or FragmentActivity to continue to interact with the user while the query is ongoing.

```java
public class PhotoThumbnailFragment extends FragmentActivity implements
        LoaderManager.LoaderCallbacks<Cursor> {
...
}
```

```java
    // Identifies a particular Loader being used in this component
    private static final int URL_LOADER = 0;
    ...
    /* When the system is ready for the Fragment to appear, this displays
     * the Fragment's View
     */
    public View onCreateView(
            LayoutInflater inflater,
            ViewGroup viewGroup,
            Bundle bundle) {
        ...
        /*
         * Initializes the CursorLoader. The URL_LOADER value is eventually passed
         * to onCreateLoader().
         */
        getLoaderManager().initLoader(URL_LOADER, null, this);
        ...
    }
```

```java
/*
* Callback that's invoked when the system has initialized the Loader and
* is ready to start the query. This usually happens when initLoader() is
* called. The loaderID argument contains the ID value passed to the
* initLoader() call.
*/
@Override
public Loader<Cursor> onCreateLoader(int loaderID, Bundle bundle)
{
    /*
     * Takes action based on the ID of the Loader that's being created
     */
    switch (loaderID) {
        case URL_LOADER:
            // Returns a new CursorLoader
            return new CursorLoader(
                        getActivity(),   // Parent activity context
                        mDataUrl,        // Table to query
                        mProjection,     // Projection to return
                        null,            // No selection clause
                        null,            // No selection arguments
                        null             // Default sort order
        );
        default:
            // An invalid id was passed in
            return null;
    }
}
```

####Handling the results

The loader then provides the query results to your Activity or FragmentActivity in your implementation of LoaderCallbacks.onLoadFinished(). One of the incoming arguments to this method is a Cursor containing the query results. You can use this object to update your data display or do further processing.

Besides ``onCreateLoader()`` and ``onLoadFinished()``, you also have to implement ``onLoaderReset()``. This method is invoked when CursorLoader detects that data associated with the Cursor has changed. When the data changes, the framework also re-runs the current query.

```java
public String[] mFromColumns = {
    DataProviderContract.IMAGE_PICTURENAME_COLUMN
};
public int[] mToFields = {
    R.id.PictureName
};
// Gets a handle to a List View
ListView mListView = (ListView) findViewById(R.id.dataList);
/*
 * Defines a SimpleCursorAdapter for the ListView
 *
 */
SimpleCursorAdapter mAdapter =
    new SimpleCursorAdapter(
            this,                // Current context
            R.layout.list_item,  // Layout for a single row
            null,                // No Cursor yet
            mFromColumns,        // Cursor columns to use
            mToFields,           // Layout fields to use
            0                    // No flags
    );
// Sets the adapter for the view
mListView.setAdapter(mAdapter);
...
/*
 * Defines the callback that CursorLoader calls
 * when it's finished its query
 */
@Override
public void onLoadFinished(Loader<Cursor> loader, Cursor cursor) {
    ...
    /*
     * Moves the query results into the adapter, causing the
     * ListView fronting this adapter to re-display
     */
    mAdapter.changeCursor(cursor);
}

/*
 * Invoked when the CursorLoader is being reset. For example, this is
 * called if the data in the provider changes and the Cursor becomes stale.
 */
@Override
public void onLoaderReset(Loader<Cursor> loader) {

    /*
     * Clears out the adapter's reference to the Cursor.
     * This prevents memory leaks.
     */
    mAdapter.changeCursor(null);
}
```


<h3 id="managing">Managing the device awake state</h3>

When an Android device is left idle, it will first dim, then turn off the screen, and ultimately turn off the CPU. This prevents the device’s battery from quickly getting drained. Yet there are times when your application might require a different behavior:

+ Apps such as games or movie apps may need to keep the screen turned on.
+ Other applications may not need the screen to remain on, but they may require the CPU to keep running until a critical operation finishes.

This class describes how to keep a device awake when necessary without draining its battery.

####Keeping the Device Awake

```java
public class MainActivity extends Activity {
  @Override
  protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);
    getWindow().addFlags(WindowManager.LayoutParams.FLAG_KEEP_SCREEN_ON);
  }
```

```java
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:keepScreenOn="true">
    ...
</RelativeLayout>
```

Using ``android:keepScreenOn="true"`` is equivalent to using ``FLAG_KEEP_SCREEN_ON``. You can use whichever approach is best for your app. The advantage of setting the flag programmatically in your activity is that it gives you the option of programmatically clearing the flag later and thereby allowing the screen to turn off.

You don’t need to clear the ``FLAG_KEEP_SCREEN_ON`` flag unless you no longer want the screen to stay on in your running application (for example, if you want the screen to time out after a certain period of inactivity). The window manager takes care of ensuring that the right things happen when the app goes into the background or returns to the foreground. But if you want to explicitly clear the flag and thereby allow the screen to turn off again, use ``clearFlags()``: ``getWindow().clearFlags(WindowManager.LayoutParams.FLAG_KEEP_SCREEN_ON)``.

If you need to keep the CPU running in order to complete some work before the device goes to sleep, you can use a ``PowerManager`` system service feature called wake locks. Wake locks allow your application to control the power state of the host device.

Creating and holding wake locks can have a dramatic impact on the host device’s battery life. Thus you should use wake locks only when strictly necessary and hold them for as short a time as possible. For example, you should never need to use a wake lock in an activity. As described above, if you want to keep the screen on in your activity, use ``FLAG_KEEP_SCREEN_ON``.

```java
PowerManager powerManager = (PowerManager) getSystemService(POWER_SERVICE);
Wakelock wakeLock = powerManager.newWakeLock(PowerManager.PARTIAL_WAKE_LOCK,
        "MyWakelockTag");
wakeLock.acquire();
```

####Scheduling Repeating Alarms

Alarms have these characteristics:

+ They let you fire Intents at set times and/or intervals.
+ You can use them in conjunction with broadcast receivers to start services and perform other operations.
+ They operate outside of your application, so you can use them to trigger events or actions even when your app is not running, and even if the device itself is asleep.
+ They help you to minimize your app’s resource requirements. You can schedule operations without relying on timers or continuously running background services.

For timing operations that are guaranteed to occur during the lifetime of your application, instead consider using the Handler class in conjunction with Timer and Thread. This approach gives Android better control over system resources.

Repeating alarms are a good choice for scheduling regular events or data lookups. A repeating alarm has the following characteristics:

+ A alarm type.
+ A trigger time. If the trigger time you specify is in the past, the alarm triggers immediately.
+ The alarm’s interval. For example, once a day, every hour, every 5 seconds, and so on.
+ A pending intent that fires when the alarm is triggered. When you set a second alarm that uses the same pending intent, it replaces the original alarm.

Follow these guidelines as you design your app:

+ Keep your alarm frequency to a minimum.
+ Don’t wake up the device unnecessarily (this behavior is determined by the alarm type, as described in Choose an alarm type).
+ Don’t make your alarm’s trigger time any more precise than it has to be:
    + Use ``setInexactRepeating()`` instead of ``setRepeating()`` whenever possible. When you use ``setInexactRepeating()``, Android synchronizes multiple inexact repeating alarms and fires them at the same time. This reduces the drain on the battery.
    + If your alarm’s behavior is based on an interval (for example, your alarm fires once an hour) rather than a precise trigger time (for example, your alarm fires at 7 a.m. sharp and every 20 minutes after that), use an ``ELAPSED_REALTIME`` alarm type.

```java
// Hopefully your alarm will have a lower frequency than this!
alarmMgr.setInexactRepeating(AlarmManager.ELAPSED_REALTIME_WAKEUP,
        AlarmManager.INTERVAL_HALF_HOUR,
        AlarmManager.INTERVAL_HALF_HOUR, alarmIntent);

private AlarmManager alarmMgr;
private PendingIntent alarmIntent;
...
alarmMgr = (AlarmManager)context.getSystemService(Context.ALARM_SERVICE);
Intent intent = new Intent(context, AlarmReceiver.class);
alarmIntent = PendingIntent.getBroadcast(context, 0, intent, 0);

alarmMgr.set(AlarmManager.ELAPSED_REALTIME_WAKEUP,
        SystemClock.elapsedRealtime() +
        60 * 1000, alarmIntent);

// Set the alarm to start at approximately 2:00 p.m.
Calendar calendar = Calendar.getInstance();
calendar.setTimeInMillis(System.currentTimeMillis());
calendar.set(Calendar.HOUR_OF_DAY, 14);

// With setInexactRepeating(), you have to use one of the AlarmManager interval
// constants--in this case, AlarmManager.INTERVAL_DAY.
alarmMgr.setInexactRepeating(AlarmManager.RTC_WAKEUP, calendar.getTimeInMillis(),
        AlarmManager.INTERVAL_DAY, alarmIntent);

private AlarmManager alarmMgr;
private PendingIntent alarmIntent;
...
alarmMgr = (AlarmManager)context.getSystemService(Context.ALARM_SERVICE);
Intent intent = new Intent(context, AlarmReceiver.class);
alarmIntent = PendingIntent.getBroadcast(context, 0, intent, 0);

// Set the alarm to start at 8:30 a.m.
Calendar calendar = Calendar.getInstance();
calendar.setTimeInMillis(System.currentTimeMillis());
calendar.set(Calendar.HOUR_OF_DAY, 8);
calendar.set(Calendar.MINUTE, 30);

// setRepeating() lets you specify a precise custom interval--in this case,
// 20 minutes.
alarmMgr.setRepeating(AlarmManager.RTC_WAKEUP, calendar.getTimeInMillis(),
        1000 * 60 * 20, alarmIntent);

// If the alarm has been set, cancel it.
if (alarmMgr!= null) {
    alarmMgr.cancel(alarmIntent);
}
```

####Start an Alarm When the Device Boots

+ Set the ``RECEIVE_BOOT_COMPLETED`` permission in your application’s manifest. This allows your app to receive the ``ACTION_BOOT_COMPLETED`` that is broadcast after the system finishes booting (this only works if the app has already been launched by the user at least once):

```xml
<uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED"/>
```

+ Implement a BroadcastReceiver to receive the broadcast:

```java
public class SampleBootReceiver extends BroadcastReceiver {

    @Override
    public void onReceive(Context context, Intent intent) {
        if (intent.getAction().equals("android.intent.action.BOOT_COMPLETED")) {
            // Set the alarm here.
        }
    }
}
```

+ Add the receiver to your app’s manifest file with an intent filter that filters on the ``ACTION_BOOT_COMPLETED`` action:

```xml
<receiver android:name=".SampleBootReceiver"
        android:enabled="false">
    <intent-filter>
        <action android:name="android.intent.action.BOOT_COMPLETED"></action>
    </intent-filter>
</receiver>
```

Notice that in the manifest, the boot receiver is set to ``android:enabled=“false”``. This means that the receiver will not be called unless the application explicitly enables it. This prevents the boot receiver from being called unnecessarily. You can enable a receiver (for example, if the user sets an alarm) as follows:

```java
ComponentName receiver = new ComponentName(context, SampleBootReceiver.class);
PackageManager pm = context.getPackageManager();

pm.setComponentEnabledSetting(receiver,
        PackageManager.COMPONENT_ENABLED_STATE_ENABLED,
        PackageManager.DONT_KILL_APP);
```

Once you enable the receiver this way, it will stay enabled, even if the user reboots the device. In other words, programmatically enabling the receiver overrides the manifest setting, even across reboots. The receiver will stay enabled until your app disables it. You can disable a receiver (for example, if the user cancels an alarm) as follows:

```java
ComponentName receiver = new ComponentName(context, SampleBootReceiver.class);
PackageManager pm = context.getPackageManager();

pm.setComponentEnabledSetting(receiver,
        PackageManager.COMPONENT_ENABLED_STATE_DISABLED,
        PackageManager.DONT_KILL_APP);
```
