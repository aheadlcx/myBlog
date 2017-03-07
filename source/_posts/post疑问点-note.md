title: 疑问点 note
date: 2017-03-02 19:57:10
tags:
---


1. activity 生命周期管理的 token
2. acitivty 和 application 的 context 是什么时候注入的，以及实现分别是什么。
3. activitythread 和 AMS ,通讯，是怎么实现的。
4. servicemanager 的服务 是什么时候注入的。
5. hook ,静态的，和实例的，分别如何实现。
6. 测试文件锁，是否可以跨进程锁。参考 MultiDex 的文件锁， master 分支有。
7. APP 进程启动的时候，ActivityManagerNative 的 asInterface 方法，是什么时候出发的。
8. APP 进程入口在那里。
9. ResourcesManager 和 AssetManager 有什么区别和联系

## 1. activity 生命周期管理
Activity里面有一个成员变量mToken代表的就是它，token可以唯一地标识一个Activity对象，  
它在Activity的attach方法里面初始化。



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
