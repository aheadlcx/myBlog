title: small-gradle-plugin 插件整理
date: 2017-06-04 15:04:10
tags: [android,gradle]
category: android
---

## 目标
+ 搞清楚 gradle 插件的生命周期。
+ 搞清楚 apk 打包流程。
+ 搞清楚 资源 ID 分区的流程
+ 混淆的流程。

## 一些关键变量的赋值
##### AndroidExtension  Task jar

````java
/** Task of R.class jar */
    Task jar
````

在 LibraryPlugin 和 HostPlugin 的 configureReleaseVariant 

````java
        small.jar = project.jarReleaseClasses
        ......
        HostPlugin show as below
        def flavor = variant.flavorName
        if (flavor != null) {
            flavor = flavor.capitalize()
            small.jar = project.tasks["jar${flavor}ReleaseClasses"]
            small.aapt = project.tasks["process${flavor}ReleaseResources"]
        } else {
            small.jar = project.jarReleaseClasses
            small.aapt = project.processReleaseResources
        }
        project.buildLib.dependsOn small.jar
````

##### AndroidExtension Map buildCaches


## 关键问题点
    * ap 文件生成时机和作用   preApDir
    * 符号表 R.txt 生成时机和作用  preIdsDir
    * pre-jar  base 的 jar 生成时机和作用 preBaseJarDir
    * pre-jar 中 lib 的 jar 生成时机和作用 preLibsJarDir
    * pre-link 中 aar   preLinkAarDir
    * pre-link 中 jar   preLinkJarDir
    * smallLibs 的生成时机
    * apk 存放路径是在 BundlePlugin configureReleaseVariant 中确定的。
    * LibraryPlugin 类型的，会保存符号表到 project 的 public.txt
    * 混淆的流程。
    * splitRJavaFile 文件是 hookAapt 后面重新生成的过滤后的R文件。
    * R 文件怎么过滤的
    * 资源文件怎么打成一个小资源文件的
    * 代码怎么仅仅打了本 project 代码的。

buildLib 方法疑问点
+ ext.jar.archivePath
+ ext.buildCaches
+ 为什么最后生成的 R.keys.txt 来源于 Mine
+ 如何所有类一起参与混淆呢
+ preBaseJarDir 的 app-r.jar 文件是什么鬼
+ AppPlugin 的 configureProguard 方法，混淆后，删除所有R$文件，之后又重新生成R文件。
+ AppPlugin hookAapt 方法中，重新生成 rJavaFile 文件，何用？
+ AppPlugin 依赖的三方库的 R 文件，怎么处理了。

### ap 文件生成时机和作用   preApDir
#### 生成时机
RootPlugin buildLib 方法中。可是目前看到 app 也生成了相关的 AP 文件。

````java
 if (it.hasProperty('buildLib')) {
                    it.small.buildIndex = ++rootExt.libCount
                    it.tasks['buildLib'].doLast {
                        buildLib(it.project)
                    }
                }
````

#### 作用 


### 符号表 R.txt 生成时机和作用  preIdsDir
和 preApDir ，基本一致。疑问点

+ R.keys.txt 文件生命时候生成

````java
        // Copy R.txt
        def preIdsDir = small.preIdsDir
        if (!preIdsDir.exists()) preIdsDir.mkdir()
        def srcIdsFile = new File(aapt.textSymbolOutputDir, 'R.txt')
        if (srcIdsFile.exists()) {
            def idsFileName = "${libName}-R.txt"
            def keysFileName = 'R.keys.txt'
            def dstIdsFile = new File(preIdsDir, idsFileName)
            def keysFile = new File(preIdsDir, keysFileName)
            def addedKeys = []
            if (keysFile.exists()) {
                keysFile.eachLine { s ->
                    addedKeys.add(SymbolParser.getResourceDeclare(s))
                }
            }
            def idsPw = new PrintWriter(dstIdsFile.newWriter(true)) // true=append mode
            def keysPw = new PrintWriter(keysFile.newWriter(true))
            srcIdsFile.eachLine { s ->
                def key = SymbolParser.getResourceDeclare(s)
                if (addedKeys.contains(key)) return
                idsPw.println(s)
                keysPw.println(key)
            }
            idsPw.flush()
            idsPw.close()
            keysPw.flush()
            keysPw.close()
        }
````


### pre-jar  base 的 jar 生成时机和作用 preBaseJarDir
#### 生成时机
目前看来是仅仅把 R 文件和 small-databinding-stub.jar 搬过去了。**undo**

#### 作用

### pre-jar 中 lib 的 jar 生成时机和作用 preLibsJarDir
#### 生成时机
LibraryPlugin 中的 configureReleaseVariant 方法

````java
variant.assemble.doLast {
            // Generate jar file to root pre-jar directory
            // FIXME: Create a task for this
            def jarName = getJarName(project)
            def jarFile = new File(rootSmall.preLibsJarDir, jarName)
            if (mMinifyJar != null) {
                FileUtils.copyFile(mMinifyJar, jarFile)
            } else {
                project.ant.jar(baseDir: small.javac.destinationDir, destFile: jarFile)
            }
        }
````

在 AppPlugin 的 getLibraryJars 方法中，也用到过。
#### 作用


### pre-link 中 aar   preLinkAarDir 和 preLinkJarDir
#### 生成时机
在 RootPlugin 的 buildLib 方法中

````java
// Backup dependencies
        if (!small.preLinkAarDir.exists()) small.preLinkAarDir.mkdirs()
        if (!small.preLinkJarDir.exists()) small.preLinkJarDir.mkdirs()
        def linkFileName = "$libName-D.txt"
        File aarLinkFile = new File(small.preLinkAarDir, linkFileName)
        File jarLinkFile = new File(small.preLinkJarDir, linkFileName)
````

AppPlugin collectAarsOfProject 

````java
    protected def collectAarsOfProject(Project project, boolean isLib, HashSet outAars) {
        String dependenciesFileName = "$project.name-D.txt"

        // Pure aars
        File file = new File(rootSmall.preLinkAarDir, dependenciesFileName)
        collectAars(file, project, outAars)

        // Jar-only aars
        file = new File(rootSmall.preLinkJarDir, dependenciesFileName)
        collectAars(file, project, outAars)

        if (isLib) {
            collectAarsOfLibrary(project, outAars)
        }
    }

````

其中 preLinkJarDir 在 AppPlugin 的 resolveReleaseDependencies 用到

````java
    protected void resolveReleaseDependencies() {
        // Pre-split all the jar dependencies (deep level)
        def compile = project.configurations.compile
        compile.exclude group: 'com.android.support', module: 'support-annotations'
        rootSmall.preLinkJarDir.listFiles().each { file ->
            if (!file.name.endsWith('D.txt')) return
            if (file.name.startsWith(project.name)) return

            file.eachLine { line ->
                def module = line.split(':')
                def group = module[0]
                def name = module[1]
                compile.exclude group: group, module: name
            }
        }

        // Provide all the jars
        def includes = ['*.jar']
        def jars = project.fileTree(dir: rootSmall.preBaseJarDir, include: includes)
        jars += project.fileTree(dir: rootSmall.preLibsJarDir, include: includes)
        project.dependencies.add("provided", jars)
    }

````

#### 作用

### 混淆的流程。
#### 在 AppPlugin 的 configureProguard 中 **undo**

+ 1. proguard keep 了 所有 getLibraryJars()，但是不包括 provided 的。
+ 2. 在 AppPlugin 的 configureProguard, 拿到混淆后生成的 jar minifyJar
+ 3. 解压 minifyJar ，并且删除 R 文件。
+ 4. 重新 javac R.java 到 R.class，然后重新压缩所有 class 文件。

### R 文件怎么过滤的


### 资源文件怎么打成一个小资源文件的
AppPlugin 的 hookAapt 

````java
 Aapt aapt = new Aapt(unzipApDir, rJavaFile, symbolFile, rev)
 if (small.retainedTypes != null && small.retainedTypes.size() > 0) {
     aapt.filterResources(small.retainedTypes, filteredResources)
````


## 几个类的作用
### BasePlugin
在 BasePlugin 的 apply 里面，先后自定义了3个方法

+ createExtension
+ configureProject
+ createTask

其中 createExtension 方法，创建了一个 small DSL 配置,也就是说 所有  plugin 都有这个配置。根据 taskName 来判断当前是 buildLib 还是  buildBundle 。

### RootPlugin
几个重要的方法有

````java
protected void configureProject() {****}
protected void configVersions(Project p, RootExtension.AndroidConfig base) {****}
protected void createTask() {****}
void buildLib(Project lib) {****}
````

#### configureProject 方法
**执行时机**在 Plugin 的apply 方法体内。  
该方法体，**执行时机**是这个 RootPlugin 对应的 Project 配置阶段后面再执行。  
做了如下一些事情。

+ userBundleTypes **undo**
+ 遍历所有的 subProject 并且把子 project 的 compileSdkVersion、buildToolsVersion  
和 suport 库的 supportVersion ，都改写成和 最外层 gradle 文件的 small DSL 内的 android 代码块内一样。
+ 根据 module name 来区分不同的类型，例如 app lib，并且给子 project apply 申请不同的 plugin 。
+ 看子 project 有 buildLib 还是 buildBundle 属性，也就是 task ，记录其 buildIndex  
**undo** 这里，暂时看不出来这个 buildIndex 什么时候用的。**undo** 并且为什么在 buildLib  这个 task 的最后 添加了一个 action ，执行 buildLib 方法。
+ 并且让宿主、 app 模块和 lib 模块，都依赖 宿主分身模块。这个执行时机是在 每个子 project 的配置阶段后面做的。


#### configVersions 方法
就是把 形参的 project 的 compileSdkVersion buildToolsVersion  supportVersion 都改成 small 里面的 android 配置项。

#### createTask 方法
创建了5个 task  ，cleanLib buildLib  cleanBundle  buildBundle  small .其中 small 仅仅是打印环境 log ，方便调试。这个 plugin 的 buildLib buildBundle 都没有依赖任何 task。  原来不是 5个 task ，其实是6个 task 。 还有一个 smallLint ，类型是 LintTask ,依赖  
:app:transformClassesWithDexFor$flavorName  。这个 transformClassesWithDexFor task 的作用是 把包含所有 class 的jar 包转换成 dex 。 **undo** LintTask 这个task 初步估计是 代码审查用的，暂时未看代码，先分析主流程。

#### buildLib 方法
**执行时机** 这个方法的执行时机，上面提到过，就是带有 buildLib 的 project 的 buildLib task 的配置阶段后面，执行的。  
这个方法体，做的事情比较多，暂时不懂的也比较多，因此先分段。


**undo**这里 jar 包，目前不确定是仅仅移了 R 文件，还是所有 class 的 jar 包。

````java
// Copy jars
        def preJarDir = small.preBaseJarDir
        if (!preJarDir.exists()) preJarDir.mkdirs()
        //TODO 这个 ext.jar 是一个 Task ，是什么时候注入的
        //  - copy package.R jar
        if (ext.jar != null) {
            def rJar = ext.jar.archivePath
            project.copy {
                from rJar
                into preJarDir
                rename {"$libName-r.jar"}
            }
            //TODO 这里表面是说 仅仅 R 文件，其实看 app情况，已经所有类了。
        }
````


**undo** 这里很奇怪，ext.explodeAarDirs 每次都是 等于1 ，但是里面的元素 是 null ，先不继续分析这段代码。


````java
def b = ext.explodeAarDirs == null
        if (!b){
            println "${TAG_ROOT}  ext.explodeAarDirs size = ${ext.explodeAarDirs.size()} "
        }
        ext.explodeAarDirs.each {
            println "${TAG_ROOT}   ---  world ${it == null}"
            // explodedDir: **/exploded-aar/$group/$artifact/$version
            if (it == null) return

            File version = it
            if (LOG_OPEN_ROOT){
                println "${TAG_ROOT}  version path -------}"
                println "${TAG_ROOT}  version path = ${version.path}"
                println "${TAG_ROOT}  version path -------}"
            }

            File jarDir = new File(version, 'jars')

            if (LOG_OPEN_ROOT) {
                println "${TAG_ROOT}   jarDir path = ${jarDir.path}"
            }

            File jarFile = new File(jarDir, 'classes.jar')
            if (!jarFile.exists()) return

            File artifact = version.parentFile
            File group = artifact.parentFile
            File destFile = new File(preJarDir,
                    "${group.name}-${artifact.name}-${version.name}.jar")
            if (destFile.exists()) return

            project.copy {
                from jarFile
                into preJarDir
                rename {destFile.name}
            }

            // Check if exists `jars/libs/*.jar' and copy
            File libDir = new File(jarDir, 'libs')
            libDir.listFiles().each { jar ->
                if (!jar.name.endsWith('.jar')) return

                destFile = new File(preJarDir, "${group.name}-${artifact.name}-${jar.name}")
                if (destFile.exists()) return

                project.copy {
                    from jar
                    into preJarDir
                    rename {destFile.name}
                }
            }
        }

````


下面这段代码，是复制资源文件。


````java
// Copy *.ap_
        def aapt = ext.aapt
        def preApDir = small.preApDir
        println "${TAG_ROOT}   preApDir = ${preApDir.path}"

        if (!preApDir.exists()) preApDir.mkdir()
        def apFile = aapt.packageOutputFile
        def preApName = "$libName-resources.ap_"
        project.copy {
            from apFile
            into preApDir
            rename {preApName}
        }
````


下面是复制 R 文件


````java
// Copy R.txt
        def preIdsDir = small.preIdsDir
        println "${TAG_ROOT}   preIdsDir = ${preIdsDir.path}"
        if (!preIdsDir.exists()) preIdsDir.mkdir()
        def srcIdsFile = new File(aapt.textSymbolOutputDir, 'R.txt')
        if (srcIdsFile.exists()) {
            def idsFileName = "${libName}-R.txt"
            def keysFileName = 'R.keys.txt'
            def dstIdsFile = new File(preIdsDir, idsFileName)
            def keysFile = new File(preIdsDir, keysFileName)
            def addedKeys = []
            if (keysFile.exists()) {
                keysFile.eachLine { s ->
                    addedKeys.add(SymbolParser.getResourceDeclare(s))
                }
            }
            def idsPw = new PrintWriter(dstIdsFile.newWriter(true)) // true=append mode
            def keysPw = new PrintWriter(keysFile.newWriter(true))
            srcIdsFile.eachLine { s ->
                def key = SymbolParser.getResourceDeclare(s)
                if (addedKeys.contains(key)) return
                idsPw.println(s)
                keysPw.println(key)
            }
            idsPw.flush()
            idsPw.close()
            keysPw.flush()
            keysPw.close()
        }
````


备份依赖库。


````java
// Backup dependencies
        if (!small.preLinkAarDir.exists()) small.preLinkAarDir.mkdirs()
        if (!small.preLinkJarDir.exists()) small.preLinkJarDir.mkdirs()
        def linkFileName = "$libName-D.txt"
        File aarLinkFile = new File(small.preLinkAarDir, linkFileName)
        File jarLinkFile = new File(small.preLinkJarDir, linkFileName)

        def allDependencies = DependenciesUtils.getAllDependencies(lib, 'compile')
        if (allDependencies.size() > 0) {
            def aarKeys = []
            if (!aarLinkFile.exists()) {
                aarLinkFile.createNewFile()
            } else {
                aarLinkFile.eachLine {
                    aarKeys.add(it)
                }
            }

            def jarKeys = []
            if (!jarLinkFile.exists()) {
                jarLinkFile.createNewFile()
            } else {
                jarLinkFile.eachLine {
                    jarKeys.add(it)
                }
            }

            def aarPw = new PrintWriter(aarLinkFile.newWriter(true))
            def jarPw = new PrintWriter(jarLinkFile.newWriter(true))

            // Cause the later aar(as fresco) may dependent by 'com.android.support:support-compat'
            // which would duplicate with the builtin 'appcompat' and 'support-v4' library in host.
            // Hereby we also mark 'support-compat' has compiled in host.
            // FIXME: any influence of this?
            if (lib == small.hostProject) {
                aarPw.println "com.android.support:support-compat:+"
                aarPw.println "com.android.support:support-core-utils:+"
            }

            allDependencies.each { d ->
                def isAar = true
                d.moduleArtifacts.each { art ->
                    // Copy deep level jar dependencies
                    File src = art.file
                    if (art.type == 'jar') {
                        isAar = false
                        project.copy {
                            from src
                            into preJarDir
                            rename { "${d.moduleGroup}-${src.name}" }
                        }
                    }
                }
                if (isAar) {
                    if (!aarKeys.contains(d.name)) {
                        aarPw.println d.name
                    }
                } else {
                    if (!jarKeys.contains(d.name)) {
                        jarPw.println d.name
                    }
                }
            }
            jarPw.flush()
            jarPw.close()
            aarPw.flush()
            aarPw.close()
        }

````

#### RootPlugin 总结
先总结 undo 部分。  
+ configureProject 方法体的 userBundleTypes 代码块，目前不知道什么时候，赋值的。这个的作用，就是给 子 project 分类，进而给不同类型的子 project apply 不同的 plugin。
+ configureProject 方法体里面。子 project 什么时候添加上  buildLib 和 buildBundle task 的，以及为什么给 buildLib 添加一个 doLast action 并且执行 buildLib 方法了。
+ buildLib 方法体内，赋值 R 文件和 jar 包、ID 文件以及 资源文件等操作，这些是什么时候赋值的。

### AndroidPlugin
做的事情并不多，根据 taskName 判断是否 release .并且在方法 configureProject 中自定义了一些方法。

````java
beforeEvaluate(released)
afterEvaluate(released)
configureProguard(variant, proguard, pt)
configureReleaseVariant(variant)
protected void hookPreReleaseBuild() { }
removeUnimplementedProviders()
````


可以说 AndroidPlugin 的其他方法 都是在 configureProject 方法中造出来的。


````java
@Override
protected void configureProject() {
    super.configureProject()

    project.beforeEvaluate {
        beforeEvaluate(released)
    }

    project.afterEvaluate {
        afterEvaluate(released)

        if (!android.hasProperty('applicationVariants')) return

        android.applicationVariants.all { BaseVariant variant ->
            // Configure ProGuard if needed
            if (variant.buildType.minifyEnabled) {
                def variantName = variant.name.capitalize()
                def proguardTaskName = "transformClassesAndResourcesWithProguardFor$variantName"
                def proguard = (TransformTask) project.tasks[proguardTaskName]
                def pt = (ProGuardTransform) proguard.getTransform()
                configureProguard(variant, proguard, pt)
            }

            // While variant created, everything of `Android Plugin' should be ready
            // and then we can do some extensions with it
            if (variant.buildType.name != 'release') {
                if (!released) {
                    configureDebugVariant(variant)
                }
            } else {
                if (released) {
                    configureReleaseVariant(variant)
                }
            }
        }
//            def buildLibInner = tasks.findByName('buildLib')
//            def allDependenTask = buildLibInner.taskDependencies.getDependencies(buildLibInner)
//            println 'buildLibInner = ' + allDependenTask
    }
}
````


**undo**在方法 configureReleaseVariant 中设置了 small 的 outputFile 和 explodeAarDirs 。这个实现细节，未完全看完。


````java
protected void configureReleaseVariant(BaseVariant variant) {
    // Init default output file (*.apk)
    small.outputFile = variant.outputs[0].outputFile
    small.explodeAarDirs = project.tasks
            .withType(PrepareLibraryTask.class)
            .collect { TaskUtils.getAarExplodedDir(it) }

    // Hook variant tasks
    variant.assemble.doLast {
        tidyUp()
    }
}
````


在 afterEvaluate 方法中，给 当前 project 加上了 small 的依赖，因此我们可以看到，没有显式的依赖 small ，但是也可以 import small 的代码。


#### AndroidPlugin 总结
+ 在 configureProject 中定义了一系列回调方法。
+ 设置了 small 的 outputFile 和 explodeAarDirs 。
+ 默认给依赖了 small ，因此每个 project 可以引用到 small 的代码。

### BundlePlugin
这个比较简单， 在 afterEvaluate 方法中，采用了 宿主模块的 signingConfig 来作为当前的  signingConfig ，并且读取命令行中的值，来判断是否打开混淆。  


在 configureReleaseVariant 方法中，重新设置了 small 的 outputFile  。覆盖了 AndroidPlugin 中的定义值。并且遍历了一下 variant.outputs ，并且把输出值都设置为上一个。


````java
@Override
    protected void configureReleaseVariant(BaseVariant variant) {
        super.configureReleaseVariant(variant)

        // Set output file (*.so)
        def outputFile = getOutputFile(variant)
        BundleExtension ext = small
        ext.outputFile = outputFile
        variant.outputs.each { out ->
            out.outputFile = outputFile
        }
    }
````


在 createTask 中，创建了 2个 task buildBundle 和 cleanBundle 分别依赖了 assembleRelease 和 clean 。


### AppPlugin
先来看看 AppPlugin 的类结构图。所有 Plugin 中，大部分工作 都是它干了。

![AppPlugin 类结构图](https://raw.githubusercontent.com/aheadlcx/myPhotos/master/small/small-appplugin.png)


这个类比较复杂，我们尽量按照生命周期顺序来分析。  
有几个重要的生命周期方法，按照顺序来。除了 configureProguard ，其他几个都确定了顺序了的。


````java
    afterEvaluate(boolean released)
    configureProguard(variant, proguard, pt)
    configureReleaseVariant(variant)
    hookPreDebugBuild()
````


先来看 afterEvaluate(boolean released) 方法

#### afterEvaluate 方法
`````java
@Override
protected void afterEvaluate(boolean released) {
    super.afterEvaluate(released)

    // Initialize a resource package id for current bundle
    initPackageId()

    // Get all dependencies with gradle script `compile project(':lib.*')'
    DependencySet compilesDependencies = project.configurations.compile.dependencies
    Set<DefaultProjectDependency> allLibs = compilesDependencies.withType(DefaultProjectDependency.class)
    Set<DefaultProjectDependency> smallLibs = []
    mUserLibAars = []
    mDependentLibProjects = []
    mProvidedProjects = []
    mCompiledProjects = []
    //TODO 这个收集这个是干嘛用的
    allLibs.each {
        // lib 已经有了
        if (rootSmall.isLibProject(it.dependencyProject)) {
            smallLibs.add(it)
            mProvidedProjects.add(it.dependencyProject)
            mDependentLibProjects.add(it.dependencyProject)
        } else {
            mCompiledProjects.add(it.dependencyProject)
            //TODO 这个暂时未明白
            collectAarsOfLibrary(it.dependencyProject, mUserLibAars)
        }
    }
    collectAarsOfLibrary(project, mUserLibAars)
    mProvidedProjects.addAll(rootSmall.hostStubProjects)

    if (rootSmall.isBuildingLibs()) {
        // While building libs, `lib.*' modules are changing to be an application
        // module and cannot be depended by any other modules. To avoid warnings,
        // remove the `compile project(':lib.*')' dependencies temporary.
        compilesDependencies.removeAll(smallLibs)
    }

    if (!released) return

    //TODO 这里暂时不明白
    // Add custom transformation to split shared libraries
    android.registerTransform(new StripAarTransform())

    //TODO 去除依赖 各种依赖 到底是怎么样的
    resolveReleaseDependencies()
}
````


做了下面几件事：

+ 给每个 module 自定义 packageId 中的 pp 字段。可以自己配置。默认是 根据 hash 值来处置。
+ 遍历所有 依赖，如果已经是 libProject 就记录下来。变成 provided 。
+ 如果正在 buildLib ，就移除 smallLib 就是 libProject
+ 增加了一个 StripAarTransform ，暂时不懂。
+ resolveReleaseDependencies 方法里面，去除了 support 和其他 priLingJar 依赖。

##### afterEvaluate 方法总结。
mark 未懂的部分先。
**undo**  


+ mProvidedProjects 、mDependentLibProjects 和 mCompiledProjects 到底是怎么用的。
+ collectAarsOfLibrary(project, mUserLibAars) ，这个是什么意思。
+ buildLib 阶段，lib module 会变成 apply application 。这个注意看对打包的影响。
+ 增加了 StripAarTransform
+ resolveReleaseDependencies 方法里面，去除了 support 和其他 priLingJar 依赖。


#### configureReleaseVariant 方法
给 small DSL 如下配置赋值了。
javac          
processManifest

packageName = net.wequick.example.small.lib.afterutils   

packagePath = net/wequick/example/small/lib/afterutils

classesDir = /Users/apple/mycode/androidFile/github/
Small/Android/Sample/lib.afterutils/build/intermediates/classes/release


bkClassesDir = /Users/apple/mycode/androidFile/github/
Small/Android/Sample/lib.afterutils/build/intermediates/classes/release~


${project.android.sourceSets.main.java.srcDirs}
project srcDirs = [/Users/apple/mycode/androidFile/github/
Small/Android/Sample/lib.afterutils/src/main/java]           


aapt           

apFile = /Users/apple/mycode/androidFile/github/
Small/
Android/Sample/lib.afterutils/build/intermediates/res/resources-release.ap_  

File symbolDir
symbolDir = /Users/apple/mycode/androidFile/github/Small/
Android/Sample/lib.afterutils/build/intermediates/symbols/release   


File sourceDir
sourceDir =  /Users/apple/mycode/androidFile/github/Small/
Android/Sample/lib.afterutils/build/generated/source/r/release    


symbolFile     
rJavaFile      

splitRJavaFile
splitRJavaFile = /Users/apple/mycode/androidFile/github/Small/
Android/Sample/lib.afterutils/build/
generated/source/r/small/net/wequick/example/small/lib/afterutils/R.java    

mergerXml     
mergerDir = /Users/apple/mycode/androidFile/github/Small/
Android/Sample/lib.afterutils/build/
intermediates/incremental/mergeReleaseResources

````java
@Override
protected void configureReleaseVariant(BaseVariant variant) {
    super.configureReleaseVariant(variant)

    // Fill extensions
    def variantName = variant.name.capitalize()
    File mergerDir = variant.mergeResources.incrementalFolder

    small.with {
        javac           = variant.javaCompile
        processManifest = project.tasks["process${variantName}Manifest"]

        packageName     = variant.applicationId
        packagePath     = packageName.replaceAll('\\.', '/')
        classesDir      = javac.destinationDir
        bkClassesDir    = new File(classesDir.parentFile, "${classesDir.name}~")

        aapt            = (ProcessAndroidResources) project
                          .tasks["process${variantName}Resources"]
        apFile          = aapt.packageOutputFile

        File symbolDir  = aapt.textSymbolOutputDir
        File sourceDir  = aapt.sourceOutputDir

        symbolFile      = new File(symbolDir, 'R.txt')
        rJavaFile       = new File(sourceDir, "${packagePath}/R.java")

        splitRJavaFile  = new File(sourceDir.parentFile, "small/${packagePath}/R.java")

        mergerXml       = new File(mergerDir, 'merger.xml')
    }

    hookVariantTask(variant)
}

````


上面的这些，赋值，应该可以解决一些 BasePlugin 或者 AndroidPlugin 的 undo 了。接下去继续看 hookVariantTask 方法。


````java
protected void hookVariantTask(BaseVariant variant) {
  **undo**
  // Hook merge-assets task to ignores the lib.* assets
    hookMergeAssets(variant.mergeAssets)

    // If an app.A dependent by lib.B and both of them declare application@name in their
// manifests, the `processManifest` task will raise an conflict error.
    //android:icon android:label   android:allowBackup    android:supportsRtl
    hookProcessManifest(small.processManifest)

    // Hook aapt task to slice asset package and resolve library resource ids
    hookAapt(small.aapt)

    // Hook javac task to split libraries' R.class
    hookJavac(small.javac, variant.buildType.minifyEnabled)

    def mergeJniLibsTask = project.tasks.withType(TransformTask.class).find {
        it.transform.name == 'mergeJniLibs' && it.variantName == variant.name
    }

    // Hook merge-jniLibs task to ignores the lib.* native libraries
    hookMergeJniLibs(mergeJniLibsTask)

    // Hook clean task to unset package id
    project.clean.doLast {
        sPackageIds.remove(project.name)
    }
}

````


##### configureReleaseVariant 方法总结
mark 一下未懂的部分。  
**undo**  


+ hookVariantTask 方法，每个子 hook 方法，全部都未看完。
+ hookMergeAssets 是去除 lib 的 assets


#### getLibraryJars 方法


````java
protected Set<File> getLibraryJars() {
    if (mLibraryJars != null) return mLibraryJars

    mLibraryJars = new LinkedHashSet<File>()

    // Collect the jars in `build-small/intermediates/small-pre-jar/base'
    def baseJars = project.fileTree(dir: rootSmall.preBaseJarDir, include: ['*.jar'])
    mLibraryJars.addAll(baseJars.files)

    // Collect the jars of `compile project(lib.*)' with absolute file path, fix issue #65
    Set<String> libJarNames = []
    Set<File> libDependentJars = []
    mTransitiveDependentLibProjects.each {
        libJarNames += getJarName(it)
        libDependentJars += getJarDependencies(it)
    }

    if (libJarNames.size() > 0) {
        def libJars = project.files(libJarNames.collect{
            new File(rootSmall.preLibsJarDir, it).path
        })
        mLibraryJars.addAll(libJars.files)
    }

    mLibraryJars.addAll(libDependentJars)

    // Collect stub and small jars in host
    Set<Project> sharedProjects = []
    sharedProjects.addAll(rootSmall.hostStubProjects)
    if (rootSmall.smallProject != null) {
        sharedProjects.add(rootSmall.smallProject)
    }
    sharedProjects.each {
        def jarTask = it.tasks.withType(TransformTask.class).find {
            it.variantName == 'release' && it.transform.name == 'syncLibJars'
        }
        if (jarTask != null) {
            mLibraryJars.addAll(jarTask.otherFileOutputs)
        }
    }

    rootSmall.hostProject.tasks.withType(TransformTask.class).each {
        if ((it.variantName == 'release' || it.variantName.contains("Release"))
                && (it.transform.name == 'dex' || it.transform.name == 'proguard')) {
            mLibraryJars.addAll(it.streamInputs.findAll { it.name.endsWith('.jar') })
        }
    }

    return mLibraryJars
}

````



## gradle 插件的生命周期


+ 初始化阶段
+ 配置阶段
+ 运行阶段

beforeEvaluate 是指配置阶段开始前做的事情。
afterEvaluate 是指配置阶段后面做的事情。


## small-gradle 插件 嵌入的 打包流程
涉及的 task 有:

+ buildBundle , 依赖 assembleRelease
+ buildLib , 依赖 assembleRelease .并且在 buildLib 的 doLast 调用了方法 RootPlugin.buildLib
+ preBuild ,doFirst 了一个方法 hookPreReleaseBuild
+ JavaC , 这个也是一个 Task ， 获取自 variant.javaCompile ,classesDir 取自这里，应该就是所有类的所在路径地址。
+ process${variantName}Manifest 。processManifest
+ aapt ,Task 组成是 process${variantName}Resources  .可以拿到 apFile 地址。这个 apFile 是包含 res 和 AndroidManifest.xml 以及 resources.arsc 文件的。


````java
  apFile          = aapt.packageOutputFile

  File symbolDir  = aapt.textSymbolOutputDir
  File sourceDir  = aapt.sourceOutputDir
  symbolFile      = new File(symbolDir, 'R.txt')
  rJavaFile       = new File(sourceDir, "${packagePath}/R.java")

  splitRJavaFile  = new File(sourceDir.parentFile, "small/${packagePath}/R.java")
````

这几个 文件含义分别， symbolFile 可以列为符号表， rJavaFile 是 R 文件。目前不知道，为何要分开2个文件。目前看起来 符号表，没必要存在。**undo**  
splitRJavaFile 文件，在 buildLib task 执行后，是这个 project 单独的 资源ID 。
+ MergeResources  ,  是一个 Task
+ MergeSourceSetFolders
+

````
File mergerDir = variant.mergeResources.incrementalFolder
````

### 笔记
#### AppPlugin.prepareSplit 方法
+ 符号表 symbolFile 文件不存在就返回。
+ 用 libEntries 保存 宿主 host 的符号表
+ 用 libEntries 保存 mTransitiveDependentLibProjects 的 public.txt ，应该也是上次保存的 符号表
+ small.publicSymbolFile 指的是这个 project 下面的 publix.txt
+ idsFile (small.symbolFile) ，指的是这个 project 下面的 符号表
+ 在 SymbolParser 的解析 符号表的时候，
 - vtype 表示第一个，一般是 int 或者 int[]  
 - type 表示 类型，dimen anim color 等.
 - key 表示 这个资源的 key ，平时 R.type.key 这样  使用的。
 - typeId 表示 在 R 文件中的 id 的 pp 字段。
 - entryId 表示 typeId 后面的4位。  
 - idStr 表示完整的 id 的字符串，
 - id 就是 idStr 的 Integr 化了。

**几个重要的变量**  

 + publicEntries 就是 project 下面的 符号表  
 + bundleEntries 当前 aaptTask 之后的 符号表
 + libEntries 是宿主和 mTransitiveDependentLibProjects 的 符号表
 + retainedPublicEntries 在当前 bundleEntries  还保持的 publicEntries
 + retainedEntries 在当前 bundleEntries 中的符号表，去除了 publicEntries 部分。
 + retainedStyleables 类似 retainedEntries ，不过他仅仅包含了 styleable 部分。
 + staticIdMaps.put(bundleEntry.id, libEntry.id)
 + staticIdMaps 会在 重造 resId 之后，staticIdMaps.put(e.id, newResId)
 + staticIdStrMaps.put(bundleEntry.idStr, libEntry.idStr)
 + staticIdStrMaps 同样，会在重造 resId 之后，staticIdStrMaps.put(e.idStr, newResIdStr)
 + vendorEntries 指的是 aar 中的 符号表
 + vendorStyleableKeys 指的是 aar 中的 styleable 部分。
 + allTypes 包含 libEntries 、retainedTypes 和 vendorEntries 的。名字冲突部分，已经处理了。
 + allStyleables ，类似 allTypes .

##### **疑问**
###### small.publicSymbolFile 和 small.symbolFile 分别是什么时候生成的。
###### project 的 public.txt 的 unusedTypeIds 以及 unusedEntryIds 是怎么产生的
unusedTypeIds 是 public.txt 中 typeId 不是从 0 开始就会产生这个问题。但是 需要 继续看 public.txt 为什么会产生这个问题。 unusedEntryIds 问题的产生类似。
###### transitiveVendorAars 和 mTransitiveDependentLibProjects 的区别和联系

````java
staticIdMaps
small.idMaps = staticIdMaps  
small.idStrMaps = staticIdStrMaps
small.retainedTypes = retainedTypes
small.retainedStyleables = retainedStyleables

small.allTypes = allTypes
small.allStyleables = allStyleables

small.vendorTypes = vendorTypes
small.vendorStyleables = vendorStyleables
````

#### AppPlugin.hookAapt 方法

### 疑问点

+ rootSmall.preLinkJarDir 什么时候赋值的。
+ AppPlugin 中的 configureReleaseVariant 方法中的 symbolFile rJavaFile 和 splitRJavaFile ，到底是什么关系和区别

symbolFile 是符号表，路径位置的确认，是通过 aapt.textSymbolOutputDir 。
rJavaFile 是 R 文件，  路径位置的确认是通过 aapt.sourceOutputDir
splitRJavaFile ，目前**未知** **undo**

+ 在 LibraryPlugin 的 configureReleaseVariant 方法中，会把 small.symbolFile 复制到  small.publicSymbolFile 。（除去了styleable） .small.symbolFile 路径地址赋值是在  AppPlugin 的 configureReleaseVariant 方法，但是内容什么时候赋值，未知。这些都是在  variant.assemble doLast 做的。
+ small.jar = project.jarReleaseClasses
+ preLibsJarDir  preLinkJarDir  preLinkAarDir 这些变量，起到什么作用了。什么时候赋值的.  
preBaseJarDir  
生成时机：  
RootPlugin 的 buildLib 方法中，把 ext.jar 复制过去了。**undo ，注释说仅仅复制了 R 文件**  
作用：
getLibraryJars 方法的返回值，组成部分。

preLibsJarDir  
是在 LibraryPlugin 的 configureReleaseVariant 方法中，assemble.doLast 中生成的。  
作用是 getLibraryJars 的时候，给返回去。应该是在 buildBundle 的时候用的 。 **undo**

在 RootPlugin 的 build 方法中赋值的。目前作用，未看。
+ RootPlugin 的 buildLib 这个 task ，为什么会触发 app 模块的 build 方法。
HostPlugin 中有 buildLib task 。因此会触发。
+ preBaseJarDir 目录下，可以看到有 app-r.jar ，还有一些 lib 依赖的 库，例如 DevSample 中的 umeng 的。未查明白。 **undo**
+ app. 模块 ，ManiFest 文件，是怎么合并的？**undo**
