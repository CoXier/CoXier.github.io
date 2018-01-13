---
layout:     post
title:      "View事件分发"
author:     "CoXier"
header-img: "img/post-bg.jpg"
tags:

- 源码解析
- Android
---

# Android Dispatches Touch Event

> 在分析Android系统是如何分发触摸事件之前，我们应该先对整个Android View的层次结构有个清晰的认识

**本次分析的源码为Android API 23**

## Android View
先看图：

![](http://ofqrfk7np.bkt.clouddn.com/DecorView.PNG)

在**ActivityThread#handleLaunchActivity**中启动Activity，之后调用Activity#onCreate方法，一般我们的Activity中会有：

```java
   @Override
   protected void onCreate(Bundle savedInstanceState) {
       super.onCreate(savedInstanceState);
       setContentView(R.layout.activity_main);
   }
```
我们关注的当然是和View有关的方法：`setContentView(...)`，具体实现：

```java
    /**
     * Set the activity content from a layout resource.  The resource will be
     * inflated, adding all top-level views to the activity.
     *
     * @param layoutResID Resource ID to be inflated.
     *
     * @see #setContentView(android.view.View)
     * @see #setContentView(android.view.View, android.view.ViewGroup.LayoutParams)
     */
    public void setContentView(@LayoutRes int layoutResID) {
        getWindow().setContentView(layoutResID);
        initWindowDecorActionBar();
    }
```
从注释中可以看出该方法设置了Activity的内容。`getWindow`返回的是一个`Window`，`Window`是一个`abstract class`,文档解释：

```java
/**
 * Abstract base class for a top-level window look and behavior policy.  An
 * instance of this class should be used as the top-level view added to the
 * window manager. It provides standard UI policies such as a background, title
 * area, default key processing, etc.
 *
 * The only existing implementation of this abstract class is
 * android.view.PhoneWindow, which you should instantiate when needing a
 * Window.
 */
```

可以知道`PhoneWindow`是`Window`唯一的实现，源码在Android Studio上看不到，[传送门](http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/com/android/internal/policy/PhoneWindow.java#163)

```java
   @Override
   public void setContentView(int layoutResID) {
       // Note: FEATURE_CONTENT_TRANSITIONS may be set in the process of installing the window
       // decor, when theme attributes and the like are crystalized. Do not check the feature
       // before this happens.
       if (mContentParent == null) {
           installDecor();
       } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
           mContentParent.removeAllViews();
       }

       if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
           final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                   getContext());
           transitionTo(newScene);
       } else {
           mLayoutInflater.inflate(layoutResID, mContentParent);
       }
       mContentParent.requestApplyInsets();
       final Callback cb = getCallback();
       if (cb != null && !isDestroyed()) {
           cb.onContentChanged();
       }
   }
```
mContentParent是个`ViewGroup`类型，进入installDecor(),由于代码量太大，简化：

```java
private void installDecor() {
    if (mContentParent == null) {
         mContentParent = generateLayout(mDecor);
      }
}
```

传入的参数是`DecorView`，它是`PhoneWindow`的一个内部类，继承FrameLayout。

`generateLayout(mDecor)`:

```java
    // Apply data from current theme.

      TypedArray a = getWindowStyle();

      if (false) {
          System.out.println("From style:");
          String s = "Attrs:";
          for (int i = 0; i < R.styleable.Window.length; i++) {
              s = s + " " + Integer.toHexString(R.styleable.Window[i]) + "="
                      + a.getString(i);
          }
          System.out.println(s);
      }

      ····
      // Inflate the window decor.
       ···
       int layoutResource;
       int features = getLocalFeatures();
       if ((features & (1 << FEATURE_SWIPE_TO_DISMISS)) != 0) {
           layoutResource = R.layout.screen_swipe_dismiss;
       }

       ···
       mDecor.startChanging();

       View in = mLayoutInflater.inflate(layoutResource, null);
       decor.addView(in, new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));
       mContentRoot = (ViewGroup) in;

       ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
       if (contentParent == null) {
           throw new RuntimeException("Window couldn't find content container view");
       }
       ···
```
获取theme的相关值后，根据features的值来给layoutResource赋相应的值，例如平常我们设置`Theme.AppCompat.Light.NoActionBar`之后，这里就会给layoutResource传相应的layout id。
然后加载layoutResource对应的View `in`，并把`in`添加到decor中。`contentParent`对应的Id是ID_ANDOIRD_CONTENT。

再回到 PhoneWindow的`setContentView`的方法中，看到如下一行：

```java
mLayoutInflater.inflate(layoutResID, mContentParent);
```
这里也表明了`mContentParent`承载了整个Activity的内容视图




## How Android Handles Touches

#### MotionEvent
用户所有的触摸操作都被包装成 `MotionEvent`，`MotionEvent`用来描述用户的动作的几个常量：

* MotionEvent.ACTION_DOWN
* MotionEvent.ACTION_MOVE
* MotionEvent.ACTION_UP
* MotionEvent.ACTION_CANCEL

`MotionEvent`描述了 **触摸的位置** 、**触摸的时间** 以及 **触摸点的个数**

任何一个`gesture`都是从`MotionEvent.ACTION_DOWN`开始，以`MotionEvent.ACTION_UP`结束

#### Event Flow
> The touch event is dispatched from top to bottom but handled from bottom to top. The event is dispatched by calling `dispatchTouchEvent` and handled by `onTouchEvent`.

这一部分我们只关注事件流向，而不关注具体的逻辑。
在Android Touch System中，事件分发从`Activity`的`dispatchTouchEvent`开始。

```java
    /**
     * Called to process touch screen events.  You can override this to
     * intercept all touch screen events before they are dispatched to the
     * window.  Be sure to call this implementation for touch screen events
     * that should be handled normally.
     *
     * @param ev The touch screen event.
     *
     * @return boolean Return true if this event was consumed.
     */
    public boolean dispatchTouchEvent(MotionEvent ev) {
        if (ev.getAction() == MotionEvent.ACTION_DOWN) {
            onUserInteraction();
        }
        if (getWindow().superDispatchTouchEvent(ev)) {
            return true;
        }
        return onTouchEvent(ev);
    }
```
如果事件被消费掉了，则返回true。方法中调用了`PhoneWindow`的`superDispatchTouchEvent`，紧接着`PhoneWindow`调用`DecorView`的`superDispatchTouchEvent`。

```java
    public boolean superDispatchTouchEvent(MotionEvent event) {
        return super.dispatchTouchEvent(event);
    }
```
由于`DecorView`继承`FrameLayout`,`FrameLayout`并没有重写`dispatchTouchEvent`，所以这里看`ViewGroup`的`dispatchTouchEvent`，代码太长，就不贴了，此方法可能会调用：

```java
/**
 * Transforms a motion event into the coordinate space of a particular child view,
 * filters out irrelevant pointer ids, and overrides its action if necessary.
 * If child is null, assumes the MotionEvent will be sent to this ViewGroup instead.
 */
private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
        View child, int desiredPointerIdBits)
```
这个方法就会向 child view传递此事件，循环上述的过程直到View不是ViewGroup，而是单个View。

再看`View`的dispatchTouchEvent:

```java
public boolean dispatchTouchEvent(MotionEvent event) {

        ···

        if (onFilterTouchEventForSecurity(event)) {
            //noinspection SimplifiableIfStatement
            ListenerInfo li = mListenerInfo;
            if (li != null && li.mOnTouchListener != null
                    && (mViewFlags & ENABLED_MASK) == ENABLED
                    && li.mOnTouchListener.onTouch(this, event)) {
                result = true;
            }

            if (!result && onTouchEvent(event)) {
                result = true;
            }
        }

        ···
        return result;
    }

```
到这里**分发**结束，如果`mOnTouchListener`不是null，先调用`mOnTouchListener.onTouch()`，之后有**可能**再调用`onTouchEvent（event）`，在`onTouchEvent`中会调用`performClick`或者间接调用`performLongClick`，这个时候就会回调clickListener或者longClickListener。最后返回`result`给他的`ViewGroup`。

同时这段代码也向我们展示了，`mOnTouchListener.onTouch`在`onTouchEvent`之前调用,而且如果前者返回`true`，则`onTouchEvent`就不会调用。
总的来说事件流是大致按照这样的顺序来的：

![](http://ofqrfk7np.bkt.clouddn.com/EventFlow.jpg)

#### Counsume Touch Handling
上面的流程只是某一个情景下的一个示意图，在实际的开发中，如果其中某一个方法（当然还有touchListener的方法）返回true，就会中断之后的传递链。

* 如何处理Touch Event:有两种方法，一种是利用setOnTouchListener、setOnClickListener、setOnLongClickListener，第二种重写onTouchEvent，注意如果使用第一种方法，应该注意返回值。

* 如何消费掉Touch Event：在接收到`MotionEvent.ACTION_DOWN`时就返回true。

#### Intercept Touch Event
`ViewGroup`中有`onInterceptTouchEvent`可以用来截断事件，让自己来处理事件。这一部分可以看[官网](https://developer.android.com/training/gestures/viewgroup.html)
