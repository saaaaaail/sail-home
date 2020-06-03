---
title: CompareAndSwap原理详解
date: 2020-06-03 09:14:28
tags:
- juc
- CAS
---

# CompareAndSwap原理详解 

谈到CAS操作就离不开原子类，在多个线程中对同一变量进行操作需要加锁，否则会存在线程安全问题，但是在多个线程对原子类进行操作的时候不需要加锁原因就是使用了CAS。

CAS即CompareAndSwap比较与交换，又称为无锁、自选锁、乐观锁。

以AtomicInteger i = new AtomicInteger(1)为例:
```
public class AtomicInteger extends Number implements java.io.Serializable {
    private static final long serialVersionUID = 6214790243416807050L;

    // setup to use Unsafe.compareAndSwapInt for updates
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    private static final long valueOffset;

    static {
        try {
            valueOffset = unsafe.objectFieldOffset
                (AtomicInteger.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }

    private volatile int value;

    /**
     * Creates a new AtomicInteger with the given initial value.
     *
     * @param initialValue the initial value
     */
    public AtomicInteger(int initialValue) {
        value = initialValue;
    }

    /**
     * Creates a new AtomicInteger with initial value {@code 0}.
     */
    public AtomicInteger() {
    }
```
类中包含两个静态变量，
- 一个是unsafe实例
- 一个是valueOffset值，表示value这个值在主内存中的偏移量，用于使用unsafe类直接对该内存地址的值进行操作。

还包含一个volatile修饰的value变量，保证每次读取value都能获得当前最新值。对于volatile详见[volatile原理详解]()

然后观察一下AtomicInteger的方法:
```

```