---
title: 'Android: Gradle Plugin 实现编译过程中 MainDex 文件的方法数打印'
date: 2021-03-19 17:34:40
tags: Android, Gradle
---

### 前言

本次需求是实现对 MainDex 文件的方法数的打印，避免项目遇到方法数超过 65536 的问题，需要提前对 Dex 文件的方法数预警。大部分 Android 开发者都知道一旦出现 Dex 方法数超过限制，按照[官方的建议配置](https://developer.android.com/studio/build/multidex)就可以解决问题，但对 Classes.dex 文件生成流程可能都会忽略。根据这次的需求，我们就来深入探索一下整体的构建流程吧。

### 复现 64K 引用限制场景

首先得造一个能够复现问题的环境，最直接快捷的办法就是新建一个 Demo 项目，添加一堆第三方依赖库，三方库的代码会一并被编译构建为 apk，把 `minSdkVersion` 设置到 20 或更低版本，在 Gradle 构建的过程中如果某个 Dex 文件超过限制，就会出现构建异常，并有相应的异常信息提示。

``` java
AGPBI: {"kind":"error","text":"Cannot fit requested classes in a single dex file (# methods: 71086 > 65536)","sources":[{}],"tool":"D8"}
com.android.builder.dexing.DexArchiveMergerException: Error while merging dex archives: 
The number of method references in a .dex file cannot exceed 64K.
Learn how to resolve this issue at https://developer.android.com/tools/building/multidex.html
	at com.android.builder.dexing.D8DexArchiveMerger.getExceptionToRethrow(D8DexArchiveMerger.java:132)
	at com.android.builder.dexing.D8DexArchiveMerger.mergeDexArchives(D8DexArchiveMerger.java:119)
	at com.android.build.gradle.internal.transforms.DexMergerTransformCallable.call(DexMergerTransformCallable.java:102)
	at com.android.build.gradle.internal.tasks.DexMergingTaskRunnable.run(DexMergingTask.kt:432)
... more
-------------------------------------------------------------------------
Caused by: com.android.tools.r8.utils.b: Cannot fit requested classes in a single dex file (# methods: 71086 > 65536)
	at com.android.tools.r8.utils.T0.error(SourceFile:1)
	at com.android.tools.r8.utils.T0.a(SourceFile:2)
	at com.android.tools.r8.dex.P.a(SourceFile:746)
	at com.android.tools.r8.dex.P$h.a(SourceFile:7)
	at com.android.tools.r8.dex.b.a(SourceFile:14)
	at com.android.tools.r8.dex.b.b(SourceFile:25)
	at com.android.tools.r8.D8.d(D8.java:133)
	at com.android.tools.r8.D8.b(D8.java:1)
	at com.android.tools.r8.utils.Y.a(SourceFile:36)
	... 38 more
```

关键的 Log 信息如下，顺着异常信息开始寻找源码对应的出处。

- `The number of method references in a .dex file cannot exceed 64K.`
- `Cannot fit requested classes in a single dex file (# methods: 71086 > 65536)`

报错出自  `D8DexArchiveMerger.java` 132 行，然而 Android Studio 中直接搜索该类无果，在 [Android Code Search](https://cs.android.com/android-studio/platform/tools/base/+/mirror-goog-studio-master-dev:build-system/builder/src/main/java/com/android/builder/dexing/D8DexArchiveMerger.java;l=47?q=d8dexarchive&sq=) 看了下是在 build-system 中的，那就准备下载源码到本地看看。

先是在 GitHub 找到的这个库 [adt-tools-base](https://github.com/korli/adt-tools-base.git) 汇总了 build 相关的代码库，后来了解到其实这部分代码就是 Android 项目根目录下 `build.gradle` 文件中依赖的 `Android Gradle Plugin`，这里就涉及到了一个比较模糊的概念 **Android Build Tools 和 Gradle 的关系**：

- **Gradle**：我们通常用到的 Gradle 的版本，是在 `gradle-wrapper.properties` 中指定的：

```groovy
#Fri Mar 19 00:27:32 CST 2021
distributionBase=GRADLE_USER_HOME
distributionPath=wrapper/dists
zipStoreBase=GRADLE_USER_HOME
zipStorePath=wrapper/dists
distributionUrl=https\://services.gradle.org/distributions/gradle-6.7.1-all.zip
```

- **Android Gradle Plugin**：实际上是专为 Android 编写的一个 Gradle 插件，这也是为什么我们需要在项目根目录的 build.gradle 中将其作为依赖引用，这和 Gradle 引用其他插件的方式一致：

```groovy
// project/build.gradle
buildscript {
    dependencies {
      	// Android Gradle Plugin 版本，该版本号通常和 Android Studio 的版本号一致
        // 也可以独立于项目中，后面实现插件的时候会用到
        classpath 'com.android.tools.build:gradle:4.1.2'
    }
}
```

解决了上面的疑问后，我们回到主线，既然要解决打印 DEX 方法数，就涉及到了本次需求的概念问题：

### 概念问题

- 什么是 DEX

  > 类似 JVM 解析 class 文件，我们编写的 .java/.kt/.groovy 文件编译器最终都会编译为 .class 文件，而 Android 中又最终会通过工具把 .class 文件转为 DEX 字节码文件供 ART/Dalvik 虚拟机使用，ART 还会涉及到 dex2oat

- [什么是 D8](https://developer.android.com/studio/command-line/d8)

  > `d8` 是一种命令行工具，Android Studio 和 Android Gradle 插件使用该工具来将项目的 Java 字节码编译为在 Android 设备上运行的 DEX 字节码，该工具支持您在应用的代码中使用 Java 8 语言功能。
  >
  > `d8` 还作为独立工具纳入了 Android 构建工具 28.0.1 及更高版本中：`android_sdk/build-tools/version/`。

- [什么是 R8](https://stackoverflow.com/questions/49549187/difference-between-d8-and-r8-android#:~:text=D8%2D%3ED8%20is%20a%20dexer,byte%20code%20to%20dex%20code.&text=class%20bytecode%20into%20.,impacts%20your%20app's%20build%20time%2C%20.)

  > R8 其实有两种概念：**R8 工具** 和 **R8 项目**
  >
  > R8 工具：实际是 R8 Shrinker/Compiler，用于优化、压缩 java 字节码的工具
  >
  > R8 项目：实际是 Google 开源的包含了 D8 工具和 R8 工具的项目

- [Android 的编译流程](http://tools.android.com/tech-docs/new-build-system/build-workflow)

  >一张图胜过千言万语：
  >
  >![Android Build Process](/Users/rmy/Desktop/Android Build Process.svg)

- [如何实现自定义的 Gradle Plugin](https://docs.gradle.org/current/userguide/custom_plugins.html)

  > 实现 Gradle Plugin 有三种方式：
  >
  > - 方式 1：在 Build.gradle 脚本中
  > - 方式 2：BuildSrc 项目
  > - 方式 3：单独的 Plugin 形态
  >
  > 具体的插件实现方式 Gradle 官网的手册已经很简洁易懂了。目前我用的是第 2 种方式 `buildSrc`，后续会以单独的项目的方式抽离出来上传到 maven

在了解了相关的概念之后，大概的方案也能想到几个：

### 方案设计

- 方案 1：编译过程中分析 -- 直接修改 D8.jar 源码实现
- 方案 2：编译过程后分析 -- 提取 apk 中的 Dex 文件
- 方案 3：编译过程中分析 -- Hook 相对应的 Gradle 构建 task 

因为时间按照 2~3 天规划的，方案的可行性是很重要的，首当其冲的是要先实现普通 Dex 文件的方法打印，再过滤 Main Dex 文件打印即可，于是也决定了我们的核心问题：

### 核心问题

- i. 如何解析 DEX 文件，实现方法数的打印
- ii. 如何找到编译 DEX 文件的切入点，哪个**流程**和**时间**取到 DEX 文件

### i. 如何解析 DEX 文件，实现方法数的打印

我当时能够想到的是先了解了 DEX 文件结构看是否能有线索，因为既然能通过 D8 工具编译为 DEX 文件，那文件结构的代码应该能在其中探个究竟，果不其然被我找到了，并且发现了新大陆，举个例子，在 Android/SDK/tools/lib 中有很多很实用的工具都是以 jar 的工具包形式存在的，那么对应这些工具包的源码往往能够帮助我们解决很多问题。而且后来翻[官方文档](https://developer.android.com/studio/build/apk-analyzer)的时候，在 User Guide 中也提到了相关工具的使用。然后再品品，**凡是 Android Studio 有的功能，理论上来说我们都是可以通过这种方式去找到相关工具库**，只要是开源的总是可以找到的。接着让我惊喜的事情一件又一件：

1. 从 `DexDisassembler.java` 类中可以看到 `DexBackedDexFile.java`  

2. 通过 `DexBackedDexFile.java` 我找到了[smali](https://github.com/JesusFreke/smali/blob/master/dexlib2/src/main/java/org/jf/dexlib2/dexbacked/DexBackedDexFile.java) 

3. 我在 smali 源码中找到了 Dex 文件的解析实现 `DexFileFactory.java`，并将相关代码拆了出来，通过 `HeaderItem.java` 这个类可以实现 Dex 文件结构信息的获取，拿到 Dex 的大部分信息：

   ```java
   getClassCount
   getClassOffset
   getFieldCount
   getHeaderSize
   getMagicForApi
   getMagicForDexVersion
   getMapOffset
   getMethodCount
   getVersion
   ...
   ```

这样就解决了 Dex 文件解析的问题，并且 `DexFileFactory` 同时支持 `.dex/.apk/oat` 文件的解析😁。

### ii. 如何找到编译 DEX 文件的切入点，哪个**流程**和**时间**取到 DEX 文件

关于这个问题，就得撸源码，硬着头皮看源码实现，找到 Gradle building 的入口实现类`TaskManager.java` 

类中可以看到几个关键的方法：

- `createPostCompilationTasks()` 

  > ```java
  > /**
  >  * Creates the post-compilation tasks for the given Variant.
  >  *
  >  * <p>These tasks create the dex file from the .class files, plus optional intermediary steps
  >  * like proguard and jacoco
  >  */
  > ```

  该方法内部包含了很重要的几个点：

  - **对自定义 Transform List 的处理**，AGP 1.5 以后提供了 Transform API 给开发者用于插入很多预处理方法，以实现编译过程中对 class 文件的改动，这个过程是发生在 class 文件转换为 Dex 文件之前的
  - **Shrinking 环节**
  - **脱糖环节**
  - **确定 Multi-Dex 的类型**，MultiDex 有三种类型，定义在 `com.android.builder.dexing.DexingType` 枚举类中
    - MONO_DEX：不启用 multidex，最终只会生成一个 DEX 文件
    - LEGACY_MULTIDEX：启用 multidex，min sdk 版本 < 21，会有多个 Dex 文件，命名规则：classes.dex, classes2.dex, classes3.dex …
    - NATIVE_MULTIDEX：启用 multidex，min sdk 版本 >= 21

	>*注：在 `DexArchiveMerger.java` 的 `mergeDexArchives()` 方法中也有相关的解释：*
	>
	>```java
	>- if it is {@link DexingType#MONO_DEX}, a single dex file is written, named classes.dex
	>- if it is {@link DexingType#LEGACY_MULTIDEX}, there can be more than 1 dex files. Files
	>are named classes.dex, classes2.dex, classes3.dex etc. In this mode, path to a file
	>containing the list of classes to be placed in the main dex file must be specified.
	>- if it is {@link DexingType#NATIVE_MULTIDEX}, there can be 1 or more dex files.
	>```

- `createDexTasks(){ createDexMergingTasks() }`

  > ```java
  > /**
  >  * Creates tasks used for DEX generation. This will use an incremental pipeline that uses dex
  >  * archives in order to enable incremental dexing support.
  >  */
  > ```

方法内部通过判断 dexingType 类型后，通过 `taskFactory.register(configAction: DexMergingTask.CreationAction)` 唤起 DexMergingTask，而这个 `DexMergingTask.kt` 就是我们这次切入点的主角。在 App 构建过程中，我们能够看到几个和 dex 相关的 task：

- mergeDex${variant}
- mergeExtDex${variant}
- mergeLibDex${variant}
- mergeProjectDex${variant}

这几个 task 其实就是 DexMergeTask，只是在 Gradle 中的任务名是根据职能命名的。通过下面的源码分析流程，我们就可以得到想要的答案了：

```kotlin
// com/android/build/gradle/internal/tasks/DexMergingTask.kt
override fun doTaskAction(inputChanges: InputChanges) {
    // TODO(132615300) Make this task incremental
  	// 在这里调用了 DexMergingTaskRunnable
    getWorkerFacadeWithWorkers().use {
        it.submit(
            DexMergingTaskRunnable::class.java,
            DexMergingParams(
                dexingType.get(),
                errorFormatMode.get(),
                dexMerger.get(),
                minSdkVersion.get(),
                debuggable.get(),
                mergingThreshold.get(),
                mainDexListFile.orNull?.asFile,
                dexFiles.files,
                fileDependencyDexFiles.orNull?.asFile,
                outputDir.get().asFile
            )
        )
    }
	  //... 代码省略
}

/** Delegate for [DexMergingTask]. It contains all logic for merging dex files. */
// 注释说的很清楚了，这是 DexMergingTask 的代理类，包含了 dex 文件合并的所有逻辑 
class DexMergingTaskRunnable @Inject constructor(
    private val params: DexMergingParams
) : Runnable {

    override fun run() {
        //... 代码省略
        var processOutput: ProcessOutput? = null
        try {
            processOutput = outputHandler.createOutput()
		        //... 代码省略
            val allDexFiles = lazy { getAllRegularFiles(dexFiles) }
            if (dexFiles.size >= params.mergingThreshold
                || allDexFiles.value.size >= params.mergingThreshold) {
              	// 关键，内部会判断是 D8 还是 DX，最终确定 ArchiveMerger，见 69 行
                DexMergerTransformCallable(
                    messageReceiver,
                    params.dexingType,
                    processOutput,
                    params.outputDir,
                    dexFiles.map { it.toPath() }.iterator(),
                    params.mainDexListFile?.toPath(),
                    forkJoinPool,
                    params.dexMerger,
                    params.minSdkVersion,
                    params.isDebuggable
                ).call()
            } else {
              //... 代码省略
            }
        } catch (e: Exception) {
        	//... 代码省略
        } finally {
          //... 代码省略
        }
    }
}
// ---------------------------------------------------------------------------

// com/android/build/gradle/internal/transforms/DexMergerTransformCallable.java
/**
 * Helper class to invoke the {@link com.android.builder.dexing.DexArchiveMerger} used to merge dex archives.
 */
public class DexMergerTransformCallable implements Callable<Void> {
		//... 代码省略
    @Override
    public Void call() throws Exception {
        DexArchiveMerger merger;
        switch (dexMerger) {
            case DX:
                DxContext dxContext =
                        new DxContext(
                                processOutput.getStandardOutput(), processOutput.getErrorOutput());
                merger = DexArchiveMerger.createDxDexMerger(dxContext, forkJoinPool, isDebuggable);
                break;
            case D8:
                int d8MinSdkVersion = minSdkVersion;
                //... 代码省略
                merger =
                        DexArchiveMerger.createD8DexMerger(
                                messageReceiver, d8MinSdkVersion, isDebuggable, forkJoinPool);
                break;
            default:
                throw new AssertionError("Unknown dex merger " + dexMerger.name());
        }

        merger.mergeDexArchives(dexArchives, dexOutputDir.toPath(), mainDexList, dexingType);
        return null;
    }
		//... 代码省略
}

// com/android/builder/dexing/D8DexArchiveMerger.java
final class D8DexArchiveMerger implements DexArchiveMerger {
	  //... 代码省略
    @Override
    public void mergeDexArchives(
            @NonNull Iterator<Path> inputs,
            @NonNull Path outputDir,
            @Nullable Path mainDexClasses,
            @NonNull DexingType dexingType)
            throws DexArchiveMergerException {
        //... 代码省略
        // 看到这里是不是很熟悉了，这就是方法数超长报错的 Diagnostics Handler，见 154 行
        D8DiagnosticsHandler d8DiagnosticsHandler = new InterceptingDiagnosticsHandler();
        D8Command.Builder builder = D8Command.builder(d8DiagnosticsHandler);
        builder.setDisableDesugaring(true);
        builder.setIncludeClassesChecksum(compilationMode == CompilationMode.DEBUG);

        for (Path input : inputsList) {
            try (DexArchive archive = DexArchives.fromInput(input)) {
                for (DexArchiveEntry dexArchiveEntry : archive.getFiles()) {
                    builder.addDexProgramData(
                            dexArchiveEntry.getDexFileContent(),
                            D8DiagnosticsHandler.getOrigin(dexArchiveEntry));
                }
            } catch (IOException e) {
                throw getExceptionToRethrow(e, d8DiagnosticsHandler);
            }
        }
        try {
            if (mainDexClasses != null) {
                builder.addMainDexListFiles(mainDexClasses);
            }
            builder.setMinApiLevel(minSdkVersion)
                    .setMode(compilationMode)
                    .setOutput(outputDir, OutputMode.DexIndexed)
                    .setDisableDesugaring(true)
                    .setIntermediate(false);
          	// 关键调用
            D8.run(builder.build(), forkJoinPool);
        } catch (CompilationFailedException e) {
            throw getExceptionToRethrow(e, d8DiagnosticsHandler);
        }
    }

    @NonNull
    private DexArchiveMergerException getExceptionToRethrow(
            @NonNull Throwable t,
            D8DiagnosticsHandler d8DiagnosticsHandler) {
        StringBuilder msg = new StringBuilder("Error while merging dex archives: ");
        for (String hint : d8DiagnosticsHandler.getPendingHints()) {
            msg.append(System.lineSeparator());
            msg.append(hint);
        }
        return new DexArchiveMergerException(msg.toString(), t);
    }


    private class InterceptingDiagnosticsHandler extends D8DiagnosticsHandler {
        public InterceptingDiagnosticsHandler() {
            super(D8DexArchiveMerger.this.messageReceiver);
        }

        @Override
        protected Message convertToMessage(Message.Kind kind, Diagnostic diagnostic) {

            if (diagnostic.getDiagnosticMessage().startsWith(ERROR_MULTIDEX)) {
              	// 这个就是方法超长提示的 String Message 见 178 行
                addHint(DexParser.DEX_LIMIT_EXCEEDED_ERROR);
            }

            if (diagnostic instanceof DuplicateTypesDiagnostic) {
                addHint(diagnostic.getDiagnosticMessage());
                addHint(ERROR_DUPLICATE_HELP_PAGE);
            }

            return super.convertToMessage(kind, diagnostic);
        }
    }
}

// com/android/ide/common/blame/parser/DexParser.java
public class DexParser implements PatternAwareOutputParser {
  	public static final String DEX_LIMIT_EXCEEDED_ERROR =
            "The number of method references in a .dex file cannot exceed 64K.\n"
                    + "Learn how to resolve this issue at "
                    + "https://developer.android.com/tools/building/multidex.html";

}

```

上面这串代码比较长，但是包含了整个 DexMergingTask 和引用类的核心逻辑代码块，我们再简单梳理一下流程：

1. 第 4 行，DexMergingTask 将 DexMergingTaskRunnable 整个 Runnable 提交给 Worker 进行异步处理
2. 第 41 行，DexMergingTaskRunnable 实际执行了 DexMergerTransformCallable.call()
3. 第 91 行，通过判断应该使用哪个 Merger 后（DX或D8）执行 merger.mergeDexArchives()
4. 第 136 行，D8DexArchiveMerger 最终调用了 D8.run()，通过这里把准备好的数据源交给 D8 去处理，也同时会在下方进行回调，整个 MergeTask 就执行完毕了，当这个 Task 执行完成后，我们就可以对其进行 hook 拿到 Dex 文件的输出路径，打印 Dex 文件即可

### 总结

通过这个流程我们得知上述的几个 mergeXXXTask 就是我们的切入点，那么核心问题都解决了，就开始写代码吧，我的想法是实现一个 Gradle 插件，通过插件 Hook 这几个 mergeTask 得到 dex 文件路径，接下来是想要 Main Dex 还是所有文件都打印，或者加入预警代码就都好说了。这时候会发现，实际方案 2、3 我们都已经可以实现了，方案 1 因为涉及到替换开发环境的 jar 包，侵入性太强，不推荐这么做。而方案 3 是我们这次最佳选择。

由于对 Groovy 语法 和 Gradle Plugin 的实现不熟悉，在这里只能参考 Tinker 和其他一些 Plugin 来照葫芦画瓢，一点点把代码逻辑捋清晰后，就实现了编译过程中 Dex 文件的方法数打印，整篇文章也匆匆完结。不过后续还剩下代码重构，各种极端情况的考虑以及接入难度简化等工作需要收尾，再接再厉。

### 代码实现

Gradle Plugin 实现代码：

```groovy
package com.raomengyang.plugin

import com.android.build.gradle.api.ApkVariant
import com.android.build.gradle.internal.tasks.DexMergingTask
import org.gradle.api.Plugin
import org.gradle.api.Project
import org.gradle.api.file.DirectoryProperty
import org.gradle.internal.impldep.org.apache.maven.plugin.PluginExecutionException
import org.jf.DexFileUtil
import org.jf.dexlib2.dexbacked.raw.HeaderItem

class DexParserPlugin implements Plugin<Project> {

    Project project

    @Override
    void apply(Project project) {
        this.project = project
        println "DexParserPlugin start"
        if (!project.plugins.hasPlugin("com.android.application")) {
            throw new PluginExecutionException(
                    "'com.android.application' plugin must be applied", null)
        }
        project.afterEvaluate {
            project.android.applicationVariants.all { ApkVariant variant ->
                project.tasks.findAll {
                    println "all task: ${it.name}"
                }
                checkDexTask(variant, project, "mergeDex")
                checkDexTask(variant, project, "mergeExtDex")
                checkDexTask(variant, project, "mergeLibDex")
                variant.outputs.each { variantOutput ->
                    if (variantOutput != null && variantOutput.getOutputFile() != null && variantOutput.getOutputFile().exists()) {
                        def api = variant.getPackageApplicationProvider().get().getTargetApi()
                        println "api:${api}"
                        // api 需要根据 dex 的编译版本传参
                        // dex 文件的 magic numbers 确认 api 是哪个版本，如： number: 37 -> api: 25
                        HeaderItem headerItem = (HeaderItem) DexFileUtil.loadDexFile(variantOutput.getOutputFile().getAbsolutePath(), api != null ? api : 25)
                        int methodCount = headerItem.getMethodCount()
                        int methodOffset = headerItem.getMethodOffset()
                        int headerSize = headerItem.getHeaderSize()
                        int classCount = headerItem.getClassCount()
                        System.out.println("loadDexFile: apk methodCount: $methodCount, methodOffset:$methodOffset, headerSize:$headerSize, classCount:$classCount")
                    }
                }
            }
        }
    }

    void checkDexTask(ApkVariant variant, Project project, String taskName) {
        def mergeDexTaskName = "$taskName${variant.name.capitalize()}"
        DexMergingTask mergeDexTask = project.tasks.findByName(mergeDexTaskName)
        if (mergeDexTask != null) {
            mergeDexTask.doLast {
                DirectoryProperty outputDir = mergeDexTask.getOutputDir()
                def outputGet = outputDir.get()
                File asFile = outputGet.asFile
                if (asFile.isDirectory()) {
                    String[] dexPaths = asFile.list()
                    dexPaths.each { dexPath ->
                        String dexAbsPath = asFile.getPath() + File.separator + dexPath
                        HeaderItem headerItem = (HeaderItem) DexFileUtil.loadDexFile(dexAbsPath, 25)
                        int methodCount = headerItem.getMethodCount()
                        int methodOffset = headerItem.getMethodOffset()
                        int headerSize = headerItem.getHeaderSize()
                        int classCount = headerItem.getClassCount()
                        System.out.println("loadDexFile:${dexAbsPath},\n methodCount: $methodCount, methodOffset:$methodOffset, headerSize:$headerSize, classCount:$classCount")
                    }
                }
            }
        }
    }
}


```

打印结果：

![WX20210324-205214@2x](/Users/rmy/work/images/WX20210324-205214@2x.png)

### 实现过程中的坑

- 一开始没有代码量较多的编译环境，复现问题比较浪费时间
- 网络环境问题，无法编译 D8.jar

- 其他注意事项

  - Dex 文件的版本与 Android API 对应关系，因为 Dex 文件有版本之分，解析时需要注意对应的版本关系

  ```java
  // Dex version 036 skipped because of an old dalvik bug on some versions
  // of android where dex files with that version number would erroneously
  // be accepted and run. See: art/runtime/dex_file.cc
  
  // V037 was introduced in API LEVEL 24
  public static final byte[] DEX_FILE_MAGIC_v037 =
  "dex\n037\0".getBytes(StandardCharsets.US_ASCII);
  
  // V038 was introduced in API LEVEL 26
  public static final byte[] DEX_FILE_MAGIC_v038 =
  "dex\n038\0".getBytes(StandardCharsets.US_ASCII);
  
  // V039 was introduced in API LEVEL 28
  public static final byte[] DEX_FILE_MAGIC_v039 =
  "dex\n039\0".getBytes(StandardCharsets.US_ASCII);
  
  ```


### 参考

- [我是如何一步一步爬上 “64K限制” 的坑](https://juejin.cn/post/6844904164540022792)

- [Gradle User Guide](https://docs.gradle.org/current/userguide/userguide.html)

- [adt-tools-base](https://github.com/korli/adt-tools-base)

- [Smali](https://github.com/JesusFreke/smali)

- [Tinker](https://github.com/Tencent/tinker)

  

  