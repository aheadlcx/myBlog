title: NestedScrolling 流程理解
date: 2016-06-13 15:57:52
tags:
---

参考文章
>[Android 嵌套滑动机制（NestedScrolling）](https://segmentfault.com/a/1190000002873657)


# 前言
传统的 Android 的 Touch 事件分发机制，主要涉及到下面3个方法  
1. dispatchTouchEvent
2. onInterceptTouchEvent
3. onTouchEvent


其中后面2个方法，是在第一个方法 dispatchTouchEvent 中调用的。在这个涉及里面下，如果父 View  
把 touch 事件传递给子 View ，并且子 View 消费了这个事件，那么父 View 就再也不能获得这个  
touch 事件了。（其实这不是绝对的，只是传统的 Android Touch 事件传递设计初衷就这样，  
子 View 的 Touch 事件也是经过父 View 的 dispatchTouchEvent 方法的嘛），这种设计，只是  
不能让父 View 和子 View ，经过交流，来回的传递 Touch 事件。  


在 design 中出现了新的设计实现，还可以兼容旧版本，就是 NestedScrolling 了，  
就是嵌套滚动的意思 。这套设计的实现，主要涉及到下面几个接口（类）  


1. NestedScrollingChild
2. NestedScrollingChildHelper
3. NestedScrollingParent
4. NestedScrollingParentHelper


# 类简介

## NestedScrollingChild
![](https://raw.githubusercontent.com/aheadlcx/myPhotos/master/NestedScroll/NestedScrollingChild.png)
如果想要配合 ViewGroup 实现嵌套滚动， View 或者 View 的子类就得实现接口  
NestedScrollingChild .并且需要创建一个 final 的 NestedScrollingChildHelper 作为  
成员变量，并且将相同点方法签名，都交给 NestedScrollingChildHelper 来代理。


## NestedScrollingChildHelper
![](https://raw.githubusercontent.com/aheadlcx/myPhotos/master/NestedScroll/NestedScrollingChildHelper.png)
这是一个帮助工具类，来实现嵌套滚动的，兼容5.0以前的版本。  
View 的子类应该创建一个 NestedScrollingChildHelper final 实例，并且把 View 和  
NestedScrollingChildHelper ，方法签名相同的方法，都委托给  NestedScrollingChildHelper  
来处理。

## NestedScrollingParent
![](https://raw.githubusercontent.com/aheadlcx/myPhotos/master/NestedScroll/NestedScrollingParent.png)
ViewGroup 的子类应该实现的接口，并且需要创建一个 final 的成员变量  
NestedScrollingParentHelper ，和 NestedScrollingChild 类似。

## NestedScrollingParentHelper
![](https://raw.githubusercontent.com/aheadlcx/myPhotos/master/NestedScroll/NestedScrollingParentHelper.png)
作用和 NestedScrollingChildHelper 类似。


# 嵌套滚动原理
我们知道新版本的 RecycleView 是支持嵌套滚动的，所以我们直接抓源码看就好了。  
我们用 RecycleView 作为 NestedScrollingChild ， CoordinatorLayout 作为  
NestedScrollingParent 来做例子分析。在这个例子中，整个嵌套 touch 事件分发的流程图如下  


![](https://raw.githubusercontent.com/aheadlcx/myPhotos/master/NestedScroll/NestedScroll_Processon.png)


整个流程涉及到的类比较多，而且逻辑也比较复杂。总体而言，需要一个 CoordinatorLayout 直接  
包裹住所有直接子 View （这也不是绝对的，不过这就是人家的设计，如果需要，当然可以重新自定义  
一个类似于 CoordinatorLayout 的 ViewGroup 了）。如果直接子 View 的 Behavior  没有拦截  
touch 事件的话，那么嵌套事件就是从子 View 开始发起的。这里先分开3部分来看，分别从 DOWN ，
MOVE 和 UP 事件来看。


## 嵌套中的 DWON 事件
### 起始于 RecycleView 中的 onTouchEvent
直接上代码 (精简版)

```java
@Override
    public boolean onTouchEvent(MotionEvent e) {

        final boolean canScrollHorizontally = mLayout.canScrollHorizontally();
        final boolean canScrollVertically = mLayout.canScrollVertically();

        boolean eventAddedToVelocityTracker = false;

        final MotionEvent vtev = MotionEvent.obtain(e);
        final int action = MotionEventCompat.getActionMasked(e);
        final int actionIndex = MotionEventCompat.getActionIndex(e);


        switch (action) {
            case MotionEvent.ACTION_DOWN: {

                int nestedScrollAxis = ViewCompat.SCROLL_AXIS_NONE;
                if (canScrollHorizontally) {
                    nestedScrollAxis |= ViewCompat.SCROLL_AXIS_HORIZONTAL;
                }
                if (canScrollVertically) {
                    nestedScrollAxis |= ViewCompat.SCROLL_AXIS_VERTICAL;
                }
                startNestedScroll(nestedScrollAxis);
            } break;



            case MotionEvent.ACTION_MOVE: {

                final int x = (int) (MotionEventCompat.getX(e, index) + 0.5f);
                final int y = (int) (MotionEventCompat.getY(e, index) + 0.5f);
                int dx = mLastTouchX - x;
                int dy = mLastTouchY - y;

                if (dispatchNestedPreScroll(dx, dy, mScrollConsumed, mScrollOffset)) {
                    dx -= mScrollConsumed[0];
                    dy -= mScrollConsumed[1];
                    vtev.offsetLocation(mScrollOffset[0], mScrollOffset[1]);
                    // Updated the nested offsets
                    mNestedOffsets[0] += mScrollOffset[0];
                    mNestedOffsets[1] += mScrollOffset[1];
                }

                if (mScrollState != SCROLL_STATE_DRAGGING) {
                    boolean startScroll = false;
                    if (canScrollHorizontally && Math.abs(dx) > mTouchSlop) {
                        if (dx > 0) {
                            dx -= mTouchSlop;
                        } else {
                            dx += mTouchSlop;
                        }
                        startScroll = true;
                    }
                    if (canScrollVertically && Math.abs(dy) > mTouchSlop) {
                        if (dy > 0) {
                            dy -= mTouchSlop;
                        } else {
                            dy += mTouchSlop;
                        }
                        startScroll = true;
                    }
                    if (startScroll) {
                        setScrollState(SCROLL_STATE_DRAGGING);
                    }
                }

                if (mScrollState == SCROLL_STATE_DRAGGING) {
                    mLastTouchX = x - mScrollOffset[0];
                    mLastTouchY = y - mScrollOffset[1];

                    if (scrollByInternal(
                            canScrollHorizontally ? dx : 0,
                            canScrollVertically ? dy : 0,
                            vtev)) {
                        getParent().requestDisallowInterceptTouchEvent(true);
                    }
                }
            } break;

            case MotionEvent.ACTION_UP: {
                resetTouch();
            } break;

        }

        if (!eventAddedToVelocityTracker) {
            mVelocityTracker.addMovement(vtev);
        }
        vtev.recycle();

        return true;
    }
```

可以看到，是直接调用的方法 startNestedScroll ，那么跟下去看看。

```java
@Override
public boolean startNestedScroll(int axes) {
    return getScrollingChildHelper().startNestedScroll(axes);
}
```

再跟看看 getScrollingChildHelper 方法。

```java
private NestedScrollingChildHelper getScrollingChildHelper() {
        if (mScrollingChildHelper == null) {
            mScrollingChildHelper = new NestedScrollingChildHelper(this);
        }
        return mScrollingChildHelper;
    }
```

一个简单的单例模式，把自己(RecycleView) 传递进去了。那么我们跟下去看看  
mScrollingChildHelper.startNestedScroll 方法。


### mScrollingChildHelper.startNestedScroll 方法

```java
/**
 * Start a new nested scroll for this view.
 *
 * <p>This is a delegate method. Call it from your {@link android.view.View View} subclass
 * method/{@link NestedScrollingChild} interface method with the same signature to implement
 * the standard policy.</p>
 *
 * @param axes Supported nested scroll axes.
 *             See {@link NestedScrollingChild#startNestedScroll(int)}.
 * @return true if a cooperating parent view was found and nested scrolling started successfully
 */
public boolean startNestedScroll(int axes) {
    if (hasNestedScrollingParent()) {
        // Already in progress
        return true;
    }
    if (isNestedScrollingEnabled()) {
        ViewParent p = mView.getParent();
        View child = mView;
        while (p != null) {
            if (ViewParentCompat.onStartNestedScroll(p, child, mView, axes)) {
                mNestedScrollingParent = p;
                ViewParentCompat.onNestedScrollAccepted(p, child, mView, axes);
                return true;
            }
            if (p instanceof View) {
                child = (View) p;
            }
            p = p.getParent();
        }
    }
    return false;
}
```

这里先做了检查，hasNestedScrollingParent 方法就检查了 mNestedScrollingParent 是否为空，  
如果不为空，就代表已经在嵌套滚动当中了，就不进行其他处理了，直接返回 true。 从后面的代码可以  
看到 mNestedScrollingParent 的赋值就是 mView 的父类（或者父父类）。
接下来我们看 ViewParentCompat.onStartNestedScroll 方法

### ViewParentCompat.onStartNestedScroll 方法

```java
public static boolean onStartNestedScroll(ViewParent parent, View child, View target,
        int nestedScrollAxes) {
    return IMPL.onStartNestedScroll(parent, child, target, nestedScrollAxes);
}
```


```java
static final ViewParentCompatImpl IMPL;
static {
    final int version = Build.VERSION.SDK_INT;
    if (version >= 21) {
        IMPL = new ViewParentCompatLollipopImpl();
    } else if (version >= 19) {
        IMPL = new ViewParentCompatKitKatImpl();
    } else if (version >= 14) {
        IMPL = new ViewParentCompatICSImpl();
    } else {
        IMPL = new ViewParentCompatStubImpl();
    }
}
```


```java
static class ViewParentCompatLollipopImpl extends ViewParentCompatKitKatImpl {
        @Override
        public boolean onStartNestedScroll(ViewParent parent, View child, View target,
                int nestedScrollAxes) {
            return ViewParentCompatLollipop.onStartNestedScroll(parent, child, target,
                    nestedScrollAxes);
        }
```


```java
static class ViewParentCompatStubImpl implements ViewParentCompatImpl {
    @Override
    public boolean requestSendAccessibilityEvent(
            ViewParent parent, View child, AccessibilityEvent event) {
        // Emulate what ViewRootImpl does in ICS and above.
        if (child == null) {
            return false;
        }
        final AccessibilityManager manager = (AccessibilityManager) child.getContext()
                .getSystemService(Context.ACCESSIBILITY_SERVICE);
        manager.sendAccessibilityEvent(event);
        return true;
    }

    @Override
    public boolean onStartNestedScroll(ViewParent parent, View child, View target,
            int nestedScrollAxes) {
        if (parent instanceof NestedScrollingParent) {
            return ((NestedScrollingParent) parent).onStartNestedScroll(child, target,
                    nestedScrollAxes);
        }
        return false;
    }
```

从上面代码可以看到，ViewParentCompat 是交给一个成员变量 IMPL 来处理的，这个 IMPL 在  
不同系统版本又有不同的实现，5.0 以上是 ViewParentCompatLollipopImpl ，5.0 以下 是  
ViewParentCompatStubImpl ， ViewParentCompat 是一个兼容库，5.0 以上  
就直接调用 View 或者 ViewGroup 的方法，5.0 以下就看是否实现了  
NestedScrollingChild 和 NestedScrollingParent ，如果实现了就调用相关的方法，否则啥也  
不干了。因此 ListView 本身是不支持嵌套滚动的，除非继承 NestedScrollingChild 并且实现  
相关的方法。  


### CoordinatorLayout 的 onStartNestedScroll 方法

```java
public boolean onStartNestedScroll(View child, View target, int nestedScrollAxes) {
        boolean handled = false;

        final int childCount = getChildCount();
        for (int i = 0; i < childCount; i++) {
            final View view = getChildAt(i);
            final LayoutParams lp = (LayoutParams) view.getLayoutParams();
            final Behavior viewBehavior = lp.getBehavior();
            if (viewBehavior != null) {
                final boolean accepted = viewBehavior.onStartNestedScroll(this, view, child, target,
                        nestedScrollAxes);
                handled |= accepted;

                lp.acceptNestedScroll(accepted);
            } else {
                lp.acceptNestedScroll(false);
            }
        }
        return handled;
    }
```

在这个方法中，就是遍历所有子 View 的 Behavior 的 onStartNestedScroll 方法，并且记录  
返回值到子 View 的 LayoutParams 中。返回 true ，代表这个 Behavior 想接受这个嵌套滚动。  
只要有一个 Behavior 返回了 true ，那么这个方法中的返回值 就为 true 。接下来再看看  
Behavior 的 onStartNestedScroll 方法。

### Behavior 的 onStartNestedScroll 方法。

```java
/**
 * Called when a descendant of the CoordinatorLayout attempts to initiate a nested scroll.
 *
 * <p>Any Behavior associated with any direct child of the CoordinatorLayout may respond
 * to this event and return true to indicate that the CoordinatorLayout should act as
 * a nested scrolling parent for this scroll. Only Behaviors that return true from
 * this method will receive subsequent nested scroll events.</p>
 *
 * @param coordinatorLayout the CoordinatorLayout parent of the view this Behavior is
 *                          associated with
 * @param child the child view of the CoordinatorLayout this Behavior is associated with
 * @param directTargetChild the child view of the CoordinatorLayout that either is or
 *                          contains the target of the nested scroll operation
 * @param target the descendant view of the CoordinatorLayout initiating the nested scroll
 * @param nestedScrollAxes the axes that this nested scroll applies to. See
 *                         {@link ViewCompat#SCROLL_AXIS_HORIZONTAL},
 *                         {@link ViewCompat#SCROLL_AXIS_VERTICAL}
 * @return true if the Behavior wishes to accept this nested scroll
 *
 * @see NestedScrollingParent#onStartNestedScroll(View, View, int)
 */
public boolean onStartNestedScroll(CoordinatorLayout coordinatorLayout,
        V child, View directTargetChild, View target, int nestedScrollAxes) {
    return false;
}
```

这个方法，默认是返回 false ，几个形参容易搞错。
1. child
是指 CoordinatorLayout 的直接子 View ，并且是提供这个 Behavior 的 view 。在这个例子中  
就是指 RecycleView  了。
2. directTargetChild
是指 CoordinatorLayout 的直接子 View ，包含 target 的 view。也就是说，可能是初始引起嵌套  
滚动的 View 的父 View 。
3. target
是指 CoordinatorLayout 的后代子 View，初始引起嵌套滚动的 View 。


回到 NestedScrollingChild 的 startNestedScroll 方法，如果我们返回 true ，那么我们就  
回进入到方法 ViewParentCompat.onNestedScrollAccepted(p, child, mView, axes) 。  
根据我们上面对 ViewParentCompat 理解，这个方法最后就是调用 NestedScrollingParent 中，  
在我们这个例子中，就是进入到 CoordinatorLayout ,那直接看 CoordinatorLayout  
的 onNestedScrollAccepted 方法。


### CoordinatorLayout 的 onNestedScrollAccepted 方法。

```java
public void onNestedScrollAccepted(View child, View target, int nestedScrollAxes) {
    mNestedScrollingParentHelper.onNestedScrollAccepted(child, target, nestedScrollAxes);
    mNestedScrollingDirectChild = child;
    mNestedScrollingTarget = target;

    final int childCount = getChildCount();
    for (int i = 0; i < childCount; i++) {
        final View view = getChildAt(i);
        final LayoutParams lp = (LayoutParams) view.getLayoutParams();
        if (!lp.isNestedScrollAccepted()) {
            continue;
        }

        final Behavior viewBehavior = lp.getBehavior();
        if (viewBehavior != null) {
            viewBehavior.onNestedScrollAccepted(this, view, child, target, nestedScrollAxes);
        }
    }
}
```

mNestedScrollingParentHelper 记录了 nestedScrollAxes ,然后就遍历所有子 View 的   
Behavior ，如果这个 Behavior 在先前的方法 onStartNestedScroll 中返回了 false ，就  
不会得到接下去的执行。如果返回了 true ，就调用 Behavior 的 onNestedScrollAccepted 方法。
意思就是，如果一个嵌套滚动被接受了，就会调用这个方法，Behavior 默认没有做任何处理。  


### 嵌套中的 Down 事件总结
在 Down 事件中，touch 事件从 NestedScrollingChild 传递到 NestedScrollingChildHelper  
, NestedScrollingParent 等，最终会传递 Behavior ，Behavior 需要表态是否接受这个  
嵌套滚动，如果不接受，那么这个 Behavior 就不会接受这个嵌套滚动的后面 touch 事件了。  
**但是一个 Behavior 不接受嵌套滚动， 并不会阻止这个 touch 事件接下来的分发流程**

## 嵌套中的 MOVE 事件

再回到上面的 RecycleView 的 onTouchEvent 源码部分。
### RecycleView 的 dispatchNestedPreScroll 方法
改方法会在 RecycleView 的 onTouchEvent 方法中调用

```java
@Override
    public boolean dispatchNestedPreScroll(int dx, int dy, int[] consumed, int[] offsetInWindow) {
        return getScrollingChildHelper().dispatchNestedPreScroll(dx, dy, consumed, offsetInWindow);
    }
```

### NestedScrollingChildHelper 的 dispatchNestedPreScroll 方法

```java
/**
     * Dispatch one step of a nested pre-scrolling operation to the current nested scrolling parent.
     *
     * <p>This is a delegate method. Call it from your {@link android.view.View View} subclass
     * method/{@link NestedScrollingChild} interface method with the same signature to implement
     * the standard policy.</p>
     *
     * @return true if the parent consumed any of the nested scroll
     */
    public boolean dispatchNestedPreScroll(int dx, int dy, int[] consumed, int[] offsetInWindow) {
        if (isNestedScrollingEnabled() && mNestedScrollingParent != null) {
            if (dx != 0 || dy != 0) {
                int startX = 0;
                int startY = 0;
                if (offsetInWindow != null) {
                    mView.getLocationInWindow(offsetInWindow);
                    startX = offsetInWindow[0];
                    startY = offsetInWindow[1];
                }

                if (consumed == null) {
                    if (mTempNestedScrollConsumed == null) {
                        mTempNestedScrollConsumed = new int[2];
                    }
                    consumed = mTempNestedScrollConsumed;
                }
                consumed[0] = 0;
                consumed[1] = 0;
                ViewParentCompat.onNestedPreScroll(mNestedScrollingParent, mView, dx, dy, consumed);

                if (offsetInWindow != null) {
                    mView.getLocationInWindow(offsetInWindow);
                    offsetInWindow[0] -= startX;
                    offsetInWindow[1] -= startY;
                }
                return consumed[0] != 0 || consumed[1] != 0;
            } else if (offsetInWindow != null) {
                offsetInWindow[0] = 0;
                offsetInWindow[1] = 0;
            }
        }
        return false;
    }
```

几个形参的意思
1. consumed
代表 NestedScrollingChild 在 NestedPred 过程中，被 NestedScrollingParent  
(应该说是他的子 View 的 Behavior )消耗了的距离.
2. offsetInWindow
在 NestedPred 流程中， NestedScrollingChild 相对屏幕左上角，位置的更改值。有可能是  
NestedScrollingChild 在 NestedPred 过程中， NestedScrollingParent 改变了位置，  
从而导致 NestedScrollingChild 相对屏幕左上角的位置改变了。  


再看 ViewParentCompat.onNestedPreScroll 方法。其实就是看 CoordinatorLayout  
的 onNestedPreScroll 方法了。


### CoordinatorLayout 的 onNestedPreScroll 方法了。

```java
public void onNestedPreScroll(View target, int dx, int dy, int[] consumed) {
    int xConsumed = 0;
    int yConsumed = 0;
    boolean accepted = false;

    final int childCount = getChildCount();
    for (int i = 0; i < childCount; i++) {
        final View view = getChildAt(i);
        final LayoutParams lp = (LayoutParams) view.getLayoutParams();
        if (!lp.isNestedScrollAccepted()) {
            continue;
        }

        final Behavior viewBehavior = lp.getBehavior();
        if (viewBehavior != null) {
            mTempIntPair[0] = mTempIntPair[1] = 0;
            viewBehavior.onNestedPreScroll(this, view, target, dx, dy, mTempIntPair);

            xConsumed = dx > 0 ? Math.max(xConsumed, mTempIntPair[0])
                    : Math.min(xConsumed, mTempIntPair[0]);
            yConsumed = dy > 0 ? Math.max(yConsumed, mTempIntPair[1])
                    : Math.min(yConsumed, mTempIntPair[1]);

            accepted = true;
        }
    }

    consumed[0] = xConsumed;
    consumed[1] = yConsumed;

    if (accepted) {
        dispatchOnDependentViewChanged(true);
    }
}
```

从这里可以看到，consumed 的值取所有所有 Behavior 消耗的最极端值（是区分了正负）。  
这意味着，NestedScrollingChildHelper.dispatchNestedPreScroll 的 consumed  
是 NestedScrollingParent 的 Behavior 消耗了的。


### 再看 CoordinatorLayout 的  dispatchOnDependentViewChanged 方法  

 ```java
 /**
      * Dispatch any dependent view changes to the relevant {@link Behavior} instances.
      *
      * Usually run as part of the pre-draw step when at least one child view has a reported
      * dependency on another view. This allows CoordinatorLayout to account for layout
      * changes and animations that occur outside of the normal layout pass.
      *
      * It can also be ran as part of the nested scrolling dispatch to ensure that any offsetting
      * is completed within the correct coordinate window.
      *
      * The offsetting behavior implemented here does not store the computed offset in
      * the LayoutParams; instead it expects that the layout process will always reconstruct
      * the proper positioning.
      *
      * @param fromNestedScroll true if this is being called from one of the nested scroll methods,
      *                         false if run as part of the pre-draw step.
      */
     void dispatchOnDependentViewChanged(final boolean fromNestedScroll) {
         final int layoutDirection = ViewCompat.getLayoutDirection(this);
         final int childCount = mDependencySortedChildren.size();
         for (int i = 0; i < childCount; i++) {
             final View child = mDependencySortedChildren.get(i);
             final LayoutParams lp = (LayoutParams) child.getLayoutParams();

             // Check child views before for anchor
             for (int j = 0; j < i; j++) {
                 final View checkChild = mDependencySortedChildren.get(j);

                 if (lp.mAnchorDirectChild == checkChild) {
                     offsetChildToAnchor(child, layoutDirection);
                 }
             }

             // Did it change? if not continue
             final Rect oldRect = mTempRect1;
             final Rect newRect = mTempRect2;
             getLastChildRect(child, oldRect);
             getChildRect(child, true, newRect);
             if (oldRect.equals(newRect)) {
                 continue;
             }
             recordLastChildRect(child, newRect);

             // Update any behavior-dependent views for the change
             for (int j = i + 1; j < childCount; j++) {
                 final View checkChild = mDependencySortedChildren.get(j);
                 final LayoutParams checkLp = (LayoutParams) checkChild.getLayoutParams();
                 final Behavior b = checkLp.getBehavior();

                 if (b != null && b.layoutDependsOn(this, checkChild, child)) {
                     if (!fromNestedScroll && checkLp.getChangedAfterNestedScroll()) {
                         // If this is not from a nested scroll and we have already been changed
                         // from a nested scroll, skip the dispatch and reset the flag
                         checkLp.resetChangedAfterNestedScroll();
                         continue;
                     }

                     final boolean handled = b.onDependentViewChanged(this, checkChild, child);

                     if (fromNestedScroll) {
                         // If this is from a nested scroll, set the flag so that we may skip
                         // any resulting onPreDraw dispatch (if needed)
                         checkLp.setChangedAfterNestedScroll(handled);
                     }
                 }
             }
         }
     }
 ```

 这里可能会调用 Behavior 的2个方法
 layoutDependsOn 和 onDependentViewChanged 方法。


 ### Behavior 的 layoutDependsOn 方法

```java
/**
 * Determine whether the supplied child view has another specific sibling view as a
 * layout dependency.
 *
 * <p>This method will be called at least once in response to a layout request. If it
 * returns true for a given child and dependency view pair, the parent CoordinatorLayout
 * will:</p>
 * <ol>
 *     <li>Always lay out this child after the dependent child is laid out, regardless
 *     of child order.</li>
 *     <li>Call {@link #onDependentViewChanged} when the dependency view's layout or
 *     position changes.</li>
 * </ol>
 *
 * @param parent the parent view of the given child
 * @param child the child view to test
 * @param dependency the proposed dependency of child
 * @return true if child's layout depends on the proposed dependency's layout,
 *         false otherwise
 *
 * @see #onDependentViewChanged(CoordinatorLayout, android.view.View, android.view.View)
 */
public boolean layoutDependsOn(CoordinatorLayout parent, V child, View dependency) {
    return false;
}

```

形参的含义
 询问 child 到底是否依赖 dependency 。如果返回 true ， CoordinatorLayout 会在  
 lay out 完 dependency 之后，不管 child 顺序，就 lay out 这个形参 child 了。

 ### Behavior 的 onDependentViewChanged 方法

```java
/**
 * Respond to a change in a child's dependent view
 *
 * <p>This method is called whenever a dependent view changes in size or position outside
 * of the standard layout flow. A Behavior may use this method to appropriately update
 * the child view in response.</p>
 *
 * <p>A view's dependency is determined by
 * {@link #layoutDependsOn(CoordinatorLayout, android.view.View, android.view.View)} or
 * if {@code child} has set another view as it's anchor.</p>
 *
 * <p>Note that if a Behavior changes the layout of a child via this method, it should
 * also be able to reconstruct the correct position in
 * {@link #onLayoutChild(CoordinatorLayout, android.view.View, int) onLayoutChild}.
 * <code>onDependentViewChanged</code> will not be called during normal layout since
 * the layout of each child view will always happen in dependency order.</p>
 *
 * <p>If the Behavior changes the child view's size or position, it should return true.
 * The default implementation returns false.</p>
 *
 * @param parent the parent view of the given child
 * @param child the child view to manipulate
 * @param dependency the dependent view that changed
 * @return true if the Behavior changed the child view's size or position, false otherwise
 */
public boolean onDependentViewChanged(CoordinatorLayout parent, V child, View dependency) {
    return false;
}
```

每当 dependency 改变了位置或者大小的时候，都会触发此方法，这个方法是一个恰当的时机去改变  
形参中的 child view 。如果在这个方法里面改变了 child 的位置或者大小，必须可以在  
CoordinatorLayout 的 onLayoutChild 中可以复原。**undo 意味着，改变 left top 没效咯？**    
**mark**  
在 Behavior 的方法 onLayoutChild 注解中可以看到，实现了 onDependentViewChanged  
，那么也应该实现 onLayoutChild 方法。
如果改变了 child ，该方法就应该返回  true 。


### 嵌套的 Move 事件中的 NestedPre 过程小结
由 NestedScrollingChild.dispatchNestedPreScroll 发起，最后给到  
NestedScrollingParent 的 Behavior 来消费。  
1. 需要注意的是，在 NestedScrollingChild 中，需要对 Behavior 消费的距离，  
以及因为 NestedScrollingParent 改变位置，导致 NestedScrollingChild   
相对屏幕左上角坐标的改变，做相应的调整。  
2. layoutDependsOn 返回 true 的话，会接着调用 onDependentViewChanged ，此方法  
是一个比较恰当的时机去改变 child ，但是需要注意需要保证在 CoordinatorLayout  的  
 onLayoutChild 方法中可以复原。

### 再回到 RecycleView 的 onTouchEvent 和 scrollByInternal 方法
可以看到调用了 RecycleView 的 scrollByInternal 方法。

```java
/**
     * Does not perform bounds checking. Used by internal methods that have already validated input.
     * <p>
     * It also reports any unused scroll request to the related EdgeEffect.
     *
     * @param x The amount of horizontal scroll request
     * @param y The amount of vertical scroll request
     * @param ev The originating MotionEvent, or null if not from a touch event.
     *
     * @return Whether any scroll was consumed in either direction.
     */
    boolean scrollByInternal(int x, int y, MotionEvent ev) {
        int unconsumedX = 0, unconsumedY = 0;
        int consumedX = 0, consumedY = 0;

        consumePendingUpdateOperations();
        if (mAdapter != null) {
            eatRequestLayout();
            onEnterLayoutOrScroll();
            TraceCompat.beginSection(TRACE_SCROLL_TAG);
            if (x != 0) {
                consumedX = mLayout.scrollHorizontallyBy(x, mRecycler, mState);
                unconsumedX = x - consumedX;
            }
            if (y != 0) {
                consumedY = mLayout.scrollVerticallyBy(y, mRecycler, mState);
                unconsumedY = y - consumedY;
            }
            TraceCompat.endSection();
            repositionShadowingViews();
            onExitLayoutOrScroll();
            resumeRequestLayout(false);
        }
        if (!mItemDecorations.isEmpty()) {
            invalidate();
        }

        if (dispatchNestedScroll(consumedX, consumedY, unconsumedX, unconsumedY, mScrollOffset)) {
            // Update the last touch co-ords, taking any scroll offset into account
            mLastTouchX -= mScrollOffset[0];
            mLastTouchY -= mScrollOffset[1];
            if (ev != null) {
                ev.offsetLocation(mScrollOffset[0], mScrollOffset[1]);
            }
            mNestedOffsets[0] += mScrollOffset[0];
            mNestedOffsets[1] += mScrollOffset[1];
        } else if (ViewCompat.getOverScrollMode(this) != ViewCompat.OVER_SCROLL_NEVER) {
            if (ev != null) {
                pullGlows(ev.getX(), unconsumedX, ev.getY(), unconsumedY);
            }
            considerReleasingGlowsOnScroll(x, y);
        }
        if (consumedX != 0 || consumedY != 0) {
            dispatchOnScrolled(consumedX, consumedY);
        }
        if (!awakenScrollBars()) {
            invalidate();
        }
        return consumedX != 0 || consumedY != 0;
    }
```

和嵌套滚动相关的，就是调用了 dispatchNestedScroll ，这个方法里面，实际就是调用了  
NestedScrollingChild 的 dispatchNestedScroll 方法。


### NestedScrollingChild 的 dispatchNestedScroll 方法。

```java
/**
     * Dispatch one step of a nested scrolling operation to the current nested scrolling parent.
     *
     * <p>This is a delegate method. Call it from your {@link android.view.View View} subclass
     * method/{@link NestedScrollingChild} interface method with the same signature to implement
     * the standard policy.</p>
     *
     * @return true if the parent consumed any of the nested scroll
     */
    public boolean dispatchNestedScroll(int dxConsumed, int dyConsumed,
            int dxUnconsumed, int dyUnconsumed, int[] offsetInWindow) {
        if (isNestedScrollingEnabled() && mNestedScrollingParent != null) {
            if (dxConsumed != 0 || dyConsumed != 0 || dxUnconsumed != 0 || dyUnconsumed != 0) {
                int startX = 0;
                int startY = 0;
                if (offsetInWindow != null) {
                    mView.getLocationInWindow(offsetInWindow);
                    startX = offsetInWindow[0];
                    startY = offsetInWindow[1];
                }

                ViewParentCompat.onNestedScroll(mNestedScrollingParent, mView, dxConsumed,
                        dyConsumed, dxUnconsumed, dyUnconsumed);

                if (offsetInWindow != null) {
                    mView.getLocationInWindow(offsetInWindow);
                    offsetInWindow[0] -= startX;
                    offsetInWindow[1] -= startY;
                }
                return true;
            } else if (offsetInWindow != null) {
                // No motion, no dispatch. Keep offsetInWindow up to date.
                offsetInWindow[0] = 0;
                offsetInWindow[1] = 0;
            }
        }
        return false;
    }
```

这里，又算了下，让 NestedScrollingParent 的位置改变，导致 NestedScrollingChild  
相对屏幕左上角的改变。用 offsetInWindow 记录了。 NestedScrollingChild 可能需要对  
这个做相应的处理。中间调用的 ViewParentCompat.onNestedScroll 实际就是调用的  
CoordinatorLayout 的 onNestedScroll


### CoordinatorLayout 的 onNestedScroll

```java
public void onNestedScroll(View target, int dxConsumed, int dyConsumed,
            int dxUnconsumed, int dyUnconsumed) {
        final int childCount = getChildCount();
        boolean accepted = false;

        for (int i = 0; i < childCount; i++) {
            final View view = getChildAt(i);
            final LayoutParams lp = (LayoutParams) view.getLayoutParams();
            if (!lp.isNestedScrollAccepted()) {
                continue;
            }

            final Behavior viewBehavior = lp.getBehavior();
            if (viewBehavior != null) {
                viewBehavior.onNestedScroll(this, view, target, dxConsumed, dyConsumed,
                        dxUnconsumed, dyUnconsumed);
                accepted = true;
            }
        }

        if (accepted) {
            dispatchOnDependentViewChanged(true);
        }
    }
```

其实都是给 Behavior 来处理了。

### 嵌套滚动中 MOVE 事件整体小结
在 NestedPre 中，NestedScrollingChild 告诉 NestedScrollingParent 有多少滚动距离，  
NestedScrollingParent 把滚动都给 Behavior 的 onNestedPreScroll 来处理，消费了多少，  
需要记录下来，并告诉 NestedScrollingParent .这个 NestedScrollingParent 的 Behavior  
消耗了多少，告诉 NestedScrollingChild 使用的方式是， NestedScrollingChild 传递  
一个引用给 NestedScrollingParent ，NestedScrollingParent 再修改这个引用指向的对象  
部分的属性.  

在 Nested 中，NestedScrollingChild 自己消费一部分，告诉 NestedScrollingParent  
自己消费了多少，没有消费的还有多少，NestedScrollingParent 再全部告诉 Behavior 。  

**标准的做法就是**

+ 自己先不消耗滚动距离，让 NestedScrollingParent 的 onNestedPreScroll   
方法优先消费滚动距离。然后 中，NestedScrollingChild 记录一下这个过程中消费的距离。
+ 中，NestedScrollingChild 自己消耗一下滚动距离.
+ 然后 中，NestedScrollingChild 把已经消费了的，和未消费的，给到 NestedScrollingParent  
的方法 onNestedScroll 。

## 嵌套中的 UP 事件
这个就比较简单了，就是调用 NestedScrollingChild 的 stopNestedScroll ，接着  
NestedScrollingChildHelper  的 stopNestedScroll ，最后就是  
NestedScrollingParent 的 Behavior 的 stopNestedScroll 了。
