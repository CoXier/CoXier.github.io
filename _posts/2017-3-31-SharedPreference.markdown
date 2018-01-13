---
layout:     post
title:      "Android 数据持久化（一）"
author:     "CoXier"
header-img: "img/post-bg.jpg"
tags:

- Android
- 源码解析
---
# 一、Android 数据持久化
Android 中常使用的数据持久化，分为三类：

* 文件：适用视频、音乐、图片等格式的数据
* SharedPreference:适用 key-value 的形式，相比于 Sqlite 更轻量级
* Sqlite ：适用存储和应用相关的对象的各个属性

PS：个人觉得网络存储并不算作 Android 数据持久化。

# 二、SharedPreference 使用

```java
mPreferences = getPreferences(MODE_PRIVATE);
mPreferences.edit().putString("Name","Coxier").apply();
```
这样就把 `Name = Coxier` 存储起来了。

取出数据： `String name = mPreferences.getString("Name","Default");`

# 三、SharedPreference 源码解析

## 3.1 getPreferences & getSharedPreferences
刚刚我在 Activity 中使用的 getPreferences：

```java
public SharedPreferences getPreferences(int mode) {
    return getSharedPreferences(getLocalClassName(), mode);
}
```
这么来看，获取 SharedPreference 都是通过直接调用或者间接调用 getSharedPreferences。

![](http://ofqrfk7np.bkt.clouddn.com/sharedpreference)



## 3.2 ContextImpl#getSharedPreferences
Context 是一个抽象类，`getSharedPreferences` 是通过 [ContextImpl](http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/app/ContextImpl.java) 来实现的。

```java
328    @Override
329    public SharedPreferences getSharedPreferences(String name, int mode) {
330        SharedPreferencesImpl sp;
331        synchronized (ContextImpl.class) {
332            if (sSharedPrefs == null) {
333                sSharedPrefs = new ArrayMap<String, ArrayMap<String, SharedPreferencesImpl>>();
334            }
335
336            final String packageName = getPackageName();
337            ArrayMap<String, SharedPreferencesImpl> packagePrefs = sSharedPrefs.get(packageName);
338            if (packagePrefs == null) {
339                packagePrefs = new ArrayMap<String, SharedPreferencesImpl>();
340                sSharedPrefs.put(packageName, packagePrefs);
341            }
342
               .....
352
353            sp = packagePrefs.get(name);
354            if (sp == null) {
355                File prefsFile = getSharedPrefsFile(name);
356                sp = new SharedPreferencesImpl(prefsFile, mode);
357                packagePrefs.put(name, sp);
358                return sp;
359            }
360        }
           ....
368        return sp;
369    }
```
省略无关代码。分析代码：

* 一个 Application 有一个 packagePrefs，通过 packageName 产生映射关系
* 如果 packagePrefs 没有和 `name` 相对应的 SharedPreferencesImpl 则新建一个 SharedPreferencesImpl ，关联后返回。


## 3.2 SharedPreferencesImpl
SharedPreferencesImpl 是 SharedPreferences 的实现类, Editor 是 Edit 的实现类。从 `putString` 看起：

```java
309        public Editor putString(String key, @Nullable String value) {
310            synchronized (this) {
311                mModified.put(key, value);
312                return this;
313            }
314        }
```
按照使用步骤，下一步应该是 apply 或者 commit。

**commit:**

```java
456        public boolean commit() {
457            MemoryCommitResult mcr = commitToMemory();
458            SharedPreferencesImpl.this.enqueueDiskWrite(
459                mcr, null /* sync write on this thread okay */);
460            try {
461                mcr.writtenToDiskLatch.await();
462            } catch (InterruptedException e) {
463                return false;
464            }
465            notifyListeners(mcr);
466            return mcr.writeToDiskResult;
467        }
```


**apply :**

```java
361        public void apply() {
362            final MemoryCommitResult mcr = commitToMemory();
363            final Runnable awaitCommit = new Runnable() {
364                    public void run() {
365                        try {
366                            mcr.writtenToDiskLatch.await();
367                        } catch (InterruptedException ignored) {
368                        }
369                    }
370                };
371
372            QueuedWork.add(awaitCommit);
373
374            Runnable postWriteRunnable = new Runnable() {
375                    public void run() {
376                        awaitCommit.run();
377                        QueuedWork.remove(awaitCommit);
378                    }
379                };
380
381            SharedPreferencesImpl.this.enqueueDiskWrite(mcr, postWriteRunnable);
               .....
388        }
```
commit 和 apply 之后都会调用 enqueueDiskWrite ：

```java
510    private void enqueueDiskWrite(final MemoryCommitResult mcr,
511                                  final Runnable postWriteRunnable) {
512        final Runnable writeToDiskRunnable = new Runnable() {
513                public void run() {
514                    synchronized (mWritingToDiskLock) {
515                        writeToFile(mcr);
516                    }
517                    synchronized (SharedPreferencesImpl.this) {
518                        mDiskWritesInFlight--;
519                    }
520                    if (postWriteRunnable != null) {
521                        postWriteRunnable.run();
522                    }
523                }
524            };
525
526        final boolean isFromSyncCommit = (postWriteRunnable == null);
527
528        // Typical #commit() path with fewer allocations, doing a write on
529        // the current thread.
530        if (isFromSyncCommit) {
531            boolean wasEmpty = false;
532            synchronized (SharedPreferencesImpl.this) {
533                wasEmpty = mDiskWritesInFlight == 1;
534            }
535            if (wasEmpty) {
536                writeToDiskRunnable.run();
537                return;
538            }
539        }
540
541        QueuedWork.singleThreadExecutor().execute(writeToDiskRunnable);
542    }
```
enqueueDiskWrite 反映了 commit 和 apply 的区别：

commit 传入的  `postWriteRunnable` 为 null ，故 isFromSyncCommit 为 true，因此 `writeToDiskRunnable` 执行完之后就直接返回了；反看 apply 通过线程池来执行，属于异步操作。

**commit 对文件的操作发生在主线程，因此要注意是否会阻塞主线程**



```java
568    private void writeToFile(MemoryCommitResult mcr) {

590
591        // Attempt to write the file, delete the backup and return true as atomically as
592        // possible.  If any exception occurs, delete the new file; next time we will restore
593        // from the backup.
594        try {
595            FileOutputStream str = createFileOutputStream(mFile);
596            if (str == null) {
597                mcr.setDiskWriteResult(false);
598                return;
599            }
600            XmlUtils.writeMapXml(mcr.mapToWriteToDisk, str);
601            FileUtils.sync(str);
602            str.close();
603            ContextImpl.setFilePermissionsFromMode(mFile.getPath(), mMode, 0);
604            try {
605                final StructStat stat = Os.stat(mFile.getPath());
606                synchronized (this) {
607                    mStatTimestamp = stat.st_mtime;
608                    mStatSize = stat.st_size;
609                }
610            } catch (ErrnoException e) {
611                // Do nothing
612            }
613            // Writing was successful, delete the backup file if there is one.
614            mBackupFile.delete();
615            mcr.setDiskWriteResult(true);
616            return;
617        } catch (XmlPullParserException e) {
618            Log.w(TAG, "writeToFile: Got exception:", e);
619        } catch (IOException e) {
620            Log.w(TAG, "writeToFile: Got exception:", e);
621        }
629    }
```
看 600 行，发现是通过把 map 写到 xml 文件中，进而保存 map。具体是怎么写的，我们就暂时不看了。

其实仔细一想，还真是挺有道理的，本来 Android 对 xml 文件（如资源文件）的解析算法已经很好了，而且 xml 文件又那么符合 key-value 的形式。

现在我们来看一下，上面的 demo 中产生的 xml 文件的内容：

```java
adb shell
run-as <app-package-id>

ls /data/data/<app-package-id>/shared_prefs/
```

```java
run-as com.hackerli.sharedpreferencesdemo
ls /data/data/com.hackerli.sharedpreferencesdemo/shared_prefs/
cat /data/data/com.hackerli.sharedpreferencesdemo/shared_prefs/MainActivity.xml
```

MainActivity.xml:

```java
<?xml version='1.0' encoding='utf-8' standalone='yes' ?>
<map>
    <string name="Name">Coxier</string>
</map>
```
