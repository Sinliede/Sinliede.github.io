# J.U.C源码解析BolckingQueue之ArrayBlockingQueue

## 前言
上篇文章中分析了BlockingQueue的其中一个实现类LinkedBlockingQueue，本篇文章继续分析BlockingQueue的实现类ArrayBlockingQueue。
## 源码分析
首先，看下ArrayBlockingQueue的类结构和类属性
```java
public class ArrayBlockingQueue<E> extends AbstractQueue<E>
        implements BlockingQueue<E>, java.io.Serializable {

          /** The queued items */
       final Object[] items;

       /** items index for next take, poll, peek or remove */
       int takeIndex;

       /** items index for next put, offer, or add */
       int putIndex;

       /** Number of elements in the queue */
       int count;

       /** Main lock guarding all access */
       final ReentrantLock lock;

       //出队条件对象
       /** Condition for waiting takes */
       private final Condition notEmpty;

       //入队条件对象
       /** Condition for waiting puts */
       private final Condition notFull;

       /**
        * Shared state for currently active iterators, or null if there
        * are known not to be any.  Allows queue operations to update
        * iterator state.
        */
       transient Itrs itrs = null;

       /**
        * Creates an {@code ArrayBlockingQueue} with the given (fixed)
        * capacity and default access policy.
        *
        * @param capacity the capacity of this queue
        * @throws IllegalArgumentException if {@code capacity < 1}
        */
       public ArrayBlockingQueue(int capacity) {
           this(capacity, false);
       }

       //构造方法
       public ArrayBlockingQueue(int capacity, boolean fair) {
           if (capacity <= 0)
               throw new IllegalArgumentException();
           this.items = new Object[capacity];
           lock = new ReentrantLock(fair);
           notEmpty = lock.newCondition();
           notFull =  lock.newCondition();
       }
}
```
上方代码块中贴出了ArrayBlockingQueue类属性的源码。和ArrayList类似,ArrayBlockingQueue使用了数组进行对象的存储，区别在于，ArrayBlockingQueue的容量是fixed（固定）的，不能进行扩容等操作。注意到notEmpty和notFull两个条件对象是由同一个锁new出来的，这也关系到入队线程和出队线程的唤醒等关键操作。接下来看核心方法的实现。
```java
public boolean offer(E e) {
    //检查入队对象是否为null，如为null则抛出异常
    checkNotNull(e);
    final ReentrantLock lock = this.lock;
    //获取锁
    lock.lock();
    try {
        //判断队列是否已满
        if (count == items.length)
            return false;
        else {
            //执行入队
            enqueue(e);
            return true;
        }
    } finally {
        //释放锁
        lock.unlock();
    }
}

private static void checkNotNull(Object v) {
    if (v == null)
        throw new NullPointerException();
}

/**
 * Inserts element at current put position, advances, and signals.
 * Call only when holding lock.
 */
private void enqueue(E x) {
    // assert lock.getHoldCount() == 1;
    // assert items[putIndex] == null;
    final Object[] items = this.items;
    items[putIndex] = x;
    //入队标识是否到达队尾，更新后续入队位置putIndex
    if (++putIndex == items.length)
        //到达队尾则putIndex重置为0，重头开始入队
        putIndex = 0;
    //更新队列中的元素个数
    count++;
    //标记当前队列不为空
    notEmpty.signal();
}
```
首先来看入队操作， 入队操作是从头到尾依次入队，可以理解为列表循环。入队前判断对象是否为null，为null则抛出异常，而后获取锁，判断队列是否已满，如果队列已满则直接返回入队失败，未满则执行入队，并标记当前队列不为空，最后释放锁。这里有两个地方需要留意<br>
1. lock.unlock()方法在finally代码块中，锁一定会被释放。<br>
2. notEmpty.signal()方法标记当前队列不为空，并使notEmpty的condition队列的head节点进入Sync队列（等待获取锁），这个设计使得每执行一次入队操作offer()方法，允许一次出队列操作。
3.
接下来看出队操作

```java
public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    //获取锁
    lock.lockInterruptibly();
    try {
        //队列为空，当前线程进入condition队列等待
        while (count == 0)
            notEmpty.await();
        //元素出队
        return dequeue();
    } finally {
        //解锁
        lock.unlock();
    }
}

/**
 * Extracts element at current take position, advances, and signals.
 * Call only when holding lock.
 */
private E dequeue() {
    // assert lock.getHoldCount() == 1;
    // assert items[takeIndex] != null;
    final Object[] items = this.items;
    //获取takeIndex位置所在对象
    @SuppressWarnings("unchecked")
    E x = (E) items[takeIndex];
    //将takeIndex位置设置为null
    items[takeIndex] = null;
    //takeIndex到达队尾，那么就重头开始
    if (++takeIndex == items.length)
        takeIndex = 0;
    count--;
    if (itrs != null)
        itrs.elementDequeued();
    //标记当前队列未满
    notFull.signal();
    //返回取得的对象
    return x;
}
```
类似于入队操作，出队操作也是从第一个位置开始，依次出队，到达队尾后再次重头开始。这里需要留意的地方有<br>
1. 当前队列为空的时候，执行了notEmpty.await()，当前出队线程进入condition队列等待，此方法会释放当前线程持有的锁。
2. 在ArrayBlockingQueue.put()方法中，如果当前队列已满，那么入队线程会进入notFull的条件队列中等待，notFull.signal()类似于notEmpty.signal()方法，每执行一次出队操作允许执行一次入队操作。
3. notEmpty.await()和notFull.await()和lock.lock()操作的是同一把锁。

看一下另外两个入队和出队的方法
```java
public void put(E e) throws InterruptedException {
    checkNotNull(e);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        while (count == items.length)
            notFull.await();
        enqueue(e);
    } finally {
        lock.unlock();
    }
}

public E poll() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return (count == 0) ? null : dequeue();
    } finally {
        lock.unlock();
    }
}

public E poll(long timeout, TimeUnit unit) throws InterruptedException {
    long nanos = unit.toNanos(timeout);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        while (count == 0) {
            if (nanos <= 0)
                return null;
            nanos = notEmpty.awaitNanos(nanos);
        }
        return dequeue();
    } finally {
        lock.unlock();
    }
}
```
put方法提供了一个阻塞的元素入队方法，当前队列已满，那么入队线程通过notFull.await()方法阻塞进入notFull的Condition队列。poll方法提供了两个方法，一个是非阻塞的获取元素的方法，当前队列中如果为空，那么直接返回null；一个是阻塞指定时间的获取元素方法。

最后，根据put()和take()方法简单总结下ArrayBlockingQueue的工作流程：

ArrayBlockingQueue提供了一个固定大小的数组存放元素，入队线程从头到尾依次入队，循环往复，每次入队队列中元素数目加1，同时唤起一个在notEmpty的Condition队列中挂起的出队线程进行抢锁，队列已满的时候入队线程挂起并进入notFull的Condition队列中等待被唤起；出队线程也从头到尾依次出队，循环往复，每次出队队列中元素数目减1，同时唤起一个在notFull的Condition队列中挂起的入队线程进行抢锁，队列已满的时候进入notEmpty的Condition队列中挂起等待被唤起。
