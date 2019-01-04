## 1.前言
上篇文章我们通过阅读ThreadPoolExecutor的源码分析了线程池的线程与任务的调度原理。同样的，这篇文章会通过阅读源码的方式，尝试分析线程池一个非常重要的特性：不主动关闭。ps:

1.食用本文前需要对AbstractQueuedSynchronizer的原理有一定的理解，如果你尚不了解，可以阅读我的博客[2018-12-11-JUC源码解析<一>ReentrantLock和AbstractQueuedSynchronizer](https://sinliede.github.io/2018/12/11/JUC%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90-%E4%B8%80-ReentrantLock%E5%92%8CAbstractQueuedSynchronizer/)。

2.如果你对线程池的任务调度原理尚不了解，可以阅读我的上篇博客[2018-12-20-JUC源码解析<二>ThreadPoolExecutor<一>任务与线程的调度原理](https://sinliede.github.io/2018/12/20/JUC%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90-%E4%BA%8C-ThreadPoolExecutor-%E4%B8%80-%E4%BB%BB%E5%8A%A1%E4%B8%8E%E7%BA%BF%E7%A8%8B%E7%9A%84%E8%B0%83%E5%BA%A6%E5%8E%9F%E7%90%86)。

## 2.问题引入
回顾上篇文章中我们对线程池的使用方式
```java
BlockingQueue<Runnable> blockingQueue = new LinkedBlockingQueue<>();
ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(20, 20, 60L, TimeUnit.SECONDS, blockingQueue);
for (int i = 0; i < 10000; i++) {
    threadPoolExecutor.submit(() -> {
      int taskCount = i;
        try {
            doTimeConsumingTask();  //耗时任务
            LOGGER.info("第{}个任务执行完毕", taskCount);
        } catch (Exception e) {
            e.printStackTrace();
        }
    });
}
```
当第10000个耗时任务实行完毕并输出日志后，我们会发现我们的程序没有停止，线程池依然在运行。

我们知道，线程池通过execute方法提交task，工作线程数目不够的情况下通过workQueue.offer方法将task存入workQueue，而后工作线程在runWorker()方法中循环从workQueue中通过getTask取出任务并执行，直到workQueue为空，核心工作线程coreThreads会在workQueue.take方法中挂起，直到workQueue不为空。

我们猜想是不是因为工作线程中workQueue.take方法的挂起导致了线程池无法主动关闭。我们通过一段测试程序验证这个猜测
```java
public static void main(String[] args) {
    BlockingQueue workQueue = new LinkedBlockingQueue();
    Thread thread = new Thread(()->{
        try {
            Object o = workQueue.take();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    });
    thread.start();
    LOGGER.info("主线程任务执行完毕");
}
```
上方的程序中我们模拟了线程池在没有任务时的工作，我们有一个空的任务队列workQueue，还有一个工作线程thread想要通过take方法取得任务。当主线程任务执行完毕的日志输出后，我们发现程序没有自动退出，这正好验证了我们的猜想，也就是workQueue.take方法阻止了线程池的结束。

## 3.源码分析
```java
/**
 * Inserts the specified element into this queue if it is possible to do
 * so immediately without violating capacity restrictions, returning
 * {@code true} upon success and {@code false} if no space is currently
 * available.  When using a capacity-restricted queue, this method is
 * generally preferable to {@link #add}, which can fail to insert an
 * element only by throwing an exception.
 *
 * @param e the element to add
 * @return {@code true} if the element was added to this queue, else
 *         {@code false}
 * @throws ClassCastException if the class of the specified element
 *         prevents it from being added to this queue
 * @throws NullPointerException if the specified element is null
 * @throws IllegalArgumentException if some property of the specified
 *         element prevents it from being added to this queue
 */
boolean offer(E e);


/**
 * Retrieves and removes the head of this queue, waiting if necessary
 * until an element becomes available.
 *
 * @return the head of this queue
 * @throws InterruptedException if interrupted while waiting
 */
E take() throws InterruptedException;
```
这是BlockingQueue接口offer方法和take方法，通过注释我们可以很清除的了解到这两个方法的作用：offer将元素立即插入到队尾，在没有超出容量限制的时候，返回true，如果队列已满，则返回false;take方法取出队列中的第一个元素并从队列中移除，如果队列为空，那么就等待，直到队列中有元素为止。我们的代码片段中使用的是LinkedBlockingQueue，这里我们通过LinkedBlockingQueue来分析这两个功能是如何实现的。
```java
static class Node<E> {
    E item; //当前节点存储的对象

    Node<E> next;//下一个节点

    Node(E x) { item = x; }
}
//链表的容量
private final int capacity;
//链表当前的节点数量
private final AtomicInteger count = new AtomicInteger();
//头结点
transient Node<E> head;
//尾节点
private transient Node<E> last;
//出队锁
private final ReentrantLock takeLock = new ReentrantLock();
//标记队列不为空的ConditionObject
private final Condition notEmpty = takeLock.newCondition();
//入队锁
private final ReentrantLock putLock = new ReentrantLock();
//标记队列未满的ConditionObject
private final Condition notFull = putLock.newCondition();
```
LinkedBlockingQueue中包含了一个用于构造单向链表的Node类，容量capacity，头结点head，尾节点last,出队锁和入队锁，以及标记队列是否为空和是否已满的两个ConditionObject。了解了LinkedBlockingQueue的属性后，我们来看下他的方法。
```java
/**
 * Inserts the specified element at the tail of this queue if it is
 * possible to do so immediately without exceeding the queue's capacity,
 * returning {@code true} upon success and {@code false} if this queue
 * is full.
 * When using a capacity-restricted queue, this method is generally
 * preferable to method {@link BlockingQueue#add add}, which can fail to
 * insert an element only by throwing an exception.
 *
 * @throws NullPointerException if the specified element is null
 */
public boolean offer(E e) {
    if (e == null) throw new NullPointerException();
    final AtomicInteger count = this.count;
    if (count.get() == capacity)
        return false;
    int c = -1;
    Node<E> node = new Node<E>(e);
    final ReentrantLock putLock = this.putLock;
    //putLock加锁
    putLock.lock();
    try {
        //判断当前队列是否已满
        if (count.get() < capacity) {
            //入队
            enqueue(node);
            //这里返回的是previous值并赋值给c
            c = count.getAndIncrement();
            if (c + 1 < capacity)
                //标记队列未满
                notFull.signal();
        }
    } finally {
        //putLock解锁
        putLock.unlock();
    }
    //c==0,说明c经过了赋值,count至少为1
    if (c == 0)
        //唤醒notEmpty出队标识
        signalNotEmpty();
    //如果元素入队，那么c最少为0，否则c为-1
    return c >= 0;
}

/**
 * Links node at end of queue.
 *
 * @param node the node
 */
private void enqueue(Node<E> node) {
    // assert putLock.isHeldByCurrentThread();
    // assert last.next == null;
    last = last.next = node;
}

/**
 * Signals a waiting take. Called only from put/offer (which do not
 * otherwise ordinarily lock takeLock.)
 */
private void signalNotEmpty() {
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lock();
    try {
        notEmpty.signal();
    } finally {
        takeLock.unlock();
    }
}
```
在offer方法中，我们在对队列进行操作前使用putLock进行加锁，队列操作完毕后解锁，这样我们使得入队操作不会因多线程竞争出现异常。如果队列已满，那么元素不入队并返回false。
```java
public E take() throws InterruptedException {
    E x;
    int c = -1;
    final AtomicInteger count = this.count;
    final ReentrantLock takeLock = this.takeLock;
    //takeLock加锁
    takeLock.lockInterruptibly();
    try {
        //队列为空
        while (count.get() == 0) {
            //标识队列为空，当前线程挂起
            notEmpty.await();
        }
        //while条件返回false队列不为空，取出队头的元素
        x = dequeue();
        //CAS队列中元素数量减1
        c = count.getAndDecrement();
        //队列不为空
        if (c > 1)
            //标识队列不为空，唤醒挂起的线程
            notEmpty.signal();
    } finally {
        //takeLock解锁
        takeLock.unlock();
    }
    //c==capacity，表明count值为capacity-1
    if (c == capacity)
        //标识队列未满
        signalNotFull();
    return x;
}
```
这个方法中有几个问题需要思考

1.结合入队方法offer，假设当前队列为空，take方法中先获取了takeLock锁，而后notEmpty.await挂起了当前线程，而在offer方法中对notEmpty执行了唤起方法signalNotEmpty，注意到signalNotEmpty方法中会先执行takeLock.lock()获取takeLock锁，这个时候takeLock锁已被持有，offer方法会在takeLock.lock()方法中挂起，这不是一个典型的死锁吗？

2.为什么count要用AtomicInteger类型,take方法中对count进行减1操作的时候为什么必须使用CAS操作getAndDecrement。同理可以思考下add和offer方法中的CAS操作，getAndIncrement

3.为什么在count.getAndDecrement操作后判断队列是否为空并进行notEmpty.signal操作。

要回答这几个问题，我们需要仔细了解notEmpty.await这个方法。我们来看下notEmpty到底是个什么东东
```java
//出队锁
private final ReentrantLock takeLock = new ReentrantLock();
//标记队列不为空的ConditionObject
private final Condition notEmpty = takeLock.newCondition();

public Condition newCondition() {
    return sync.newCondition();
}

abstract static class Sync extends AbstractQueuedSynchronizer {
  final ConditionObject newCondition() {
    return new ConditionObject();
  }
  ...
}
```

代码块中前两行是LinkedBlockingQueue的代码，我们可以知道notEmpty是由调用ReentrantLock的newCondition方法得到的Condition对象。第三行是ReentrantLock中的newCondition方法，第四行是ReentrantLock中的Sync类的结构。通过这段代码我可以知道我们的notEmpty最终是一个AbstractQueuedSynchronizer.ConditionObject对象。这下再次回到j.u.c的核心类AbstractQueuedSynchronizer上。

再次ps一下，在阅读下面的文章前需要对AbstractQueuedSynchronizer的锁原理有所了解，如果你还不了解，可以参考阅读我的博客[2018-12-11-JUC源码解析<一>ReentrantLock和AbstractQueuedSynchronizer](https://sinliede.github.io/2018/12/11/JUC%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90-%E4%B8%80-ReentrantLock%E5%92%8CAbstractQueuedSynchronizer/)。

```java
public class ConditionObject implements Condition, java.io.Serializable {
  private static final long serialVersionUID = 1173984872572414699L;
  //头节点
  private transient Node firstWaiter;
  //尾节点
  private transient Node lastWaiter;
  //当前线程挂起等待
  public final void await() throws InterruptedException {
      if (Thread.interrupted())
          throw new InterruptedException();
      //向conditionQueue中加入新的节点
      Node node = addConditionWaiter();
      //完全释放被占用的锁
      int savedState = fullyRelease(node);
      int interruptMode = 0;
      //当前节点是否进入syncQueue队列
      while (!isOnSyncQueue(node)) {
          //不在syncQueue队列，挂起当前线程
          LockSupport.park(this);
          if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
              break;
      }
      if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
          interruptMode = REINTERRUPT;
      if (node.nextWaiter != null) // clean up if cancelled
          unlinkCancelledWaiters();
      if (interruptMode != 0)
          reportInterruptAfterWait(interruptMode);
  }
```

又一次看到了AbstractQueuedSynchronizer的Node类，ConditionObject类的对象通过Node.nextWaiter属性维护一个单向链表，AbstractQueuedSynchronizer在锁竞争的时候会维护一个双向链表,称之为syncQueue，ConditionObject维护的链表我们成为conditionQueue。首先来看addConditionWaiter方法。

```java
//封装当前线程的Node对象并入Condition队列
private Node addConditionWaiter() {
    Node t = lastWaiter;
    //condition队列中的节点状态必须是Node.CONDITION
    //如果不是，说明取消了等待，需要从队列中剔除
    if (t != null && t.waitStatus != Node.CONDITION) {
        unlinkCancelledWaiters();
        t = lastWaiter;
    }
    //封装当前线程的Node对象
    Node node = new Node(Thread.currentThread(), Node.CONDITION);
    //队列为空
    if (t == null)
        firstWaiter = node;
    else
        //node进入队尾
        t.nextWaiter = node;
    //更新lastWaiter节点
    lastWaiter = node;
    return node;
}

//剔除队列中已取消等待的节点
private void unlinkCancelledWaiters() {
    //由于Condition链表是从头到尾单向的，所以从头到尾遍历
    Node t = firstWaiter;
    //跟踪节点，代表当前节点前一个未取消的节点
    Node trail = null;
    while (t != null) {
        Node next = t.nextWaiter;
        //当前节点已取消
        if (t.waitStatus != Node.CONDITION) {
            //当前节点出队
            t.nextWaiter = null;
            //当前节点是头结点firstWaiter
            if (trail == null)
                //first节点赋值为第二个节点
                firstWaiter = next;
            else
                trail.nextWaiter = next;
            //当前节点的下一个节点为空，说明已经遍历到了队尾
            if (next == null)
                //尾节点赋值为当前节点
                lastWaiter = trail;
        }
        //当前节点未取消
        else
            //跟踪节点后移
            trail = t;
        //当前节点后移
        t = next;
    }
}
```
unlinkCancelledWaiters方法实现了一个空间复杂度O(1)，时间复杂度O(n)的算法，将当前Condition队列中状态不是Node.CONDITION的节点取消排队。注意到addConditionWaiter并不是一个线程安全的方法，但是在await方法执行前，我们执行了takeLock.lock获取锁，这样保证了每次只有一个线程对conditionQueue进行操作。接下来执行的是fullyRelease方法，彻底释放锁。
```java
final int fullyRelease(Node node) {
    boolean failed = true;
    try {
        //获得当前state数值
        int savedState = getState();
        //通过release方法将state设置为0，释放锁
        if (release(savedState)) {
            //锁释放成功
            failed = false;
            return savedState;
        } else {
            throw new IllegalMonitorStateException();
        }
    } finally {
        //锁释放失败
        if (failed)
            //当前节点取消Condition队列排队
            node.waitStatus = Node.CANCELLED;
    }
}

public final boolean release(int arg) {
    //state释放arg次，每次释放减1，如果释放后state值为0，返回true，否则返回false。
    if (tryRelease(arg)) {
        Node h = head;
        //head节点不为空且队列不是初始化状态
        if (h != null && h.waitStatus != 0)
            //尝试唤起head节点的next节点
            unparkSuccessor(h);
        return true;
    }
    return false;
    ```
}
release方法会将state设置为0，代表当前锁未被占用，并尝试唤起syncQueue的head节点的next节点进行抢锁操作。这里之所以要唤醒head节点的next节点，是因为当前节点释放锁，说明当前处于两种可能的情况：
1.当前节点处于head节点，因此需要唤起head节点的next节点
2.当前节点不在syncQueue队列中，因此head节点是一个空节点（初始化节点），因此需要唤醒head几点的next节点进行抢锁。
如果对其中的原理不了解，可以阅读我第一篇关于ReentrantLock和AQS的文章。
结合LinkedBlockingQueue的take方法和ConditionObject的await方法，我们可以知道，当LinkedBlockingQueue中节点数目为0时，所有消费线程会先进入syncQueue中排队获取锁，而后被唤起，进入conditionQueue中挂起等待被唤醒。

await方法暂时先看到这里，接下来看下如何唤起在conditionQueue中挂起的线程。
```java
private void signalNotEmpty() {
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lock();
    try {
        notEmpty.signal();
    } finally {
        takeLock.unlock();
    }
}
```
这段代码展示了如何唤起conditionQueue中的节点：获得takeLock锁，执行notEmpty.signal()方法，而后释放锁。
这里回答下我们之前提出的第一个问题：生产者线程在takeLock.lock()方法会首先尝试获取锁，如果获取失败，比如因为LinkedBlockingQueue.take()方法正在持有锁，那么会进入syncQueue队列中挂起，等待被唤起后获取锁。
在消费者线程执行LinkedBlockingQueue.take()方法过程中，如果没有执行notEmpty.await()方法，那么最终会执行takeLock.unlock()方法释放锁并唤起后续线程；如果执行了notEmpty.await()方法，那么在该方法中会将锁完全释放并唤起后续线程后再挂起当前线程，因此不会产生因当前线程挂起而导致锁无法被释放的问题，那么最终生产者线程的takeLock.lock()会获取到锁。
```java
public final void signal() {
    if (!isHeldExclusively())
        //持有锁的线程不是当前线程，抛出异常
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
        //唤起节点
        doSignal(first);
}

//持有锁的线程是否是当前线程
protected final boolean isHeldExclusively() {
    // While we must in general read state before owner,
    // we don't need to do so to check if current thread is owner
    return getExclusiveOwnerThread() == Thread.currentThread();
}

private void doSignal(Node first) {
    do {
        if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        first.nextWaiter = null;
    //将当前节点从conditionQueue转移到syncQueue队列
    } while (!transferForSignal(first) &&
             (first = firstWaiter) != null);
}
```
doSignal方法中会尝试将firstWaiter节点转移到syncQueue队列
```java
/**
 * Transfers a node from a condition queue onto sync queue.
 * Returns true if successful.
 * @param node the node
 * @return true if successfully transferred (else the node was
 * cancelled before signal)
 */
final boolean transferForSignal(Node node) {
    /*
     * If cannot change waitStatus, the node has been cancelled.
     */
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;

    /*
     * Splice onto queue and try to set waitStatus of predecessor to
     * indicate that thread is (probably) waiting. If cancelled or
     * attempt to set waitStatus fails, wake up to resync (in which
     * case the waitStatus can be transiently and harmlessly wrong).
     */
    Node p = enq(node);
    int ws = p.waitStatus;
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        LockSupport.unpark(node.thread);
    return true;
}
```
transferForSignal方法中，首先将节点的状态设置为0，也就是初始化状态，而后enq入队，入队后node节点成为syncQueue的尾节点tail节点；而后将tail节点的prev节点的waitStatus设置为SIGNAL状态，也就是等待获取锁状态。如果prev节点已经取消（waitStatus>0）或者设置waitStatus为SIGNAL失败，那么唤起当前线程。注意只要node节点执行了enq方法，那么就返回true，也就是成功进入syncQueue队列。

这里我们分开讨论下：
1. node节点线程被唤起，那么会进入进入await方法中的LockSupport.park(this)处，执行后续代码。
2. node节点线程未被唤起，退出doSignal方法的循环，singal方法执行完毕，生产者线程在LinkedBlockingQueue.signalNotEmpty()方法中最终会执行takeLock.unlock()方法，释放锁并唤起后续线程，因此node节点最终会被唤起。

ok，我们再次进入ConditionObject.await()方法中，看看消费者线程最终被唤起后又做了些什么。
```java
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    Node node = addConditionWaiter();
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}

/**
 * Checks for interrupt, returning THROW_IE if interrupted
 * before signalled, REINTERRUPT if after signalled, or
 * 0 if not interrupted.
 */
private int checkInterruptWhileWaiting(Node node) {
    return Thread.interrupted() ?
        (transferAfterCancelledWait(node) ? THROW_IE : REINTERRUPT) :
        0;
}

/** Mode meaning to reinterrupt on exit from wait */
private static final int REINTERRUPT =  1;
/** Mode meaning to throw InterruptedException on exit from wait */
private static final int THROW_IE    = -1;
```
