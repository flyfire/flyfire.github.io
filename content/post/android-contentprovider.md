+++
title = "Android Contentprovider"
date = "2014-10-24"
slug = "2014/10/24/android-contentprovider"
Categories = ["dev", "android"]
+++
<center><p><img src="/images/android_logo.jpg"/></p></center>

*	[Android ContentProvider](#overview)
	* [Content Provider Basics](#basics)
	* [Using Content Providers](#using)
	* [Writing your own Content Provider](#writing)
	* [Better Performance with ContentProviderOperation](#performance)

<h2 id="overview">Android ContentProvider</h2>

This is a tutorial covering Android's ContentProvider.

<!-- more -->


<h3 id="basics">ContentProvider Basics</h3>

#### What are content providers?

Content providers are Android’s central mechanism that enables you to access data of other applications – mostly information stored in databases or flat files. As such content providers are one of Android’s central component types to support the modular approach common to Android. Without content providers accessing data of other apps would be a mess.

Content providers support the four basic operations, normally called CRUD-operations. CRUD is the acronym for ``create``, ``read``, ``update`` and ``delete``. With content providers those objects simply represent data – most often a record (tuple) of a database – but they could also be a photo on your SD-card or a video on the web.Android provides some standard content providers to access contacts, media files, preferences and so on. 

#### Content URIs

The most important concept to understand when dealing with content providers is the content URI. Whenever you want to access data from a content provider you have to specify a URI. URIs for content providers look like this:

```bash
content://authority/optionalPath/optionalId
```

They contain four parts: The scheme to use, an authority, an optional path and an optional id.

+ The scheme for content providers is always “content”. The colon and double-slash “://” are a fixed part of the URI-RFC and separate the scheme from the authority.
+ The next part is the authority for the content provider. Authorities have to be unique for every content provider. Thus the naming conventions should follow the Java package name rules. That is you should use the reversed domain name of your organization plus a qualifier for each and every content provider you publish. The Android documentation recommends to use the fully qualified class name of your ContentProvider-subclass.
+ The third part, the optional path, is used to distinguish the kinds of data your content provider offers. The content provider for Android’s mediastore, for example, distinguishes between audio files, video files and images using different paths for each of these types of media. This way a content provider can support different types of data that are nevertheless related. For totally unrelated data though you should use different content providers – and thus different authorities.
+ The last element is the optional id, which – if present – must be numeric. The id is used whenever you want to access a single record (e.g. a specific video file).

There are two types of URIs: ``directory-based`` and ``id-based`` URIs. If no id is specified a URI is automatically a directory-based URI.

+ You use ``directory-based`` URIs to access multiple elements of the same type (e.g. all songs of a band). All CRUD-operations are possible with directory-based URIs.
+ You use ``id-based`` URIs if you want to access a specific element. You cannot create objects using an id-based URI – but reading, updating and deleting is possible.

The path of content URIs can contain additional information to limit the scope. The ``MediaStore`` content provider for example distinguishes between audio and other types. In addition to this it offers URIs that limit its operations to albums only or others to genres only. Content providers normally have constants for the URIs they support.

#### Content Types

Besides URIs another important concept to understand is the use of content types. Content types also have a standardized format which was first defined in RFC 1049 and refined in RFC 2045.

A content type consist of a media type and a subtype divided by a slash. A typical example is “image/png”. The media type “image” describes the content as an image file which is further specified to be of the Portable Network Graphic variety by the subtype “png”.

As with URIs there is also a standard for content types in Android. Table below lists the only two media types that Android accepts for content providers. As you can see, those two media types match the two types of URIs mentioned above.

<table>
<caption><center>The media types used for content providers</center></caption>
<tr>
<th>Type</th>
<th>Usage</th>
<th>Constant</th>
</tr>
<tr>
<td>vnd.android.cursor.item</td>
<td>Used for single records</td>
<td>ContentResolver.CURSOR_ITEM_BASE_TYPE</td>
</tr>
<tr>
<td>vnd.android.cursor.dir</td>
<td>Used for multiple records</td>
<td>ContentResolver.CURSOR_DIR_BASE_TYPE</td>
</tr>
</table>

The subtype on the other hand is used for content provider specific details and should differ for all types your content provider supports. The naming convention for subtypes is vnd.yourcompanyname.contenttype. Most content providers support multiple subtypes. In the case of a media player for example you might have subtypes for genre, band, titles, musicians and so on.

#### Which standard Content Providers are available?

A number of content providers are part of Android’s API. All these standard providers are defined in the package ``android.provider``. The following table lists the standard providers and what they are used for.

<table>
<caption>The standard content providers of Android</caption>
<tr>
<th>Provider</th>
<th>Since&nbsp;&nbsp;&nbsp;&nbsp;</th>
<th>Usage</th>
</tr>
<tr>
<td>Browser</td>
<td>SDK 1</td>
<td>Manages your web-searches, bookmarks and browsing-history.</td>
</tr>
<tr>
<td>CalendarContract</td>
<td>SDK 14</td>
<td>Manages the calendars on the user&#8217;s device.</td>
</tr>
<tr>
<td>CallLog</td>
<td>SDK 1</td>
<td>Keeps track of your call history.</td>
</tr>
<tr>
<td>Contacts</td>
<td>SDK 1</td>
<td>The old and deprecated content provider for managing contacts. You should only use this provider if you need to support an SDK prior to SDK 5!</td>
</tr>
<tr>
<td>ContactsContract</td>
<td>SDK 5</td>
<td>Deals with all aspects of contact management. Supersedes the Contacts-content provider.</td>
</tr>
<tr>
<td>MediaStore</td>
<td>SDK 1</td>
<td>The content provider responsible for all your media files like music, video and pictures.</td>
</tr>
<tr>
<td>Settings</td>
<td>SDK 1</td>
<td>Manages all global settings of your device.</td>
</tr>
<tr>
<td>UserDictionary</td>
<td>SDK 3</td>
<td>Keeps track of words you add to the default dictionary.</td>
</tr>
</table>

Please make use of the standard providers whenever you need data they provide. For example there are apps that ignore the UserDictionary (e.g. Swype). So the user might end up adding the same words in multiple apps – which is pretty annoying. Something like this will not help your rating in Android’s Market.

Of course you should always keep in mind that not all of the standard providers might be present on certain devices. E.g. a tablet might have no CallLog. Thus you should always test for availability. **One way to do so is querying a content provider and checking if the returned cursor is null**. That’s the return value when the provider doesn’t exist. The other CRUD-methods though throw an exception in case you pass in an unknown URI.

<h3 id="using">Using ContentProviders</h3>

#### ContentResolver

Whenever you want to use another content provider you first have to access a ``ContentResolver`` object. This object is responsible for finding the correct content provider.

The ``ContentResolver`` decides which provider to use based on the authority part of the URI. **A content provider must provide its authority within the manifest file.** From these entries Android creates a mapping between the authorities and the corresponding ContentProvider implementations to use.

You always interact with the ``ContentResolver`` and never with ``ContentProvider`` objects themselves. This supports loose coupling and also guarantees the correct lifecycle of the content provider.

You can get the ``ContentResolver`` object by calling ``getContentResolver()`` on the ``Context`` object. The ``Context`` object should always be available since the Activity and Service classes inherit from Context and the other components also provide easy access to it.

Since you only use the class ``ContentResolver`` it has to provide all necessary CRUD-methods. Any arguments provided to the methods of the ``ContentResolver`` are passed on to the respective methods of the ``ContentProvider`` subclass.

<table>
<caption>The CRUD methods of the ContentResolver object</caption>
<tr>
<th>Method</th>
<th>Usage</th>
</tr>
<tr>
<td>delete</td>
<td>Deletes the object(s) for the URI provided. The URI can be item- or directory-based</td>
</tr>
<tr>
<td>insert</td>
<td>Inserts one object. The URI must be directory-based</td>
</tr>
<tr>
<td>query</td>
<td>Queries for all objects that fit the URI. The URI can be item- or directory-based</td>
</tr>
<tr>
<td>update</td>
<td>Updates one or all object(s). The URI can be item- or directory-based</td>
</tr>
</table>


There are also two methods for applying multiple CRUD operations at once as shown in the next table.

<table>
<caption>ContentResolver methods for dealing with multiple content provider operations</caption>
<tr>
<th>Method</th>
<th>Usage</th>
</tr>
<tr>
<td>applyBatch</td>
<td>Allows you to execute a list of <code>ContentProviderOperation</code> objects. Each <code>ContentProviderOperation</code> object can take its own URI and type of operation</td>
</tr>
<tr>
<td>bulkInsert</td>
<td>Allows you to insert an array of <code>ContentValues</code> for a directory-based URI. You can only specify one URI for all objects you want to add</td>
</tr>
</table>

#### Querying for data

Querying data is probably the operation you will use most often. That’s true for your own providers but also for standard providers which offer some very valuable information.

<table>
<caption>The arguments of the query method</caption>
<tr>
<th>Type</th>
<th>Name</th>
<th>Usage</th>
</tr>
<tr>
<td>URI</td>
<td>uri</td>
<td>The URI of the object(s) to access. This is the only argument that must not be null</td>
</tr>
<tr>
<td>String[]</td>
<td>projection</td>
<td>This String array indicates which columns/attributes of the objects you want to access</td>
</tr>
<tr>
<td>String</td>
<td>selection</td>
<td>With this argument you can determine which records to return</td>
</tr>
<tr>
<td>String[]</td>
<td>selectionArgs</td>
<td>The binding parameters to the previous selection argument</td>
</tr>
<tr>
<td>String</td>
<td>sortOrder</td>
<td>If the result should be ordered you must use this argument to determine the sort order</td>
</tr>
</table>

The return value of the query method is a ``Cursor`` object. The cursor is used to navigate between the rows of the result and to read the columns of the current row. Cursors are important resources that have to be closed whenever you have no more use for them – otherwise you keep valuable resources from being released.

The following code snippet shows how to make use of this provider. I use the ``CONTENT_URI`` of ``UserDictionary.Words`` to access the dictionary. For this example I am only interested in the IDs of the words and the words itself.


```java
ContentResolver resolver = getContentResolver();
String[] projection = new String[]{BaseColumns._ID, UserDictionary.Words.WORD};
Cursor cursor = 
      resolver.query(UserDictionary.Words.CONTENT_URI, 
            projection, 
            null, 
            null, 
            null);
if (cursor.moveToFirst()) {
   do {
      long id = cursor.getLong(0);
      String word = cursor.getString(1);
      // do something meaningful
   } while (cursor.moveToNext());
}
```

#### Inserting new records

Very often your app needs to insert data. For the built-in content providers of Android this could be because you want to add events to the Calendar provider, people to the Contacts provider, words to the UserDictionary provider and so on.

The correct content URI for inserts can only be a ``directory-based`` URI because only these represent a collection of related items.

The values to insert are specified using a ``ContentValues`` object.This object is not much more than a collection of key/value pairs. Of course, the keys of your ContentValues object must match columns/attributes of the objects you want to update – otherwise you will get an exception. For all columns of the new object for which no key/value-pair is provided the default value is used – which most often is null.

<table>
<caption>The arguments of the insert method</caption>
<tr>
<th>Type</th>
<th>Name</th>
<th>Usage</th>
</tr>
<tr>
<td>URI</td>
<td>uri</td>
<td>The directory-based URI to which to add the object. This argument must not be null</td>
</tr>
<tr>
<td>ContentValues</td>
<td>values</td>
<td>The values for the object to add. This argument also must not be null</td>
</tr>
</table>

The following code snippet shows you how to add data using a content provider:

```java
ContentValues values = new ContentValues();
values.put(Words.WORD, "Beeblebrox");
resolver.insert(UserDictionary.Words.CONTENT_URI, values);
```

If you want to add multiple records to the same URI you can use the ``bulkInsert()`` method. This method differs from the normal ``insert()`` method only in that it takes an array of ``ContentValue`` objects instead of just one ``ContentValues`` object. So for each record you want to add, there must be an entry within the ``ContentValues`` array. If you want to add to different URIs though – or if you want to mix insert, update and delete operations, you should use ``applyBatch()``.

#### Updating data

To update records you basically provide the URI, a ``ContentValues`` object and optionally also a where-clause and arguments for this where-clause.

<table>
<caption>The arguments of the update method</caption>
<tr>
<th>Type</th>
<th>Name</th>
<th>Usage</th>
</tr>
<tr>
<td>URI</td>
<td>uri</td>
<td>The URI of the object(s) to access. This argument must not be null</td>
</tr>
<tr>
<td>ContentValues</td>
<td>values</td>
<td>The values to substitute the current data with. This argument also must not be null</td>
</tr>
<tr>
<td>String</td>
<td>selection</td>
<td>With this argument you can determine which records to update</td>
</tr>
<tr>
<td>String[]</td>
<td>selectionArgs</td>
<td>The binding parameters to the previous selection argument</td>
</tr>
</table>

In the next snippet I am going to change the word I’ve just added in the previous section.

```java
values.clear();
values.put(Words.WORD, "Zaphod");
Uri uri = ContentUris.withAppendedId(Words.CONTENT_URI, id);
long noUpdated = resolver.update(uri, values, null, null);
```

Here we use a ContentValues object again. The keys of your ContentValues object must of course match columns/attributes of the objects you want to update – otherwise you would get an exception. The update method changes only those columns for which keys are present in the ContentValues object.

Note the call to ``ContentUris.withAppendedId()``. This is a helper method to create an ``id-based`` URI from a directory-based one. You use it all the time since content providers only provide constants for directory-based URIs. So whenever you want to access a specific object you should use this method.

Since I changed only one record, a URI with an appended ID is sufficient. But if you want to update multiple values, you should use the normal URI and a selection clause. You will see an example for the latter when I show you how to delete entries.

There is also the call to ``values.clear()``. This resets the ContentValues object and thus recycles the object. This way you are reducing costly garbage collector operations.

#### Deleting data

The next snippet shows how to delete records. It finally deletes the word. In this code sample you can see how to use the selection and selectionArgs arguments. The array of the selectionArgs argument is used to substitute all question marks found in the selection argument.

```java
long noDeleted = resolver.delete
      (Words.CONTENT_URI, 
      Words.WORD + " = ? ", 
      new String[]{"Zaphod"});
```

The delete method takes the same arguments as the update method with the exception being the values argument. Since the record is deleted anyway, substitute values are not needed.

<table>
<caption>The arguments of the delete method</caption>
<tr>
<th>Type</th>
<th>Name</th>
<th>Usage</th>
</tr>
<tr>
<td>URI</td>
<td>uri</td>
<td>The URI of the object(s) to access. This is the only argument which must not be null</td>
</tr>
<tr>
<td>String</td>
<td>selection</td>
<td>With this argument you can determine which records to delete</td>
</tr>
<tr>
<td>String[]</td>
<td>selectionArgs</td>
<td>The binding parameters to the previous selection argument</td>
</tr>
</table>

<h3 id="writing">Writing your own ContentProvider</h3>

Implementing a content provider involves always the following steps:

- Create a class that extends ``ContentProvider``
- Create a contract class
- Create the ``UriMatcher`` definition
- Implement the ``onCreate()`` method
- Implement the ``getType()`` method
- Implement the CRUD methods
- Add the content provider to your ``AndroidManifest.xml``

#### Create a class that extends ContentProvider

You start by sub-classing ContentProvider. Since ContentProvider is an abstract class you have to implement the six abstract methods. These methods are explained in detail later on, for now simply use the stubs created by the IDE of your choice.

<table>
<caption>The abstract methods you have to implement</caption>
<tr>
<th>Method</th>
<th>Usage</th>
</tr>
<tr>
<td>onCreate()</td>
<td>Prepares the content provider</td>
</tr>
<tr>
<td>getType(Uri)</td>
<td>Returns the MIME type for this URI</td>
</tr>
<tr>
<td>delete(Uri uri, String selection, String[] selectionArgs)</td>
<td>Deletes records</td>
</tr>
<tr>
<td>insert(Uri uri, ContentValues values)</td>
<td>Adds records</td>
</tr>
<tr>
<td>query(Uri uri, String[] projection, String selection, String[] selectionArgs, String sortOrder)</td>
<td>Return records based on selection criteria</td>
</tr>
<tr>
<td>update(Uri uri, ContentValues values, String selection, String[] selectionArgs)</td>
<td>Modifies data</td>
</tr>
</table>

#### Create a contract class

Up to now your code is still missing most of the functionality. But before implementing the CRUD methods you should think about your role as a provider. Content providers by its very definition provide data to clients. Those clients need to know how to access your data. And you should treat your URIs and authority like an API. You basically enter into a contract with your client. And your public API should reflect this.

Thus the official Android documentation recommends to create a contract class. This class defines all publicly available elements, like the authority, the content URIs of your tables, the columns, the content types and also any intents your app offers in addition to your provider.

This class is your public API. What you define here is what clients can use. It’s also the abstraction you provide. You can do behind the scenes whatever you like. The client won’t notice. You can change the data structure without problems – if your contract class remains unchanged.

The downside is: You shouldn’t change the contract in any way that might break existing clients. It’s a contract after all :-)

If you really think you must get rid of something you provided earlier on, use analytics to find out when a deprecated feature isn’t used anymore.

So here is what a typical contract class looks like:

```java
public final class LentItemsContract {

	/**
	 * The authority of the lentitems provider.
	 */
	public static final String AUTHORITY = 
	      "de.openminds.samples.cpsample.lentitems";
	/**
	 * The content URI for the top-level 
	 * lentitems authority.
	 */
	public static final Uri CONTENT_URI = 
	      Uri.parse("content://" + AUTHORITY);
	
	/**
	 * Constants for the Items table 
	 * of the lentitems provider.
	 */
	public static final class Items 
	      implements CommonColumns { ... }
	
	/**
	 * Constants for the Photos table of the 
	 * lentitems provider. For each item there 
	 * is exactly one photo. You can 
	 * safely call insert with the an already 
	 * existing ITEMS_ID. You won't get constraint 
	 * violations. The content provider takes care 
	 * of this.<br> 
	 * Note: The _ID of the new record in this case
	 * differs from the _ID of the old record.
	 */
	public static final class Photos 
	      implements BaseColumns { ... }

	/**
	 * Constants for a joined view of Items and 
	 * Photos. The _id of this joined view is 
	 * the _id of the Items table.
	 */
	public static final class ItemEntities 
	      implements CommonColumns { ...}
	
	/**
	 * This interface defines common columns 
	 * found in multiple tables.
	 */
	public static interface CommonColumns 
	      extends BaseColumns { ... }
}
```

You can see the exported tables Photos, Items and ItemEntities which are separate inner classes of the contract class. There is also the authority of the provider and the root content URI. If your app also exports activities accessible via intents you should document those here as well. And you should document any permissions your provider uses. 

One thing in the above snippet is noteworthy: The inner class ItemEntities represents a virtual table that doesn’t exist in the database. In this case I simply use joins in the query method but you could also use views within the database to back up virtual tables. Clients cannot join tables of content providers, thus you should consider offering plausible joins yourself.

The following snippet shows what to do within the inner classes of your contract class:

```java
/**
 * Constants for the Items table 
 * of the lentitems provider.
 */
public static final class Items 
      implements CommonColumns {
	/**
	 * The content URI for this table. 
	 */
	public static final Uri CONTENT_URI =
	      Uri.withAppendedPath(
	            LentItemsContract.CONTENT_URI, 
	            "items");
	/**
	 * The mime type of a directory of items.
	 */
	public static final String CONTENT_TYPE = 
	      ContentResolver.CURSOR_DIR_BASE_TYPE + 
	      "/vnd.de.openminds.lentitems_items";
	/**
	 * The mime type of a single item.
	 */
	public static final String CONTENT_ITEM_TYPE = 
	      ContentResolver.CURSOR_ITEM_BASE_TYPE + 
	      "/vnd.de.openminds.lentitems_items";
	/**
	 * A projection of all columns 
	 * in the items table.
	 */
	public static final String[] PROJECTION_ALL =
	      {_ID, NAME, BORROWER};
	/**
	 * The default sort order for 
	 * queries containing NAME fields.
	 */
	public static final String SORT_ORDER_DEFAULT = 
	      NAME + " ASC";
}
```

As you can see the inner classes are the place for any column definitions, the content URIs and the content types of the respective tables.

#### Create the UriMatcher definitions

To deal with multiple URIs Android provides the helper class UriMatcher. This class eases the parsing of URIs. In the next code sample you can see that you initialize the UriMatcher by adding a set of paths with correspondig int values. Whenever you ask if a URI matches, the UriMatcher returns the corresponding int value to indicate which one matches. It is common practise to use constants for these int values. The code sample shows how to add id based paths as well as dir based paths.

```java
// helper constants for use with the UriMatcher
private static final int ITEM_LIST = 1;
private static final int ITEM_ID = 2;
private static final int PHOTO_LIST = 5;
private static final int PHOTO_ID = 6;
private static final int ENTITY_LIST = 10;
private static final int ENTITY_ID = 11;
private static final UriMatcher URI_MATCHER;

// prepare the UriMatcher
static {
	URI_MATCHER = new UriMatcher(UriMatcher.NO_MATCH);
	URI_MATCHER.addURI(LentItemsContract.AUTHORITY, 
	      "items", 
	      ITEM_LIST);
	URI_MATCHER.addURI(LentItemsContract.AUTHORITY, 
	      "items/#", 
	      ITEM_ID);
	URI_MATCHER.addURI(LentItemsContract.AUTHORITY, 
	      "photos", 
	      PHOTO_LIST);
	URI_MATCHER.addURI(LentItemsContract.AUTHORITY, 
	      "photos/#", 
	      PHOTO_ID);
   URI_MATCHER.addURI(LentItemsContract.AUTHORITY, 
         "entities", 
         ENTITY_LIST);
   URI_MATCHER.addURI(LentItemsContract.AUTHORITY, 
         "entities/#", 
         ENTITY_ID);
}
```

You pass the authority, a path pattern and an int value to the addURI() method. Android returns the int value later on when you try to match patterns.

The patterns of the sample above are the most common patterns. But you are not limited to those. Android’s calendar content provider for examples offers search URIs to find certain instances of events. And the content provider for contacts also offers non standard URIs – for example to provide access to the contact photo. You can take a look at the source of CalendarProvider2 or ContactsProvider2 to see how to use those non-standard URIs with the UriMatcher.

#### Implement the onCreate() method

Back to the actual content provider class. First implement the onCreate() method.

The onCreate() method is a lifecycle method and runs on the UI thread. Thus you should avoid executing any long-lasting tasks in this method. Your content provider is usually created at the start of your app. And you want this to be as fast as possible. So even if your users do not get any “Application Not Responding” error messages, they won’t like anything that delays the perceived starting time of your app. Thus consider to do any long lasting stuff within the CRUD methods.

Normally content providers use a database as the underlying data store. In this case you would create a reference to your SQLiteOpenHelper in onCreate() – but you wouldn’t try to get hold of an SQLiteDatabase object. In the original version of this post, my recommendation has been to get a reference to the database in this method. Please, do not do this!

```java
public class LentItemsProvider extends ContentProvider {

	private LentItemsOpenHelper mHelper = null;
	@Override
	public boolean onCreate() {
		mHelper = new LentItemsOpenHelper(getContext());
		return true;
	}

   //...
}
```

#### Implement the getType() method

Every content provider must return the content type for its supported URIs. The signature of the method takes a URI and returns a String. The next code sample shows the getType() method of the sample application.

```java
@Override
public String getType(Uri uri) {
   switch (URI_MATCHER.match(uri)) {
   case ITEM_LIST:
      return Items.CONTENT_TYPE;
   case ITEM_ID:
      return Items.CONTENT_ITEM_TYPE;
   case PHOTO_ID:
      return Photos.CONTENT_PHOTO_TYPE;
   case PHOTO_LIST:
      return Photos.CONTENT_TYPE;
   case ENTITY_ID:
      return ItemEntities.CONTENT_ENTITY_TYPE;
   case ENTITY_LIST:
      return ItemEntities.CONTENT_TYPE;
   default:
      throw new IllegalArgumentException("Unsupported URI: " + uri);
   }
}
```

As you can see this method is pretty simple. You just have to return the appropriate content type – defined within your contract class – for the URI passed into this method.

#### UriMatcher

The previous code sample shows how to make use of the UriMatcher. The pattern is also repeated within each of the CRUD methods, so let’s digg into it.

When you initialize the UriMatcher you state for each URI which int value belongs to it. Now whenever you need to react diffently depending on the URI you use the UriMatcher. It’s match() method returns the int value used during initialization. And usually you use this within a switch statement. This switch statement has case branches for the constants used during initialization.

You can use a “#” as a placeholder for an arbitrary numeric value and a “*” as a placeholder for arbitrary text. All other parts must be exactly as passed to the addURI() method.

#### Adding records using insert()

As expected this method is used by your clients to insert records into your datastore. The method only makes sense for dir based URIs, thus you first have to check if the right kind of URI is passed to the insert() method. Only then can you actually insert the values into the datastore you use.

The content provider API is record-based – probably since most underlying datastores are record based databases anyway. This makes implementing the insert() method very easy, since it allows you to simply pass the ContentValues object on to the SQLiteDatabase’s insert() method.

```java
public Uri insert(Uri uri, ContentValues values) {
   if (URI_MATCHER.match(uri) != ITEM_LIST
         && URI_MATCHER.match(uri) != PHOTO_LIST) {
         throw new IllegalArgumentException(
               "Unsupported URI for insertion: " + uri);
   }
   SQLiteDatabase db = mHelper.getWritableDatabase();
   if (URI_MATCHER.match(uri) == ITEM_LIST) {
      long id = 
            db.insert(
                  DBSchema.TBL_ITEMS, 
                  null, 
                  values);
      return getUriForId(id, uri);
   } else {
      // this insertWithOnConflict is a special case; 
      // CONFLICT_REPLACE means that an existing entry 
      // which violates the UNIQUE constraint on the 
      // item_id column gets deleted. In this case this 
      // INSERT behaves nearly like an UPDATE. Though 
      // the new row has a new primary key.
      // See how I mentioned this in the Contract class.
      long id = 
            db.insertWithOnConflict(
                  DBSchema.TBL_PHOTOS, 
                  null, 
                  values, 
                  SQLiteDatabase.CONFLICT_REPLACE);
      return getUriForId(id, uri);
   }
}

private Uri getUriForId(long id, Uri uri) {
   if (id > 0) {
      Uri itemUri = ContentUris.withAppendedId(uri, id);
      if (!isInBatchMode()) {
         // notify all listeners of changes:
         getContext().
               getContentResolver().
                     notifyChange(itemUri, null);
      }
      return itemUri;
   }
   // s.th. went wrong:
   throw new SQLException(
         "Problem while inserting into uri: " + uri);
}
```

#### Notifying listeners of dataset changes

Clients often want to be notified about changes in the underlying datastore of your content provider. So inserting data, as well as deleting or updating data should trigger this notification.

That’s why I used the following line in the code sample above:

```java
getContext().getContentResolver().notifyChange(itemUri, null); 
```

Of course you should only call this method when there really has been a change – that’s why I test if the id is a positive number.

Alas, it is not possible to notify the client of the actual change that has occurred. You can only state which URI has changed. Most often this is enough – though sometimes a client would like to know if this was an addition of a record, a deletion of entries or an update. Sadly, Android offers us no possibility to help those clients 

#### Querying the content provider for records

Returning records from your ContentProvider is pretty easy. Android has a helper class, you can use here: The SQLiteQueryBuilder. Within the query() method of a content provider you can use this class as shown below.

```java
@Override
public Cursor query(Uri uri, String[] projection,
      String selection, String[] selectionArgs, 
      String sortOrder) {
   SQLiteDatabase db = mHelper.getReadableDatabase();
   SQLiteQueryBuilder builder = new SQLiteQueryBuilder();
   boolean useAuthorityUri = false;
   switch (URI_MATCHER.match(uri)) {
   case ITEM_LIST:
      builder.setTables(DBSchema.TBL_ITEMS);
      if (TextUtils.isEmpty(sortOrder)) {
         sortOrder = Items.SORT_ORDER_DEFAULT;
      }
      break;
   case ITEM_ID:
      builder.setTables(DBSchema.TBL_ITEMS);
      // limit query to one row at most:
      builder.appendWhere(Items._ID + " = " +
            uri.getLastPathSegment());
      break;
   case PHOTO_LIST:
      builder.setTables(DBSchema.TBL_PHOTOS);
      break;
   case PHOTO_ID:
      builder.setTables(DBSchema.TBL_PHOTOS);
      // limit query to one row at most:
      builder.appendWhere(Photos._ID + 
            " = " + 
            uri.getLastPathSegment());
      break;
   case ENTITY_LIST:
      builder.setTables(DBSchema.LEFT_OUTER_JOIN_STATEMENT);
      if (TextUtils.isEmpty(sortOrder)) {
         sortOrder = ItemEntities.SORT_ORDER_DEFAULT;
      }
      useAuthorityUri = true;
      break;
   case ENTITY_ID:
      builder.setTables(DBSchema.LEFT_OUTER_JOIN_STATEMENT);
      // limit query to one row at most:
      builder.appendWhere(DBSchema.TBL_ITEMS + 
            "." + 
            Items._ID + 
            " = " +
            uri.getLastPathSegment());
      useAuthorityUri = true;
      break;
   default:
      throw new IllegalArgumentException(
            "Unsupported URI: " + uri);
   }
   Cursor cursor = 
         builder.query(
         db, 
         projection, 
         selection, 
         selectionArgs,
         null, 
         null, 
         sortOrder);
   // if we want to be notified of any changes:
   if (useAuthorityUri) {
      cursor.setNotificationUri(
            getContext().getContentResolver(), 
            LentItemsContract.CONTENT_URI);
   }
   else {
      cursor.setNotificationUri(
            getContext().getContentResolver(), 
            uri);
   }
   return cursor;
}
```

After creating a new object you first define into which table(s) to insert. Then you might want to add a default sort order if none is specified by the caller of your code. Next you have to add a WHERE clause that matches the id for id based queries before you finally pass all arguments on to the builder’s query() method. This method returns a Cursor object which you simply return to your caller.

There is one additional thing though you have to take care of. That is the SQLiteDatabase object you have to pass to the SQLiteQueryBuilder's query() method. I recommend to get access to the SQLiteDatabase object within each CRUD method.

Now you might wonder if I ever close the object. Well, I don’t :-) But this is no problem. SQLiteOpenHelper keeps just one SQLiteDatabase object per SQLiteSession. This makes accessing the object very efficient. With version checking and all that is happening in the background it wouldn’t work too well otherwise. I will delve into all these details in one of my upcoming posts within my SQLite series. So for now simply believe me: There is no leak here! If you use just one SQLiteOpenHelper object within your app you will only have at most one SQLiteDatabase object at any time – unless you pass this object needlessly around and leak it that way. Finally: Android destroys the content provider when it’s no longer needed and cleans up any ressources. Very convenient!

#### Updating and deleting records of your content provider

The code for updating and deleting records looks pretty much the same. I’m going to show you how to update records and afterwards I will briefly describe the necessary changes for deleting.

```java
@Override
public int update(Uri uri, ContentValues values, String selection,
      String[] selectionArgs) {
   SQLiteDatabase db = mHelper.getWritableDatabase();
   int updateCount = 0;
   switch (URI_MATCHER.match(uri)) {
   case ITEM_LIST:
      updateCount = db.update(
            DBSchema.TBL_ITEMS, 
            values, 
            selection,
            selectionArgs);
      break;
   case ITEM_ID:
      String idStr = uri.getLastPathSegment();
      String where = Items._ID + " = " + idStr;
      if (!TextUtils.isEmpty(selection)) {
         where += " AND " + selection;
      }
      updateCount = db.update(
            DBSchema.TBL_ITEMS, 
            values, 
            where,
            selectionArgs);
      break;
   default:
      // no support for updating photos or entities!
      throw new IllegalArgumentException("Unsupported URI: " + uri);
   }
   // notify all listeners of changes:
   if (updateCount > 0 && !isInBatchMode()) {
      getContext().getContentResolver().notifyChange(uri, null);
   }
   return updateCount;
}
```

First you once again use your UriMatcher to distinguish between dir based and id based URIs. If you use a dir based URI, you simply pass this call on to the SQLiteDatabase's update() method substituting the URI with the correct table name. In case of an id based URI you have to extract the id from the URI to form a valid WHERE clause. After this call SQLiteDatabase‘s update() method. Finally you return the number of modified records.

The only changes you have to make for your delete() method is to change the method names and to get rid of the ContentValues object. Everthing else is exactly as shown in the update() method’s code sample.


```java
@Override
public int delete(Uri uri, String selection, String[] selectionArgs) {
   SQLiteDatabase db = mHelper.getWritableDatabase();
   int delCount = 0;
   switch (URI_MATCHER.match(uri)) {
   case ITEM_LIST:
      delCount = db.delete(
            DBSchema.TBL_ITEMS, 
            selection, 
            selectionArgs);
      break;
   case ITEM_ID:
      String idStr = uri.getLastPathSegment();
      String where = Items._ID + " = " + idStr;
      if (!TextUtils.isEmpty(selection)) {
         where += " AND " + selection;
      }
      delCount = db.delete(
            DBSchema.TBL_ITEMS, 
            where, 
            selectionArgs);
      break;
   default:
      // no support for deleting photos or entities -
      // photos are deleted by a trigger when the item is deleted
      throw new IllegalArgumentException("Unsupported URI: " + uri);
   }
   // notify all listeners of changes:
   if (delCount > 0 && !isInBatchMode()) {
      getContext().getContentResolver().notifyChange(uri, null);
   }
   return delCount;
}
```

Lifecycle

As with all components Android also manages the creation and destruction of a content provider. But a content provider has no visible state and there is also nothing the user has entered that should not be lost. Because of this Android can shut down the content provider whenever it sees fit.

So instead of the many methods you usually have to take care of in your activities or fragments, you have to implement just this one method: onCreate(). Keep this method as fast as possible. It runs on the UI thread when your app starts up. So it has to be fast otherwise the first impression with your app won’t be too impressive.

Normally that’s just it. But there are two other callback methods you might want to react to depending on circumstances:

+ When your content provider needs to react to changed settings (e.g. the user selected language) you have to implement the onConfigurationChanged() method that takes a Configuration object as parameter.
+ When your content provider keeps large amounts of data around (which you should strive to avoid) you should also implement the onLowMemory() method to release these data.

#### Configuring your content provider

As with any component in Android you also have to register your content provider within the AndroidManifest.xml file. The next code sample shows the configuration of the sample content provider.

```xml
<provider
   android:name=".provider.LentItemsProvider"
   android:authorities="de.openminds.samples.cpsample.lentitems"
   android:exported="true"
   android:grantUriPermissions="true"
   android:label="LentItemsProvider"
   android:readPermission="de.openminds.samples.cpsample.lentitems.READ"
   android:writePermission="de.openminds.samples.cpsample.lentitems.WRITE" />
```

You use a <provider> element for each content provider you want to register. Of its many attributes you usually will use authorities, name and possibly one of the permission attributes.

The name attribute simply is the fully qualified class name of your ContentProvider subclass. As always it is possible to use a simple “.” for the default-package of your application.

I have explained the rules for authorities in detail in the post about content provider clients. It should be akin to Java’s package naming conventions and must be unique among all content providers on any device it is going to be installed on. So don’t be sloppy with the provider’s authority.

#### Think about necessary permissions

As you have seen above, you can – and probably should – add restrictions to your content provider. By using one of the permission attributes you force clients to be open to their users about the use of your content provider’s data.

If you do not export your content provider this is no issue for you. But if you do, think carefully about what you want clients to be able to do and how to set permissions accordingly.

#### When to use content providers

As described in the introductory post, one reason to use content providers, is to export data. So, should you use content providers only to provide data to external applications? Or when would it be appropriate to prefer them for your own application over directly accessing the database?

I recommend to use a content provider whenever your app needs to react to changes in the underlying data store – even to changes from within your app itself. And this of course is more often true than not. You might for example have a ListView which displays summary information of the data held in your data store. Now if you delete or add an entry and return to your list the list should reflect these changes. This is easily done using a content provider in combination with using Loaders. Using the database directly you would have to trigger a new query on your own.

You must use content providers if you want to use search suggestions. In that case Android leaves you no choice.

Android also demands that you use content providers with sync adapters. But contrary to search suggestions you can work around this requirement with a stub provider. But, well, I like content providers, so I recommend to use a proper one with sync adapters.

<h3 id="performance">Better Performance with ContentProviderOperation</h3>

ContentProviders are one of Android’s core building blocks. They represent a relational interface to data – either in databases or (cached) data from the cloud.

Sometimes you want to use them for multiple operations in a row. Like updating different sources and so on. In those cases you could call the respective ContentResolver methods multiple times or you could execute a batch of operations. The latter is the recommended practise.

To create, delete or update a set of data in a batch like fashion you should use the class ContentProviderOperation.

According to Android’s documentation it is recommended to use ContentProviderOperations for multiple reasons:

+ All operations execute within the same transaction – thus data integrity is assured
+ This helps improve performance since starting, running and closing one transaction offers far better performance than opening and committing multiple transactions
+ Finally using one batch operation instead of multiple isolated operations reduces the number of context switches between your app and the content provider you are using. This of course also helps to improve the performance of your app – and by using less cpu cycles also reduces the power consumption.
+ To create an object of ``ContentProviderOperation`` you need to build it using the inner class ``ContentProviderOperation.Builder``. You obtain an object of the Builder class by calling one of the three static methods newInsert, newUpdate or newDelete:

<table>
<caption>Methods to obtain a Builder object</caption>
<tr>
<th>Method</th>
<th>Usage</th>
</tr>
<tr>
<td>newInsert</td>
<td>Create a Builder object suitable for an insert operation</td>
</tr>
<tr>
<td>newUpdate</td>
<td>Create a Builder object suitable for an update operation</td>
</tr>
<tr>
<td>newDelete</td>
<td>Create a Builder object suitable for a delete operation</td>
</tr>
</table>

The Builder is an example of the Gang of Four Builder pattern. A Builder defines an interface for how to create objects. Concrete instances then create specific objects for the task at hand. In this case we have three different Builders for creating ContentProviderOperation objects. These objects can be used to create, update or delete ContentProvider data sets.

Typically all steps necessary to create a ContentProviderOperation object are done in one round of method chaining. That’s possible because all methods of the Builder class return a Builder object themself. The one exception is the build() method, which instead returns the desired object: Our completely created ContentProviderOperation object. So a typical chain might look like this:

```java
ArrayList<ContentProviderOperation> ops = 
   new ArrayList<ContentProviderOperation>();
ops.add(
   ContentProviderOperation.newInsert(RawContacts.CONTENT_URI)
       .withValue(RawContacts.ACCOUNT_TYPE, "someAccountType")
       .withValue(RawContacts.ACCOUNT_NAME, "someAccountName")
       .withYieldAllowed(true)
       .build());
```

Of course you could also use a ContentValues object as usual and use the withValues(values) method instead.

The Builder class has among others these methods you can use to define which objects to delete or how to create or update an object:

<table>
<caption>Some important methods of the Builder object</caption>
<tr>
<th>Method</th>
<th>Usage</th>
</tr>
<tr>
<td>withSelection (String selection, String[] selectionArgs)</td>
<td>Specifies on which subset of the existing data set to operate. Only usable with ContentProviderOperation objects used to update or delete data</td>
</tr>
<tr>
<td>withValue (String key, Object value)</td>
<td>Defines the desired value for one column. Only usable with ContentProviderOperation objects used to create or update data</td>
</tr>
<tr>
<td>withValues (ContentValues values)</td>
<td>Defines the desired values for multiple columns. Only usable with ContentProviderOperation objects used to create or update data</td>
</tr>
</table>

As you can see in the code sample I presented above you need an ArrayList of ContentProviderOperation objects. For every ContentProvider-CRUD method you have to use one ContentProviderOperation object and add it to this list. I will explain in a later blog post about the method withValueBackReference() why it has to be an ArrayList and not say a LinkedList.

The list is finally passed to the applyBatch() method of the ContentResolver object:

```java
try {
   getContentResolver().
      applyBatch(ContactsContract.AUTHORITY, ops);
} catch (RemoteException e) {
   // do s.th.
} catch (OperationApplicationException e) {
   // do s.th.
}
```


