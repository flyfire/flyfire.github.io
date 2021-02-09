+++
title = "Datastructure and Algorithms"
date = "2018-07-01"
slug = "2018/07/01/datastructure-and-algorithms"
Categories = ["algorithm", "notes"]
+++

## ArrayList

+ 尾插效率高，支持随机访问
+ 中间插入或者删除效率低
+ 空间不够用时，自动增长为现有空间的1.5倍
+ 底层使用Object数组来存储数据，使用System.arrayCopy来移动元素

## LinkedList

+ 头插，中间插，删除效率高
+ 不支持随机访问
+ MessageQueue中Message根据msg.when来进行插入，mMessages指向头结点

## Vector

+ 底层使用数组实现，增长看``capacityIncrement``，若``capacityIncrement``小于0，翻倍增长
+ 方法有``synchronized``修饰，线程安全
+ ``Stack``栈底层实现使用的``Vector``
+ ``Stack``在同一端进行插入和删除，``FILO``
+ 队列``Queue``是只允许在一端进行插入操作，而在另一端进行删除操作的线性表，插入的一端称为队尾，删除的一端称为队头
+ 把队列的头尾相接的顺序存储结构称为循环队列
+ 双端队列``Deque``是一种具有队列和栈的性质的数据结构，双端队列中的元素可以从两端弹出，其限定插入和删除操作在队列的两端进行。``LinkedList``，``ArrayDeque``实现了``Deque``接口,``LinkedBlockingQueue``实现了``BlockingDeque``接口
+ 优先级队列，``PriorityQueue``，``MessageQueue``根据``Message.when``来进行插入
