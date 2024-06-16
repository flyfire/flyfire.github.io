---
title: "Java Io"
date: 2022-03-12T13:30:48+08:00
draft: false
---

内存：读入，写出

装饰器模式

+ ``Component``抽象构建接口

+ ``ConcreteComponent``具体的构建对象，实现``Component``接口，通常就是被装饰的原始对象，就对这个对象添加功能


+ ``Decorator``所有装饰器的抽象父类，需要定义一个与``Component``接口一致的接口，内部持有一个``Component``对象，就是持有一个被装饰的对象。

+ ``ConcreteDecoratorA/ConcreteDecoratorB``，实际的装饰器对象，实现具体添加功能。


```java
val source = RandomAccessFile(sourceFile, "r")
val target = RandomAccessFile(targetFile, "rw)

val sourceChannel = source.channel
val targetChannel = source.channel

val byteBuffer = ByteBuffer.allocate(1024 * 1024)

while(sourceChannel.read(byteBuffer) != -1) {
    byteBuffer.flip()
    targetChannel.write(byteBuffer)
    byteBuffer.clear()
}
```