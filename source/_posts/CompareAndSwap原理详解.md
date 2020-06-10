---
title: CompareAndSwap原理详解
date: 2020-06-03 09:14:28
tags:
- juc
- CAS
- jdk1.8
---

# CompareAndSwap原理详解 

谈到CAS操作就离不开原子类，在多个线程中对同一变量进行操作需要加锁，否则会存在线程安全问题，但是在多个线程对原子类进行操作的时候不需要加锁原因就是使用了CAS。

CAS即CompareAndSwap比较与交换，又称为无锁、自选锁、乐观锁。

以AtomicInteger i = new AtomicInteger(1)为例:
```java
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

还包含一个volatile修饰的value变量，保证每次读取value都能获得当前最新值。对于volatile详见[volatile原理详解](https://saaaaaail.gitee.io/2020/05/24/volatile%E5%8E%9F%E7%90%86%E5%88%86%E6%9E%90/)

然后观察一下AtomicInteger的成员方法:
```java
    /**
     * Gets the current value.
     */
    public final int get() {
        return value;
    }

    /**
     * Sets to the given value.
     */
    public final void set(int newValue) {
        value = newValue;
    }

    /**
     * Eventually sets to the given value.
     */
    public final void lazySet(int newValue) {
        unsafe.putOrderedInt(this, valueOffset, newValue);
    }

    /**
     * Atomically sets to the given value and returns the old value.
     */
    public final int getAndSet(int newValue) {
        return unsafe.getAndSetInt(this, valueOffset, newValue);
    }

    /**
     * Atomically sets the value to the given updated value
     * if the current value {@code ==} the expected value.
     */
    public final boolean compareAndSet(int expect, int update) {
        return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
    }
        /**
     * Atomically adds the given value to the current value.
     *
     * @param delta the value to add
     * @return the previous value
     */
    public final int getAndAdd(int delta) {
        return unsafe.getAndAddInt(this, valueOffset, delta);
    }

```
AtomicInteger的所有成员方法总结一下最终会调用到unsafe类的
```
unsafe.putOrderedInt(this, valueOffset, newValue);
```
以上 unsafe类的有序写入native方法，只保证写入有序性，不保证可见性，即一个线程写入这个值不保证其他线程立即可见。
```
unsafe.compareAndSwapInt(this, valueOffset, expect, update);//比较valueOffset偏移地址对应的值是否等于expect，等于则更新update到偏移地址
```
以上 unsafe类的cas native方法。


```java
unsafe.getAndAddInt(this, valueOffset, delta);//加delta到当前偏移地址的值上面去

    public final int getAndAddInt(Object var1, long var2, int var4) {
        int var5;
        do {
            var5 = this.getIntVolatile(var1, var2);
        } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));

        return var5;
    }
```
getAndAddInt方法不是unsafe类的native方法，采用循环的方式完成cas加法:
1. 先读取内存偏移地址var2的值var5
2. 然后使用cas方法，比较值var5与当前偏移地址var2里的值，相等的话就更新var5+var4
3. 否则循环重新读取内存偏移地址var2的值var5，执行1、2两步

图解:  
![cas原理图解](图1.jpeg)

因此compareAndSwapInt必须是原子性的操作，保证E与当前值N比较与更新V的操作不会被其他线程中断。

## hotspot层面CompareAndSwap源码

jdk8u: unsafe.cpp:

cmpxchg = compare and exchange

```c++
UNSAFE_ENTRY(jboolean, Unsafe_CompareAndSwapInt(JNIEnv *env, jobject unsafe, jobject obj, jlong offset, jint e, jint x))
  UnsafeWrapper("Unsafe_CompareAndSwapInt");
  oop p = JNIHandles::resolve(obj);
  jint* addr = (jint *) index_oop_from_field_offset_long(p, offset);
  return (jint)(Atomic::cmpxchg(x, addr, e)) == e;
UNSAFE_END
```

jdk8u: atomic_linux_x86.inline.hpp

is_MP = Multi Processor  

```c++
inline jint     Atomic::cmpxchg    (jint     exchange_value, volatile jint*     dest, jint     compare_value) {
  int mp = os::is_MP();
  __asm__ volatile (LOCK_IF_MP(%4) "cmpxchgl %1,(%3)"
                    : "=a" (exchange_value)
                    : "r" (exchange_value), "a" (compare_value), "r" (dest), "r" (mp)
                    : "cc", "memory");
  return exchange_value;
}
```

jdk8u: os.hpp is_MP()

```c++
  static inline bool is_MP() {
    // During bootstrap if _processor_count is not yet initialized
    // we claim to be MP as that is safest. If any platform has a
    // stub generator that might be triggered in this phase and for
    // which being declared MP when in fact not, is a problem - then
    // the bootstrap routine for the stub generator needs to check
    // the processor count directly and leave the bootstrap routine
    // in place until called after initialization has ocurred.
    return (_processor_count != 1) || AssumeMP;
  }
```

jdk8u: atomic_linux_x86.inline.hpp

```c++
#define LOCK_IF_MP(mp) "cmp $0, " #mp "; je 1f; lock; 1: "
```
LOCK_IF_MP(mp)宏定义 如果是多个cpu就加lock。

这段汇编代码

lock cmpxchgl 对cpu的读、比较、写操作加了锁，至于cpu加锁的方式有总线锁、高速缓存锁+缓存一致性协议，目前一般的cpu能支持缓存锁的都会使用缓存锁，不支持或者不能使用缓存锁时才会使用总线锁。

总线锁保证了某一时刻只有一个cpu核心能使用总线。

高速缓存锁+缓存一致性协议只得是cpu核心对自己得高速缓存进行管理，如果它读取得数据只位于一个缓存行中，就会将
