---
layout: post
title: "Building a RecyclerView LayoutManager part II"
date: 2016-01-11 00:48
comments: true
categories: 
- dev
- android
tags:
- android
- recyclerview
---

In the last post,  we walked through the core functionality necessary for building a RecyclerView LayoutManager. In this post, we are going to add support for a few additional features that the average adapter-based view is expected to have.A reminder that the entire sample application can be found [here on GitHub](https://github.com/devunwired/recyclerview-playground).

##Supporting Item Decorations

``RecyclerView`` has a really neat feature in which an ``RecyclerView.ItemDecoration`` instance can be supplied to do custom drawing alongside the child view content, as well as provide insets (margins) that will apply to the child views without the need for modifying layout parameters. The latter places a constraint on how the children should be laid out that the ``LayoutManager`` implementation must support.The [RecyclerPlayground](https://github.com/devunwired/recyclerview-playground) repository uses a few different decorators in the examples to illustrate how they are implemented.

<!-- more -->

LayoutManager gives us helper methods to account for decorations so we don’t have to think about them:

+ To get the left edge of a child view, use ``getDecoratedLeft()`` instead of ``child.getLeft()``
+ To get the top edge of a child view, use ``getDecoratedTop()`` instead of ``child.getTop()``
+ To get the right edge of a child view, use ``getDecoratedRight()`` instead of ``child.getRight()``
+ To get the bottom edge of a child view, use ``getDecoratedBottom()`` instead of ``child.getBottom()``
+ Use ``measureChild()`` or ``measureChildWithMargins()`` instead of ``child.measure()`` to measure new views coming from the ``Recycler``.
+ Use ``layoutDecorated()`` instead of ``child.layout()`` to lay out new views coming from the ``Recycler``.
+ Use ``getDecoratedMeasuredWidth()`` or ``getDecoratedMeasuredHeight()`` instead of ``child.getMeasuredWidth()`` or ``child.getMeasuredHeight()`` to get the measurements of a child view.

As long as you take into account using the proper methods for getting view properties and measurments, ``RecyclerView`` will handle dealing with decorations so you don’t have to.

##Data Set Changes
When the attached ``RecyclerView.Adapter`` triggers an update via ``notifyDataSetChanged()``, the ``LayoutManager`` will be responsible for updating the layout in the view. In this case, ``onLayoutChildren()`` will be called again. To support this we need to make some adjustments to our sample to make the distinction between a fresh layout and a layout change due to an adapter update. Below is the fully fleshed out method from the ``FixedGridLayoutManager``:

```java
@Override
public void onLayoutChildren(RecyclerView.Recycler recycler, RecyclerView.State state) {
    //We have nothing to show for an empty data set but clear any existing views
    if (getItemCount() == 0) {
        detachAndScrapAttachedViews(recycler);
        return;
    }

    //...on empty layout, update child size measurements
    if (getChildCount() == 0) {
        //Scrap measure one child
        View scrap = recycler.getViewForPosition(0);
        addView(scrap);
        measureChildWithMargins(scrap, 0, 0);

        /*
         * We make some assumptions in this code based on every child
         * view being the same size (i.e. a uniform grid). This allows
         * us to compute the following values up front because they
         * won't change.
         */
        mDecoratedChildWidth = getDecoratedMeasuredWidth(scrap);
        mDecoratedChildHeight = getDecoratedMeasuredHeight(scrap);

        detachAndScrapView(scrap, recycler);
    }

    updateWindowSizing();

    int childLeft;
    int childTop;
    if (getChildCount() == 0) { //First or empty layout
        /*
         * Reset the visible and scroll positions
         */
        mFirstVisiblePosition = 0;
        childLeft = childTop = 0;
    } else if (getVisibleChildCount() > getItemCount()) {
        //Data set is too small to scroll fully, just reset position
        mFirstVisiblePosition = 0;
        childLeft = childTop = 0;
    } else { //Adapter data set changes
        /*
         * Keep the existing initial position, and save off
         * the current scrolled offset.
         */
        final View topChild = getChildAt(0);
        if (mForceClearOffsets) {
            childLeft = childTop = 0;
            mForceClearOffsets = false;
        } else {
            childLeft = getDecoratedLeft(topChild);
            childTop = getDecoratedTop(topChild);
        }

        /*
         * Adjust the visible position if out of bounds in the
         * new layout. This occurs when the new item count in an adapter
         * is much smaller than it was before, and you are scrolled to
         * a location where no items would exist.
         */
        int lastVisiblePosition = positionOfIndex(getVisibleChildCount() - 1);
        if (lastVisiblePosition >= getItemCount()) {
            lastVisiblePosition = (getItemCount() - 1);
            int lastColumn = mVisibleColumnCount - 1;
            int lastRow = mVisibleRowCount - 1;

            //Adjust to align the last position in the bottom-right
            mFirstVisiblePosition = Math.max(
                    lastVisiblePosition - lastColumn - (lastRow * getTotalColumnCount()), 0);

            childLeft = getHorizontalSpace() - (mDecoratedChildWidth * mVisibleColumnCount);
            childTop = getVerticalSpace() - (mDecoratedChildHeight * mVisibleRowCount);

            //Correct cases where shifting to the bottom-right overscrolls the top-left
            // This happens on data sets too small to scroll in a direction.
            if (getFirstVisibleRow() == 0) {
                childTop = Math.min(childTop, 0);
            }
            if (getFirstVisibleColumn() == 0) {
                childLeft = Math.min(childLeft, 0);
            }
        }
    }

    //Clear all attached views into the recycle bin
    detachAndScrapAttachedViews(recycler);

    //Fill the grid for the initial layout of views
    fillGrid(DIRECTION_NONE, childLeft, childTop, recycler);
}
```

Our implementation determines if this is a new layout or an update based on whether we have child views attached already. In the case of an update, the first visible position (i.e. the top-left view, which we track continuously) and the current scrolled x/y offset give us enough information to do a new ``fillGrid()`` while preserving that the same item position remain in the top-left.

There are a few special cases we handle as well.

+ When the new data set is too small to scroll, the layout is reset with position 0 in the top-left.
+ If the new data set is smaller, and preserving the current position would cause the layout to be scrolled beyond the allowed boundary (on the right and/or bottom). Here we adjust the first position so the layout aligns with the bottom-right of the grid.

###onAdapterChanged()

This method provides you an additional opportunity to reset the layout in the event that the entire adapter is swapped out (i.e. setAdapter() is invoked again on the view). In this event, it’s safer to assume that the views returned will be completely different than from the previous adapter. Therefore, our example simply removes all current views (without recycling them):

```java
@Override
public void onAdapterChanged(RecyclerView.Adapter oldAdapter, RecyclerView.Adapter newAdapter) {
    //Completely scrap the existing layout
    removeAllViews();
}
```

The view removal will trigger a new layout pass, and when ``onLayoutChildren()`` is called again, our code can perform a fresh layout since there are no longer any child views attached.

##Scroll to Position
Another important feature you will likely want from your LayoutManager is the ability to tell the view to scroll to a specific position. This can be done with or without animation, and there is a callback for each.

###scrollToPosition()
This method is invoked from the RecyclerView when the layout should immediately update with the given position as the first visible item. In a vertical list, the element would be placed at the top; in a horizontal list, it would generally be on the left. In our grid, the “selected” position will be placed at the top-left of the view.

```java
@Override
public void scrollToPosition(int position) {
    if (position >= getItemCount()) {
        Log.e(TAG, "Cannot scroll to "+position+", item count is "+getItemCount());
        return;
    }

    //Ignore current scroll offset, snap to top-left
    mForceClearOffsets = true;
    //Set requested position as first visible
    mFirstVisiblePosition = position;
    //Trigger a new view layout
    requestLayout();
}
```

With a proper implementation of ``onLayoutChildren()``, this can be as simple as updating the target position and triggering a new fill.

###smoothScrollToPosition()

In the case where the selection should be animated, we need to take a slightly different approach. The contract of this method is for the ``LayoutManager`` to construct an instance of a ``RecyclerView.SmoothScroller``, and begin the animation by invoking ``startSmoothScroll()`` before the method returns.

``RecyclerView.SmoothScroller`` is an abstract class with an API that consists of four required methods:

+ ``onStart()``: Triggered when the scroller animation begins.
+ ``onStop()``: Triggered when the scroller animation ends.
+ ``onSeekTargetStep()``: Invoked incrementally as the scroller searches for the target view. The implementation is responsible for reading the provided dx/dy and updating how far the view should actually scroll in both directions.
  + A ``RecyclerView.SmoothScroller.Action`` instance is passed to this method. Notify the view how it should animate the next increment by passing a new dx, dy, duration, and ``Interpolator`` to the action’s ``update()`` method.
  + NOTE: The framework will warn you if you are taking too long to animate (i.e. your increments are too small); try to tune your animation steps to match a standard animation duration from the framework.
+ ``onTargetFound()``: Called only once, after a view for the target position has been attached. This is one final chance to animate the target view to its exact position.
  + Internally, this uses ``findViewByPosition()`` from the ``LayoutManager`` to determine when the view is attached. If your ``LayoutManager`` is efficient about mapping views to positions, override this method to improve performance. The default implementation iterates over all child views…all the time.

You can provide your own scroller implementation if you really want to fine-tune your scrolling animations. We have chosen to use the framework’s ``LinearSmoothScroller`` instead, which implements the callback work for us. We only need to implement a single method, ``computeScrollVectorForPosition()``, to tell the scroller the initial direction and approximate distance it needs to travel to get from its current location to the target location.

```java
@Override
public void smoothScrollToPosition(RecyclerView recyclerView, RecyclerView.State state, final int position) {
    if (position >= getItemCount()) {
        Log.e(TAG, "Cannot scroll to "+position+", item count is "+getItemCount());
        return;
    }

    /*
     * LinearSmoothScroller's default behavior is to scroll the contents until
     * the child is fully visible. It will snap to the top-left or bottom-right
     * of the parent depending on whether the direction of travel was positive
     * or negative.
     */
    LinearSmoothScroller scroller = new LinearSmoothScroller(recyclerView.getContext()) {
        /*
         * LinearSmoothScroller, at a minimum, just need to know the vector
         * (x/y distance) to travel in order to get from the current positioning
         * to the target.
         */
        @Override
        public PointF computeScrollVectorForPosition(int targetPosition) {
            final int rowOffset = getGlobalRowOfPosition(targetPosition)
                    - getGlobalRowOfPosition(mFirstVisiblePosition);
            final int columnOffset = getGlobalColumnOfPosition(targetPosition)
                    - getGlobalColumnOfPosition(mFirstVisiblePosition);

            return new PointF(columnOffset * mDecoratedChildWidth, rowOffset * mDecoratedChildHeight);
        }
    };
    scroller.setTargetPosition(position);
    startSmoothScroll(scroller);
}
```

This implementation, similar to the existing behavior of ListView, will stop scrolling as soon as the view becomes fully visible; whether that be on the left, top, right, or bottom of the ``RecyclerView``.

##Now What?

You mean that wasn’t enough? Things are starting to look pretty good! In fact, for many the implementation could be considered complete. But we’re going to go just one step further. In the next, and final post of this series, we will look at supporting animations for data set changes in your LayoutManager.

##reference
+ [Building a RecyclerView LayoutManager – Part 2](http://wiresareobsolete.com/2014/09/recyclerview-layoutmanager-2/)
