---
layout:     post
title:      "Android View Constructors"
author:     "CoXier"
header-img: "https://cdn.ruguoapp.com/FtIYYF6rnQLvMyloJpnHqtWZ9AkI.jpg"
tags:

- Android
- Android View
---

# 一、Style And Theme
Android View 有四个构造方法，其中两个和 Style，Theme 有关，所以在深入了解 View 的四个构造方法之前，有必要了解一下 Style 和 Theme。

## 1.1 Style
Style 是关于某一个 View 或窗口的属性集合，包括常见的长度、宽度、颜色、字体等属性。下面通过一个简单的布局 xml 来介绍如何使用：

```java
<TextView
    android:layout_width="fill_parent"
    android:layout_height="wrap_content"
    android:textColor="#00FF00"
    android:typeface="monospace"
    android:text="@string/hello" />
```
上面的布局文件定义了 TextView 的长宽、字体颜色、字体类型。如果现在有三个或者多个和该 TextView 类似的 TextView，我们就可以把这些具有**相同属性值**的属性结合为一个 Style。

### 1.1.1 定义 Style
Style 的定义是通过 xml 文件来实现的，这个 xml 文件必须放在 `res/values` 下。对于上面的 TextView，可以定义如下 Style ：

```java
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <style name="CodeFont" parent="@android:style/TextAppearance.Medium">
        <item name="android:layout_width">fill_parent</item>
        <item name="android:layout_height">wrap_content</item>
        <item name="android:textColor">#00FF00</item>
        <item name="android:typeface">monospace</item>
    </style>
</resources>
```
- 任何一个 `<style></style>` 节点都应该在 `resources` 下
- 通过 `name` 来唯一标识一个 Style，相当于 key-value 中的 key
- 前面说到 Style 是属性的集合，集合的一个子元素也就是 `<item></item>`

定义了 Style 之后，可以应用此 Style 到多个 View 上，后期修改属性也减少了重复工作。

```java
<TextView
    style="@style/CodeFont"
    android:text="@string/hello" />
```


### 1.1.2 Style 继承关系
Style 间可以存在继承关系，和面向对象语言中的继承类似，继承之后按照需要，对自己想要的属性进行重新「赋值」。继承分为两种类型：
- 继承 Android 内置 Style，需要使用 `parent` 关键字指明继承的对象，见 `1.1.1`
- 继承自定义 Style，规则上更随意，可采取下面的方式继承 Style:CodeFont

```java
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <style name="CodeFont" parent="@android:style/TextAppearance.Medium">
        <item name="android:layout_width">fill_parent</item>
        <item name="android:layout_height">wrap_content</item>
        <item name="android:textColor">#00FF00</item>
        <item name="android:typeface">monospace</item>
    </style>

    <style name="CodeFont.Red">
    <item name="android:textColor">#FF0000</item>
</style>
</resources>
```
> 除了通过「名字前缀」来表明继承关系，当然也可以通过 `parent` 来声明。

### 1.1.3 Style 属性
相应的 类引用 最便于查找某些 View 的属性，例如上面例子中的 TextView 就可以查找 [TextView 属性表](https://developer.android.com/reference/android/widget/TextView.html?hl=zh-cn#lattrs)。

## 1.2 Theme
Style 的「作用域」是他修饰的 View，ViewGroup 并不能对它的子 View 产生作用。比如下面代码：

```java
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout
    style="@style/CustomLayout"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Hello World!"/>

</RelativeLayout>
```
给 RelativeLayout 指定了 Style，在 `@style/CustomLayout` 中定义了 textSize，但是对其子 TextView 的字体大小并没有起到作用。如果需要应用在 ViewGroup 的 style 起到作用，那么就可以考虑 Theme 了。定义 Theme 和 Style 一样，其实本质而言 Theme 也是 Style。

```java
<style name="CustomLayoutTheme">
    <item name="android:textSize">20sp</item>
</style>
```

```java
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout
    android:theme="@style/CustomLayoutTheme"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Hello World!"/>

</RelativeLayout>
```
这样 CustomLayoutTheme 就可以控制 TextView 的字体大小了。

注意：**Theme 下的 item 颜色属性仅支持对另一资源的引用，不支持字面常量。**

```java
<color name="custom_theme_color">#b0b0ff</color>
<style name="CustomTheme" parent="android:Theme.Light">
    <item name="android:windowBackground">@color/custom_theme_color</item>
    <item name="android:colorBackground">@color/custom_theme_color</item>
</style>
```
# 二、View Constructors
View 有4个构造方法，经常用到的是下面两个：

```java
/**
 * Simple constructor to use when creating a view from code.
 *
 * @param context The Context the view is running in, through which it can
 *        access the current theme, resources, etc.
 */
public View(Context context)
```
此构造方法适用于 java 代码动态创建 View，创建时只需要指定当前的上下文 Context，如 `View view = new View(context)`

```java
// Constructor that is called when inflating a view from XML.
public View(Context context, @Nullable AttributeSet attrs)
```
此构造方法用于从布局文件创建 View。在这个构造函数中，出现了 AttributeSet，从表面意思上来看这个参数代表了属性集合，在了解 AttributeSet 前先了解一下 Attribute。

## 2.1 Attribute
「技术支撑业务」，每一个自定义 View 都有着特定的业务需求场景，需要一些特定的 Attribute 来满足业务需求，例如动画的时间、圆环的宽度等等。在日常的开发中，我们会经常用到诸如 TextView，ImageView 等基础控件，以 ImageView 为例，我们可以在 xml 中很方便设置 `android:src="xxx"`，那你可曾想过为什么在 xml 文件中可以指定 `src` 等属性的值呢？

Android 源码通过 `<declare-styleable name="ImageView">` 标签声明了 ImageView 的属性。详见： **frameworks/base/core/res/res/values/attrs.xml**

```xml
<declare-styleable name="ImageView">
    <attr name="src" format="reference|color" />
    <attr name="scaleType">
        <enum name="matrix" value="0" />
        <enum name="fitXY" value="1" />
        <enum name="fitStart" value="2" />
        <enum name="fitCenter" value="3" />
        <enum name="fitEnd" value="4" />
        <enum name="center" value="5" />
        <enum name="centerCrop" value="6" />
        <enum name="centerInside" value="7" />
    </attr>
    <attr name="adjustViewBounds" format="boolean" />
    <attr name="maxWidth" format="dimension" />
    <attr name="maxHeight" format="dimension" />
    <attr name="tint" format="color" />
    <attr name="baselineAlignBottom" format="boolean" />
    <attr name="cropToPadding" format="boolean" />
    <attr name="baseline" format="dimension" />
    <attr name="drawableAlpha" format="integer" />
    <attr name="tintMode" />
</declare-styleable>
```
Android 系统在 make build 的过程中，会生成 `com.android.internal.R.styleable.ImageView`，每一个 Attribute Item 生成 `R.styleable.ImageView_xxx` ，如 src 属性就会被表示成 `R.styleable.ImageView_src`。注意其实 attribute 的定义不一定需要在 &lt;declare-styleable&gt; 定义。

> 上面的代码对于属性的定义很规范也很全面，之后写自定义 View 非常值得借鉴。


## 2.2 AttributeSet
形如下面的布局文件，定义了基本的宽高等属性，如何获取这些属性的值呢？我们在代码中无法直接获取，Android 将这些属性值通过 AttributeSet 传递。AttributeSet 是属性值的集合，Attribute 是集合中每一个 item 的 key。

```java
<ImageView
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:src="xx"/>
```
以获取 `android:src` 为例，我们首先可以获取 TypedArray，然后通过 index (com.android.internal.R.styleable.ImageView_src) 来获取 `src` 属性值。

```java
final TypedArray a = context.obtainStyledAttributes(
                attrs, com.android.internal.R.styleable.ImageView, defStyleAttr, defStyleRes);
Drawable d = a.getDrawable(com.android.internal.R.styleable.ImageView_src);
.....
a.recycle();
```
## 2.3 int defStyleAttr
在第三、四个构造方法中，有一个参数 `int defStyleAttr`。文档对其的解释是：

> attribute in the current theme that contains a reference to a style resource that supplies default values for the view. Can be 0 to not look for defaults.

从文档来看，我认为 defStyleAttr 是当前 Theme 中的一个 attribute，指向某个 Style 资源。在日常开发中我们传入 `defStyleAttr = 0` 就能满足我们自定义 View 的需求，但是在有些情况下并不是如此。下面结合我在实际的开发中遇到的问题，谈谈 defStyleAttr。

support-v7 SwitchCompat 从 24 版本开始支持 `thumbTint` 和 `trackTint`，而 support-v7-23 未支持，如果要在 23 版本中支持这两个属性，我选择的做法是自定义 View 继承 SwitchCompat，模仿 24 版本对这两个属性进行处理。最开始我是这样写的：

```java
public SupportedSwitchCompat(Context context) {
    this(context, null);
}

public SupportedSwitchCompat(Context context, AttributeSet attrs) {
    this(context, attrs, 0);
}

public SupportedSwitchCompat(Context context, AttributeSet attrs, int defStyleAttr) {
    super(context, attrs, defStyleAttr);
    init();
}

private void init(){
    mThumbDrawable = getThumbDrawable();
    mTrackDrawable = getTrackDrawable();
    applyTint();
}
```
`getThumbDrawable` 和 `getTrackDrawable` 是父类 SwitchCompat 的方法，整个逻辑看上去没有问题，但是代码跑起来后，发现视觉效果缺少了好多元素，通过 debug，发现 mThumbDrawable 和 mTrackDrawable 都是 null。回过头来，再看 SwitchCompat 源码：

```java
public SwitchCompat(Context context, AttributeSet attrs) {
    this(context, attrs, R.attr.switchStyle);
}
```

之前说过 defStyleAttr 是一个 attribute，所以这里的 `R.attr.switchStyle` 也是一个 attribute，Android 源码对 switchStyle 的声明详见 **frameworks/base/core/res/res/values/attrs.xml**

```java
<declare-styleable name="Theme">
	<!-- Default style for the Switch widget. -->
    <attr name="switchStyle" format="reference" />
</declare-styleable>
```

也和之前 defStyleAttr 的文档解释遥相呼应——defStyleAttr 的类型是一个 reference。

定义了 switchStyle，那该属性赋值在什么地方呢？详见：**frameworks/base/core/res/res/values/themes.xml**

```java
<style name="Theme">
	<item name="switchStyle">@style/Widget.CompoundButton.Switch</item>
</style>
```

通过指定 defStyleAttr，在获取 TypedArray 时就可以拿到当前 Theme 中关于此 SwitchCompact 的一系列属性值。通过这个 bug，以后自定义 View 的时候，如果没有特殊需求，可以尽量避免 telescoping constructor。从功能层面来说，defStyleAttr 是用来指定 style 资源的，但是因为需要定义一个 attribute，所以操作步骤看起来稍显复杂，如果想要在代码中**直接**指定 style 资源，我们可以使用 View 构造方法的第四个参数 `int defStyleRes`。

## 2.4 int defStyleRes

第四个构造方法在 API 21 上才能使用，但是如果在低版本想要用 `int defStyleRes`，只需要手动调用 `obtainStyledAttributes`，传入 defStyleRes。`int defStyleRes` 参数只有在 `int defStyleAttr=0 ` 时才能起作用，用来直接指定一个 style 资源，比如：

```java
<style name="ImageViewStyle">
    <item name="android:src">@drawable/xx</item>
</style>
```

然后在获取 TypedArray 时传入即可：

```java
TypedArray a = context.getTheme().obtainStyledAttributes(attrs,R.styleable.CustomImageView, 0, R.style.ImageViewStyle);
```

从功能上来看，第三个参数和第四个参数有些交集，甚至初次看来有一点重复，觉得第三个参数有点多余。其实不然，第三个参数强调的是在 Theme 中指定某种 View 的 Style，强调的是View 和 Style 之间的关联关系，比起第四个参数 defStyleAttr 更加的“宏观”。
