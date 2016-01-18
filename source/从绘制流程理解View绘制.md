title: 从绘制流程理解View绘制
date: 2016-01-07 11:59:09
tags: android
categories: android
---
本文是本人对View绘制的学习的笔记文章，借鉴了一些前辈的经验
<!--more  -->

>本文参考了众多前辈的文章和书籍
>[任玉刚的相关文章和书籍](http://blog.csdn.net/singwhatiwanna)
>[郭霖博客](http://blog.csdn.net/guolin_blog)

# Measure 过程
## View的Measure过程
### View的measure方法
view的measure方法为final，意味着，子view不可以复写此方法。此外，onMeasure方法会在measure  
方法中调用。并且子View **必须复写onMeasure方法**（源码注解提到，至于为何，下面分析下去）。  
widthMeasureSpec和heightMeasureSpec是父View传递过来的宽高参数（32位，高2位表示模式，低30位表示大小），
 源码可以理解为下面代码（仅截取部分代码，方便理解，下同）

```java
public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
  ....
  onMeasure(widthMeasureSpec, heightMeasureSpec);
  ....
}
```

### View的onMeasure方法
源码注释中提到
* setMeasuredDimension(int, int)，这是必须调用的，这是决定view宽高的方法
* 子View都应该复写此方法。  
* 子View复写此方法时，应该确保宽高最小是View的最小宽高。（getSuggestedMinimumWidth，getSuggestedMinimumHeight）
* The base class implementation of measure defaults to the background size,
  unless a larger size is allowed by the MeasureSpec  
   翻译过来就是，除非父View给的MeasureSpec提供了一个更大的值，View的默认测量宽高都是背景宽高值

```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
       setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
               getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
   }
```

下面方法，意思就是，选择最小宽度和背景（mBackground）最小宽度之间的最大值。而mBackground是的最小宽度  
则是图片的原始宽度，而不同的drawable，原始宽度是不同的。其中，shapwDrawable没有原始宽度，BitmapDrawable则有原始宽度

```java
protected int getSuggestedMinimumWidth() {
        return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinimumWidth());
    }
```

下面方法，意思是，当模式是UNSPECIFIED时，就是入参size（View默认传入值就是，view的最小宽高值），
当模式是AT_MOST或者EXACTLY时，返回值都是父View给的大小specSize。

```java
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
```

## View的measure过程总结
1. 子View都应该复写onMeasure方法，并且需要注意，宽高应该大于等于，View的建议最小宽高。
2. 计算完宽高之后，必须调用setMeasuredDimension(int, int)，设定宽高。***需要特别处理wrap-content的情况***  
默认情况下，wrap-content和Match-parent的效果是一样的，都是取的父View给的measureSpec的specSize宽高值。  
3. 处理wrap-content，可以自己自定义一个属性来实现。


## ViewGroup的Measure过程
ViewGroup的measure方法，就是View的measure方法（这是final的方法），ViewGroup 的onMeasure方法，也没有复写。  
意味着，自定义View继承ViewGroup的话，必须复写了。虽说没有复写onMeasure方法，但是ViewGroup也为我们做了一些事情的  
提供了一些方法，方便我们实现自定义ViewGroup。


widthUsed是指父View已经用了的宽度。

```java
protected void measureChildWithMargins(View child,
            int parentWidthMeasureSpec, int widthUsed,
            int parentHeightMeasureSpec, int heightUsed) {
        final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();

        final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
                mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                        + widthUsed, lp.width);
        final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
                mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin
                        + heightUsed, lp.height);

        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    }
```

 接下来，再来看看getChildMeasureSpec方法。这个方法，返回值是用来传递给child.measure 方法的，  
 这个方法，就是用来制造子View的MeasureSpec宽高值的
 结合上面的方法，
 1. spec是父View的宽高要求parentWidthMeasureSpec或者parentHeightMeasureSpec  
 2. padding，在上述方法中，是把父View的padding+ widthUsed + 子View的margin。
 3. childDimension，是子View的MarginLayoutParams的width或者height


```java
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
        return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
    }
```

上面方法中， specMode 是父View的 MeasureSpec 中的高二位，代表模式，specSize则是低30位，代表大小  
size 代表父View的大小减去这个padding，剩下可用的宽高值。resultSize和resultMode，是准备返回给子View的MeasureSpec。
LayoutParams.MATCH_PARENT == -1;LayoutParams.WRAP_CONTENT == -2;  
上面一共有3因素3水平的一个逻辑  

* 如果父View是EXACTLY的情况下.（父View有一个精确的宽高给子View）
1. 如果childDimension >0。  
其实也就是代表，我们写死了dp值，这个时候就，resultSize = childDimension；  
resultMode = EXACTLY；这个时候，**有问题1**，如果父View写死了100dp，但是我子View写死了200dp，这个怎么办呢。

2. 如果childDimension = LayoutParams.MATCH_PARENT。  
resultSize = size；resultMode = EXACTLY；从这也可以看出，  
如果我们是写死dp值或者Match-parent，view的resultMode = EXACTLY。

3. 如果childDimension = LayoutParams.WRAP_CONTENT。  
resultSize = size；resultMode = AT_MOST;  


* 如果父View是 AT_MOST 的情况下.（父View有一个最大值的宽高给子View）
1. 如果childDimension >0。  
其实也就是代表，我们写死了dp值，这个时候就， resultSize = childDimension；  
resultMode = EXACTLY；这个时候，同样有问题1，同上。

2. 如果childDimension = LayoutParams.MATCH_PARENT .  
resultSize = size；resultMode = AT_MOST;这个可以这么理解，父View是给了一个最大值你，  
你想跟父View一样大，但是父View的大小是不确定的，所以限制子View不能比我大就好了。  
你的 MeasureSpec SpecSize就是父View的SpecSize.  
SpecMode 也和父View一样，也就是SpecMode;

3. 如果childDimension = LayoutParams.WRAP_CONTENT。   
resultSize = size；resultMode = AT_MOST;这个可以这么理解，父View是给了一个最大值你，  
你想自己决定大小，这可以，但是你不能比父View要大。  
你的 MeasureSpec SpecSize就是父View的SpecSize.  
SpecMode 也和父View一样，也就是SpecMode;


* 如果父View是 UNSPECIFIED 的情况下.（这种情况，一般用于ListView和RecycleView，父View 问 子View想要多大，源码注解原话Parent asked to see   how big we want to be）
1. 如果childDimension >0。  
其实也就是代表，我们写死了dp值，这个时候就， resultSize = childDimension；  
resultMode = EXACTLY；这个时候，同样有问题1，同上。

2. 如果childDimension = LayoutParams.MATCH_PARENT .  

3. 如果childDimension = LayoutParams.WRAP_CONTENT。   

以上2种情况，未明白。涉及到View的一个标示值，View.sUseZeroUnspecifiedMeasureSpec。关于这个变量，  
源码有以下注释，具体原理和用法，**待查明**。

```java
// In M and newer, our widgets can pass a "hint" value in the size
            // for UNSPECIFIED MeasureSpecs. This lets child views of scrolling containers
            // know what the expected parent size is going to be, so e.g. list items can size
            // themselves at 1/3 the size of their container. It breaks older apps though,
            // specifically apps that use some popular open source libraries.
            sUseZeroUnspecifiedMeasureSpec = targetSdkVersion < M;
```

## LinearLayout的measure 过程
onMeasure方法中，代码较多。暂时简单分析下measureVertical方法，我们暂时不考了weight属性  
用了mTotalLength变量来保存，暂时已经使用了的高度，遍历所有子View，遇到gone的不考虑。  
由此可见，INVISIBLE的view，是占据了位置的。而且还会加上分割线的高度（如果这2个子View之间有分割线）  


```java
if (hasDividerBeforeChildAt(i)) {
                mTotalLength += mDividerHeight;
            }
```

每算完一个子View的高度，就把这个view的高度添加到mTotalLength中。算完子View高度，最后再调用

```java
setMeasuredDimension(resolveSizeAndState(maxWidth, widthMeasureSpec, childState),
        heightSizeAndState);

```

这样来设定自身的宽高。

# MeasureSpec 本身的含义
本身代表一个32位的int值，高二位代表SpecMode，低30位代表SpecSize，SpecMode有三个类
* UNSPECIFIED, 父容器不对View有任何限制。
* EXACTLY 精确值，已经对View检测出精确的大小。这个时候View的最终大小就是SpecSize，这个一般对应LayoutParams中的Match-parent，以及具体写死的dp值。
* AT_MOST 最多多少，给出一个值SpecSize，表示view最大可以这么多。一般对应LayoutParams中的wrap-content
* MeasureSpec是由自身的LayoutParams以及父View的MeasureSpecs，来共同决定的。


# view 的 Layout 过程

## view的layout方法
 从下面代码可以看到，主要的工作时，调用 setFrame 方法.setFrame 方法内部会判断左上右下  
 坐标，是否和old值一致，如果不一致，返回值 changed = true；如果changed （true）改变了  
 就会回调 onLayout 方法。view里面的 onLayout 方法是空实现的。对于view而言，这不要紧，毕竟在layout里面  
 已经为左上右下坐标赋值了。

```java
public void layout(int l, int t, int r, int b) {
        if ((mPrivateFlags3 & PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT) != 0) {
            onMeasure(mOldWidthMeasureSpec, mOldHeightMeasureSpec);
            mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
        }

        int oldL = mLeft;
        int oldT = mTop;
        int oldB = mBottom;
        int oldR = mRight;

        boolean changed = isLayoutModeOptical(mParent) ?
                setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);

        if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
            onLayout(changed, l, t, r, b);
            mPrivateFlags &= ~PFLAG_LAYOUT_REQUIRED;

            ListenerInfo li = mListenerInfo;
            if (li != null && li.mOnLayoutChangeListeners != null) {
                ArrayList<OnLayoutChangeListener> listenersCopy =
                        (ArrayList<OnLayoutChangeListener>)li.mOnLayoutChangeListeners.clone();
                int numListeners = listenersCopy.size();
                for (int i = 0; i < numListeners; ++i) {
                    listenersCopy.get(i).onLayoutChange(this, l, t, r, b, oldL, oldT, oldR, oldB);
                }
            }
        }

        mPrivateFlags &= ~PFLAG_FORCE_LAYOUT;
        mPrivateFlags3 |= PFLAG3_IS_LAID_OUT;
    }

```


```java
protected boolean setFrame(int left, int top, int right, int bottom) {
        boolean changed = false;

        if (mLeft != left || mRight != right || mTop != top || mBottom != bottom) {
            changed = true;

            // Remember our drawn bit
            int drawn = mPrivateFlags & PFLAG_DRAWN;

            int oldWidth = mRight - mLeft;
            int oldHeight = mBottom - mTop;
            int newWidth = right - left;
            int newHeight = bottom - top;
            boolean sizeChanged = (newWidth != oldWidth) || (newHeight != oldHeight);

            // Invalidate our old position
            invalidate(sizeChanged);

            mLeft = left;
            mTop = top;
            mRight = right;
            mBottom = bottom;

        }
        return changed;
    }

```


```java
protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
   }
```

## ViewGroup 的 layout

layout方法，是final类型，并且基本相当于直接调用view的layout方法，而onLayout方法，还是一个空方法。  
这意味着，自定义ViewGroup 要想layout他的子View，就必须重写 onLayout 方法了。

先挑个简单的看看。
### FrameLayout 的layout
先拿到自己的左上右下坐标parentLeft、parentTop、parentRight、parentBottom。然后遍历所有子View，  
再拿到子View的LayoutParams，根据gravity 和 margin 属性来进行 子View的左上右下坐标确定。
**解疑-问题1** 目前来看，如果子View的MeasureWidth 比父View的要大的话，在父View的layout过程中，  
给子View的layout方法传递的参数，是有可能为负的。后面，我们再看看draw过程中，是怎么处理负数的。

```java
protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
       layoutChildren(left, top, right, bottom, false /* no force left gravity */);
   }

   void layoutChildren(int left, int top, int right, int bottom,
                                 boolean forceLeftGravity) {
       final int count = getChildCount();

       final int parentLeft = getPaddingLeftWithForeground();
       final int parentRight = right - left - getPaddingRightWithForeground();

       final int parentTop = getPaddingTopWithForeground();
       final int parentBottom = bottom - top - getPaddingBottomWithForeground();

       for (int i = 0; i < count; i++) {
           final View child = getChildAt(i);
           if (child.getVisibility() != GONE) {
               final LayoutParams lp = (LayoutParams) child.getLayoutParams();

               final int width = child.getMeasuredWidth();
               final int height = child.getMeasuredHeight();

               int childLeft;
               int childTop;

               int gravity = lp.gravity;
               if (gravity == -1) {
                   gravity = DEFAULT_CHILD_GRAVITY;
               }

               final int layoutDirection = getLayoutDirection();
               final int absoluteGravity = Gravity.getAbsoluteGravity(gravity, layoutDirection);
               final int verticalGravity = gravity & Gravity.VERTICAL_GRAVITY_MASK;

               switch (absoluteGravity & Gravity.HORIZONTAL_GRAVITY_MASK) {
                   case Gravity.CENTER_HORIZONTAL:
                       childLeft = parentLeft + (parentRight - parentLeft - width) / 2 +
                       lp.leftMargin - lp.rightMargin;
                       break;
                   case Gravity.RIGHT:
                       if (!forceLeftGravity) {
                           childLeft = parentRight - width - lp.rightMargin;
                           break;
                       }
                   case Gravity.LEFT:
                   default:
                       childLeft = parentLeft + lp.leftMargin;
               }

               switch (verticalGravity) {
                   case Gravity.TOP:
                       childTop = parentTop + lp.topMargin;
                       break;
                   case Gravity.CENTER_VERTICAL:
                       childTop = parentTop + (parentBottom - parentTop - height) / 2 +
                       lp.topMargin - lp.bottomMargin;
                       break;
                   case Gravity.BOTTOM:
                       childTop = parentBottom - height - lp.bottomMargin;
                       break;
                   default:
                       childTop = parentTop + lp.topMargin;
               }

               child.layout(childLeft, childTop, childLeft + width, childTop + height);
           }
       }
   }
```

# View 的 draw 过程

## 先看view的draw 方法
从代码注解中，可以得知，总共有6step，其中2&5可以跳过。
* 第一步主要绘制背景
* 第三步 调用 ondraw 方法。此方法在，view中是空的。源码注解，也是建议我们在这实现，我们的绘制。
* 第四步 调用dispatchDraw(canvas) 方法。此方法，在view中是空的。
* 第六步 绘制scrollbars等。  


```java
public void draw(Canvas canvas) {
        final int privateFlags = mPrivateFlags;
        final boolean dirtyOpaque = (privateFlags & PFLAG_DIRTY_MASK) == PFLAG_DIRTY_OPAQUE &&
                (mAttachInfo == null || !mAttachInfo.mIgnoreDirtyState);
        mPrivateFlags = (privateFlags & ~PFLAG_DIRTY_MASK) | PFLAG_DRAWN;

        /*
         * Draw traversal performs several drawing steps which must be executed
         * in the appropriate order:
         *
         *      1. Draw the background
         *      2. If necessary, save the canvas' layers to prepare for fading
         *      3. Draw view's content
         *      4. Draw children
         *      5. If necessary, draw the fading edges and restore layers
         *      6. Draw decorations (scrollbars for instance)
         */

        // Step 1, draw the background, if needed
        int saveCount;

        if (!dirtyOpaque) {
            drawBackground(canvas);
        }

        // skip step 2 & 5 if possible (common case)
        final int viewFlags = mViewFlags;
        boolean horizontalEdges = (viewFlags & FADING_EDGE_HORIZONTAL) != 0;
        boolean verticalEdges = (viewFlags & FADING_EDGE_VERTICAL) != 0;
        if (!verticalEdges && !horizontalEdges) {
            // Step 3, draw the content
            if (!dirtyOpaque) onDraw(canvas);

            // Step 4, draw the children
            dispatchDraw(canvas);

            // Overlay is part of the content and draws beneath Foreground
            if (mOverlay != null && !mOverlay.isEmpty()) {
                mOverlay.getOverlayView().dispatchDraw(canvas);
            }

            // Step 6, draw decorations (foreground, scrollbars)
            onDrawForeground(canvas);

            // we're done...
            return;
        }

        /*
         * Here we do the full fledged routine...
         * (this is an uncommon case where speed matters less,
         * this is why we repeat some of the tests that have been
         * done above)
         */

        boolean drawTop = false;
        boolean drawBottom = false;
        boolean drawLeft = false;
        boolean drawRight = false;

        float topFadeStrength = 0.0f;
        float bottomFadeStrength = 0.0f;
        float leftFadeStrength = 0.0f;
        float rightFadeStrength = 0.0f;

        // Step 2, save the canvas' layers
        int paddingLeft = mPaddingLeft;

        final boolean offsetRequired = isPaddingOffsetRequired();
        if (offsetRequired) {
            paddingLeft += getLeftPaddingOffset();
        }

        int left = mScrollX + paddingLeft;
        int right = left + mRight - mLeft - mPaddingRight - paddingLeft;
        int top = mScrollY + getFadeTop(offsetRequired);
        int bottom = top + getFadeHeight(offsetRequired);

        if (offsetRequired) {
            right += getRightPaddingOffset();
            bottom += getBottomPaddingOffset();
        }

        final ScrollabilityCache scrollabilityCache = mScrollCache;
        final float fadeHeight = scrollabilityCache.fadingEdgeLength;
        int length = (int) fadeHeight;

        // clip the fade length if top and bottom fades overlap
        // overlapping fades produce odd-looking artifacts
        if (verticalEdges && (top + length > bottom - length)) {
            length = (bottom - top) / 2;
        }

        // also clip horizontal fades if necessary
        if (horizontalEdges && (left + length > right - length)) {
            length = (right - left) / 2;
        }

        if (verticalEdges) {
            topFadeStrength = Math.max(0.0f, Math.min(1.0f, getTopFadingEdgeStrength()));
            drawTop = topFadeStrength * fadeHeight > 1.0f;
            bottomFadeStrength = Math.max(0.0f, Math.min(1.0f, getBottomFadingEdgeStrength()));
            drawBottom = bottomFadeStrength * fadeHeight > 1.0f;
        }

        if (horizontalEdges) {
            leftFadeStrength = Math.max(0.0f, Math.min(1.0f, getLeftFadingEdgeStrength()));
            drawLeft = leftFadeStrength * fadeHeight > 1.0f;
            rightFadeStrength = Math.max(0.0f, Math.min(1.0f, getRightFadingEdgeStrength()));
            drawRight = rightFadeStrength * fadeHeight > 1.0f;
        }

        saveCount = canvas.getSaveCount();

        int solidColor = getSolidColor();
        if (solidColor == 0) {
            final int flags = Canvas.HAS_ALPHA_LAYER_SAVE_FLAG;

            if (drawTop) {
                canvas.saveLayer(left, top, right, top + length, null, flags);
            }

            if (drawBottom) {
                canvas.saveLayer(left, bottom - length, right, bottom, null, flags);
            }

            if (drawLeft) {
                canvas.saveLayer(left, top, left + length, bottom, null, flags);
            }

            if (drawRight) {
                canvas.saveLayer(right - length, top, right, bottom, null, flags);
            }
        } else {
            scrollabilityCache.setFadeColor(solidColor);
        }

        // Step 3, draw the content
        if (!dirtyOpaque) onDraw(canvas);

        // Step 4, draw the children
        dispatchDraw(canvas);

        // Step 5, draw the fade effect and restore layers
        final Paint p = scrollabilityCache.paint;
        final Matrix matrix = scrollabilityCache.matrix;
        final Shader fade = scrollabilityCache.shader;

        if (drawTop) {
            matrix.setScale(1, fadeHeight * topFadeStrength);
            matrix.postTranslate(left, top);
            fade.setLocalMatrix(matrix);
            p.setShader(fade);
            canvas.drawRect(left, top, right, top + length, p);
        }

        if (drawBottom) {
            matrix.setScale(1, fadeHeight * bottomFadeStrength);
            matrix.postRotate(180);
            matrix.postTranslate(left, bottom);
            fade.setLocalMatrix(matrix);
            p.setShader(fade);
            canvas.drawRect(left, bottom - length, right, bottom, p);
        }

        if (drawLeft) {
            matrix.setScale(1, fadeHeight * leftFadeStrength);
            matrix.postRotate(-90);
            matrix.postTranslate(left, top);
            fade.setLocalMatrix(matrix);
            p.setShader(fade);
            canvas.drawRect(left, top, left + length, bottom, p);
        }

        if (drawRight) {
            matrix.setScale(1, fadeHeight * rightFadeStrength);
            matrix.postRotate(90);
            matrix.postTranslate(right, top);
            fade.setLocalMatrix(matrix);
            p.setShader(fade);
            canvas.drawRect(right - length, top, right, bottom, p);
        }

        canvas.restoreToCount(saveCount);

        // Overlay is part of the content and draws beneath Foreground
        if (mOverlay != null && !mOverlay.isEmpty()) {
            mOverlay.getOverlayView().dispatchDraw(canvas);
        }

        // Step 6, draw decorations (foreground, scrollbars)
        onDrawForeground(canvas);
    }
```

## ViewGroup 的draw过程
viewgroup，没有重写draw 和ondraw方法，仅仅重写了 dispatchDraw 方法，在这个方法里面， 绘制子view。  
**任玉刚** 的开发艺术中，提到，有个特殊的方法，如果你不希望绘制自身，就设置为true。view默认关闭，viewgroup默认打开。

```java
/**
    * If this view doesn't do any drawing on its own, set this flag to
    * allow further optimizations. By default, this flag is not set on
    * View, but could be set on some View subclasses such as ViewGroup.
    *
    * Typically, if you override {@link #onDraw(android.graphics.Canvas)}
    * you should clear this flag.
    *  翻译如下，如果你复写了onDraw方法，你就应该清楚此flag。简单理解就是，这个标示位  
    * 是用来判断是否绘制ondraw方法的。
    *
    * @param willNotDraw whether or not this View draw on its own
    */
public void setWillNotDraw(boolean willNotDraw) {
        setFlags(willNotDraw ? WILL_NOT_DRAW : 0, DRAW_MASK);
    }
```
