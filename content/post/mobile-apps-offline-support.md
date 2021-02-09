+++
title = "Mobile Apps Offline Support"
date = "2016-03-14"
slug = "2016/03/14/mobile-apps-offline-support"
Categories = ["dev", "android"]
+++
Offline support for mobile applications can be thought of as the ability for the application to react gracefully to the lack of stability of the network connection. The rather new context of mobile devices introduced problems such as presence or absence of a network connection or even high latency and low bandwidth. These problems are rather new and thus not very well known to engineers starting with mobile development. Among other things building a mobile application which resilient to different network scenarios could mean:

+ Displaying comprehensive error messages when network calls fail.
+ Allowing the use of the application in “guest mode”, where certain features can be delayed until the user actually signs in.
+ Visually displaying the absence of network connectivity on the UI (connected mode/offline mode).
+ Disabling controls in the absence of network connectivity.
+ Allowing the user to query and act on data while no network connection (offline data access).
+ Testing the application under different network conditions!

While all these things are extremely important from the usability point of view, there is one of these that can be particular complex, “offline data access”. There are several different scenarios or levels of offline data access that applications might need to support, and I’ll go through each next.

<!-- more -->

## Local Caching

The application needs to be able to display information even when there is no connection, however under connectivity conditions the data needs to be refreshed. This is achieved by somehow persisting the data on the mobile device, usually for a healthy period of time.

<center><img src="/images/local_cache.png"></center>

There are 3 different “strategies” for refreshing the data on the cache which I would like to cover next.

### Network first

Always try to retrieve the data from the server, and whenever that is not possible, then resort to retrieve the data from the local cache. This strategy can be very useful if you are particularly interested in showing the latest and more updated information.

### Local first

For a specified period of time, don’t even try to go the network, just return from local cache. This approach is very well suited when there’s no risk in showing cached data. On the other hand, it has a better user experience since there’s usually no latency involved.

### Hybrid / Smart

This approach will return from local cache before fetching data from a service. It can either wait for a notification from the server or simply poll the service in background to refresh the data to cache it locally. This mechanism hits a balance between a good performance/UX, while still refreshing the local cache regularly, reducing the risk of showing “stale” data.

Furthermore, local caching can be complemented with some way of server-side caching support as well. Just as in HTTP caching, when retrieving the data from the server, the client can send a “revision” to see if the data has been updated. The server can check the clients revision against the current one on the server, and either inform the client that there is no need to update or return the latest data.

### Sample scenario

The improvement in performance and user experience makes local caching extremely useful in many scenarios. The key condition for it to be useful is that the data does not have to be displayed in real-time. The longer the data can be locally cached, the more sense this approach will make.

Think for instance of a list of interesting locations or contacts for users on the “field”. This is information while very useful on the go, is unlikely to change very frequently, so it is ideal for being cached locally.

## Local Queuing

Whenever the application does not have a network connection, server requests can be locally queued for later processing. This will allow the user to fire and forget operations and be notified whenever (and if) those operations were successfully processed by the server.

<center><img src="/images/local_queuing.png"></center>

When working with local queues of operations you should take into account the following things:

+ Users should be notified that the operation has been queued.
+ Users will most probably be interested in seeing the actual status of the queue. Which are the items that went through and which are the ones that are still pending?
+ It could be important to be able to cancel or retry manually an operation while it is still in the queue.
+ Whenever one of these operations is sent to the server, the user will want to know the outcome (success, failed).
+ The flow or process the user initiated could potentially need to be resumed where it was left off at the time the operation was queued.

Local queuing is particularly a good idea when having people doing auditing or field work like measuring things, and sending reports. If these operations are not updating records, but rather only inserting new ones, the implementation of this is rather simple and requires no concurrency management or conflict resolution.

### Sample Scenario

Local queuing helps to not loose work while on the go. This can be extremely important in scenarios of inventory checks or audits, where the user on the field must not loose time waiting for a connection in order to use the app or submit those reports.

### Data-Sync

By leveraging local caching and queuing, you can keep the data in your device and your server up to date. This is known as “synchronizing”. There are different ways to synchronize the data.

<center><img src="/images/data_sync.png"></center>

### Mobile Data up-to-date

In this case, you worry about the data in you mobile application being up to date. This can be achieved in two ways: by just using local caching as described above, or it can be done by querying the server for the latest changes. These latest changes, also known as “delta”, allow the mobile application to apply and reconstruct the current state of the server. In order to be able to query for the latest changes, you can leverage audit fields like `UpdatedOn`, `CreatedOn` and `DeletedOn`.

In this second case, the data is not being modified in the device, so there is no need to resolve conflicts, so the server is always right.

### Server Data up-to-date

This can be achieved by using local queues, but queuing is not enough. What happens if by the time my request is sent to the server, the data on the server was no longer in the same state as when I attempted to modify it? Delaying the execution of the request, for example due to network loss, can result in increased concurrency conflicts. At this point, the developer (or the user) must decide how to “merge” the changes on the server and the app. For every conflict in the data, the merge could be:

+ Keep the device version
+ Keep the server version
+ Keep both versions

More often than not, the logic for merging records can be automated by the mobile developer. Which algorithm is used, will be tied to the business rules of that application. Whenever this is not possible to fully automate, the user can be prompted to make a decision.

### Keep both Mobile and Server Up-to-date

This is also referred to as two-way sync. As you can probably tell by now, this would be a combination of the two previous techniques. This is the most complete and powerful of the scenarios so far described. Notice however, that while it might be tempting to build applications to support two-way sync, it is by far the most complex scenario of all. Apart from being complex, as I have covered in this article, it might not always be necessary.

### Sample Scenario

Two-way-sync gives the mobile application a whole new level of user experience. However, one of the key conditions for two-way sync to be a must-have is the need to keep a team or group of users up to date with everybody else’s activity. An example of such a thing could be collaborating applications with updates, comments or status changes. Think about a collaborative address book where everybody on the team is allowed to update contacts at any given time.

## Considerations

Building your mobile application with support for offline scenarios, can drastically improve the user experience, however choosing the right level of support, and later on implementing this is not trivial. Below I will be listing some of the things to consider when planning to add offline support to your apps.

### Data Size

When caching data locally, try to be conscious of the size of the data you’ll be storing. Striking the right balance between the amount of data that is stored and the perceived UX improvement is important. In cases where there are lots of data (ie: a full Sharepoint site), you might have to consider giving the user the option of choosing what he wants to cache for offline reading afterwards.

### Data Storage

Make sure to choose wisely how and where you will be storing your data. Is that data sensitive? If so, you will want to encrypt the data while at rest (storage). If you choose to encrypt the data, make sure to also store the key for decrypting the data in a safe place and consider leveraging operating system functionality for this. Also keep in mind that in some platforms your application’s code can be read (or reflected), so consider obfuscating your code. And last but not least, make sure to have a mechanism for remotely wiping the data on the handset. Some tools like mobile device management (MDM) platforms can help to achieve this, but it can also be handled by the application itself.

### Battery Usage

If you plan to have polling mechanisms and background jobs, make sure to take the battery status into account. Some processes and network usage might drain the battery in detriment of the user experience. You can check the status of the battery, and whether the device is connected to a charger before you start a rather consuming process.

### Consuming the Data

Depending on your application’s needs, you might have to query and operate (create, update, delete) on your data. In non-trivial scenarios using a database as the persistence mechanism is not a bad idea. There are several things to take into account for choosing the right database:

+ platform support: Will I be able to use this database from all the versions of my app? (iOS, Android, Web, Hybrid, etc…)
+ relational vs NoSQL database technology
+ ORM support for conveniently mapping the object model to the database
+ data size
+ existing support for sync protocols (ie: CouchDB)

Next we will go through a list of libraries and databases that can be useful when implementing Offline Support.

## Useful Libraries & Databases

+ SQLite,SQLite is an Open Source relational database that works very well in mobile devices. It uses a single file to store all the data, so managing the persistence side is simple. It will not solve too much on the sync- and conflict resolution side, but it’s a simple and easy to use alternative for caching or queuing information. There are implementations for the main mobile platforms like iOS, Android, Xamarin and Windows Phone.
+ SQLCypher,As previously said, when the data you are caching or queuing is rather sensitive, you might want to encrypt the data at rest. SQLCypher is a very robust alternative for encrypting SQLite databases. It has versions for every major mobile platform, but it is a paid library. It is available in different editions depending on the level of security and support you need.
+ Couchbase Mobile,Originally known as Membase, Couchbase is an open source distributed NoSQL database. It is particularly interesting in offline scenarios due to its ability to synchronize back and forth with Couchbase Mobile along with the addition of a sync gateway. It supports the main mobile platforms including Xamarin and PhoneGap, and provides local file encryption.
+ Meteor,[Meteor](https://www.meteor.com/)is an open source platform for building web applications, with built in support for Live Updates. Meteor is based on the open source Node.js platform and MongoDB. It comes with a publish-subscriber mechanism that enables Meteor to propagate changes on the data to every connected clients in real-time.

It supports mobile all platforms through hybrid tools like PhoneGap and Cordova.

## Summary

In times where mobile users are starting to expect the same level of user experience in enterprise applications as they do on their personal consumer applications, offline Support can no longer be ignored. Providing the right level of support for offline scenarios will dramatically improve the mobile application user’s experience and be vital for employee’s productivity.

Keep in mind the security aspects of storing data locally in your device, and try to not to underestimate the impact that your application can have on your users battery.


## reference

+ [Mobile Apps Offline Support](http://www.infoq.com/articles/mobile-apps-offline-support)
+ [Android Application Architecture](https://www.youtube.com/watch?v=BlkJzgjzL0c)
+ [dev-summit-architecture-demo](https://github.com/yigit/dev-summit-architecture-demo)
+ [为移动应用提供离线支持](http://www.infoq.com/cn/articles/mobile-apps-offline-support)
