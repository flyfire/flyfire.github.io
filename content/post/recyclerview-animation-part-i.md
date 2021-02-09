+++
title = "Recyclerview Animation Part I"
date = "2016-02-11"
slug = "2016/02/11/recyclerview-animation-part-i"
Categories = ["dev", "android"]
+++
ListView is one of the most popular widgets of the Android Framework. It has many features, yet it is fairly complex and hard to modify. As the UX paradigms evolved and phones got faster, its limitations started to overshadow its feature set.

With Lollipop, the Android team decided to release a new widget that will make writing different collection views much easier with a pluggable architecture. Many different behaviors can be controlled easily by implementing simple contracts to change:

+ how items are laid out
+ animations!
+ item decorations
+ recycling strategy

This great flexibility comes with the additional complexity of a bigger architecture. Also, there are more things to learn.

<!-- more -->

In this post, I want to deep dive into RecyclerView internals, particularly on how animations work.

On Honeycomb, the Android Framework introduced LayoutTransition, which was a very easy way to animate changes inside a ViewGroup. It works by taking a snapshot of the ViewGroup before and after the layout changes, then creating Animators to move between these two states. This process is fairly similar to what RecyclerView needs to do to animate changes in the adapter.

<center><img src="/images/trans_man_default.gif" alt="LayoutTransition example"></center>

Unfortunately, lists have one major difference which makes LayoutTransitions a bad fit for their animations. Specifically, items in lists are not the same as views in a ViewGroup. This is an important distinction that needs to be understood to handle animations on “items” while using mechanisms that animate the “views” that show the item contents.

In a normal ViewGroup, if a View is newly added to the View hierarchy, it can be treated like a newly added View and thus it can be animated accordingly (e.g. fade in). For collections, it is a bit different. For example, a View for an item may become visible just because an item before it has been removed from the Adapter. In this case, running a fade in animation for the new item would be misleading because it was already in the list though the view is new because the item just entered the viewport. RecyclerView knows if the item is new or not but it does not know where it was if the item is not new. The same case is valid for disappearing Views, RecyclerView does not know where the view went if it is not removed from the Adapter.

<center><img src="/images/trans_man_bad.gif" alt="LayoutTransition failure for a list"></center>

To overcome this problem, ``RecyclerView`` could ask ``LayoutManager`` for the previous location of the new View. Although this would work, it would require some bookkeeping on the ``LayoutManager`` end and may not be trivial to calculate for more complex ``LayoutManagers``.

The way that ``RecyclerView`` handles animating appearing and disappearing items (that is, animating the appearance and disappearance of views that refer to items that were and are still in the list) is by relying on the ``LayoutManager`` to handle predictive layout logic. One one hand, the ``RecyclerView`` wants to know where views would have been had they been laid out prior to this change. On the other hand, the ``RecyclerView`` wants to know where views would be laid out after this change if the ``LayoutManager`` went to the trouble of laying out items that are not currently visible.

To make it easy for the ``LayoutManager`` to provide this information, ``RecyclerView`` uses a two step layout process when there are adapter changes which should be animated. The mechanisms for handling these predictive layout passes are described below.

+ In the first layout pass (preLayout), RecyclerView asks LayoutManager to layout the previous state with the knowledge of the additional information. For the example above, it would be like requesting “Layout items again, btw, ‘C’ has been removed”. The LayoutManager runs its usual layout step but knowing that ‘C’ will be removed, it lays out View(s) to fill the space left by ‘C’.The cool part of this contract is that RecyclerView still behaves as if ‘C’ is still in the backing Adapter. For example, when LayoutManager asks for the View for position 2, RecyclerView returns ‘C’ (``getViewForPosition(2) == View('C')``) and if LayoutManager asks for position 4, RecyclerView returns the View for ‘E’ (although ‘D’ is the 4th item in the Adapter). LayoutParams of the returned View has an ``isItemRemoved`` method which LayoutManager can use to check if this is a disappearing item.
+ In the second layout pass (postLayout), RecyclerView asks LayoutManager to re-layout its items. This time, ‘C’ is not in the Adapter anymore. ``getViewForPosition(2)`` will return ‘D’ and ``getViewForPosition(4)`` will return ‘F’.Keep in mind that the backing item for ‘C’ was already removed from the Adapter, but since RecyclerView has the View representation of it, it can behave as if ‘C’ is still there. In other words, RecyclerView does the bookkeeping for the LayoutManager.

Every time ``onLayoutChildren`` is called on the ``LayoutManager``, it temporarily detaches all views and lays them out from scratch again. Unchanged Views are returned from the scrap cache so their measurements stay valid, making this relayout fairly cheap and simple.

<center><img src="/images/pre-layout.jpeg" alt="LinearLayoutManager pre layout result"></center>

<center><img src="/images/post-layout.jpeg" alt="LinearLayoutManager post layout result"></center>

After these two layout passes, RecyclerView knows where the Views came from so it can run the correct animation.

<center><img src="/images/predictive_animations.gif" alt="predictive animations"></center>

You might ask: The View ‘C’ was not laid out by the LayoutManager, how come it is still visible?

To be clear, ‘C’ was laid out by the LayoutManager in the pre-layout pass because it looked like it was in the Adapter. It is true that ‘C’ was not laid out by the LayoutManager in the post-layout pass because it does not exist in the Adapter anymore. It is also true for the LayoutManager that ‘C’ is not its child anymore but not true for the RecyclerView. When a View is removed by the LayoutManager, if ItemAnimator wants to animate it, RecyclerView keeps it as a child (so that animations can run properly). More details on this in Part2.

## Disappearing Items

With these two layout passes, RecyclerView is able to animate new Views properly. But now, there is another problem with Views that are disappearing. Consider the following case where a new item is added to the list, pushing some other items outside the visible area. This is how it would look with LayoutTransitions:

<center><img src="/images/layout_transition_add.gif" alt="layout transition add"></center>

When ‘X’ was added after ‘A’, it pushed ‘F’ outside the screen. Since LayoutManager will not layout ‘F’, LayoutTransition thinks it has been removed from the UI and runs a fade out animation for it. In reality, ‘F’ is still in the adapter but has been pushed out of bounds.

To solve this issue, RecyclerView provides an additional API to LayoutManager to get this information. At the end of a postLayout pass, LayoutManager can call ``getScrapList`` to get list of Views which are in this situation (not laid out by the ``LayoutManager`` but still present in the ``Adapter``). Then, it lays out these views as well, as if the size of RecyclerView was big enough to show them.

<center><img src="/images/add_post_layout_with_frame.png"></center>

One important detail is that, since these Views are not necessary after their animations are complete, ``LayoutManager`` calls ``addDisappearingView`` instead of ``addView``. This gives the clue to the RecyclerView that this View should be removed after its animations is complete. RecyclerView also adds the View to the list of hidden views so that it will disappear from LayoutManager’s children list as soon as postLayout method returns. This way, LayoutManager can forget about it.

<center><img src="/images/predictive_add.gif"></center>

Initially, at least for a LinearLayoutManager, you might think that it can calculate where the Views came from or where they went (if disappeared) and thus won’t need a two pass layout calculation. Unfortunately, there are many edge cases when multiple types of adapter changes happen in the same layout pass. In addition to that, for a more complex LayoutManager, it is not always trivial to calculate where an item would be placed (e.g. StaggeredGridLayout). This approach removes all burden from the LayoutManager and it can support proper animations with little effort.

So far, I’ve covered the main idea on how predictive animations run in RecyclerView. There is actually a lot more going on to achieve this simplicity (for the LayoutManager). You can read about how all this works in Part 2 – Behind The Scenes.

## reference

+ [RecyclerView Animations Part 1 – How Animations Work](http://www.birbit.com/recyclerview-animations-part-1-how-animations-work/)
