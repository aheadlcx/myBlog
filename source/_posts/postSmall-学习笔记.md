title: Small 学习笔记
date: 2017-02-04 15:08:09
tags:
---

##  疑问点

* 资源加载
AssetManager 的 addAssetPath 方法可以添加一个资源搜索路径，然后就可以拿到一个新的   
Resource 的了，代码如下：

```java
AssetManager mAssetManager;
        try {
            AssetManager assetManager = AssetManager.class.newInstance();
            Method addAssetPath = assetManager.getClass().getMethod("addAssetPath", String.class);
            addAssetPath.invoke(assetManager, skinPath);
            mAssetManager = assetManager;
        } catch (Exception e) {
            e.printStackTrace();
            return null;
        }
        Resources skinResources = new Resources(mAssetManager, metrics, configuration());
```

但是 Activity 的 getResource 方法是 ContextThemeWrapper 提供的。这2个具体是怎么  
结合起来的。

* 其他组件包打包成 so 文件并存放在宿主中。这个到底是怎么安装的

* dex 会不会重复插入了
* 资源合并
* Small.open(Uri uri)
查询参数
*  assets 目录下的文件 bundle.json 可以这样打开

```java
sPatchManifestFile = new File(Small.getContext().getFilesDir(), "bundle.json");
```

## Small 框架的 API 理解

### Small API 的调用顺序
* Small.preSetUp(Application context)
* Small.setBaseUri()
* Small.setWebViewClient
* Small.setLoadFromAssets
* Small.setUp(Context context, OnCompleteListener listener)

其中较重要的是 preSetUp 和 setUp 方法。

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


setUp 方法，调用链如下
Bundle.loadLaunchableBundles(listener) ->   
Bundle.loadBundles(context) ->   
Bundle.parseManifest(JSONObject data) ->  
Bundle.setupLaunchers(Context context) ->  
Bundle.loadBundles(List<Bundle> bundles) ->  
Bundle.prepareForLaunch ->  
executor.execute(sIOActions) ->


loadBundles(context)  方法做了 读取 bundle.json 的工作，优先级是
sharepreference cache > file cache > assets


parseManifest(JSONObject data) 方法做了，解析 bundle.json 对应的 json 对象，  
json 中 bundles 对应的 value ，是一个数组，解析为 Small 的 Bundle 数组。理解为  
插件数组。


### BundleLauncher 类的方法的调用顺序
BundleLauncher 类的方法列表

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

BundleLauncher 类的方法，调用顺序如下  

* public void onCreate(Application app)
* public void setUp(Context context)
* public boolean resolveBundle(Bundle bundle)
* public boolean preloadBundle(Bundle bundle)
* public void loadBundle(Bundle bundle)
* public void postSetUp()






#### public void onCreate(Application app)
目前只有 ApkBundleLauncher 覆盖了该方法，干活了。
#### public void setUp(Context context)
ActivityLauncher 和 ApkBundleLauncher 以及 WebBundleLauncher ，都有重写该方法。

* ActivityLauncher ， 在这做了把所有 Activity 名称都收集起来了。**todo, 初步判断是仅仅  
包含宿主 host 的 Activity **
* ApkBundleLauncher ，让 sBundleInstrumentation 对 getPendingIntent   
做了一层 AOP 代理。**todo，暂时不大明白**
* WebBundleLauncher ， 对 7.0 以上版本的 WebView 做了一层兼容处理，7.0 的 WebView  
在第一次启动的时候会把  WebView 的 assets 覆盖到 Application 的 assets 。  

#### public boolean resolveBundle(Bundle bundle)

```java
public boolean resolveBundle(Bundle bundle) {
        if (!preloadBundle(bundle)) return false;

        loadBundle(bundle);
        return true;
    }
```

先调用 preloadBundle ，返回 true ，再调用 loadBundle 方法，preloadBundle 方法，  
默认返回 true。重载了 preloadBundle 方法的只有 ActivityLauncher 和 SoBundleLauncher。
这个方法的定义是，加载插件之前做的准备操作。  

* ActivityLauncher

```java
    public boolean preloadBundle(Bundle bundle) {
        if (sActivityClasses == null) return false;

        String pkg = bundle.getPackageName();
        return (pkg == null || pkg.equals("main"));
    }
```

检查一下 setup 过程手机的 activity 名字。**todo, 检查包名，意义未明白**

* SoBundleLauncher。
不是 Host ，才会调用方法 resolveBundle 。
解析了插件的 XML 中的 application 等相关信息(versionCode，versionName)。解析了插件的路径等  
暂时发现，并没有解析插件中的 activity 等。仅仅解析了 application 以及其主题。

#### public void loadBundle(Bundle bundle)
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



#### public void postSetUp()
只有 ApkBundleLauncher 重载了该方法.做了以下几个事情：  

* 合并所有插件以及宿主的资源。
* 把所有 dex 插入进去。
* 插件的 so 文件的插入。
* 手动调用插件的 application 的 onCreate 方法.
* 延迟初始化 mLazyInitProviders .

**todo ，查看资源的合并，资源的插入，dex 的插入兼容性方面，以及第二次插入会不会  
造成重复插入的问题**
