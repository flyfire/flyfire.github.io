---
layout: post
title: "Android Snippets"
date: 2014-09-27 12:50
comments: true
categories: 
- dev
- android
tags:
- android
---
<center><p><img src="/images/android_robot.png" width="255" height="300"></p></center>
* [CheckingWifiConnectivity](#checking-wifi-connectivity)
* [ParseJson](#parse-json)
* [GetLauncherApplication](#get-launcher-applications)
* [GetSdcardSize](#get-sdcard-size)
* [AndroidBootupTime](#android-bootup-time)
* [JsonParser](#json-parser)
* [FullScreen](#full-screen)
* [RestoreListViewPosition](#restore-listview-position)
* [StartApplication](#start-application)
* [StringUtils](#string-utils)
* [CheckingExternalStorageState](#checking-external-storage-state)
* [SavingViewAsBitmap](#saving-view-as-bitmap)
* [ConvertViewToBitmap](#convert-view-to-bitmap)
* [HideKeyboard](#hide-window-keyboard)

<h2 id="checking-wifi-connectivity">CheckingWifiConnectivity</h2>

```java
public static boolean isConnected(Context context) {
    ConnectivityManager connectivityManager = (ConnectivityManager)
        context.getSystemService(Context.CONNECTIVITY_SERVICE);
    NetworkInfo networkInfo = null;
    if (connectivityManager != null) {
        networkInfo =
            connectivityManager.getNetworkInfo(ConnectivityManager.TYPE_WIFI);
    }
    return networkInfo == null ? false : networkInfo.isConnected();
}
```

<!-- more -->

<h2 id="parse-json">ParseJson</h2>

```java
/*
 *Steps to be followed for JSON parsing with Gson Library :-
 *	Download Gson library from above link & add jar to eclipse project.
 *	Create AsyncTask class to handle background operations.
 *	Use the DefaultHttpClient to retrieve the data if this is a web resource.
 *	Create respective model classes to handle response data.
 *	Create instance of Gson class.
 *	Use the fromJson() in order to parse the JSON input and return the model object.
 *	Update the UI elements by using model objetcs.

/*
 * parse json
 */
 
package com.sample;
 
import java.io.IOException;
import java.io.InputStream;
import java.io.InputStreamReader;
import java.io.Reader;
 
import org.apache.http.HttpEntity;
import org.apache.http.HttpResponse;
import org.apache.http.HttpStatus;
import org.apache.http.client.methods.HttpGet;
import org.apache.http.impl.client.DefaultHttpClient;
 
import android.app.Activity;
import android.app.ProgressDialog;
import android.content.DialogInterface;
import android.content.DialogInterface.OnCancelListener;
import android.os.AsyncTask;
import android.os.Bundle;
import android.text.Html;
import android.util.Log;
import android.widget.TextView;
 
import com.google.gson.Gson;
 
public class MyClass extends Activity {
 
       TextView capitalTextView;
       ProgressDialog progressDialog;
 
       /** Called when the activity is first created. */
       @Override
       public void onCreate(Bundle savedInstanceState) {
              super.onCreate(savedInstanceState);   
              setContentView(R.layout.main);
              capitalTextView = (TextView) findViewById(R.id.capital_textview);
              this.retrieveCapitals();
       }
 
       void retrieveCapitals() {
              progressDialog = ProgressDialog.show(this, "Please wait...", "Retrieving data...", true, true);
 
              CapitalsRetrieverAsyncTask task = new CapitalsRetrieverAsyncTask();
              task.execute();
              progressDialog.setOnCancelListener(new CancelListener(task));       
       }
 
       private class CapitalsRetrieverAsyncTask extends AsyncTask<Void, Void, Response> {
              @Override
              protected Response doInBackground(Void... params) {
                     String url = "http://api.androidsmith.com/capitals.php";
                     Reader inputStreamReader = getInputStreamReader(url);
                     return parseResponse(inputStreamReader);
              }
 
              @Override
              protected void onPostExecute(Response myresponse) {
                     super.onPostExecute(myresponse);
                     StringBuilder builder = new StringBuilder();
 
                     for (Country country : myresponse.data) {
                           builder.append(String.format("<br>Country: <b>%s</b><br>Capital: <b>%s</b><br><br>", country.getCountry(), country.getCapital()));
                     }
                     capitalTextView.setText(Html.fromHtml(builder.toString()));
                     progressDialog.cancel();
              }
       }
 
       private class CancelListener implements OnCancelListener {
              AsyncTask<?, ?, ?> cancellableTask;
              public CancelListener(AsyncTask<?, ?, ?> task) {
                     cancellableTask = task;
              }
 
              @Override
              public void onCancel(DialogInterface dialog) {
                     cancellableTask.cancel(true);
              }
       }
 
       private Reader getInputStreamReader(String url) {
              Reader inputStreamReader = null;
              HttpGet getRequest = new HttpGet(url);
              try {
                     DefaultHttpClient httpClient = new DefaultHttpClient();
                     HttpResponse getResponse = httpClient.execute(getRequest);
                     final int statusCode = getResponse.getStatusLine().getStatusCode();
 
                     if (statusCode != HttpStatus.SC_OK) {
                           Log.w(getClass().getSimpleName(), "Error " + statusCode + " for URL " + url);
                           return null;
                     }
 
                     HttpEntity getResponseEntity = getResponse.getEntity();
                     InputStream httpResponseStream = getResponseEntity.getContent();
                     inputStreamReader = new InputStreamReader(httpResponseStream);
              }
              catch (IOException e) {
                     getRequest.abort();
                     Log.w(getClass().getSimpleName(), "Error for URL " + url, e);
              }
              return inputStreamReader;
       }
 
       private Response parseResponse(Reader inputStreamReader){
              Gson gson = new Gson();
              Response response = gson.fromJson(inputStreamReader, Response.class);
              return response;
       }
}
 
Country.Java:-
 
package com.sample;
 
public class Country {
 
       String country;
       String capital;
      
       public Country() {
              this.country = "";
              this.capital = "";
       }
 
       public String getCountry() {
              return this.country;
       }
 
       public String getCapital() {
              return this.capital;
       }
}
 
 
Response.Java:-
package com.sample;
import java.util.ArrayList;
import com.androidsmith.sacc.model.Country;
public class Response {
       ArrayList<Country> data;
       public Response() {
              data = new ArrayList<Country>();
       }
}
```

<h2 id="get-launcher-applications">GetLauncherApplications</h2>

```java
PackageManager manager = getPackageManager();
Intent mainIntent = new Intent(Intent.ACTION_MAIN, null);
mainIntent.addCategory(Intent.CATEGORY_LAUNCHER);
 
List<ResolveInfo> resolveInfos= manager.queryIntentActivities(mainIntent, 0);
// Below line is new code i added to your code 
Collections.sort(resolveInfos, new ResolveInfo.DisplayNameComparator(manager));
 
for(ResolveInfo info : resolveInfos) {
     ApplicationInfo applicationInfo = info.activityInfo.applicationInfo;
     //... 
     //get package name, icon and label from applicationInfo object
} 
```

<h2 id="get-sdcard-size">GetSdcardSize</h2>

```java
public void getSdSize(String path){
    File filePath;
    if(!TextUtils.isEmpty(path)){
        filePath = new File(path);
    }else {
        filePath = Environment.getExternalStorageDirectory();
    }
    StatFs stat = new StatFs(filePath.getPath());
    long blockSize = stat.getBlockSize();
    long totalBlocks = stat.getBlockCount();
    long availableBlocks = stat.getAvailableBlocks();
	
    mSdSize.setSummary(blockSize * totalBlocks);
    mAvailSize.setSummary(blockSize * availableBlocks);
}
```

<h2 id="android-bootup-time">AndroidBootupTime</h2>

```java
java.lang.System.currentTimeMillis() - android.os.SystemClock.elapsedRealtime();
```

<h2 id="json-parser">JsonParser</h2>

```java
public class JSONParser { 
 
static InputStream is = null;
static JSONObject jObj = null;
static String json = "";
 
// constructor 
public JSONParser() { 
 
} 
 
// function get json from url 
// by making HTTP POST or GET method 
public JSONObject makeHttpRequest(String url, String method,
        List<NameValuePair> params) throws IOException {
 
    // Making HTTP request 
    try { 
 
        // check for request method 
        if(method == "POST"){
            // request method is POST 
            // defaultHttpClient 
            DefaultHttpClient httpClient = new DefaultHttpClient();
            HttpPost httpPost = new HttpPost(url);
            httpPost.setEntity(new UrlEncodedFormEntity(params));
 
            HttpResponse httpResponse = httpClient.execute(httpPost);
 
            HttpEntity httpEntity = httpResponse.getEntity();
            is = httpEntity.getContent();
 
        }else if(method == "GET"){
            // request method is GET 
            DefaultHttpClient httpClient = new DefaultHttpClient(httpParameters);
            String paramString = URLEncodedUtils.format(params, "utf-8");
            url += "?" + paramString;
            HttpGet httpGet = new HttpGet(url);
 
            HttpResponse httpResponse = httpClient.execute(httpGet);
            HttpEntity httpEntity = httpResponse.getEntity();
            is = httpEntity.getContent();
        }            
 
 
    } catch (UnsupportedEncodingException e) {
        e.printStackTrace();
    } catch (ClientProtocolException e) {
        e.printStackTrace();
    } catch (Exception ex) {
        Log.d("Networking", ex.getLocalizedMessage());
        throw new IOException("Error connecting");
    } 
 
    try { 
        BufferedReader reader = new BufferedReader(new InputStreamReader(
                is, "iso-8859-1"), 8);
        StringBuilder sb = new StringBuilder();
        String line = null;
        while ((line = reader.readLine()) != null) {
            sb.append(line + "\n");
        } 
        is.close();
        json = sb.toString();
    } catch (Exception e) {
        Log.e("Buffer Error", "Error converting result " + e.toString());
    } 
 
    // try parse the string to a JSON object 
    try { 
        jObj = new JSONObject(json);
    } catch (JSONException e) {
        Log.e("JSON Parser", "Error parsing data " + e.toString());
    } 
 
    // return JSON String 
    return jObj;
 
} 
```

<h2 id="full-screen">FullScreen</h2>


```java
// show fullscreen
requestWindowFeature(Window.FEATURE_NO_TITLE);
getWindow().setFlags(WindowManager.LayoutParams.FLAG_FULLSCREEN, WindowManager.LayoutParams.FLAG_FULLSCREEN);
```

<h2 id="restore-listview-position">RestoreListViewPosition</h2>

```java
// restore listview position
// save index and top position 
// https://stackoverflow.com/questions/3014089/maintain-save-restore-scroll-position-when-returning-to-a-listview
 
int index = mList.getFirstVisiblePosition();
View v = mList.getChildAt(0);
int top = (v == null) ? 0 : v.getTop();
 
// ... 
 
// restore index and position 
mList.setSelectionFromTop(index, top);
```

<h2 id="start-application">StartApplication</h2>

```java
// https://stackoverflow.com/questions/3872063/launch-an-application-from-another-application-on-android
 
Intent LaunchIntent = getPackageManager().getLaunchIntentForPackage("com.package.address");
startActivity(LaunchIntent);
```

<h2 id="string-utils">StringUtils</h2>

```java
// http://my.oschina.net/kymjs/blog/221854

import java.io.UnsupportedEncodingException;
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.text.ParseException;
import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.regex.Pattern;
 
import android.app.Activity;
import android.content.Context;
import android.content.pm.ApplicationInfo;
import android.content.pm.PackageManager;
import android.content.pm.PackageManager.NameNotFoundException;
import android.os.Bundle;
import android.telephony.TelephonyManager;
 
/**
 * 字符串操作工具包
 */
public class StringUtils {
    private final static Pattern emailer = Pattern
            .compile("\\w+([-+.]\\w+)*@\\w+([-.]\\w+)*\\.\\w+([-.]\\w+)*");
    private final static Pattern phone = Pattern
            .compile("^((13[0-9])|(15[^4,\\D])|(18[0,5-9]))\\d{8}$");
 
    private final static ThreadLocal<SimpleDateFormat> dateFormater = new ThreadLocal<SimpleDateFormat>() {
        @Override
        protected SimpleDateFormat initialValue() {
            return new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        }
    };
 
    private final static ThreadLocal<SimpleDateFormat> dateFormater2 = new ThreadLocal<SimpleDateFormat>() {
        @Override
        protected SimpleDateFormat initialValue() {
            return new SimpleDateFormat("yyyy-MM-dd");
        }
    };
 
    /**
     * 返回当前系统时间
     */
    public static String getDataTime(String format) {
        SimpleDateFormat df = new SimpleDateFormat(format);
        return df.format(new Date());
    }
 
    /**
     * 返回当前系统时间
     */
    public static String getDataTime() {
        return getDataTime("HH:mm");
    }
 
    /**
     * 毫秒值转换为mm:ss
     * 
     * @author kymjs
     * @param ms
     */
    public static String timeFormat(int ms) {
        StringBuilder time = new StringBuilder();
        time.delete(0, time.length());
        ms /= 1000;
        int s = ms % 60;
        int min = ms / 60;
        if (min < 10) {
            time.append(0);
        }
        time.append(min).append(":");
        if (s < 10) {
            time.append(0);
        }
        time.append(s);
        return time.toString();
    }
 
    /**
     * 将字符串转位日期类型
     * 
     * @return
     */
    public static Date toDate(String sdate) {
        try {
            return dateFormater.get().parse(sdate);
        } catch (ParseException e) {
            return null;
        }
    }
 
    /**
     * 判断给定字符串时间是否为今日
     * 
     * @param sdate
     * @return boolean
     */
    public static boolean isToday(String sdate) {
        boolean b = false;
        Date time = toDate(sdate);
        Date today = new Date();
        if (time != null) {
            String nowDate = dateFormater2.get().format(today);
            String timeDate = dateFormater2.get().format(time);
            if (nowDate.equals(timeDate)) {
                b = true;
            }
        }
        return b;
    }
 
    /**
     * 判断给定字符串是否空白串 空白串是指由空格、制表符、回车符、换行符组成的字符串 若输入字符串为null或空字符串，返回true
     */
    public static boolean isEmpty(String input) {
        if (input == null || "".equals(input))
            return true;
 
        for (int i = 0; i < input.length(); i++) {
            char c = input.charAt(i);
            if (c != ' ' && c != '\t' && c != '\r' && c != '\n') {
                return false;
            }
        }
        return true;
    }
 
    /**
     * 判断是不是一个合法的电子邮件地址
     */
    public static boolean isEmail(String email) {
        if (email == null || email.trim().length() == 0)
            return false;
        return emailer.matcher(email).matches();
    }
 
    /**
     * 判断是不是一个合法的手机号码
     */
    public static boolean isPhone(String phoneNum) {
        if (phoneNum == null || phoneNum.trim().length() == 0)
            return false;
        return phone.matcher(phoneNum).matches();
    }
 
    /**
     * 字符串转整数
     * 
     * @param str
     * @param defValue
     * @return
     */
    public static int toInt(String str, int defValue) {
        try {
            return Integer.parseInt(str);
        } catch (Exception e) {
        }
        return defValue;
    }
 
    /**
     * 对象转整
     * 
     * @param obj
     * @return 转换异常返回 0
     */
    public static int toInt(Object obj) {
        if (obj == null)
            return 0;
        return toInt(obj.toString(), 0);
    }
 
    /**
     * String转long
     * 
     * @param obj
     * @return 转换异常返回 0
     */
    public static long toLong(String obj) {
        try {
            return Long.parseLong(obj);
        } catch (Exception e) {
        }
        return 0;
    }
 
    /**
     * String转double
     * 
     * @param obj
     * @return 转换异常返回 0
     */
    public static double toDouble(String obj) {
        try {
            return Double.parseDouble(obj);
        } catch (Exception e) {
        }
        return 0D;
    }
 
    /**
     * 字符串转布尔
     * 
     * @param b
     * @return 转换异常返回 false
     */
    public static boolean toBool(String b) {
        try {
            return Boolean.parseBoolean(b);
        } catch (Exception e) {
        }
        return false;
    }
 
    /**
     * 判断一个字符串是不是数字
     */
    public static boolean isNumber(String str) {
        try {
            Integer.parseInt(str);
        } catch (Exception e) {
            return false;
        }
        return true;
    }
 
    /**
     * 获取AppKey
     */
    public static String getMetaValue(Context context, String metaKey) {
        Bundle metaData = null;
        String apiKey = null;
        if (context == null || metaKey == null) {
            return null;
        }
        try {
            ApplicationInfo ai = context.getPackageManager()
                    .getApplicationInfo(context.getPackageName(),
                            PackageManager.GET_META_DATA);
            if (null != ai) {
                metaData = ai.metaData;
            }
            if (null != metaData) {
                apiKey = metaData.getString(metaKey);
            }
        } catch (NameNotFoundException e) {
 
        }
        return apiKey;
    }
 
    /**
     * 获取手机IMEI码
     */
    public static String getPhoneIMEI(Activity aty) {
        TelephonyManager tm = (TelephonyManager) aty
                .getSystemService(Activity.TELEPHONY_SERVICE);
        return tm.getDeviceId();
    }
 
    /**
     * MD5加密
     */
    public static String md5(String string) {
        byte[] hash;
        try {
            hash = MessageDigest.getInstance("MD5").digest(
                    string.getBytes("UTF-8"));
        } catch (NoSuchAlgorithmException e) {
            throw new RuntimeException("Huh, MD5 should be supported?", e);
        } catch (UnsupportedEncodingException e) {
            throw new RuntimeException("Huh, UTF-8 should be supported?", e);
        }
 
        StringBuilder hex = new StringBuilder(hash.length * 2);
        for (byte b : hash) {
            if ((b & 0xFF) < 0x10)
                hex.append("0");
            hex.append(Integer.toHexString(b & 0xFF));
        }
        return hex.toString();
    }
 
    /**
     * KJ加密
     */
    public static String KJencrypt(String str) {
        char[] cstr = str.toCharArray();
        StringBuilder hex = new StringBuilder();
        for (char c : cstr) {
            hex.append((char) (c + 5));
        }
        return hex.toString();
    }
 
    /**
     * KJ解密
     */
    public static String KJdecipher(String str) {
        char[] cstr = str.toCharArray();
        StringBuilder hex = new StringBuilder();
        for (char c : cstr) {
            hex.append((char) (c - 5));
        }
        return hex.toString();
    }
}

```

<h2 id="checking-external-storage-state">CheckingExternalStorageState</h2>

```java
/* Checks if external storage is available for read and write */
public boolean isExternalStorageWritable() {
    String state = Environment.getExternalStorageState();
    if (Environment.MEDIA_MOUNTED.equals(state)) {
        return true;
    }
    return false;
}
 
/* Checks if external storage is available to at least read */
public boolean isExternalStorageReadable() {
    String state = Environment.getExternalStorageState();
    if (Environment.MEDIA_MOUNTED.equals(state) ||
        Environment.MEDIA_MOUNTED_READ_ONLY.equals(state)) {
        return true;
    }
    return false;
}
```

<h2 id="saving-view-as-bitmap">SavingViewAsBitmap</h2>

```java
public Bitmap saveViewBitmap(View v){
  Bitmap bitmap = Bitmap.createBitmap(v.getWidth(), v.getHeight(), Bitmap.Config.ARGB_8888);
  Canvas canvas = new Canvas(bitmap);
  v.draw(canvas);
  return bitmap;
}
```

<h2 id="convert-view-to-bitmap">ConvertViewToBitmap</h2>

```java
public static Bitmap convertViewToBitmap(View view) {
    view.destroyDrawingCache();
    view.measure(View.MeasureSpec.makeMeasureSpec(0, View.MeasureSpec.UNSPECIFIED),
    View.MeasureSpec.makeMeasureSpec(0, View.MeasureSpec.UNSPECIFIED));
    view.layout(0, 0, view.getMeasuredWidth(), view.getMeasuredHeight());
    view.setDrawingCacheEnabled(true);
    return view.getDrawingCache(true);
}
```

<h2 id="hide-window-keyboard">HideKeyboard</h2>
```java
public static void hideKeyboard(Activity context, IBinder windowToken){
  InputMethodManager imm = (InputMethodManager) context.getSystemService(Context.INPUT_METHOD_SERVICE);
  imm.hideSoftInputFromWindow(windowToken, 0);
  //usage
  // hideKeyboard(activity, viewComponent.getWindowToken());
}
```













