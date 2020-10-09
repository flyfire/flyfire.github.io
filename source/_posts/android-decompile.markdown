---
layout: post
title: "Android反编译"
date: 2014-10-12 09:21
comments: true
categories: 
- dev
- android
tags:
- android
---
有时候在Google Play Store看到一些有趣的应用，或者对有些应用的资源图片之类的很感兴趣，这时候就需要用到Android反编译APK的一些工具了。

<!-- more -->

在Google Play上安装应用默认安装完成后是不保留应用apk文件的，要下载apk文件，可以从[Online APK Downloader](http://apkleecher.com/)或[Apk Downloader](http://apps.evozi.com/apk-downloader/)下载。

其实反编译APK主要用到3个工具，[apktool](https://code.google.com/p/android-apktool/)用来获取资源文件，[dex2jar](https://code.google.com/p/dex2jar/)用来将dex文件转换为jar文件格式，[jd-gui](http://jd.benow.ca/)用来查看jar文件中源码。

+ 获取资源文件``java -jar apktool.jar d example.apk``
+ 获取jar文件

```bash
mv example.apk example.zip
unzip example.zip
dex2jar.sh classes.dex
```

+ 使用jd-dui查看jar文件``jd-gui classes.jar``

上述工具在linux平台下的我已经打包了，可以在<a href="/downloads/files/decompile.tgz">这里</a>下载。

