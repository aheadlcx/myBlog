title: Activity 启动以及插件化知识点 Note
date: 2017-03-02 19:57:10
tags:
---


1. activity 生命周期管理的 token
2. acitivty 和 application 的 context 是什么时候注入的，以及实现分别是什么。
3. activitythread 和 AMS ,通讯，是怎么实现的。
4. servicemanager 的服务 是什么时候注入的。 **[undo]**
5. hook ,静态的，和实例的，分别如何实现。
6. 测试文件锁，是否可以跨进程锁。参考 MultiDex 的文件锁， master 分支有。**[undo]**
7. APP 进程启动的时候，ActivityManagerNative 的 asInterface 方法，是什么时候触发的。
8. APP 进程入口在那里。
9. ResourcesManager 和 AssetManager 有什么区别和联系

## 1. activity 生命周期管理
Activity里面有一个成员变量mToken代表的就是它，token可以唯一地标识一个Activity对象，  
它在Activity的attach方法里面初始化，是一个 IBinder 对象。  

```java
final ArrayMap<IBinder, ActivityClientRecord> mActivities = new ArrayMap<>();
创建 Activity 的时候，会传递进去.
mActivities.put(r.token, r);

在进行其他生命周期的时候，则是( performResumeActivity 举例 )
ActivityClientRecord r = mActivities.get(token);
r.activity.performResume();

```

因此在插件化开发中，通过占坑的方式实现，动态加载 Activity ，送给 AMS 的是占坑的 Activity ，  
但是在 ActivityThread 的回调中，操作的是，目标的 Activity ，因此生命周期得以实现。


传入的值是 ActivityClientRecord.token . 而这个值，又是从何而来的呢。可以从调用链追  
ApplicationThread 是 IBinder ，并且实现了 IApplicationThread 接口， 追查得知  
远端调用是 ApplicationThreadNative .  

```java
ApplicationThread 的 scheduleLaunchActivity(IBinder token) ->  
ApplicationThreadNative 的 scheduleLaunchActivity (IBinder token) ->  
ActivityStackSupervisor 的 ealStartActivityLocked （ActivityRecord r）->  
ActivityStackSupervisor 的 attachApplicationLocked (ProcessRecord app)
ActivityRecord hr = stack.topRunningActivityLocked();
```

看到这里，可以直接看 ActivityRecord 的构造方法了，不再跟 AMS 源码调用链了，我们这里的目的是  
想知道 Activity 里面的 token 是怎么构造的，而不是 AMS 本身的逻辑。  

```java
简化版
ActivityRecord(ActivityManagerService _service){
  service = _service;
  appToken = new Token(this, service);
}
```

## 2, Activity 和 Application 的 context
Activity 的 context 的 ContextImpl . 在 ActivityThread 的 performLaunchActivity  
方法中，创建.( createBaseContextForActivity(r,activity) ) ，然后通过 activity 的  
attach 方法传入。


Application 同样式 ContextImpl ，创建和传入时机是：  
ActivityThread( performLaunchActivity ) -> LoadedApk( makeApplication )  
mInstrumentation( newApplication ) -> Application( attach )

## 3. APP 进程 和 AMS 的通讯。
APP 进程，向 AMS 通信，是通过 ActivityManagerNative 来的。具体实现是在  
ActivityManagerService 。

AMS 向 APP 进程通信，则是通过 ApplicationThreadNative 来实现的。具体实现是在
ActivityThread 的 内部类 ApplicationThread 。

## 5. Hook 的实现
### 类的静态变量

```java
Class<?> targetClass = Class.forName("targetClass");
Field cacheField = targetClass.getDeClaredField("sCache");
cacheField.setAccessible(true);
Map<String, String> cache = (Map<String, String>)cacheField.get(null);
cache.put("key", "value");
```

### AOP 动态代理
条件，需要代理的目标类，实现了接口, 并且有原始对象。

```java
public static Object newProxyInstance(ClassLoader loader,
                                          Class<?>[] interfaces,
                                          InvocationHandler h)
sample as follows.

IBinder hookedBinder = (IBinder) Proxy.newProxyInstance(serviceManager.getClassLoader(), new
                    Class[]{IBinder
                    .class},
                    new IClipboardHookBinderHandler(rawBinder));

InvocationHandler sample as follows

public class IClipboardHookBinderHandler implements InvocationHandler {
  public IClipboardHookBinderHandler(IBinder IBinder) {
}
@Override
    public Object invoke(Object o, Method method, Object[] objects) throws Throwable {
        if ("queryLocalInterface".equals(method.getName())) {
            return your feature object;
        }
        return method.invoke(mIBinder, objects);
    }
}

```

## 8. APP 进程入口 **【undo】**
入口是在 AcitivityThread 的 main 方法。ActivityManagerService 的 startProcessLocked  
方法，调用  Process.start ，然后启动的。启动原理，暂时未明白。

## 9. ResourcesManager 和 AssetManager
ResourcesManager 以及 Resources 是在 ContextImpl 构造的时候，就创建。代表资源列表。  
AssetManager 常用的一个 hide 方法是 addAssetPath ，添加 资源列表 进去 。插件化和换肤  
功能中，可以常常见到。
