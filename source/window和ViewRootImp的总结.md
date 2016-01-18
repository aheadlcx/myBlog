title: Window和ViewRootImp的总结
date: 2016-01-14 14:18:48
tags: [android, Window, ViewRootImp, PhoneWindow]
categories: android
---
本文是View绘制和事件分发的知识整理收尾篇，顺便也简单的整理了一下window相关的知识。
<!--more  -->

>本文参考了众多前辈的文章和书籍
>[任玉刚的相关文章和书籍](http://blog.csdn.net/singwhatiwanna)
>[郭霖博客](http://blog.csdn.net/guolin_blog)


# activity的View 视图创建。
我们先来看看activity的 setContentView 方法

```java
public void setContentView(@LayoutRes int layoutResID) {
        getWindow().setContentView(layoutResID);
        initWindowDecorActionBar();
    }
```

这个getWindow 拿到的window对象，对于activity来说就是 PhoneWindow （这点，目前是通过任玉刚的数据，android开发艺术，初步确定的，  
从PhoneWindow代码以及注解来看，也侧面认证了这点），那我们接着来看看 PhoneWindow 的 setContentView 方法   


```java
@Override
    public void setContentView(int layoutResID) {
        if (mContentParent == null) {
            installDecor();
        } else {
            mContentParent.removeAllViews();
        }
        mLayoutInflater.inflate(layoutResID, mContentParent);
        final Callback cb = getCallback();
        if (cb != null) {
            cb.onContentChanged();
        }
    }
```

mContentParent == null，就调用 installDecor 方法，那我们继续看下面的方法。


```java
private void installDecor() {
        if (mDecor == null) {
            mDecor = generateDecor();
            mDecor.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
            mDecor.setIsRootNamespace(true);
        }
        if (mContentParent == null) {
            mContentParent = generateLayout(mDecor);

            mTitleView = (TextView)findViewById(com.android.internal.R.id.title);
            if (mTitleView != null) {
                if ((getLocalFeatures() & (1 << FEATURE_NO_TITLE)) != 0) {
                    View titleContainer = findViewById(com.android.internal.R.id.title_container);
                    if (titleContainer != null) {
                        titleContainer.setVisibility(View.GONE);
                    } else {
                        mTitleView.setVisibility(View.GONE);
                    }
                    if (mContentParent instanceof FrameLayout) {
                        ((FrameLayout)mContentParent).setForeground(null);
                    }
                } else {
                    mTitleView.setText(mTitle);
                }
            }
        }
    }
```

mDecor == null，就 generateDecor（） 方法，继续跟源码看下去。


```java
protected DecorView generateDecor() {
        return new DecorView(getContext(), -1);
    }
```


就是创建了一个 DecorView 对象，可惜留个心眼，这里为什么写死了一个-1啊，继续跟 DecorView 看下去，源码较长，仅截取需要的代码看看。

```java
private final class DecorView extends FrameLayout {
       /* package */int mDefaultOpacity = PixelFormat.OPAQUE;

       /** The feature ID of the panel, or -1 if this is the application's DecorView */
       private final int mFeatureId;

       private final Rect mDrawingBounds = new Rect();

       private final Rect mBackgroundPadding = new Rect();

       private final Rect mFramePadding = new Rect();

       private final Rect mFrameOffsets = new Rect();

       private boolean mChanging;

       private Drawable mMenuBackground;
       private boolean mWatchingForMenu;
       private int mDownY;

       public DecorView(Context context, int featureId) {
           super(context);
           mFeatureId = featureId;
       }
       @Override
       public boolean dispatchTouchEvent(MotionEvent ev) {
           final Callback cb = getCallback();
           return cb != null && mFeatureId < 0 ? cb.dispatchTouchEvent(ev) : super
                   .dispatchTouchEvent(ev);
       }

       @Override
       public boolean dispatchTrackballEvent(MotionEvent ev) {
           final Callback cb = getCallback();
           return cb != null && mFeatureId < 0 ? cb.dispatchTrackballEvent(ev) : super
                   .dispatchTrackballEvent(ev);
       }

       public boolean superDispatchKeyEvent(KeyEvent event) {
           return super.dispatchKeyEvent(event);
       }

       public boolean superDispatchTouchEvent(MotionEvent event) {
           return super.dispatchTouchEvent(event);
       }

       public boolean superDispatchTrackballEvent(MotionEvent event) {
           return super.dispatchTrackballEvent(event);
       }

       @Override
       public boolean onTouchEvent(MotionEvent event) {
           return onInterceptTouchEvent(event);
       }

   }

```

这个 DecorView 其实就是个FrameLayout ，不过他多了好多事件分发相关的 方法。暂时不管。  
说完 generateDecor  方法，我们接下来再看看 mContentParent = generateLayout(mDecor);  
同样的，我们删除部分和我们主流程无关的代码，方便查看。  

```java
protected ViewGroup generateLayout(DecorView decor) {
        // Apply data from current theme.
        TypedArray a = getWindowStyle();

        if (false) {
            System.out.println("From style:");
            String s = "Attrs:";
            for (int i = 0; i < com.android.internal.R.styleable.Window.length; i++) {
                s = s + " " + Integer.toHexString(com.android.internal.R.styleable.Window[i]) + "="
                        + a.getString(i);
            }
            System.out.println(s);
        }

        mIsFloating = a.getBoolean(com.android.internal.R.styleable.Window_windowIsFloating, false);
        int flagsToUpdate = (FLAG_LAYOUT_IN_SCREEN|FLAG_LAYOUT_INSET_DECOR)
                & (~getForcedWindowFlags());
        if (mIsFloating) {
            setLayout(WRAP_CONTENT, WRAP_CONTENT);
            setFlags(0, flagsToUpdate);
        } else {
            setFlags(FLAG_LAYOUT_IN_SCREEN|FLAG_LAYOUT_INSET_DECOR, flagsToUpdate);
        }

        if (a.getBoolean(com.android.internal.R.styleable.Window_windowNoTitle, false)) {
            requestFeature(FEATURE_NO_TITLE);
        }

        if (a.getBoolean(com.android.internal.R.styleable.Window_windowFullscreen, false)) {
            setFlags(FLAG_FULLSCREEN, FLAG_FULLSCREEN&(~getForcedWindowFlags()));
        }

        if (a.getBoolean(com.android.internal.R.styleable.Window_windowShowWallpaper, false)) {
            setFlags(FLAG_SHOW_WALLPAPER, FLAG_SHOW_WALLPAPER&(~getForcedWindowFlags()));
        }

        WindowManager.LayoutParams params = getAttributes();

        if (!hasSoftInputMode()) {
            params.softInputMode = a.getInt(
                    com.android.internal.R.styleable.Window_windowSoftInputMode,
                    params.softInputMode);
        }

        if (a.getBoolean(com.android.internal.R.styleable.Window_backgroundDimEnabled,
                mIsFloating)) {
            /* All dialogs should have the window dimmed */
            if ((getForcedWindowFlags()&WindowManager.LayoutParams.FLAG_DIM_BEHIND) == 0) {
                params.flags |= WindowManager.LayoutParams.FLAG_DIM_BEHIND;
            }
            params.dimAmount = a.getFloat(
                    android.R.styleable.Window_backgroundDimAmount, 0.5f);
        }

        if (params.windowAnimations == 0) {
            params.windowAnimations = a.getResourceId(
                    com.android.internal.R.styleable.Window_windowAnimationStyle, 0);
        }

        // The rest are only done if this window is not embedded; otherwise,
        // the values are inherited from our container.
        if (getContainer() == null) {
            if (mBackgroundDrawable == null) {
                if (mBackgroundResource == 0) {
                    mBackgroundResource = a.getResourceId(
                            com.android.internal.R.styleable.Window_windowBackground, 0);
                }
                if (mFrameResource == 0) {
                    mFrameResource = a.getResourceId(com.android.internal.R.styleable.Window_windowFrame, 0);
                }
                if (false) {
                    System.out.println("Background: "
                            + Integer.toHexString(mBackgroundResource) + " Frame: "
                            + Integer.toHexString(mFrameResource));
                }
            }
            mTextColor = a.getColor(com.android.internal.R.styleable.Window_textColor, 0xFF000000);
        }

        // Inflate the window decor.

        int layoutResource;
        int features = getLocalFeatures();
        // System.out.println("Features: 0x" + Integer.toHexString(features));
        if ((features & ((1 << FEATURE_LEFT_ICON) | (1 << FEATURE_RIGHT_ICON))) != 0) {
            if (mIsFloating) {
                layoutResource = com.android.internal.R.layout.dialog_title_icons;
            } else {
                layoutResource = com.android.internal.R.layout.screen_title_icons;
            }
            // System.out.println("Title Icons!");
        } else if ((features & ((1 << FEATURE_PROGRESS) | (1 << FEATURE_INDETERMINATE_PROGRESS))) != 0) {
            // Special case for a window with only a progress bar (and title).
            // XXX Need to have a no-title version of embedded windows.
            layoutResource = com.android.internal.R.layout.screen_progress;
            // System.out.println("Progress!");
        } else if ((features & (1 << FEATURE_CUSTOM_TITLE)) != 0) {
            // Special case for a window with a custom title.
            // If the window is floating, we need a dialog layout
            if (mIsFloating) {
                layoutResource = com.android.internal.R.layout.dialog_custom_title;
            } else {
                layoutResource = com.android.internal.R.layout.screen_custom_title;
            }
        } else if ((features & (1 << FEATURE_NO_TITLE)) == 0) {
            // If no other features and not embedded, only need a title.
            // If the window is floating, we need a dialog layout
            if (mIsFloating) {
                layoutResource = com.android.internal.R.layout.dialog_title;
            } else {
                layoutResource = com.android.internal.R.layout.screen_title;
            }
            // System.out.println("Title!");
        } else {
            // Embedded, so no decoration is needed.
            layoutResource = com.android.internal.R.layout.screen_simple;
            // System.out.println("Simple!");
        }

        mDecor.startChanging();
        //主要关键代码
        View in = mLayoutInflater.inflate(layoutResource, null);
        decor.addView(in, new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));

        ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
        if (contentParent == null) {
            throw new RuntimeException("Window couldn't find content container view");
        }

        if ((features & (1 << FEATURE_INDETERMINATE_PROGRESS)) != 0) {
            ProgressBar progress = getCircularProgressBar(false);
            if (progress != null) {
                progress.setIndeterminate(true);
            }
        }

        // Remaining setup -- of background and title -- that only applies
        // to top-level windows.
        if (getContainer() == null) {
            Drawable drawable = mBackgroundDrawable;
            if (mBackgroundResource != 0) {
                drawable = getContext().getResources().getDrawable(mBackgroundResource);
            }
            mDecor.setWindowBackground(drawable);
            drawable = null;
            if (mFrameResource != 0) {
                drawable = getContext().getResources().getDrawable(mFrameResource);
            }
            mDecor.setWindowFrame(drawable);

            // System.out.println("Text=" + Integer.toHexString(mTextColor) +
            // " Sel=" + Integer.toHexString(mTextSelectedColor) +
            // " Title=" + Integer.toHexString(mTitleColor));

            if (mTitleColor == 0) {
                mTitleColor = mTextColor;
            }

            if (mTitle != null) {
                setTitle(mTitle);
            }
            setTitleColor(mTitleColor);
        }

        mDecor.finishChanging();

        return contentParent;
    }
```

这个方法里面主要的逻辑就是，加载 layoutResource 对应的view in 出来，然后给 decor , addView 上去。  
再从view in 中，找出 id 是 ID_ANDROID_CONTENT 的 ViewGroup contentParent 出来，校验下是否为空，再返回。  
其中对于activity 来说，layoutResource，就是 com.android.internal.R.layout.screen_simple。  
我们看看代码，  

```java
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:fitsSystemWindows="true"
    android:orientation="vertical">
    <ViewStub android:id="@+id/action_mode_bar_stub"
              android:inflatedId="@+id/action_mode_bar"
              android:layout="@layout/action_mode_bar"
              android:layout_width="match_parent"
              android:layout_height="wrap_content"
              android:theme="?attr/actionBarTheme" />
    <FrameLayout
         android:id="@android:id/content"
         android:layout_width="match_parent"
         android:layout_height="match_parent"
         android:foregroundInsidePadding="false"
         android:foregroundGravity="fill_horizontal|top"
         android:foreground="?android:attr/windowContentOverlay" />
</LinearLayout>
```

从上面可以看出，我们要找的 contentParent ，就是 id 为content的view，也是一个FrameLayout。  
现在开始，我们activity 的view二叉树，已经完成加载出来了，注意这个时候，还是没显示出来的。  
**要理解的话，需要继续看window和ViewRootImp了**


# activity 的绑定 （ attach方法 ）
对于 activity 的创建流程，比较复杂，这里暂时不作详细描述。接下来我们看看 attach 方法

```java
final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config, String referrer, IVoiceInteractor voiceInteractor) {
        attachBaseContext(context);

        mFragments.attachHost(null /*parent*/);

        mWindow = new PhoneWindow(this);
        mWindow.setCallback(this);
        mWindow.setOnWindowDismissedCallback(this);
        mWindow.getLayoutInflater().setPrivateFactory(this);
        if (info.softInputMode != WindowManager.LayoutParams.SOFT_INPUT_STATE_UNSPECIFIED) {
            mWindow.setSoftInputMode(info.softInputMode);
        }
        if (info.uiOptions != 0) {
            mWindow.setUiOptions(info.uiOptions);
        }
        mUiThread = Thread.currentThread();

        mMainThread = aThread;
        mInstrumentation = instr;
        mToken = token;
        mIdent = ident;
        mApplication = application;
        mIntent = intent;
        mReferrer = referrer;
        mComponent = intent.getComponent();
        mActivityInfo = info;
        mTitle = title;
        mParent = parent;
        mEmbeddedID = id;
        mLastNonConfigurationInstances = lastNonConfigurationInstances;
        if (voiceInteractor != null) {
            if (lastNonConfigurationInstances != null) {
                mVoiceInteractor = lastNonConfigurationInstances.voiceInteractor;
            } else {
                mVoiceInteractor = new VoiceInteractor(voiceInteractor, this, this,
                        Looper.myLooper());
            }
        }

        mWindow.setWindowManager(
                (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
                mToken, mComponent.flattenToString(),
                (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
        if (mParent != null) {
            mWindow.setContainer(mParent.getWindow());
        }
        mWindowManager = mWindow.getWindowManager();
        mCurrentConfig = config;
    }
```

我们注意到其中2行代码，
mWindow = new PhoneWindow(this);  
mWindow.setCallback(this);  
在这里，我们终于可以确定，对于activity 来说，window 就是 PhoneWindow 了。但是下面的 setCallBack  
这个CallBack，是Window 里面的接口，是一系列的事件回调。this，就是指activity。activity源码，果然是实现了  
Window.Callback。  
这个CallBack 有一些我们很常见的回调方法名称，  

* public boolean dispatchKeyEvent(KeyEvent event);
* public boolean dispatchTouchEvent(MotionEvent event);

这么看，难道activity的事件分发，是由window回调过来的？暂时放下吧。**问题一**

# activity 的 makeVisible 方法。
在这个过程中，activity的view，才真正开始显示出来。我们先看看源码。

```java
void makeVisible() {
        if (!mWindowAdded) {
            ViewManager wm = getWindowManager();
            wm.addView(mDecor, getWindow().getAttributes());
            mWindowAdded = true;
        }
        mDecor.setVisibility(View.VISIBLE);
    }
```

 getWindowManager 拿到的 ViewManager 实际是 WindowManagerImp 。那接下来，我们看看  
WindowManagerImp 的addView 方法。  

```java
@Override
   public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
       applyDefaultToken(params);
       mGlobal.addView(view, params, mDisplay, mParentWindow);
   }
```

我们先看看 applyDefaultToken 方法，这个就是赋值了token，这个token，不是字符串，而是一个IBinder对象，  
就是用来区分window的吧，暂时没仔细看这点。  

```java
/**
        * Identifier for this window.  This will usually be filled in for
        * you.
        */
       public IBinder token = null;
```


```java
private void applyDefaultToken(@NonNull ViewGroup.LayoutParams params) {
    // Only use the default token if we don't have a parent window.
    if (mDefaultToken != null && mParentWindow == null) {
        if (!(params instanceof WindowManager.LayoutParams)) {
            throw new IllegalArgumentException("Params must be WindowManager.LayoutParams");
        }

        // Only use the default token if we don't already have a token.
        final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams) params;
        if (wparams.token == null) {
            wparams.token = mDefaultToken;
        }
    }
}

```

我们继续看  
mGlobal.addView(view, params, mDisplay, mParentWindow);  
mGlobal 其实是 WindowManagerGlobal 。那我们跟进去，看看addView源码  

```java
public void addView(View view, ViewGroup.LayoutParams params,
                        Display display, Window parentWindow) {
        if (view == null) {
            throw new IllegalArgumentException("view must not be null");
        }
        if (display == null) {
            throw new IllegalArgumentException("display must not be null");
        }
        if (!(params instanceof WindowManager.LayoutParams)) {
            throw new IllegalArgumentException("Params must be WindowManager.LayoutParams");
        }

        final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams) params;
        if (parentWindow != null) {
            parentWindow.adjustLayoutParamsForSubWindow(wparams);
        } else {
            // If there's no parent, then hardware acceleration for this view is
            // set from the application's hardware acceleration setting.
            final Context context = view.getContext();
            if (context != null
                    && (context.getApplicationInfo().flags
                    & ApplicationInfo.FLAG_HARDWARE_ACCELERATED) != 0) {
                wparams.flags |= WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED;
            }
        }

        ViewRootImpl root;
        View panelParentView = null;

        synchronized (mLock) {
            // Start watching for system property changes.
            if (mSystemPropertyUpdater == null) {
                mSystemPropertyUpdater = new Runnable() {
                    @Override public void run() {
                        synchronized (mLock) {
                            for (int i = mRoots.size() - 1; i >= 0; --i) {
                                mRoots.get(i).loadSystemProperties();
                            }
                        }
                    }
                };
                SystemProperties.addChangeCallback(mSystemPropertyUpdater);
            }

            int index = findViewLocked(view, false);
            if (index >= 0) {
                if (mDyingViews.contains(view)) {
                    // Don't wait for MSG_DIE to make it's way through root's queue.
                    mRoots.get(index).doDie();
                } else {
                    throw new IllegalStateException("View " + view
                            + " has already been added to the window manager.");
                }
                // The previous removeView() had not completed executing. Now it has.
            }

            // If this is a panel window, then find the window it is being
            // attached to for future reference.
            if (wparams.type >= WindowManager.LayoutParams.FIRST_SUB_WINDOW &&
                    wparams.type <= WindowManager.LayoutParams.LAST_SUB_WINDOW) {
                final int count = mViews.size();
                for (int i = 0; i < count; i++) {
                    if (mRoots.get(i).mWindow.asBinder() == wparams.token) {
                        panelParentView = mViews.get(i);
                    }
                }
            }

            root = new ViewRootImpl(view.getContext(), display);

            view.setLayoutParams(wparams);

            mViews.add(view);
            mRoots.add(root);
            mParams.add(wparams);
        }

        // do this last because it fires off messages to start doing things
        try {
            root.setView(view, wparams, panelParentView);
        } catch (RuntimeException e) {
            // BadTokenException or InvalidDisplayException, clean up.
            synchronized (mLock) {
                final int index = findViewLocked(view, false);
                if (index >= 0) {
                    removeViewLocked(index, true);
                }
            }
            throw e;
        }
    }

```

这里面，有几行重要的代码，需要我们仔细看看的。
1. root = new ViewRootImpl(view.getContext(), display);
创建了一个 ViewRootImpl 对象，这个对象，到底是干嘛的，我们留到下面再看。
2. mViews.add(view); mRoots.add(root); mParams.add(wparams);  
这分别保留下来：
* view就是我们在 activity 中传过来的 decorView。  
* root 就是我们刚刚创建的 ViewRootImpl 对象。  
* wparams 其实是 WindowManager.LayoutParams 对象。该对象在创建的时候，
宽高都是默认是LayoutParams.MATCH_PARENT, LayoutParams.MATCH_PARENT。
3. root.setView(view, wparams, panelParentView);
这个方法，里面的实现，我们看下一节。

# ViewRootImpl

```java
 我们先来看看类的注解

/**
 * The top of a view hierarchy, implementing the needed protocol between View
 * and the WindowManager.  This is for the most part an internal implementation
 * detail of {@link WindowManagerGlobal}.
 *
 * {@hide}
 */
```

大概意思就是，这是view 二叉树的顶级。连接了 View 和 WindowManager 之间的协议。他是大部分 WindowManagerGlobal  
中的内部实现细节。

 ```java
 /**
      * We have one child
      */
     public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
         synchronized (this) {
             if (mView == null) {
                 mView = view;

                 mAttachInfo.mDisplayState = mDisplay.getState();
                 mDisplayManager.registerDisplayListener(mDisplayListener, mHandler);

                 mViewLayoutDirectionInitial = mView.getRawLayoutDirection();
                 mFallbackEventHandler.setView(view);
                 mWindowAttributes.copyFrom(attrs);
                 if (mWindowAttributes.packageName == null) {
                     mWindowAttributes.packageName = mBasePackageName;
                 }
                 attrs = mWindowAttributes;
                 // Keep track of the actual window flags supplied by the client.
                 mClientWindowLayoutFlags = attrs.flags;

                 setAccessibilityFocus(null, null);

                 if (view instanceof RootViewSurfaceTaker) {
                     mSurfaceHolderCallback =
                             ((RootViewSurfaceTaker)view).willYouTakeTheSurface();
                     if (mSurfaceHolderCallback != null) {
                         mSurfaceHolder = new TakenSurfaceHolder();
                         mSurfaceHolder.setFormat(PixelFormat.UNKNOWN);
                     }
                 }

                 // Compute surface insets required to draw at specified Z value.
                 // TODO: Use real shadow insets for a constant max Z.
                 if (!attrs.hasManualSurfaceInsets) {
                     final int surfaceInset = (int) Math.ceil(view.getZ() * 2);
                     attrs.surfaceInsets.set(surfaceInset, surfaceInset, surfaceInset, surfaceInset);
                 }

                 CompatibilityInfo compatibilityInfo = mDisplayAdjustments.getCompatibilityInfo();
                 mTranslator = compatibilityInfo.getTranslator();

                 // If the application owns the surface, don't enable hardware acceleration
                 if (mSurfaceHolder == null) {
                     enableHardwareAcceleration(attrs);
                 }

                 boolean restore = false;
                 if (mTranslator != null) {
                     mSurface.setCompatibilityTranslator(mTranslator);
                     restore = true;
                     attrs.backup();
                     mTranslator.translateWindowLayout(attrs);
                 }
                 if (DEBUG_LAYOUT) Log.d(TAG, "WindowLayout in setView:" + attrs);

                 if (!compatibilityInfo.supportsScreen()) {
                     attrs.privateFlags |= WindowManager.LayoutParams.PRIVATE_FLAG_COMPATIBLE_WINDOW;
                     mLastInCompatMode = true;
                 }

                 mSoftInputMode = attrs.softInputMode;
                 mWindowAttributesChanged = true;
                 mWindowAttributesChangesFlag = WindowManager.LayoutParams.EVERYTHING_CHANGED;
                 mAttachInfo.mRootView = view;
                 mAttachInfo.mScalingRequired = mTranslator != null;
                 mAttachInfo.mApplicationScale =
                         mTranslator == null ? 1.0f : mTranslator.applicationScale;
                 if (panelParentView != null) {
                     mAttachInfo.mPanelParentWindowToken
                             = panelParentView.getApplicationWindowToken();
                 }
                 mAdded = true;
                 int res; /* = WindowManagerImpl.ADD_OKAY; */

                 // Schedule the first layout -before- adding to the window
                 // manager, to make sure we do the relayout before receiving
                 // any other events from the system.
                 requestLayout();
                 if ((mWindowAttributes.inputFeatures
                         & WindowManager.LayoutParams.INPUT_FEATURE_NO_INPUT_CHANNEL) == 0) {
                     mInputChannel = new InputChannel();
                 }
                 try {
                     mOrigWindowType = mWindowAttributes.type;
                     mAttachInfo.mRecomputeGlobalAttributes = true;
                     collectViewAttributes();
                     res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                             getHostVisibility(), mDisplay.getDisplayId(),
                             mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                             mAttachInfo.mOutsets, mInputChannel);
                 } catch (RemoteException e) {
                     mAdded = false;
                     mView = null;
                     mAttachInfo.mRootView = null;
                     mInputChannel = null;
                     mFallbackEventHandler.setView(null);
                     unscheduleTraversals();
                     setAccessibilityFocus(null, null);
                     throw new RuntimeException("Adding window failed", e);
                 } finally {
                     if (restore) {
                         attrs.restore();
                     }
                 }

                 if (mTranslator != null) {
                     mTranslator.translateRectInScreenToAppWindow(mAttachInfo.mContentInsets);
                 }
                 mPendingOverscanInsets.set(0, 0, 0, 0);
                 mPendingContentInsets.set(mAttachInfo.mContentInsets);
                 mPendingStableInsets.set(mAttachInfo.mStableInsets);
                 mPendingVisibleInsets.set(0, 0, 0, 0);
                 if (DEBUG_LAYOUT) Log.v(TAG, "Added window " + mWindow);
                 if (res < WindowManagerGlobal.ADD_OKAY) {
                     mAttachInfo.mRootView = null;
                     mAdded = false;
                     mFallbackEventHandler.setView(null);
                     unscheduleTraversals();
                     setAccessibilityFocus(null, null);
                     switch (res) {
                         case WindowManagerGlobal.ADD_BAD_APP_TOKEN:
                         case WindowManagerGlobal.ADD_BAD_SUBWINDOW_TOKEN:
                             throw new WindowManager.BadTokenException(
                                     "Unable to add window -- token " + attrs.token
                                             + " is not valid; is your activity running?");
                         case WindowManagerGlobal.ADD_NOT_APP_TOKEN:
                             throw new WindowManager.BadTokenException(
                                     "Unable to add window -- token " + attrs.token
                                             + " is not for an application");
                         case WindowManagerGlobal.ADD_APP_EXITING:
                             throw new WindowManager.BadTokenException(
                                     "Unable to add window -- app for token " + attrs.token
                                             + " is exiting");
                         case WindowManagerGlobal.ADD_DUPLICATE_ADD:
                             throw new WindowManager.BadTokenException(
                                     "Unable to add window -- window " + mWindow
                                             + " has already been added");
                         case WindowManagerGlobal.ADD_STARTING_NOT_NEEDED:
                             // Silently ignore -- we would have just removed it
                             // right away, anyway.
                             return;
                         case WindowManagerGlobal.ADD_MULTIPLE_SINGLETON:
                             throw new WindowManager.BadTokenException(
                                     "Unable to add window " + mWindow +
                                             " -- another window of this type already exists");
                         case WindowManagerGlobal.ADD_PERMISSION_DENIED:
                             throw new WindowManager.BadTokenException(
                                     "Unable to add window " + mWindow +
                                             " -- permission denied for this window type");
                         case WindowManagerGlobal.ADD_INVALID_DISPLAY:
                             throw new WindowManager.InvalidDisplayException(
                                     "Unable to add window " + mWindow +
                                             " -- the specified display can not be found");
                         case WindowManagerGlobal.ADD_INVALID_TYPE:
                             throw new WindowManager.InvalidDisplayException(
                                     "Unable to add window " + mWindow
                                             + " -- the specified window type is not valid");
                     }
                     throw new RuntimeException(
                             "Unable to add window -- unknown error code " + res);
                 }

                 if (view instanceof RootViewSurfaceTaker) {
                     mInputQueueCallback =
                             ((RootViewSurfaceTaker)view).willYouTakeTheInputQueue();
                 }
                 if (mInputChannel != null) {
                     if (mInputQueueCallback != null) {
                         mInputQueue = new InputQueue();
                         mInputQueueCallback.onInputQueueCreated(mInputQueue);
                     }
                     mInputEventReceiver = new WindowInputEventReceiver(mInputChannel,
                             Looper.myLooper());
                 }

                 view.assignParent(this);
                 mAddedTouchMode = (res & WindowManagerGlobal.ADD_FLAG_IN_TOUCH_MODE) != 0;
                 mAppVisible = (res & WindowManagerGlobal.ADD_FLAG_APP_VISIBLE) != 0;

                 if (mAccessibilityManager.isEnabled()) {
                     mAccessibilityInteractionConnectionManager.ensureConnection();
                 }

                 if (view.getImportantForAccessibility() == View.IMPORTANT_FOR_ACCESSIBILITY_AUTO) {
                     view.setImportantForAccessibility(View.IMPORTANT_FOR_ACCESSIBILITY_YES);
                 }

                 // Set up the input pipeline.
                 CharSequence counterSuffix = attrs.getTitle();
                 mSyntheticInputStage = new SyntheticInputStage();
                 InputStage viewPostImeStage = new ViewPostImeInputStage(mSyntheticInputStage);
                 InputStage nativePostImeStage = new NativePostImeInputStage(viewPostImeStage,
                         "aq:native-post-ime:" + counterSuffix);
                 InputStage earlyPostImeStage = new EarlyPostImeInputStage(nativePostImeStage);
                 InputStage imeStage = new ImeInputStage(earlyPostImeStage,
                         "aq:ime:" + counterSuffix);
                 InputStage viewPreImeStage = new ViewPreImeInputStage(imeStage);
                 InputStage nativePreImeStage = new NativePreImeInputStage(viewPreImeStage,
                         "aq:native-pre-ime:" + counterSuffix);

                 mFirstInputStage = nativePreImeStage;
                 mFirstPostImeInputStage = earlyPostImeStage;
                 mPendingInputEventQueueLengthCounterName = "aq:pending:" + counterSuffix;
             }
         }
     }

 ```
