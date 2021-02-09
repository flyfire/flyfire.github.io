+++
title = "Checking Intent Availability"
date = "2014-06-02"
slug = "2014/06/02/checking-intent-availability"
Categories = ["android", "dev"]
+++
Intents are probably Android’s most powerful tool. They tie together loosely coupled components and make it possible to start Activities that are not part of your own app. Using intents you can delegate to the Contacts app to select a contact or you can start the browser, share data and so on.

The receiving apps declare intent filters which describe the intents its component react to. If you know which intent filters the app provides and if the components are publicly available you can start any Activity of any app on the users device.

The problem is: How do you know if the app is installed – if you can use the Intent?

The solution is to create the Intent and check if it is available:

```java
public static boolean isIntentAvailable(Context ctx, Intent intent) {
   final PackageManager mgr = ctx.getPackageManager();
   List<ResolveInfo> list =
      mgr.queryIntentActivities(intent, 
         PackageManager.MATCH_DEFAULT_ONLY);
   return list.size() > 0;
}
```

And we can use it this way:

```java
public boolean onPrepareOptionsMenu(Menu menu) {
    final boolean scanAvailable = isIntentAvailable(this,
        "com.google.zxing.client.android.SCAN");

    MenuItem item;
    item = menu.findItem(R.id.menu_item_add);
    item.setEnabled(scanAvailable);

    return super.onPrepareOptionsMenu(menu);
}
```

If you wanna check this before ``startActivity`` in case not caught by ``ActivityNotFoundException``,here is another way:

```java
if (intent.resolveActivity(ctx.getPackageManager()) != null) {
    ctx.startActivity(intent);
} else {
    log("No component can react to this intent = " + intent);
}
```
