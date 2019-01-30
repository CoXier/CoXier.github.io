---
layout:     post
title:      "TextView-Invalidate-Not-Working"
author:     "CoXier"
header-img: "https://cdn.ruguoapp.com/FmKSNlibZLA9N3zTL6zHkmLcQecD.jpg"
tags:

- Android
---

# 一、Problem Background

最近遇到一个问题——TextView 的 Invalidate 没有起作用。问题背景是这样的，实现一个 Loading 动画，Loading 的样式是末尾三个点的数目在不断的变化，一个点、两个点、三个点、零个点。类似下图：

![image-20190127200410606](https://cdn.ruguoapp.com/FrrYz7jLJWuY5Y9BUFwp5K7gaoaC.png)

问题本身不难解决，通过 ValueAnimator 执行动画，然后动态的更新 text 就能实现，但是产品和设计同事希望 TextView 在显示 Loading 时保持**水平居中**，换言之期望的效果就是1. TextView 的左上角坐标不变 2.并且 TextView 宽度也不变。所以想到一个很 tricky 的方法，动态的改变后面三个点的颜色，使之透明。

```java
void init() {
    mTransparentSpan = new ForegroundColorSpan(Color.TRANSPARENT);
    mSpannableString = new SpannableString(mText);
    mTextView.setText(mSpannableString);
}
// 执行动画
void startWaitAnimator() {
    if (mAnimator != null) {
        mAnimator.cancel();
    }
    mAnimator = ValueAnimator.ofInt(0, 4);
    mAnimator.setDuration(2000L);
    mAnimator.setRepeatCount(ValueAnimator.INFINITE);
    mAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
        @Override
        public void onAnimationUpdate(ValueAnimator animation) {
            Object o = animation.getAnimatedValue();
            if (o instanceof Integer) {
                int pos = (Integer) o;
                setForegroundColorSpan(pos);
            }
        }
    });
    mAnimator.start();
}

// 设置三个点的颜色
private void setForegroundColorSpan(int pos) {
    if (pos < 4) {
        mSpannableString.setSpan(mTransparentSpan, mText.length() - 3 + pos, mText.length(), Spannable.SPAN_EXCLUSIVE_EXCLUSIVE);
        mTextView.invalidate();
    }
}
```

虽然我改变了 mSpannableString 并且调用了 invalidate，但是上面的代码是不起作用的，三个点的颜色没有改变。

# 二、Why invalidate not working

通过 Debug 发现，虽然在初始化时设置了 `mTextView.setText(mSpannableString)` ，但是 TextView 的 `mText` 和 mSpannableString 并不是一个对象。看源码：

```java
public final void setText(CharSequence text) {
    setText(text, mBufferType);
}

 private void setText(CharSequence text, BufferType type,
                      boolean notifyBefore, int oldlen) {
   	 ...
     if (type == BufferType.EDITABLE || getKeyListener() != null
         || needEditableForNotification) {
         ....
     } else if (precomputed != null) {
         ...
     } else if (type == BufferType.SPANNABLE || mMovement != null) {
         text = mSpannableFactory.newSpannable(text);
     } else if (!(text instanceof CharWrapper)) {
         text = TextUtils.stringOrSpannedString(text);
     }
     mBufferType = type;
     setTextInternal(text);
 }


private void setTextInternal(@Nullable CharSequence text) {
    mText = text;
    mSpannable = (text instanceof Spannable) ? (Spannable) text : null;
    mPrecomputed = (text instanceof PrecomputedText) ? (PrecomputedText) text : null;
}


```

setText(mSpannableString) 最终会调用四个参数的 setText，默认的 Buffertype 是 `BufferType.NORMAL` ，所以会走到最后一个 else if，text 经过 TextUtils.stringOrSpannedString 转化后，SpannableString 转换成了 SpannedString，所以这就解释了 **mSpannableString 和 TextView 里面的 text 不是同一个对象**，所以设置 mSpannableString 并不会改变 TextView 的内容，因为 mSpannableString 此时和 TextView 没有半毛钱关系。

要解决问题，可以这么在每次动画执行时都去调一次 `mTextView.setText`，但是 setText 会带来不必要的内存开销和代码执行，因为很显然此时我只需要重新 draw 一次。

# 三、Solution

首先了解一下：SpannedString、SpannableString 和 SpannableStringBuilder 三者之间的区别。

|          | SpannedString | SpannableString | SpannableStringBuilder |
| :------: | :-----------: | :-------------: | :--------------------: |
| 内容可变 |      No       |       No        |          Yes           |
| 标记可变 |      No       |       Yes       |          Yes           |

Loading 场景下 SpannableString 就能满足，为了让 TextView 的 Text 也是 SpannableString，所以在初始化设置 setText 时，传入 `TextView.BufferType.SPANNABLE`，注意：**尽管传入的 Buffer 是TextView.BufferType.SPANNABLE，但是 TextView 的 mSpannableFactory 仍然会构造一个新的 SpannableString**，所以执行动画时，可以将 TextView 的 mText 取出来。

```java
private void setForegroundColorSpan(int pos) {
    if (pos < 4) {
        Spannable spannable = (Spannable) mReplayTip.getText();
        spannable.setSpan(mTransparentSpan, mText.length() + pos - 3, mText.length(), Spannable.SPAN_EXCLUSIVE_EXCLUSIVE);
    }
}
```

你可能好奇为什么现在连 invalidate 都不用调用了，因为 TextView#mChangeWatcher 会监听 Span 的变化，一旦变化就会调用 invalidate。看代码 TextView#spanChange：

```java
void spanChange(Spanned buf, Object what, int oldStart, int newStart, int oldEnd, int newEnd) {
    ...
    if (what instanceof UpdateAppearance || what instanceof ParagraphStyle
            || what instanceof CharacterStyle) {
        if (ims == null || ims.mBatchEditNesting == 0) {
            invalidate();
            mHighlightPathBogus = true;
            checkForResize();
        } else {
            ims.mContentChanged = true;
        }
        if (mEditor != null) {
            if (oldStart >= 0) mEditor.invalidateTextDisplayList(mLayout, oldStart, oldEnd);
            if (newStart >= 0) mEditor.invalidateTextDisplayList(mLayout, newStart, newEnd);
            mEditor.invalidateHandlesAndActionMode();
        }
    }
    ...
}
```

what 是 ForegroundColorSpan，继承自 UpdateAppearance，所以会调用 invalidate。