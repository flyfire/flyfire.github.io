+++
title = "Summary of Custom Views Part Ii"
date = "2019-02-12"
slug = "2019/02/12/summary-of-custom-views-part-ii"
Categories = ["dev", "android"]
+++

# 自定义View总结 - 布局

### 布局基础

布局过程，就是程序在运行时利用布局文件中的代码来计算出实际尺寸和位置的过程。有两个阶段，测量阶段和布局阶段，分别对应``measure``和``layout``。

对于一个``View``而言，默认的``onMeasure``实现是：

<!-- more -->

```java
	/**
     * <p>
     * Measure the view and its content to determine the measured width and the
     * measured height. This method is invoked by {@link #measure(int, int)} and
     * should be overridden by subclasses to provide accurate and efficient
     * measurement of their contents.
     * </p>
     *
     * <p>
     * <strong>CONTRACT:</strong> When overriding this method, you
     * <em>must</em> call {@link #setMeasuredDimension(int, int)} to store the
     * measured width and height of this view. Failure to do so will trigger an
     * <code>IllegalStateException</code>, thrown by
     * {@link #measure(int, int)}. Calling the superclass'
     * {@link #onMeasure(int, int)} is a valid use.
     * </p>
     *
     * <p>
     * The base class implementation of measure defaults to the background size,
     * unless a larger size is allowed by the MeasureSpec. Subclasses should
     * override {@link #onMeasure(int, int)} to provide better measurements of
     * their content.
     * </p>
     *
     * <p>
     * If this method is overridden, it is the subclass's responsibility to make
     * sure the measured height and width are at least the view's minimum height
     * and width ({@link #getSuggestedMinimumHeight()} and
     * {@link #getSuggestedMinimumWidth()}).
     * </p>
     *
     * @param widthMeasureSpec horizontal space requirements as imposed by the parent.
     *                         The requirements are encoded with
     *                         {@link android.view.View.MeasureSpec}.
     * @param heightMeasureSpec vertical space requirements as imposed by the parent.
     *                         The requirements are encoded with
     *                         {@link android.view.View.MeasureSpec}.
     *
     * @see #getMeasuredWidth()
     * @see #getMeasuredHeight()
     * @see #setMeasuredDimension(int, int)
     * @see #getSuggestedMinimumHeight()
     * @see #getSuggestedMinimumWidth()
     * @see android.view.View.MeasureSpec#getMode(int)
     * @see android.view.View.MeasureSpec#getSize(int)
     */
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
                getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
    }

	/**
     * Utility to return a default size. Uses the supplied size if the
     * MeasureSpec imposed no constraints. Will get larger if allowed
     * by the MeasureSpec.
     *
     * @param size Default size for this view
     * @param measureSpec Constraints imposed by the parent
     * @return The size this view should be.
     */
    public static int getDefaultSize(int size, int measureSpec) {
        int result = size;
        int specMode = MeasureSpec.getMode(measureSpec);
        int specSize = MeasureSpec.getSize(measureSpec);

        switch (specMode) {
        case MeasureSpec.UNSPECIFIED:
            result = size;
            break;
        case MeasureSpec.AT_MOST:
        case MeasureSpec.EXACTLY:
            result = specSize;
            break;
        }
        return result;
    }
	
	/**
     * Returns the suggested minimum width that the view should use. This
     * returns the maximum of the view's minimum width
     * and the background's minimum width
     *  ({@link android.graphics.drawable.Drawable#getMinimumWidth()}).
     * <p>
     * When being used in {@link #onMeasure(int, int)}, the caller should still
     * ensure the returned width is within the requirements of the parent.
     *
     * @return The suggested minimum width of the view.
     */
    protected int getSuggestedMinimumWidth() {
        return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinimumWidth());
    }
```

可以看到在``getDefault``方法中对``AT_MOST``(``wrap_content``)和``EXACTLY``(``match_parent``)的处理都是用父View的size来作为了子View的size，这种处理有的时候是不合适的，需要我们额外做些处理，下面是一个在写自定义View的时候的一个utils方法：

```java
	public static int measure(int measureSpec, int defaultSize){
        int result = defaultSize;
        int specMode = View.MeasureSpec.getMode(measureSpec);
        int specSize = View.MeasureSpec.getSize(measureSpec);
        if (specMode == View.MeasureSpec.EXACTLY){
            result = specSize;
        } else if (specMode == View.MeasureSpec.AT_MOST){
            result = Math.min(specSize, defaultSize);
        }
        return result;
    }
```

测量阶段，``measure()``方法被父View调用，在``measure()``中做一些准备和优化工作后，调用``onMeasure()``来进行实际的自我测量。``onMeasure()``中做的事，``View``和``ViewGroup``不太一样：

+ ``View``在``onMeasure()``中根据父View传过来的MeasureSpec约束计算自己的大小并调用``setMeasuredDimension``保存下来。
+ ``ViewGroup``在``onMeasure``中调用``measureChildren``测量子View，并根据子View计算出的期望大小来计算出它们的实际尺寸和位置然后保存。同时根据子View的尺寸和位置来计算出自己的尺寸并保存。

在``ViewGroup``测量子View的时候，也就是调用``childView.measure()``的时候需要将自己的约束MeasureSpec传递给子View，这个MeasureSpec如何计算，下面会说。回到最顶层的父View，也即``DecorView``，它的MeasureSpec是``LayoutParams.MATCH_PARENT, LayoutParams.MATCH_PARENT``。

布局过程，``layout()``方法被父View调用，在``layout()``方法中它会保存父View传进来的自己的位置和尺寸，并且调用``onLayout``来进行实际的内部布局。``onLayout``中做的事，``View``和``ViewGroup``也不一样：

+ ``View``由于没有子View，它的``onLayout``什么也不做
+ ``ViewGroup``在``onLayout``中会调用自己所有子View的``layout``方法，把它们的尺寸和位置传给它们，让它们完成自我的内部布局。

### 全新定义 View 的尺寸

子View在计算的时候需要保证计算结果满足父View MeasureSpec对自己的尺寸限制。``ViewGroup``提供了几个工具方法供我们调用：

```
    /**
     * Ask all of the children of this view to measure themselves, taking into
     * account both the MeasureSpec requirements for this view and its padding.
     * We skip children that are in the GONE state The heavy lifting is done in
     * getChildMeasureSpec.
     *
     * @param widthMeasureSpec The width requirements for this view
     * @param heightMeasureSpec The height requirements for this view
     */
    protected void measureChildren(int widthMeasureSpec, int heightMeasureSpec) {
        final int size = mChildrenCount;
        final View[] children = mChildren;
        for (int i = 0; i < size; ++i) {
            final View child = children[i];
            if ((child.mViewFlags & VISIBILITY_MASK) != GONE) {
                measureChild(child, widthMeasureSpec, heightMeasureSpec);
            }
        }
    }

    /**
     * Ask one of the children of this view to measure itself, taking into
     * account both the MeasureSpec requirements for this view and its padding.
     * The heavy lifting is done in getChildMeasureSpec.
     *
     * @param child The child to measure
     * @param parentWidthMeasureSpec The width requirements for this view
     * @param parentHeightMeasureSpec The height requirements for this view
     */
    protected void measureChild(View child, int parentWidthMeasureSpec,
            int parentHeightMeasureSpec) {
        final LayoutParams lp = child.getLayoutParams();

        final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
                mPaddingLeft + mPaddingRight, lp.width);
        final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
                mPaddingTop + mPaddingBottom, lp.height);

        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    }

    /**
     * Does the hard part of measureChildren: figuring out the MeasureSpec to
     * pass to a particular child. This method figures out the right MeasureSpec
     * for one dimension (height or width) of one child view.
     *
     * The goal is to combine information from our MeasureSpec with the
     * LayoutParams of the child to get the best possible results. For example,
     * if the this view knows its size (because its MeasureSpec has a mode of
     * EXACTLY), and the child has indicated in its LayoutParams that it wants
     * to be the same size as the parent, the parent should ask the child to
     * layout given an exact size.
     *
     * @param spec The requirements for this view
     * @param padding The padding of this view for the current dimension and
     *        margins, if applicable
     * @param childDimension How big the child wants to be in the current
     *        dimension
     * @return a MeasureSpec integer for the child
     */
    public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
        int specMode = MeasureSpec.getMode(spec);
        int specSize = MeasureSpec.getSize(spec);

        int size = Math.max(0, specSize - padding);

        int resultSize = 0;
        int resultMode = 0;

        switch (specMode) {
        // Parent has imposed an exact size on us
        case MeasureSpec.EXACTLY:
            if (childDimension >= 0) {
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size. So be it.
                resultSize = size;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size. It can't be
                // bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;

        // Parent has imposed a maximum size on us
        case MeasureSpec.AT_MOST:
            if (childDimension >= 0) {
                // Child wants a specific size... so be it
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size, but our size is not fixed.
                // Constrain child to not be bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size. It can't be
                // bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;

        // Parent asked to see how big we want to be
        case MeasureSpec.UNSPECIFIED:
            if (childDimension >= 0) {
                // Child wants a specific size... let him have it
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size... find out how big it should
                // be
                resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                resultMode = MeasureSpec.UNSPECIFIED;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size.... find out how
                // big it should be
                resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                resultMode = MeasureSpec.UNSPECIFIED;
            }
            break;
        }
        //noinspection ResourceType
        return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
    }
```

可以看到子View`MeasureSpec`是由3个因素共同决定的，父View的`MeasureSpec`，`padding`，`LayoutParams`中的值，其中父View的`MeasureSpec`是从`DecorView`一层层约束下来得到的，不难理解，那`LayoutParams`是从哪里来的呢？	其实`LayoutParams`是在`LayoutInflater`解析xml布局的时候，父布局会根据xml中的`layout_width`和`layout_height`来给子View`generateLayoutParams`保存在子View中，所以，可以认为子View的`LayoutParams`实际上保存的是布局中的开发者对View的要求。

着重看下``getChildMeasureSpec``这个方法，这个方法对父View尺寸有无限制的情况下子View的大小应该如何进行了处理，注释的比较清楚，就不解释了。

### 定制 Layout(ViewGroup) 的内部布局

通过重写``onMeasure()``来计算内部布局，重写``onLayout``来摆放子View。

重写``onMeasure()``的三个步骤：

+ 调用每个子View的``measure``方法来计算子View的尺寸
+ 计算子View的位置并保存子View的位置和尺寸
+ 计算自己的尺寸并用``setMeasuredDimension()``保存

计算子View尺寸的关键在于``measure()``方法的两个MeasureSpec参数的计算。子View的``MeasureSpec``的计算方式：

+ 结合开发者的要求（xml中的layout_打头的属性）和自己的可用空间（自己的尺寸上限-已用尺寸）
+ 尺寸上限根据自己的``MeasureSpec``中的mode而定，``EXACTLY/AT_MOST``尺寸上限为``MeasureSpec``中的size，``UNSPECIFIED``尺寸无上限。

重写``onLayout``的方式，在``onLayout``里调用每个子View的``layout()``。

### reference

+ [自定义 View - 布局](https://hencoder.com/tag/bu-ju/)

