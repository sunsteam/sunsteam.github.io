---
layout:     post
title:      "View 绘制流程"
subtitle:   ""
date:       2016-02-19 12:00:00
author:     "Yomii"
catalog:    true
tags:
    - Android
---

## View 树的绘图流程

当 Activity 接收到焦点的时候，它会被请求绘制布局,该请求由 Android framework 处理.绘制是从根节点开始，对布局树进行 measure 和 draw。整个 View 树的绘图流程在ViewRoot.java类的performTraversals()函数展开，该函数所做 的工作可简单概况为是否需要重新计算视图大小(measure)、是否需要重新安置视图的位置(layout)、以及是否需要重绘(draw)，流程图如下：

![](https://ws4.sinaimg.cn/large/006tNc79gy1foo0k77nigj30ob072jrd.jpg)

## View 绘制流程函数调用链

![](https://ws4.sinaimg.cn/large/006tNc79gy1foo0k9j70nj30fx0emmxr.jpg)

## Measure过程

View中的measure(int,int)方法是final修饰不可复写的，其中先完成了一些准备工作和优化操作，比如sdk17的左右转换和判断当前是否需要重新测量。之后会调用onMeasure(int,int)方法，通过复写该方法完成具体测量工作，在这个方法中必须调用setMeasuredDimension(int,int)方法设置测量的结果，否则回到measure方法后会抛异常。

### 2个int参数MeasureSpecs 类的意义

该类封装了父控件对子控件测量的指导要求，getMode方法可以获取测量模式，有以下3种：

- `UNSPECIFIED` 父视图不对子视图有任何约束，它可以达到所期望的任意尺寸。比如 ListView、ScrollView，一般自定义 View 中用不到, 类似使用measure(0,0);

- `EXACTLY` 父视图为子视图指定一个确切的尺寸，而且无论子视图期望多大，它都必须在该指定大小的边界内，对应的属性为 match_parent 或具体值，比如 100dp，子控件可以通过MeasureSpec.getSize(measureSpec)直接得到父控件的尺寸。

- `AT_MOST` 父视图为子视图指定一个最大尺寸。子视图必须确保它自己所有子视图可以适应在该尺寸范围内，对应的属性为 wrap_content，这种模式下，父控件无法确定子 View 的尺寸，只能由子控件自己根据需求去计算自己的尺寸，这种模式就是我们自定义视图需要实现测量逻辑的情况。

### onMeasure的过程

不同控件的具体实现不同，但大致可以概括为：

1. 根据获取到的MeasureSpecs测量大小, 组合控件应逐个测量子控件大小，ViewGroup中有提供measureChildWithMargins方法，特殊控件如ListView和ScrollView没有直接的子View，因此可以用measureChild方法或measureChildren方法。

2. 根据所有子控件的大小要求，计算自身确切大小并setMeasuredDimension，有时候不能得出确切结果时可能需要根据新的大小重新计算子控件并修正大小。

3. 其他操作，比如再次要求子控件根据精确模式重新计算自身大小。

**下图表现了控件onMeasure中的执行流程，中间的Measure和onMeasure都属于它的子控件的调用。具体可参考FrameLayout之类简单的系统控件的源码**

![](https://ws2.sinaimg.cn/large/006tNc79gy1foo0kb5gh2j30ks0gaaa1.jpg)

### onLayout的过程

View 的 onLayout 方法为空实现，而 ViewGroup 的 onLayout 为 abstract 的，因此，如果自定义的 View 要继承 ViewGroup 时，必须实现 onLayout 函数。

在 layout 过程中，子视图会调用getMeasuredWidth()和getMeasuredHeight()方法获取到 measure 过程得到的 mMeasuredWidth 和 mMeasuredHeight，作为自己的 width 和 height。然后调用每一个子视图的layout(l, t, r, b)函数，来确定每个子视图在父视图中的位置。

#### LinearLayout 的 onLayout 源码分析

```java
@Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        if (mOrientation == VERTICAL) {
            layoutVertical(l, t, r, b);
        } else {
            layoutHorizontal(l, t, r, b);
        }
    }

    /**
     * 遍历所有的子 View，为其设置相对父视图的坐标
     */
    void layoutVertical(int left, int top, int right, int bottom) {
    for (int i = 0; i < count; i++) {
                final View child = getVirtualChildAt(i);
                if (child == null) {
                    childTop += measureNullChild(i);
                } else if (child.getVisibility() != GONE) {//不需要立即展示的 View 设置为 GONE 可加快绘制
                    final int childWidth = child.getMeasuredWidth();//measure 过程确定的 Width
                    final int childHeight = child.getMeasuredHeight();//measure 过程确定的 height

                    ...确定 childLeft、childTop 的值

                    setChildFrame(child, childLeft, childTop + getLocationOffset(child),
                            childWidth, childHeight);
                }
            }
    }

    private void setChildFrame(View child, int left, int top, int width, int height) {        
        child.layout(left, top, left + width, top + height);
    }    

    View.java
    public void layout(int l, int t, int r, int b) {
        ...
        setFrame(l, t, r, b)
    }

    /**
     * 为该子 View 设置相对其父视图上的坐标
     */
     protected boolean setFrame(int left, int top, int right, int bottom) {
         ...
     }
```

### 绘制流程相关概念及核心方法

先来看下与 draw 过程相关的函数：

- **View.draw(Canvas canvas)**： 由于 ViewGroup 并没有复写此方法，因此，所有的视图最终都是调用 View 的 draw 方法进行绘制的。在自定义的视图中，也不应该复写该方法，而是复写 onDraw(Canvas) 方法进行绘制，如果自定义的视图确实要复写该方法，那么请先调用 super.draw(canvas)完成系统的绘制，然后再进行自定义的绘制。

- **View.onDraw()**：View 的onDraw（Canvas）默认是空实现，自定义绘制过程需要复写的方法，绘制自身的内容。

- **dispatchDraw()** 发起对子视图的绘制。View 中默认是空实现，ViewGroup 复写了dispatchDraw()来对其子视图进行绘制。该方法我们不用去管，自定义的 ViewGroup 不应该对dispatchDraw()进行复写。

![](https://ws3.sinaimg.cn/large/006tNc79gy1foo0kcffddj30ph0fjq2w.jpg)

#### View.draw(Canvas) 源码分析

```java
/**
     * Manually render this view (and all of its children) to the given Canvas.
     * The view must have already done a full layout before this function is
     * called.  When implementing a view, implement
     * {@link #onDraw(android.graphics.Canvas)} instead of overriding this method.
     * If you do need to override this method, call the superclass version.
     *
     * @param canvas The Canvas to which the View is rendered.  
     *
     * 根据给定的 Canvas 自动渲染 View（包括其所有子 View）。在调用该方法之前必须要完成 layout。当你自定义 view 的时候，
     * 应该去是实现 onDraw(Canvas) 方法，而不是 draw(canvas) 方法。如果你确实需要复写该方法，请记得先调用父类的方法。
     */
    public void draw(Canvas canvas) {

        / * Draw traversal performs several drawing steps which must be executed
         * in the appropriate order:
         *
         *      1\. Draw the background if need
         *      2\. If necessary, save the canvas' layers to prepare for fading
         *      3\. Draw view's content
         *      4\. Draw children (dispatchDraw)
         *      5\. If necessary, draw the fading edges and restore layers
         *      6\. Draw decorations (scrollbars for instance)
         */

     // Step 1, draw the background, if needed
        if (!dirtyOpaque) {
            drawBackground(canvas);
        }

         // skip step 2 & 5 if possible (common case)
        final int viewFlags = mViewFlags;
        if (!verticalEdges && !horizontalEdges) {
            // Step 3, draw the content
            if (!dirtyOpaque) onDraw(canvas);

            // Step 4, draw the children
            dispatchDraw(canvas);

            // Step 6, draw decorations (scrollbars)
            onDrawScrollBars(canvas);

            if (mOverlay != null && !mOverlay.isEmpty()) {
                mOverlay.getOverlayView().dispatchDraw(canvas);
            }

            // we're done...
            return;
        }

        // Step 2, save the canvas' layers
        ...

        // Step 3, draw the content
        if (!dirtyOpaque)
            onDraw(canvas);

        // Step 4, draw the children
        dispatchDraw(canvas);

        // Step 5, draw the fade effect and restore layers

        // Step 6, draw decorations (scrollbars)
        onDrawScrollBars(canvas);
    }
```

```java
dispatchDraw(Canvas canvas){

...

 if ((flags & FLAG_RUN_ANIMATION) != 0 && canAnimate()) {//处理 ChildView 的动画
     final boolean buildCache = !isHardwareAccelerated();
            for (int i = 0; i < childrenCount; i++) {
                final View child = children[i];
                if ((child.mViewFlags & VISIBILITY_MASK) == VISIBLE) {//只绘制 Visible 状态的布局，因此可以通过延时加载来提高效率
                    final LayoutParams params = child.getLayoutParams();
                    attachLayoutAnimationParameters(child, params, i, childrenCount);// 添加布局变化的动画
                    bindLayoutAnimation(child);//为 Child 绑定动画
                    if (cache) {
                        child.setDrawingCacheEnabled(true);
                        if (buildCache) {
                            child.buildDrawingCache(true);
                        }
                    }
                }
            }

     final LayoutAnimationController controller = mLayoutAnimationController;
            if (controller.willOverlap()) {
                mGroupFlags |= FLAG_OPTIMIZE_INVALIDATE;
            }

    controller.start();// 启动 View 的动画
}

 // 绘制 ChildView
 for (int i = 0; i < childrenCount; i++) {
            int childIndex = customOrder ? getChildDrawingOrder(childrenCount, i) : i;
            final View child = (preorderedList == null)
                    ? children[childIndex] : preorderedList.get(childIndex);
            if ((child.mViewFlags & VISIBILITY_MASK) == VISIBLE || child.getAnimation() != null) {
                more |= drawChild(canvas, child, drawingTime);
            }
        }

...

}
```

```java
protected boolean drawChild(Canvas canvas, View child, long drawingTime) {
        return child.draw(canvas, this, drawingTime);
}
```

```java
/**
     * This method is called by ViewGroup.drawChild() to have each child view draw itself.
     * This draw() method is an implementation detail and is not intended to be overridden or
     * to be called from anywhere else other than ViewGroup.drawChild().
     */
    boolean draw(Canvas canvas, ViewGroup parent, long drawingTime) {
        ...
    }
```

- `drawChild(canvas, this, drawingTime)` 直接调用了 View 的child.draw(canvas, this,drawingTime)方法，文档中也说明了，除了被ViewGroup.drawChild()方法外，你不应该在其它任何地方去复写或调用该方法，它属于 ViewGroup。而View.draw(Canvas)方法是我们自定义控件中可以复写的方法，具体可以参考上述对view.draw(Canvas)的说明。从参数中可以看到，child.draw(canvas, this, drawingTime) 肯定是处理了和父视图相关的逻辑，但 View 的最终绘制，还是 View.draw(Canvas)方法。

- `invalidate()` 请求重绘 View 树，即 draw 过程，假如视图发生大小没有变化就不会调用layout()过程，并且只绘制那些调用了invalidate()方法的 View。

- `requestLayout()` 当布局变化的时候，比如方向变化，尺寸的变化，会调用该方法，在自定义的视图中，如果某些情况下希望重新测量尺寸大小，应该手动去调用该方法，它会触发measure()和layout()过程，但不会进行 draw。特殊情况下，比如requestLayout时，视图正好需要更新，比如动画效果等，onDraw方法也会调用。
