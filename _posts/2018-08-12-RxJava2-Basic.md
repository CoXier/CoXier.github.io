---

layout:     post
title:      "RxJava Basic"
author:     "CoXier"
header-img: "https://images-cdn.shimo.im/vBhbxDllRBgqA17X/stephan_zabini_775109_unsplash.jpg"
tags:

- RxJava
---

# RxJava 入门分享

RxJava 遵循了 [ReactiveStreams]( https://github.com/reactive-streams/reactive-streams-jvm) 的定义，一些重要的设计思想写在了 README，定义了四个关键概念：

- Publisher：发送数据


- Subscriber：接收数据


- Subscription：描述 publisher 和 subscriber 的关系，而且只受 subscriber 的控制


- Processor：数据处理阶段，包括发送前的处理，和接收后的处理

Publisher 将数据发送给 Subscriber，Subscriber 处理数据流按顺序执行：

**onSubscribe onNext\* (onError \| onComplete)?**

Subscrier 执行一次 onSubscribe，可能执行 0 次或者多次 onNext。一个好的事件流应该需要一次 onError 或者 一次 onComplete。Reactive Stream 常见的图示：

![img](https://raw.github.com/wiki/ReactiveX/RxJava/images/rx-operators/legend.png)

# 一、Reactive Stream in RxJava2

在 RxJava 2.x 中，reactive class 有 Flowable、Observable、Single、Maybe、Completable，他们类似上面提到的 Publisher，起到发送数据的作用，其中 Flowable 实现了 Publisher 的接口，具备了 Backpressure，而 Observable 等其他的 reactive class 从 Flowable 衍生而来，不具备 Backpressure。

初看 Flowable 对应 Subscriber，Observable 对应 Observer，Single 对应 SingleObserver，会觉得很晕，但联系起 RS，会发现整体模型是符合或者从 **Publisher/Subscriber** 衍生而来，RxJava 保持了这种命名规范和约束。

为了方便描述，下文除了特别声明，Flowable、Observable、Single、Maybe、Completable 均叫做 Observable。

# 二、RxJava常用操作符

## 2.1 Observable Creation——创建操作符

RxJava 的方法数多的原因之一是其拥有大量的操作符，RxJava 中所有的操作都是通过操作符进行的， 虽然 RxJava 中每个操作符都有与之对应的 Observable ，但是只有一类操作符是用来作为源头发射数据或者事件的，这类操作符是[创建操作符](https://github.com/ReactiveX/RxJava/wiki/Creating-Observables)，常见的创建操作符如下:

- **just**：接收1~10个参数，创建发射一个或者多个对象的 Observable，作者手写了 1 个参数 ~ 10 个参数的方法，just(T item1.....T item10)


- **from**: fromArray, ObservableFromArray，fromCallable，fromFuture ，fromIterable 发射一个“集合”或者 call、furture


- **create**：创建一个 Observable，需要手动去调用 onNext，onComplete，onError 方法


- **range**: 接收一个数序列，类似 fromArray，按序发射


- **timer**: 延迟时间xx，发射一个数据


- **interval**: 延迟时间xx，并每隔时间xx，周期性发射一个数据

使用 fromArray 小试牛刀，创建一个 Observable 按照顺序发送两个字符串 "jianxin" 和 "fucheng"，Observer 收到字符串后打印出来，当 Observable 发送完后，下游打印 "onComplete"，也就是遵循 onSubscribe onNext*2  onComplete 的协议。


![img](https://cdn.ruguoapp.com/FpNFib-RIwRU8X3Y5iPToyMaPc5V.png?imageMogr2/auto-orient/heic-exif/1/thumbnail/2500x9999%3E)


上面的代码可以分成 3 个部分来看：

* Observable.fromArray：创建一个 Observable，Observable.fromArray 操作符对应 ObservableFromArray，**在 RxJava 中每个操作符都是对应一个 Observable**。
* subscribe：订阅数据，开始发射 "jianxin" 和 "fucheng"
* observer: 接收数据，打印字符串

subscribe(Observer) 实际上调用的是 subscribeActual(Observer)，subscribeActual 是一个抽象方法，所以调用的是 `ObservableFromArray#subscribeActual` 见下图。然后循环发射调用 observer#onNext，如果中间遇到 error，就调用 observer#onError，如果顺利地发射完所有数据，则调用 observer#onComplete。

![img](https://cdn.ruguoapp.com/FrTHEM4QsHHsD_z4OiCkmtMm9p-Z.png?imageMogr2/auto-orient/heic-exif/1/thumbnail/2500x9999%3E)



## 2.2 Filtering——过滤操作符

![img](https://images-cdn.shimo.im/zFnVdrBBsKQpBpRg/2.png!thumbnail)

RxJava 有很多过滤操作符，但是出发点都是过滤出 Observer 需要的数据。例如上面的代码，Observer 只想获取以字母 `j` 开头的 name。

再来一个稍微复杂一点的例子，下游只需要 `j` 开头的 name ，并且相同的 name 只接收一次，这个时候就可以使用操作符 [Distinct ](http://reactivex.io/documentation/operators/distinct.html) 进行去重操作。

![img](https://images-cdn.shimo.im/xHyA9kyWk0IZ3Hox/3.png!thumbnail)

## 2.3 Observable Transformation——转换操作符

RxJava 能转换 Observable 发出的数据，经过变换后交给 Observer 处理。下面有个例子，上游发送了 1， 2， 3 三个数据，经过 [map](http://reactivex.io/documentation/operators/map.html) 操作符处理后，下游接收到的是 20，40 ，60。

![](https://images-cdn.shimo.im/wtCELXbBsxk9llRs/carbon_1_.png!thumbnail)

常见的变换操作符有：

* **map** ：将上游发送的数据，变换成另一个数据
* **flatMap**: 将上游发送的数据，变换成一个 Observable，新的 Observable 会发送新的数据

上面已经介绍了 map 的使用，下面再来看看如何使用 flatMap 也将上游发送的数据每个增大 20 倍。

![](https://images-cdn.shimo.im/RKRAjAw4YW0F0qrG/carbon_2_.png!thumbnail)

大多数情况下，map 和 flatMap 都能实现一样的功能，**但是 flatMap 可以用来做一些和异步有关的事情**，但可能 map 无法做到。比如我们同时获取了一系列用户的 userId，然后希望通过这些 userId 分别请求每个用户的详细信息 UserInfo，每请求到一个 UserInfo 就更新相应的 UI，请求用户详细信息属于网络异步操作，所以应该不必等到所有的请求完成才更新 UI。

![](https://images-cdn.shimo.im/ALUflCVm1vUuPVHY/carbon_3_.png!thumbnail)

这里为了说明 flatMap 对于异步操作的支持，我们把每个数据转换为一个 Observable，但 Observable 会进行 delay 操作，传入的数据越大，延迟反而越小，最终，我们将看到完全逆序的事件处理。但是如果使用 map 操作，请求用户信息就会挨个排队，只有前一个完成后面才能继续，**当所有的转换（请求）完成之后，下游才能收到数据**。

## 2.4 Observable Combination——组合操作符

在异步事件流中，有时候会遇到这种需求：在 A 事件和 B 事件均处理好以后，再执行某个操作，其中 A 事件和 B 事件均是耗时操作。举个例子，有两个接口，一个用来获取喜欢周杰伦的粉丝军团，一个用来获取喜欢五月天的粉丝军团，这个时候 PM 提出希望在客户端显示同时喜欢周杰伦和五月天的粉丝（当然这个需求可以让服务端完成）。

这个需求的关键点在于需要在两个接口的数据都返回时再做查找操作，常规写法应该会借助 Thread 和 Handler 这样写：

![](https://cdn.ruguoapp.com/FoFrd0dzDd0XjUIlW5NZMnqZ1A2n.png?imageMogr2/auto-orient/heic-exif/1/thumbnail/2500x9999%3E)

上面的代码省略了 UI 线程更新的具体逻辑，UI线程更新还是需要借助 Handler ，一次 sendMessage 和 handlerMsg，其实代码是很**分散**的。如果换用 RxJava + Retrofit + RxJava2CallAdapter 整个流程就会收敛到一条线上。 [zip](http://reactivex.io/documentation/operators/zip.html) 操作符通过一个函数将多个 Observable 发出的数据结合为一个新的数据。

![](https://cdn.ruguoapp.com/FlUMacjF5hzb_EVNph6nGEUsnAIR.png)

# 三、线程切换

RxJava 线程切换特别方便，subscribeOn 指定 observable 发射数据所在的线程，observeOn 指定 observer 处理数据所在的线程。前面提到过 RxJava 每个操作符都对应一个 Observable，subscribeOn 和 observeOn 也不例外，subscribeOn 操作符对应 ObservableSubscribeOn，observeOn 操作符 ObservableObserveOn。

下面结合一个 demo 来分析 RxJava 是如何切换发送数据和处理数据所在的线程。

![](https://cdn.ruguoapp.com/FuwmCKcpea1LnlZEMjBruQX9YJ5q.png?imageMogr2/auto-orient/heic-exif/1/thumbnail/2500x9999%3E)

在 RxJava 中将发送数据的一方叫做 **upstream**，接收数据的一方叫 **downstream**。如：

![img](https://images-cdn.shimo.im/SV3uIdMoz0MZzRlC/%E4%B8%8B%E8%BD%BD.png!thumbnail)

在多个操作符链式调用的时候，站在 operator2() 的角度，左侧就是 **upstream** , 右侧是 **downstream**。所以在上面的 Demo 中，ObservableFromArray 的 downstream 是 ObservableSubscribeOn，ObservableSubscribeOn 的 downstream 是 ObservableObserveOn，ObservableSubscribeOn 的 upstream 是 ObservableFromArray。

每个 Observable 对应关系见下表：

|      Observable       | Operator    |      downstream       |       upstream        |      Observer       |
| :-------------------: | ----------- | :-------------------: | :-------------------: | :-----------------: |
|  ObservableFromArray  | from        | ObservableSubscribeOn |           -           | SubscribeOnObserver |
| ObservableSubscribeOn | subscribeOn |  ObservableObserveOn  |  ObservableFromArray  |  ObserveOnObserver  |
|  ObservableObserveOn  | observeOn   |           -           | ObservableSubscribeOn |   LambdaObserver    |

在 2.1 中简单分析了 Observable.fromArray 的流程，fromArray 对应 ObservableFromArray。每经过一个操作符都会将当前的 Observable 包装成操作符对应的 Observable，经过 subscribeOn 之后，ObservableFromArray 就会被包装成 ObservableSubscriberOn，再经过 observeOn 之后， ObservableSubscribeOn 会被包装成 ObservableObserveOn，所以直接发生订阅关系的是 ObservableObserveOn。

RxJava 的事件流可以分为三部分：

* 事件源头发送数据，在 RS 定义中，事件是在发生订阅关系时开始的
* 事件中间处理如过滤、转换等
* 事件接收处理

上面的代码未涉及到中间处理的过程，所以下面分两个小部分展开：1.事件源头发送数据 2.事件接收处理

## 3.1 订阅事件

![](https://cdn.ruguoapp.com/FsfQlX9DS2ycwDCRbGmZuBOw7Pdo.png?imageMogr2/auto-orient/heic-exif/1/thumbnail/2500x9999%3E)


* 在 ObservableObserveOn#subscribeActual 会调用 upstream ObservableSubcribeOn#subscribe
* 在 ObservableSubscribeOn 中为了实现上游 ObservableFromArray 的 subscribe 运行在另一个线程，**使用了 Scheduler 和 Wroker 来调度 SubscribeTask，进而调用 ScheduledExecutorService 的 submit**
* ObservableSubscribeOn#SubscribeTask 中调用 upstream ObservableFromArray 的 subscribe，之后事件源头开始发送数据

## 3.2 接收事件

![](https://cdn.ruguoapp.com/Fo3f04VqOrNsx9ONbmLwROAb1o0h.png?imageMogr2/auto-orient/heic-exif/1/thumbnail/2500x9999%3E)

* 3.1 中讲到 ObservableFromArray 是数据源头，ObservableFromArray 的 downstream 是 ObservableSubscriberOn，所以第一个接收到事件源发出数据的是 SubscribeOnObserver。
* SubscribeOnObserver#onNext 将数据发送给 ObserveOnObserver。
* ObserveOnObserver 接收到数据后，先将数据入队。
* **ObserveOnObserver 为了向 LambdaObserver 发送数据时切换线程，用到了 worker#schedule**
* ObserveOnObserver 实现了 Runnable 接口，在 ObserveOnObserver 执行 run 时，将队列的数据出队，然后向 LambdaObserver 发送。

## 3.2.1 RxAndroid

在 Android 中使用 RxJava 时大多数 observeOn 场景是期望发生在 main thread 来更新 UI，RxAndroid 提供了 `observeOn(AndroidSchedulers.mainThread()) ` 。具体实现原理是自定义了 HandlerScheduler 和 HandlerWorker，在 3.2 中我们知道 observeOn 具体实现是依靠 Worker#schedule。

![](https://cdn.ruguoapp.com/FsEdmCg1TGjzu2qnuCpWo1nvQ9Vj.png?imageMogr2/auto-orient/heic-exif/1/thumbnail/2500x9999%3E)

将 ObserverOnObserver 赋值给 Message#callback，然后 Handler 将 Message send 出去，delay 一定时间后从 Looper#loop 取出数据，然后 ObserverOnObserver#run 即运行在 main thread 。
