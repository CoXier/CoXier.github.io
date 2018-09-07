---
layout:     post
title:      "自定义CheckBox"
author:     "CoXier"
header-img: "https://cdn.ruguoapp.com/FidDe11rkwVJnIXbZRQij6wc734U.jpg"
tags:

- 源码解析
- Android
- Custom View
---



## 继承View还是CheckBox

要实现的效果是类似![](https://d13yacurqjgara.cloudfront.net/users/107759/screenshots/1631598/onoff.gif)

考虑到关键是动画效果，所以直接继承View。不过`CheckBox`的超类`CompoundButton`实现了`Checkable`，这一点值得借鉴。

下面记录一下遇到的问题，并从源码的角度解决。

## 问题一: 支持 wrap_content
由于是直接继承自View，`wrap_content`需要进行处理。
View measure流程的`MeasureSpec`：


```java
 /**
     * A MeasureSpec encapsulates the layout requirements passed from parent to child.
     * Each MeasureSpec represents a requirement for either the width or the height.
     * A MeasureSpec is comprised of a size and a mode.
     * MeasureSpecs are implemented as ints to reduce object allocation. This class
     * is provided to pack and unpack the &lt;size, mode&gt; tuple into the int.
     */
    public static class MeasureSpec {
        private static final int MODE_SHIFT = 30;
        private static final int MODE_MASK  = 0x3 << MODE_SHIFT;

        /**
         * Measure specification mode: The parent has not imposed any constraint
         * on the child. It can be whatever size it wants.
         */
        public static final int UNSPECIFIED = 0 << MODE_SHIFT;

        /**
         * Measure specification mode: The parent has determined an exact size
         * for the child. The child is going to be given those bounds regardless
         * of how big it wants to be.
         */
        public static final int EXACTLY     = 1 << MODE_SHIFT;

        /**
         * Measure specification mode: The child can be as large as it wants up
         * to the specified size.
         */
        public static final int AT_MOST     = 2 << MODE_SHIFT;

        /**
         * Extracts the mode from the supplied measure specification.
         *
         * @param measureSpec the measure specification to extract the mode from
         * @return {@link android.view.View.MeasureSpec#UNSPECIFIED},
         *         {@link android.view.View.MeasureSpec#AT_MOST} or
         *         {@link android.view.View.MeasureSpec#EXACTLY}
         */
        public static int getMode(int measureSpec) {
            return (measureSpec & MODE_MASK);
        }

        /**
         * Extracts the size from the supplied measure specification.
         *
         * @param measureSpec the measure specification to extract the size from
         * @return the size in pixels defined in the supplied measure specification
         */
        public static int getSize(int measureSpec) {
            return (measureSpec & ~MODE_MASK);
        }
    }
```
从文档注解很容易知道android为了节约内存，设计了MeasureSpec，它由mode和size两部分构成，做这么多终究是为了从父容器向子view传达长宽的要求。mode有三种模式：

- UNSPECIFIED:父容器对子view的宽高无限制
- EXACTLY：父容器已指定了子view确切的宽高
- AT_MOST：父容器指定子view最大的宽高

wrap_content 属于AT_MOST模式。

来看一下大致的measure过程：
在View中首先调用`measure()`在方法中最终调用`onMeasure()`

```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
                getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
    }

```
`setMeasuredDimension`设置view的宽高。再来看看`getDefaultSize()`

```java
public static int getDefaultSize(int size, int measureSpec) {
        int result = size;
        int specMode = MeasureSpec.getMode(measureSpec);
        int specSize = MeasureSpec.getSize(measureSpec);

        switch (specMode) {
        case MeasureSpec.UNSPECIFIED:
            result = size;
            break;
        case MeasureSpec.AT_MOST:
        case MeasureSpec.EXACTLY:
            result = specSize;
            break;
        }
        return result;
    }
```
由于wrap_content属于模式AT_MOST，所以宽高为specSize，也就是父容器的size，这就和match_parent一样了。解决这个问题总的思路是重写onMeasure()具体点来说，模仿getDefaultSize()重新获取宽高。

```java
 @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        int widthMode = MeasureSpec.getMode(widthMeasureSpec);
        int widthSize = MeasureSpec.getSize(widthMeasureSpec);
        int heightMode = MeasureSpec.getMode(heightMeasureSpec);
        int heightSize = MeasureSpec.getSize(heightMeasureSpec);

        int width = widthSize, height = heightSize;

        if (widthMode == MeasureSpec.AT_MOST) {
            width = dp2px(DEFAULT_SIZE);
        }

        if (heightMode == MeasureSpec.AT_MOST) {
            height = dp2px(DEFAULT_SIZE);
        }
        setMeasuredDimension(width, height);
    }
```

## 问题二：Path.addPath()和PathMeasure结合使用
举例子说明：

```java
    mTickPath.addPath(entryPath);
    mTickPath.addPath(leftPath);
    mTickPath.addPath(rightPath);
    mTickMeasure = new PathMeasure(mTickPath, false);
    // mTickMeasure is a PathMeasure
```
尽管`mTickPath`现在是由三个path构成，但是`mTickMeasure`此时的`length`和`entryPath`长度是一样的，到这里我就很奇怪了。看一下`getLength()`的源码：

```java
    /**
     * Return the total length of the current contour, or 0 if no path is
     * associated with this measure object.
     */
    public float getLength() {
        return native_getLength(native_instance);
    }
```
从注释来看，获取的是**当前轮廓线**的总长。

`getLength`调用了native层的方法，到这里不得不看底层的实现了。
通过阅读源代码不难发现，`Path`和`PathMeasure`实际分别对应底层的`SKPath`和`SKPathMeasure`。

查看native层的`getLength()`源码：

```c
    SkScalar SkPathMeasure::getLength() {
       if (fPath == NULL) {
        return 0;
       }
    if (fLength < 0) {
        this->buildSegments();
    }
    SkASSERT(fLength >= 0);
    return fLength;
}
```

实际上调用的`buildSegments()`来对`fLength`赋值，这里底层的设计有一个很聪明的地方，在初始化`SKPathMeasure`时对`fLength`做了特殊处理：

```c
SkPathMeasure::SkPathMeasure(const SkPath& path, bool forceClosed) {
    fPath = &path;
    fLength = -1;   // signal we need to compute it
    fForceClosed = forceClosed;
    fFirstPtIndex = -1;

   fIter.setPath(path, forceClosed);
}
```

 也就是说当还没有执行过getLength()方法时，fLength一直是-1，一旦执行则fLength>=0，则下一次就不会执行`buildSegments()`,这样避免了重复计算.


截取`buildSegments()`部分代码：

```c
void SkPathMeasure::buildSegments() {
157    SkPoint         pts[4];
158    int             ptIndex = fFirstPtIndex;
159    SkScalar        distance = 0;
160    bool            isClosed = fForceClosed;
161    bool            firstMoveTo = ptIndex < 0;
162    Segment*        seg;
163
164    /*  Note:
165     *  as we accumulate distance, we have to check that the result of +=
166     *  actually made it larger, since a very small delta might be > 0, but
167     *  still have no effect on distance (if distance >>> delta).
168     *
169     *  We do this check below, and in compute_quad_segs and compute_cubic_segs
170     */
171    fSegments.reset();
172    bool done = false;
173    do {
174        switch (fIter.next(pts)) {
175            case SkPath::kMove_Verb:
176                ptIndex += 1;
177                fPts.append(1, pts);
178                if (!firstMoveTo) {
179                    done = true;
180                    break;
181                }
182                firstMoveTo = false;
183                break;
184
185            case SkPath::kLine_Verb: {
186                SkScalar d = SkPoint::Distance(pts[0], pts[1]);
187                SkASSERT(d >= 0);
188                SkScalar prevD = distance;
189                distance += d;
190                if (distance > prevD) {
191                    seg = fSegments.append();
192                    seg->fDistance = distance;
193                    seg->fPtIndex = ptIndex;
194                    seg->fType = kLine_SegType;
195                    seg->fTValue = kMaxTValue;
196                    fPts.append(1, pts + 1);
197                    ptIndex++;
198                }
199            } break;
200
201            case SkPath::kQuad_Verb: {
202                SkScalar prevD = distance;
203                distance = this->compute_quad_segs(pts, distance, 0, kMaxTValue, ptIndex);
204                if (distance > prevD) {
205                    fPts.append(2, pts + 1);
206                    ptIndex += 2;
207                }
208            } break;
209
210            case SkPath::kConic_Verb: {
211                const SkConic conic(pts, fIter.conicWeight());
212                SkScalar prevD = distance;
213                distance = this->compute_conic_segs(conic, distance, 0, kMaxTValue, ptIndex);
214                if (distance > prevD) {
215                    // we store the conic weight in our next point, followed by the last 2 pts
216                    // thus to reconstitue a conic, you'd need to say
217                    // SkConic(pts[0], pts[2], pts[3], weight = pts[1].fX)
218                    fPts.append()->set(conic.fW, 0);
219                    fPts.append(2, pts + 1);
220                    ptIndex += 3;
221                }
222            } break;
223
224            case SkPath::kCubic_Verb: {
225                SkScalar prevD = distance;
226                distance = this->compute_cubic_segs(pts, distance, 0, kMaxTValue, ptIndex);
227                if (distance > prevD) {
228                    fPts.append(3, pts + 1);
229                    ptIndex += 3;
230                }
231            } break;
232
233            case SkPath::kClose_Verb:
234                isClosed = true;
235                break;
236
237            case SkPath::kDone_Verb:
238                done = true;
239                break;
240        }
241    } while (!done);
242
243    fLength = distance;
244    fIsClosed = isClosed;
245    fFirstPtIndex = ptIndex;

```

代码较长需要慢慢思考。`fIter`是一个`Iter`类型，在`SKPath.h`中的声明：

>>
		/** Iterate through all of the segments (lines, quadratics, cubics) of
	        each contours in a path.
	        The iterator cleans up the segments along the way, removing degenerate
	        segments and adding close verbs where necessary. When the forceClose
	        argument is provided, each contour (as defined by a new starting
	        move command) will be completed with a close verb regardless of the
	        contour's contents.
	    */


从这个声明中就可以明白Iter的作用就是遍历在path中的每一个contour。看一下`Iter.next()`方法：

```c
    Verb next(SkPoint pts[4], bool doConsumeDegerates = true) {
           if (doConsumeDegerates) {
               this->consumeDegenerateSegments();
           }
            return this->doNext(pts);
    }

```

doNext()方法的代码就不贴出来了，作用就是判断contour的类型并把相应的点的坐标取出传给`pts[4]`

当`fIter.next()`返回`kDone_Verb`时，一次遍历结束。
buildSegments中的循环正是在做此事，而且从case `kLine_Verb`模式的`distance += d;`不难发现这个length是累加起来的。在举的例子当中，mTickPath有三个contour（mEntryPath,mLeftPath,mRightPath），我们调用mTickMeasure.getLength()时，首先会**累计** 获取mEntryPath这个contour的长度。

这就不难解释为什么mTickMeasure获取的长度和mEntryPath的一样了。那么想一想，怎么让buildSegments()对下一个contour进行操作呢？关键是把`fLength`置为-1

```c
/** Move to the next contour in the path. Return true if one exists, or false if
    we're done with the path.
*/
bool SkPathMeasure::nextContour() {
    fLength = -1;
    return this->getLength() > 0;
}
```
与native层对应的API是PathMeasure.nextContour();
