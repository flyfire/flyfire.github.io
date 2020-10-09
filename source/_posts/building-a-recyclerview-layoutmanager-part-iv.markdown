---
layout: post
title: "Building a RecyclerView LayoutManager part IV"
date: 2016-01-15 16:14
comments: true
categories: 
- dev
- android
tags:
- android
- recyclerview
---
While writing what was intended to be the final post of this series, a discussion of predictive animations, I ran into a number of interesting challenges that I thought warranted their own discussion. This series began as an investigation into whether RecyclerView could easily handle a layout structure that could scroll in both the horizontal and vertical axes, and how difficult it would be for the developer to build their own LayoutManager. I chose a basic grid of uniform items as the structure, thinking it would be the most straightforward to implement.


The following graphic represents the basic goal of what the implementation ought to achieve while the user scrolls around the screen.

<center><img src="/images/building_recyclerview_layoutmanager_redux.gif" width="600" height="360"></center>

<!-- more --> 

##Broken By Design

No matter how you attempt to organize the adapter positions (left-to-right, top-to-bottom, etc.), the viewport into the content will always show a broken, disconnected range of the data set. To be more precise, there are items inside the adapter range between the first and last visible positions that are also outside the visible range of views.

This is an important point, because it’s contrary to layouts that scroll on a single axis (and consequently, all the default layouts provided by the current framework). These standard widgets show the data set range in a contiguous block–from the first to the last visible position with no breaks.

The RecyclerView LayoutManager API, as it is released today, makes a few implicit assumptions about the visible data set. These assumptions tend to favor a contiguous visible range, and make producing a layout like the grid illustrated above a bit more challenging. Nowhere is this more apparent than in the predictive animation APIs. For posterity’s sake, I felt compelled to explain where some of those shortcomings came into play during this process.

###Assumption #1: Removing an item from outside the visible range doesn’t affect the currently visible views.

When you think about the way LinearLayoutManager or GridLayoutManager react when an adapter item is removed, both are fairly similar at a high level. If the removed item is currently visible, a space will be left empty that needs to be filled with the surrounding views. This means extra appearing views must be laid out to fill the gap. However, there isn’t really a case where a removal would send disappearing views off-screen…the only disappearing views are those explicitly removed. If the removed view is outside the visible range (either before or after), it won’t affect the visible layout at all–there are no views appearing or disappearing. In these cases you typically see no animation. All that may change is the absolute positioning of that data block within the larger range.

The above cases, as stated, are also mostly true for a disconnected layout. However, the discontinuous nature of the visible range allows for items to be removed that are inside the visible range without actually being visible! Stated another way, their position is between the first and last visible position, but the item view is not currently in the layout. The consequence is that item removals which happen off-screen can and will affect both appearing and disappearing views that you need to animate in that layout pass.

Pre-layout is the critical phase of a ``RecyclerView`` animation when you have the chance to lay out appearing views. To assist you, ``RecyclerView`` returns views back (including the removed ones) by their initial position values so you can lay contents out in their initial state. However, when view removals don’t intersect with the visible range, ``RecyclerView`` instead returns views by their final position values. This makes handling appearing views in this case much more difficult without additional bookkeeping…difficult, but doable.

For ``FixedGridLayoutManager``, as we saw in the last post, we were required us to listen to the ``onItemsRemoved()`` callback in addition to parsing through visible views to find removals and properly handle all appearing view cases. ``RecyclerView`` made sure that this callback came before pre-layout when we needed it (the off-screen case), even though it comes after pre-layout otherwise. ``RecyclerView`` does this to avoid conflicting the posting of these events with your layout–the timing of it was just a happy accident for us.

We also had to track the fact that visible removals would offset the view positions in a way we expected, but off-screen removals would not. This is why the removals were marked with different types. A snippet left out of the last post shows that we would supply a manual offset back to the appearing view logic when the removals were off-screen…so the positions would match what they were when the removal was visible.

```java
private void fillGrid(int direction, int emptyLeft, int emptyTop, RecyclerView.Recycler recycler,
        boolean preLayout, SparseIntArray removedPositions) {
    …

    for (int i = 0; i < getVisibleChildCount(); i++) {
        int nextPosition = positionOfIndex(i);

        /*
         * When a removal happens out of bounds, the pre-layout positions of items
         * after the removal are shifted to their final positions ahead of schedule.
         * We have to track off-screen removals and shift those positions back
         * so we can properly lay out all current (and appearing) views in their
         * initial locations.
         */
        int offsetPositionDelta = 0;
        if (preLayout) {
            int offsetPosition = nextPosition;

            for (int offset = 0; offset < removedPositions.size(); offset++) {
                //Look for off-screen removals that are less-than this
                if (removedPositions.valueAt(offset) == REMOVE_INVISIBLE
                        && removedPositions.keyAt(offset) < nextPosition) {
                    //Offset position to match
                    offsetPosition--;
                }
            }
            offsetPositionDelta = nextPosition - offsetPosition;
            nextPosition = offsetPosition;
        }

        if (nextPosition < 0 || nextPosition >= getItemCount()) {
            //Item space beyond the data set, don't attempt to add a view
            continue;
        }

        …

        if (i % mVisibleColumnCount == (mVisibleColumnCount - 1)) {
            leftOffset = startLeftOffset;
            topOffset += mDecoratedChildHeight;

            //During pre-layout, on each column end, apply any additional appearing views
            if (preLayout) {
                layoutAppearingViews(recycler, view, nextPosition, removedPositions.size(), offsetPositionDelta);
            }
        } else {
            leftOffset += mDecoratedChildWidth;
        }
    }

    …
}
```

This ``offsetPositionDelta`` value was then passed to ``layoutAppearingViews()`` as a global offset to what the real row/column position were that we should be using during pre-layout. This offset would not need to exist if not for this additional bookkeeping requirement.

###Assumption #2: Adding a new item only results in disappearing sibling views, not appearing views.

With item additions, the reverse is true. If the new item should be visible when added, standard layout managers will push disappearing views off-screen to make room. There isn’t really a case where this action would also trigger one or more sibling views to slide into place as appearing children. As with removals, an addition outside the visible range doesn’t really have any bearing on the visible views, so no animation is typically in play.

For ``FixedGridLayoutManager``, or any disconnected range layout, it doesn’t really matter if the addition happens inside or outside the visible range. In both cases we would need to manage possible appearing and disappearing views. The same option we used for remove is not available to us because ``onItemsAdded()`` is always called after pre-layout…we don’t get our happy accident this time around.

Without that callback, we don’t really have much to go on during pre-layout when it comes to an add. It becomes a compromise between laying out extra views in hopes that we need them, and not laying out so many extra views we damage performance. ``FixedGridLayoutManager`` does not support predicting appearing views during an item add.

##Just the Beginning…

The RecyclerView APIs are very new, and there are tons of changes already in the works with more to follow after that. They are also extremely complex, and hard to get right. For every amount of effort RecyclerView requires of you, it is doing 10x more behind the scenes. These types of growing pains are expected. Hopefully those of you trying to do similar things find this as a caution that saves you time, while we both wait for the framework to mature.