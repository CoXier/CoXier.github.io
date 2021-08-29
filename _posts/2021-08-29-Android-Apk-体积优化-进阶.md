---
layout:     post
title:      "Apk 体积优化进阶"
author:     "CoXier"
header-img: "https://1.bp.blogspot.com/-cmcRT5BGOTY/YQBKC6asA0I/AAAAAAAAQzg/hZrde9Sgx881Wdf-c__VMkTvsKoVjOwsACLcBGAsYHQ/s0/Arctic_Fox_Splash_2x%2B%25281%2529.png"
tags:

- Android
- Android 编译流程
- 程序员的成长之路
---

# 一、背景
2021 年主要从事 SDK 相关的开发工作，其中一个需求是和一个业务方进行对接，他们的需求非常简单， 只需要拉流+消息轮训两个核心功能，其他的功能都可以删除缩减。

Google 一直推荐的 gradle 工具，是从 Module 的维度进行工程划分、构建的。理想情况下，将仅用到的几个 Module 进行打包，就能满足上面的需求。
但是理想很丰满，现实很骨感，工程架构一直都十分不合理，比如仅要用到 A Module 中的某个类，但是 A Module 中确有大量的类是我们用不到的。

碰巧这个业务接入方，对 gradle plugin 保持开发的态度，于是我有了施展拳脚的地方。

# 二、删除无用的 keep root
Android proguard 阶段会有一系列代码、资源的优化，其中对代码的部分优化是通过 keep root 来实现的。从 keep root 节点出发，就可以知道需要保留多少个 class(可能被混淆也可能没有混淆) 了。

keep root 据我所知有三部分组成：

1. proguard rule 文件中声明的规则
2. AndroidManifest.xml 中声明的 Activity、Service、Provider 等
3. @keep 注解的 class

删除无用的 keep root ，就可以拔出萝卜带出泥，清理很多无用的 class。对于第一部分，没有太多可以说的，我们主要看 2 和 3 部分。

## 2.1 删除 AndroidManifest 无用的 root
我们知道，Android 在构建的时候，会将各个 aar 和 project 下的 AndroidManifest.xml 合并成一个。于是我们可以 Hook `processReleaseManifest` 这个 task。

Hook gradle task 有两种方式：Task 和 Action。如果使用 Task 的话，要注意 Task 的插入顺序，需要用到 finalizedBy 和 dependsOn。如果是 Action 的话，会灵活很多，我们只需要用到 doFirst 和 doLast。

我们这里选择使用 Action 的方式，在 processReleaseManifest 执行完毕后，也就是说插入一个 doLast。删除 AndroidManifest 的节点：

```kotlin
object OptAMHandler {

    fun optAndroidManifest(amFile: File, unUsedActivities: List<String>) {
        val root = XmlParser().parse(amFile)

        val children = root.children()
        val applicationNode = children.firstOrNull { it is Node && it.name() == "application" } as? Node ?: return
        val iterator = applicationNode.iterator()
        while (iterator.hasNext()) {
            val node = iterator.next()
            if (node is Node && node.name() == "activity") {
                (node as Node).attributes().values.forEach { value ->
                    if (unUsedActivities.contains(value)) {
                        println("\tremove $value from AndroidManifest.")
                        iterator.remove()
                    }
                }
            }
        }
        val content = XmlUtil.serialize(root)
        amFile.writeText(content)
    }
}
```
## 2.2 删除 @keep 作用的 class
这部分需要借助 ASM 工具，在 transform 阶段插入一个 transform task，然后读取所有的 class，并将无用的 class，删除掉 @keep 注解。


核心代码如下：

```kotlin
    fun process(inputStream: InputStream): ByteArray? {
        val classReader = ClassReader(inputStream)
        val classNode = ClassNode()
        classReader.accept(classNode, ClassReader.EXPAND_FRAMES)
        val classAnnotations = classNode.invisibleAnnotations?.iterator()

        var shouldRemovedKeep = false
        while (classAnnotations?.hasNext() == true) {
            val classAnnotation = classAnnotations.next()
            if (classAnnotation.desc == "Landroid/support/annotation/Keep;" || classAnnotation.desc == "Landroidx/annotation/Keep;") {
                val className = classNode.name.replace("/", ".")
                if (removeKeepClasses?.contains(className) == true) {
                    println("remove @Keep in $className")
                    shouldRemovedKeep = true
                    classAnnotations.remove()
                }
                break
            }
        }
        if (!shouldRemovedKeep) {
            return null
        }

        val writer = ClassWriter(classReader, ClassWriter.COMPUTE_MAXS)
        classNode.accept(writer)

        return writer.toByteArray()
    }
```
删除完 keep 注解后，我们还需要回写 jar，替换掉 jar 中的 class。

```kotlin
    fun handleClassFromJar(jarInput: File) {
        val jarFile = JarFile(jarInput)
        val enumeration = jarFile.entries()

        val uri: URI by lazy {
            URI.create("jar:file:${jarInput.absolutePath}")
        }

        val zipfs: FileSystem by lazy {
            FileSystems.newFileSystem(uri, HashMap<String, String>())
        }

        var shouldReplace = false

        while (enumeration.hasMoreElements()) {
            val entry = enumeration.nextElement()
            val entryName = entry.name
            if (!entryName.endsWith(".class")) continue
            val inputStream = jarFile.getInputStream(entry)
            val res = process(inputStream, options)
            if (res != null) {
                shouldReplace = true
                val pathInZipfile = zipfs.getPath(entryName)
                Files.copy(ByteArrayInputStream(res), pathInZipfile, StandardCopyOption.REPLACE_EXISTING)
            }
        }

        if (shouldReplace) {
            zipfs.close()
        }
    }
```

**zipfs.close** 非常关键，有两次都是忘了这个，然后查找错误了很久。

# 三、删除无用资源
Android 会在 proguard 阶段，对 drawable 和 layout 等资源做 shrink 处理。shrink 处理是指当探测到某个资源无用时，Android 并不会删除，而是会用一个体积最小的资源进行替换，比如 50B 的图片资源。

虽然 shrink 可以做很大的优化，但是有些情况下并不会，比如：

![](https://gitee.com/coxier/tuchuang/raw/master/uPic/aJQg7J.jpg)

总而言之，如果默认的检测模式下，大概率会因为代码中用到的 `Resources.getIdentifier` 而保留很多不会用到的资源，从而此优化达不到效果。

我们无法要求接入方使用严格的检测模式，所以我们能做的是，删除我们确保不会用到的 sdk 资源。我们知道 android 编译流程中，aapt 会先对资源进行编译，再进行链接，最后 shrink 优化。为了不干扰接入方对资源的处理流程，我们选择在 package 之前做删除操作。

packageRelease 任务会将 dex、resource、assert、so 进行打包，在这一阶段我们可以很方便拿到 resource。resource 存在的形式是 `.ap_`，本质上也是一个 zip。核心代码如下：

```kotlin
# intermediates/res_stripped/release/resources-release-stripped.ap_
    fun removeRes(input: File, shrinkRes: ShrinkRes) {
        val originalSize = Files.size(input.toPath())
        val jarFile = JarFile(input)
        val enumeration = jarFile.entries()
        val uri: URI by lazy {
            URI.create("jar:file:${input.absolutePath}")
        }

        val properties = mapOf(
            "create" to "false"
        )
        val zipfs: FileSystem by lazy {
            FileSystems.newFileSystem(uri, properties)
        }

        while (enumeration.hasMoreElements()) {
            val entry = enumeration.nextElement()
            val entryName = entry.name
            val fileName = entryName.split("/").last()
            shrinkRes.prefix?.forEach {prefix ->
                if (fileName.startsWith(prefix)) {
                    val pathInZipFile = zipfs.getPath(entryName)
                    println("delete $entryName")
                    Files.delete(pathInZipFile)
                }
            }
        }

        zipfs.close()

        val afterOptSize = Files.size(input.toPath())
        println("original size: $originalSize, after PureLiveRemoveRes, current size is: $afterOptSize")
    }
```