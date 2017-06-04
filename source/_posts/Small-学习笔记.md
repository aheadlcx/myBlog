title: Small 学习笔记 - 流程篇
date: 2017-02-04 15:08:09
tags:
---
记录 Small 启动流程
<!--more  -->
## Small 框架的 API 理解

### Small API 的调用顺序
从 sample 中可以看出，small 的启动流程，主要分下面几部。

* Small.preSetUp(Application context)
* Small.setBaseUri()
* Small.setWebViewClient
* Small.setLoadFromAssets
* Small.setUp(Context context, OnCompleteListener listener)

其中较重要的是 preSetUp 和 setUp 方法。

#### Small.preSetUp 方法

```java
public static void preSetUp(Application context) {
        if (sContext != null) {
            return;
        }

        sContext = context;

        // Register default bundle launchers
        registerLauncher(new ActivityLauncher());
        registerLauncher(new ApkBundleLauncher());
        registerLauncher(new WebBundleLauncher());
        Bundle.onCreateLaunchers(context);
    }
```

其中 preSetup 方法就是注册了几个 BundleLauncher ，并且调用了他们的 onCreate 方法。

#### Small.setUp 方法

setUp 方法，调用链如下
Bundle.loadLaunchableBundles(listener) ->   
Bundle.loadBundles(context) ->   
Bundle.parseManifest(JSONObject data) ->  
Bundle 的构造方法 以及 Bundle.initWithMap(JSONObject map) ->
Bundle.setupLaunchers(Context context) ->  
Bundle.loadBundles(List<Bundle> bundles) ->  
Bundle.prepareForLaunch ->  
executor.execute(sIOActions) ->

##### Bundle.loadLaunchableBundles(listener)
判断了，同步还是异步运行 Bundle.loadBundles(Context) 方法。如果 listener 不为空，就异步。

##### Bundle.loadBundles
loadBundles(context)  方法做了 读取 bundle.json 的工作，优先级是
sharepreference cache > file cache > assets

##### Bundle.parseManifest
parseManifest(JSONObject data) 方法做了，解析 bundle.json 对应的 json 对象，  
json 中 bundles 对应的 value ，是一个数组，解析为 Small 的 Bundle 数组。理解为  
插件数组。

##### Bundle. initWithMap
在 Bundle 的构造方法以及 initWithMap 方法中,主要做了 解析 bundle.json 为对应的 Bundle ，
主要包含  插件的存放路径 mBuiltinFile 以及 uri 和 rules 等变量。

##### Bundle.setupLaunchers(Context context)
直接调用了几个 sBundleLaunchers 的 setUp 方法，这几个 sBundleLaunchers 的 setUp 方法  
实现，下面继续看。

````java
protected static void setupLaunchers(Context context) {
        if (sBundleLaunchers == null) return;

        for (BundleLauncher launcher : sBundleLaunchers) {
            launcher.setUp(context);
        }
    }
````

##### Bundle.loadBundles(List<Bundle> bundles)
这里做了真正的 加载 插件的工作，以及为 插件跳转做了准备工作。  
插件的加载工作室由 BundleLauncher 去做的。插件之间跳转的工作，也是由 BundleLauncher 做的。



### BundleLauncher 类理解
这里先查看 BundleLauncher 的方法列表，以及方法调用顺序，以及简单的调用链。最后，分析几个  
BundleLauncher 的实现类的方法实现体。
#### BundleLauncher 类的方法列表


```java
public abstract class BundleLauncher {
  public void onCreate(Application app) { }
  public void setUp(Context context) { }
  public void postSetUp() { }
  public boolean resolveBundle(Bundle bundle) {}
  public boolean preloadBundle(Bundle bundle) {}
  public void loadBundle(Bundle bundle) { }
  public void prelaunchBundle(Bundle bundle) { }
  public void launchBundle(Bundle bundle, Context context) {}
  public void upgradeBundle(Bundle bundle) {}
  public <T> T createObject(Bundle bundle, Context context, String type) {}
  private boolean shouldFinishPreviousActivity(Activity activity) {}
}
```

#### BundleLauncher 类的方法，调用顺序如下  

* public void onCreate(Application app)     
invoked by Small.preSetUp ->
Bundle.onCreateLaunchers

* public void setUp(Context context)        
invoked by Bundle.loadBundles(Context) ->
Bundle.setupLaunchers

* public boolean resolveBundle(Bundle bundle)
invoked by Bundle.loadBundles(List<Bundle> bundles) ->
Bundle.prepareForLaunch

* public boolean preloadBundle(Bundle bundle)
见如下代码，由 BundleLauncher.resolveBundle 调用

````java
public boolean resolveBundle(Bundle bundle) {
        if (!preloadBundle(bundle)) return false;

        loadBundle(bundle);
        return true;
    }
````

* public void loadBundle(Bundle bundle)
同上

* public void postSetUp()
invoked by Bundle.loadBundles(List<Bundle> bundles)


#### BundleLauncher 的各个方法实现体
##### public void onCreate(Application app)
目前只有 ApkBundleLauncher 覆盖了该方法，干活了。
主要做了，hook mInstrumentation 和 mH ，都是为了实现 Acitivty 生命周期的。以及  
保存了所有的 provider 。**undo 这暂时没跟完源码**
##### public void setUp(Context context)
ActivityLauncher 和 ApkBundleLauncher 以及 WebBundleLauncher ，都有重写该方法。

* ActivityLauncher ， 在这做了把所有 Activity 名称都收集起来了。**todo, 初步判断是仅仅  
包含宿主 host 的 Activity **
* ApkBundleLauncher ，让 sBundleInstrumentation 对 getPendingIntent   
做了一层 AOP 代理。**todo，暂时不大明白**
* WebBundleLauncher ， 对 7.0 以上版本的 WebView 做了一层兼容处理，7.0 的 WebView  
在第一次启动的时候会把  WebView 的 assets 覆盖到 Application 的 assets 。  

##### public boolean resolveBundle(Bundle bundle)

```java
public boolean resolveBundle(Bundle bundle) {
        if (!preloadBundle(bundle)) return false;

        loadBundle(bundle);
        return true;
    }
```

先调用 preloadBundle ，返回 true ，再调用 loadBundle 方法，preloadBundle 方法，  
默认返回 true。 没有人重载该方法。

##### preloadBundle(Bundle bundle)

重载了 preloadBundle 方法的只有 ActivityLauncher 和 SoBundleLauncher。
这个方法的定义是，加载插件之前做的准备操作。  

* ActivityLauncher

```java
    public boolean preloadBundle(Bundle bundle) {
        if (sActivityClasses == null) return false;

        String pkg = bundle.getPackageName();
        return (pkg == null || pkg.equals("main"));
    }
```

检查一下 setup 过程收集的 activity 是否为空。**todo, 检查包名，意义未明白**

* SoBundleLauncher。
不是 Host ，才会调用方法 resolveBundle 。
解析了插件的 XML 中的 application 等相关信息(versionCode，versionName)。解析了插件的路径等  
暂时发现，并没有解析插件中的 activity 等。仅仅解析了 application 以及其主题。

##### public void loadBundle(Bundle bundle)
重载了该方法的有 AssetBundleLauncher 和 ApkBundleLauncher 。  
看看 ApkBundleLauncher 的。  
把 dex 从 so 文件中加载了出来，变成了 optDex ,这时还没有插入进去。收集了所有的 Activity  
和 intentFilter ，为以后启动做准备。  


这里手机的 activity 以及其 intentFilter 信息是在 BundleParser 的 collectActivities  
方法中收集的。  
BundleParser 里面含有这个成员变量，mPackageInfo.activities 是保存了所有 activity 基本  
信息， mIntentFilters 是一个 map 对象，格式如下  

```java
ConcurrentHashMap<String, List<IntentFilter>>
```

以 activity 为名字，保存了其对应的所有 intentFilter 。


最后，将 activity 相关信息都转移到了 ApkBundleLauncher 中。

```java
sLoadedActivities  = new ConcurrentHashMap<String, ActivityInfo>()

sLoadedIntentFilters = new ConcurrentHashMap<String, List<IntentFilter>>();
```



##### public void postSetUp()
只有 ApkBundleLauncher 重载了该方法.做了以下几个事情：  

* 资源的合并插入
* 把所有 dex 插入进去。
* 插件的 so 文件的插入。
* 手动调用插件的 application 的 onCreate 方法.
* 延迟初始化 mLazyInitProviders .

**todo ，查看资源的合并，资源的插入，dex 的插入兼容性方面，以及第二次插入会不会  
造成重复插入的问题**
