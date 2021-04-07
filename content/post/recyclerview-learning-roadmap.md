---
title: "Recyclerview Learning Roadmap"
date: 2021-04-05T20:00:33+08:00
draft: false
categories: ["android"]
---

去年的时候组内分技术储备方向，由于我主要负责练习题这块，用到RecyclerView比较多，就被分到了RecyclerView深入理解这块，但是去年一直在赶需求，没空做这个，最近迭代任务不太多了，准备把这块补起来。

<!--more-->

这篇文章主要是索引性质，以后会慢慢完善，对RecyclerView的分析，我想从以下几个方面：

+ 作为普通的ViewGroup来看measure/layout/draw三大流程
+ 触摸事件的处理，NestedScrolling机制
+ 缓存处理，这是RecyclerView的精华所在，需要好好分析
+ Adapter/ItemDecoration/ItemAnimator
+ LayoutManager

暂时想到这些，如果有其他的会慢慢补充。我建立了一个源码分析工程[RecyclerViewLearningDemo](https://github.com/flyfire/RecyclerViewLearningDemo)，放入了recyclerview-1.2.0-alpha05的源码。

记得之前跟教授一起出去吃饭的时候说到这个RecyclerView技术储备的问题，教授说我今年能搞定就算不错了，当时我想要这么久吗，最近进入4月份了，回头看RecyclerView的分析还是没有进展，不禁捏了一把冷汗，希望自己能按住性子，按部就班分而治之的把RecyclerView分析完。千里之行始于足下，这篇就算一个索引，也算一个flag吧。

