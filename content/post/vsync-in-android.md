+++
title = "Vsync in Android"
date = "2016-10-09"
slug = "2016/10/09/vsync-in-android"
Categories = ["dev", "android"]
+++
## In Android What is VSYNC and its usage?

VSYNC is the event posted periodically by the kernal at fixed interval where the input handling, Animation and Window drawing happening synchroniiously at the same VSYNC interval. Below have given the detail explanation about the VSYNC event before VSYNC is introduced and after VSYNC is introduced.

<!-- more -->

### Before VSYNC is introduced:

Before VSYNC is introduced then there was no synchronization happening for Input, animation and draw. As when there is an input it will be handled, also as when there is an animation or change in view then it will be drawn which results in too many CPU operations and some complex animations where some time operations or drawing happens exceeds human identifcatrion of view change. Suppose human eye can see clearly and differentiate 60FPS / seconds. In this case it might happen more then 60FPS some time.
 

As there is no sync happens between these 3 handling so input handling might redraw the view, animation also might redraw the view and some changes in view is also redraw. So too many redraw will happen.

### After VSYNC is introduced:

Not this VSYNC is been delivered at an interval of 16.67 MS which is around 60 FPS/second. In this VSYNC event only handling of input will be happend, if input arrived before that then it will be queued and handled at VSYNC event only. After handling this VSYNC event, will handle the animation and followed by draw. Now all these 3 handling is synchronized and handling will be initiated on VSYNC event. VSYNC event make sure handling and drawing of the window happen at fixed interval and thus by avoid unnecessary drawing and handling.

Lets check below Google butter document for VSYNC handling.[For Butter or Worse : Smoothing out performance in Android UIs](https://docs.google.com/viewer?url=http%3A%2F%2Fcommondatastorage.googleapis.com%2Fio2012%2Fpresentations%2Flive%2520to%2520website%2F109.pdf)

## reference

+ [WHAT IS VSYNC IN ANDROID?](https://nayaneshguptetechstuff.wordpress.com/2014/07/01/what-is-vsyc-in-android/)
