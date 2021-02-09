+++
title = "Summary of Custom Views Part Iii"
date = "2019-03-12"
slug = "2019/03/12/summary-of-custom-views-part-iii"
Categories = ["dev", "android"]
+++

## 自定义View总结 - 触摸反馈

之前分析了[Android触摸事件分发机制](http://solarex.github.io/blog/2018/03/25/android-touch-system/)，在自定义View的时候进行触摸反馈，一般都是重写``onTouchEvent``，当然也有一些工具类可以使用，本文就对这些工具类进行总结，他们是``ViewConfiguration``，``Scroller``，``OverScroller``，``VelocityTracker``，``GestureDetector``，``ScaleGestureDetector``，``ViewDragHelper``。

<!-- more -->

### ViewConfiguration

``ViewConfiguration``定义了一些UI系统用用到的常量，包括timeouts,sizes,distances。timeouts比如``DEFAULT_LONG_PRESS_TIMEOUT``,``DOUBLE_TAP_TIMEOUT``等，sizes包括``SCROLL_BAR_SIZE``等，distances我们平时自定义View的时候可能用的比较多，常用的有``getScaledTouchSlop``来判断是否是滑动，``getScaledPagingTouchSlop``来判断是否是翻页滑动，自己写``ViewPager``的时候可以用到，``getScaledMaximumFlingVelocity``和``getMinimumFlingVelocity``来对惯性滑动进行判断处理。

### Scroller & OverScroller

``View``的``scrollTo``和``scrollBy``是瞬间完成的，如果需要``View``的滑动有个动画效果，说白了，就是View的位置移动有段时间间隔，可以使用``Scroller``或``OverScroller``来完成。``Scroller``本身无法让View滑动，它主要是个计算器，得配合View的``computeScroll``使用才能完成这个功能，或者不使用View的``computeScroll``，你自己写个``Runnable``，在``Runnable``里面进行``Scroller``计算完成的判断并调用View的``scrollTo``，然后再``postOnAnimation(this)``将自身传入再次调用即可。

使用``computeScroll``的样板代码如下：

```java
Scroller scroller = new Scroller(context);

// 缓慢滚动到指定位置
private void smoothScrollTo(int destX, int destY) {
    int scrollx = getScrollX();
    int dx = destX - scrollx;
    scroller.startScroll(scrollx, 0, dx, 0, 1000);
    // 这步很重要，触发下面的 computeScroll
    invalidate();
}

@Override
public void computeScroll() {
    if(scroller.computeScrollOffset()) {
        scrollTo(scroller.getCurrX(), scroller.getCurrY());
        // 这步继续触发 computeScroll，Scroller会更新x,y，View继续scrollTo新位置
        postInvalidate();或者invalidate();
    }
}
```

``OverScroller``的``startScroll``，``fling``方法和``Scroller``类似，不再赘述，除此之外``OverScroller``还有一个带over参数的``fling``函数``public void fling(int startX, int startY, int velocityX, int velocityY, int minX, int maxX, int minY, int maxY, int overX, int overY)``可以滑动超出View的边界。

### VelocityTracker

速度追踪，用于追踪手指在滑动过程中的速度，包括水平和垂直方向的速度，一般配合``Scroller``的``fling``使用。它的使用过程很简单，首先，在View的``onTouchEvent``方法中追踪当前单击事件的速度：

```java
// onTouchEvent
VelocityTracker velocityTracker = VelocityTracker.obtain();
velocityTracker.addMovement(event);
```

接着，在手指抬起，也就是``ACTION_UP``的时候，获取速度：

```java
// onTouchEvent ACTION_UP
velocityTracker.computeCurrentVelocity(1000);
int xVel = (int) velocityTracker.getXVelocity();
int yVel = (int) velocityTracker.getYVelocity();
/* do something like fling */
velocityTracker.clear();
velocityTracker.recycle(); // 重置并回收
```

速度的计算公式是``速度=(终点位置 - 起点位置)/时间段``，所以逆着手机坐标系的正方向滑动，所产生的速度为负值。另外记得要重置并回收``VelocityTracker``。

### GestureDetector & ScaleGestureDetector

``GestureDetector``手势检测，用于辅助检测用户的单击、滑动、长按、双击等行为。``ScaleGestureDetector``主要是双指或多指的pinch zoom放大缩小行为。要使用``GestureDetector``也很简单，参考如下过程。

首先，创建``GestureDetector``对象并实现``GestureDetector.OnGestureListener``接口，根据需要也可以实现``GestureDetector.OnDoubleTapListener``接口或者``GestureDetector.OnContextClickListener``接口，或者使用``SimpleOnGestureListener``来在自己感兴趣的方法中做处理。

|        方法名        |                             描述                             |      所属接口       |
| :------------------: | :----------------------------------------------------------: | :-----------------: |
|        onDown        |            手指轻触屏幕，由1个``ACTION_DOWN``触发            |  OnGestureListener  
|     onShowPress      |    手指轻触屏幕，尚未松开或拖动，由1个``ACTION_DOWN``触发    |  OnGestureListener  
|    onSingleTapUp     |   手指轻触屏幕后松开，随着``ACTION_UP``触发，这是单击行为    |  OnGestureListener  
|       onScroll       | 手指按下屏幕并拖动，由1个``ACTION_DOWN``及多个``ACTION_MOVE``触发，这是拖动行为 |  OnGestureListener  
|     onLongPress      |                             长按                             |  OnGestureListener  
|       onFling        | 按下屏幕快速滑动后松开，由1个``ACTION_DOWN``多个``ACTION_MOVE``和1个``ACTION_UP``触发，快速滑动 |  OnGestureListener  
|     onDoubleTap      | 双击，由2次连续的单击组成，不可能和onSingleTapConfirmed共存  | OnDoubleTapListener 
| onSingleTapConfirmed |                        严格的单击行为                        | OnDoubleTapListener 
|   onDoubleTapEvent   | 发生了双击行为，在双击期间，``ACTION_DOWN``、``ACTION_MOVE``、``ACTION_UP``都会触发此回调 | OnDoubleTapListener 

接着，接管目标View的``onTouchEvent``方法

```java
// onTouchEvent
boolean consume = mGestureDetector.onTouchEvent(event);
return consume;
```

事件经过判断后就会回调我们实现的listener中的方法。如果只是监听滑动相关的可以自己在``onTouchEvent``方法的``ACTION_MOVE``中调用``View``的``scrollTo(x,y)``来实现View的滑动，如果是监听双击这种行为的话，就使用``GestureDetector``。

``ScaleGestureDetector``是处理放大缩小手势的，使用和``GestureDetector``类似。

|                           方法名                            |                             描述                             |        所属接口        |
| :---------------------------------------------------------: | :----------------------------------------------------------: | :--------------------: |
|   public boolean onScale(ScaleGestureDetector detector);    | 通过调用detector.getScaleFactor来获得放大的系数，来进行进一步处理，比如对ImageView的Matrix进行操作等等，返回值代表事件有没有被消费 | OnScaleGestureListener |
| public boolean onScaleBegin(ScaleGestureDetector detector); | 如果要检测放大缩小手势，返回true，类似于``ACTION_DOWN``对事件感兴趣返回true | OnScaleGestureListener |
|   public void onScaleEnd(ScaleGestureDetector detector);    | 放大或缩小结束，可以调用detector.getFocusX()或detector.getFocusY()来获取焦点 | OnScaleGestureListener |

### ViewDragHelper & View.OnDragListener

``ViewDragHelper``可以实现各种不同的滑动、拖放需求，使用参考如下过程。``ViewDragHelper``一般在自定义``ViewGroup``中使用。

首先，初始化``ViewDragHelper``，实现``ViewDragHelper.Callback``。``mViewDragHelper = ViewDragHelper.create(viewgroup, callback);``

然后，接管ViewGroup的事件处理，样板代码如下：

```java
@Override
public boolean onInterceptTouchEvent(MotionEvent ev) {
    return mViewDragHelper.shouldInterceptTouchEvent(ev);
}

@Override
public boolean onTouchEvent(MotionEvent ev) {
    mViewDragHelper.processTouchEvent(ev);
    return true;
}
```

接着，处理``computeScroll``，``ViewDragHelper``内部也是通过``Scroller``来实现平滑移动的，样板代码如下：

```java
@Override
public void computeScroll() {
    if(mViewDragHelper.continueSettling(true)) {
        ViewCompat.postInvalidateOnAnimation(this);
    }
}
```

|                            方法名                            |                             描述                             |        所属接口         |
| :----------------------------------------------------------: | :----------------------------------------------------------: | :---------------------: |
| public abstract boolean tryCaptureView(View child, int pointerId); |                哪个子View可以被拖动就返回true                | ViewDragHelper.Callback |
| public int clampViewPositionVertical(View child, int top, int dy) {     return 0; } |             限制被捕捉的View垂直方向上活动的范围             | ViewDragHelper.Callback |
| public int clampViewPositionHorizontal(View child, int left, int dx) {     return 0; } |              限制被捕捉View水平方向上活动的范围              | ViewDragHelper.Callback |
| public void onViewCaptured(View capturedChild, int activePointerId) {} |                    View被捕捉的时候被调用                    | ViewDragHelper.Callback |
| public void onViewPositionChanged(View changedView, int left, int top, int dx, int dy) {} |                被捕捉的View位置发生变化时调用                | ViewDragHelper.Callback |
|       public void onViewDragStateChanged(int state) {}       | drag state变化时调用，STATE_IDLE，STATE_DRAGGING，STATE_SETTLING | ViewDragHelper.Callback |
| public void onViewReleased(View releasedChild, float xvel, float yvel) {} |                       View被松开时调用                       | ViewDragHelper.Callback |
|  public void onEdgeTouched(int edgeFlags, int pointerId) {}  |            没有View被捕捉，父View的边缘被touch到             | ViewDragHelper.Callback |
| public boolean onEdgeLock(int edgeFlags) {     return false; } |                                                              | ViewDragHelper.Callback |
| public void onEdgeDragStarted(int edgeFlags, int pointerId) {} |                                                              | ViewDragHelper.Callback |
| public int getOrderedChildIndex(int index) {     return index; } |                                                              | ViewDragHelper.Callback |
| public int getViewHorizontalDragRange(View child) {     return 0; } |                                                              | ViewDragHelper.Callback |
| public int getViewVerticalDragRange(View child) {     return 0; } |                                                              | ViewDragHelper.Callback |

``View.OnDragListener``只有一个方法``boolean onDrag(View v, DragEvent event);``，当拖拽事件被分发到View时调用。``DragEvent``有几个状态可以在其中做处理``ACTION_DRAG_STARTED``，``ACTION_DRAG_ENDED``，``ACTION_DRAG_ENTERED``，``ACTION_DRAG_EXITED``。View开始拖动可以调用``ViewCompat.startDragAndDrop(@NonNull View v, ClipData data,View.DragShadowBuilder shadowBuilder, Object localState, int flags)``来开始拖动，这样会在View上方出现一个Shadow来表示被拖动的View。
