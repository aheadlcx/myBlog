title: AndroidTV 开发之焦点处理
date: 2016-09-23 14:30:16
tags:
---
TV 知识点学习整理笔记。
<!--more  -->
## 焦点的获得
焦点的获得，分为2种情况，主动和被动  

+ 主动获得焦点  
+ 被动获得焦点


### 主动获得焦点
这里也是有2种情况，分为 View 和 ViewGroup 。

+ View 主动获得焦点
+ ViewGroup 主动获得焦点

#### View 主动获得焦点
方法有如下几种：  

+ public final boolean requestFocus();
+ public final boolean requestFocus(int direction);
+ public boolean requestFocus(int direction, Rect previouslyFocusedRect);


以上方法最终都是调用一个私有的方法 requestFocusNoSearch 里面

```java
private boolean requestFocusNoSearch(int direction, Rect previouslyFocusedRect) {
        // need to be focusable
        if ((mViewFlags & FOCUSABLE_MASK) != FOCUSABLE ||
                (mViewFlags & VISIBILITY_MASK) != VISIBLE) {
            return false;
        }

        // need to be focusable in touch mode if in touch mode
        if (isInTouchMode() &&
            (FOCUSABLE_IN_TOUCH_MODE != (mViewFlags & FOCUSABLE_IN_TOUCH_MODE))) {
               return false;
        }

        // need to not have any parents blocking us
        if (hasAncestorThatBlocksDescendantFocus()) {
            return false;
        }

        handleFocusGainInternal(direction, previouslyFocusedRect);
        return true;
    }
```

上面的逻辑，去除了不能获得焦点和不是可见状态的 View ，以及触摸状态下，不可获得焦点的情景。  
hasAncestorThatBlocksDescendantFocus 方法是咨询 是否有父 ViewGroup 需要先于  
自己获得焦点。这个后面在 ViewGroup 分析时再讨论。  
上面代码，进入到 handleFocusGainInternal 方法里面，接着看。  

```java
void handleFocusGainInternal(@FocusRealDirection int direction, Rect previouslyFocusedRect) {
        if (DBG) {
            System.out.println(this + " requestFocus()");
        }

        if ((mPrivateFlags & PFLAG_FOCUSED) == 0) {
            mPrivateFlags |= PFLAG_FOCUSED;

            View oldFocus = (mAttachInfo != null) ? getRootView().findFocus() : null;

            if (mParent != null) {
                mParent.requestChildFocus(this, this);
            }

            if (mAttachInfo != null) {
                mAttachInfo.mTreeObserver.dispatchOnGlobalFocusChange(oldFocus, this);
            }

            onFocusChanged(true, direction, previouslyFocusedRect);
            refreshDrawableState();
        }
    }
```

上面的逻辑如下：  

+ 先改变焦点标志位
+ 调用父 ViewGroup 的 requestChildFocus 方法。
+ 告诉整个 View 二叉树坚挺着，焦点变化了。
+ 调用自身 onFocusChanged 方法。
+ 刷新自身状态变化。

上面唯一不清晰的逻辑是 **mParent.requestChildFocus(this, this);** 这里。  

```java
public void requestChildFocus(View child, View focused) {
        if (DBG) {
            System.out.println(this + " requestChildFocus()");
        }
        if (getDescendantFocusability() == FOCUS_BLOCK_DESCENDANTS) {
            return;
        }

        // Unfocus us, if necessary
        super.unFocus(focused);

        // We had a previous notion of who had focus. Clear it.
        if (mFocused != child) {
            if (mFocused != null) {
                mFocused.unFocus(focused);
            }

            mFocused = child;
        }
        if (mParent != null) {
            mParent.requestChildFocus(this, focused);
        }
    }
```

这里就是，

+ 先清除自己保存的之前获得焦点的 View ，并且保存新的。
+ 继续往上调用 mParent.requestChildFocus 方法，让整个 View 二叉树，对焦点的处理一致。  


#### ViewGroup 主动获得焦点
ViewGroup 只有一个方法 requestFocus：

```java
@Override
    public boolean requestFocus(int direction, Rect previouslyFocusedRect) {
        if (DBG) {
            System.out.println(this + " ViewGroup.requestFocus direction="
                    + direction);
        }
        int descendantFocusability = getDescendantFocusability();

        switch (descendantFocusability) {
            case FOCUS_BLOCK_DESCENDANTS:
                return super.requestFocus(direction, previouslyFocusedRect);
            case FOCUS_BEFORE_DESCENDANTS: {
                final boolean took = super.requestFocus(direction, previouslyFocusedRect);
                return took ? took : onRequestFocusInDescendants(direction, previouslyFocusedRect);
            }
            case FOCUS_AFTER_DESCENDANTS: {
                final boolean took = onRequestFocusInDescendants(direction, previouslyFocusedRect);
                return took ? took : super.requestFocus(direction, previouslyFocusedRect);
            }
            default:
                throw new IllegalStateException("descendant focusability must be "
                        + "one of FOCUS_BEFORE_DESCENDANTS, FOCUS_AFTER_DESCENDANTS, FOCUS_BLOCK_DESCENDANTS "
                        + "but is " + descendantFocusability);
        }
    }
```


descendantFocusability 代表3种模式，方法 onRequestFocusInDescendants ，代表给  
自己的子 View 来处理。

+ FOCUS_BLOCK_DESCENDANTS ，只允许自己处理，并且交给自己的父类（即 View）处理。
+ FOCUS_BEFORE_DESCENDANTS ，先尝试给自己的父类处理，如果不处理就交给子 Views 来处理。
+ FOCUS_AFTER_DESCENDANTS ,先让子 View是 处理，如果读没人处理了，再给自己的父类处理。

默认是 FOCUS_BEFORE_DESCENDANTS 。这个可以从 ViewGroup 的 initViewGroup 方法看出。

### 焦点的被动获得
这个涉及到整个事件的传递了，必须从 ViewRootImpl 里面找答案了。  
.....  
找到了，是在 ViewPostImeInputStage 的 processKeyEvent 方法里面。  

```java
private int processKeyEvent(QueuedInputEvent q) {
  // Deliver the key to the view hierarchy.
             if (mView.dispatchKeyEvent(event)) {
                 return FINISH_HANDLED;
             }
            删除了部分代码

             // Handle automatic focus changes.
             if (event.getAction() == KeyEvent.ACTION_DOWN) {
                 int direction = 0;
                 switch (event.getKeyCode()) {
                     case KeyEvent.KEYCODE_DPAD_LEFT:
                         if (event.hasNoModifiers()) {
                             direction = View.FOCUS_LEFT;
                         }
                         break;
                     case KeyEvent.KEYCODE_DPAD_RIGHT:
                         if (event.hasNoModifiers()) {
                             direction = View.FOCUS_RIGHT;
                         }
                         break;
                     case KeyEvent.KEYCODE_DPAD_UP:
                         if (event.hasNoModifiers()) {
                             direction = View.FOCUS_UP;
                         }
                         break;
                     case KeyEvent.KEYCODE_DPAD_DOWN:
                         if (event.hasNoModifiers()) {
                             direction = View.FOCUS_DOWN;
                         }
                         break;
                     case KeyEvent.KEYCODE_TAB:
                         if (event.hasNoModifiers()) {
                             direction = View.FOCUS_FORWARD;
                         } else if (event.hasModifiers(KeyEvent.META_SHIFT_ON)) {
                             direction = View.FOCUS_BACKWARD;
                         }
                         break;
                 }
                 if (direction != 0) {
                     View focused = mView.findFocus();
                     if (focused != null) {
                         View v = focused.focusSearch(direction);
                         if (v != null && v != focused) {
                             // do the math the get the interesting rect
                             // of previous focused into the coord system of
                             // newly focused view
                             focused.getFocusedRect(mTempRect);
                             if (mView instanceof ViewGroup) {
                                 ((ViewGroup) mView).offsetDescendantRectToMyCoords(
                                         focused, mTempRect);
                                 ((ViewGroup) mView).offsetRectIntoDescendantCoords(
                                         v, mTempRect);
                             }
                             if (v.requestFocus(direction, mTempRect)) {
                                 playSoundEffect(SoundEffectConstants
                                         .getContantForFocusDirection(direction));
                                 return FINISH_HANDLED;
                             }
                         }

                         // Give the focused view a last chance to handle the dpad key.
                         if (mView.dispatchUnhandledMove(focused, direction)) {
                             return FINISH_HANDLED;
                         }
                     } else {
                         // find the best view to give focus to in this non-touch-mode with no-focus
                         View v = focusSearch(null, direction);
                         if (v != null && v.requestFocus(direction)) {
                             return FINISH_HANDLED;
                         }
                     }
                 }
             }
             return FORWARD;

}
```

首先这个事件传递路线和一般的触摸事件分发类似，顺序如下：  

+ DecorView dispatchKeyEvent
+ Activity dispatchKeyEvent
+ PhoneWindow superDispatchKeyEvent
+ DecorView superDispatchKeyEvent
+ DecorView super.dispatchKeyEvent
+ ViewRootImpl  ViewPostImeInputStage processKeyEvent
+ DecorView dispatchUnhandledMove

如果没有人处理，最后还是传递回到 ViewPostImeInputStage 中。继续看方法 processKeyEvent    
把按键换算为方向，通过之前的有焦点的 View **focused.focusSearch(direction)** 。  
先来看 View 的 focusSearch 方法：  

 ```java
 public View focusSearch(@FocusRealDirection int direction) {
         if (mParent != null) {
             return mParent.focusSearch(this, direction);
         } else {
             return null;
         }
     }
 ```

 啥都没干，只能看 ViewGroup 的了。

 ```java
 public View focusSearch(View focused, int direction) {
         if (isRootNamespace()) {
             // root namespace means we should consider ourselves the top of the
             // tree for focus searching; otherwise we could be focus searching
             // into other tabs.  see LocalActivityManager and TabHost for more info
             return FocusFinder.getInstance().findNextFocus(this, focused, direction);
         } else if (mParent != null) {
             return mParent.focusSearch(focused, direction);
         }
         return null;
     }
 ```

 来到了 FocusFinder 的方法 findNextFocus 了。

 ```java
 public final View findNextFocus(ViewGroup root, View focused, int direction) {
        return findNextFocus(root, focused, null, direction);
    }

     private View findNextFocus(ViewGroup root, View focused, Rect focusedRect, int direction) {
         View next = null;
         if (focused != null) {
             next = findNextUserSpecifiedFocus(root, focused, direction);
         }
         if (next != null) {
             return next;
         }
         ArrayList<View> focusables = mTempList;
         try {
             focusables.clear();
             root.addFocusables(focusables, direction);
             if (!focusables.isEmpty()) {
                 next = findNextFocus(root, focused, focusedRect, direction, focusables);
             }
         } finally {
             focusables.clear();
         }
         return next;
     }

     private View findNextUserSpecifiedFocus(ViewGroup root, View focused, int direction) {
    // check for user specified next focus
    View userSetNextFocus = focused.findUserSetNextFocus(root, direction);
    if (userSetNextFocus != null && userSetNextFocus.isFocusable()
            && (!userSetNextFocus.isInTouchMode()
                    || userSetNextFocus.isFocusableInTouchMode())) {
        return userSetNextFocus;
    }
    return null;
}
 ```

上面代码，找下一个焦点 View 的时候，是通过 View.findUserSetNextFocus 方法，可以看出，是  
通过设置 mNextFocusLeftId 等 id 来实现的。

 ```java
 View findUserSetNextFocus(View root, @FocusDirection int direction) {
         switch (direction) {
             case FOCUS_LEFT:
                 if (mNextFocusLeftId == View.NO_ID) return null;
                 return findViewInsideOutShouldExist(root, mNextFocusLeftId);
             case FOCUS_RIGHT:
                 if (mNextFocusRightId == View.NO_ID) return null;
                 return findViewInsideOutShouldExist(root, mNextFocusRightId);
             case FOCUS_UP:
                 if (mNextFocusUpId == View.NO_ID) return null;
                 return findViewInsideOutShouldExist(root, mNextFocusUpId);
             case FOCUS_DOWN:
                 if (mNextFocusDownId == View.NO_ID) return null;
                 return findViewInsideOutShouldExist(root, mNextFocusDownId);
             case FOCUS_FORWARD:
                 if (mNextFocusForwardId == View.NO_ID) return null;
                 return findViewInsideOutShouldExist(root, mNextFocusForwardId);
             case FOCUS_BACKWARD: {
                 if (mID == View.NO_ID) return null;
                 final int id = mID;
                 return root.findViewByPredicateInsideOut(this, new Predicate<View>() {
                     @Override
                     public boolean apply(View t) {
                         return t.mNextFocusForwardId == id;
                     }
                 });
             }
         }
         return null;
     }
 ```

如果没有设置这些 id ，那么最后是走到 DecorView dispatchUnhandledMove 方法里面，其实就  
没有处理了。。当初还抱着 系统会根据位置默认给分给一个 View ，现在看下来系统根本没做这个处理。

## 焦点的清除
这个也分2种情况，被动和主动。

+ 被动的情况下就是当别人获得了焦点了，就会清除上一个获得焦点的 View 了，  
可以从上面的分析得知。是直接调用了 unFocus 方法。
+ 主动清除焦点也比较简单，直接调用 clearFocus 方法即可。
