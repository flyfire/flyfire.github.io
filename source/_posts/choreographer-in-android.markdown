---
layout: post
title: "Choreographer in Android"
date: 2016-10-10 20:38
comments: true
categories: 
- dev
- android
tags:
- android
---
Choreographer is the one which acts like a interface between application view system and lower layer display system for rendering the views.

ViewRootImpl is the ViewParent or root below which only Activity window DecorView will be attached. All the view layouts set by the activity through ``setContentView()`` will attached the DecorView whose parent is ViewRootImpl. Actually ViewRootImpl is not a View its just a ViewParent which handled and manages the View Hierarchies for displaying, handling input events etc.

The number of root views that are active in your process. Each root view is associated with a window, so this can help you identify memory leaks involving dialogs or other windows.

<!-- more -->

ViewRootImpl will get the requests for refresh or view update from its child and interact with Choreographer for drawing the view hierarchy to the display system.

Choreographer instance will be created for each thread wise. Each application thread will have separate instance of Choreographer. When this instance is created then it itself register to for VSYNC event to the lower layer and now it ready to handle the ViewRootImpl/Application refresh request and interaction with the lower layer.

When ever ViewRootImpl got request from View hierarchy to refresh or update or invalidate then it request Choreographer for refresh by registering a callback. When Choreographer got any request then it request for next VSYNC event from the lower layer and when it got he VSYNC event then it asks ViewRootImpl by calling the registered Callback to handle the drawing accordingly. Also when ever ViewRootImpl wants to redraw the view then this thing will repeat.

When Choreographer receives VSYNC event then it handled the below event handling in order to handle the drawing or display.

+ Input  handling : All received input event received and maintained in the query and all these input event will be processed now only.

+ Animation : Handle all the registered animations from ViewRootImpl or from Application or Activity.

+ View Traversal (Drawing/Rendering of Views) : Handles the View drawing if ViewRootImpl has registered for refresh.

Choreographer is the main component which registers application main thread to display system and coordinate between the application view drawing component to the display system for synchronizing VSYNC with the application event, animation and draw handling.

##reference
+ [WHAT IS CHOREOGRAPHER IN ANDROID?](https://nayaneshguptetechstuff.wordpress.com/2014/07/01/what-is-choreographeri-in-android/)
