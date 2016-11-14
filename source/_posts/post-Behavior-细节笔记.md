title: Behavior 细节笔记
date: 2016-06-20 15:27:27
tags:
---

##  AppBarLayout 细节笔记
1. 滚动的实现
和 CoordinatorLayout 配合时，如果触摸到 AppBarLayout 部分， AppBarLayout 的滚动  
是在 Behavior (HeaderBehavior) 的 onTouchEvent 方法中处理的。  
在 ACTION_MOVE 中可以看到，是通过 scroll 方法实现滚动的。实际实现就是改变 view 的 top 值。  
在 ACTION_UP 中，调用 fling 来实现滑动至完整的 view 。

```java
@Override
    public boolean onTouchEvent(CoordinatorLayout parent, V child, MotionEvent ev) {
        if (mTouchSlop < 0) {
            mTouchSlop = ViewConfiguration.get(parent.getContext()).getScaledTouchSlop();
        }

        switch (MotionEventCompat.getActionMasked(ev)) {
            case MotionEvent.ACTION_DOWN: {
                final int x = (int) ev.getX();
                final int y = (int) ev.getY();

                if (parent.isPointInChildBounds(child, x, y) && canDragView(child)) {
                    mLastMotionY = y;
                    mActivePointerId = MotionEventCompat.getPointerId(ev, 0);
                    ensureVelocityTracker();
                } else {
                    return false;
                }
                break;
            }

            case MotionEvent.ACTION_MOVE: {
                final int activePointerIndex = MotionEventCompat.findPointerIndex(ev,
                        mActivePointerId);
                if (activePointerIndex == -1) {
                    return false;
                }

                final int y = (int) MotionEventCompat.getY(ev, activePointerIndex);
                int dy = mLastMotionY - y;

                if (!mIsBeingDragged && Math.abs(dy) > mTouchSlop) {
                    mIsBeingDragged = true;
                    if (dy > 0) {
                        dy -= mTouchSlop;
                    } else {
                        dy += mTouchSlop;
                    }
                }

                if (mIsBeingDragged) {
                    mLastMotionY = y;
                    // We're being dragged so scroll the ABL
                    scroll(parent, child, dy, getMaxDragOffset(child), 0);
                }
                break;
            }

            case MotionEvent.ACTION_UP:
                if (mVelocityTracker != null) {
                    mVelocityTracker.addMovement(ev);
                    mVelocityTracker.computeCurrentVelocity(1000);
                    float yvel = VelocityTrackerCompat.getYVelocity(mVelocityTracker,
                            mActivePointerId);
                    fling(parent, child, -getScrollRangeForDragFling(child), 0, yvel);
                }
                // $FALLTHROUGH
            case MotionEvent.ACTION_CANCEL: {
                mIsBeingDragged = false;
                mActivePointerId = INVALID_POINTER;
                if (mVelocityTracker != null) {
                    mVelocityTracker.recycle();
                    mVelocityTracker = null;
                }
                break;
            }
        }

        if (mVelocityTracker != null) {
            mVelocityTracker.addMovement(ev);
        }

        return true;
    }
```


2. layout_anchorGravity 和 layout_gravity 的区别
layout_anchorGravity 是相对于依赖的 View 的，layout_gravity 在




**undo** 依赖于一个 View ，是否可以解决，一个 View 的位置需要根据另外一个 View   
的位置而定的需求。应该可以的，改天实现以下。
