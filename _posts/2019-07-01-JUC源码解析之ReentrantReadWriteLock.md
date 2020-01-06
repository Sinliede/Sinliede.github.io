# JUC源码阅读之ReentrantReadWriteLock

## 前言
ReentrantReadWriteLock原理与ReentrantLock极为相近, 因此本片文章是对ReentrantLock和AbstractQueuedSynchronizer源码阅读的后续, 会比较简略.

## 源码分析
首先来看ReentrantReadWriteLock的属性

```java
/** Inner class providing readlock */
private final ReentrantReadWriteLock.ReadLock readerLock;
/** Inner class providing writelock */
private final ReentrantReadWriteLock.WriteLock writerLock;
/** Performs all synchronization mechanics */
final Sync sync;
```

ReentrantReadWriteLock的属性较为简单, 一个读锁readerLock, 一个写锁writerLock以及一个Sync同步器.再来看下构造方法

```java
/**
 * Creates a new {@code ReentrantReadWriteLock} with
 * default (nonfair) ordering properties.
 */
public ReentrantReadWriteLock() {
    this(false);
}

/**
 * Creates a new {@code ReentrantReadWriteLock} with
 * the given fairness policy.
 *
 * @param fair {@code true} if this lock should use a fair ordering policy
 */
public ReentrantReadWriteLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
    readerLock = new ReadLock(this);
    writerLock = new WriteLock(this);
}
```

ReentrantReadWriteLock提供了两个构造方法, 包含fair和unfair两种模式, 默认是unfair模式.在构造方法中执行读锁和写锁的构造方法, 并将读写锁本身作为参数传递进去. Ok, 接下来看读锁和写锁的代码

```java
public static class ReadLock implements Lock, java.io.Serializable {
    private static final long serialVersionUID = -5992448646407690164L;
    private final Sync sync;

    /**
     * Constructor for use by subclasses
     *
     * @param lock the outer lock object
     * @throws NullPointerException if the lock is null
     */
    protected ReadLock(ReentrantReadWriteLock lock) {
        sync = lock.sync;
    }
    ...
}

public static class WriteLock implements Lock, java.io.Serializable {
    private static final long serialVersionUID = -4992448646407690164L;
    private final Sync sync;

    /**
     * Constructor for use by subclasses
     *
     * @param lock the outer lock object
     * @throws NullPointerException if the lock is null
     */
    protected WriteLock(ReentrantReadWriteLock lock) {
        sync = lock.sync;
    }
    ...
}
```

很清晰地看到读锁和写锁在构造方法中将ReentrantReadWriteLock的同步器lock对象作为自己的lock对象, 因此整个ReentrantReadWriteLock只有一个同步器. 和ReentrantLock一样, ReentrantReadWriteLock使用Sync同步器进行获取锁和释放锁的动作. 接下来看ReadLock和WriteLock如何通过Sync同步器进行锁的获取和释放.

## 写锁

首先看WriteLock
```java
public static class WriteLock implements Lock, java.io.Serializable {

    public void lock() {
        sync.acquire(1);
    }

    public void unlock() {
        sync.release(1);
    }
}

abstract static class Sync extends AbstractQueuedSynchronizer {
    ...
}
```

WriteLock是一个独占锁，当锁未被持有或者被当前线程持有时，才能持有锁，否则挂起等待，和ReentrantLock及其类似, 公平模式下首先查看AQS的等待队列中是否有元素, 有则进入队列, 并阻塞等待, 直到成为队头被唤起获取锁; 否则直接CAS获取锁, 失败则进入队列. 非公平模式少了第一步查看队列中是否有元素等待, 而是直接尝试CAS获取锁, 失败则进入队列，区别在于ReentrantLock中state上限(可重入次数)是Integer.MAX_VALUE, 而WriterLock的上限是2^16, 这里不过多介绍, 详情可参考[2018-12-11-JUC源码解析<一>ReentrantLock和AbstractQueuedSynchronizer](https://sinliede.github.io/2018/12/11/JUC%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90-%E4%B8%80-ReentrantLock%E5%92%8CAbstractQueuedSynchronizer/)。

## 读锁
接下来看读锁
```java
public static class ReadLock implements Lock, java.io.Serializable {
    public void lock() {
        sync.acquireShared(1);
    }

    public void unlock() {
        sync.releaseShared(1);
    }
}

public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}
```

读锁使用sync.acquireShared方法，这是和ReentrantLock不一样的地方, 也是ReentrantReadWriteLock的核心. 看下Sync类对于acquireShared方法的实现

```java
abstract static class Sync extends AbstractQueuedSynchronizer {

  private static final long serialVersionUID = 6317671515068378041L;

  /*
   * Read vs write count extraction constants and functions.
   * Lock state is logically divided into two unsigned shorts:
   * The lower one representing the exclusive (writer) lock hold count,
   * and the upper the shared (reader) hold count.
   * 读锁和写锁共享AQS.state，写锁最大可重入2^16 - 1次，读锁从state = 2^16开始计算。
   */

  static final int SHARED_SHIFT   = 16;
  static final int SHARED_UNIT    = (1 << SHARED_SHIFT);
  static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;
  static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;

  /** Returns the number of shared holds represented in count  */
  /** 无符号右移16位，读锁从2^16开始计数，返回读锁持有锁的次数 */
  static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }
  /** Returns the number of exclusive holds represented in count  */
  /** 因为读锁是从2^16开始计数，因此可以通过按位与计算当前state中写锁持有的次数 */
  static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }

  /** 记录线程号和线程持有锁次数 */
  static final class HoldCounter {
    int count = 0;
    // Use id, not reference, to avoid garbage retention
    final long tid = getThreadId(Thread.currentThread());
  }

  /**
   * ThreadLocal subclass. Easiest to explicitly define for sake
   * of deserialization mechanics.
   */
  static final class ThreadLocalHoldCounter
      extends ThreadLocal<HoldCounter> {
      public HoldCounter initialValue() {
          return new HoldCounter();
      }
  }

  /** 记录当前线程持有锁的次数 */
  private transient ThreadLocalHoldCounter readHolds;

  /** 记录最近一次持有锁的线程 */
  private transient HoldCounter cachedHoldCounter;

  /**
   * 读线程是否需要挂起，在NonFairSync中判断AQS等待队列队头(head节点)是否持有锁，持有则挂起入队；
   * 在FairSync判断AQS等待队列中是否有线程在等待，有则挂起入队；
   */
  abstract boolean readerShouldBlock();
}

介绍完了ReentrantReadWriteLock.Sync 类的基础内容，那么可以看下读线程获取锁的实现了

```java
protected final int tryAcquireShared(int unused) {
    /*
     * Walkthrough:
     * 1. If write lock held by another thread, fail.
     * 2. Otherwise, this thread is eligible for
     *    lock wrt state, so ask if it should block
     *    because of queue policy. If not, try
     *    to grant by CASing state and updating count.
     *    Note that step does not check for reentrant
     *    acquires, which is postponed to full version
     *    to avoid having to check hold count in
     *    the more typical non-reentrant case.
     * 3. If step 2 fails either because thread
     *    apparently not eligible or CAS fails or count
     *    saturated, chain to version with full retry loop.
     */
    Thread current = Thread.currentThread();
    int c = getState();
    //判断是否有写线程持有锁
    if (exclusiveCount(c) != 0 &&
        //当前线程没有持有锁
        getExclusiveOwnerThread() != current)
        //返回尝试持有锁失败，进入AQS等待队列
        return -1;
    //获取读锁当前持有次数
    int r = sharedCount(c);
    //读线程不需要等待
    if (!readerShouldBlock() &&
        //读锁持有次数未超限
        r < MAX_COUNT &&
        //CAS尝试获取锁成功，注意这段代码，意味着多个线程可以同时持有读锁，只要CAS能够成功
        compareAndSetState(c, c + SHARED_UNIT)) {
        //读锁第一次持有锁
        if (r == 0) {
            //记录第一个读线程和持有锁次数
            firstReader = current;
            firstReaderHoldCount = 1;
        //当前线程正是第一个读线程
        } else if (firstReader == current) {
            //第一个读线程持有锁次数自增
            firstReaderHoldCount++;
        } else {
            /不是第一个读线程，将线程信息存入HoldCounter并更新HoldCounter缓存
            HoldCounter rh = cachedHoldCounter;
            if (rh == null || rh.tid != getThreadId(current))
                cachedHoldCounter = rh = readHolds.get();
            else if (rh.count == 0)
                readHolds.set(rh);
            rh.count++;
        }
        //返回获取锁成功
        return 1;
    }
    //如果读线程需要等待or读锁持有次数达到上限or CAS竞争失败，进入此方法
    return fullTryAcquireShared(current);
}


final int fullTryAcquireShared(Thread current) {
    /*
     * This code is in part redundant with that in
     * tryAcquireShared but is simpler overall by not
     * complicating tryAcquireShared with interactions between
     * retries and lazily reading hold counts.
     */
    HoldCounter rh = null;
    for (;;) {
        int c = getState();
        //再次判断锁是否被写线程持有，是则返回尝试获取锁失败
        if (exclusiveCount(c) != 0) {
            if (getExclusiveOwnerThread() != current)
                return -1;
            // else we hold the exclusive lock; blocking here
            // would cause deadlock.
            //读线程需要等待
        } else if (readerShouldBlock()) {
            // Make sure we're not acquiring read lock reentrantly
            if (firstReader == current) {
                // assert firstReaderHoldCount > 0;
            } else {
                if (rh == null) {
                    rh = cachedHoldCounter;
                    if (rh == null || rh.tid != getThreadId(current)) {
                        rh = readHolds.get();
                        if (rh.count == 0)
                            readHolds.remove();
                    }
                }
                //当前线程没有持有锁，又需要等待，那么直接返回尝试失败，入队
                if (rh.count == 0)
                    return -1;
            }
        }
        //读锁达到上限
        if (sharedCount(c) == MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        //再次尝试获取锁，成功则更新cachedHoldCounter缓存为当前HoldCounter
        if (compareAndSetState(c, c + SHARED_UNIT)) {
            if (sharedCount(c) == 0) {
                firstReader = current;
                firstReaderHoldCount = 1;
            } else if (firstReader == current) {
                firstReaderHoldCount++;
            } else {
                if (rh == null)
                    rh = cachedHoldCounter;
                if (rh == null || rh.tid != getThreadId(current))
                    rh = readHolds.get();
                else if (rh.count == 0)
                    readHolds.set(rh);
                rh.count++;
                cachedHoldCounter = rh; // cache for release
            }
            return 1;
        }
        //再次CAS失败，继续循环，直到成功或者
    }
}
```

ReentrantReadWriteLock在AQS的基础上对state标志位进行了非常精巧地利用，将一把锁同时作为写锁和读锁，实现了读锁和写锁互斥，读锁并行的锁策略。大致流程如下图

![png](https://raw.githubusercontent.com/Sinliede/Sinliede.github.io/master/static/img/ReentrantReadWriteLock.png)
