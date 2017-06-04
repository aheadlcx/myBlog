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

## gradle 插件的生命周期


+ 初始化阶段
+ 配置阶段
+ 运行阶段

beforeEvaluate 是指配置阶段开始前做的事情。
afterEvaluate 是指配置阶段后面做的事情。
