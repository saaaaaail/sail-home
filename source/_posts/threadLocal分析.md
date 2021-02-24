---
title: threadLocal分析
date: 2020-05-14 10:44:04
tags:
---

# ThreadLocal分析 jdk1.8

## ThreadLocal的get、set方法

```
    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
```
```
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
```
```
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
```
```
   ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
```
以上可见 会根据当前线程对象获得线程内置的ThreadLocalMap属性。

在分析ThreadLocalMap之前先关注ThreadLocal的关键属性。

## ThreadLocal基本属性

```
    private final int threadLocalHashCode = nextHashCode();

    private static AtomicInteger nextHashCode =
        new AtomicInteger();

    private static final int HASH_INCREMENT = 0x61c88647;

    private static int nextHashCode() {
        return nextHashCode.getAndAdd(HASH_INCREMENT);
    }
```
 其中threadLocalHashCode表示当前ThreadLocal对象在对应线程的ThreadLocalMap里的位置。

 每次都通过
 ```
 nextHashCode()
 ```
 方法获得下一个ThreadLocal对象的偏移量的地址，之所以确认HASH_INCREMENT的值为0x61c88647是应用了某种算法，暂不考虑，保证分布均匀。

 ## ThreadLocalMap分析

 ### ThreadLocalMap基本属性
 
 ```
        static class Entry extends WeakReference<ThreadLocal<?>> 
        {
            Object value;

            Entry(ThreadLocal<?> k, Object v) 
            { 
                super(k);
                value = v;
            }
        }

        private static final int INITIAL_CAPACITY = 16;

        private Entry[] table;

        private int size = 0;

        private int threshold; // Default to 0
        private void setThreshold(int len) {
            threshold = len * 2 / 3;
        }
 ```
 包括内置的实现了WeakReference弱引用的Entry静态对象，初始化的默认容量16，entry数组，thresholld为重hash阈值，此处为总容量的2/3。

 **这里注意构造方法**，传进来的ThreadLocal对象，传到了WeakReference中将这条指向threadLocal的引用封装成了弱引用，那么这个key作为对象的引用在外部的强引用置空以后，这个弱引用就只能活到下一次GC，但是在ThreadLocalMap中仍然会残留一个null，value的键值对。造成内存泄漏，因此当一个对象不使用了以后，要主动调用threadLocal.remove()方法删除这条记录。

 ### ThreadLocalMap构造方法

 ```
        ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
            table = new Entry[INITIAL_CAPACITY];
            int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
            table[i] = new Entry(firstKey, firstValue);
            size = 1;
            setThreshold(INITIAL_CAPACITY);
        }
 ```
 做了这么几件事：
 1. 初始entry数组
 2. hashCode与2次幂相与求余数作为table里的索引
 3. new一个Entry保存key与value，key是threadLocal对象，value是值
 4. size=1
 5. 设置重hash阈值

```
        private ThreadLocalMap(ThreadLocalMap parentMap) {
            Entry[] parentTable = parentMap.table;
            int len = parentTable.length;
            setThreshold(len);
            table = new Entry[len];

            for (int j = 0; j < len; j++) {
                Entry e = parentTable[j];
                if (e != null) {
                    @SuppressWarnings("unchecked")
                    ThreadLocal<Object> key = (ThreadLocal<Object>) e.get();
                    if (key != null) {
                        Object value = key.childValue(e.value);
                        Entry c = new Entry(key, value);
                        int h = key.threadLocalHashCode & (len - 1);
                        while (table[h] != null)
                            h = nextIndex(h, len);
                        table[h] = c;
                        size++;
                    }
                }
            }
        }
```
第二个构造方法，用于把父线程的entry数组拷贝到当前线程中。

### ThreadLocalMap重要方法

```
        private Entry getEntry(ThreadLocal<?> key) {
            int i = key.threadLocalHashCode & (table.length - 1);
            Entry e = table[i];
            if (e != null && e.get() == key)
                return e;
            else
                return getEntryAfterMiss(key, i, e);
        }
```
根据ThreadLocal对象获得Entry。

```
        private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
            Entry[] tab = table;
            int len = tab.length;

            while (e != null) {
                ThreadLocal<?> k = e.get();
                if (k == key)
                    return e;
                if (k == null)
                    expungeStaleEntry(i);
                else
                    i = nextIndex(i, len);
                e = tab[i];
            }
            return null;
        }
```
如果Entry不存在或者entry的key不等于当前threadLocal对象就调用这个方法。

如果entry的key不等于当前threadLocal对象，则从i往后遍历entry数组直到找到相等的entry返回，若碰到k==null调用expungeStaleEntry
```
        private int expungeStaleEntry(int staleSlot) {
            Entry[] tab = table;
            int len = tab.length;

            // expunge entry at staleSlot
            tab[staleSlot].value = null;
            tab[staleSlot] = null;
            size--;

            // Rehash until we encounter null
            Entry e;
            int i;
            for (i = nextIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = nextIndex(i, len)) {
                ThreadLocal<?> k = e.get();
                if (k == null) {
                    e.value = null;
                    tab[i] = null;
                    size--;
                } else {
                    int h = k.threadLocalHashCode & (len - 1);
                    if (h != i) {
                        tab[i] = null;

                        // Unlike Knuth 6.4 Algorithm R, we must scan until
                        // null because multiple entries could have been stale.
                        while (tab[h] != null)
                            h = nextIndex(h, len);
                        tab[h] = e;
                    }
                }
            }
            return i;
        }
```
expungeStaleEntry(int staleSlot)方法将staleSlot索引的entry从table里面清理掉，范围是从staleSlot索引位置往后到下一个entry为空的i，返回i，然后rehash将entry数组重新定位索引，有重复的索引就线性探测法往后找空的。之所以要循环判断k==null并清理，是因为entry为弱引用，活到下一次GC就回收了很容易为空。

```
        private void set(ThreadLocal<?> key, Object value) {

            // We don't use a fast path as with get() because it is at
            // least as common to use set() to create new entries as
            // it is to replace existing ones, in which case, a fast
            // path would fail more often than not.

            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);

            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                ThreadLocal<?> k = e.get();

                if (k == key) {
                    e.value = value;
                    return;
                }

                if (k == null) {
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }

            tab[i] = new Entry(key, value);
            int sz = ++size;
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }
```
set方法 hash值与len-1相与获得索引值，从i往后遍历entry数组
- 期间碰到key相同的就意味着找到了，更新value返回。
- 如果entry的key为空调用replaceStaleEntry(key, value, i)方法返回

遍历结束找到一个索引i对应的entry为空的位置，保存新entry，更新size，然后不满足cleanSomeSlots(i, sz)方法（表示从i往后的一个范围里没有过时的entry，返回false）且size比重hash阈值要大就rehash

```
        private void replaceStaleEntry(ThreadLocal<?> key, Object value,int staleSlot) {
            Entry[] tab = table;
            int len = tab.length;
            Entry e;

            int slotToExpunge = staleSlot;
            /**
            * 往前遍历找到staleSlot索引最前边儿key是空的entry的索引i，记录到slotToExpunge里面
            */
            for (int i = prevIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = prevIndex(i, len))
                if (e.get() == null)
                    slotToExpunge = i;

            /**
            * 从staleSlot往后遍历i
            */
            for (int i = nextIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = nextIndex(i, len)) {
                ThreadLocal<?> k = e.get();

                /**
                * 找到一个key相等的entry则意味着命中了，更新entry的value，然后做一个值的交换，把tab[staleSlot]的entry赋给tab[i]，然后把i的entry赋给tab[staleSlot]，staleSlot就是当前threadLocal的索引
                */
                if (k == key) {
                    e.value = value;

                    tab[i] = tab[staleSlot];
                    tab[staleSlot] = e;

                    /**
                    * 如果此时仍然没有找到一个key为空的entry，则更新slotToExpunge为当前索引i
                    */
                    if (slotToExpunge == staleSlot){
                        slotToExpunge = i;
                    }
                    /**
                    * 从slotToExpunge位置开始往后遍历清理过时数据，然后cleanSomeSlots方法，这个方法
                    */
                    cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
                    return;
                }

                /**
                * 这是往后找到key为空，并且要slotToExpunge == staleSlot（相等说明staleSlot往前遍历时前面的索引没有key空的entry存在，则更新一次slotToExpunge为staleSlot往后最近的key为空的索引，这么做是为了保证这个索引是整个table里最靠前的key为空的索引，之后会从它开始清理过时的entry）
                */
                if (k == null && slotToExpunge == staleSlot)
                    slotToExpunge = i;
            }
            
            /**
            * 遍历完一遍没有找到entry不为空且key相等的索引位置，就直接赋值就好了
            */
            tab[staleSlot].value = null;
            tab[staleSlot] = new Entry(key, value);

            /**
            * 这里不等说明找到了entry不为空且key为空的索引位置
            */
            if (slotToExpunge != staleSlot){
                cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
            }
        }
```
replaceStaleEntry(ThreadLocal<?> key, Object value,int staleSlot)方法用于清洗map里的过期数据，即entry不为空但是k为空的数据。


```
        private boolean cleanSomeSlots(int i, int n) {
            boolean removed = false;
            Entry[] tab = table;
            int len = tab.length;
            do {
                i = nextIndex(i, len);
                Entry e = tab[i];
                if (e != null && e.get() == null) {
                    n = len;
                    removed = true;
                    i = expungeStaleEntry(i);
                }
            } while ( (n >>>= 1) != 0);
            return removed;
        }
```
cleanSomeSlots(int i, int n)方法主要从i往后寻找entry不为空且key为空的entry，调用expungeStaleEntry(i)方法清理掉这些过时的entry，n的作用是每次除以2，起到控制循环次数的作用。如果在n变为0的次数里面没有过时的entry，则返回false。

```
        private void rehash() {
            expungeStaleEntries();

            // Use lower threshold for doubling to avoid hysteresis
            if (size >= threshold - threshold / 4)
                resize();
        }
```
rehash()方法:
- 调用expungeStaleEntries()方法从0到len清理过时的entry
- 如果size大于rehash阈值的3/4就扩容


```
        private void resize() {
            Entry[] oldTab = table;
            int oldLen = oldTab.length;
            int newLen = oldLen * 2;
            Entry[] newTab = new Entry[newLen];
            int count = 0;

            for (int j = 0; j < oldLen; ++j) {
                Entry e = oldTab[j];
                if (e != null) {
                    ThreadLocal<?> k = e.get();
                    if (k == null) {
                        e.value = null; // Help the GC
                    } else {
                        int h = k.threadLocalHashCode & (newLen - 1);
                        while (newTab[h] != null)
                            h = nextIndex(h, newLen);
                        newTab[h] = e;
                        count++;
                    }
                }
            }

            setThreshold(newLen);
            size = count;
            table = newTab;
        }
```
resize()方法为扩容方法，新创建一个容量2倍的entry数组，遍历旧entry数组，同时使用线性探测法解决冲突，将旧entry保存到新entry数组，更新重hash阈值、size、table。