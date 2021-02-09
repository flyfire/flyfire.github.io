+++
title = "Android Custom View"
date = "2015-05-05"
slug = "2015/05/05/android-custom-view"
Categories = ["dev", "android"]
+++
<p><center><img src="/images/android_training.jpg"/></center></p>

<!-- more --> 

<p><center><iframe width="560" height="315" src="https://www.youtube.com/embed/NYtB6mlu7vA" frameborder="0" allowfullscreen></iframe></center></p>

### Subclass View
+ onMeasure
  + MeasureSpec
  + setMeasureDimension
+ onLayout
  + layout(left, top, right, bottom)
+ onDraw
  + Canvas

While viewgroups dont generally draw any content of their own,there are many situations where this could be useful,there are help instances where we can ask viewgroup to do some drawing.The first is inside the method ``dispatchDraw()`` after ``super.dispatchDraw`` has been called,at this stage,child views have already been drawn and we have an opportunity to do additional drawing on top.The second is using the same ``onDraw()`` callback as in ``view``.Anything we draw here will be drawn before the child view and thus will show us underneath them,this can be helpful for drawing any type of dynamic backgrounds or selector states if you wish to put code into the ``onDraw`` of a ViewGroup.Dont forget to enable drawing callbacks of ViewGroup by ``setWillNotDraw(false)``,otherwise ViewGroup's ``onDraw`` callback will never be triggered,this is because ViewGroup by default has its ``onDraw`` callback disabled.

### Applying Custom Attributes

### LayoutParams
+ ``checkLayoutParams(ViewGroup.LayoutParams)``
+ ``generateLayoutParams(ViewGroup.LayoutParams)``
+ ``generateLayoutParams(AttributeSet)``
+ ``generateDefaultLayoutParams()``

### Handling Layout Events

+ ``onSizeChanged``

### Making the view interactive

#### Handling Input Gestures

### Use hardware acceleration
You can control hardware acceleration at the following levels:

+ ``Application level``,``<application android:hardwareAccelerated="true" ...>``
+ ``Activity level``,``<activity android:hardwareAccelerated="false" />``
+ ``Window level``,``getWindow().setFlags(WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED,WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED);``
+ ``View level``,``myView.setLayerType(View.LAYER_TYPE_SOFTWARE, null);``

If you must check whether your view is hardware acclerated in your drawing code,use ``Canvas.isHardwareAccelerated()`` instead of ``View.isHardwareAccelerated()`` when possible. When a view is attached to a hardware accelerated window, it can still be drawn using a non-hardware accelerated ``Canvas``. This happens, for instance, when drawing a view into a bitmap for caching purposes.

