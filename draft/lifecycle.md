---
title: "Lifecycle"
date: 2022-03-09T14:38:59+08:00
draft: false
---

事件触发会改变状态，``Event``一共有``ON_CREATE``,``ON_START``,``ON_RESUME``,``ON_PAUSE``,``ON_STOP``,``ON_DESTROY``,``ON_ANY``

状态也分为几种

```java
public enum State {
        /**
         * Destroyed state for a LifecycleOwner. After this event, this Lifecycle will not dispatch
         * any more events. For instance, for an {@link android.app.Activity}, this state is reached
         * <b>right before</b> Activity's {@link android.app.Activity#onDestroy() onDestroy} call.
         对Activity来说，在onDestroy方法调用之前达到。
         */
        DESTROYED,

        /**
         * Initialized state for a LifecycleOwner. For an {@link android.app.Activity}, this is
         * the state when it is constructed but has not received
         * {@link android.app.Activity#onCreate(android.os.Bundle) onCreate} yet.
         对Activity来说，在Activity创建出来，调用onCreate方法之前达到INITIALIZED状态
         */
        INITIALIZED,

        /**
         * Created state for a LifecycleOwner. For an {@link android.app.Activity}, this state
         * is reached in two cases:
         * <ul>
         *     <li>after {@link android.app.Activity#onCreate(android.os.Bundle) onCreate} call;
         *     <li><b>right before</b> {@link android.app.Activity#onStop() onStop} call.
         * </ul>
         对Activity来说，在onCreate方法调用之后或者onStop调用之前达到CREATED状态
         */
        CREATED,

        /**
         * Started state for a LifecycleOwner. For an {@link android.app.Activity}, this state
         * is reached in two cases:
         * <ul>
         *     <li>after {@link android.app.Activity#onStart() onStart} call;
         *     <li><b>right before</b> {@link android.app.Activity#onPause() onPause} call.
         对Activity来说，在onStart调用之后或onPause调用之前达到STARTED状态
         * </ul>
         */
        STARTED,

        /**
         * Resumed state for a LifecycleOwner. For an {@link android.app.Activity}, this state
         * is reached after {@link android.app.Activity#onResume() onResume} is called.
         对Activity来说，在onResume调用之后达到RESUMED状态。
         */
        RESUMED;
}
```

``LifecycleRegistry``为代码重入做了很多工作，比如在添加Observer的时候改变了LifecycleOwner的状态或者在添加Observer的时候到达了某个状态的时候又添加了Observer。

观察者模式分发时要解决的三个问题

+ 分发时改变了状态

+ 分发时增加了新的观察者

+ 分发保证一定的次序

分发过程中又改了状态，分发过程中又增加了新的观察者。

``moveToState``->``sync``->``while(!isSynced())只要不平衡一直循环下去``

``backward``或者``forward``的时候在``observer.dispatchEvent``的时候可能更改state或者添加新的observer

```java
void dispatchEvent(LifecycleOwner owner, Event event) {
        State newState = getStateAfter(event);
        mState = min(mState, newState);
        mLifecycleObserver.onStateChanged(owner, event);
        mState = newState; //onStateChanged调用之后才更改状态
}
```