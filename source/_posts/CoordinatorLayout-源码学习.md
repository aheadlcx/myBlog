title: CoordinatorLayout 源码学习
date: 2016-06-12 10:56:19
tags:
---

源码版本号
design : 23.3.0

## 触摸事件分发
并没有重载 dispatchTouchEvent 方法，仅仅重载了 onInterceptTouchEvent 和  
onTouchEvent 方法。

### onInterceptTouchEvent 方法

```java
@Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        MotionEvent cancelEvent = null;

        final int action = MotionEventCompat.getActionMasked(ev);

        // Make sure we reset in case we had missed a previous important event.
        if (action == MotionEvent.ACTION_DOWN) {
            resetTouchBehaviors();
        }

        final boolean intercepted = performIntercept(ev, TYPE_ON_INTERCEPT);

        if (cancelEvent != null) {
            cancelEvent.recycle();
        }

        if (action == MotionEvent.ACTION_UP || action == MotionEvent.ACTION_CANCEL) {
            resetTouchBehaviors();
        }

        return intercepted;
    }
```

可以看到，这个方法体内，如果检测到时 DOWN 或者 UP 或者 CANCEL 事件，都会调用方法  
resetTouchBehaviors ，尝试让恢复到初始状态。 onInterceptTouchEvent 方法返回值取方法  
performIntercept 的返回值。
接下来，跟踪下方法 resetTouchBehaviors 和 performIntercept 。


### resetTouchBehaviors 方法

```java
/**
     * Reset all Behavior-related tracking records either to clean up or in preparation
     * for a new event stream. This should be called when attached or detached from a window,
     * in response to an UP or CANCEL event, when intercept is request-disallowed
     * and similar cases where an event stream in progress will be aborted.
     */
    private void resetTouchBehaviors() {
        if (mBehaviorTouchView != null) {
            final Behavior b = ((LayoutParams) mBehaviorTouchView.getLayoutParams()).getBehavior();
            if (b != null) {
                final long now = SystemClock.uptimeMillis();
                final MotionEvent cancelEvent = MotionEvent.obtain(now, now,
                        MotionEvent.ACTION_CANCEL, 0.0f, 0.0f, 0);
                b.onTouchEvent(this, mBehaviorTouchView, cancelEvent);
                cancelEvent.recycle();
            }
            mBehaviorTouchView = null;
        }

        final int childCount = getChildCount();
        for (int i = 0; i < childCount; i++) {
            final View child = getChildAt(i);
            final LayoutParams lp = (LayoutParams) child.getLayoutParams();
            lp.resetTouchBehaviorTracking();
        }
        mDisallowInterceptReset = false;
    }
```

重置 touch 状态。这方法体内，做的事情是，发送一个 cancelEvent 给 Behavior 。并且  
把 mBehaviorTouchView 置空。并且遍历所有子 View ，调用子 View 的  
LayoutParams.resetTouchBehaviorTracking(). 改方法做的事情，就是把标志位改为 false。

```java
/**
 * Reset tracking of Behavior-specific touch interactions. This includes
 * interaction blocking.
 *
 * @see #isBlockingInteractionBelow(CoordinatorLayout, android.view.View)
 * @see #didBlockInteraction()
 */
void resetTouchBehaviorTracking() {
    mDidBlockInteraction = false;
}

```

### performIntercept 方法

```java
private boolean performIntercept(MotionEvent ev, final int type) {
        boolean intercepted = false;
        boolean newBlock = false;

        MotionEvent cancelEvent = null;

        final int action = MotionEventCompat.getActionMasked(ev);

        final List<View> topmostChildList = mTempList1;
        getTopSortedChildren(topmostChildList);

        // Let topmost child views inspect first
        final int childCount = topmostChildList.size();
        for (int i = 0; i < childCount; i++) {
            final View child = topmostChildList.get(i);
            final LayoutParams lp = (LayoutParams) child.getLayoutParams();
            final Behavior b = lp.getBehavior();

            if ((intercepted || newBlock) && action != MotionEvent.ACTION_DOWN) {
                // Cancel all behaviors beneath the one that intercepted.
                // If the event is "down" then we don't have anything to cancel yet.
                if (b != null) {
                    if (cancelEvent == null) {
                        final long now = SystemClock.uptimeMillis();
                        cancelEvent = MotionEvent.obtain(now, now,
                                MotionEvent.ACTION_CANCEL, 0.0f, 0.0f, 0);
                    }
                    switch (type) {
                        case TYPE_ON_INTERCEPT:
                            b.onInterceptTouchEvent(this, child, cancelEvent);
                            break;
                        case TYPE_ON_TOUCH:
                            b.onTouchEvent(this, child, cancelEvent);
                            break;
                    }
                }
                continue;
            }

            if (!intercepted && b != null) {
                switch (type) {
                    case TYPE_ON_INTERCEPT:
                        intercepted = b.onInterceptTouchEvent(this, child, ev);
                        break;
                    case TYPE_ON_TOUCH:
                        intercepted = b.onTouchEvent(this, child, ev);
                        break;
                }
                if (intercepted) {
                    mBehaviorTouchView = child;
                }
            }

            // Don't keep going if we're not allowing interaction below this.
            // Setting newBlock will make sure we cancel the rest of the behaviors.
            final boolean wasBlocking = lp.didBlockInteraction();
            final boolean isBlocking = lp.isBlockingInteractionBelow(this, child);
            newBlock = isBlocking && !wasBlocking;
            if (isBlocking && !newBlock) {
                // Stop here since we don't have anything more to cancel - we already did
                // when the behavior first started blocking things below this point.
                break;
            }
        }

        topmostChildList.clear();

        return intercepted;
    }
```

这个方法体内，做了如下
1. 首先将所有子 View ，根据 Z 轴排序。然后根据这个排序，遍历所有子 View ，  
2. 如果发现这个 event 事件，已经被拦截，或者被阻塞了，就发送一个 cancelEvent 给子 View  
的 Behavior 。
3. 如果这个 event 事件，还没有被拦截并且子 View 的 Behavior 不为空，就把这个 event 交给  
 该子 View 的 Behavior 来处理。至于调用 Behavior 的 onInterceptTouchEvent 方法，还是  
 onTouchEvent 方法，取决于，这个方法的入参 type 了。
4. 如果有子 View 拦截了（即 intercepted == true），则把该子 View 赋值给  
mBehaviorTouchView .
5. 检查下，是否正在阻塞或者新的阻塞，并且做了一些处理。


### onTouchEvent 方法

```java
@Override
    public boolean onTouchEvent(MotionEvent ev) {
        boolean handled = false;
        boolean cancelSuper = false;
        MotionEvent cancelEvent = null;

        final int action = MotionEventCompat.getActionMasked(ev);

        if (mBehaviorTouchView != null || (cancelSuper = performIntercept(ev, TYPE_ON_TOUCH))) {
            // Safe since performIntercept guarantees that
            // mBehaviorTouchView != null if it returns true
            final LayoutParams lp = (LayoutParams) mBehaviorTouchView.getLayoutParams();
            final Behavior b = lp.getBehavior();
            if (b != null) {
                handled = b.onTouchEvent(this, mBehaviorTouchView, ev);
            }
        }

        // Keep the super implementation correct
        if (mBehaviorTouchView == null) {
            handled |= super.onTouchEvent(ev);
        } else if (cancelSuper) {
            if (cancelEvent == null) {
                final long now = SystemClock.uptimeMillis();
                cancelEvent = MotionEvent.obtain(now, now,
                        MotionEvent.ACTION_CANCEL, 0.0f, 0.0f, 0);
            }
            super.onTouchEvent(cancelEvent);
        }

        if (!handled && action == MotionEvent.ACTION_DOWN) {

        }

        if (cancelEvent != null) {
            cancelEvent.recycle();
        }

        if (action == MotionEvent.ACTION_UP || action == MotionEvent.ACTION_CANCEL) {
            resetTouchBehaviors();
        }

        return handled;
    }
```

在这个方法体内，做了如下事情
1. 如果 mBehaviorTouchView 不为空，就把这个事件，交给 mBehaviorTouchView 的  
LayoutParams 的 Behavior 的方法 onTouchEvent 来处理。并取返回值，赋值给 handled 。  
2. 如果 mBehaviorTouchView 为空，则把这个事件交给 CoordinatorLayout 的父类来处理。  
直接调用的 handled |= super.onTouchEvent(ev);
3. 如果 cancelSuper  == true （cancelSuper 是取 performIntercept 的返回值的），  
那么就制造一个 cancelEvent 给 CoordinatorLayout 的父类来处理。  
super.onTouchEvent(cancelEvent).
4. 如果是 UP 或者 CANCEL 事件，就调用 resetTouchBehaviors 方法
5. 整个方法，返回值取  handled .


## 一些方法作用简介
记录一些方法的作用
1. parseBehavior

```java
static Behavior parseBehavior(Context context, AttributeSet attrs, String name) {
..
}
```

根据 name 来构造一个 Behavior


2. getResolvedLayoutParams

```java
LayoutParams getResolvedLayoutParams(View child) {
...
}
```

根据这个 child 来找到这个 child 类的 DefaultBehavior 。并且赋值给 这个 child 的  
CoordinatorLayout.LayoutParams  
result.setBehavior(defaultBehavior.value().newInstance());


3. prepareChildren

```java
private void prepareChildren() {
..
}
```

遍历所有子 View ，准备好 LayoutParams 并且找好 AnchorView 。并且按照约定，排序好。  
mDependencySortedChildren


4. dispatchOnDependentViewChanged

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
}
```



## Behavior 相关
Behavior 的对象，其实是存在于 CoordinatorLayout 的 LayoutParams 上的，是 LayoutParams  
的一个成员变量。因此，这个 Behavior 仅仅存在于 CoordinatorLayout 的直接子类中，才  
会有所作用。

```java
/**
 * Parameters describing the desired layout for a child of a {@link CoordinatorLayout}.
 */
public static class LayoutParams extends ViewGroup.MarginLayoutParams {
    /**
     * A {@link Behavior} that the child view should obey.
     */
    Behavior mBehavior;
  }
```

### Behavior 的设置方法
Behavior 的设置方式有2种。注解方式和在 XML 中直接指定。他们不仅在使用上有点区别，在  
Behavior 的具体生成时机，也是有点不一样的。
#### 注解方式
我们可以看看 AppBarLayout 学习一下。

```java
@CoordinatorLayout.DefaultBehavior(AppBarLayout.Behavior.class)
public class AppBarLayout extends LinearLayout {
..
}
```

这种方式，Behavior 的确立生成时机是:


getResolvedLayoutParams -> prepareChildren ->  onMeasure

先看看 getResolvedLayoutParams 方法的实现原理：涉及到的逻辑有，看看是否已经添加  
Behavior ，查看这个 view child 的注解，是否包含 DefaultBehavior 。如果没有就找这个  
view child 的父类。由此可见，我们可以给一个通用的 View 添加一个默认的 DefaultBehavior  
，如果有更深层的自定义需求，就可以继承这个通用 view 来实现。

```java
LayoutParams getResolvedLayoutParams(View child) {
        final LayoutParams result = (LayoutParams) child.getLayoutParams();
        if (!result.mBehaviorResolved) {
            Class<?> childClass = child.getClass();
            DefaultBehavior defaultBehavior = null;
            while (childClass != null &&
                    (defaultBehavior = childClass.getAnnotation(DefaultBehavior.class)) == null) {
                childClass = childClass.getSuperclass();
            }
            if (defaultBehavior != null) {
                try {
                    result.setBehavior(defaultBehavior.value().newInstance());
                } catch (Exception e) {
                    Log.e(TAG, "Default behavior class " + defaultBehavior.value().getName() +
                            " could not be instantiated. Did you forget a default constructor?", e);
                }
            }
            result.mBehaviorResolved = true;
        }
        return result;
    }
```

再看看 prepareChildren 方法，实现原理。就是遍历所有子 View ，尝试找到 Behavior 并且  
生成一个实例，并绑定到自己身上。

```java
private void prepareChildren() {
        mDependencySortedChildren.clear();
        for (int i = 0, count = getChildCount(); i < count; i++) {
            final View child = getChildAt(i);

            final LayoutParams lp = getResolvedLayoutParams(child);
            lp.findAnchorView(this, child);

            mDependencySortedChildren.add(child);
        }
        // We need to use a selection sort here to make sure that every item is compared
        // against each other
        selectionSort(mDependencySortedChildren, mLayoutDependencyComparator);
    }
```

#### 在 XML 中直接指定

```java
<android.support.v4.view.ViewPager
              android:id="@+id/viewpager"
              android:layout_width="match_parent"
              android:layout_height="match_parent"
              app:layout_behavior="@string/appbar_scrolling_view_behavior"
              />
```

Behavior  生成时机是：在 inflate 流程中，我们可以知道，子 View 的 LayoutParams  
是由父 View的 generateLayoutParams 方法提供的。而 Behavior 又是  
CoordinatorLayout.LayoutParams 的成员变量，很自然的，可以死追索到如下生成链  
generateLayoutParams -> LayoutParams 的 构造方法 -> parseBehavior  
来看看 parseBehavior 的源码实现。

```java
static Behavior parseBehavior(Context context, AttributeSet attrs, String name) {
        if (TextUtils.isEmpty(name)) {
            return null;
        }

        final String fullName;
        if (name.startsWith(".")) {
            // Relative to the app package. Prepend the app package name.
            fullName = context.getPackageName() + name;
        } else if (name.indexOf('.') >= 0) {
            // Fully qualified package name.
            fullName = name;
        } else {
            // Assume stock behavior in this package (if we have one)
            fullName = !TextUtils.isEmpty(WIDGET_PACKAGE_NAME)
                    ? (WIDGET_PACKAGE_NAME + '.' + name)
                    : name;
        }

        try {
            Map<String, Constructor<Behavior>> constructors = sConstructors.get();
            if (constructors == null) {
                constructors = new HashMap<>();
                sConstructors.set(constructors);
            }
            Constructor<Behavior> c = constructors.get(fullName);
            if (c == null) {
                final Class<Behavior> clazz = (Class<Behavior>) Class.forName(fullName, true,
                        context.getClassLoader());
                c = clazz.getConstructor(CONSTRUCTOR_PARAMS);
                c.setAccessible(true);
                constructors.put(fullName, c);
            }
            return c.newInstance(context, attrs);
        } catch (Exception e) {
            throw new RuntimeException("Could not inflate Behavior subclass " + fullName, e);
        }
    }
```

## CoordinatorLayout 绘制流程
CoordinatorLayout  重写了，onMeasure onLayout onDraw 这几个流程。

### onMeasure on CoordinatorLayout 方法

```java
prepareChildren();
        ensurePreDrawListener();
```

先调用上面2个方法，为下面做准备。  
大致流程就是，遍历所有子 View ，如果子 View 的 Behavior 不为空，就交给 Behavior 处理，  
如果 Behavior 消费了对这个子 View 的 measure 流程，则 CoordinatorLayout 就不再对此  
子 View 进行 measure 了。  
关键代码如下  

```java
if (b == null || !b.onMeasureChild(this, child, childWidthMeasureSpec, keylineWidthUsed,
                    childHeightMeasureSpec, 0)) {
                onMeasureChild(child, childWidthMeasureSpec, keylineWidthUsed,
                        childHeightMeasureSpec, 0);
            }
```

### onLayout on CoordinatorLayout 方法

```java
@Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        final int layoutDirection = ViewCompat.getLayoutDirection(this);
        final int childCount = mDependencySortedChildren.size();
        for (int i = 0; i < childCount; i++) {
            final View child = mDependencySortedChildren.get(i);
            final LayoutParams lp = (LayoutParams) child.getLayoutParams();
            final Behavior behavior = lp.getBehavior();

            if (behavior == null || !behavior.onLayoutChild(this, child, layoutDirection)) {
                onLayoutChild(child, layoutDirection);
            }
        }
    }
```

简单理解的话，就是 如果 Behavior 不为空，并且消费了这个 layout 过程的话，就不处理了。否则就  
交给自己处理。

### onDraw on CoordinatorLayout 方法

```java
@Override
    public void onDraw(Canvas c) {
        super.onDraw(c);
        if (mDrawStatusBarBackground && mStatusBarBackground != null) {
            final int inset = mLastInsets != null ? mLastInsets.getSystemWindowInsetTop() : 0;
            if (inset > 0) {
                mStatusBarBackground.setBounds(0, 0, getWidth(), inset);
                mStatusBarBackground.draw(c);
            }
        }
    }
```

特别绘制了，状态栏的背景。
