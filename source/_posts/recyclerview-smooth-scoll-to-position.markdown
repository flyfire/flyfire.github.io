---
layout: post
title: "RecyclerView平滑滑动到指定位置"
date: 2018-05-30 14:33
comments: true
categories: 
- dev
- android
tags:
- android
- recyclerview
---
最近在做一个周日历的新需求，其中有个要求是要RecyclerView平滑的滑动到指定位置，刚开始的时候我以为只要调用``smoothScrollToPosition``这个方法就可以了，测试的时候发现，``smoothScrollToPosition``只会对不可见的item有效，对于已经可见的item滑动没有效果，于是翻看了一下``smoothScrollToPosition``的源码，发现是调用了``LayoutManger``的``smoothScrollToPosition``方法。

<!-- more -->

```java
// LinearLayoutManager
    @Override
    public void smoothScrollToPosition(RecyclerView recyclerView, RecyclerView.State state,
            int position) {
        LinearSmoothScroller linearSmoothScroller =
                new LinearSmoothScroller(recyclerView.getContext());
        linearSmoothScroller.setTargetPosition(position);
        startSmoothScroll(linearSmoothScroller);
    }
```

发现其实是实例化了一个``LinearSmoothScroller``，然后调用``startSmoothScroll``将``LinearSmoothScroller``传入进去。在[StackOverflow](https://stackoverflow.com/questions/31235183/recyclerview-how-to-smooth-scroll-to-top-of-item-on-a-certain-position/32819067)上看到一个回答，复写了``getVerticalSnapPreference``方法，返回``SNAP_START``，由于我的``RecyclerView``是水平滑动的，于是复写了``getHorizontalSnapPreference``返回``SNAP_START``，测试发现对可见的item也有滑动效果了，可是会有闪烁的现象。

继续看``LinearSmoothScroller``源码，发现有``calculateSpeedPerPixel``方法，默认是用``25``去计算，复写这个方法，换一个大点的数去计算，发现滑动的速度慢下来了，闪烁的现象消失了。具体的代码可以参考[WeeklyCalendarView](https://github.com/flyfire/WeeklyCalendarViewDemo/blob/master/weeklycalendarview/src/main/java/com/solarexsoft/weeklycalendarview/WeeklyCalendarView.java)。

问题虽然解决了，但是``RecyclerView``与各个插件的协同工作机制，``LayoutManager``,``SmoothScroller``原理没来的及分析，这个留待以后分析。

