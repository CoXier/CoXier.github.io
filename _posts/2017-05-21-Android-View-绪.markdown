---
layout:     post
title:      "Android View（绪）"
author:     "CoXier"
header-img: "img/post-bg.jpg"
tags:

- Android
- Android View
---

# 前言
经过一段时间的找实习后，懈怠快一个月了，准备为入职做点准备。断断续续写博客已经快一年了，很惭愧，没有一个能成系列的文章。
之后的初步计划是总结 Android View 系列的知识。

# 一、Android View
Android View 在整个 Andoird Application 中扮演着非常重要的角色。按照我的理解：App 以 Activity 为载体与用户进行交互，例如点击、长按、侧滑。Activity 把可见的视图交给 Window 管理，Window 通过创建一系列 View 来展示 User Interface 。

Android View 的学习包括：
- Custom View（extend base view or ViewGroup）
- Android Performance for View
- TextureView、SurfaceView

## 1.1 Android Custom View
Android Custom View 的知识点很杂，但是不妨从一下几个方面入手：
- Theme and Style
- View's four constructor
- How android draws view
- Paint and Canvas
- Implement animation
- Handle Touch Event

以上是基础中的基础，如果要写出性能较好的自定义View，那么还需要充分了解性能方面的问题。

## 1.2 TextureView、SurfaceView
`TextureView、SurfaceView` 的用途有别于其他原生的 View,他们的适用场景是 视频播放、摄像、拍照，近几年市场已经出现越来越多基于这些 View 的产品，如美图秀秀、Vue视频、抖音，所以学习他们也是十分有必要。因为 Android 底层也是用 OpenGL 来绘图的，学习 `TextureView、SurfaceView` 肯定免不了要学习 OpenGL ,甚至还需要了解 「解码」和「编码」等方面的知识。
