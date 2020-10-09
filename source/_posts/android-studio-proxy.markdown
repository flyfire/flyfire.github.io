---
layout: post
title: "Android Studio 代理问题"
date: 2019-09-18 14:15
comments: true
categories: 
- tools
tags:
- android
---

Android Studio 设置代理后gradle sync失败，解决方法。

<!-- more -->

今天到公司后，新需求下来了，同步代码，sync工程，开完晨会发现gradle sync失败了。

报错类似``could not resolve xxx``，AS是开了代理的，http代理8123端口，在AS里check connection连接baidu居然报错read timeout。心想不会是墙又升高了吧，最近要国庆啥的，在Chrome里面打开了一下，google访问正常，梯子还是好的。看了下v2ray config文件，1080 socks端口，8123 http端口，对的啊，感觉一脸懵逼。把AS代理切换到socks，check connection通了，可是gradle sync的时候v2ray报错``rejected  v2ray.com/core/proxy/socks: unknown Socks version: 67``，AS的socks代理貌似是有问题的，所以之前一直用http代理，现在的问题是socks代理工作正常，http代理出问题了。

``lsof -i:8123``发现是``privoxy``，尝试``kill -9 PID``了一下，发现杀死之后又会重启一个进程。开Activity Monitor，搜索``privoxy``发现``privoxy``路径是``~/Library/Application Support/ShadowsocksX-NG``，parent process是``launchd``，问题比较清晰了，``privoxy``是在开机的时候自动启动的，而且是SSR里面的，而SSR被我冷藏后``launch at login``取消了，SSR的socks端口号和v2ray的端口号不一样，导致``privoxy``转发socks流量出现了问题，而且v2ray命令行启动的时候没有报8123端口被占用，直接启动成功了。

于是尝试取消开机启动SSR的``privoxy``，在``Users&Groups``里``Login Items``看了一下没发现，搜索了一下发现了plist文件在``~/Library/LaunchAgents``，同时发现了一个工具[App Cleaner & Uninstaller](https://nektony.com/mac-app-cleaner)可以管理启动项。

``brew cask install app-cleaner``之后关闭ShadowsocksX-NG的启动项，重启电脑，启动v2ray，切换AS代理到到http代理，sync成功了。

吐槽下AS的socks代理，真不给力。
