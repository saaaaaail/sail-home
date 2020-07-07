---
title: ReentrantLock原理详解
date: 2020-07-02 15:33:03
tags:
---

# ReentrantLock原理详解

> 提问：介绍一下ReentrantLock？

## 基础方法
```
Lock lock = new ReentrantLock();
```

1. lock()/unlock() 
2. tryLock()
   ```
   可以进行加锁尝试 ，如果一段时间没有获得锁，就停止尝试获得锁返回false
   ```

3. lockInterruptibly()
    ```
    通过这个方式加锁的，可以被线程的interrupt()方法打断
    ```

### 公平锁

```
Lock lock = new ReentrantLock(true);
```
> 提问：Lock公平锁为什么不能保证绝对的公平？


## 源码解读

ps:以下是ReentrantLock源码分析，AQS原理不再赘述噢，AQS源码解读-> [AQS源码解读](https://saaaaaail.gitee.io/2020/07/04/AQS%E5%8E%9F%E7%90%86%E8%AF%A6%E8%A7%A3/#AQS%E5%8E%9F%E7%90%86-%E6%BA%90%E7%A0%81%E8%AF%A6%E8%A7%A3)

```java
public class ReentrantLock implements Lock, java.io.Serializable {
    private static final long serialVersionUID = 7373984872572414699L;

    private final Sync sync;

    abstract static class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = -5179523762034025860L;

         //非公平锁与公平锁子类实现
        abstract void lock();

         //这个是非公平锁的tryAcquire实现，acquires==1
        final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            //获得最新的同步状态
            int c = getState();
            //如果c==0表示资源为空，可以占用
            if (c == 0) {
                //通过cas操作加锁，如果成功设置占用线程，并返回true
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                //如果资源已经被占用，判断是不是当前线程，如果是，将同步状态重入次数+1
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            //上面都不成功返回false
            return false;
        }

        //尝试释放同步资源 这里release==1       
        protected final boolean tryRelease(int releases) {
            //获得最新的同步资源状态state-1
            int c = getState() - releases;
            //如果访问线程不是独线程的话抛错
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            //如果c==0说明重入次数仅为1，可以直接释放
            if (c == 0) {
                free = true;
                //情况独占线程
                setExclusiveOwnerThread(null);
            }
            //设置同步状态位
            setState(c);
            return free;
        }

        protected final boolean isHeldExclusively() {
            return getExclusiveOwnerThread() == Thread.currentThread();
        }

        final ConditionObject newCondition() {
            return new ConditionObject();
        }

        // Methods relayed from outer class

        final Thread getOwner() {
            return getState() == 0 ? null : getExclusiveOwnerThread();
        }

        final int getHoldCount() {
            return isHeldExclusively() ? getState() : 0;
        }

        final boolean isLocked() {
            return getState() != 0;
        }

        /**
         * Reconstitutes the instance from a stream (that is, deserializes it).
         */
        private void readObject(java.io.ObjectInputStream s)
            throws java.io.IOException, ClassNotFoundException {
            s.defaultReadObject();
            setState(0); // reset to unlocked state
        }
    }
```
ReentrantLock实现了Lock接口，还包含一个继承自AQS的静态内部类sync同步器，这个同步器实现了AQS里没有实现tryAcquire方法和tryRelease方法，下面看看非公平锁与公平锁的同步器。

```java
    //非公平锁的同步器
    static final class NonfairSync extends Sync {
        private static final long serialVersionUID = 7316153563782823691L;

        //非公平锁的lock方法
        final void lock() {
            //cas尝试设置state为1
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                //如果失败了就调用AQS的acquire方法去获得资源或者放入AQS同步队列里阻塞
                acquire(1);
        }
        //就是直接调用的sync里的tryAcquire方法
        protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
    }

    //公平锁的同步器
    static final class FairSync extends Sync {
        private static final long serialVersionUID = -3000897897090466540L;

        //公平锁就直接走AQS的同步队列了，也不用cas尝试一下
        final void lock() {
            acquire(1);
        }

        //公平锁的tryAcquire方法
        protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                //判断一下AQS同步队列中没有元素，尝试加锁
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                //这里依然是可重入判断
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
    }
```
由上叙述可知，公平锁与非公平锁的区别在于，公平锁获取资源完全交由AQS同步队列去安排，尝试加锁还要判断队列中有无元素，非公平锁虽然也会交给AQS同步队列，但是新来的线程会自己cas尝试获得锁，只要同步状态为0，就能抢夺资源。


ReentrantLock构造方法，默认非公平锁，设置为true就是公平锁
```java
    public ReentrantLock() {
        sync = new NonfairSync();
    }

    public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }
```
下面看一看ReentrantLock的成员方法
```java
    public void lock() {
        //同步器里的lock方法
        sync.lock();
    }

    public void lockInterruptibly() throws InterruptedException {
        //这是AQS里的可中断版本的acquire
        sync.acquireInterruptibly(1);
    }

    public boolean tryLock() {
        //这个写在sync里，就是公平锁也可以调这个
        return sync.nonfairTryAcquire(1);
    }

    public boolean tryLock(long timeout, TimeUnit unit)
            throws InterruptedException {
        return sync.tryAcquireNanos(1, unit.toNanos(timeout));
    }
    
    public void unlock() {
        sync.release(1);
    }

    public Condition newCondition() {
        return sync.newCondition();
    }

    public int getHoldCount() {
        return sync.getHoldCount();
    }

    public boolean isHeldByCurrentThread() {
        return sync.isHeldExclusively();
    }

    public boolean isLocked() {
        return sync.isLocked();
    }

    public final boolean isFair() {
        return sync instanceof FairSync;
    }
```

以上方法基本都看过了，其中有几个aqs的方法没有分析过，下面来看一看

```java
    public final boolean tryAcquireNanos(int arg, long nanosTimeout)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        return tryAcquire(arg) ||
            doAcquireNanos(arg, nanosTimeout);
    }
    private boolean doAcquireNanos(int arg, long nanosTimeout)
            throws InterruptedException {
        if (nanosTimeout <= 0L)
            return false;
        final long deadline = System.nanoTime() + nanosTimeout;
        final Node node = addWaiter(Node.EXCLUSIVE);
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return true;
                }
                //存在一个计时器，每次自旋计算剩下多少时间
                nanosTimeout = deadline - System.nanoTime();
                if (nanosTimeout <= 0L)
                    return false;
                if (shouldParkAfterFailedAcquire(p, node) &&
                    nanosTimeout > spinForTimeoutThreshold)
                    //这个parkNanos方法会调用unsafe的parkNanos方法阻塞这个线程剩余nanosTimeout的时间
                    LockSupport.parkNanos(this, nanosTimeout);
                if (Thread.interrupted())
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
```
tryAcquireNanos方法实现了一个有时间限制的自旋尝试获取资源的操作，其中添加了deadline与当前时间比较，即使被park阻塞也只会阻塞剩余时间就自动被唤醒了。其他方法与AQS中的acquireQueued相同。

```java
    public final void acquireInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        if (!tryAcquire(arg))
            doAcquireInterruptibly(arg);
    }
    private void doAcquireInterruptibly(int arg)
        throws InterruptedException {
        final Node node = addWaiter(Node.EXCLUSIVE);
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    //可见这个方法检测到了中断标志位就
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```