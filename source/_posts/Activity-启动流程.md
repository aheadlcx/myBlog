title: Activity 启动流程
date: 2016-08-08 18:09:49
tags:
---

## 基本概念
客户端和服务端，Android 设计上是这样区分的，这2个是在不同的进程里面，下面分别用  
进程 C（client） 代指客户端，进程 S （server），代指 服务端。
## Activity 启动流程
### Activity 代码部分
都是通过 Activity 的 startActivityForResult 方法来实现的。

```java
public void startActivityForResult(Intent intent, int requestCode, @Nullable Bundle options) {
        if (mParent == null) {
            Instrumentation.ActivityResult ar =
                mInstrumentation.execStartActivity(
                    this, mMainThread.getApplicationThread(), mToken, this,
                    intent, requestCode, options);
            if (ar != null) {
                mMainThread.sendActivityResult(
                    mToken, mEmbeddedID, requestCode, ar.getResultCode(),
                    ar.getResultData());
            }
        }
        //.... ignore some code
    }
```

可以发现是通过 mInstrumentation 的 execStartActivity 方法来实现的。跟下去看看。而  
mMainThread 是 ActivityThread ，这个本质不是线程，没有继承 Thread ，但是他代表了这个  
进程的主线程，他里面有一个叫 H 的 Handler ，负责处理各种消息。

### mInstrumentation 部分

```java
public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
        IApplicationThread whoThread = (IApplicationThread) contextThread;
        if (mActivityMonitors != null) {
            synchronized (mSync) {
                final int N = mActivityMonitors.size();
                for (int i=0; i<N; i++) {
                    final ActivityMonitor am = mActivityMonitors.get(i);
                    if (am.match(who, null, intent)) {
                        am.mHits++;
                        if (am.isBlocking()) {
                            return requestCode >= 0 ? am.getResult() : null;
                        }
                        break;
                    }
                }
            }
        }
        try {
            intent.migrateExtraStreamToClipData();
            intent.prepareToLeaveProcess();
            int result = ActivityManagerNative.getDefault()
                .startActivity(whoThread, who.getBasePackageName(), intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, target != null ? target.mEmbeddedID : null,
                        requestCode, 0, null, options);
            checkStartActivityResult(result, intent);
        } catch (RemoteException e) {
        }
        return null;
    }
```

ActivityManagerNative.getDefault() 获得的是 ActivityManagerProxy 。这个东西的作用  
是 进程 C 单向的和 进程 S 通讯，即进程 C 发消息给进程 S。

### ActivityManagerProxy 部分
有2个问题需要关注  

+ ActivityManagerProxy 是如何创建的。
+ ActivityManagerProxy 是如何和 AMS 通信的。

#### 创建
这里是用了一个单例模式（ Singleton ），来确保这个 ActivityManagerProxy 是单例。然后  
再用这个 IBinder 来作为参数构造 ActivityManagerProxy ，并且把这个 IBinder 保存在  
ActivityManagerProxy 内部的成员变量中。**mark,unkown** ServiceManager 的工作  
原理，暂时不明白，先不追究细节。

```java
private static final Singleton<IActivityManager> gDefault = new Singleton<IActivityManager>() {
    protected IActivityManager create() {
        IBinder b = ServiceManager.getService("activity");
        if (false) {
            Log.v("ActivityManager", "default service binder = " + b);
        }
        IActivityManager am = asInterface(b);
        if (false) {
            Log.v("ActivityManager", "default service = " + am);
        }
        return am;
    }
};

```


```java
static public IActivityManager asInterface(IBinder obj) {
        if (obj == null) {
            return null;
        }
        IActivityManager in =
            (IActivityManager)obj.queryLocalInterface(descriptor);
        if (in != null) {
            return in;
        }

        return new ActivityManagerProxy(obj);
    }
```


#### ActivityManagerProxy 和 AMS 的单向通信
ActivityManagerNative.getDefault().startActivity() 方法，最终调用的是，  
ActivityManagerProxy 的方法。  
先来看看几个类的继承关系  

```java
class ActivityManagerProxy implements IActivityManager{}
public final class ActivityManagerService extends ActivityManagerNative{}
public abstract class ActivityManagerNative extends Binder   
implements IActivityManager{}
```

再来看看这个方法的源码  

```java
public int startActivity(IApplicationThread caller, String callingPackage, Intent intent,
            String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle options) throws RemoteException {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IActivityManager.descriptor);
        data.writeStrongBinder(caller != null ? caller.asBinder() : null);
        data.writeString(callingPackage);
        intent.writeToParcel(data, 0);
        data.writeString(resolvedType);
        data.writeStrongBinder(resultTo);
        data.writeString(resultWho);
        data.writeInt(requestCode);
        data.writeInt(startFlags);
        if (profilerInfo != null) {
            data.writeInt(1);
            profilerInfo.writeToParcel(data, Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
        } else {
            data.writeInt(0);
        }
        if (options != null) {
            data.writeInt(1);
            options.writeToParcel(data, 0);
        } else {
            data.writeInt(0);
        }
        mRemote.transact(START_ACTIVITY_TRANSACTION, data, reply, 0);
        reply.readException();
        int result = reply.readInt();
        reply.recycle();
        data.recycle();
        return result;
    }
```

ActivityManagerProxy 对于接口 IActivityManager 的方法实现，都是交给 Binder 来实现，
对于具体是调用的那个方法，可以看到 mRemote.transact(START_ACTIVITY_TRANSACTION);  
有这么一个标识。可以让服务端识别，到底调用那个方法。  
**mark unkown** Binder 的具体实现，暂时不是很理解。
## 疑问点
### Instrumentation
整个进程只会存在一个 Instrumentation 对象。他是什么实际构造的，以及怎么保证进程唯一的。

### Acitivity 合法性
怎样检查合法性的

### Application 的创建以及，创建之后怎么创建第一个 Activity

### ApplicationThread
这货本质是一个 Binder ，是什么时候创建出来的，由谁创建的，干什么用。

## 学习笔录

### Activity 整个启动流程

+ Activity 的 startActivity 方法
+ AMS 里面，包含 ActivityStackSupervisor 以及 ActivityStack 的一系列操作
+ 通过 Binder 驱动，回到 ApplicationThread 的 scheduleLaunchActivity 方法
+ 来到一个 ActivityThread 的成员对象 mH （类型为 H 的 handler）
+ AcitivityThread 的 handleLaunchActivity 方法
+ AcitivityThread 的 performLaunchActivity 方法
  + 通过 Instrumentation 创建 Activity 的实例
  + 看是否需要创建 Application
     - 如果需要创建，就先调用 Application 的 attach 方法
     - 再调用 Application 的 onCreate 方法
  + 调用 Activity 的 attach 方法
  + 调用 Activity 的 onCreate 方法

#### AcitivityThread 的 performLaunchActivity 方法
##### 创建 Activity
通过 Instrumentation 来创建。

```java
Activity activity = null;
        try {
            java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
            StrictMode.incrementExpectedActivityCount(activity.getClass());
            r.intent.setExtrasClassLoader(cl);
            r.intent.prepareToEnterProcess();
            if (r.state != null) {
                r.state.setClassLoader(cl);
            }
        } catch (Exception e) {
            if (!mInstrumentation.onException(activity, e)) {
                throw new RuntimeException(
                    "Unable to instantiate activity " + component
                    + ": " + e.toString(), e);
            }
        }
```

##### 通过 LoadedApk 看是否需要创建 Application
ActivityClientRecord 里面的 packageInfo 类型是 LoadedApk ，他包含了一个  
Application ，如果已经存在了，就直接返回，如果没有则创建一个(还是通过 mInstrumentation )。

```java
Application app = r.packageInfo.makeApplication(false, mInstrumentation);
```


```java
public Application makeApplication(boolean forceDefaultAppClass,
            Instrumentation instrumentation) {
        if (mApplication != null) {
            return mApplication;
        }

        Application app = null;

        String appClass = mApplicationInfo.className;
        if (forceDefaultAppClass || (appClass == null)) {
            appClass = "android.app.Application";
        }

        try {
            java.lang.ClassLoader cl = getClassLoader();
            if (!mPackageName.equals("android")) {
                initializeJavaContextClassLoader();
            }
            ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
            app = mActivityThread.mInstrumentation.newApplication(
                    cl, appClass, appContext);
            appContext.setOuterContext(app);
        } catch (Exception e) {
            if (!mActivityThread.mInstrumentation.onException(app, e)) {
                throw new RuntimeException(
                    "Unable to instantiate application " + appClass
                    + ": " + e.toString(), e);
            }
        }
        mActivityThread.mAllApplications.add(app);
        mApplication = app;

        if (instrumentation != null) {
            try {
                instrumentation.callApplicationOnCreate(app);
            } catch (Exception e) {
                if (!instrumentation.onException(app, e)) {
                    throw new RuntimeException(
                        "Unable to create application " + app.getClass().getName()
                        + ": " + e.toString(), e);
                }
            }
        }

        // Rewrite the R 'constants' for all library apks.
        SparseArray<String> packageIdentifiers = getAssets(mActivityThread)
                .getAssignedPackageIdentifiers();
        final int N = packageIdentifiers.size();
        for (int i = 0; i < N; i++) {
            final int id = packageIdentifiers.keyAt(i);
            if (id == 0x01 || id == 0x7f) {
                continue;
            }

            rewriteRValues(getClassLoader(), packageIdentifiers.valueAt(i), id);
        }

        return app;
    }
```

###### 在 Instrumentation 中 Application 的创建过程
结合 LoadedApk 的 makeApplication 方法，可以看出，application是先调用的 attach方法，  
再通过 instrumentation.callApplicationOnCreate(app);  ，来调用 application 的  
onCreate 方法.  



```java
static public Application newApplication(Class<?> clazz, Context context)
            throws InstantiationException, IllegalAccessException,
            ClassNotFoundException {
        Application app = (Application)clazz.newInstance();
        app.attach(context);
        return app;
    }
```


##### 创建 Context 并且先后调用 activity 的 attach 和 onCreate 方法
Acitivity 的 attach 方法会创建 Window （实际是 PhoneWindow ）。在 activity 的  
onCreate 方法中，setContentView ，仅仅是会让 window 初始化好 DecorView ，这个时候  
并没有把这个 DecorView 添加到 WindowManagerGlobal


```java
Context appContext = createBaseContextForActivity(r, activity);
                CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
                Configuration config = new Configuration(mCompatConfiguration);
                if (DEBUG_CONFIGURATION) Slog.v(TAG, "Launching activity "
                        + r.activityInfo.name + " with config " + config);
                activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor);
```

##### 在 ActivityThread 的 HandleResumeActivity 方法
先手调用 activity 的 onResume 和 makeVisible 方法，在 makeVisible 方法中，会让  
Windowmanager 的 addView 方法。这其实最后调用的是 WindowManagerGlobal 的  
addView 方法。在这里，创建 ViewRootImp 并且调用它的 setView 方法，这里会通过  
WindowmanagerService 来做处理。这里涉及到 Binder 驱动。还是没明白，怎样才算真正  
添加到屏幕了，这个 View 。

### Activity 的 View 显示流程
一个 View 的显示，需要 WindowManager.addView 之后才可以显示的。  
View 显示的流程如下：  

+ 1. Window 的创建，在 Activity 的 attach 方法中
+ 2. DecorView 的创建，我们一般会在 Activity 的 onCreate 方法中会调用 setContentView  
方法，在这个方法中，会调用 PhoneWindow 的 setContentView 方法。
+ 3. View 的最终显示，是在 Activity 的 makeVisible 方法中，该方法会调用 WindowManager  
的 addView 方法，该 WindowManager 实现者是 WindowManagerImpl ，在其 addView 方法里面  
调用的是 WindowManagerGlobal 的 addView 方法
  + 1. 创建 ViewRootImpl
  + 2. 调用 ViewRootImpl 的 setView 方法
  + 3. 在 ViewRootImpl 的 setView 方法里面，会调用一个 IWindowSession ( 实际实现是  
    Session ) 的 addToDisplay 方法，完成 Window 的添加

# 遗留疑问
+ Activity 的 View 什么时候被销毁。销毁逻辑代码在何处。 added 2016.12.10
