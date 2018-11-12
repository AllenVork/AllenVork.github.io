---
layout:     post
title:      SharedPreferenceImpl 源码解析
subtitle:   解决使用异步提交操作 apply 也会出现 ANR 的困惑
header-img: img/android3.png
author:     Allen Vork
catalog: true
tags:
    - android basics
---

## Synopsis
如果有使用到 SharedPreference 的同学，采用 apply 操作来异步提交数据到本地的话，可能就会在崩溃平台上遇到这种 ANR：
```
DALVIK THREADS (62):
"main" prio=5 tid=1 Waiting
  | group="main" sCount=1 dsCount=0 obj=0x76eb4a20 self=0x7f8f09a000
  | sysTid=20196 nice=0 cgrp=default sched=0/0 handle=0x7f9333beb0
  | state=S schedstat=( 49400463809 21820262476 111431 ) utm=4144 stm=796 core=0 HZ=100
  | stack=0x7fd4bdc000-0x7fd4bde000 stackSize=8MB
  | held mutexes=
  at java.lang.Object.wait!(Native method)
  - waiting on <0x39ae9327> (a java.lang.Object)
  at java.lang.Thread.parkFor(Thread.java:1220)
  - locked <0x39ae9327> (a java.lang.Object)
  at sun.misc.Unsafe.park(Unsafe.java:299)
  at java.util.concurrent.locks.LockSupport.park(LockSupport.java:157)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer.parkAndCheckInterrupt(AbstractQueuedSynchronizer.java:813)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer.doAcquireSharedInterruptibly(AbstractQueuedSynchronizer.java:973)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireSharedInterruptibly(AbstractQueuedSynchronizer.java:1281)
  at java.util.concurrent.CountDownLatch.await(CountDownLatch.java:202)
  at android.app.SharedPreferencesImpl$EditorImpl$1.run(SharedPreferencesImpl.java:363)
  at android.app.QueuedWork.waitToFinish(QueuedWork.java:88)
  at android.app.ActivityThread.handleStopActivity(ActivityThread.java:3900)
  at android.app.ActivityThread.access$1200(ActivityThread.java:187)
  at android.app.ActivityThread$H.handleMessage(ActivityThread.java:1614)
  at android.os.Handler.dispatchMessage(Handler.java:111)
  at android.os.Looper.loop(Looper.java:192)
  at android.app.ActivityThread.main(ActivityThread.java:5886)
  at java.lang.reflect.Method.invoke!(Native method)
  at java.lang.reflect.Method.invoke(Method.java:372)
  at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:1031)
  at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:826)
```
我们通过解析 SharedPreferenceImpl 源码来看问题是怎么出现的。

## Analysis
```java
final class SharedPreferencesImpl implements SharedPreferences {

    SharedPreferencesImpl(File file, int mode) {
        mFile = file;
        mBackupFile = makeBackupFile(file);
        mMode = mode;
        mLoaded = false; // 判断文件是否加载完成
        mMap = null; //
        startLoadFromDisk();
    }

    // 根据传进来的 file 的路径，创建一个 mBackupFile
    static File makeBackupFile(File prefsFile) {
        return new File(prefsFile.getPath() + ".bak");
    }

    private void startLoadFromDisk() {
        synchronized (mLock) { // 将状态置为未加载状态
            mLoaded = false;
        }
        new Thread("SharedPreferencesImpl-load") {
            public void run() {
                loadFromDisk();
            }
        }.start();
    }

}
```
构造函数里面主要做2件事，根据传进来的 file 创建一个 mBackupFile，然后在子线程中调用 loadFromDisk()。
```java
    private void loadFromDisk() {
        synchronized (mLock) {
            if (mLoaded) { // 这句为什么不放到锁外面？
                return;
            }
            if (mBackupFile.exists()) { // 如果存再备份文件，则优先使用备份文件
                mFile.delete();
                mBackupFile.renameTo(mFile);
            }
        }

        Map map = null;
        StructStat stat = null;
        try {
            stat = Os.stat(mFile.getPath());
            if (mFile.canRead()) {
                BufferedInputStream str = null;
                try {
                    // 将文件里的数据都出来，存放到 map 中
                    str = new BufferedInputStream(
                            new FileInputStream(mFile), 16*1024);
                    map = XmlUtils.readMapXml(str);
                } catch (Exception e) {
                    Log.w(TAG, "Cannot read " + mFile.getAbsolutePath(), e);
                } finally {
                    IoUtils.closeQuietly(str);
                }
            }
        } catch (ErrnoException e) {
            /* ignore */
        }

        synchronized (mLock) {
            mLoaded = true; // 加载完成
            if (map != null) {
                mMap = map;
                mStatTimestamp = stat.st_mtime;
                mStatSize = stat.st_size;
            } else {
                mMap = new HashMap<>();
            }
            mLock.notifyAll(); // 唤醒等待线程
        }
    }
```
可以看出，这个方法就是读取传进来的 file 文件里的数据并存放到 map 中。构造完成后就可以读数据了：
```java
    public long getLong(String key, long defValue) {
        synchronized (mLock) {
            awaitLoadedLocked();
            // 从 mMap 中读取数据返回
            Long v = (Long)mMap.get(key);
            return v != null ? v : defValue;
        }
    }

    private void awaitLoadedLocked() {
        if (!mLoaded) {
            // Raise an explicit StrictMode onReadFromDisk for this
            // thread, since the real read will be in a different
            // thread and otherwise ignored by StrictMode.
            BlockGuard.getThreadPolicy().onReadFromDisk();
        }
        while (!mLoaded) { // 如果没加载完就等待
            try {
                mLock.wait();
            } catch (InterruptedException unused) {
            }
        }
    }
```
可以看出读取操作会先判断数据是否已经从本地读取到内存，如果没有的话，会等待。当数据读取完成后就会调用 mLock.notifyAll()，这时就会将 key 对应的 value 从 map 中返回。 再来看下写操作，写操作包括 commit() 和 apply() 进行提交。先看 commit()：
```java
    public Editor edit() {
        // TODO: remove the need to call awaitLoadedLocked() when
        // requesting an editor.  will require some work on the
        // Editor, but then we should be able to do:
        //
        //      context.getSharedPreferences(..).edit().putString(..).apply()
        //
        // ... all without blocking.
        synchronized (mLock) {
            awaitLoadedLocked();
        }

        return new EditorImpl();
    }

    public final class EditorImpl implements Editor {
        private final Object mLock = new Object();

        @GuardedBy("mLock")
        private final Map<String, Object> mModified = Maps.newHashMap();

        public Editor putStringSet(String key, @Nullable Set<String> values) {
            synchronized (mLock) {
                // 写操作是单独存放到一个新的 mModified 中，而不是前面的 mMap
                mModified.put(key,
                        (values == null) ? null : new HashSet<String>(values));
                return this;
            }
        }

        public boolean commit() {
            long startTime = 0;

            MemoryCommitResult mcr = commitToMemory(); // 先提交到内存

            SharedPreferencesImpl.this.enqueueDiskWrite(
                mcr, null /* sync write on this thread okay */); // 然后提交到硬盘
            try {
                // 这里就是导致 ANR 的核心，后面会讲
                mcr.writtenToDiskLatch.await(); // 等硬盘写完
            } catch (InterruptedException e) {
                return false;
            }
            notifyListeners(mcr);
            return mcr.writeToDiskResult;
        }
```
调用 context.getSharedPreferences(..).edit().putString(..).commit() 的 edit() 方法会返回一个 EditorImpl 对象。然后调用该方法的 putxxx() 将数据保存到一个新的 HashMap 中。然后调用 commit() 方法将数据提交到内存和硬盘中。我们先来看下这个提交到内存的方法是干啥的：
```java
        // Returns true if any changes were made
        private MemoryCommitResult commitToMemory() {
            long memoryStateGeneration;
            List<String> keysModified = null;
            Set<OnSharedPreferenceChangeListener> listeners = null;
            Map<String, Object> mapToWriteToDisk;

            synchronized (SharedPreferencesImpl.this.mLock) { // 加 mLock 锁代表写内存的时候不允许读
                // We optimistically don't make a deep copy until
                // a memory commit comes in when we're already
                // writing to disk.
                if (mDiskWritesInFlight > 0) {
                    // We can't modify our mMap as a currently
                    // in-flight write owns it.  Clone it before
                    // modifying it.
                    // noinspection unchecked
                    mMap = new HashMap<String, Object>(mMap);
                }
                mapToWriteToDisk = mMap;
                mDiskWritesInFlight++;

                synchronized (mLock) { //对当前的Editor加锁
                    boolean changesMade = false;

                    if (mClear) { // 当调用了 clear() 方法后，mClear 才为 true
                        if (!mMap.isEmpty()) {
                            changesMade = true;
                            mMap.clear(); // 清空mMap。mMap里面存的是整个的Preferences
                        }
                        mClear = false;
                    }

                    // 遍历 mModified，并将新数据添加到 mMap 中
                    for (Map.Entry<String, Object> e : mModified.entrySet()) {
                        String k = e.getKey();
                        Object v = e.getValue();
                        // "this" is the magic value for a removal mutation. In addition,
                        // setting a value to "null" for a given key is specified to be
                        // equivalent to calling remove on that key.
                        if (v == this || v == null) { // 调用 remove 方法时，v 被设置为 this
                            if (!mMap.containsKey(k)) {
                                continue;
                            }
                            mMap.remove(k);
                        } else {
                            if (mMap.containsKey(k)) {
                                Object existingValue = mMap.get(k);
                                if (existingValue != null && existingValue.equals(v)) {
                                    continue;
                                }
                            }
                            mMap.put(k, v); // 将 put 进来的有效值存放到 mMap 中
                        }

                        changesMade = true; // 表示 mMap 中的数据又被修改
                    }

                    mModified.clear();

                    if (changesMade) {
                        mCurrentMemoryStateGeneration++;
                    }

                    memoryStateGeneration = mCurrentMemoryStateGeneration;
                }
            }
            // 返回 MemoryCommitResult 对象，该对象是用于存放这些数据的
            return new MemoryCommitResult(memoryStateGeneration, keysModified, listeners,
                    mapToWriteToDisk);
        }
```
commit 操作就是将 put 到 mModified 中的数据存放到 mMap 中。来看下 MemoryCommitResult：
```java
    private static class MemoryCommitResult {
        final long memoryStateGeneration;
        @Nullable final List<String> keysModified;
        @Nullable final Set<OnSharedPreferenceChangeListener> listeners;
        final Map<String, Object> mapToWriteToDisk;
        final CountDownLatch writtenToDiskLatch = new CountDownLatch(1);

        @GuardedBy("mWritingToDiskLock")
        volatile boolean writeToDiskResult = false;
        boolean wasWritten = false;

        private MemoryCommitResult(long memoryStateGeneration, @Nullable List<String> keysModified,
                @Nullable Set<OnSharedPreferenceChangeListener> listeners,
                Map<String, Object> mapToWriteToDisk) {
            this.memoryStateGeneration = memoryStateGeneration;
            this.keysModified = keysModified;
            this.listeners = listeners;
            this.mapToWriteToDisk = mapToWriteToDisk;
        }

        // 外部调用 writtenToDiskLatch.await() 来等待 enqueueDiskWrite 执行完成，
        // 然后通过调用这个方法来继续执行后面的操作
        void setDiskWriteResult(boolean wasWritten, boolean result) {
            this.wasWritten = wasWritten;
            writeToDiskResult = result;
            writtenToDiskLatch.countDown();
        }
    }
```
这个类里面主要是由一个 CountDownLatch，在上面 commit() 中，在执行写硬盘操作后会调用 mcr.writtenToDiskLatch.await() 来等待写操作完成。我们来看写硬盘操作：
```java
    /**
     * Enqueue an already-committed-to-memory result to be written
     * to disk.
     *
     * They will be written to disk one-at-a-time in the order
     * that they're enqueued.
     *
     * @param postWriteRunnable if non-null, we're being called
     *   from apply() and this is the runnable to run after
     *   the write proceeds.  if null (from a regular commit()),
     *   then we're allowed to do this disk write on the main
     *   thread (which in addition to reducing allocations and
     *   creating a background thread, this has the advantage that
     *   we catch them in userdebug StrictMode reports to convert
     *   them where possible to apply() ...)
     */
    private void enqueueDiskWrite(final MemoryCommitResult mcr,
                                  final Runnable postWriteRunnable) {
        // commit() 方法传进来的为 null，此时为 true
        final boolean isFromSyncCommit = (postWriteRunnable == null);

        final Runnable writeToDiskRunnable = new Runnable() {
                public void run() {
                    synchronized (mWritingToDiskLock) {
                        writeToFile(mcr, isFromSyncCommit); // 写文件
                    }
                    synchronized (mLock) {
                        mDiskWritesInFlight--;
                    }
                    if (postWriteRunnable != null) {
                        postWriteRunnable.run();
                    }
                }
            };

        // commit() 方法会执行这里，它在当前线程中执行，占用更少的内存
        if (isFromSyncCommit) {
            boolean wasEmpty = false;
            synchronized (mLock) { 
                wasEmpty = mDiskWritesInFlight == 1;
            }
            if (wasEmpty) { // 当前只有一个批次等待写入
                writeToDiskRunnable.run(); // 直接在当前线程执行
                return;
            }
        }

        // 如果调用的是 apply() 或者当前有多个批次等待写入，那么另起线程写入
        QueuedWork.queue(writeToDiskRunnable, !isFromSyncCommit);
    }

```
前面传进来的第二个参数为 null，那么会执行 if 语句，如果当前只有一个批次等待写入，就会在当前线程执行 writeToDiskRunnable 调用 writeToFile 方法。否则就调用 QueuedWork.queue() 。我们来看下 writeToFile：
```java
    private void writeToFile(MemoryCommitResult mcr, boolean isFromSyncCommit) {
        long startTime = 0;
        long existsTime = 0;
        long backupExistsTime = 0;
        long outputStreamCreateTime = 0;
        long writeTime = 0;
        long fsyncTime = 0;
        long setPermTime = 0;
        long fstatTime = 0;
        long deleteTime = 0;

        boolean fileExists = mFile.exists();

        // Rename the current file so it may be used as a backup during the next read
        if (fileExists) {
            boolean needsWrite = false;

            // Only need to write if the disk state is older than this commit
            if (mDiskStateGeneration < mcr.memoryStateGeneration) {
                if (isFromSyncCommit) {
                    needsWrite = true; // commit 会调用到这里
                } else {
                    synchronized (mLock) {
                        // No need to persist intermediate states. Just wait for the latest state to
                        // be persisted.
                        if (mCurrentMemoryStateGeneration == mcr.memoryStateGeneration) {
                            needsWrite = true;
                        }
                    }
                }
            }

            if (!needsWrite) {
                mcr.setDiskWriteResult(false, true);
                return;
            }

            boolean backupFileExists = mBackupFile.exists();

            // 从前面的实现可以看出，它是将文件中所有的数据都取出来，然后添加/修改里面的数据
            // 那么写的时候就是全量写到该文件中，老文件的数据没用了，但为了防止出现异常导致数据丢失
            // 如果没有备份的话，先要备份下。否则的话，将源文件删除。
            if (!backupFileExists) { // 先备份
                if (!mFile.renameTo(mBackupFile)) {
                    Log.e(TAG, "Couldn't rename file " + mFile
                          + " to backup file " + mBackupFile);
                    mcr.setDiskWriteResult(false, false);
                    return;
                }
            } else {
                mFile.delete();
            }
        }

        // Attempt to write the file, delete the backup and return true as atomically as
        // possible.  If any exception occurs, delete the new file; next time we will restore
        // from the backup.
        try {
            FileOutputStream str = createFileOutputStream(mFile);

            if (DEBUG) {
                outputStreamCreateTime = System.currentTimeMillis();
            }

            if (str == null) { // 释放锁
                mcr.setDiskWriteResult(false, false);
                return;
            }
            XmlUtils.writeMapXml(mcr.mapToWriteToDisk, str);

            writeTime = System.currentTimeMillis();

            FileUtils.sync(str); // 写硬盘

            fsyncTime = System.currentTimeMillis();

            str.close();
            ContextImpl.setFilePermissionsFromMode(mFile.getPath(), mMode, 0);

            try {
                final StructStat stat = Os.stat(mFile.getPath());
                synchronized (mLock) {
                    mStatTimestamp = stat.st_mtime;
                    mStatSize = stat.st_size;
                }
            } catch (ErrnoException e) {
                // Do nothing
            }

            // 写成功后，备份文件就不需要了
            mBackupFile.delete();
            // 更新 mDiskStateGeneration 用于上面的判断
            mDiskStateGeneration = mcr.memoryStateGeneration;

            // 硬盘写完了后，这里就会调用 countDown() 方法来释放锁
            mcr.setDiskWriteResult(true, true);

            long fsyncDuration = fsyncTime - writeTime;
            mSyncTimes.add(Long.valueOf(fsyncDuration).intValue());
            mNumSync++;

            return;
        } catch (XmlPullParserException e) {
            Log.w(TAG, "writeToFile: Got exception:", e);
        } catch (IOException e) {
            Log.w(TAG, "writeToFile: Got exception:", e);
        }

        // Clean up an unsuccessfully written file
        if (mFile.exists()) {
            if (!mFile.delete()) {
                Log.e(TAG, "Couldn't clean up partially-written file " + mFile);
            }
        }
        mcr.setDiskWriteResult(false, false);
    }
```
这里主要就是将变更写到硬盘中，写完后就会调用 mcr.setDiskWriteResult(true, true) 来执行 countDown()。我们再来看 apply() 方法：
```java
        public void apply() {
            final long startTime = System.currentTimeMillis();
            // 和 commit 一样，将数据从 mModified 存放到 mMap 中
            final MemoryCommitResult mcr = commitToMemory();
            final Runnable awaitCommit = new Runnable() {
                    public void run() {
                        try {
                            mcr.writtenToDiskLatch.await(); // 等待写硬盘
                        } catch (InterruptedException ignored) {
                        }
                    }
                };

            // 产生 ANR 的关键
            QueuedWork.addFinisher(awaitCommit);

            // 这个 runnable 就是为了执行上面的 awaitCommit Runnable，从而调用 mcr.writtenToDiskLatch.await()
            Runnable postWriteRunnable = new Runnable() {
                    public void run() {
                        awaitCommit.run();
                        QueuedWork.removeFinisher(awaitCommit);
                    }
                };

            SharedPreferencesImpl.this.enqueueDiskWrite(mcr, postWriteRunnable);

            // Okay to notify the listeners before it's hit disk
            // because the listeners should always get the same
            // SharedPreferences instance back, which has the
            // changes reflected in memory.
            notifyListeners(mcr);
        }

```
我们来看下 enqueueDiskWrite()：
```java
    private void enqueueDiskWrite(final MemoryCommitResult mcr,
                                  final Runnable postWriteRunnable) {
        final boolean isFromSyncCommit = (postWriteRunnable == null); // 此时为 false

        // 调用 writeToFile 来写硬盘，并调用 mcr.writtenToDiskLatch.await() 来等待写入完成
        final Runnable writeToDiskRunnable = new Runnable() {
                public void run() {
                    synchronized (mWritingToDiskLock) {
                        writeToFile(mcr, isFromSyncCommit);
                    }
                    synchronized (mLock) {
                        mDiskWritesInFlight--;
                    }
                    if (postWriteRunnable != null) {
                        // 执行 mcr.writtenToDiskLatch.await() 来等待写硬盘完成 
                        postWriteRunnable.run();
                    }
                }
            };
        ...
        // 此时直接将该 runnable 传到 QueuedWork 中
        QueuedWork.queue(writeToDiskRunnable, !isFromSyncCommit);
    }
```
可以看出 apply() 方法就是将一个 runnable 传给 QueuedWork，这个 runnable 是用来执行写硬盘操作，并且会阻塞当前线程。
```java
public class QueuedWork {
    public static void queue(Runnable work, boolean shouldDelay) {
        Handler handler = getHandler();

        synchronized (sLock) {
            sWork.add(work); // 将 runnable 添加到 sWork 中

            // 发送消息
            if (shouldDelay && sCanDelay) {
                handler.sendEmptyMessageDelayed(QueuedWorkHandler.MSG_RUN, DELAY);
            } else {
                handler.sendEmptyMessage(QueuedWorkHandler.MSG_RUN);
            }
        }
    }

    private static Handler getHandler() {
        synchronized (sLock) {
            if (sHandler == null) {
                HandlerThread handlerThread = new HandlerThread("queued-work-looper",
                        Process.THREAD_PRIORITY_FOREGROUND);
                handlerThread.start();

                sHandler = new QueuedWorkHandler(handlerThread.getLooper());
            }
            return sHandler;
        }
    }

    private static class QueuedWorkHandler extends Handler {
        static final int MSG_RUN = 1;

        QueuedWorkHandler(Looper looper) {
            super(looper);
        }

        public void handleMessage(Message msg) {
            if (msg.what == MSG_RUN) {
                processPendingWork();
            }
        }
    }

    private static void processPendingWork() {
        long startTime = 0;

        synchronized (sProcessingWork) {
            LinkedList<Runnable> work;

            synchronized (sLock) {
                work = (LinkedList<Runnable>) sWork.clone(); // 取出前面放进去的 Runnable 列表
                sWork.clear();

                // Remove all msg-s as all work will be processed now
                getHandler().removeMessages(QueuedWorkHandler.MSG_RUN);
            }

            if (work.size() > 0) {
                for (Runnable w : work) {
                    // 在 HandlerThread 中执行写硬盘操作，并调用 mcr.writtenToDiskLatch.await()
                    // 来等待写完成
                    w.run();
                }
            }
        }
    }
}
```
可以看出 apply() 方法就是将写硬盘操作放到 HandlerThread 中异步执行的。    


到此，SharedPreferenceImpl 源码都分析完了，大致就是创建 SharedPreferenceImpl 时会去本地读取文件中所有的数据到 mMap，中，后面的读取操作就是直接从 mMap 中获取 value。写操作会先写到 mModified 的 HashMap 中，然后将其写到 mMap 中，再将 mMap 写到硬盘。写硬盘要区分 commit() 操作还是 apply() 操作，commit() 操作是在当前线程中写到硬盘中，而 apply 操作会将写硬盘操作封装到 Runnable 交给 QueuedWork 中的 Handler 在 HandlerThread 中异步执行。    

我们再回到上面的 ANR 问题，出现 ANR 的堆栈信息为：
```java
  at java.util.concurrent.CountDownLatch.await(CountDownLatch.java:202)
  at android.app.SharedPreferencesImpl$EditorImpl$1.run(SharedPreferencesImpl.java:363)
  at android.app.QueuedWork.waitToFinish(QueuedWork.java:88)
  at android.app.ActivityThread.handleStopActivity(ActivityThread.java:3900)
```
我们来看下 waitToFinish()：
```java
    /**
     * Trigger queued work to be processed immediately. The queued work is processed on a separate
     * thread asynchronous. While doing that run and process all finishers on this thread. The
     * finishers can be implemented in a way to check weather the queued work is finished.
     *
     * Is called from the Activity base class's onPause(), after BroadcastReceiver's onReceive,
     * after Service command handling, etc. (so async work is never lost)
     */
    public static void waitToFinish() {
        ...
        try {
            processPendingWork();
        } finally {
            StrictMode.setThreadPolicy(oldPolicy);
        }

        try {
            while (true) {
                Runnable finisher;

                // 将前面 apply() 放进来的封装有 mcr.writtenToDiskLatch.await() 的 runnalbe 取出来
                synchronized (sLock) {
                    finisher = sFinishers.poll();
                }

                if (finisher == null) {
                    break;
                }
                // 在当前线程等待，导致 ANR
                finisher.run();
            }
        } finally {
            sCanDelay = true;
        }
    }

    private static void processPendingWork() {
        long startTime = 0;

        // 当 下面的 run 方法没执行完，则会一直等它执行完
        synchronized (sProcessingWork) {
            LinkedList<Runnable> work;

            synchronized (sLock) {
                work = (LinkedList<Runnable>) sWork.clone();
                sWork.clear();

                // Remove all msg-s as all work will be processed now
                getHandler().removeMessages(QueuedWorkHandler.MSG_RUN);
            }

            if (work.size() > 0) {
                for (Runnable w : work) {
                    w.run();
                }
            }
        }
    }
```
可以看出 waitToFinish 会一直等待 run 方法执行完，所以虽然 run 是在 HandlerThread 中执行的，它依然会阻塞主线程。所以问题就是 Activity 在 stop 的时候要在当前线程等待子线程中所有的 runnable 执行完成导致 ANR。

## solution
+ 直接异步调用 commit() 方法，这样就不会调用 apply() 的 QueuedWork.addFinisher(awaitCommit)，那么 sFinishers 就为空，当 ActivityThread 调用 pause 之类的方法时，也不会在主线程等待写入操作完成。
+ 手动清空 sFinishers 中的 Runnable:    
```java
/**
 * 摘自文末的引用
 */
public static void tryHookActityThreadH() {
    boolean hookSuccess = false;
    try {
        Class activityThread = Class.forName("android.app.ActivityThread");
        Method mH = ReflectionCache.build().getMethod(activityThread, "currentActivityThread");
        if (mH != null) {
            Object obj = mH.invoke(activityThread);
            if (obj != null) {
                Handler handler = (Handler) ReflectHelper.getField(obj, "mH");
                if (handler != null) {
                    Field mCallbackField = ReflectionCache.build().getDeclaredField(Class.forName("android.os.Handler"), "mCallback");
                    if (mCallbackField != null) {
                        mCallbackField.setAccessible(true);
                        ActivityThreadHCallbackProxy activityThreadHandler = new ActivityThreadHCallbackProxy((Handler.Callback) mCallbackField.get(handler));
                        mCallbackField.set(handler, activityThreadHandler);
                        hookSuccess = true;
                    }
                }
            }
        }
    } catch (Exception e) {
        Mlog.tag(TAG).i("HookActityThreadH:{}", e.getLocalizedMessage());
    }
    Mlog.tag(TAG).i("HookActityThreadH:{}", hookSuccess);
}


public class ActivityThreadHCallbackProxy implements Handler.Callback {

    public static final String TAG = "ActivityThreadHCallbackProxy";
    /**
    * {@link ActivityThread#H}
    */
        public static final int PAUSE_ACTIVITY          = 101;
        public static final int PAUSE_ACTIVITY_FINISHING= 102;
        public static final int STOP_ACTIVITY_SHOW      = 103;
        public static final int STOP_ACTIVITY_HIDE      = 104;
        public static final int SERVICE_ARGS            = 115;
        public static final int STOP_SERVICE            = 116;
        public static final int SLEEPING                = 137;


    private Handler.Callback mRawCallback;

    public ActivityThreadHCallbackProxy(Handler.Callback callback) {
        mRawCallback = callback;
    }

    @Override
    public boolean handleMessage(Message message) {
        switch (message.what) {
            case STOP_ACTIVITY_HIDE:
            case STOP_ACTIVITY_SHOW:
                //stop activity
                beforeWaitToFinished();
                break;
            case SERVICE_ARGS:
                //SERVICE ARGS
                beforeWaitToFinished();
                break;
            case STOP_SERVICE:
                //STOP SERVICE
                beforeWaitToFinished();
                break;

            case SLEEPING:
                //SLEEPING
                beforeWaitToFinished();
                break;
            case PAUSE_ACTIVITY:
            case PAUSE_ACTIVITY_FINISHING:
                //pause activity
                beforeWaitToFinished();
                break;
            default:
                break;
        }
        if (mRawCallback != null) {
            mRawCallback.handleMessage(message);
        }
        return false;//不能返回true，否则会消耗掉事件
    }

    private void beforeWaitToFinished() {
        QuenedWorkProxy.cleanAll();
    }
}

public class QuenedWorkProxy {
    private static final String TAG = "QuenedWorkProxy";
    private static final String CLASS_NAME = "android.app.QueuedWork";
    private static final String FILE_NAME_PENDDING_WORK_FINISH = "sPendingWorkFinishers";

    public static Collection<Runnable> sPendingWorkFinishers = null;
    private static boolean sSupportHook = true;

        /**
        * 不支持android O
        * android O变量名改为sFinishers
        */
    public static void cleanAll(){
            if (sPendingWorkFinishers == null && sSupportHook) {
            try {
               sPendingWorkFinishers = (ConcurrentLinkedQueue<Runnable>) ReflectHelper.getStaticField(CLASS_NAME, FILE_NAME_PENDDING_WORK_FINISH);
                } catch (Exception e) {
                    Mlog.tag(TAG).w("{}", e.getLocalizedMessage());
                    sSupportHook = false;
                }
            }
            if(sPendingWorkFinishers != null){
                Mlog.tag(TAG).d("clean QuenedWork.sPendingWorkFinishers({}) size {}" , sPendingWorkFinishers.hashCode() , sPendingWorkFinishers.size());
                sPendingWorkFinishers.clear();
            }
        }
}
```

## 参考文献
+ android-26 源码
+ [Android-SharedPreferences源码学习与最佳实践](https://www.2cto.com/kf/201312/268547.html)   
+ [一个由SHAREDPREFERENCES引起的ANR](https://microstudent.github.io/2018/09/10/android-hack1/)