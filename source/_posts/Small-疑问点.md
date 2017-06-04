title: Small 学习笔记-疑问点
date: 2017-05-05 16:14:59
tags:
---
记录 Small 疑惑点
<!--more  -->
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

* AssetManager 打开 xml
其中一段代码如下
用 AssetManager 添加一个资源路径，返回的 cookie ，然后用这个 cookie 和 想要打开的 xml 文件名  
即可得到想要打开的 xml 的 XmlResourceParser 。  

**undo 这里，添加一个路径，是不限制文件格式的么。 apk so 都可以？**

````java
int cookie = ReflectAccelerator.addAssetPath(assmgr, mArchiveSourcePath);
            if(cookie != 0) {
                parser = assmgr.openXmlResourceParser(cookie, "AndroidManifest.xml");
                assetError = false;
            } else {
                Log.w(TAG, "Failed adding asset path:"+mArchiveSourcePath);
            }
````

* 判断插件，是否有资源
用了变量 nonResources 保存。

````java
int flags = parser.getAttributeIntValue(null, "platformBuildVersionCode", 0);
            int abiFlags = (flags & 0xFFFFF000) >> 12;
            mNonResources = (flags & 0x800) != 0;
````

* 插件更新，原理
目前看到的相关代码如下  

ApkBundleLauncher 的 postSetUp 方法里面  
**目前未明白，为何删除了 optDexFile  ，下次重启就可以更新插件了。**初步推测，是删了之后，下次  
DexFile.loadDex 就会返回新的 DexFile 了。

````java
for (LoadedApk apk : apks) {
    dexPaths[i] = apk.path;
    dexFiles[i] = apk.dexFile;
    if (Small.getBundleUpgraded(apk.packageName)) {
        // If upgraded, delete the opt dex file for recreating
        if (apk.optDexFile.exists()) apk.optDexFile.delete();
        Small.setBundleUpgraded(apk.packageName, false);
    }
    i++;
}
````

* public.txt 更新和删除
这份文件是保存的 R 文件的 id ，目前还没搞清楚，删除和更新的时机。
