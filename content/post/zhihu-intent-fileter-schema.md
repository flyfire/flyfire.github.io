+++
title = "Zhihu Intent Fileter Schema"
date = "2014-09-16"
slug = "2014/09/16/zhihu-intent-fileter-schema"
Categories = ["dev", "android"]
+++
<center><p><img src="/images/zhihu_logo.png" alt="zhihu"></p></center>

之前使用浏览器浏览知乎网站时，网站上会显示打开应用，不同的浏览器点击会有不同的反应(跳转网页或者打开知乎应用)，当时思考了一下具体的实现方式，觉得可能是知乎网站根据UC或者Chrome浏览器不同的UA显示不同的网页代码，在点击应用的时候会触发超链接，知乎客户端根据特定的Schema来在客户端中打开不同的问题页面，最近在网上浏览到一片文章，作者对这种冲浏览器跳转到客户端应用的实现方式进行了具体的分析，我只想到文章中的前两种方式，后面的Chrome浏览器的Chrome Intent以及后台启动http服务来响应网页链接的方式囿于眼界没有想到，不得不佩服文章作者视野之开阔。文章不错，特分享在此。

Ref:[#黑科技# 跳出浏览器](http://zhuanlan.zhihu.com/andlib/19848910)

当移动浪潮来袭，不论是传统 PC 网站/应用，还是新兴的移动互联网，都一并蜂拥的走进用户的手机。提供一个便于手机浏览的 Web 页面，再造一个功能丰富的移动 App 成了每个产品的标配。提供移动 Web 页面，可以使得用户更易获取产品信息，在微信上、微博中，搜索里点个链接，就可以立刻享用产品功能；而提供移动 App，则可以提供更好的产品体验，使用 native api 能构建更丰富的特色功能，提供更出众的性能表现。

来想象一下这个场景，当别人发给你一个链接，是知乎问题「豌豆荚的员工工作方式是什么样的？」，你在手机浏览器上慢吞吞的加载好了，答案看得特别激动想点个赞。结果发现，还！要！登！录！我明明已经安装了知乎的 App 好吗！为啥不让我愉快的在知乎 App 上操作呐？

这就是本 #黑科技# 的主题，有了移动 Web 页面，又提供了移动 App，如何能让两者更完美的结合在一起呢？当用户已经安装了 App 的前提下，访问移动端 Web 页面时，可以无缝的跳转到 App 中对应的位置去？

<center><p><img src="/images/zhihu_question_wandoulab.jpg" alt="wandoulabs"></p></center>

依然举上面那个栗子，知乎的同学其实已经有了解决方案，就是「打开应用」那个按钮，如果已经安装了知乎 App，点击后就会从浏览器跳转到应用中了。类似于这样的按钮，背后的技术方案是什么？有哪些局限性？有没有什么一招搞定的必杀技？这就是本文讨论的主要内容。

## 第一招：拦截 Http 跳转

在 Android 中，最标准的方式，就是在应用的配置文件 ``AndroidManifest.xml`` 中，通过 ``<activity>`` 标签里的 ``<intent-filter>`` 来声明：“本应用可以更好的处理某些 url 对应的页面，浏览器你交给我吧”。套在例子上，声明形如：

```xml
<activity android:name=”com.zhihu.android.QuestionActivity”>
  <intent-filter>
    <action android:name=”android.intent.action.VIEW” />
    <category android:name=”android.intent.category.DEFAULT” />
    <category android:name=”android.intent.category.BROWSABLE” />
    <!-- 关键所在，匹配相应域名和 url 模式 -->
    <data android:scheme=”http” android:host=”www.zhihu.com” 
android:pathPattern=”/question/.*” />
  </intent-filter>
</activity>
```

做了上述的声明之后，在 Chrome 里访问 豌豆荚的员工工作方式是什么样的？，便可以跳转到知乎 Android 客户端，并打开这个问题的页面。不过这个解决方案有挺多问题，最重要的一个原因是：“兼容性”。

除了 Chrome，从豌豆荚上的下载量看，最热门的手机浏览器是这些：

<center><p><img src="/images/mobile_browser_share.jpg" alt="mobile browser"></p></center>

而以上浏览器，大都不遵守 Android 的协定，不支持通过匹配 url 跳转到更适合的应用中去。臆测其原因，大抵是国内浏览器都不愿将流量导给其他应用吧。


## 第二招：自定义 Scheme

如此，那就另辟蹊径，既然 http 协议的 url 会被很多浏览器拦自行处理掉，那就不用 http 协议而采用自定义的 scheme 试试看。

将 ``AndroidManifest.xml`` 中的声明修改如下：

```xml
<activity android:name=”com.zhihu.android.QuestionActivity”>
  <intent-filter>
    <action android:name=”android.intent.action.VIEW” />
    <category android:name=”android.intent.category.DEFAULT” />
    <category android:name=”android.intent.category.BROWSABLE” />
    <!-- 关键所在，匹配相应的 scheme -->
    <data android:scheme=”zhihu” android:host=”questions” />
  </intent-filter>
</activity>
```

把「打开应用」的跳转链接设置为形如 "zhihu://questions/…" 的 url，点击后就可以匹配跳转到应用对应的 activity 中去。当然，如果简单的使用 ``<a>`` 标签来做这件事情，若手机中未安装知乎客户端，点击后就会跳转到一个错误页面（地址是 zhihu://questions/…）。解决方案也简单，使用 ``<iframe>`` 即可，详情就不在此赘述。

<!-- more -->

## 第三招：Chrome Intent

自定义的 scheme 可以搞定很多浏览器，但 Chrome 除外。原因是为了更有序的打通浏览器页面和本地应用，Chrome 25 后不再支持自定义的 scheme，而推出了 [Chrome Intent](https://developer.chrome.com/multidevice/android/intents)，作为标准协议进行推广，其格式形如：

```xml
intent:
  //scan/
  #Intent; 
    package=com.google.zxing.client.android; 
    scheme=zxing; 
    end;
```

Chrome Intent 首先将 scheme 统一为 ”intent“，大量信息放到了锚点 ”#“ 之后，称作 ”fragment“（此 fragment 非彼 fragment），它描述了由谁来接收这个 uri。fragment 中可以指定打开这个 uri 的包名，或者是 action、extra，等等。使用 Intent.parseUri 函数可以将这样的 uri 直接转成一个 intent 对象，反之调用 Intent.toUri 函数可将 intent 对象序列化如此格式的 uri。

应用到知乎这个例子里，在 AndroidManifest.xml 中的声明与自定义 scheme 写法完全一致，只是在调用时，需要在跳转链接中写成如下格式：

```xml
intent:
  //questions/...
  #Intent; 
    package=com.zhihu.android;
    scheme=zhihu; 
    end; 
```

## 必杀技：内嵌 Http 服务

至此，只要利用 UA 信息，合理使用自定义 scheme 和 Chrome Intent，就可以搞定市面上几乎全部的浏览器，这就完了吗？当然没有！

随着以微信为代表的社交应用的不断发展，它内嵌的 WebView 已然成为一个轻型浏览器了，坐拥巨大的用户和内容分享量，微信等应用带来的页面访问量是不容忽视的。但这些应用的 WebView 通常是禁止外链的，不论是什么 scheme 在这里一律不好使，这就使得分享到微信的知乎问题，就是再点击「打开应用」都是无效的。

有办法解决么？当然，是有的，”黑科技“ 粉墨登场的时刻到了。大家都知道，web 页面可以发起 ajax 请求用来与服务器交互，如果这个 ”服务器“ 不在云端，而是在本机呢？没错，解决方案就是在应用中绑定本地端口，启动一个 http 服务，来响应发送过来的请求，打开应用或者是做其他事情。

如果，知乎应用在后台启动 http 服务，绑定一个端口，比如：12306 吧。那 web 页面可以发送如下的 ajax 请求来实现打开应用：

```javascript
$.ajax({
  url: "http://127.0.0.1:12306/open?intent=...",
 }).done(function() {
  // do what you want
});
```

当然，要做的足够细致，还需要实现类似于 ”http://127.0.0.1:12306/is_installed“ 这样的 api，如果知乎安装了，返回 200，如果服务未启动或者知乎未安装，自然是会返回 404，由此可以在 web 页面中判断是否安装了知乎应用，进而决定是否要显示「打开应用」的按钮。

通常，必杀技都是有副作用的，如果需要准确的判断是否安装了知乎，就需要这个 http 服务始终存活，否则就没启动和没安装傻傻分不清楚了。至于如何使得 “一个已安装应用在各种情况下都保持后台运行”，则是另一个充满了黑科技的领域，待日后再聊聊这个话题。

PS：启动后台 http 服务的代码，在豌豆荚 git 中保持 review 不通过的状态估计都得有一年了，因为豌豆荚 Android 应用 “用户不主动开启相关功能则不允许后台常驻” 的原则相悖。所以如果有天你在微信页面中突然打开了某个 “安全” 应用、“搜索” 应用，千万别觉得神奇，而是在看看本文，琢磨一下那是付出什么换来的。

