---
layout: post
title: "自定义View总结一"
date: 2019-01-12 00:33
comments: true
categories: 
- dev
- android
tags:
- android
---

## 自定义View总结 - 绘制

### 绘制基础

+ ``Canvas.drawColor(@ColorInt int color)`` 颜色填充
+ ``drawCircle(float centerX, float centerY, float radius, Paint paint)`` 画圆
+ `` Paint.setColor(int color)``,``Paint.setStyle(Paint.Style style)``,``Paint.setStrokeWidth(float width)``,``Paint.setAntiAlias(boolean aa)``
+ ``drawRect(float left, float top, float right, float bottom, Paint paint)`` 画矩形
+ ``drawPoint(float x, float y, Paint paint)`` 画点
+ ``drawPoints(float[] pts, int offset, int count, Paint paint) / drawPoints(float[] pts, Paint paint)`` 画点（批量）
+ ``drawOval(float left, float top, float right, float bottom, Paint paint) ``画椭圆
+ ``drawLine(float startX, float startY, float stopX, float stopY, Paint paint)`` 画线
+ ``drawLines(float[] pts, int offset, int count, Paint paint) / drawLines(float[] pts, Paint paint) `` 画线（批量）
+ ``drawRoundRect(float left, float top, float right, float bottom, float rx, float ry, Paint paint)`` 画圆角矩形
+ ``drawArc(float left, float top, float right, float bottom, float startAngle, float sweepAngle, boolean useCenter, Paint paint)`` 绘制弧形或扇形
+ ``drawPath(Path path, Paint paint)`` 画自定义图形
+ ``drawBitmap(Bitmap bitmap, float left, float top, Paint paint)`` 画 Bitmap
+ ``drawText(String text, float x, float y, Paint paint) ``绘制文字

<!-- more -->

#### Path

+ ``addCircle(float x, float y, float radius, Direction dir)`` 添加圆
+ ``addOval(float left, float top, float right, float bottom, Direction dir) / addOval(RectF oval, Direction dir) ``添加椭圆
+ ``addRect(float left, float top, float right, float bottom, Direction dir) / addRect(RectF rect, Direction dir) ``添加矩形
+ ``addRoundRect(RectF rect, float rx, float ry, Direction dir) / addRoundRect(float left, float top, float right, float bottom, float rx, float ry, Direction dir) / addRoundRect(RectF rect, float[] radii, Direction dir) / addRoundRect(float left, float top, float right, float bottom, float[] radii, Direction dir) ``添加圆角矩形
+ ``addPath(Path path)`` 添加另一个 Path
+ ``lineTo(float x, float y) / rLineTo(float x, float y) ``画直线
+ ``quadTo(float x1, float y1, float x2, float y2) / rQuadTo(float dx1, float dy1, float dx2, float dy2) ``画二次贝塞尔曲线
+ ``cubicTo(float x1, float y1, float x2, float y2, float x3, float y3) / rCubicTo(float x1, float y1, float x2, float y2, float x3, float y3)`` 画三次贝塞尔曲线
+ ``moveTo(float x, float y) / rMoveTo(float x, float y) ``移动到目标位置
+ ``arcTo(RectF oval, float startAngle, float sweepAngle, boolean forceMoveTo) / arcTo(float left, float top, float right, float bottom, float startAngle, float sweepAngle, boolean forceMoveTo) / arcTo(RectF oval, float startAngle, float sweepAngle)`` 画弧形
+ ``addArc(float left, float top, float right, float bottom, float startAngle, float sweepAngle) / addArc(RectF oval, float startAngle, float sweepAngle)``,`addArc()` 只是一个直接使用了 `forceMoveTo = true` 的简化版 `arcTo()`
+ ``close() ``封闭当前子图形
+ ``Path.setFillType(Path.FillType ft)`` 设置填充方式

### Paint详解

<center><p><img src="/images/canvas-color.jpg" alt="Canvas绘制的内容，有三层对颜色的处理"></p></center>

#### 颜色

|     canvas 方法      | 像素颜色的设置方式 |
| :------------------: | ------------------ |
| drawColor/RGB/ARGB() | 直接作为参数传入   |
| drawBitmap() | 与``bitmap``参数的像素颜色相同 |
| 图形和文字(drawCircle()/drawPath()/drawText()...) | 在``paint``参数中设置 |

##### 直接设置颜色``Paint.setColor(int color)``,``Paint.setARGB(int a,int r,int g,int b)``

##### setShader(Shader shader) 设置shader

+ shader着色器，它和直接设置颜色的区别是，着色器设置的是一个颜色方案，或者说是一套着色规则。

+ LinearGradient 线性渐变

+ RadialGradient 辐射渐变，辐射渐变很好理解，就是从中心向周围辐射状的渐变。

+ SweepGradient 扫描渐变

+ BitmapShader 用 Bitmap 来着色，其实也就是用 Bitmap 的像素来作为图形或文字的填充。

+ ComposeShader 混合着色器 所谓混合，就是把两个 Shader 一起使用。

##### setColorFilter(ColorFilter colorFilter)

为绘制设置颜色过滤。颜色过滤的意思，就是为绘制的内容设置一个统一的过滤策略，然后 Canvas.drawXXX() 方法会对每个像素都进行过滤后再绘制出来。

+ ``LightingColorFilter(int mul, int add)``

```java
R' = R * mul.R / 0xff + add.R  
G' = G * mul.G / 0xff + add.G  
B' = B * mul.B / 0xff + add.B  
```

+ ``PorterDuffColorFilter(int color, PorterDuff.Mode mode)``
+ ``ColorMatrixColorFilter``

`ColorMatrixColorFilter` 使用一个 `ColorMatrix` 来对颜色进行处理。 `ColorMatrix` 这个类，内部是一个 4x5 的矩阵：

```java
[ a, b, c, d, e,
  f, g, h, i, j,
  k, l, m, n, o,
  p, q, r, s, t ]
  
R’ = a*R + b*G + c*B + d*A + e;  
G’ = f*R + g*G + h*B + i*A + j;  
B’ = k*R + l*G + m*B + n*A + o;  
A’ = p*R + q*G + r*B + s*A + t; 
```

[StyleImageView](https://github.com/chengdazhi/StyleImageView)

##### setXfermode(Xfermode xfermode)

+ 使用离屏缓冲（Off-screen Buffer）

+ 控制好透明区域

#### 效果

##### setAntiAlias (boolean aa) 设置抗锯齿

##### setStyle(Paint.Style style)

##### 线条形状

+ ``setStrokeWidth(float width)``

+ ``setStrokeCap(Paint.Cap cap)``

+ ``setStrokeJoin(Paint.Join join)``

+ ``setStrokeMiter(float miter)``

##### 色彩优化

+ ``setDither(boolean dither)`` 设置图像的抖动

+ ``setFilterBitmap(boolean filter)`` 设置是否使用双线性过滤来绘制 Bitmap

##### setPathEffect(PathEffect effect)

使用 ``PathEffect`` 来给图形的轮廓设置效果。对 Canvas 所有的图形绘制有效，也就是  ``drawLine() drawCircle() drawPath()`` 这些方法。

+ ``CornerPathEffect`` 把所有拐角变成圆角

+ ``DiscretePathEffect`` 把线条进行随机的偏离，让轮廓变得乱七八糟。

+ ``DashPathEffect`` 使用虚线来绘制线条

+ ``PathDashPathEffect`` 这个方法比 DashPathEffect 多一个前缀 Path ，所以顾名思义，它是使用一个 Path 来绘制「虚线」。

+ ``SumPathEffect`` 这是一个组合效果类的 PathEffect 。它的行为特别简单，就是分别按照两种 PathEffect 分别对目标进行绘制。

+ ``ComposePathEffect`` 这也是一个组合效果类的 PathEffect 。不过它是先对目标 Path 使用一个 PathEffect，然后再对这个改变后的 Path 使用另一个 PathEffect。它的构造方法 ``ComposePathEffect(PathEffect outerpe, PathEffect innerpe)`` 中的两个  PathEffect 参数， innerpe 是先应用的， outerpe 是后应用的。

##### setShadowLayer(float radius, float dx, float dy, int shadowColor)

在之后的绘制内容下面加一层阴影。如果要清除阴影层，使用 clearShadowLayer() 。

+ 在硬件加速开启的情况下， setShadowLayer() 只支持文字的绘制，文字之外的绘制必须关闭硬件加速才能正常绘制阴影。

+ 如果 shadowColor 是半透明的，阴影的透明度就使用 shadowColor 自己的透明度；而如果 shadowColor 是不透明的，阴影的透明度就使用 paint 的透明度。

##### setMaskFilter(MaskFilter maskfilter)

为之后的绘制设置 `MaskFilter`。上一个方法 `setShadowLayer()` 是设置的在绘制层下方的附加效果；而这个 `MaskFilter` 和它相反，设置的是在绘制层上方的附加效果。

+ ``BlurMaskFilter`` 模糊效果的 MaskFilter。``BlurMaskFilter(float radius, BlurMaskFilter.Blur style)`` 中， radius 参数是模糊的范围， style 是模糊的类型。NORMAL: 内外都模糊绘制，SOLID: 内部正常绘制，外部模糊，INNER: 内部模糊，外部不绘制，OUTER: 内部不绘制，外部模糊。

+ ``EmbossMaskFilter`` 浮雕效果的 MaskFilter。

##### 获取绘制的 Path

根据 paint 的设置，计算出绘制 Path 或文字时的实际 Path。所谓实际 Path ，指的就是 drawPath() 的绘制内容的轮廓，要算上线条宽度和设置的 PathEffect。

+ ``getFillPath(Path src, Path dst)``，``getFillPath(src, dst)`` 方法就能获取这个实际 Path。方法的参数里，src 是原 Path ，而 dst 就是实际 Path 的保存位置。 ``getFillPath(src, dst)`` 会计算出实际 Path，然后把结果保存在 dst 里。

+ ``getTextPath(String text, int start, int end, float x, float y, Path path) / getTextPath(char[] text, int index, int count, float x, float y, Path path)`` 文字的绘制，虽然是使用 Canvas.drawText() 方法，但其实在下层，文字信息全是被转化成图形，对图形进行绘制的。  getTextPath() 方法，获取的就是目标文字所对应的 Path 

#### Paint初始化类

+ ``reset()``

+ ``set(Paint src)``

+ ``setFlags(int flags)``

### 文字的绘制

##### Canvas 绘制文字的方式

+ ``drawText(String text, float x, float y, Paint paint)``

+ ``drawTextRun()``

+ ``drawTextOnPath()``

+ ``StaticLayout``

##### Paint 对文字绘制的辅助

+ ``setTextSize(float textSize)``

+ ``setTypeface(Typeface typeface)``

+ ``setFakeBoldText(boolean fakeBoldText)`` 伪粗体（ fake bold ），因为它并不是通过选用更高 weight 的字体让文字变粗，而是通过程序在运行时把文字给「描粗」了

+ ``setStrikeThruText(boolean strikeThruText)`` 是否加删除线

+ ``setUnderlineText(boolean underlineText)`` 是否加下划线

+ ``setTextSkewX(float skewX)`` 设置文字横向错切角度。其实就是文字倾斜度的啦。

+ ``setTextScaleX(float scaleX)`` 设置文字横向放缩。也就是文字变胖变瘦。

+ ``setLetterSpacing(float letterSpacing)`` 设置字符间距。默认值是 0。

+ ``setFontFeatureSettings(String settings)``

+ ``setTextAlign(Paint.Align align)`` 设置文字的对齐方式。一共有三个值：LEFT CETNER 和 RIGHT。默认值为 LEFT。

+ ``setTextLocale(Locale locale) / setTextLocales(LocaleList locales)`` 设置绘制所使用的 Locale。

+ ``setHinting(int mode)`` 设置是否启用字体的 hinting （字体微调）。

+ ``setElegantTextHeight(boolean elegant)`` 设置是否开启文字的 elegant height 。开启之后，文字的高度就变优雅了

+ ``setSubpixelText(boolean subpixelText)`` 是否开启次像素级的抗锯齿（ sub-pixel anti-aliasing ）。

+ ``setLinearText(boolean linearText)`` 

+ ``hasGlyph(String string)`` 检查指定的字符串中是否是一个单独的字形 (glyph）。

##### 测量文字尺寸类

+ ``float getFontSpacing()`` 获取推荐的行距。

+ ``FontMetircs getFontMetrics()`` 获取 Paint 的 FontMetrics。``FontMetrics`` 是个相对专业的工具类，它提供了几个文字排印方面的数值：``ascent``,  ``descent``, ``top``, ``bottom``, ``leading``。``ascent`` 和 ``descent`` 这两个值还可以通过 ``Paint.ascent()`` 和 ``Paint.descent()`` 来快捷获取。

+ ``getTextBounds(String text, int start, int end, Rect bounds)`` 获取文字的显示范围。

+ ``float measureText(String text)`` 测量文字的宽度并返回。

如果你用代码分别使用 getTextBounds() 和 measureText() 来测量文字的宽度，你会发现  measureText() 测出来的宽度总是比 getTextBounds() 大一点点。这是因为这两个方法其实测量的是两个不一样的东西。getTextBounds: 它测量的是文字的显示范围（关键词：显示）。形象点来说，你这段文字外放置一个可变的矩形，然后把矩形尽可能地缩小，一直小到这个矩形恰好紧紧包裹住文字，那么这个矩形的范围，就是这段文字的 bounds。measureText(): 它测量的是文字绘制时所占用的宽度（关键词：占用）。前面已经讲过，一个文字在界面中，往往需要占用比他的实际显示宽度更多一点的宽度，以此来让文字和文字之间保留一些间距，不会显得过于拥挤。

+ ``getTextWidths(String text, float[] widths)`` 获取字符串中每个字符的宽度，并把结果填入参数 widths。

+ ``int breakText(String text, boolean measureForwards, float maxWidth, float[] measuredWidth)`` 这个方法也是用来测量文字宽度的。但和 measureText() 的区别是， breakText() 是在给出宽度上限的前提下测量文字的宽度。如果文字的宽度超出了上限，那么在临近超限的位置截断文字。

##### 光标相关

+ ``getRunAdvance(CharSequence text, int start, int end, int contextStart, int contextEnd, boolean isRtl, int offset)`` 对于一段文字，计算出某个字符处光标的 x 坐标。

+ ``getOffsetForAdvance(CharSequence text, int start, int end, int contextStart, int contextEnd, boolean isRtl, float advance)`` 给出一个位置的像素值，计算出文字中最接近这个位置的字符偏移量

### Canvas 对绘制的辅助 clipXXX() 和 Matrix

##### 范围裁切

范围裁切有两个方法： ``clipRect()`` 和 ``clipPath()``。裁切方法之后的绘制代码，都会被限制在裁切范围内。

+ ``clipRect()``

+ `` clipPath()``

##### 几何变换

几何变换的使用大概分为三类：

+ 使用 Canvas 来做常见的二维变换；
+ 使用 Matrix 来做常见和不常见的二维变换；
+ 使用 Camera 来做三维变换。

###### 使用 Canvas 来做常见的二维变换

+ ``Canvas.translate(float dx, float dy)`` 平移

+ ``Canvas.rotate(float degrees, float px, float py)`` 旋转

+ ``Canvas.scale(float sx, float sy, float px, float py)`` 放缩

+ ``skew(float sx, float sy)`` 错切

###### 使用 Matrix 来做变换

Matrix 做常见变换的方式：

- 创建 Matrix 对象；
- 调用 Matrix 的 ``pre/postTranslate/Rotate/Scale/Skew()`` 方法来设置几何变换；
- 使用 ``Canvas.setMatrix(matrix)`` 或 ``Canvas.concat(matrix)`` 来把几何变换应用到 Canvas。

把 Matrix 应用到 Canvas 有两个方法： ``Canvas.setMatrix(matrix)`` 和 ``Canvas.concat(matrix)``。

- ``Canvas.setMatrix(matrix)``：用 Matrix 直接替换 Canvas 当前的变换矩阵，即抛弃 Canvas 当前的变换，改用 Matrix 的变换（注：根据下面评论里以及我在微信公众号中收到的反馈，不同的系统中 setMatrix(matrix) 的行为可能不一致，所以还是尽量用  concat(matrix) 吧）；
- ``Canvas.concat(matrix)``：用 Canvas 当前的变换矩阵和 Matrix 相乘，即基于 Canvas 当前的变换，叠加上 Matrix 中的变换。

使用 Matrix 来做自定义变换

+ ``Matrix.setPolyToPoly(float[] src, int srcIndex, float[] dst, int dstIndex, int pointCount)`` 用点对点映射的方式设置变换。poly 就是「多」的意思。setPolyToPoly() 的作用是通过多点的映射的方式来直接设置变换。「多点映射」的意思就是把指定的点移动到给出的位置，从而发生形变。例如：(0, 0) -> (100, 100) 表示把 (0, 0) 位置的像素移动到 (100, 100) 的位置，这个是单点的映射，单点映射可以实现平移。而多点的映射，就可以让绘制内容任意地扭曲。

###### 使用 Camera 来做三维变换

Camera 的三维变换有三类：旋转、平移、移动相机。

+ ``Camera.rotate*()`` 三维旋转 ``Camera.rotate*()`` 一共有四个方法： ``rotateX(deg) rotateY(deg) rotateZ(deg) rotate(x, y, z)``。

+ ``Camera.translate(float x, float y, float z)`` 移动

+ ``Camera.setLocation(x, y, z)`` 设置虚拟相机的位置。在 Camera 中，相机的默认位置是 (0, 0, -8)（英寸）。8 x 72 = 576，所以它的默认位置是 (0, 0, -576)（像素）。

### 绘制顺序

##### super.onDraw() 前 or 后？

##### ``dispatchDraw()``：绘制子 View 的方法

##### 绘制过程简述

绘制过程中最典型的两个部分是上面讲到的主体和子 View，但它们并不是绘制过程的全部。除此之外，绘制过程还包含一些其他内容的绘制。具体来讲，一个完整的绘制过程会依次绘制以下几个内容：

- 背景
- 主体（onDraw()）
- 子 View（dispatchDraw()）
- 滑动边缘渐变和滑动条
- 前景

<center><p><img src="/images/canvas-draw-process.jpg"/></p></center>

##### onDrawForeground()

在 onDrawForeground() 中，会依次绘制滑动边缘渐变、滑动条和前景。

##### draw() 总调度方法

<center><p><img src="/images/canvas-draw.jpg"/></p></center>

关于绘制方法，有两点需要注意一下：

- 出于效率的考虑，ViewGroup 默认会绕过 ``draw()`` 方法，换而直接执行  ``dispatchDraw()``，以此来简化绘制流程。所以如果你自定义了某个 ViewGroup 的子类（比如 LinearLayout）并且需要在它的除 ``dispatchDraw()`` 以外的任何一个绘制方法内绘制内容，你可能会需要调用 ``View.setWillNotDraw(false)`` 这行代码来切换到完整的绘制流程（是「可能」而不是「必须」的原因是，有些 ViewGroup 是已经调用过  setWillNotDraw(false) 了的，例如 ScrollView）。

- 有的时候，一段绘制代码写在不同的绘制方法中效果是一样的，这时你可以选一个自己喜欢或者习惯的绘制方法来重写。但有一个例外：如果绘制代码既可以写在  ``onDraw()`` 里，也可以写在其他绘制方法里，那么优先写在 ``onDraw()`` ，因为 Android 有相关的优化，可以在不需要重绘的时候自动跳过 ``onDraw()`` 的重复执行，以提升开发效率。享受这种优化的只有 ``onDraw()`` 一个方法。


### 属性动画 Property Animation

 Android 里动画是有一些分类的：动画可以分为两类：Animation 和 Transition；其中 Animation 又可以再分为 View Animation 和 Property Animation 两类： View Animation 是纯粹基于 framework 的绘制转变，Property Animation，属性动画，这是在 Android 3.0 开始引入的新的动画形式。

 ##### ViewPropertyAnimator

 <center><p><img src="/images/view-animate.jpg"/></p></center>

 ##### ObjectAnimator

使用方式：

+ 如果是自定义控件，需要添加 setter / getter 方法；
+ 用 ObjectAnimator.ofXXX() 创建 ObjectAnimator 对象；
+ 用 start() 方法执行动画。

##### 通用方法

+ ``setDuration(int duration)`` 设置动画时长

+ ``setInterpolator(Interpolator interpolator)`` 设置 Interpolator,``AccelerateDecelerateInterpolator``,``LinearInterpolator``,``AccelerateInterpolator``,``DecelerateInterpolator``,``AnticipateInterpolator``,``OvershootInterpolator``,``AnticipateOvershootInterpolator``,``BounceInterpolator``,``CycleInterpolator``,``PathInterpolator``,``FastOutLinearInInterpolator``,``FastOutSlowInInterpolator``,``LinearOutSlowInInterpolator``

##### 设置监听器

设置监听器的方法， ViewPropertyAnimator 和 ObjectAnimator 略微不一样：  ViewPropertyAnimator 用的是 setListener() 和 setUpdateListener() 方法，可以设置一个监听器，要移除监听器时通过 set[Update]Listener(null) 填 null 值来移除；而  ObjectAnimator 则是用 addListener() 和 addUpdateListener() 来添加一个或多个监听器，移除监听器则是通过 remove[Update]Listener() 来指定移除对象。另外，由于 ObjectAnimator 支持使用 pause() 方法暂停，所以它还多了一个  addPauseListener() / removePauseListener() 的支持；而 ViewPropertyAnimator 则独有  withStartAction() 和 withEndAction() 方法，可以设置一次性的动画开始或结束的监听。

AnimatorListener 共有 4 个回调方法：

+ ``onAnimationStart(Animator animation)``

+ ``onAnimationEnd(Animator animation)``

+ ``onAnimationCancel(Animator animation)``

+ ``onAnimationRepeat(Animator animation)``

``AnimatorUpdateListener``它只有一个回调方法：``onAnimationUpdate(ValueAnimator animation)``

``ViewPropertyAnimator.withStartAction/EndAction()``，``withStartAction() / withEndAction()`` 是一次性的，在动画执行结束后就自动弃掉了，就算之后再重用 ``ViewPropertyAnimator`` 来做别的动画，用它们设置的回调也不会再被调用。而 ``set/addListener()`` 所设置的 ``AnimatorListener`` 是持续有效的，当动画重复执行时，回调总会被调用。``withEndAction()`` 设置的回调只有在动画正常结束时才会被调用，而在动画被取消时不会被执行。这点和 ``AnimatorListener.onAnimationEnd()`` 的行为是不一致的。

##### TypeEvaluator

+ ``ArgbEvaluator``

+ 自定义 Evaluator

借助于 TypeEvaluator，属性动画就可以通过 ofObject() 来对不限定类型的属性做动画了。方式很简单：

+ 为目标属性写一个自定义的 TypeEvaluator

+ 使用 ofObject() 来创建 Animator，并把自定义的 TypeEvaluator 作为参数填入

##### PropertyValuesHolder 同一个动画中改变多个属性

##### AnimatorSet 多个动画配合执行

##### PropertyValuesHolders.ofKeyframe() 把同一个属性拆分

### 硬件加速

+ [Hardware Acceleration | Android Developers](https://developer.android.google.cn/guide/topics/graphics/hardware-accel.html)

+ [Google I/O 2011: Accelerated Android Rendering](https://www.youtube.com/watch?v=v9S5EO7CLjo)

所谓硬件加速，指的是把某些计算工作交给专门的硬件来做，而不是和普通的计算工作一样交给 CPU 来处理。这样不仅减轻了 CPU 的压力，而且由于有了「专人」的处理，这份计算工作的速度也被加快了。这就是「硬件加速」。

而对于 Android 来说，硬件加速有它专属的意思：在 Android 里，硬件加速专指把 View 中绘制的计算工作交给 GPU 来处理。进一步地再明确一下，这个「绘制的计算工作」指的就是把绘制方法中的那些 Canvas.drawXXX() 变成实际的像素这件事。

在硬件加速关闭的时候，Canvas 绘制的工作方式是：把要绘制的内容写进一个  Bitmap，然后在之后的渲染过程中，这个 Bitmap 的像素内容被直接用于渲染到屏幕。这种绘制方式的主要计算工作在于把绘制操作转换为像素的过程（例如由一句  Canvas.drawCircle() 来获得一个具体的圆的像素信息），这个过程的计算是由 CPU 来完成的。而在硬件加速开启时，Canvas 的工作方式改变了：它只是把绘制的内容转换为 GPU 的操作保存了下来，然后就把它交给 GPU，最终由 GPU 来完成实际的显示工作。

硬件加速不只是好处，也有它的限制：受到 GPU 绘制方式的限制，Canvas 的有些方法在硬件加速开启式会失效或无法正常工作。比如，在硬件加速开启时， clipPath() 在 API 18 及以上的系统中才有效。具体的 API 限制和 API 版本的关系如下图：

<center><p><img src="/images/hardware-acceleration.jpg"></p></center>

##### View Layer

setLayerType() 这个方法，它的作用其实就是名字里的意思：设置 View Layer 的类型。所谓 View Layer，又称为离屏缓冲（Off-screen Buffer），它的作用是单独启用一块地方来绘制这个 View ，而不是使用软件绘制的 Bitmap 或者通过硬件加速的 GPU。这块「地方」可能是一块单独的 Bitmap，也可能是一块 OpenGL 的纹理（texture，OpenGL 的纹理可以简单理解为图像的意思），具体取决于硬件加速是否开启。采用什么来绘制 View 不是关键，关键在于当设置了 View Layer 的时候，它的绘制会被缓存下来，而且缓存的是最终的绘制结果，而不是像硬件加速那样只是把 GPU 的操作保存下来再交给 GPU 去计算。通过这样更进一步的缓存方式，View 的重绘效率进一步提高了：只要绘制的内容没有变，那么不论是 CPU 绘制还是 GPU 绘制，它们都不用重新计算，而只要只用之前缓存的绘制结果就可以了。

基于这样的原理，在进行移动、旋转等（无需调用 invalidate()）的属性动画的时候开启 Hardware Layer 将会极大地提升动画的效率，因为在动画过程中 View 本身并没有发生改变，只是它的位置或角度改变了，而这种改变是可以由 GPU 通过简单计算就完成的，并不需要重绘整个 View。所以在这种动画的过程中开启 Hardware Layer，可以让本来就依靠硬件加速而变流畅了的动画变得更加流畅。

不过一定要注意，只有你在对 translationX translationY rotation alpha 等无需调用  invalidate() 的属性做动画的时候，这种方法才适用，因为这种方法本身利用的就是当界面不发生时，缓存未更新所带来的时间的节省。所以简单地说——这种方式不适用于基于自定义属性绘制的动画。

另外，由于设置了 View Layer 后，View 在初次绘制时以及每次 invalidate() 后重绘时，需要进行两次的绘制工作（一次绘制到 Layer，一次从 Layer 绘制到显示屏），所以其实它的每次绘制的效率是被降低了的。所以一定要慎重使用 View Layer，在需要用到它的时候再去使用。

### reference

+ [awesome-view](https://github.com/xinghongfei/awesome-view)
+ [GcsSloop CustomView](https://github.com/GcsSloop/AndroidNote/tree/master/CustomView)
+ [Android自定义控件其实很简单](https://blog.csdn.net/aigestudio/column/info/androidcustomview)
+ [自定义View-绘制](https://hencoder.com/tag/hui-zhi/)