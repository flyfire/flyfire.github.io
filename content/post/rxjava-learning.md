---
title: "Rxjava Learning"
date: 2022-03-09T20:22:23+08:00
draft: false
---

操作符链条中的一环，调用上游的订阅与取消订阅，监听上游的成功失败，监听下游的订阅与取消订阅，回调下游的成功失败。

``subscribeOn``切个线程去调用上游的订阅。如果在上游还没有订阅完成的时候下游要取消订阅呢，``disposable()``方法里面调用``DisableHelper.disposable(this)``将AtomicReference设置为``DISPOSED``，在上游回调它的``onSubscribe(disposable)``的时候调用``DisposableHelper.setOnce(this, disposable)``，在``setOnce``方法中会检查AtomicReference是不是null，如果不是null，cas不会成功，会调用上游的``disposable.dispose()``

