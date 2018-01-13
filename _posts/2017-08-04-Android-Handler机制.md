---
layout:     post
title:      "Handler 机制"
author:     "CoXier"
header-img: "img/post-bg.jpg"
tags:

- Android
- 源码解析
---

# 一、Handler 机制
Handler机制 是**线程间通信**很关键的机制，在应用层中，可以利用 Handler 做定时任务、消息统一的转发和处理、View 动画。Handler 消息机制在 java 层涉及：

 > framework/base/core/java/andorid/os/
 > - Handler.java
 > - Looper.java
 > - Message.java
 > - MessageQueue.java

```java

private static Handler mHandler = new Handler(Looper.getMainLooper()){
    @Override
    public void handleMessage(Message msg) {
        Log.d(TAG,msg.obj.toString());
        super.handleMessage(msg);
    }
};

private static final Thread mThread = new Thread(new Runnable() {
    @Override
    public void run() {
        try {
            Thread.sleep(10000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        Message message = new Message();
        message.obj = "Hello World";
        mHandler.sendMessage(message);
    }
});

@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);
    Log.d(TAG,"Activity onCreate");
    mThread.start();
}
```

以上就是一个简单的Android**异步消机制**。

# 二、Message

因为 Handler 会用到 Message、Looper 和 MessageQueue，所以先从 Message 看起。

Message 作为 Handler 机制的信息载体，内置了两个 `int` 和 `Object` 对象用于存储数据。内部数据结构：

|   数据类型   |   成员变量    |            描述             |
| :------: | :-------: | :-----------------------: |
|   int    |   what    | 消息的数据类型，每个Handler有自己的命名空间 |
|   int    | arg1，arg2 |          存储简单数据           |
|  Object  |    obj    |          存储复杂数据           |
|   long   |   when    |          消息触发的时间          |
| Handler  |  target   |           接受消息            |
| Runnable | Callback  |         处理消息时的回调          |
| Message  |   next    |       按照when排序的下个消息       |

文档提醒我们，虽然 Message 的构造方法是 public，但是建议不要直接构造 Message,推荐使用 `Meobtain` 和 `obtain(Message orig)`，因为 Message 内部维护了一个复用池，避免重复创建 Message。

## 2.1 obtain message	

```java
public static Message obtain() {
    synchronized (sPoolSync) {
        if (sPool != null) {
            Message m = sPool;
            sPool = m.next;
            m.next = null;
            m.flags = 0; // clear in-use flag
            sPoolSize--;
            return m;
        }
    }
    return new Message();
}
```

`sPool` 是「复用池」中的首个空闲 message，当取 message 时，如果池子中有可用的 message，则移动 sPool 到 sPool.next，池子的实际大小减一；否则新建一个 message 。 

## 2.2 recycle message

```java
void recycleUnchecked() {
    // Mark the message as in use while it remains in the recycled object pool.
    // Clear out all other details.
 	// 清理其它数据信息，但是 flags 是在 obtain 时才清除。
    flags = FLAG_IN_USE;
    what = 0;
    arg1 = 0;
    arg2 = 0;
    obj = null;
    replyTo = null;
    sendingUid = -1;
    when = 0;
    target = null;
    callback = null;
    data = null;

    synchronized (sPoolSync) {
      	// MAX_POOL_SIZE 的默认大小为 50
        if (sPoolSize < MAX_POOL_SIZE) {
          	// 将当前被回收的 message 放到池子的首位
            next = sPool;
            sPool = this;
            sPoolSize++;
        }
    }
}
```

结合 2.1 来看，通过 `Obejct sPoolSync` 来给 `obtain` 和 `recycleUnchecked`上锁，这个上锁方式可以学习借鉴一下。

# 三、MessageQueue

`MessageQueue` 在 Handler 机制中，作为一个消息队列，可以让消息「入队」和「出队」。借用 Gityuan 的一句话：

> `MessageQueue`是连接Java层和Native层的纽带，换言之，Java层可以向`MessageQueue`消息队列中添加消息，Native层也可以向`MessageQueue`消息队列中添加消息。

## 3.1 MessageQueue 构造

```java
MessageQueue(boolean quitAllowed) {
    mQuitAllowed = quitAllowed;
  	// mPtr 供 native 方法使用
    mPtr = nativeInit();
}
```

## 3.2 「入队」

```java
boolean enqueueMessage(Message msg, long when) {
  	// msg 必须有 handler
    if (msg.target == null) {
        throw new IllegalArgumentException("Message must have a target.");
    }
  	// msg 同时只能被一个 handler 使用
    if (msg.isInUse()) {
        throw new IllegalStateException(msg + " This message is already in use.");
    }
	
    synchronized (this) {
        if (mQuitting) {
            IllegalStateException e = new IllegalStateException(
                    msg.target + " sending message to a Handler on a dead thread");
            Log.w(TAG, e.getMessage(), e);
          	// 如果当前线程的looper正在退出则回收 msg
            msg.recycle();
            return false;
        }

        msg.markInUse();
      	// msg.when 外部无法直接操作
        msg.when = when;
        Message p = mMessages;
        boolean needWake;
        if (p == null || when == 0 || when < p.when) {
          	// 如果 当前 messagequeue 为空，或者 时间小于队首 message 的时间，则插入队首
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked;
        } else {
			// 按照时间戳来插入 messagequeue
            needWake = mBlocked && p.target == null && msg.isAsynchronous();
            Message prev;
            for (;;) {
                prev = p;
                p = p.next;
                if (p == null || when < p.when) {
                    break;
                }
                if (needWake && p.isAsynchronous()) {
                    needWake = false;
                }
            }
            msg.next = p; // invariant: p == prev.next
            prev.next = msg;
        }

        // We can assume mPtr != 0 because mQuitting is false.
        if (needWake) {
            nativeWake(mPtr);
        }
    }
    return true;
}
```

实际上 MessageQueue 并不是一个队列，而是一个链表（不过链表也可以实现队列），将 message 按照触发的时间顺序来入队。

## 3.3 出队

```java
Message next() {
    // 如果消息循环退出，则直接返回
    final long ptr = mPtr;
    if (ptr == 0) {
        return null;
    }

    int pendingIdleHandlerCount = -1; // 第一次循环时为-1
    int nextPollTimeoutMillis = 0;
    for (;;) {
        if (nextPollTimeoutMillis != 0) {
            Binder.flushPendingCommands();
        }
		
      	// 阻塞操作，当等待 nextPollTimeoutMillis 时长返回
        nativePollOnce(ptr, nextPollTimeoutMillis);

        synchronized (this) {
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;
            if (msg != null && msg.target == null) {
                // Stalled by a barrier.  Find the next asynchronous message in the queue.
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
            if (msg != null) {
                if (now < msg.when) {
                    // 当前 message 的触发时间还未到，则在下次循环时阻塞 nextPollTimeoutMillis
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
                    // 获取到下一条 message
                    mBlocked = false;
                    if (prevMsg != null) {
                        prevMsg.next = msg.next;
                    } else {
                        mMessages = msg.next;
                    }
                    msg.next = null;
                    if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                    msg.markInUse();
                    return msg;
                }
            } else {
                // 队列为空
                nextPollTimeoutMillis = -1;
            }

    }
}
```

# 四、Looper

Looper 内部有一个 MessageQueue ，通过 MessageQueue 不断的取出 message 。

## 4.1 Looper 构造

```java
private Looper(boolean quitAllowed) {
    // 创建 MessageQueue
    mQueue = new MessageQueue(quitAllowed);
  	// 将 Looper 与当前线程绑定
    mThread = Thread.currentThread();
}
```

注意到构造函数是 private ，暴露给外部的是：`prepare` 方法：

```java
public static void prepare() {
    prepare(true);
}

private static void prepare(boolean quitAllowed) {
    // 一个线程只能调用一次 Looper.prepare() 
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper(quitAllowed));
}
```

通过 `ThreadLocal` 来保存 looper 对象。ThreadLocal(Thread Local Storage) 线程本地存储区，每个线程都有自己的存储区，不同线程不能访问对方的 TSL 。

## 4.2 loop

```java
public static void loop() {
    final Looper me = myLooper();
    if (me == null) {
        throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
    }
    final MessageQueue queue = me.mQueue;

    // Make sure the identity of this thread is that of the local process,
    // and keep track of what that identity token actually is.
    Binder.clearCallingIdentity();
    final long ident = Binder.clearCallingIdentity();

    for (;;) {
        Message msg = queue.next(); // 见 [3.3]
        if (msg == null) {
            // No message indicates that the message queue is quitting.
            return;
        }

        final long traceTag = me.mTraceTag;
        if (traceTag != 0 && Trace.isTagEnabled(traceTag)) {
            Trace.traceBegin(traceTag, msg.target.getTraceName(msg));
        }
        try {
            //  Handler 分发消息 见[5.3]
            msg.target.dispatchMessage(msg);
        } finally {
            if (traceTag != 0) {
                Trace.traceEnd(traceTag);
            }
        }
        // message 使用完毕，回收
        msg.recycleUnchecked();
    }
}
```

当调用 looper.loop() 时，looper 就开始不断的取数据，然后交给 Handler 分发。你可能注意到，我们在主线程使用 Handler 机制时，并没有调用 looper.loop() ，其实不然，这些在 `ActivityThread`已经调用了。

`ActivityThread#main`

```java

public static void main(String[] args) {
  	// 创建主线程对应的 looper
    Looper.prepareMainLooper();

    ActivityThread thread = new ActivityThread();
    thread.attach(false);

    if (sMainThreadHandler == null) {
        sMainThreadHandler = thread.getHandler();
    }

    if (false) {
        Looper.myLooper().setMessageLogging(new
                LogPrinter(Log.DEBUG, "ActivityThread"));
    }

    // End of event ActivityThreadMain.
    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
    // 开始取数据
    Looper.loop();

    throw new RuntimeException("Main thread loop unexpectedly exited");
}
```



# 五、 Handler

## 5.1 Handler 构造

Handler 的构造按照我个人的理解可以分为两大类：无 Looper 参数，有 Looper 参数。

### 5.1.1 Handler 无 looper 参数构造

```java
public Handler() {
  this(null, false);
}

public Handler(Callback callback) {
  this(callback, false);
}

public Handler(Callback callback, boolean async) {
    if (FIND_POTENTIAL_LEAKS) {
        final Class<? extends Handler> klass = getClass();
        if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                (klass.getModifiers() & Modifier.STATIC) == 0) {
            Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                klass.getCanonicalName());
        }
    }

    mLooper = Looper.myLooper();
    if (mLooper == null) {
        throw new RuntimeException(
            "Can't create handler inside thread that has not called Looper.prepare()");
    }
    mQueue = mLooper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}
```

在无 Looper 构造函数中，Looper 默认是当前线程保存的 Looper 。

* `FIND_POTENTIAL_LEAKS` 用来探测当前 Handler 是否是存在内存泄漏的可能性（匿名类、内部类、[LocalClass](https://docs.oracle.com/javase/tutorial/java/javaOO/localclasses.html) 、非静态）。但是从其定义来看，`FIND_POTENTIAL_LEAKS` 似乎一直为 false，除非通过反射。
* 如果 looper 是 null，则 Handler 就无法获取到 message ，所以这里会抛出 `RuntimeException`

### 5.1.2 Handler 有 Looper 参数构造

```java
public Handler(Looper looper, Callback callback, boolean async) {
    mLooper = looper;
    mQueue = looper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}
```

到目前为止，我没有设置过 async 为 true，应该用的不是很多。

## 5.2 发送消息
Handler中总的来看有两种「发送消息」方法，post 和 sendMessage。

post：
* post(Runnable r)
* postAtTime(Runnable r, long uptimeMillis)
* postDelayed(Runnable r, long delayMillis)

sendMessage:
* sendMessage(Message msg)
* sendMessageAtTime(Message msg, long uptimeMillis)
* sendMessageDelayed(Message msg, long delayMillis)

post 操作会把 Runnable 包装成 Message：

```java
private static Message getPostMessage(Runnable r) {
    Message m = Message.obtain();
    m.callback = r;
    return m;
}
```

post  和 sendMessage 最后都会调用: sendMessageAtTime

```java
public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
    MessageQueue queue = mQueue;
    if (queue == null) {
        RuntimeException e = new RuntimeException(
                this + " sendMessageAtTime() called with no mQueue");
        Log.w("Looper", e.getMessage(), e);
        return false;
    }
    return enqueueMessage(queue, msg, uptimeMillis);
}

private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
    // 设置 msg 的目标是当前的 Handler
    msg.target = this;
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    // message 入队
    return queue.enqueueMessage(msg, uptimeMillis);
}
```

## 5.3 Handler#dispatchMessage

在 **[4.2]Looper#loop** 中看到当取出 message 后，会调用 `Handler#dispatchMessage()`, 让 Handler 分发该 message。

```java
public void dispatchMessage(Message msg) {
    if (msg.callback != null) {
        handleCallback(msg);
    } else {
        if (mCallback != null) {
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        handleMessage(msg);
    }
}

private static void handleCallback(Message message) {
    message.callback.run();
}

public interface Callback {
    public boolean handleMessage(Message msg);
}
```
处理 message 的优先级:

* 如果 message 是包装的 `Runnable`，则调用 runnable 
* 如果构造 Handler 时传入的 Callback 参数不为 null，则调用 Callback
* 调用重写的 `handlerMessage`



