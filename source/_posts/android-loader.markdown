---
layout: post
title: "Android Loader"
date: 2014-10-24 06:34
comments: true
categories: 
- dev
- android
tags:
- android
---
<p><center><img src="/images/android_logo.jpg"/></center></p>

*	[Android Loader](#overview)
	* [Life before Loaders](#before)
	* [Understanding the LoaderManager](#loadermanager)
	* [Implementing Loaders](#implementing)
	
<h2 id="overview">Android Loader</h2>

This post gives a brief introduction to Loaders and the LoaderManager. The first section describes how data was loaded prior to the release of Android 3.0, pointing out out some of the flaws of the pre-Honeycomb APIs. The second section defines the purpose of each class and summarizes their powerful ability in asynchronously loading data.

<!-- more -->

<h3 id="before">Life before Loaders</h3>
Before Android 3.0, many Android applications lacked in responsiveness. UI interactions glitched, transitions between activities lagged, and ANR (Application Not Responding) dialogs rendered apps totally useless. This lack of responsiveness stemmed mostly from the fact that developers were performing queries on the UI thread—a very poor choice for lengthy operations like loading data.

While the [documentation](http://developer.android.com/guide/practices/responsiveness.html) has always stressed the importance of instant feedback, the pre-Honeycomb APIs simply did not encourage this behavior. Before Loaders, cursors were primarily managed and queried for with two (now deprecated) Activity methods:

+ ``public void startManagingCursor(Cursor)``:Tells the activity to take care of managing the cursor's lifecycle based on the activity's lifecycle. The cursor will automatically be deactivated (``deactivate()``) when the activity is stopped, and will automatically be closed (``close()``) when the activity is destroyed. When the activity is stopped and then later restarted, the Cursor is re-queried (``requery()``) for the most up-to-date data.

+ ``public Cursor managedQuery(Uri, String, String, String, String)``:A wrapper around the ``ContentResolver's query()`` method. In addition to performing the query, it begins management of the cursor (that is, ``startManagingCursor(cursor)`` is called before it is returned).

While convenient, these methods were deeply flawed in that they performed queries on the UI thread. What's more, the "managed cursors" did not retain their data across Activity configuration changes. The need to ``requery()`` the cursor's data in these situations was unnecessary, inefficient, and made orientation changes clunky and sluggish as a result.

####The Problem with "Managed Cursors"

Let's illustrate the problem with "managed cursors" through a simple code sample. Given below is a ListActivity that loads data using the pre-Honeycomb APIs. The activity makes a query to the ContentProvider and begins management of the returned cursor. The results are then bound to a SimpleCursorAdapter, and are displayed on the screen in a ListView. The code has been condensed for simplicity.

```java
public class SampleListActivity extends ListActivity {

  private static final String[] PROJECTION = new String[] {"_id", "text_column"};

  @Override
  protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);

    // Performs a "managed query" to the ContentProvider. The Activity 
    // will handle closing and requerying the cursor.
    //
    // WARNING!! This query (and any subsequent re-queries) will be
    // performed on the UI Thread!!
    Cursor cursor = managedQuery(
        CONTENT_URI,  // The Uri constant in your ContentProvider class
        PROJECTION,   // The columns to return for each data row
        null,         // No where clause
        null,         // No where clause
        null);        // No sort order

    String[] dataColumns = { "text_column" };
    int[] viewIDs = { R.id.text_view };
 
    // Create the backing adapter for the ListView.
    //
    // WARNING!! While not readily obvious, using this constructor will 
    // tell the CursorAdapter to register a ContentObserver that will
    // monitor the underlying data source. As part of the monitoring
    // process, the ContentObserver will call requery() on the cursor 
    // each time the data is updated. Since Cursor#requery() is performed 
    // on the UI thread, this constructor should be avoided at all costs!
    SimpleCursorAdapter adapter = new SimpleCursorAdapter(
        this,                // The Activity context
        R.layout.list_item,  // Points to the XML for a list item
        cursor,              // Cursor that contains the data to display
        dataColumns,         // Bind the data in column "text_column"...
        viewIDs);            // ...to the TextView with id "R.id.text_view"

    // Sets the ListView's adapter to be the cursor adapter that was 
    // just created.
    setListAdapter(adapter);
  }
}
```

There are three problems with the code above. If you have understood this post so far, the first two shouldn't be difficult to spot:

+ managedQuery performs a query on the main UI thread. This leads to unresponsive apps and should no longer be used.
+ As seen in the [Activity.java](http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/1.5_r4/android/app/Activity.java#Activity.managedQuery%28android.net.Uri%2Cjava.lang.String%5B%5D%2Cjava.lang.String%2Cjava.lang.String%29), the call to ``managedQuery`` begins management of the returned cursor with a call to ``startManagingCursor(cursor)``. Having the activity manage the cursor seems convenient at first, as we no longer need to worry about deactivating/closing the cursor ourselves. However, this signals the activity to call ``requery()`` on the cursor each time the activity returns from a stopped state, and therefore puts the UI thread at risk. This cost significantly outweighs the convenience of having the activity deactivate/close the cursor for us.
+ The ``SimpleCursorAdapter`` constructor (line 32) is deprecated and should not be used. The problem with this constructor is that it will have the ``SimpleCursorAdapter`` auto-requery its data when changes are made. More specifically, the ``CursorAdapter`` will register a ``ContentObserver`` that monitors the underlying data source for changes, calling ``requery()`` on its bound cursor each time the data is modified. The standard constructor should be used instead (if you intend on loading the adapter's data with a CursorLoader, make sure you pass 0 as the last argument). Don't worry if you couldn't spot this one... it's a very subtle bug.

With the first Android tablet about to be released, something had to be done to encourage UI-friendly development. The larger, 7-10" Honeycomb tablets called for more complicated, interactive, multi-paned layouts. Further, the introduction of the Fragment meant that applications were about to become more dynamic and event-driven. A simple, single-threaded approach to loading data could no longer be encouraged. Thus, the Loader and the LoaderManager were born.

Prior to Honeycomb, it was difficult to manage cursors, synchronize correctly with the UI thread, and ensure all queries occurred on a background thread. Android 3.0 introduced the Loader and LoaderManager classes to help simplify the process. Both classes are available for use in the Android Support Library, which supports all Android platforms back to Android 1.6.

The new Loader API is a huge step forward, and significantly improves the user experience. Loaders ensure that all cursor operations are done asynchronously, thus eliminating the possibility of blocking the UI thread. Further, when managed by the LoaderManager, Loaders retain their existing cursor data across the activity instance (for example, when it is restarted due to a configuration change), thus saving the cursor from unnecessary, potentially expensive re-queries. As an added bonus, Loaders are intelligent enough to monitor the underlying data source for updates, re-querying automatically when the data is changed.

Since the introduction of Loaders in Honeycomb and Compatibility Library, Android applications have changed for the better. Making use of the now deprecated startManagingCursor and managedQuery methods are extremely discouraged; not only do they slow down your app, but they can potentially bring it to a screeching halt. Loaders, on the other hand, significantly speed up the user experience by offloading the work to a separate background thread.

<h3 id="loadermanager">Understanding the LoaderManager</h3>

For now, you should think of Loaders as simple, self-contained objects that (1) load data on a separate thread, and (2) monitor the underlying data source for updates, re-querying when changes are detected. This is more than enough to get you through the contents of this post. 

####What is the LoaderManager?

Simply stated, the ``LoaderManager`` is responsible for managing one or more Loaders associated with an ``Activity`` or ``Fragment``. Each Activity and each Fragment has exactly one LoaderManager instance that is in charge of starting, stopping, retaining, restarting, and destroying its Loaders. These events are sometimes initiated directly by the client, by calling ``initLoader()``, ``restartLoader()``, or ``destroyLoader()``. Just as often, however, these events are triggered by major Activity/Fragment lifecycle events. For example, when an Activity is destroyed, the Activity instructs its LoaderManager to destroy and close its Loaders (as well as any resources associated with them, such as a Cursor).

The ``LoaderManager`` does not know how data is loaded, nor does it need to. Rather, the ``LoaderManager`` instructs its ``Loaders`` when to start/stop/reset their load, retaining their state across configuration changes and providing a simple interface for delivering results back to the client. In this way, the ``LoaderManager`` is a much more intelligent and generic implementation of the now-deprecated ``startManagingCursor`` method. While both manage data across the twists and turns of the Activity lifecycle, the ``LoaderManager`` is far superior for several reasons:

+ ``startManagingCursor`` manages Cursors, whereas the ``LoaderManager`` manages ``Loader<D>`` objects. The advantage here is that ``Loader<D>`` is generic, where D is the container object that holds the loaded data. In other words, the data source doesn't have to be a ``Cursor``; it could be a ``List``, a ``JSONArray``... anything. The ``LoaderManager`` is independent of the container object that holds the data and is much more flexible as a result.

+ Calling ``startManagingCursor`` will make the ``Activity`` call ``requery()`` on the managed cursor. As mentioned in the previous post, ``requery()`` is a potentially expensive operation that is performed on the main UI thread. Subclasses of the ``Loader<D>`` class, on the other hand, are expected to load their data asynchronously, so using the ``LoaderManager`` will never block the UI thread.

+ ``startManagingCursor`` does not retain the Cursor's state across configuration changes. Instead, each time the Activity is destroyed due to a configuration change (a simple orientation change, for example), the ``Cursor`` is destroyed and must be requeried. The ``LoaderManager`` is much more intelligent in that it retains its Loaders' state across configuration changes, and thus doesn't need to requery its data.

+ The ``LoaderManager`` provides seamless monitoring of data! Whenever the Loader's data source is modified, the ``LoaderManager`` will receive a new asynchronous load from the corresponding Loader, and will return the updated data to the client. (Note: the LoaderManager will only be notified of these changes if the Loader is implemented correctly. ).

If you feel overwhelmed by the details above, I wouldn't stress over it. The most important thing to take away from this is that the ``LoaderManager`` makes your life easy. It initializes, manages, and destroys Loaders for you, reducing both coding complexity and subtle lifecycle-related bugs in your Activitys and Fragments. Further, interacting with the ``LoaderManager`` involves implementing three simple callback methods. We discuss the ``LoaderManager.LoaderCallbacks<D>`` in the next section.

####Implementing the LoaderManager.LoaderCallbacks<D> Interface

The ``LoaderManager.LoaderCallbacks<D>`` interface is a simple contract that the ``LoaderManager`` uses to report data back to the client. Each ``Loader`` gets its own callback object that the ``LoaderManager`` will interact with. This callback object fills in the gaps of the abstract ``LoaderManager`` implementation, telling it how to instantiate the ``Loader (onCreateLoader)`` and providing instructions when its load is complete/reset (``onLoadFinished`` and ``onLoadReset``, respectively). Most often you will implement the callbacks as part of the component itself, by having your Activity or Fragment implement the ``LoaderManager.LoaderCallbacks<D>`` interface:

```java
public class SampleActivity extends Activity implements LoaderManager.LoaderCallbacks<D> {

  public Loader<D> onCreateLoader(int id, Bundle args) { ... }

  public void onLoadFinished(Loader<D> loader, D data) { ... }

  public void onLoaderReset(Loader<D> loader) { ... }

  /* ... */
}
```

Once instantiated, the client passes the callbacks object ("this", in this case) as the third argument to the ``LoaderManager``'s ``initLoader`` method, and will be bound to the ``Loader`` as soon as it is created.

Overall, implementing the callbacks is straightforward. Each callback method serves a specific purpose that makes interacting with the ``LoaderManager`` easy:

+ ``onCreateLoader`` is a factory method that simply returns a new ``Loader``. The ``LoaderManager`` will call this method when it first creates the ``Loader``.

+ ``onLoadFinished`` is called automatically when a ``Loader`` has finished its load. This method is typically where the client will update the application's UI with the loaded data. The client may (and should) assume that new data will be returned to this method each time new data is made available. Remember that it is the Loader's job to monitor the data source and to perform the actual asynchronous loads. The ``LoaderManager`` will receive these loads once they have completed, and then pass the result to the callback object's ``onLoadFinished`` method for the client (i.e. the Activity/Fragment) to use.

+ Lastly, ``onLoadReset`` is called when the Loader's data is about to be reset. This method gives you the opportunity to remove any references to old data that may no longer be available.

####Transitioning from Managed Cursors to the LoaderManager

The code below is similar in behavior to the sample in my previous section. The difference, of course, is that it has been updated to use the ``LoaderManager``. The ``CursorLoader`` ensures that all queries are performed asynchronously, thus guaranteeing that we won't block the UI thread. Further, the ``LoaderManager`` manages the ``CursorLoader`` across the ``Activity`` lifecycle, retaining its data on configuration changes and directing each new data load to the callback's ``onLoadFinished`` method, where the Activity is finally free to make use of the queried ``Cursor``.

```java
public class SampleListActivity extends ListActivity implements
    LoaderManager.LoaderCallbacks<Cursor> {

  private static final String[] PROJECTION = new String[] { "_id", "text_column" };

  // The loader's unique id. Loader ids are specific to the Activity or
  // Fragment in which they reside.
  private static final int LOADER_ID = 1;

  // The callbacks through which we will interact with the LoaderManager.
  private LoaderManager.LoaderCallbacks<Cursor> mCallbacks;

  // The adapter that binds our data to the ListView
  private SimpleCursorAdapter mAdapter;

  @Override
  public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);

    String[] dataColumns = { "text_column" };
    int[] viewIDs = { R.id.text_view };

    // Initialize the adapter. Note that we pass a 'null' Cursor as the
    // third argument. We will pass the adapter a Cursor only when the
    // data has finished loading for the first time (i.e. when the
    // LoaderManager delivers the data to onLoadFinished). Also note
    // that we have passed the '0' flag as the last argument. This
    // prevents the adapter from registering a ContentObserver for the
    // Cursor (the CursorLoader will do this for us!).
    mAdapter = new SimpleCursorAdapter(this, R.layout.list_item,
        null, dataColumns, viewIDs, 0);

    // Associate the (now empty) adapter with the ListView.
    setListAdapter(mAdapter);

    // The Activity (which implements the LoaderCallbacks<Cursor>
    // interface) is the callbacks object through which we will interact
    // with the LoaderManager. The LoaderManager uses this object to
    // instantiate the Loader and to notify the client when data is made
    // available/unavailable.
    mCallbacks = this;

    // Initialize the Loader with id '1' and callbacks 'mCallbacks'.
    // If the loader doesn't already exist, one is created. Otherwise,
    // the already created Loader is reused. In either case, the
    // LoaderManager will manage the Loader across the Activity/Fragment
    // lifecycle, will receive any new loads once they have completed,
    // and will report this new data back to the 'mCallbacks' object.
    LoaderManager lm = getLoaderManager();
    lm.initLoader(LOADER_ID, null, mCallbacks);
  }

  @Override
  public Loader<Cursor> onCreateLoader(int id, Bundle args) {
    // Create a new CursorLoader with the following query parameters.
    return new CursorLoader(SampleListActivity.this, CONTENT_URI,
        PROJECTION, null, null, null);
  }

  @Override
  public void onLoadFinished(Loader<Cursor> loader, Cursor cursor) {
    // A switch-case is useful when dealing with multiple Loaders/IDs
    switch (loader.getId()) {
      case LOADER_ID:
        // The asynchronous load is complete and the data
        // is now available for use. Only now can we associate
        // the queried Cursor with the SimpleCursorAdapter.
        mAdapter.swapCursor(cursor);
        break;
    }
    // The listview now displays the queried data.
  }

  @Override
  public void onLoaderReset(Loader<Cursor> loader) {
    // For whatever reason, the Loader's data is now unavailable.
    // Remove any references to the old data by replacing it with
    // a null Cursor.
    mAdapter.swapCursor(null);
  }
}
```

As its name suggests, the ``LoaderManager`` is responsible for managing Loaders across the Activity/Fragment lifecycle. The ``LoaderManager`` is simple and its implementation usually requires very little code. 

<h3 id="implementing">Implementing Loaders</h3>

This section introduces the ``Loader<D>`` class as well as custom ``Loader`` implementations. 

####Loader Basics

``Loaders`` are responsible for performing queries on a separate thread, monitoring the data source for changes, and delivering new results to a registered listener (usually the ``LoaderManager``) when changes are detected. These characteristics make ``Loaders`` a powerful addition to the Android SDK for several reasons:

+ They encapsulate the actual loading of data. The ``Activity/Fragment`` no longer needs to know how to load data. Instead, the ``Activity/Fragment`` delegates the task to the ``Loader``, which carries out the request behind the scenes and has its results delivered back to the ``Activity/Fragment``.

+ They abstract out the idea of threads from the client. The Activity/Fragment does not need to worry about offloading queries to a separate thread, as the ``Loader`` will do this automatically. This reduces code complexity and eliminates potential thread-related bugs.

+ They are entirely event-driven. ``Loaders`` monitor the underlying data source and automatically perform new loads for up-to-date results when changes are detected. This makes working with Loaders easy, as the client can simply trust that the Loader will auto-update its data on its own. All the Activity/Fragment has to do is initialize the ``Loader`` and respond to any results that might be delivered. Everything in between is done by the Loader.

Loaders are a somewhat advanced topic and may take some time getting used to. We begin by analyzing its four defining characteristics in the next section.

####What Makes Up a Loader?

There are four characteristics which ultimately determine a Loader’s behavior:

+ A task to perform the asynchronous load. To ensure that loads are done on a separate thread, subclasses should extend ``AsyncTaskLoader<D>`` as opposed to the ``Loader<D>`` class. ``AsyncTaskLoader<D>`` is an abstract Loader which provides an ``AsyncTask`` to do its work. When subclassed, implementing the asynchronous task is as simple as implementing the abstract ``loadInBackground()`` method, which is called on a worker thread to perform the data load.

+ A registered listener to receive the Loader's results when it completes a load. For each of its ``Loaders``, the ``LoaderManager`` registers an ``OnLoadCompleteListener<D>`` which will forward the Loader’s delivered results to the client with a call to ``onLoadFinished(Loader<D> loader, D result)``. Loaders should deliver results to these registered listeners with a call to ``Loader#deliverResult(D result)``.You don't need to worry about registering a listener for your Loader unless you plan on using it without the ``LoaderManager``. The ``LoaderManager`` will act as this "listener" and will forward any results that the Loader delivers to the ``LoaderCallbacks#onLoadFinished`` method.

+ One of three distinct states. Any given Loader will either be in a started, stopped, or reset state:

  + ``Loaders`` in a started state execute loads and may deliver their results to the listener at any time. Started Loaders should monitor for changes and perform new loads when changes are detected. Once started, the Loader will remain in a started state until it is either stopped or reset. This is the only state in which ``onLoadFinished`` will ever be called.
  + ``Loaders`` in a stopped state continue to monitor for changes but should not deliver results to the client. From a stopped state, the ``Loader`` may either be started or reset.
  + ``Loaders`` in a reset state should not execute new loads, should not deliver new results, and should not monitor for changes. When a loader enters a reset state, it should invalidate and free any data associated with it for garbage collection (likewise, the client should make sure they remove any references to this data, since it will no longer be available). More often than not, reset Loaders will never be called again; however, in some cases they may be started, so they should be able to start running properly again if necessary.
+ An observer to receive notifications when the data source has changed. ``Loaders`` should implement an observer of some sort (i.e. a ``ContentObserver``, a ``BroadcastReceiver``, etc.) to monitor the underlying data source for changes. When a change is detected, the observer should call ``Loader#onContentChanged()``, which will either (a) force a new load if the Loader is in a started state or, (b) raise a flag indicating that a change has been made so that if the ``Loader`` is ever started again, it will know that it should reload its data.

By now you should have a basic understanding of how Loaders work. If not, I suggest you let it sink in for a bit and come back later to read through once more (reading the [documentation](http://developer.android.com/reference/android/content/Loader.html) never hurts either!). That being said, let’s get our hands dirty with the actual code!

####Implementing the Loader

As I stated earlier, there is a lot that you must keep in mind when implementing your own custom Loaders. Subclasses must implement ``loadInBackground()`` and should override ``onStartLoading()``, ``onStopLoading()``, ``onReset()``, ``onCanceled()``, and ``deliverResult(D results)`` to achieve a fully functioning ``Loader``. Overriding these methods is very important as the ``LoaderManager`` will call them regularly depending on the state of the Activity/Fragment lifecycle. For example, when an Activity is first started, the Activity instructs the ``LoaderManager`` to start each of its ``Loaders`` in ``Activity#onStart()``. If a ``Loader`` is not already started, the ``LoaderManager`` calls ``startLoading()``, which puts the ``Loader`` in a started state and immediately calls the ``Loader’s`` ``onStartLoading()`` method. In other words, a lot of work that the ``LoaderManager`` does behind the scenes relies on the Loader being correctly implemented, so don’t take the task of implementing these methods lightly!

The code below serves as a template of what a ``Loader`` implementation typically looks like. The ``SampleLoader`` queries a list of ``SampleItem`` objects and delivers a ``List<SampleItem>`` to the client:

```java
public class SampleLoader extends AsyncTaskLoader<List<SampleItem>> {

  // We hold a reference to the Loader’s data here.
  private List<SampleItem> mData;

  public SampleLoader(Context ctx) {
    // Loaders may be used across multiple Activitys (assuming they aren't
    // bound to the LoaderManager), so NEVER hold a reference to the context
    // directly. Doing so will cause you to leak an entire Activity's context.
    // The superclass constructor will store a reference to the Application
    // Context instead, and can be retrieved with a call to getContext().
    super(ctx);
  }

  /****************************************************/
  /** (1) A task that performs the asynchronous load **/
  /****************************************************/

  @Override
  public List<SampleItem> loadInBackground() {
    // This method is called on a background thread and should generate a
    // new set of data to be delivered back to the client.
    List<SampleItem> data = new ArrayList<SampleItem>();

    // TODO: Perform the query here and add the results to 'data'.

    return data;
  }

  /********************************************************/
  /** (2) Deliver the results to the registered listener **/
  /********************************************************/

  @Override
  public void deliverResult(List<SampleItem> data) {
    if (isReset()) {
      // The Loader has been reset; ignore the result and invalidate the data.
      releaseResources(data);
      return;
    }

    // Hold a reference to the old data so it doesn't get garbage collected.
    // We must protect it until the new data has been delivered.
    List<SampleItem> oldData = mData;
    mData = data;

    if (isStarted()) {
      // If the Loader is in a started state, deliver the results to the
      // client. The superclass method does this for us.
      super.deliverResult(data);
    }

    // Invalidate the old data as we don't need it any more.
    if (oldData != null && oldData != data) {
      releaseResources(oldData);
    }
  }

  /*********************************************************/
  /** (3) Implement the Loader’s state-dependent behavior **/
  /*********************************************************/

  @Override
  protected void onStartLoading() {
    if (mData != null) {
      // Deliver any previously loaded data immediately.
      deliverResult(mData);
    }

    // Begin monitoring the underlying data source.
    if (mObserver == null) {
      mObserver = new SampleObserver();
      // TODO: register the observer
    }

    if (takeContentChanged() || mData == null) {
      // When the observer detects a change, it should call onContentChanged()
      // on the Loader, which will cause the next call to takeContentChanged()
      // to return true. If this is ever the case (or if the current data is
      // null), we force a new load.
      forceLoad();
    }
  }

  @Override
  protected void onStopLoading() {
    // The Loader is in a stopped state, so we should attempt to cancel the 
    // current load (if there is one).
    cancelLoad();

    // Note that we leave the observer as is. Loaders in a stopped state
    // should still monitor the data source for changes so that the Loader
    // will know to force a new load if it is ever started again.
  }

  @Override
  protected void onReset() {
    // Ensure the loader has been stopped.
    onStopLoading();

    // At this point we can release the resources associated with 'mData'.
    if (mData != null) {
      releaseResources(mData);
      mData = null;
    }

    // The Loader is being reset, so we should stop monitoring for changes.
    if (mObserver != null) {
      // TODO: unregister the observer
      mObserver = null;
    }
  }

  @Override
  public void onCanceled(List<SampleItem> data) {
    // Attempt to cancel the current asynchronous load.
    super.onCanceled(data);

    // The load has been canceled, so we should release the resources
    // associated with 'data'.
    releaseResources(data);
  }

  private void releaseResources(List<SampleItem> data) {
    // For a simple List, there is nothing to do. For something like a Cursor, we 
    // would close it in this method. All resources associated with the Loader
    // should be released here.
  }

  /*********************************************************************/
  /** (4) Observer which receives notifications when the data changes **/
  /*********************************************************************/
 
  // NOTE: Implementing an observer is outside the scope of this post (this example
  // uses a made-up "SampleObserver" to illustrate when/where the observer should 
  // be initialized). 
  
  // The observer could be anything so long as it is able to detect content changes
  // and report them to the loader with a call to onContentChanged(). For example,
  // if you were writing a Loader which loads a list of all installed applications
  // on the device, the observer could be a BroadcastReceiver that listens for the
  // ACTION_PACKAGE_ADDED intent, and calls onContentChanged() on the particular 
  // Loader whenever the receiver detects that a new application has been installed.
  // Please don’t hesitate to leave a comment if you still find this confusing! :)
  private SampleObserver mObserver;
}
```

REF:

+ [Life Before Loaders](http://www.androiddesignpatterns.com/2012/07/loaders-and-loadermanager-background.html)
+ [Understanding the LoaderManager](http://www.androiddesignpatterns.com/2012/07/understanding-loadermanager.html)
+ [Implementing the Loaders](http://www.androiddesignpatterns.com/2012/08/implementing-loaders.html)
+ [Tutorial:AppListLoader](http://www.androiddesignpatterns.com/2012/09/tutorial-loader-loadermanager.html),[AppListLoader](https://github.com/alexjlockwood/AppListLoader)

