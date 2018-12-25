## 1.前言
上篇文章我们从ThreadPoolExecutor中的一小部分切入，分析了ReentrantLock和AbstractQueuedSynchronizer在进行加锁和解锁操作时的原理，这篇文章将尝试解读下ThreadPoolExecutor是如何进行线程和任务调度的。
ps:本篇博客是上一篇的延续，其中涉及AQS原理相关的内容，如果对AQS的原理不了解,可以先翻阅上一篇。个人水平有限，如有谬误的地方，还请留言指正。

## 2.问题引入
假设现在有10000个耗时任务，我们希望在较短的时间内将这批耗时任务完成，此时我们肯定会想到多线程并发。通常，为了复用线程对象，控制同一时间内线程的并发数，我们我使用线程池ThreadPoolExecutor进行多线程的调度。
```java
BlockingQueue<Runnable> blockingQueue = new LinkedBlockingQueue<>();
ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(20, 40, 60L, TimeUnit.SECONDS, blockingQueue);
for (int i = 0; i < 10000; i++) {
    threadPoolExecutor.submit(() -> {
        try {
            doTimeConsumingTask();  //耗时任务
        } catch (Exception e) {
            e.printStackTrace();
        }
    });
}
```
上面演示了ThreadPoolExecutor的简单用法，我们设置corePoolSize为20，maxPoolSize为40,最大允许40个线程同时运行。我们在主线程MainThread中将所有耗时操作通过submit方法提交到ThreadPoolExecutor并由其编排运行。
## 3.ThreadPoolExecutor源码分析
ThreadPoolExecutor继承了AbstractExecutorService了submit方法，submit方法调用了由ThreadPoolExecutor本身实现的execute方法
```java
public Future<?> submit(Runnable task) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<Void> ftask = newTaskFor(task, null);
    execute(ftask);
    return ftask;
}
```
接下来看下ThreadPoolExecutor
```java
//线程池中活跃的线程数量（未死亡），这里用了int的原子类AtomicInteger
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0)); //-2^29
private static final int COUNT_BITS = Integer.SIZE - 3;                 //29
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;            //2^29 -1

//RUNNING状态，代表线程池正常工作，注意从-2^29到-1，线程池都是处于RUNNING状态
private static final int RUNNING    = -1 << COUNT_BITS;                 //-2^29
//SHUTDOWN状态
private static final int SHUTDOWN   =  0 << COUNT_BITS;                 //0，
//STOP状态
private static final int STOP       =  1 << COUNT_BITS;                 //2^29
//TIDYING状态
private static final int TIDYING    =  2 << COUNT_BITS;                 //2*2^29
//TERMINATED状态
private static final int TERMINATED =  3 << COUNT_BITS;                 //3*2^29

// Packing and unpacking ctl
//c<0时返回-2^29, c>=0时返回c,此方法可以用于判断线程池处于何种状态
private static int runStateOf(int c)     { return c & ~CAPACITY; }
//按位与操作，注意CAPACITY后28位都是1，第29位是0，符号位是0，此操作后的结果是保留入参c的前28位并取正
//注意如果入参c是负数的话按照返回的是c的补码的前28位并取正。c从-2^29到-1的结果分别是0到2^29-1
//线程池当前线程数亮
private static int workerCountOf(int c)  { return c & CAPACITY; }
private static int ctlOf(int rs, int wc) { return rs | wc; }
//线程池是否处于工作RUNNING状态
private static boolean isRunning(int c) { return c < SHUTDOWN; }
//线程池的任务队列
private final BlockingQueue<Runnable> workQueue;
//重入锁,用于对共有资源如workers的锁操作
private final ReentrantLock mainLock = new ReentrantLock();
//正在执行任务的线程对象的集合
private final HashSet<Worker> workers = new HashSet<Worker>();
//当allowCoreThreadTimeOut为true时的超时时间
private volatile long keepAliveTime;
//对于核心线程(数目小于等于corePoolSize的线程)，是否设置取任务的等待超时时间
//默认值为0（boolean值为false),不设置核心线程超时时间
//如果为true，那么超时时间为keepAliveTime
private volatile boolean allowCoreThreadTimeOut;
//线程池维护的最小线程数量
private volatile int corePoolSize;
//线程池维护的最大线程数量
private volatile int maximumPoolSize;

public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    /*
     * Proceed in 3 steps:
     *
     * 1. If fewer than corePoolSize threads are running, try to
     * start a new thread with the given command as its first
     * task.  The call to addWorker atomically checks runState and
     * workerCount, and so prevents false alarms that would add
     * threads when it shouldn't, by returning false.
     *
     * 2. If a task can be successfully queued, then we still need
     * to double-check whether we should have added a thread
     * (because existing ones died since last checking) or that
     * the pool shut down since entry into this method. So we
     * recheck state and if necessary roll back the enqueuing if
     * stopped, or start a new thread if there are none.
     *
     * 3. If we cannot queue task, then we try to add a new
     * thread.  If it fails, we know we are shut down or saturated
     * and so reject the task.
     */
        //当前线程池中工作线程的数量
        int c = ctl.get();
        //工作线程数量小于corePoolSize，生成新的线程对象并存储到线程池
        //第一次调用时c=-2^29，workerCountOf(c)的结果应该是0
        if (workerCountOf(c) < corePoolSize) {
            //新增工作线程并启动当前任务，ctl增加1
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
         //池中工作线程数量大于或等于corePoolSize且小于CAPACITY(线程池处于RUNNING状态)，将任务存入workQueue
        if (isRunning(c) && workQueue.offer(command)) {
            //重复检查当前池中线程数量
            int recheck = ctl.get();
            //线程池不处于RUNNING状态，移除已经入队的任务并抛出任务被拒绝异常
            if (! isRunning(recheck) && remove(command))
                reject(command);
            //当前池中没有工作线程,可能是因为只有一个线程工作且这个线程死掉了或者线程池被关闭
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        //当前池中工作线程数量大于corePoolSize，新增线程
        else if (!addWorker(command, false))
            //新增线程失败，有别的线程捷足先登，使得ctl>maximumPoolSize,
            //或线程池被关闭，拒绝任务并抛出异常
            reject(command);
}
```
上述的代码给出了ThreadPoolExecutor在收到一个新的任务时的简略执行过程。我们可以通过翻译源码注释理解其中的工作流程。
1.如果正在运行的线程数量少于corePoolSize，尝试启动一个新的线程并以当前任务(command)作为此线程的第一个task。启动新线程的addWorker方法会自动检查线程池中的线程数和线程的工作状态，通过返回false的方式防止不应启动新线程时启动新线程。
2.如果一个任务能够成功的被存储到队列，那么我们依然需要重复检查是否应该启动一个新的线程，因为当我们进入这个方法的时候，线程池中已有线程可能会死亡或者线程池可能会被关闭。
3.如果我们不能成功存储当前任务到队列中，那么我们就尝试启动一个新的线程，如果启动新的线程失败，我们就知道线程池已经满了或者线程池已经关闭，因此我们拒绝这个任务被提交到线程池。
再来看下addWorker方法
```java
private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            //线程池不是RUNNING状态
            if (rs >= SHUTDOWN &&
                //线程池处于SHUTDOWN状态
                ! (rs == SHUTDOWN &&
                   //当前线程是否是主线程,如果firstTask==null,那么当前方法
                   //是由processWorkerExit方法调用的,说明当前线程是任务处理线程(子线程);
                   //如果firstTask!=null,那么当前方法是由execute()方法调用的,
                   //说明当前线程是主线程
                   firstTask == null &&
                   //任务队列不为空
                   ! workQueue.isEmpty()))
                return false;
            //循环
            for (;;) {
                int wc = workerCountOf(c);
                //判断线程池是超出CAPACITY
                if (wc >= CAPACITY ||
                //判断线程池线程数量是否超出corePoolSize 或者maximumPoolSize
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                //CAS成功增加ctl
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                //CAS失败，继续循环尝试增加ctl
                c = ctl.get();  // Re-read ctl
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            //生成新的包含线程的worker对象
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                //轻量级锁加锁
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int rs = runStateOf(ctl.get());
                    //线程池处于RUNNING状态
                    if (rs < SHUTDOWN ||
                        //线程池处于SHUTDOWN状态且当前不处于主线程
                        (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        //将worker存储到集合(池子)
                        workers.add(w);
                        int s = workers.size();
                        //记录线程池当前的大小
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    //解锁，后续线程可以获取锁
                    mainLock.unlock();
                }
                if (workerAdded) {
                    //异步启动线程
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }
```
在addWorker方法，线程池生成了一个新的线程，将此线程封装进一个Worker对象存入workers中，最后启动Worker对象中的线程。接下来看下Worker类是如何通过启动线程执行addWorker方法的入参（Runnable对象），也就是我们的task。
```java
private final class Worker extends AbstractQueuedSynchronizer implements Runnable {
    private static final long serialVersionUID = 6138294804551838833L;

    /** Thread this worker is running in.  Null if factory fails. */
    final Thread thread;
    /** Initial task to run.  Possibly null. */
    Runnable firstTask;
    /** Per-thread task counter */
    volatile long completedTasks;

    Worker(Runnable firstTask) {
        //设置state为-1，标记当前线程忙碌，防止worker在被添加到workers(池子)中后被用来执行后续任务。
        //在runWorker方法中重置为0
        setState(-1); // inhibit interrupts until runWorker
        this.firstTask = firstTask;
        //由于Worker类实现了Runnable，这里线程的Runnable对象设置为Worker本身，
        //这样当启动此线程，会执行Worker类实现的run()方法
        this.thread = getThreadFactory().newThread(this);
    }

    //实现的Runnable接口的方法
    public void run() {
        //执行任务
        runWorker(this);
    }

    //根据锁状态，判断当前线程是否有任务在执行，0代表没有，1代表有
    protected boolean isHeldExclusively() {
        return getState() != 0;
    }

    //尝试获取锁，成功则执行当前任务
    protected boolean tryAcquire(int unused) {
        //compareAndSetState的expect参数为0，只有当前state值为0时才会尝试CAS
        if (compareAndSetState(0, 1)) {
            setExclusiveOwnerThread(Thread.currentThread());
            return true;
        }
        return false;
    }

    //释放锁，标记当前线程空闲
    protected boolean tryRelease(int unused) {
        setExclusiveOwnerThread(null);
        setState(0);
        return true;
    }

    public void lock()        { acquire(1); }
    public boolean tryLock()  { return tryAcquire(1); }
    public void unlock()      { release(1); }
    public boolean isLocked() { return isHeldExclusively(); }

    void interruptIfStarted() {
        Thread t;
        if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
            try {
                t.interrupt();
            } catch (SecurityException ignore) {

            }
        }
    }
}

final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    //由于Worker在初始化的时候会被标记为工作状态，所以这里先标记为空闲状态
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {
        //Worker初始化任务是否为空，为空则从workQueue中取任务，循环执行任务
        //当workQueue为空时，getTask方法在allowCoreThreadTimeOut为false的时候
        //会一直等待直到workQueue不为空，在allowCoreThreadTimeOut为true的时候
        //等待的超时时间为keepAliveTime
        while (task != null || (task = getTask()) != null) {
            //标记当前线程为工作状态
            w.lock();
            // If pool is stopping, ensure thread is interrupted;
            // if not, ensure thread is not interrupted.  This
            // requires a recheck in second case to deal with
            // shutdownNow race while clearing interrupt
            // 线程池状态是STOP,TIDYING或者TERMINATED
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 //当前线程已经标记为中断状态
                 (Thread.interrupted() &&
                    //再次检测线程池是否STOP
                    runStateAtLeast(ctl.get(), STOP))) &&
                        //当前线程未中断
                        !wt.isInterrupted())
                //中断当前线程
                wt.interrupt();
            try {
                //默认什么都不做，ThreadPoolExecutor的继承类可以实现此方法
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    //真正执行我们提交的任务
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    //默认什么都不做，ThreadPoolExecutor的继承类可以实现此方法
                    afterExecute(task, thrown);
                }
            } finally {
                task = null;
                w.completedTasks++;
                w.unlock();
            }
        }
        //workQueue已经为空
        completedAbruptly = false;
    } finally {
        //任务执行完毕，清除当前任务
        processWorkerExit(w, completedAbruptly);
    }
}
```
1.Worker类实现了Runnable接口，并且在构造方法中将其Thread属性thread的Runnable属性设置为本身，这样当我们调用worker.thread.start()的时候会执行woker.run()方法,worker.run()方法会执行ThreadPoolExecutor.runWorker方法。
2.当worker.firstTask为null的时候，会通过getTask方法从workQueue中取任务来执行，当所有workQueue中的所有任务执行完毕后，如果我们设置allowCoreThreadTimeOut=true和keepAliveTime,那么getTask方法会在等待keepAliveTime后返回null,并执行ctl--，这样我们退出循环并执行processWorkerExit方法。
看一下getTask方法
```java
private Runnable getTask() {
    boolean timedOut = false; // Did the last poll() time out?

    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        //线程池处于STOP状态或者线程池处于SHUTDOWN状态且workQueue为空
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }

        int wc = workerCountOf(c);

        // Are workers subject to culling?
        //是否设置取任务时限
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
        //线程池线程数量超过maximumPoolSize(已满)或者取任务超时
        if ((wc > maximumPoolSize || (timed && timedOut))
            //线程池线程数不为1或者workQueue为空
            && (wc > 1 || workQueue.isEmpty())) {
            //工作线程数量ctl减1
            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;
        }

        try {
            Runnable r = timed ?
                //取任务时限为keepAliveTime
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                //不设置取任务时限，取不到就一直等待，直到取到为止
                workQueue.take();
            //超时前取到了任务
            if (r != null)
                return r;
            //超时，进入下次循环并处理超时
            timedOut = true;
        } catch (InterruptedException retry) {
            timedOut = false;
        }
    }
}
```
1.当allowCoreThreadTimeOut为true时,总是设置超时时间，当池中线程数量大于corePoolSize时，也会设置超时时间
2.如果取任务超时，那么ctl减1，意味着当前线程被线程池放弃。
3.从这个方法我们知道，当任务数量不足时，线程池总是趋向于保持corePoolSize个线程。
4.workQueue是一个BlockingQueue,在执行take()方法的时候，如果queue为空，当前线程会挂起在此处，直到queue不为空且取到元素为止，关于BlockingQueue后续文章会进行分析。
5.workQueue在执行poll方法时有超时时限设置，如果超时则返回null,此时queue一定为空。
```java
private void processWorkerExit(Worker w, boolean completedAbruptly) {
    //当前线程是否被中断
    if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
        //工作线程数量-1
        decrementWorkerCount();

    final ReentrantLock mainLock = this.mainLock;
    //completedTaskCount是多线程共有资源，加锁
    mainLock.lock();
    try {
        //将worker对象完成的任务数计入完成的总任务数量
        completedTaskCount += w.completedTasks;
        //worker移除出正在执行任务的线程集合
        workers.remove(w);
    } finally {
        //解锁
        mainLock.unlock();
    }
    //尝试在线程池关闭或者任务列表已经为空时终止线程池
    tryTerminate();

    int c = ctl.get();
    //线程池当前不处于STOP状态（RUNNING或者SHUTDOWN状态）
    if (runStateLessThan(c, STOP)) {
        //线程未被中断
        if (!completedAbruptly) {
            //线程池是否设置了线程挂起（空闲）超时时限
            int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
            //设置了时限，默认保留一个空闲线程，否则默认保留corePoolSize个线程
            if (min == 0 && ! workQueue.isEmpty())
                min = 1;
            //如果当前池中线程数量大于需要保留的线程数量
            if (workerCountOf(c) >= min)
                //结束当前线程
                return; // replacement not needed
        }
        //当前线程存活，调用addWorker方法从workQueue中取任务执行
        addWorker(null, false);
    }
}
```
1.如果completedAbruptly值为true,说明runWorker方法中的while循环没有被正确的执行，getTask方法没有将ctl减1，这里需要将工作线程数量ctl减1。
2.在allowCoreThreadTimeOut为true时，如果workQueue为空，那么一个核心线程不保留，如果workQueue不为空，那么保留一个核心线程。

## 4.总结
这篇文章从源码的角度分析了线程池的工作原理，我们引用这篇博文（[一文带你进入Java之ThreadPool](http://hulong.me/2017/01/19/Java-ThreadPool.html)）对于线程池的通俗描述已方便大家对于线程池的理解:

把线程池理解成一个医院，在医院成立之初，医生数量为 0，当有患者时，没有医生来诊疗患者，医院会去招聘新的医生，一旦这些医生忙不过来时，继续招聘，直到达到corePoolSize数量，停止招聘。此时的corePoolSize个医生为正式员工，即使没有患者，也不会辞退他们（销毁线程）。

医生达到corePoolSize后，当有新患者来就诊，医生忙不过来时，直接让他们在候诊区（workQueue）取号等候，当医生看完上一个病人时，会去候诊区叫下一个号进去，如果没有患者，则可以休息。

当患者数量急剧上升，候诊区座位数不够了，这时，医院会再去招聘临时工医生，这些临时工医生会让没有座位的患者立即就诊，医院按需求逐个招聘，直到达到maximumPoolSize数量，停止招聘。

当临时招聘的医生长时间（keepAliveTime）处于空闲状态时，医院就会解雇他们，毕竟要额外付工资啊～

如果你实在觉得医院效益不好，那就可以通过allowsCoreThreadTimeOut方法设置允许解雇空闲的正式员工，只留一个就好了～
