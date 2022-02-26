---
title: "Java 序列化"
date: 2022-02-26T19:43:52+08:00
draft: false
---

序列化：将数据结构或对象转换成二进制串的过程。

持久化：把数据结构或对象存储起来。

序列化方案：

+ Seriable

+ json/xml/protobuf

+ Parcelable


``Seriable``tag接口，``ExternalSeriable``接口

``ObjectOutput``,``ObjectStreamClass``

serialVersionUID是一个private static final long型ID，当它被印在对象上时，它通常是对象的哈希码，你可以使用serialver这个JDK工具来查看序列化对象的serialVersionUID。serialVersionUID用于对象的版本控制，也可以在类文件中指定serialVersionUID。不指定serialVersionUID的后果是，当你添加或修改类的任何字段时，则已序列化类将无法恢复，因为新类和旧序列化对象生成的serialVersionUID将有所不同。Java序列化过程依赖于正确的序列化对象恢复状态，在序列号对象版本不匹配的时候会引发java.io.InvalidClassException无效类异常。

单例类``readResolve``

``Parcel``native,``writeAligned``,``readAligned``

| Serialable | Parcelabe |
| ------ | ------ |
| 通过IO对磁盘操作，速度较慢 | 直接在内存操作，效率高，性能好 |
| 大小不受限制 | 一般不能超过1M，修改内核也只能4M |
| 大量使用反射，产生内存碎片 | | 

序列化前后的对象，深复制

Enum.valueOf

``Bundle``内部使用``ArrayMap``，空间性比HashMap好一些。