title: 我所理解的View绘制
date: 2015-12-21 19:25:49
tags: android
---
<!-- 本文是对View绘制的理解的笔记文章，以及一些三方View的源码分析 -->
<!--more test  -->

>本文参考了众多前辈的文章和书籍
>>[任玉刚的相关文章和书籍](http://blog.csdn.net/singwhatiwanna)
>>[郭霖博客](http://blog.csdn.net/guolin_blog)

# Activity的window，ViewRootImpl,DecorView
* ViewRoot对应于ViewRoortImpl,连接WindowManager和DecorView
* View的绘制流程，是从ViewRoot的performTraversals方法开始，先后调用，
  1,performMeasure
  2,performLayout
  3,performDraw


# MeasureSpec在View绘制中的作用
childView.measure(childWidthMeasureSpec, childHeightMeasureSpec)
## MeasureSpec本身的含义
本身代表一个32位的int值，高二位代表SpecMode，低30位代表SpecSize，SpecMode有三个类
* UNSPECIFIED, 父容器不对View有任何限制。
* EXACTLY 精确值，已经对View检测出精确的大小。这个时候View的最终大小就是SpecSize，这个一般对应LayoutParams中的Match-parent，以及具体写死的dp值。
* AT_MOST 最多多少，给出一个值SpecSize，表示view最大可以这么多。一般对应LayoutParams中的wrap-content

## MeasureSpec和LayoutParams的关系
MeasureSpec可以决定view的宽高，我们布局中的LayoutParams可以影响MeasureSpec, 但不是唯一可以影响MeasureSpec的
* 对于DecorView，他的MeasureSpec是由窗口的大小（其实就是屏幕的宽高）和自身的LayoutParams来决定的
* 对于普通的View来说，MeasureSpec是由父View的MeasureSpec 和父View的padding和margin， 以及自身的LayoutParams有关。可以参考ViewGroup代码

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

## ViewGroup的measure过程
ViewGroup没有实现，交给子View去实现。
***值得注意的是， 在measure之后，拿GetMeasureWidth(),在极端情况下，拿不到或者不准确或者需要系统多次Measure之后才可以拿到，  
建议在Layout过程之后在去拿GetMeasureWidth***

## View在ViewGroup中，被出发measure的过程
由此可见，计算宽高的时候，是把父View的padding和子View的margin算进去的，如果给子View设置背景，  
margin值部分是没有背景的。

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

下面代码可以看出父View的MeasureSpec和子View的LayoutParams(下面代码体现为childDimension，这个值，这个值在子View构造方法中就从XML中加载出来了)，  共同来决定子View的MeasureSpec值。
在不考了UNSPECIFIED情况下，只有子View的宽高是写死dp的情况下，子View的MeasureSpec中的specSize都是  
父View的可用宽高（其实就是，父View宽高减去（padding和子View的margin值）之和）//- [需要注意，未明 ]

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

## View的自身measure过程
widthMeasureSpec和heightMeasureSpec，都是由父View传递过来的。
下面方法就是设置宽高值

```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
            getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
}
```

由下面代码可以看出，返回的值，就是specSize(在UNSPECIFIED情况选，就是size)
**值得注意的是AT_MOST和EXACTLY返回的值是一样的**  
所以自定义View直接继承View的话，需要自己处理区分这2种情况，可以参考TextView的写法

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

没有背景就取android:minWidth,这个值就是mMinWidth

```java
protected int getSuggestedMinimumWidth() {
    return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinimumWidth());
}
```

获得图片的原始宽度，其中，shapwDrawable没有原始宽度，BitmapDrawable则有原始宽度

```java
public int getMinimumWidth() {
    final int intrinsicWidth = getIntrinsicWidth();
    return intrinsicWidth > 0 ? intrinsicWidth : 0;
}
```
