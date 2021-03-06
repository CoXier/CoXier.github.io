---
layout:     post
title:      "LeakCanary源码解析（一）"
author:     "CoXier"
header-img: "https://gitee.com/coxier/tuchuang/raw/master/photo-1443890923422-7819ed4101c0.jpeg"
tags:

- Android
- 源码解析
- 内存泄漏
---

本次源码解析的 commit 是：660d6742fccda2f9f3f84a8c4acc437412a262a3

# 一、LeakCanary 初始化

LeakCanary 是通过 ContentProvider 初始化的，目前很多第三方库比如 picasso 都采用了这种方式进行初始化，这种初始化有个好处，可以避免入侵使用方的代码。具体方式如下：

* 自定义 ContentProvider，如 LeakSentryInstaller，然后在 onCreate 方法中执行第三方库的初始化操作
* 在 AndroidManifest.xml 中声明 ContentProvider

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest
    xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.squareup.leakcanary.leaksentry"
    >

  <application>
    <provider
        android:name="leakcanary.internal.LeakSentryInstaller"
        android:authorities="${applicationId}.leak-sentry-installer"
        android:exported="false"/>
  </application>
</manifest>
```

通过[官方](<https://square.github.io/leakcanary/fundamentals/>)可以了解到 LeakCanary 工作原理是由哪几步构成的：

* Detecting retained instances： 探测存活的实例
* Dumping the heap：将发生内存泄漏时的 java heap dump 到 `.hprof` 文件中
* Analyzing the heap：分析 `.hprof` 文件，找到内存泄漏的路径
* Grouping leaks：有一些内存泄漏是同一个原因造成，LeakCanary 会将这些归类

# 二、Detecting retained instances

探测 retained 实例是检查内存泄漏的第一步，探测的时机是 Activity/Fragment destroyed，负责这个功能的模块是 leaksentry，leaksentry 依赖了 watcher。ActivityDestroyWatcher 负责 Activity 的探测，AndroidOFragmentDestroyWatcher 负责 Fragment 的探测，LeakCanary 的探测都是无侵入式的，借助了 Android Framework 提供的 Activity/Fragment 生命回调。

Activity 的触发时机：

```kotlin
  private val lifecycleCallbacks =
    object : Application.ActivityLifecycleCallbacks by noOpDelegate() {
      override fun onActivityDestroyed(activity: Activity) {
        if (configProvider().watchActivities) {
          refWatcher.watch(activity)
        }
      }
    }
```

这里使用了 kotlin 的语言特性——委托代理，在之前的 Java 版本是空实现方法，然后复写 onActivityDestroyed。对比之下，kotlin 版本似乎更加的高级，kotlin 的委托是在编译层面做的，反编译 lifecycleCallbacks，可以发现 kotlin 在编译时会把委托的类改成内部成员静态代理的方式。

```java
public final class ActivityDestroyWatcher$lifecycleCallbacks$1 implements ActivityLifecycleCallbacks {
    private final /* synthetic */ ActivityLifecycleCallbacks $$delegate_0;
    final /* synthetic */ ActivityDestroyWatcher this$0;

    public void onActivityCreated(@RecentlyNonNull Activity activity, @RecentlyNullable Bundle bundle) {
        this.$$delegate_0.onActivityCreated(activity, bundle);
    }
  
    ...
        
    ActivityDestroyWatcher$lifecycleCallbacks$1(ActivityDestroyWatcher $outer) {
        this.this$0 = $outer;
        InternalHelper internalHelper = InternalHelper.INSTANCE;
        Class javaClass$iv = ActivityLifecycleCallbacks.class;
        InvocationHandler noOpHandler$iv = InternalHelper$noOpDelegate$noOpHandler$1.INSTANCE;
        Object newProxyInstance = Proxy.newProxyInstance(javaClass$iv.getClassLoader(), new Class[]{javaClass$iv}, noOpHandler$iv);
        if (newProxyInstance != null) {
            this.$$delegate_0 = (ActivityLifecycleCallbacks) newProxyInstance;
            return;
        }
        throw new TypeCastException("null cannot be cast to non-null type android.app.Application.ActivityLifecycleCallbacks");
    }

    public void onActivityDestroyed(@NotNull Activity activity) {
        Intrinsics.checkParameterIsNotNull(activity, "activity");
        if (((Config) this.this$0.configProvider.invoke()).getWatchActivities()) {
            this.this$0.refWatcher.watch(activity);
        }
    }
}
```

上面都是些和 leakcanary 的题外话，下面开始正式的分析，在 Activity#onDestroy 调用时，RefWatcher 会开始进行探测。

## 1.1 RefWatcher#watch

```kotlin
  @Synchronized fun watch(
    watchedInstance: Any,
    name: String
  ) {
    if (!isEnabled()) {
      return
    }
    removeWeaklyReachableInstances()
    val key = UUID.randomUUID()
        .toString()
    val watchUptimeMillis = clock.uptimeMillis()
    val reference =
      KeyedWeakReference(watchedInstance, key, name, watchUptimeMillis, queue)
    CanaryLog.d(
        "Watching %s with key %s",
        ((if (watchedInstance is Class<*>) watchedInstance.toString() else "instance of ${watchedInstance.javaClass.name}") + if (name.isNotEmpty()) " named $name" else ""), key
    )

    watchedInstances[key] = reference
    checkRetainedExecutor.execute {
      moveToRetained(key)
    }
  }
```

leakcanary 中很多操作都会调用 removeWeaklyReachableInstances，这个方法主要是做了这样一件事：**移除那些实例，哪些实例呢？实例是一个 WeakRef，他指向的对象被回收了**。看一下实现:

```java
  private fun removeWeaklyReachableInstances() {
    // WeakReferences are enqueued as soon as the object to which they point to becomes weakly
    // reachable. This is before finalization or garbage collection has actually happened.
    var ref: KeyedWeakReference?
    do {
      ref = queue.poll() as KeyedWeakReference?
      if (ref != null) {
        watchedInstances.remove(ref.key)
      }
    } while (ref != null)
  }
```

从注释可以得知，**当 WeakRef 指向的对象被回收以后，WeakRef 就会被加入 queue**，果然看源码还是能学到不少东西的。如果发现 WeakRef 指向的对象已经被回收了，那么就从 watchedInstances 将 key 移除掉，因为已经无内存泄漏的风险。

接着往下看，构造完 reference 后就会进行对该 reference 进行延迟 moveToRetained 操作，checkRetainedExecutor 是包装的 handler，默认一个 runnable 是延迟 5s 执行。5s 后开始执行 moveToRetained：

```kotlin
  @Synchronized private fun moveToRetained(key: String) {
    removeWeaklyReachableInstances()
    val retainedRef = watchedInstances[key]
    if (retainedRef != null) {
      retainedRef.retainedUptimeMillis = clock.uptimeMillis()
      onInstanceRetained()
    }
  }
```

执行 moveToRetained 时如果发现 watchedInstances[key] 不为 null，说明 watchedInstances[key] 指向的对象没有被回收，则表明有内存泄漏的风险，然后给 retainedRef#retainedUptimeMillis 赋值供之后筛选。此时只是怀疑可能有内存风险，因为在内存不紧张的情况下，可能 5s 以内对象不会被回收，所以接下来需要进行更进一步的检测——gc回收。

## 1.2 HeapDumpTrigger#onReferenceRetained

当 RefWatcher 检测到可能的风险之后就会通知 HeapDumpTrigger 进行 gc 检测。主要方法是 HeapDumpTrigger#onReferenceRetained，其实也就是调用 HeapDumpTrigger#scheduleRetainedInstanceCheck，在 scheduleRetainedInstanceCheck 中最后调用的 checkRetainedInstances

```kotlin
  private fun checkRetainedInstances(reason: String) {
    val config = configProvider()

    var retainedReferenceCount = refWatcher.retainedInstanceCount
    // 如果有实例还未被回收，则执行 gc
    if (retainedReferenceCount > 0) {
      gcTrigger.runGc()
      // 执行完 gc，再看看是否还有实例存活
      retainedReferenceCount = refWatcher.retainedInstanceCount
    }

    // 校验留存的实例数量是否达到了阈值
    if (checkRetainedCount(retainedReferenceCount, config.retainedVisibleThreshold)) return

    CanaryLog.d("Found %d retained references, dumping the heap", retainedReferenceCount)
    val heapDumpUptimeMillis = SystemClock.uptimeMillis()
    KeyedWeakReference.heapDumpUptimeMillis = heapDumpUptimeMillis
    dismissRetainedCountNotification()
    // dump file
    val heapDumpFile = heapDumper.dumpHeap()
    if (heapDumpFile == null) {
      CanaryLog.d("Failed to dump heap, will retry in %d ms", WAIT_AFTER_DUMP_FAILED_MILLIS)
      scheduleRetainedInstanceCheck("failed to dump heap", WAIT_AFTER_DUMP_FAILED_MILLIS)
      showRetainedCountWithHeapDumpFailed(retainedReferenceCount)
      return
    }
    lastDisplayedRetainedInstanceCount = 0
    // 移除 dump file 之前的实例，dump 之后的实例保留
    refWatcher.removeInstancesWatchedBeforeHeapDump(heapDumpUptimeMillis)
    // 分析 dumpFile
    HeapAnalyzerService.runAnalysis(application, heapDumpFile)
  }
```

之前说过每次去获取存活的实例时都会进行一次 removeWeaklyReachableInstances：

```kotlin
  val retainedInstanceCount: Int
    @Synchronized get() {
      removeWeaklyReachableInstances()
      return watchedInstances.count { it.value.retainedUptimeMillis != -1L }
    }
```

然后看一下 LeakCanary 的 **gc** 操作：

```kotlin
    override fun runGc() {
      // Code taken from AOSP FinalizationTest:
      // https://android.googlesource.com/platform/libcore/+/master/support/src/test/java/libcore/
      // java/lang/ref/FinalizationTester.java
      // System.gc() does not garbage collect every time. Runtime.gc() is
      // more likely to perform a gc.
      Runtime.getRuntime()
          .gc()
      enqueueReferences()
      System.runFinalization()
    }

    private fun enqueueReferences() {
      // Hack. We don't have a programmatic way to wait for the reference queue daemon to move
      // references to the appropriate queues.
      try {
        Thread.sleep(100)
      } catch (e: InterruptedException) {
        throw AssertionError()
      }
    }
```

通过 Runtime.getRuntime().gc() 操作 gc，然后线程 sleep 100ms 等待 WeakRef 进入 queue，100ms 是个 magic number ，可能是个经验值。

```kotlin
  private fun checkRetainedCount(
    retainedKeysCount: Int,
    retainedVisibleThreshold: Int
  ): Boolean {
    val countChanged = lastDisplayedRetainedInstanceCount != retainedKeysCount
    lastDisplayedRetainedInstanceCount = retainedKeysCount
    if (retainedKeysCount == 0) {
      CanaryLog.d("No retained instances")
      if (countChanged) {
        showNoMoreRetainedInstanceNotification()
      }
      return true
    }

    if (retainedKeysCount < retainedVisibleThreshold) {
      if (applicationVisible || applicationInvisibleLessThanWatchPeriod) {
        CanaryLog.d(
            "Found %d retained instances, which is less than the visible threshold of %d",
            retainedKeysCount,
            retainedVisibleThreshold
        )
        showRetainedCountBelowThresholdNotification(retainedKeysCount, retainedVisibleThreshold)
        scheduleRetainedInstanceCheck(
            "Showing retained instance notification", WAIT_FOR_INSTANCE_THRESHOLD_MILLIS
        )
        return true
      }
    }
    return false
  }
```

Dump heap 会阻塞 UI 线程，为了避免影响平常的开发，LeakCanary  设置了内存泄漏的实例数量阈值，默认是 5，如果在 App 在前台可见时，而且未达到阈值，则会再次 scheduleRetainedInstanceCheck，相当于一个周期轮训任务，在一个周期任务中不断检测内存泄漏的数量。

# 三、Dumping the heap

LeakCanary 使用 AndroidHeapDumper 进行 dump，使用的是 Debug#dumpHprofData，这个没啥好写的，系统提供了 Api。

# 四、Analyzing the heap

分析内存是通过 HeapAnalyzerService 完成的，HeapAnalyzerService 是一个前台服务，继承自 IntentService。

```kotlin
  override fun onHandleIntentInForeground(intent: Intent?) {
    if (intent == null) {
      CanaryLog.d("HeapAnalyzerService received a null intent, ignoring.")
      return
    }
    // Since we're running in the main process we should be careful not to impact it.
    Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND)
    val heapDumpFile = intent.getSerializableExtra(HEAPDUMP_FILE_EXTRA) as File

    if (!heapDumpFile.exists()) {
      throw IllegalStateException(
          "Hprof file missing due to: [${LeakDirectoryProvider.hprofDeleteReason(
              heapDumpFile
          )}] $heapDumpFile"
      )
    }

    val heapAnalyzer = HeapAnalyzer(this)
    val config = LeakCanary.config

    val exclusions = AndroidKnownReference.mapToExclusions(config.knownReferences)

    val heapAnalysis =
      heapAnalyzer.checkForLeaks(
          heapDumpFile, exclusions, config.computeRetainedHeapSize, config.objectInspectors,
          if (config.useExperimentalLeakFinders) config.objectInspectors else listOf(
              AndroidObjectInspectors.KEYED_WEAK_REFERENCE
          )
      )

    config.analysisResultListener(application, heapAnalysis)
  }
```

在 LeakCanary 启动 HeapAnalyzerService 时，会传入一个 File 参数（原来 File 是序列化的），最后是通过 HeapAnalyzer 完成分析的。由于分析过程较为繁琐，请看下一篇。









