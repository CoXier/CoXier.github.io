---
layout:     post
title:      "将ViewPager改造成VerticalViewPager"
author:     "CoXier"
header-img: "https://cdn.ruguoapp.com/FmKSNlibZLA9N3zTL6zHkmLcQecD.jpg"
tags:

- Android
- View
---



网上已经有 [VerticalViewPager](https://github.com/castorflex/VerticalViewPager)，但是是基于 Android support library 19.0 修改的，代码比较老了。具体的思路是将 x -> y，y -> x，left -> top，right -> bottom，但是改动的地方会特别多，ViewPager源代码 3000+ 行，所以需要很仔细和专注。可以使用 compare 工具对比 [VerticalViewPager](https://github.com/castorflex/VerticalViewPager) 和原生的 ViewPager 的 diff，避免改动遗漏。从头到尾试着改了一下，大概需要 2 个小时左右的时间。

# 一、删除无用的接口

为了兼容 ViewPager 的相关接口，比如 OnPageChangeListener、PageTransformer，首先将复制出来的 ViewPager 的 OnPageChangeListener 和 PageTransformer 移除，然后替换成 ViewPager 的相关类。

这一步需要注意的是，PagerAdapter#setViewPagerObserver 的访问权限是 default，所以只能 ViewPager 能访问这个方法，为了解决访问权限的问题，可以反射解决。参考代码如下：

```ja
static Field sObserverField;

static {
    try {
        sObserverField = PagerAdapter.class.getDeclaredField("mViewPagerObserver");
        sObserverField.setAccessible(true);
    } catch (NoSuchFieldException e) {
        throw new RuntimeException(e);
    }
}

void invokeSetPagerObserver(PagerAdapter pagerAdapter, PagerObserver pagerObserver) {
    try {
        sObserverField.set(pagerAdapter, pagerObserver);
    } catch (IllegalAccessException e) {
        throw new RuntimeException(e);
    }
}
```

代码里面的 setViewPagerObserver 全部换成 invokeSetPagerObserver 即可

# 二、实现 ViewPager 的 EdgeEffect

先看看原生的 ViewPager 是如何实现 EdgeEffect 的：

```java
// 最左边的 EdgeEffect
if (!mLeftEdge.isFinished()) {
    final int restoreCount = canvas.save();
    final int height = getHeight() - getPaddingTop() - getPaddingBottom();
    final int width = getWidth();

    canvas.rotate(270);
    canvas.translate(-height + getPaddingTop(), mFirstOffset * width);
    mLeftEdge.setSize(height, width);
    needsInvalidate |= mLeftEdge.draw(canvas);
    canvas.restoreToCount(restoreCount);
}
// 最右边的 EdgeEffect
if (!mRightEdge.isFinished()) {
    final int restoreCount = canvas.save();
    final int width = getWidth();
    final int height = getHeight() - getPaddingTop() - getPaddingBottom();

    canvas.rotate(90);
    canvas.translate(-getPaddingTop(), -(mLastOffset + 1) * width);
    mRightEdge.setSize(height, width);
    needsInvalidate |= mRightEdge.draw(canvas);
    canvas.restoreToCount(restoreCount);
}
```

下面的示意图：

![](https://cdn.ruguoapp.com/Fozr8mSBURd5g4GT11A9kvAB1_LG.jpg)

原本 EdgeEffect 是往下凸出来的，为了实现水平方向的 EdgeEffect，需要先将画布顺时针旋转 270 度，旋转之后 canvas 已经处于屏幕之外了，所以需要沿着 BA 方向平移，平移： height - paddingTop，水平方向不用移动，mFirstOffset = 0。需要思考一下为什么是 height - paddingTop（**这个是错误的**），假设我们不考虑 paddingTop，也就是说 paddingTop = 0 。为了让 EdgeEffect 能完全显示出来，则必须移动 height，因为 EdgeEffect 的宽度是 height（注意方向坐标系的转换）。然后我们再考虑 paddingTop 的问题，期望的是 ViewPager 的 EdgeEffect 不应该被 padding 截断，所以应该往下移动 height + paddingTop，然后取负值 -height - paddingTop，所以看到 Google 也是会写 bug 的（滑稽脸）。

另外 Google 也没有考虑 paddingLeft、paddingRight，EdgeEffect 的宽度应该是 getWidth() - getPaddingLeft() - getPaddingRight()。Fix 之后的原生 ViewPager ：

```java
if (!mLeftEdge.isFinished()) {
    final int restoreCount = canvas.save();
    final int height = getHeight() - getPaddingTop() - getPaddingBottom();
    final int width = getWidth() - getPaddingLeft() - getPaddingRight();

    canvas.rotate(270);
    canvas.translate(-height - getPaddingTop(), mFirstOffset * width + getPaddingLeft());
    mLeftEdge.setSize(height, width);
    needsInvalidate |= mLeftEdge.draw(canvas);
    canvas.restoreToCount(restoreCount);
}
if (!mRightEdge.isFinished()) {
    final int restoreCount = canvas.save();
    final int width = getWidth() - getPaddingLeft() - getPaddingRight();
    final int height = getHeight() - getPaddingTop() - getPaddingBottom();

    canvas.rotate(90);
    canvas.translate(-getPaddingTop(), -(mLastOffset + 1) * width - getPaddingLeft());
    mRightEdge.setSize(height, width);
    needsInvalidate |= mRightEdge.draw(canvas);
    canvas.restoreToCount(restoreCount);
}
```

对比一下原生的 ViewPager 和 fix 之后的 ViewPager：

![](https://cdn.ruguoapp.com/FqflO5e2tGH72MgsVg1430j3kZsd.png)

![](https://cdn.ruguoapp.com/FtWnxvL7I6LSkPTxDhz9C5-8UUHD.png)

看完原生的 ViewPager 之后，再看如何实现 VerticalViewPager 的 EdgeEffect。

```java
if (!mTopEdge.isFinished()) {
    final int restoreCount = canvas.save();
    final int width = getWidth() - getPaddingLeft() - getPaddingRight();
    final int height = getHeight() - getPaddingTop() - getPaddingBottom();

    canvas.translate(getPaddingLeft(), getPaddingTop());
    mTopEdge.setSize(width, height);
    needsInvalidate |= mTopEdge.draw(canvas);
    canvas.restoreToCount(restoreCount);
}
if (!mBottomEdge.isFinished()) {
    final int restoreCount = canvas.save();
    final int height = getHeight() - getPaddingTop() - getPaddingBottom();
    final int width = getWidth() - getPaddingLeft() - getPaddingRight();

    canvas.rotate(180);
    canvas.translate(-width - getPaddingLeft(), -(mLastOffset + 1) * height - getPaddingTop());
    mBottomEdge.setSize(width, height);
    needsInvalidate |= mBottomEdge.draw(canvas);
    canvas.restoreToCount(restoreCount);
}
```

