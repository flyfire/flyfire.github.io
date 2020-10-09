---
layout: post
title: "RecyclerView Animation part II"
date: 2016-02-12 00:47
comments: true
categories: 
- dev
- android
tags:
- android
- recyclerview
---
In the first article, I’ve covered the main idea on how predictive animations run in RecyclerView. There is actually a lot more going on to achieve this simplicity (for the LayoutManager). Here are some important points that you should know about.

RecyclerView keeps some children attached although they have been removed by the LayoutManager. How does it work? Does it invalidate the contract between the LayoutManager and RecyclerView?

Yes it does ‘kind of’ violate the contract with LayoutManager, but:

RecyclerView does keep the View as a child of the ViewGroup but hides it from the LayoutManager. Each time LayoutManager calls a method to access its children, RecyclerView takes into account the hidden Views. Lets look at the example at Part 1 where ‘C’ was being removed from the adapter.

<center><img src="/images/predictive_animations.gif"></center>

<!-- more -->

While ‘C’ is fading out, if LayoutManager calls ``getChildCount()``, RecyclerView returns 6 although it has 7 children. If ``LayoutManager`` calls ``getChildAt(int)``, RecyclerView offsets that call properly to skip child ‘C’ (or any hidden children). If LayoutManager calls ``addView(View, position)``, RecyclerView offsets the index properly before calling ``ViewGroup#addView``.

When the animation ends, RecyclerView will remove the View and recycle it.

For more details, you can check ``ChildHelper`` internal class.

How does RecyclerView handle item positions in the preLayout pass since they don’t match Adapter contents?

This is doable thanks to the specific notify events added to the Adapter. When Adapter dispatches ``notify**`` events, RecyclerView records them and requests a layout to apply them. Any events that arrives before the next layout pass will be applied together.

When ``onLayout`` is called by the system, RecyclerView does the following:

- Reorder update events such that move events are pushed to the end of the list of update ops. Moving move events to the end of the list is a simplification step so I’ll not go into details here. You can check ``OpReorderer`` class for details if you are interested.
- Process events one by one and update existing ViewHolders’ positions with respect to the update. If a ViewHolder is removed, it is also marked as ``removed``. While doing this, RecyclerView also decides whether the adapter change should be dispatched to the LayoutManager before or after the preLayout step. This decision process is as follows: 
  + If it is an ``add`` operation, it is deferred because item should not exist in preLayout.
  + If it is an ``update`` or ``remove`` operation and if it affects existing ViewHolders, it is postponed. If it does not effect existing ViewHolders, it is dispatched to the LayoutManager because RecyclerView cannot resurrect the previous state of the item (because it does not have a ViewHolder that represents the previous state of that Item).
  + If it is a ``move`` operation, it is deferred because RecyclerView can fake its location in the pre-layout pass. For example, if item at position 3 moved to position 5, RecyclerView can return the View for position 5 in pre-layout when View for position 3 is asked.
  + RecyclerView rewrites update operations as necessary. For example, if an update or delete operation affects some of the ViewHolders, RecyclerView divides that operation. If an operation should be dispatched to LayoutManager but a deferred operation may affect it, RecyclerView re-orders these operations so that they are still consistent.

For example, if there is an Add 1 at 3 operation which is deferred followed by a Remove 1 at 5 operation which cannot be deferred, RecyclerView dispatches it to the LayoutManager as Remove 1 at 4. This is done because the original Remove 1 at 5 was notified by the Adapter after Add 1 at 3 so it includes that item. Since RecyclerView did not tell LayoutManager about the Add 1 at 3, it rewrites the remove operation to be consistent.

This approach makes tracking items dead simple for a LayoutManager. The abstraction between the Adapter and the LayoutManager makes all of this possible, which is why RecyclerView never passes the Adapter to the LayoutManager, instead, provides methods to access Adapter via State and Recycler.

ViewHolders also have their old position, pre layout position and final adapter positions. When ``ViewHolder#getPosition`` is called, they return either preLayout position or final adapter position depending on the current layout state (pre or post). LayoutManager doesn’t need to know about this because it will always be consistent with the previous events that were dispatched to the LayoutManager.

- After Adapter updates are processed, RecyclerView saves positions and dimensions of existing Views which will later be used for animations.
- RecyclerView calls ``LayoutManager#onLayoutChildren`` for the preLayout step. As I’ve mentioned in the first article, LayoutManager runs its regular layout logic. All it has to do is to layout more items for those which are being deleted or changed (``LayoutParams#isItemRemoved`` , ``LayoutParams#isItemChanged``). As a reminder, the deleted or changed item still ‘appears’ in the Adapter API given to the LayoutManager. This way, LayoutManager simply treats it as any other View (adds, measures, positions etc).
- After preLayout is complete, RecyclerView records the positions of the Views again and dispatches the remaining Adapter updates to the LayoutManager.
- RecyclerView calls LayoutManager’s ``onLayout`` again (postLayout). This time, all item positions match the current contents of the Adapter. LayoutManager runs its regular layout logic again.
- After post layout is complete, RecyclerView checks positions of Views again and decides which items are added, removed, changed and moved. It ‘hides’ removed Views and for views not added by the LayoutManager, adds them to the RecyclerView (because they should be animated).
- Items which require an animation are passed to the ``ItemAnimator`` to start their animations. After the animation is complete, Item Animator calls a callback in RecyclerView which removes and recycles the View if it is no longer necessary.

###What happens if LayoutManager keeps some internal data structure using item positions?

Everything works… kind of :). Thanks to the re-writing of Adapter updates by the RecyclerView, all LayoutManager has to do is to update its own bookkeeping when one of its adapter data changed callbacks is called due to Adapter changes. RecyclerView ensures that these updates are called at the appropriate time and order.

At any time during a layout, if LayoutManager needs to access the adapter for additional data (some custom API), it can call ``Recycler#convertPreLayoutPositionToPostLayout`` to get the item’s Adapter position. For example, GridLayoutManager uses this API to get the span size of items.

###What happens if notifyDataSetChanged is called? How do predictive animations run?

They don’t, which is why ``notifyDataSetChanged`` should be your last resort. When ``notifyDataSetChanged`` is called on the adapter, RecyclerView does not know where items moved so it cannot properly fake ``getViewForPosition`` calls. It simply runs animations as a LayoutTransition would do.

##reference
+ [RecyclerView Animations Part 2 – Behind The Scenes](http://www.birbit.com/recyclerview-animations-part-2-behind-the-scenes/)

