---
title: ForkJoinPool详解
date: 2020-08-18 20:57:25
tags:
- juc
- java并发
- 线程池
---
> 文章参考博客[分析jdk-1.8-ForkJoinPool实现原理(上)](https://www.jianshu.com/p/de025df55363)、[分析jdk-1.8-ForkJoinPool实现原理(下)](https://www.jianshu.com/p/44b09f52a225)


# ForkJoinPool详解

> 介绍一下forkJoinPool?
> 这个线程池是ExecutorsService的另一个分支，用在多核CPU，而且任务可以拆分为子任务去解决的那种情况。里面会默认创建cpu核数的工作线程，每个工作线程具有自己的阻塞队列，从对尾取自己得任务，如果自己阻塞队列里面没有任务了去其他工作线程的队头去偷任务。

分开又合并，分而治之。

核心算法是work-stealing算法，翻译过来是工作窃取算法。

有三个重要角色:
- ForkJoinWorkerThread ，包装Thread，后面简称worker线程
- workQueue ， 双向工作队列
- ForkJoinTask ，worker线程执行对象，实现了Future。两种类型，一种叫submission，另一种就叫task。


ForkJoinPool使用数组保存了所有WorkQueue（workQueue[]），每个worker都有自己的workQueue，不是所有workQueue都有自己的worker。
- 没有worker的WorkQueue：保存的是submission，来自外部提交，在WorkQueue[]的下标是偶数；
- 属于worker的WorkQueue：保存的是task，在WorkQueue[]的下标是奇数。

WorkerQueue是一个双端队列，同时支持作为栈的push、pop操作，也支持作为队列的offer、poll操作，worker线程处理自己的workerQueue与之前的线程池取任务不一样，这里是当作栈来使用的，先进后出的方式处理workerQueue里的任务，除此之外worker线程偷其他worker的workerQueue是当作队列来处理的，也就是当队列里有多个任务的时候，worker去取任务不会与其他worker争用任务。

## 构造方法
```java
private ForkJoinPool(int parallelism,
                     ForkJoinWorkerThreadFactory factory,
                     UncaughtExceptionHandler handler,
                     int mode,
                     String workerNamePrefix) {
    this.workerNamePrefix = workerNamePrefix;
    this.factory = factory;
    this.ueh = handler;
    this.config = (parallelism & SMASK) | mode;
    long np = (long)(-parallelism); // offset ctl counts
    this.ctl = ((np << AC_SHIFT) & AC_MASK) | ((np << TC_SHIFT) & TC_MASK);
}
```
**parallelism**：默认是cpu的核心数，ForkJoinPool的线程数量依据与它，但不表示ThreadPoolExecutor的核心线程数和最大线程数。  

**factory**：ThreadFactory，实现newThread方法即可。

**config**：config保存不变的参数，包括parallelism和mode。

**ctl**：是ForkJoinPool中最重要的控制手段，将下面信息按16bit为一组保存在ctl这个long整型里。
- AC: 活动的worker数量
- TC: 总共的worker数量
- SS: WorkerQueue状态，第1位表示active还是inactive，其余十五位表示版本号，ABA
- ID：保存一个WorkQueue在WorkQueue[]的下标，和其他worker通过字段stackPred组成一个TreiberStack。后文讲的栈顶，指这里下标所在的WorkQueue。

TreiberStack：这个栈的pull和pop使用了CAS，所以支持并发下的无锁操作。

## ForkJoinPool状态修改
在介绍里面的方法之前先看看线程池的状态
- STARTED
- STOP
- TERMINATED
- SHUTDOWN  
以上四种，线程池的标准状态
- RSLOCK
- RSIGNAL  
runState是线程池的状态，在修改状态之前要获得锁
```java
    // runState bits: SHUTDOWN must be negative, others arbitrary powers of two
    private static final int  RSLOCK     = 1;
    private static final int  RSIGNAL    = 1 << 1;
    private static final int  STARTED    = 1 << 2;
    private static final int  STOP       = 1 << 29;
    private static final int  TERMINATED = 1 << 30;
    private static final int  SHUTDOWN   = 1 << 31;
```


## 任务ForkJoinTask
ForkJoinPool执行任务的对象是ForkJoinTask，它是一个抽象类，有两个具体实现类RecursiveAction和RecursiveTask。

```java
public abstract class RecursiveAction extends ForkJoinTask<Void> {
    protected abstract void compute();

    public final Void getRawResult() { return null; }

    protected final void setRawResult(Void mustBeNull) { }

    protected final boolean exec() {
        compute();
        return true;
    }
}
```

```java
public abstract class RecursiveTask<V> extends ForkJoinTask<V> {
    V result;

    protected abstract V compute();

    public final V getRawResult() {
        return result;
    }

    protected final void setRawResult(V value) {
        result = value;
    }

    protected final boolean exec() {
        result = compute();
        return true;
    }
}
```

ForkJoinTask的抽象方法exec由RecursiveAction和RecursiveTask实现，它被定义为final，具体的执行步骤compute延迟到子类实现。很容易看出RecursiveAction和RecursiveTask的区别，前者没有result，getRawResult返回空，它们对应不需要返回结果和需要返回结果两种场景。

ForkJoinTask里很重要的字段是它的状态status，默认是0，当得出结果时变更为负数，有三种结果：

NORMAL
CANCELLED
EXCEPTIONAL
除此之外，在得出结果之前，任务状态能够被设置为SIGNAL，表示有线程等待这个任务的结果，执行完成后需要notify通知，具体看后文的join。

ForkJoinTask在触发执行后，并不支持其他什么特别操作，只能等待任务执行完成。CountedCompleter是ForkJoinTask的子类，它在子任务协作方面扩展了更多操作。

## ForkJoinTask在pool里的提交
有三种，分别是submit、execute、invoke
```java
//submit
    public <T> ForkJoinTask<T> submit(ForkJoinTask<T> task) {
        if (task == null)
            throw new NullPointerException();
        externalPush(task);
        return task;
    }

    public <T> ForkJoinTask<T> submit(Callable<T> task) {
        ForkJoinTask<T> job = new ForkJoinTask.AdaptedCallable<T>(task);
        externalPush(job);
        return job;
    }

    /**
     * @throws NullPointerException if the task is null
     * @throws RejectedExecutionException if the task cannot be
     *         scheduled for execution
     */
    public <T> ForkJoinTask<T> submit(Runnable task, T result) {
        ForkJoinTask<T> job = new ForkJoinTask.AdaptedRunnable<T>(task, result);
        externalPush(job);
        return job;
    }

    /**
     * @throws NullPointerException if the task is null
     * @throws RejectedExecutionException if the task cannot be
     *         scheduled for execution
     */
    public ForkJoinTask<?> submit(Runnable task) {
        if (task == null)
            throw new NullPointerException();
        ForkJoinTask<?> job;
        if (task instanceof ForkJoinTask<?>) // avoid re-wrap
            job = (ForkJoinTask<?>) task;
        else
            job = new ForkJoinTask.AdaptedRunnableAction(task);
        externalPush(job);
        return job;
    }
```
submit方法支持Runnable和Callable，通过适配器适配成ForkJoinTask。返回的值当前的FoekJoinTask

```java
//execute
   /**
     * Arranges for (asynchronous) execution of the given task.
     *
     * @param task the task
     * @throws NullPointerException if the task is null
     * @throws RejectedExecutionException if the task cannot be
     *         scheduled for execution
     */
    public void execute(ForkJoinTask<?> task) {
        if (task == null)
            throw new NullPointerException();
        externalPush(task);
    }

    // AbstractExecutorService methods

    /**
     * @throws NullPointerException if the task is null
     * @throws RejectedExecutionException if the task cannot be
     *         scheduled for execution
     */
    public void execute(Runnable task) {
        if (task == null)
            throw new NullPointerException();
        ForkJoinTask<?> job;
        if (task instanceof ForkJoinTask<?>) // avoid re-wrap
            job = (ForkJoinTask<?>) task;
        else
            job = new ForkJoinTask.RunnableExecuteAction(task);
        externalPush(job);
    }
```
execute是没有返回值的提交。

```java
//invoke
    public <T> T invoke(ForkJoinTask<T> task) {
        if (task == null)
            throw new NullPointerException();
        externalPush(task);
        return task.join();
    }
```
invoke是返回join结果的提交。

可以看到所有提交的任务都进externalPush方法。
```java
    final void externalPush(ForkJoinTask<?> task) {
        WorkQueue[] ws; WorkQueue q; int m;
        int r = ThreadLocalRandom.getProbe();
        int rs = runState;
        //工作队列数组不是空，从数组中选定一个queue也不是空，
        if ((ws = workQueues) != null && (m = (ws.length - 1)) >= 0 &&
            (q = ws[m & r & SQMASK]) != null && r != 0 && rs > 0 &&
            U.compareAndSwapInt(q, QLOCK, 0, 1)) {
            ForkJoinTask<?>[] a; int am, n, s;
            if ((a = q.array) != null &&
                (am = a.length - 1) > (n = (s = q.top) - q.base)) {
                int j = ((am & s) << ASHIFT) + ABASE;
                //在队尾top处添加task
                U.putOrderedObject(a, j, task);
                U.putOrderedInt(q, QTOP, s + 1);
                U.putIntVolatile(q, QLOCK, 0);
                //那么现在将task加入到队列中，之后要启用worker线程来执行它，signalWork
                if (n <= 1)
                    signalWork(ws, q);
                return;
            }
            U.compareAndSwapInt(q, QLOCK, 1, 0);
        }
        externalSubmit(task);
    }
private void externalSubmit(ForkJoinTask<?> task) {
        int r;                                    // initialize caller's probe
        if ((r = ThreadLocalRandom.getProbe()) == 0) {
            ThreadLocalRandom.localInit();
            r = ThreadLocalRandom.getProbe();
        }
        for (;;) {
            WorkQueue[] ws; WorkQueue q; int rs, m, k;
            boolean move = false;
            //1
            if ((rs = runState) < 0) {
                tryTerminate(false, false);     // help terminate
                throw new RejectedExecutionException();
            }
            //2
            else if ((rs & STARTED) == 0 ||     // initialize
                     ((ws = workQueues) == null || (m = ws.length - 1) < 0)) {
                int ns = 0;
                rs = lockRunState();
                try {
                    if ((rs & STARTED) == 0) {
                        U.compareAndSwapObject(this, STEALCOUNTER, null,
                                               new AtomicLong());
                        // create workQueues array with size a power of two
                        int p = config & SMASK; // ensure at least 2 slots
                        int n = (p > 1) ? p - 1 : 1;
                        n |= n >>> 1; n |= n >>> 2;  n |= n >>> 4;
                        n |= n >>> 8; n |= n >>> 16; n = (n + 1) << 1;
                        workQueues = new WorkQueue[n];
                        ns = STARTED;
                    }
                } finally {
                    unlockRunState(rs, (rs & ~RSLOCK) | ns);
                }
            }
            //3
            else if ((q = ws[k = r & m & SQMASK]) != null) {
                if (q.qlock == 0 && U.compareAndSwapInt(q, QLOCK, 0, 1)) {
                    ForkJoinTask<?>[] a = q.array;
                    int s = q.top;
                    boolean submitted = false; // initial submission or resizing
                    try {                      // locked version of push
                        if ((a != null && a.length > s + 1 - q.base) ||
                            (a = q.growArray()) != null) {
                            int j = (((a.length - 1) & s) << ASHIFT) + ABASE;
                            U.putOrderedObject(a, j, task);
                            U.putOrderedInt(q, QTOP, s + 1);
                            submitted = true;
                        }
                    } finally {
                        U.compareAndSwapInt(q, QLOCK, 1, 0);
                    }
                    if (submitted) {
                        signalWork(ws, q);
                        return;
                    }
                }
                move = true;                   // move on failure
            }
            //4
            else if (((rs = runState) & RSLOCK) == 0) { // create new queue
                q = new WorkQueue(this, null);
                q.hint = r;
                q.config = k | SHARED_QUEUE;
                q.scanState = INACTIVE;
                rs = lockRunState();           // publish index
                if (rs > 0 &&  (ws = workQueues) != null &&
                    k < ws.length && ws[k] == null)
                    ws[k] = q;                 // else terminated
                unlockRunState(rs, rs & ~RSLOCK);
            }
            else
                move = true;                   // move if busy
            if (move)
                r = ThreadLocalRandom.advanceProbe(r);
        }
    }
```
以上就是线程池提交任务的流程，对于externalSubmit方法是externalPush方法的完整版。
只介绍一下externalSubmit:  
mark1检查运行状态是否已经进入SHUTDOWN，抛出拒收的异常。对于ForkJoinPool的关闭，见后文“关闭ForkJoinPool”一节。

第一次执行externalSubmit时，运行状态还没有STARTED，执行mark2进行初始化操作：

按2的幂设置WorkQueue[]的长度
设置原子对象stealCounter
运行状态进入STARTED
第二次循环中，执行mark4，创建第一个WorkQueue。这个工作队列是没有worker线程对应的，因此它位于workQueue[]的偶数位。

> ForkJoinPool中的这些WorkQueue和工作线程ForkJoinWorkerThread并不是一对一的关系，而是随时都有多于ForkJoinWorkerThread数量的WorkQueue元素。而这个ForkJoinPool中的WorkQueue数组中，索引位为非奇数的工作队列用于存储从外部提交到ForkJoinPool中的任务，也就是所谓的submissions queue；索引位为奇数的工作队列用于存储归并计算过程中等待处理的子任务，也就是task queue。
> 
> 第三次循环中，执行mark3，会找到刚才创建的WorkQueue，从队列的top端加入任务，调用后面要讲的signalWork激活或者创建worker。

WorkQueue在WorkQueue[]的下标，取的是k = r & m & SQMASK。r是线程的probe，来自随机数ThreadLocalRandom；m是WorkQueue[]的长度减一；SQMASK是固定值0x007e，转为二进制是1111110，末尾是0，在&操作后，得出的k必定是偶数。所以创建的第一个WorkQueue没有对应worker，保存的任务是submission，scanState默认是INACTIVE。

externalSubmit是长了点，不过逻辑清晰，不难理解。除了初始化，大部分时间其实不需要externalSubmit，使用简单版的externalPush即可。

以上如果队列里只有一个任务，就调用signalWork

## worker管理

一个worker线程的生命周期里面有注册、创建、执行、注销

## worker的创建
首先是worker的创建在signalWork()方法里面，signalWork方法就是当任务到队列中等候以后，判断没有空闲线程就创建新的
```java
final void signalWork(WorkQueue[] ws, WorkQueue q) {
    long c; int sp, i; WorkQueue v; Thread p;
    while ((c = ctl) < 0L) {                       // too few active
        //1
        if ((sp = (int)c) == 0) {                  // no idle workers
            if ((c & ADD_WORKER) != 0L)            // too few workers
                tryAddWorker(c);
            break;
        }
        //2
        if (ws == null)                            // unstarted/terminated
            break;
        if (ws.length <= (i = sp & SMASK))         // terminated
            break;
        if ((v = ws[i]) == null)                   // terminating
            break;
        int vs = (sp + SS_SEQ) & ~INACTIVE;        // next scanState
        int d = sp - v.scanState;                  // screen CAS
        long nc = (UC_MASK & (c + AC_UNIT)) | (SP_MASK & v.stackPred);
        if (d == 0 && U.compareAndSwapLong(this, CTL, c, nc)) {
            v.scanState = vs;                      // activate v
            if ((p = v.parker) != null)
                U.unpark(p);
            break;
        }
        if (q != null && q.base == q.top)          // no more work
            break;
    }
}
```
首先是进入循环的条件，判断了ctl的正负，我们知道ctl的第一个16bit表示AC，为负时表示活动的worker还未达到预定的Parallelism，需要新增或者激活。mark1通过sp判断现在没有空闲worker，需要执行增加，调用tryAddWorker。

有空闲worker的情况进入mark2，sp取栈顶WorkQueue的下标，具体解挂worker的过程和tryRelease几乎一样，这里合起来介绍。  
解卦worker的意思是将worker线程解除挂起。  

在sp上，将状态从inactive改为active，累加版本号 ，解挂线程，通过stackPred取得前一个WorkQueue的index，设回sp里。

```java
private void tryAddWorker(long c) {
        boolean add = false;
        do {
            long nc = ((AC_MASK & (c + AC_UNIT)) |
                       (TC_MASK & (c + TC_UNIT)));
            if (ctl == c) {
                int rs, stop;                 // check if terminating
                if ((stop = (rs = lockRunState()) & STOP) == 0)
                    add = U.compareAndSwapLong(this, CTL, c, nc);
                unlockRunState(rs, rs & ~RSLOCK);
                if (stop != 0)
                    break;
                if (add) {
                    createWorker();
                    break;
                }
            }
        } while (((c = ctl) & ADD_WORKER) != 0L && (int)c == 0);
    }
private boolean createWorker() {
    ForkJoinWorkerThreadFactory fac = factory;
    Throwable ex = null;
    ForkJoinWorkerThread wt = null;
    try {
        if (fac != null && (wt = fac.newThread(this)) != null) {
            wt.start();
            return true;
        }
    } catch (Throwable rex) {
        ex = rex;
    }
    deregisterWorker(wt, ex);
    return false;
}
```
增加worker，需要将AC和TC都加1，成功后调用createWorker。

createWorker的代码很简单，通过线程工厂创建worker的实例并启动。如果没有异常，直接返回就行；否则，需要逆操作撤销worker的注册。worker什么时候注册了？看ForkJoinWorkerThread的构造函数，里面调用ForkJoinPool.registerWorker，所以就是在创建worker的时候会注册工作线程。
```java
final WorkQueue registerWorker(ForkJoinWorkerThread wt) {
    UncaughtExceptionHandler handler;
    wt.setDaemon(true);                           // configure thread
    if ((handler = ueh) != null)
        wt.setUncaughtExceptionHandler(handler);
    WorkQueue w = new WorkQueue(this, wt);
    int i = 0;                                    // assign a pool index
    int mode = config & MODE_MASK;
    int rs = lockRunState();
    try {
        WorkQueue[] ws; int n;                    // skip if no array
        if ((ws = workQueues) != null && (n = ws.length) > 0) {
           //1
            int s = indexSeed += SEED_INCREMENT;  // unlikely to collide
            int m = n - 1;
            i = ((s << 1) | 1) & m;               // odd-numbered indices
            if (ws[i] != null) {                  // collision
                int probes = 0;                   // step by approx half n
                int step = (n <= 4) ? 2 : ((n >>> 1) & EVENMASK) + 2;
                while (ws[i = (i + step) & m] != null) {
                    if (++probes >= n) {
                        workQueues = ws = Arrays.copyOf(ws, n <<= 1);
                        m = n - 1;
                        probes = 0;
                    }
                }
            }
            //2
            w.hint = s;                           // use as random seed
            w.config = i | mode;
            w.scanState = i;                      // publication fence
            ws[i] = w;
        }
    } finally {
        unlockRunState(rs, rs & ~RSLOCK);
    }
    wt.setName(workerNamePrefix.concat(Integer.toString(i >>> 1)));
    return w;
}
```
一开始，线程就被设置为守护线程。重温知识点，当只剩下守护线程时，JVM就会退出，垃圾回收线程也是一个典型的守护线程。

mark1，前文讲过有对应worker的WorkQueue只能出现在WorkQueue[]奇数index，代码里取初始index用的是：

i = ((s << 1) | 1) & m; 
seed左移再“或”1，是奇数。m是WorkQueue\[]长度减1，也是奇数。两者再“与”，保证取得的i是奇数。若该位置已经存在其他WorkQueue，需要重新计算下一个位置，有需要还要扩容WorkQueue[]。

mark2设置新创建WorkQueue的scanState为index，表示了两种意思：

- 非负表示有对应的worker；
- 默认scanState使用SCANNING。
  
就此描述清楚worker的创建、WorkQueue的创建和加入WorkQueue[]。

## workter执行
```java
public void run() {
    if (workQueue.array == null) { // only run once
        Throwable exception = null;
        try {
            onStart();
            pool.runWorker(workQueue);
        } catch (Throwable ex) {
            exception = ex;
        } finally {
            try {
                onTermination(exception);
            } catch (Throwable ex) {
                if (exception == null)
                    exception = ex;
            } finally {
                pool.deregisterWorker(this, exception);
            }
        }
    }
}
final void runWorker(WorkQueue w) {
    w.growArray();                   // allocate queue
    int seed = w.hint;               // initially holds randomization hint
    int r = (seed == 0) ? 1 : seed;  // avoid 0 for xorShift
    for (ForkJoinTask<?> t;;) {
        if ((t = scan(w, r)) != null)
            w.runTask(t);
        else if (!awaitWork(w, r))//这里如果为false，进入到break出了循环，这个线程就结束了
            break;
        r ^= r << 13; r ^= r >>> 17; r ^= r << 5; // xorshift
    }
}
```
这个run方法是ForkJoinWorkerThread里的run，重写了Thread的run方法，当启动这个线程执行的就是这个run。

调用了ForkJoinPool的runWorker方法。

worker执行流程就是三部曲：

- scan：尝试获取一个任务；
- runTask：执行取得的任务；
- awaitWork：没有任务进入等待。

如果awaitWork返回false，等不到任务，跳出runWorker的循环，回到run中执行finally，最后调用deregisterWorker撤销注册。

首先是scan
```java
private ForkJoinTask<?> scan(WorkQueue w, int r) {
    WorkQueue[] ws; int m;
    if ((ws = workQueues) != null && (m = ws.length - 1) > 0 && w != null) {
        int ss = w.scanState;                     // initially non-negative
        //1
        for (int origin = r & m, k = origin, oldSum = 0, checkSum = 0;;) {
            WorkQueue q; ForkJoinTask<?>[] a; ForkJoinTask<?> t;
            int b, n; long c;
            if ((q = ws[k]) != null) {
                if ((n = (b = q.base) - q.top) < 0 &&
                    (a = q.array) != null) {      // non-empty
                    long i = (((a.length - 1) & b) << ASHIFT) + ABASE;
                    if ((t = ((ForkJoinTask<?>)
                              U.getObjectVolatile(a, i))) != null &&
                        q.base == b) {
                        //2
                        if (ss >= 0) {
                            if (U.compareAndSwapObject(a, i, t, null)) {
                                q.base = b + 1;
                                if (n < -1)       // signal others
                                    signalWork(ws, q);
                                return t;
                            }
                        }
                        else if (oldSum == 0 &&   // try to activate
                                 w.scanState < 0)
                            tryRelease(c = ctl, ws[m & (int)c], AC_UNIT);
                    }
                    if (ss < 0)                   // refresh
                        ss = w.scanState;
                    r ^= r << 1; r ^= r >>> 3; r ^= r << 10;
                    origin = k = r & m;           // move and rescan
                    oldSum = checkSum = 0;
                    continue;
                }
                checkSum += b;
            }
            //3
            if ((k = (k + 1) & m) == origin) {    // continue until stable
                if ((ss >= 0 || (ss == (ss = w.scanState))) &&
                    oldSum == (oldSum = checkSum)) {
                    if (ss < 0 || w.qlock < 0)    // already inactive
                        break;
                    int ns = ss | INACTIVE;       // try to inactivate
                    long nc = ((SP_MASK & ns) |
                               (UC_MASK & ((c = ctl) - AC_UNIT)));
                    w.stackPred = (int)c;         // hold prev stack top
                    U.putInt(w, QSCANSTATE, ns);
                    if (U.compareAndSwapLong(this, CTL, c, nc))
                        ss = ns;
                    else
                        w.scanState = ss;         // back out
                }
                checkSum = 0;
            }
        }
    }
    return null;
}
```
首先是获得了当前线程对应的workQueue的scanState状态。
在mark2的时候要判断这个状态:
- 如果是active，那直接拿着偷到的task返回。
- 如果是inactive，调用tryRelease激活这个workQueue

在mark3处，每次循环会校验新取的index是不是等于第一次取的index。如果相等，说明遍历了一圈还没有steal到任务，当前worker是过剩的，执行如下操作：
- 当前WorkQueue的scanState修改为inactive；
- 当前WorkQueue挂到栈顶，AC减一。


偷到一个任务以后就执行它
```java
final void runTask(ForkJoinTask<?> task) {
    if (task != null) {
        scanState &= ~SCANNING; // mark as busy
        (currentSteal = task).doExec();
        U.putOrderedObject(this, QCURRENTSTEAL, null); // release for GC
        execLocalTasks();
        ForkJoinWorkerThread thread = owner;
        if (++nsteals < 0)      // collect on overflow
            transferStealCount(pool);
        scanState |= SCANNING;
        if (thread != null)
            thread.afterTopLevelExec();
    }
}
```
steal到一个任务后，就可以开始执行：

- 将WorkQueue的scanState从SCANNING转为RUNNING；
- 记录当前任务是steal来的，保存在currentSteal，并执行doExec；
- 执行自己WorkQueue里的任务execLocalTasks（根据mode控制取任务是- LIFO还是FIFO，调用doExec执行，直到WorkQueue为空）；
- 累加steal数量；
- 能执行的都执行了，scanState转回SCANNING。

```java
final int doExec() {
    int s; boolean completed;
    if ((s = status) >= 0) {
        try {
            completed = exec();
        } catch (Throwable rex) {
            return setExceptionalCompletion(rex);
        }
        if (completed)
            s = setCompletion(NORMAL);
    }
    return s;
}

private int setCompletion(int completion) {
   for (int s;;) {
       if ((s = status) < 0)
           return s;
       if (U.compareAndSwapInt(this, STATUS, s, s | completion)) {
           if ((s >>> 16) != 0)
               synchronized (this) { notifyAll(); }
           return completion;
       }
   }
}
```
doExec方法，里面最终调用ForkJoinTask的核心方法exec，前文介绍过，RecursiveAction和RecursiveTask它们override的exce调用了compute。这样子，源码和使用的方法关联起来了。

当任务执行完成，调用setCompletion，将任务状态改为NORMAL。注意，使用CAS修改状态时，目标状态使用s|NORMAL。

- 原状态是NORMAL，无符号右移为0；
- 原状态是SIGNAL，无符号右移不为0。
如果任务原状态是SIGNAL，表示有线程由于join而进入了wait，等着任务完成，这时需要额外操作notify触发唤醒。


没有偷到任务就阻塞这个工作线程
```java
private boolean awaitWork(WorkQueue w, int r) {
    if (w == null || w.qlock < 0)                 // w is terminating
        return false;
    for (int pred = w.stackPred, spins = SPINS, ss;;) {
        //1
        if ((ss = w.scanState) >= 0)
            break;
       //2
        else if (spins > 0) {
            r ^= r << 6; r ^= r >>> 21; r ^= r << 7;
            if (r >= 0 && --spins == 0) {         // randomize spins
                WorkQueue v; WorkQueue[] ws; int s, j; AtomicLong sc;
                if (pred != 0 && (ws = workQueues) != null &&
                    (j = pred & SMASK) < ws.length &&
                    (v = ws[j]) != null &&        // see if pred parking
                    (v.parker == null || v.scanState >= 0))
                    spins = SPINS;                // continue spinning
            }
        }
        else if (w.qlock < 0)                     // recheck after spins
            return false;
       //3
        else if (!Thread.interrupted()) {
            long c, prevctl, parkTime, deadline;
            int ac = (int)((c = ctl) >> AC_SHIFT) + (config & SMASK);
           //4
            if ((ac <= 0 && tryTerminate(false, false)) ||
                (runState & STOP) != 0)           // pool terminating
                return false;
            //5
            if (ac <= 0 && ss == (int)c) {        // is last waiter
                prevctl = (UC_MASK & (c + AC_UNIT)) | (SP_MASK & pred);
                int t = (short)(c >>> TC_SHIFT);  // shrink excess spares
                if (t > 2 && U.compareAndSwapLong(this, CTL, c, prevctl))
                    return false;                 // else use timed wait
                parkTime = IDLE_TIMEOUT * ((t >= 0) ? 1 : 1 - t);
                deadline = System.nanoTime() + parkTime - TIMEOUT_SLOP;
            }
            else
                prevctl = parkTime = deadline = 0L;
            Thread wt = Thread.currentThread();
            U.putObject(wt, PARKBLOCKER, this);   // emulate LockSupport
            w.parker = wt;
            if (w.scanState < 0 && ctl == c)      // recheck before park
                U.park(false, parkTime);
            U.putOrderedObject(w, QPARKER, null);
            U.putObject(wt, PARKBLOCKER, null);
            if (w.scanState >= 0)
                break;
            if (parkTime != 0L && ctl == c &&
                deadline - System.nanoTime() <= 0L &&
                U.compareAndSwapLong(this, CTL, c, prevctl))
                return false;                     // shrink pool
        }
    }
    return true;
}
```
awaitWork里核心是一个无限循环，我们重点看里面的等待操作和跳出条件。

mark1判断WorkQueue的scanState，非负表示WorkQueue要不在RUNNING，要不在SCANNING，直接跳出。mark2里，SPINS初始为0，没有启用自旋等待的控制。

重点来看mark3，只要没有中断，就会一直循环执行（tryTerminate终止ForkJoinPool时会中断所有worker）。啰嗦一句，要分清楚return和break的不同含义：

- break：回到runWorker继续执行scan、runTask、awaitWork；
- return false：worker需要终止了。
mark4检查ForkJoinPool的状态，如果走向中止那边，当前worker也就无必要存在，return false。

mark5判断worker的存在是否有必要，如果满足下面条件：

- AC为零；
- TC超过2个；
- 当前WorkQueue在栈顶。

说明当前worker过剩，存在也没有任务执行，所以WorkQueue从栈顶释放，return false终止worker。

其他情况计算一个等待时间，挂起线程，被唤醒有两种可能：

- 外部唤醒：如果scanState非负，break出循环，继续执行scan；
- 时间到达唤醒：还是老样子，自己过剩，return false终止。


最后回到ForkJoinTask里的方法中。
## ForkJoinTask Fork
```java
public final ForkJoinTask<V> fork() {
    Thread t;
    if ((t = Thread.currentThread()) instanceof ForkJoinWorkerThread)
        ((ForkJoinWorkerThread)t).workQueue.push(this);
    else
        ForkJoinPool.common.externalPush(this);
    return this;
}
```
fork的代码很简单，如果当前线程是一个worker，直接将任务从top端加入自己的WorkQueue。对于非worker提交的task，执行externalPush，这个前面详细分析过了。

## ForkJoinTask Join
```java
public final V join() {
    int s;
    if ((s = doJoin() & DONE_MASK) != NORMAL)
        reportException(s);
    return getRawResult();
}

private int doJoin() {
    int s; Thread t; ForkJoinWorkerThread wt; ForkJoinPool.WorkQueue w;
    return (s = status) < 0 ? s :
        ((t = Thread.currentThread()) instanceof ForkJoinWorkerThread) ?
        (w = (wt = (ForkJoinWorkerThread)t).workQueue).
        tryUnpush(this) && (s = doExec()) < 0 ? s :
        wt.pool.awaitJoin(w, this, 0L) :
        externalAwaitDone();
}
```
join的目的是得到任务的运行结果，核心调用doJoin，根据任务状态返回结果，或者抛出异常。要注意的是，任务在ForkJoinPool中可能处于各种各样的状况，有可能刚好要被执行啊，有可能正在队列里排队啊，有可能已经被别人偷走啊。

doJoin的return是花一样的一串判断，先分解出头两个判断：

- status为负表示任务执行已经有结果，直接返回；
- 区分当前线程是否worker。

如果当前线程不是ForkJoinWorkerThread这种情况，调用externalAwaitDone

```java
private int externalAwaitDone() {
    int s = ((this instanceof CountedCompleter) ? // try helping
             ForkJoinPool.common.externalHelpComplete(
                 (CountedCompleter<?>)this, 0) :
             ForkJoinPool.common.tryExternalUnpush(this) ? doExec() : 0);
    if (s >= 0 && (s = status) >= 0) {
        boolean interrupted = false;
        do {
            if (U.compareAndSwapInt(this, STATUS, s, s | SIGNAL)) {
                synchronized (this) {
                    if (status >= 0) {
                        try {
                            wait(0L);
                        } catch (InterruptedException ie) {
                            interrupted = true;
                        }
                    }
                    else
                        notifyAll();
                }
            }
        } while ((s = status) >= 0);
        if (interrupted)
            Thread.currentThread().interrupt();
    }
    return s;
}
```
先不讲CountedCompleter的协作，将任务状态设置为SIGNAL，然后是使用wait/notify机制，线程进入等待。既然不是worker，不属于ForkJoinPool的管理范围，你挂起等通知就是了。

然后是dojoin里的剩下的判断
```java
(w = (wt = (ForkJoinWorkerThread)t).workQueue).
        tryUnpush(this) && (s = doExec()) < 0 ? s :
        wt.pool.awaitJoin(w, this, 0L) 
```
首先调用tryUnpush，如果WorkQueue的top端任务正好是等待join的任务，毫无疑问，下个就是执行它，直接doExec；否则调用ForkJoinPool的awaitJoin。

# awaitJoin看晕了，有时间再看