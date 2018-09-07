---
layout:     post
title:      "编译 IjkPlayer"
author:     "CoXier"
header-img: "https://cdn.ruguoapp.com/FlZsV6SHbgAkS3bTzJrZ1KGtIP1V.jpg"
tags:

- 视频
- Android
---

## 编译 支持 https 的 IjkPlayer

近期想研究一下几个常见 Player 的解码效率。其中 IjkPlayer 默认是不支持 https 的，所以需要自己手动编译。

### 准备工作

- 配置 ANDROID_SDK ANDROID_NDK

  `export ANDROID_SDK= <PATH>`

  `export ANDROID_NDK= <PATH>`

  注：临时的引用
  
  **建议使用 ndkr10e**

### 下载 FFmpeg 、OpenSSL

- git clone https://github.com/Bilibili/ijkplayer.git ijkplayer-android
- git checkout -B latest kx.x.x
- ./init-android.sh
- ./init-android-openssl.sh

### 编译 OpenSSL

- cd android/contrib 
- ./compile-openssl.sh clean
- ./compile-openssl.sh all

### 编译 FFmpeg

* ./compile-ffmpeg.sh clean
* ./compile-ffmpeg.sh all

### 编译 IjkPlayer

* // 进入 ijkplayer/android


* ./compile-ijk.sh all

完成以上操作后，在 `ijkplayer-android/android/ijkplayer/ijkplayer-armv7a/src/main/libs/armeabi-v7a` 下存在 .so 文件：libijkffmpeg.so  libijkplayer.so   libijksdl.so



