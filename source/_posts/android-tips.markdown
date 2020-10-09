---
layout: post
title: "Android Tips"
date: 2014-10-19 13:00
comments: true
categories: 
- dev
- android
tags:
- android
---
<p><center><img src="/images/android_training.jpg"/></center></p>
*	[Android Tips and Tricks](#overview)
	* [Part I](#part1)
	* [Part II](#part2)
	* [Part III](#part3)
	* [Part IV](#part4)
	* [Part V](#part5)
	
<h2 id="overview">Android Tips and Tricks</h2>

<h3 id="part1">Part I</h3>

+ ``Activity.startActivities()`` - Nice for launching to the middle of an app flow.

+ ``TextUtils.isEmpty()`` - Simple utility I use everywhere.

+ ``Html.fromHtml()`` - Quick method for formatting Html. It's not particularly fast so I wouldn't use it constantly (e.g., don't use it just to bold part of a string - construct the Spannable manually instead), but it's fine for rendering text obtained from the web.

+ ``TextView.setError()`` - Nice UI when validating user input.

+ ``Build.VERSION_CODES`` - Not only is it handy for routing code, it's also summarizes behavioral differences between each version of Android.

<!-- more -->

+ ``Log.getStackTraceString()`` - Convenience utility for logging.

+ ``LayoutInflater.from()`` - Wraps the long-winded getSystemService() call in a simple utility.

+ ``ViewConfiguration.getScaledTouchSlop()`` - Using the values provided in ViewConfiguration ensures all touch interaction feels consistent across the OS.

+ ``PhoneNumberUtils.convertKeypadLettersToDigits`` - Makes handling phone number data a snap, as some companies provide them as letters.

+ ``Context.getCacheDir()`` - Use the cache dir for caching data. Simple enough but some don't know it exists.

+ ``ArgbEvaluator`` - Transition from one color to another. As was pointed out by Chris Banes, this class creates a lot of autoboxing churn so it'd be better to just rip out the logic and run it yourself.

+ ``ContextThemeWrapper`` - Nice class for changing the theme of a Context on the fly.

+ ``Space`` - Lightweight View which skips drawing. Great for any situation that might require a placeholder.

+ ``ValueAnimator.reverse()`` - I love this for canceling animations smoothly.

<h3 id="part2">Part II</h3>

+ ``DateUtils.formatDateTime()`` - One-stop shop for localized date/time strings.

+ ``AlarmManager.setInexactRepeating`` - Saves on battery life by grouping multiple alarms together. Even if you're only calling a single alarm this is better (just make sure to call AlarmManager.cancel() when done).

+ ``Formatter.formatFileSize()`` - A localized file size formatter.

+ ``ActionBar.hide()/.show()`` - Animates the action bar hiding/showing. Lets you switch to full-screen gracefully.

+ ``Linkify.addLinks()`` - If you need to control how links are added to text.

+ ``StaticLayout`` - Useful for measuring text that you're about to render into a custom View.

+ ``Activity.onBackPressed()`` - Easy way to manage the back button. While I wouldn't normally hijack back, sometimes it's necessary to make a flow work.

+ ``GestureDetector`` - Listens to motion events and fires listener events for common actions (like clicks, scrolls and flings). So much easier than implementing your own motion event system.

+ ``DrawFilter`` - Lets you manipulate a Canvas even if you're not calling the draw commands. For example, you could create a custom View which sets a DrawFilter which anti-aliases the draws of the parent View.

+ ``ActivityManager.getMemoryClass()`` - Gives you an idea of how much memory the device has. Great for figuring out how large to make your caches.

+ ``SystemClock.sleep()`` - Convenience method which guarantees sleeping the amount of time entered. I use it for debugging and simulating network delays.

+ ``ViewStub`` - A View that initially does nothing, but can later inflate a layout. This is a great placeholder for lazy-loading Views. Its only drawback is that it doesn't support <merge> tags, so it can create unnecessary nesting in the hierarchy if you're not careful.

+ ``DisplayMetrics.density`` - You can get the density of the screen this way. Most of the time you'll be better off letting the system scale dimensions automatically, but occasionally it's useful to have more control (especially with custom Views).

+ ``Pair.create()`` - Handy class, handy creator method.

<h3 id="part3">Part III</h3>

+ ``UrlQuerySanitizer`` - Sanitize URLs with this handy utility.

+ ``Fragment.setArguments`` - Since you can't use a Fragment constructor w/ parameters this is the second best thing. Arguments set before creation last throughout the entire Fragment's lifecycle (even if it's destroyed/recreated due to a configuration change).

+ ``DialogFragment.setShowsDialog()`` - Neat trick - DialogFragments can act like normal Fragments! That way you can have the same Fragment do double-duty. I usually create a third View generation method that both onCreateView() and onCreateDialog() call into when creating a dual-purpose Fragment.

+ ``FragmentManager.enableDebugLogging()`` - Help when you need it when figuring out Fragments.

+ ``LocalBroadcastManager`` - Safer than global broadcasts. Simple and quick. Event buses like otto may make more sense for your use case though.

+ ``PhoneNumberUtils.formatNumber()`` - Let someone else figure out this problem for you.

+ ``Region.op()`` - I found this useful for comparing two generic areas before rendering. If I've got two Paths, do they overlap? I can figure that out with this method.

+ ``Application.registerActivityLifecycleCallbacks`` - Though lacking documentation I feel this is self-evident. Just a handy tool.

+ ``versionNameSuffix`` - This gradle setting lets you modify the versionName field in your manifest based on different build types. For example, I would setup my debug build type to end in "-SNAPSHOT"; that way you can easily tell if you're on a debug build or release build.

+ ``CursorJoiner`` - If you're using a single database then a join in SQL is the natural solution, but what if you've received data from two separate ContentProviders? In that case CursorJoiner can be helpful.

+ ``Genymotion`` - A much faster Android emulator. I use it all day.

+ ``-nodpi`` - Most qualifiers (-mdpi, -hdpi, -xhdpi, etc.) automatically scale assets/dimensions if you're on a device that isn't explicitly defined. Sometimes you just want something consistent though; in that case use -nodpi.

+ ``BroadcastRecevier.setDebugUnregister()`` - Another handy debugging tool.

+ ``Activity.recreate()`` - Forces an Activity to recreate itself for whatever reason.

+ ``PackageManager.checkSignatures()`` - You can use this to find out if two apps (presumably your own) are installed at the same time. Without checking signatures someone could imitate your app easily by just using the same package name.

<h3 id="part4">Part IV</h3>

+ ``Activity.isChangingConfigurations()`` - Often times you don't need to do quite as much saving of state if all that's happening is the configuration is changing.

+ ``SearchRecentSuggestionsProvider`` - A quick and easy way to create a recents suggestion provider.

+ ``ViewTreeObserver`` - This is an amazing utility; it can be grabbed from any View and used to monitor the state of the View hierarchy. My most often use for it is to determine when Views have been measured (usually for animation purposes).

+ ``org.gradle.daemon=true`` - Helps reduce the startup time of of Gradle builds. Only really applies to command-line builds as Android Studio already tries to use the daemon.

+ ``DatabaseUtils`` - A variety of useful tools for database operations.

+ ``android:weightSum`` (LinearLayout) - Want to use layout weights, but don't want them to fill the entire LinearLayout? That's what weightSum can do by defining the total weight.

+ ``android:duplicateParentState`` (View) - Makes the child duplicate the state of the parent - for example, if you've got a ViewGroup that is clickable, then you can use this to make its children change state when it is clicked.

+ ``android:clipChildren`` (ViewGroup) - If disabled, this lets the children of a ViewGroup draw outside their parent's bounds. Great for animations.

+ ``android:fillViewport`` (ScrollView) - Best explained in this post, this helps solve a problem with ScrollViews that may not always have enough content to actually fill the height of the screen.

+ ``android:tileMode`` (BitmapDrawable) - Lets you create repeated patterns with images.

+ ``android:enterFadeDuration/android:exitFadeDuration`` (Drawables) - For Drawables that have multiple states, this lets you define a fade before/after the drawable shows.

+ ``android:scaleType`` (ImageView) - Defines how to scale/crop a drawable within an ImageView. "centerCrop" and "centerInside" are regular settings for me.

+ ``<merge>`` - Lets you include a layout in another without creating a duplicate ViewGroup (more info). Also good for custom ViewGroups; you can inflate a layout with <merge> inside the constructor to define its children automatically.

+ ``AtomicFile`` - Manipulates a file atomically by using a backup file. I've written this myself before, it's good to have an official (and better-written) version of it.

<h3 id="part5">Part V</h3>

+ ``ViewDragHelper`` - Dragging Views is a complex problem and this class helps a lot. If you want an example, DrawerLayout uses it for swiping. Flavient Laurent also wrote an excellent article about it.

+ ``PopupWindow`` - Used all around Android without you even realizing it (action bars, autocomplete, edittext errors), this class is the primary method for creating floating content.

+ ``ActionBar.getThemedContext()`` - ActionBar theming is surprisingly complex (and can be different from the theming of the rest of the Activity). This gets you a Context so if you create your own Views they will be properly themed.

+ ``ThumbnailUtils`` - Helps create thumbnails; in general I'd just use whatever image library was already in place (e.g. Picasso or Volley), but it can also create video thumbnails!

+ ``Context.getExternalFilesDir()`` - While you do have permission to write anywhere on the SD card if you ask for it, it's much more polite to write your data in the correct designated folder. That way it gets cleaned up and users get a common experience. Additionally, as of Kit Kat you can write to this folder without permission, and each user has their own external files dir.

+ ``SparseArray`` - A more efficient version of Map<Integer, Object>. Be sure to check out sister classes SparseBooleanArray, SparseIntArray and SparseLongArray as well.

+ ``PackageManager.setComponentEnabledSetting()`` - Lets you enable/disable components in your app's manifest. What's nice here is being able to shut off unnecessary functionality - for example, a BroadcastReceiver that is unnecessary due to the current app configuration.

+ ``SQLiteDatabase.yieldIfContendedSafely()`` - Lets you temporarily stop a db transaction so you don't tie up too much of the system.

+ ``Environment.getExternalStoragePublicDirectory()`` - Again, users like a consistent experience with their SD card; using this method will grab the correct directory for placing typed files (music, pictures, etc.) on their drive.

+ ``View.generateViewId()`` - Every once in a while I've wanted to dynamically generate view IDs. The problem is ensuring you aren't clobbering existing IDs (or other generated ones).

+ ``ActivityManager.clearApplicationUserData()`` - A reset button for your app. Perhaps the easiest way to log out a user, ever.

+ ``Context.createConfigurationContext()`` - Customize your configuration context. Common problem I've run into: forcing part of an app to render in a particular locale (not that I normally condone this sort of behavior, but you never know). This would make it a lot easier to do so.

+ ``ActivityOptions`` - Nice custom animations when moving between Activities. ActivityOptionsCompat is good for backwards compatible functionality.

+ ``AdapterViewFlipper.fyiWillBeAdvancedByHostKThx()`` - Because it's funny and for no other reason. There are other amusing tidbits in AOSP (like GRAVITY_DEATH_STAR_I) but unlike those this one is actually useful.

+ ``ViewParent.requestDisallowInterceptTouchEvent()`` - The Android touch event system defaults handle what you want most of the time, but sometimes you need this method to wrest event control from parents. (By the way, if you want to know about the touch system, this talk is amazing.)
