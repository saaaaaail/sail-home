---
title: AQS原理详解
date: 2020-07-04 15:17:42
tags:
- juc
- aqs
---

# AQS原理/源码详解

AQS全称为AbstractQueuedSynchronizeder（抽象队列同步器），juc包中Lock、CountDownLatch、CyclicBarrier、Phaser、Exchanger等线程同步工具都是基于AQS实现的。

AQS的基本思想是 如果共享资源是空闲的，那么将当前请求资源的线程设定为获得这个资源的有效线程，将共享资源设置为锁定的状态不能被其他线程访问，这时候新来的线程发现这个资源处于占用的状态，无法获得使用资源的锁，就将这些线程阻塞并加入到队列中等待。

# 源码解读

## 基本数据结构

AQS中定义了一个Node静态内部类，维护双向的fifo链表。包装需要阻塞的线程进入队列。

```java
    static final class Node {
        /** 表示这个节点在等待的锁是共享的模式 */
        static final Node SHARED = new Node();
        /** 表示这个节点等待的锁是独占的模式 */
        static final Node EXCLUSIVE = null;

        /** 表示取消状态，可以理解为线程失效了 */
        static final int CANCELLED =  1;
        /** 此节点状态为SIGNAL表示他的next节点可以被唤醒  */
        static final int SIGNAL    = -1;
        /** 表示这个节点的线程是从condition里的单向等待队列移出来的，也是可以唤醒的节点，根据后面代码的操作，不能唤醒额节点为 */
        static final int CONDITION = -2;
        /**
         * 这个状态用于共享模式，解释待定稍等一下啊
         */
        static final int PROPAGATE = -3;

        /**
         * Status field, taking on only the values:
         *   SIGNAL:     The successor of this node is (or will soon be)
         *               blocked (via park), so the current node must
         *               unpark its successor when it releases or
         *               cancels. To avoid races, acquire methods must
         *               first indicate they need a signal,
         *               then retry the atomic acquire, and then,
         *               on failure, block.
         *   CANCELLED:  This node is cancelled due to timeout or interrupt.
         *               Nodes never leave this state. In particular,
         *               a thread with cancelled node never again blocks.
         *   CONDITION:  This node is currently on a condition queue.
         *               It will not be used as a sync queue node
         *               until transferred, at which time the status
         *               will be set to 0. (Use of this value here has
         *               nothing to do with the other uses of the
         *               field, but simplifies mechanics.)
         *   PROPAGATE:  A releaseShared should be propagated to other
         *               nodes. This is set (for head node only) in
         *               doReleaseShared to ensure propagation
         *               continues, even if other operations have
         *               since intervened.
         *   0:          None of the above
         *
         * The values are arranged numerically to simplify use.
         * Non-negative values mean that a node doesn't need to
         * signal. So, most code doesn't need to check for particular
         * values, just for sign.
         *
         * The field is initialized to 0 for normal sync nodes, and
         * CONDITION for condition nodes.  It is modified using CAS
         * (or when possible, unconditional volatile writes).
         */
        volatile int waitStatus;

        /**
         * Link to predecessor node that current node/thread relies on
         * for checking waitStatus. Assigned during enqueuing, and nulled
         * out (for sake of GC) only upon dequeuing.  Also, upon
         * cancellation of a predecessor, we short-circuit while
         * finding a non-cancelled one, which will always exist
         * because the head node is never cancelled: A node becomes
         * head only as a result of successful acquire. A
         * cancelled thread never succeeds in acquiring, and a thread only
         * cancels itself, not any other node.
         */
        volatile Node prev;

        /**
         * Link to the successor node that the current node/thread
         * unparks upon release. Assigned during enqueuing, adjusted
         * when bypassing cancelled predecessors, and nulled out (for
         * sake of GC) when dequeued.  The enq operation does not
         * assign next field of a predecessor until after attachment,
         * so seeing a null next field does not necessarily mean that
         * node is at end of queue. However, if a next field appears
         * to be null, we can scan prev's from the tail to
         * double-check.  The next field of cancelled nodes is set to
         * point to the node itself instead of null, to make life
         * easier for isOnSyncQueue.
         */
        volatile Node next;

        /**
         * The thread that enqueued this node.  Initialized on
         * construction and nulled out after use.
         */
        volatile Thread thread;

        /**
         * Link to next node waiting on condition, or the special
         * value SHARED.  Because condition queues are accessed only
         * when holding in exclusive mode, we just need a simple
         * linked queue to hold nodes while they are waiting on
         * conditions. They are then transferred to the queue to
         * re-acquire. And because conditions can only be exclusive,
         * we save a field by using special value to indicate shared
         * mode.
         */
        Node nextWaiter;

        /**
         * Returns true if node is waiting in shared mode.
         */
        final boolean isShared() {
            return nextWaiter == SHARED;
        }

        /**
         * Returns previous node, or throws NullPointerException if null.
         * Use when predecessor cannot be null.  The null check could
         * be elided, but is present to help the VM.
         *
         * @return the predecessor of this node
         */
        final Node predecessor() throws NullPointerException {
            Node p = prev;
            if (p == null)
                throw new NullPointerException();
            else
                return p;
        }

        Node() {    // Used to establish initial head or SHARED marker
        }

        Node(Thread thread, Node mode) {     // Used by addWaiter
            this.nextWaiter = mode;
            this.thread = thread;
        }

        Node(Thread thread, int waitStatus) { // Used by Condition
            this.waitStatus = waitStatus;
            this.thread = thread;
        }
    }
```

还包括双向同步队列的头指针尾指针和state同步状态，以及对state进行原子性更新的cas操作。

```java
    /**
     * Head of the wait queue, lazily initialized.  Except for
     * initialization, it is modified only via method setHead.  Note:
     * If head exists, its waitStatus is guaranteed not to be
     * CANCELLED.
     */
    private transient volatile Node head;

    /**
     * Tail of the wait queue, lazily initialized.  Modified only via
     * method enq to add new wait node.
     */
    private transient volatile Node tail;

    /**
     * The synchronization state.
     */
    private volatile int state;

    protected final boolean compareAndSetState(int expect, int update) {
        // See below for intrinsics setup to support this
        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
    }
```

## 独占锁的获取

```java
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```
上述方法用于获取一个独占锁，首先尝试获取一下锁，如果获取失败就会调用 <kbc>addWaiter</kbc> 方法添加到线程到同步队列的末尾，然后通过 <kbc>acquireQueued</kbc> 方法再次尝试加锁，以及判断是否应该中断线程。

<kbc>tryAcquire</kbc>为模板方法，其操作由其子类去实现。

```java
    private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        //队尾节点不为空
        if (pred != null) {
            node.prev = pred;
            //通过cas操作，将队尾节点设置为node，注意这里仅仅进行了一次cas，如果其尾节点已经被修改了，这里的cas操作就会失败。
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        //如果尾节点为空，或者上面cas操作失败了，就接下来继续进行循环的cas操作，直到入队成功
        enq(node);
        //这里紧接着就把当前线程的node返回出去了
        return node;
    }

    private Node enq(final Node node) {
        //循环着的cas操作，设置尾节点或者在尾节点插入当前节点，如果失败了就重新获得尾节点再插直到成功。
        for (;;) {
            Node t = tail;
            if (t == null) { // Must initialize
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }

```
如上面分析可知，新来的线程如果获得state失败，就会通过自旋的方式插入同步队列的末尾。

下面接着分析<kbc>acquireQueued</kbc>方法
```java
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            //又见一个自旋操作，自旋的设置为头节点，如果不成功就阻塞掉
            for (;;) {
                //首先获得前置节点
                final Node p = node.predecessor();
                //在同步队列中，头部节点是正在占用资源的节点，能够获得资源的节点必须是头部节点的下一个节点  
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                //如果上面的操作失败，就判断是否应该阻塞当前线程node，如果前一个节点的waitStatus状态为SIGNAL，则阻塞，如果前一个节点waitStatus没有初始化，则改为SIGNAL
                //parkAndCheckInterrupt()阻塞当前线程，并检查是否中断
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }

    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        //一旦当前线程node的前置节点设置为SIGNAL，直接返回true，要注意上面那个方法 shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt() 这个连接结构，如果前面的方法返回为true了，后面的才会执行
        if (ws == Node.SIGNAL)
            /*
             * This node has already set status asking a release
             * to signal it, so it can safely park.
             */
            return true;
        if (ws > 0) {
            /*
             * 过滤掉CANCELED状态的node
             */
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            /*
             * 又是一个cas操作，通过外层循环，不断设置前置节点的状态为SIGNAL
             */
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }
    //如果shouldParkAfterFailedAcquire成功设置前一个节点为SIGNAL状态以后，就阻塞当前线程，并检查是否中断
    //因为只有当前一个节点为SIGNAL状态后才表明后一个节点可以被唤醒，才能将后一个节点阻塞，不一定立即被唤醒，比较队列前面也可能有很多阻塞节点
     private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
    }
```

以上解释了AQS的独占模式的获取state。

## 独占锁的释放

```java
    public final boolean release(int arg) {
        //调用tryRelease释放资源，释放成功就state-arg，然后就LockSupport.unpark头节点的下一个节点
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }

    private void unparkSuccessor(Node node) {
        /*
         * If status is negative (i.e., possibly needing signal) try
         * to clear in anticipation of signalling.  It is OK if this
         * fails or if status is changed by waiting thread.
         */
        int ws = node.waitStatus;
        //首先cas将当前节点waitStatus初始化为0
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);

        /*
         * Thread to unpark is held in successor, which is normally
         * just the next node.  But if cancelled or apparently null,
         * traverse backwards from tail to find the actual
         * non-cancelled successor.
         */
        Node s = node.next;
        //考虑异常情况，找到head的下一个节点
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        //唤醒这个节点
        if (s != null)
            LockSupport.unpark(s.thread);
    }
```

tryRelease方法也是模板方法，要求子类实现。

**release方法总结**：线程资源释放调用release方法，使用tryRelease尝试释放资源，如果成功将state-1，再调用AQS的父类AbstractOwnableSynchronizer设置独占线程为null，再使用LockSupport.unpark头节点的next节点。

## 共享锁的获取与释放

```java
    public final void acquireShared(int arg) {
        //模板方法要求子类实现，尝试获得共享资源 小于0表示失败，0表示成功但是没有剩余资源，大于0表示成功还有剩余资源
        if (tryAcquireShared(arg) < 0)
            doAcquireShared(arg);
    }

    private void doAcquireShared(int arg) {
        //在同步队列尾添加共享模式节点
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head) {
                    //如果前置节点是头节点，占用资源的线程，那么当前节点可能被唤醒，因此再次尝试获取资源
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        //成功以后设置为头节点，资源有剩余的情况下唤醒下一个节点
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        if (interrupted)
                            selfInterrupt();
                        failed = false;
                        return;
                    }
                }
                //这里的方法与独占模式相同，如果没有获得资源就将前置节点状态设为SIGNAL表明下一个节点可唤醒，然后阻塞当前线程，同时判断是否需要中断
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }

     private void setHeadAndPropagate(Node node, int propagate) {
        Node h = head; // Record old head for check below
        setHead(node);
        
        //除了设置为头节点以为，如果还有剩余资源，或者头节点为SIGNAL状态
        //这里的语义是 资源有剩余 || h(旧头节点)为空 || h(旧头节点)状态<0 || 赋值为新到head，且新head为空 || 新head状态<0
        if (propagate > 0 || h == null || h.waitStatus < 0 ||
            (h = head) == null || h.waitStatus < 0) {
            Node s = node.next;
            if (s == null || s.isShared())
                doReleaseShared();
        }
    }
```

由上面的代码可以知道，与独占模式的区别在于当前节点设为头节点的时候，如果有剩余资源会继续唤醒线程doReleaseShared。

```java
    public final boolean releaseShared(int arg) {
        //模板方法，尝试释放资源
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }

    private void doReleaseShared() {
        /*
         * 作用总的来说是不断唤醒下一个节点，可以唤醒多余的节点（没有获取资源就又park掉），但是不能主动让唤醒链断掉
         */
        for (;;) {
            Node h = head;
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                if (ws == Node.SIGNAL) {
                    //初始化头节点的waitStatus状态
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;            // loop to recheck cases
                    //这个方法是unpark h.next节点
                    unparkSuccessor(h);
                }
                //这里的操作是 如果后继节点还没有将head设置为SIGNAL，表示可能无需唤醒后继节点或者还没有加入后继节点，将其设置为PROPAGATE，用意是为了防止唤醒hang住。     
                else if (ws == 0 &&
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;                // loop on failed CAS
            }
            if (h == head)                   // loop if head changed
                break;
        }
    }
```
这里是唤醒当前头节点的下一个节点，然后在setHeadAndPropagate方法中设置新头。

其中有一个没有显式使用的状态 **PROPAGATE** 需要重点关注下， 首先关注上述唤醒下一个节点的操作，这里有两个操作，一个是SIGNAL状态那就直接唤醒下一个节点，还一个如果是初始化状态就设置为PROPAGATE状态，然后如果head被改过了就重来一遍，然后结束。

第一个操作好理解，问题是第二个操作的用意是什么，为什么要将0状态的head改为PROPAGATE状态，要知道上面的操作还将SIGNAL初始化成0了，为什么呢？

因为首先要考虑被唤醒以后的操作，在doAcquiredShare方法里park位置被唤醒，然后判断前置节点是父节点就尝试获取锁，（如果不成功就又被park不考虑）如果成功了，就setHeadAndPropagate，这个方法除了设置新头节点外，有下面这个判断条件决定是否唤醒下一个节点：
```java
        if (propagate > 0 || h == null || h.waitStatus < 0 ||
            (h = head) == null || h.waitStatus < 0) {
     
            Node s = node.next;
            if (s == null || s.isShared())
                doReleaseShared();
        }
```
其中资源大于0自然可以唤醒下一个，旧头等于空也尝试唤醒（虽然很可能没有回收），旧头的waitStatus状态小于0也尝试唤醒，新头为空尝试唤醒（可能很小），新头waitStatus小于0尝试唤醒，重点关注下这俩waitStatus。

根据这篇博文的叙述[AQS中Propagate的作用解释](https://www.cnblogs.com/lqlqlq/p/12991275.html)。

当前队列唤醒一个节点A以后，其waitStatus状态未被修改，为0，那么A执行尝试获取资源成功，修改头节点，在执行到上述if语句的的时候，新来一个线程node B加入队列，B尝试获取资源失败，执行shouldParkAfterFailedAcquire方法之前，A判断if propagate==0 || (old)h!=null || (old)h.ws==0 (唤醒下个节点前会设置成0)|| (new)h!=null ||(new)h.ws==0  然后就判断失败了，不继续唤醒下一个节点，然后B执行shouldParkAfterFailedAcquire完成设置前置节点的状态，此时B不再被唤醒。

说实话，如果仅仅是因为这个原因，感觉没有PROPAGATE，问题也不严重，这个问题待定。


## ConditionObject
```java
    public class ConditionObject implements Condition, java.io.Serializable {
        private static final long serialVersionUID = 1173984872572414699L;
        /** First node of condition queue. */
        private transient Node firstWaiter;
        /** Last node of condition queue. */
        private transient Node lastWaiter;

        /**
         * Creates a new {@code ConditionObject} instance.
         */
        public ConditionObject() {}
```
condition内部维护的第一条单向链表的头尾节点。

### await方法
```java
        public final void await() throws InterruptedException {
            if (Thread.interrupted())
                throw new InterruptedException();
            //把当前线程封装成node节点加入condition单向列表
            Node node = addConditionWaiter();
            //这个方法会调用AQS的release方法，释放同步资源，然后唤醒同步队列头节点的下一个节点
            int savedState = fullyRelease(node);
            int interruptMode = 0;
            //这里是判断当前node是不是在AQS的同步队列里，如果不在，堵塞线程。
            //这里之所以循环就是 当signal还没有将节点移到同步队列上时，即使唤醒了仍然阻塞
            while (!isOnSyncQueue(node)) {
                //这里就是condition等待队列阻塞的位置
                LockSupport.park(this);
                //这里是唤醒以后检查中断标志
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
            //这里是被从condition队列里唤醒以后，再在同步队列里等待获取资源
            //再次尝试给node获取同步资源，如果不成，将node park
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null) // clean up if cancelled
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
        }

        //这个方法很明确，就是将线程封装成node添加到单向队列尾
        private Node addConditionWaiter() {
            Node t = lastWaiter;
            // If lastWaiter is cancelled, clean out.
            if (t != null && t.waitStatus != Node.CONDITION) {
                unlinkCancelledWaiters();
                t = lastWaiter;
            }
            Node node = new Node(Thread.currentThread(), Node.CONDITION);
            if (t == null)
                firstWaiter = node;
            else
                t.nextWaiter = node;
            lastWaiter = node;
            return node;
        }

        final int fullyRelease(Node node) {
            //传进来的node是当前线程node
            boolean failed = true;
            try {
                //获得同步状态
                int savedState = getState();
                //调用AQS的release释放资源
                if (release(savedState)) {
                    failed = false;
                    return savedState;
                } else {
                    throw new IllegalMonitorStateException();
                }
            } finally {
                //如果释放失败的话，就会将当前node置为canceled状态
                if (failed)
                    node.waitStatus = Node.CANCELLED;
            }
        }

        //这个方法也很明确吖，就是判断这个node在不在AQS同步队列里
        final boolean isOnSyncQueue(Node node) {
        if (node.waitStatus == Node.CONDITION || node.prev == null)
            return false;
        if (node.next != null) // If has successor, it must be on queue
            return true;
        /*
         * node.prev can be non-null, but not yet on queue because
         * the CAS to place it on queue can fail. So we have to
         * traverse from tail to make sure it actually made it.  It
         * will always be near the tail in calls to this method, and
         * unless the CAS failed (which is unlikely), it will be
         * there, so we hardly ever traverse much.
         */
        return findNodeFromTail(node);
    }
```
**await** 方法总结：调用condition.await()方法，意味着当前线程进入等待状态。将当前节点包装成node节点，放入condition链表的尾部。然后调用AQS的release方法，释放state，locksupport.unpack()双向node链表的头结点线程。之后，再将自身线程堵塞。


### signal方法

```java
        public final void signal() {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            Node first = firstWaiter;
            if (first != null)
                doSignal(first);
        }

        private void doSignal(Node first) {
            do {
                if ( (firstWaiter = first.nextWaiter) == null)
                    lastWaiter = null;
                //这里是将first的下一个等待队列节点置空
                first.nextWaiter = null;
                //下面这个transfer方法就是将当前node即等待队列第一个节点移动到AQS同步队列的末尾，同时修改ws状态
            } while (!transferForSignal(first) &&
                     (first = firstWaiter) != null);
        }

        final boolean transferForSignal(Node node) {
            /*
            * If cannot change waitStatus, the node has been cancelled.
            */
            //cas操作将当前node状态设置为0
            if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
                return false;

            //通过自旋将node添加到同步队列末尾,返回前置节点，设置前置节点的状态为SIGNAL
            Node p = enq(node);
            int ws = p.waitStatus;
            if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
                LockSupport.unpark(node.thread);
            return true;
        }

        //同时简单看一下signalAll里面的方法，就是遍历condition的等待队列，全部移动到AQS的同步对列
        private void doSignalAll(Node first) {
            lastWaiter = firstWaiter = null;
            do {
                Node next = first.nextWaiter;
                first.nextWaiter = null;
                transferForSignal(first);
                first = next;
            } while (first != null);
        }
```
**signal**方法总结：signal就是将condition的等待队列的第一个node移动到AQS的同步队列中同时修改ws状态，signalAll就是将condition的等待队列所有node移动到AQS的同步队列中同时修改ws状态。