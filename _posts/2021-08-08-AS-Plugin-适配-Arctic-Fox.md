---
layout:     post
title:      "AS Plugin 适配 Arctic Fox"
author:     "CoXier"
header-img: "https://1.bp.blogspot.com/-cmcRT5BGOTY/YQBKC6asA0I/AAAAAAAAQzg/hZrde9Sgx881Wdf-c__VMkTvsKoVjOwsACLcBGAsYHQ/s0/Arctic_Fox_Splash_2x%2B%25281%2529.png"
tags:

- 疑难问题追查
- 程序员的成长之路
---

# 程序员 = 工程师
我一直认为，程序员就是工程师。比起一般的工程师，程序员在日常的工作中遇到的问题可能会更多、更难，在解决问题过程中体现出来的就是工程师知识沉淀。

一个资深的程序员在遇到问题时，会不慌不乱，从容不迫，用合理的方式解决各种各样的问题。在最近的博客中，我会用工作中遇到的问题来刻画我是如何一步一步解决那些看上去很难解决的问题的。


# 适配 Arctic Fox 之编译问题
最新版本的 Android Studio 更改了版本的命名方法，和 IDEA 的版本严格保持一致。Intellij 自 `2020.3.1` 开始使用 Java 11 SDK 作为运行环境。

为了适配 Arctic Fox，我首先做的事情是更换 runIde 为最新版本的 AS。

```groovy
androidStudioPath=/Users/jianxinli/Library/Application Support/JetBrains/Toolbox/apps/AndroidStudio/ch-1/203.7583922/Android Studio Preview.app
```

## class file has wrong version 55.0, should be 52.0
修改运行的版本之后，开始编译，果不其然，编译就直接出错了：

```bash
bad class file: /Users/jianxinli/Library/Application Support/JetBrains/Toolbox/apps/AndroidStudio/ch-1/203.7583922/Android Studio Preview.app/Contents/lib/platform-api.jar(com/intellij/ui/ColoredTreeCellRenderer.class)
    class file has wrong version 55.0, should be 52.0
```

从错误信息可以看出来，我们更换运行环境之后，我们依赖的 AS jar 是 55.0 版本，也就是 Java 11，而我们的编译 JDK 版本是 Java 1.8，所以就出错了。

于是接下来要解决的是修改编译环境 JDK 从 8 -> 11，我们直接在 `Project Structure` 中修改 Project SDK:

![](https://gitee.com/coxier/tuchuang/raw/master/uPic/xoIg1j.jpg)

我们其实也可以针对我们需要的 module 进行修改，而不是像上图那样对整个 Project SDK 进行修改。 但是由于我们使用的 Gradle 来构建整个工程，Gradle 默认的 JVM 版本是跟 Project SDK 对齐的。

![](https://gitee.com/coxier/tuchuang/raw/master/uPic/oumJqE.jpg)

## Could not find com.jetbrains.intellij.java:java-compiler-ant-tasks:203.7717.56.2031.7583922
修改完编译环境之后，进行编译，遇到了新的编译问题。对这个错误有点摸不着头脑，于是 Google 搜索，在 github 中找到了相关 issue：https://github.com/JetBrains/gradle-intellij-plugin/issues/369#issuecomment-893311439

总结就是，我们需要声明我们的 compilerVersion。

```groovy
instrumentCode {
    compilerVersion = "203.7717.56"
}
```
修改完成之后，AS 插件工程就可以编译并运行起来了。