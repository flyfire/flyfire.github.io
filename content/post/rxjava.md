---
title: "Rxjava"
date: 2022-03-06T13:04:08+08:00
draft: false
---

+ 起点，终点

+ 响应式编程

+ 核心思想：有一个起点和一个终点，起点开始流向我们的事件，把事件流向到终点，只不过在流向的过程中，可以增加拦截，拦截时可以对事件进行改变，终点只关心它上一个拦截。

+ 标准的观察者模式：一个被观察者，多个观察者。

+ RxJava:多个被观察者，一个观察者。

+ ``ObservableCreate``,``ObservableMap``,``MapObserver``,``CreateEmitter``

+ ``RxJavaPlugins``

+ ``subscribeOn(1).subscribeOn(2).subscribeOn(3)``,3调2,2调1,最终是1起作用

+ ``observeOn(1).observeOn(2).observeOn(3)``1调2,2调3,最终observer执行在3