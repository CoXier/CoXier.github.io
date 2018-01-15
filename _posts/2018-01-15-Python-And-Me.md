---
layout:     post
title:      "Python 虚拟环境和包管理"
author:     "CoXier"
header-img: "img/post-bg.jpg"
tags:

- Python
---

# 一、Python 和 我

从「西瓜视频」回校后，一直在做一个自己的小项目——「悦看」，由于项目需要后台，所以学习了一段时间的 python。个人认为每一种计算机编程语言都有着语言自己的特色，比如 c/c++ 对内存的创建回收，Java 高移植性，python 的“人生苦短，我用 python”。

随着 AI 和 CV 的兴起，python 成了新时代程序员的宠儿，这其中我认为有两个原因：1）python 的语法上手极快，代码编写效率极高 2）python 有功能异常强大的 package。python 使用起来确实易用，但是当项目庞大之后，项目代码的可读性会很低而且个人认为不好 debug。对一个新语言的学习，很难说你能马上掌握，除非它是你的第一语言（工作中最常用的语言），但是能借助 google 编程应付基本的功能需要都是足够的。前段时间微信推出了「跳一跳」，网上就有大牛结合图像识别做了一个[辅助工具](https://github.com/wangshub/wechat_jump_game/) ，这更加让我明白编程是为了解决实际问题。

在学习 python 的过程中，我一直都偏向基础的功能实现，而对 python 的一些独有的点都是一扫而过，碰巧学院开始毕业实训了，所以趁着这个机会总结一下。

# 二、Python 虚拟环境

虚拟环境对 python 应用开发是十分方便的。举个例子：应用 A 依赖某个 package 1.1 版本和应用 B 依赖 1.3 版本，那么如果是全局安装该 package 的话就会导致 A 或者 B 有一个运行不成功，此时虚拟环境就能起到作用，应用 A 的虚拟环境安装 1.1 版本，应用 B 的虚拟环境安装 1.3 版本，这样就互不影响了。

## 2.1 virtualenv

Python 目前有两个大版本：2.x 和 3.x ，官方宣布 2.x 将在 2020 年左右正式“退休”，所以之后的使用应该刻意偏向 3.X 。对 2.x 版本，虚拟环境的初始化需要借助 virtualenv。

安装 virtualenv

```bash
pip install virtualenv
sudo /usr/bin/easy_install virtualenv
```

初始化虚拟环境

```b
virtualenv venv
```

venv 就是一个虚拟 Python 环境。

## 2.2 venv

Python 3.x 的虚拟环境要借助 venv ，Python 3.x 自带 venv 无需安装。初始化虚拟环境：

```b
python3 -m venv tutorial-env
```

## 2.3 激活和关闭虚拟环境

* 激活虚拟环境：`source venv/bin/activate`，激活后当前终端就可以使用虚拟环境中的配置了
* 关闭虚拟环境：deactivate

# 三、Python 包管理

pip 可以安装、升级、卸载 Python 包。下面介绍一些 pip 常用的命令：

```ba
# 安装(默认安装最新版本)
pip install novas

# 安装具体的某一个版本
pip install requests==2.6.0

# 升级
pip install --upgrade requests
```

另一些比较少见但是很有用的命令：

```bash
# 展示相信信息
pip show requests

# 显示已经安装的包
pip list
```

强烈推荐生成 requirements.txt 命令：`pip freeze > requirements.txt` 项目中的 requirements.txt 包含了所有的依赖信息，可以通过 `pip install -r requirements.txt` 安装。