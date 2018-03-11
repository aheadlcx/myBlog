title: small 关键点小结
date: 2018-03-04 14:41:57
tags:
---



### hookVariantTask 方法
hookVariantTask 方法是 small plugin 插件的核心所在。

````java
    protected void hookVariantTask(BaseVariant variant) {
        hookMergeAssets(variant.mergeAssets)

        hookProcessManifest(small.processManifest)

        hookAapt(small.aapt)

        hookJavac(small.javac, variant)

        hookKotlinCompile()

        def transformTasks = project.tasks.withType(TransformTask.class)
        def mergeJniLibsTask = transformTasks.find {
            it.transform.name == 'mergeJniLibs' && it.variantName == variant.name
        }
        hookMergeJniLibs(mergeJniLibsTask)

        def mergeJavaResTask = transformTasks.find {
            it.transform.name == 'mergeJavaRes' && it.variantName == variant.name
        }
        hookMergeJavaRes(mergeJavaResTask)

        // Hook clean task to unset package id
        project.clean.doLast {
            sPackageIds.remove(project.name)
        }
    }
````

hookMergeAssets 方法，先放下，先跟主流程。先来跟踪 hookProcessManifest 方法。

#### hookProcessManifest 方法 
合并 manifest中，有可能会 application@name 冲突，因此方法体内，做了如下几件事

+ 把 providers 清空
+ 移除了自身依赖 libs 中，rootExtension 的 libProjects 。
+ 移除 'android:icon', 'android:label','android:allowBackup', 'android:supportsRtl' 这几个标识。

#### hookAapt 方法
解析库资源ID 

+ 在 tash ProcessAndroidResources 最后，把 ap_ 文件 解压到临时文件 ap_unzip 中。
+ 接着调用 prepareSplit 方法，该方法作用是 把资源包，保持资源类型和ID。后面，该方法见下面详细分析。


##### prepareSplit 方法
几个重要变量

+ small.symbolFile 本 project 的符号表
+ rootSmall.strictSplitResources 严格模式下，插件不允许使用第三方aar，需要在宿主里同时申明，或者将strictSplitResources设置为false。即“非强制模式” 为什么默认为严格模式是因为 最初的设想是第三方依赖一般变动少，可以直接放宿主定义。这样插件里就可以剥离，达到插件最小化。后为了灵活性，增加了strictSplitResources的关闭选项。
+ mUserLibAars ，
+ small.retainedAars    [issue 194](https://github.com/wequick/Small/issues/194)
+ transitiveVendorAars
+ firstLevelVendorAars
+ libEntries 是加上宿主的 符号表 +  mTransitiveDependentLibProjects 的符号表
+ mTransitiveDependentLibProjects


这个方法，最终是给下面几个变量赋值了

````java
small.idMaps            
small.idStrMaps         
small.retainedTypes     
small.retainedStyleables
small.allTypes          
small.allStyleables     
small.vendorTypes       
small.vendorStyleables  
````


#### buildLib 方法


















+ 