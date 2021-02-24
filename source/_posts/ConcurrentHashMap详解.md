---
title: ConcurrentHashMap详解
date: 2020-08-18 21:02:00
tags:
- juc
- java并发
---

# ConcurrentHashMap详解

# jdk1.7
数据结构由一个Segment数组构成，每个Segment数组元素是一个HashEntry数组，一个HashEntry节点就是一个链表的kv键值对节点。
Segment数组的大小由size来表示。  
```java
    int size =1;
    while(size < concurrencyLevel) {
    ++a;
    size <<=1;
    }      
```
size的大小为2的n次方，没有指定concurrencyLevel默认值为16，如果指定的话size最大值能到65536。

每个HashEntry的数组大小用cap来表示，也是2的n次方

```java

static class  Segment<K,V> extends  ReentrantLock implements  Serializable {
}
```

## put操作
从上Segment的继承体系可以看出，Segment实现了ReentrantLock,也就带有锁的功能，当执行put操作时，会进行第一次key的hash来定位Segment的位置，如果该Segment还没有初始化，即通过CAS操作进行赋值，然后进行第二次hash操作，找到相应的HashEntry的位置，这里会利用继承过来的锁的特性，在将数据插入指定的HashEntry位置时（链表的尾端），会通过继承ReentrantLock的tryLock（）方法尝试去获取锁，如果获取成功就直接插入相应的位置，如果已经有线程获取该Segment的锁，那当前线程会以自旋的方式去继续的调用tryLock（）方法去获取锁，超过指定次数就挂起，等待唤醒

## get操作
ConcurrentHashMap第一次需要经过一次hash定位到Segment的位置，然后再hash定位到指定的HashEntry，遍历该HashEntry下的链表进行对比，成功就返回，不成功就返回null

## resize操作

计算ConcurrentHashMap的元素大小是一个有趣的问题，因为他是并发操作的，就是在你计算size的时候，他还在并发的插入数据，可能会导致你计算出来的size和你实际的size有相差（在你return size的时候，插入了多个数据），要解决这个问题，JDK1.7版本用两种方案

- 第一种方案他会使用不加锁的模式去尝试多次计算ConcurrentHashMap的size，最多三次，比较前后两次计算的结果，结果一致就认为当前没有元素加入，计算的结果是准确的
- 第二种方案是如果第一种方案不符合，他就会给每个Segment加上锁，然后计算ConcurrentHashMap的size返回


# jdk1.8
摒弃了Segment的概念，直接用Node数组+链表+红黑树的数据结构来实现，并发控制使用Synchronized和CAS来操作，整个看起来就像是优化过且线程安全的HashMap，虽然在JDK1.8中还能看到Segment的数据结构，但是已经简化了属性，只是为了兼容旧版本。

## ConcurrentHashMap基本变量
```java
  // node数组最大容量：2^30=1073741824  

  private  static  final  int  MAXIMUM_CAPACITY =  1  <<  30    ;  

  // 默认初始值，必须是2的幂数  

  private  static  final  int  DEFAULT_CAPACITY =  16    ;  

  //数组可能最大值，需要与toArray（）相关方法关联  

  static  final  int  MAX_ARRAY_SIZE = Integer.MAX_VALUE -  8    ;  

  //并发级别，遗留下来的，为兼容以前的版本  

  private  static  final  int  DEFAULT_CONCURRENCY_LEVEL =  16    ;  

  // 负载因子  

  private  static  final  float  LOAD_FACTOR =  0    .75f;  

  // 链表转红黑树阀值,> 8 链表转换为红黑树  

  static  final  int  TREEIFY_THRESHOLD =  8    ;  

  //树转链表阀值，小于等于6（tranfer时，lc、hc=0两个计数器分别++记录原bin、新binTreeNode数量，<=UNTREEIFY_THRESHOLD 则untreeify(lo)）  

  static  final  int  UNTREEIFY_THRESHOLD =  6    ;  

  static  final  int  MIN_TREEIFY_CAPACITY =  64    ;  

  private  static  final  int  MIN_TRANSFER_STRIDE =  16    ;  

  private  static  int  RESIZE_STAMP_BITS =  16    ;  

  // 2^15-1，help resize的最大线程数  

  private  static  final  int  MAX_RESIZERS = (    1  << (    32  - RESIZE_STAMP_BITS)) -  1    ;  

  // 32-16=16，sizeCtl中记录size大小的偏移量  

  private  static  final  int  RESIZE_STAMP_SHIFT =  32  - RESIZE_STAMP_BITS;  

  // forwarding nodes的hash值 

  static  final  int  MOVED     = -    1    ;  

  // 树根节点的hash值  

  static  final  int  TREEBIN   = -    2    ;  

  // ReservationNode的hash值  

  static  final  int  RESERVED  = -    3    ;  

  // 可用处理器数量  

  static  final  int  NCPU = Runtime.getRuntime().availableProcessors();  

  //存放node的数组  

  transient  volatile  Node<K,V>[] table;  

  /*控制标识符，用来控制table的初始化和扩容的操作，不同的值有不同的含义  

  *当为负数时：-1 代表正在初始化，-N代表有 N-1 个线程正在 进行扩容  

  *当为    0    时：代表当时的table还没有被初始化  

  *当为正数时：表示初始化或者下一次进行扩容阈值的大小  
*/

  private  transient  volatile  int  sizeCtl;  

```

## Node
Node是ConcurrentHashMap存储结构的基本单元，继承于HashMap中的Entry，用于存储数据。
```java
    static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        volatile V val;
        volatile Node<K,V> next;

        Node(int hash, K key, V val, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.val = val;
            this.next = next;
        }

        public final K getKey()       { return key; }
        public final V getValue()     { return val; }
        public final int hashCode()   { return key.hashCode() ^ val.hashCode(); }
        public final String toString(){ return key + "=" + val; }
        public final V setValue(V value) {
            throw new UnsupportedOperationException();
        }

        public final boolean equals(Object o) {
            Object k, v, u; Map.Entry<?,?> e;
            return ((o instanceof Map.Entry) &&
                    (k = (e = (Map.Entry<?,?>)o).getKey()) != null &&
                    (v = e.getValue()) != null &&
                    (k == key || k.equals(key)) &&
                    (v == (u = val) || v.equals(u)));
        }

        /**
         * Virtualized support for map.get(); overridden in subclasses.
         */
        Node<K,V> find(int h, Object k) {
            Node<K,V> e = this;
            if (k != null) {
                do {
                    K ek;
                    if (e.hash == h &&
                        ((ek = e.key) == k || (ek != null && k.equals(ek))))
                        return e;
                } while ((e = e.next) != null);
            }
            return null;
        }
    }
```
Node数据结构很简单，从上可知，就是一个链表，但是只允许对数据进行查找，不允许进行修改。

## TreeNode
TreeNode继承与Node，但是数据结构换成了二叉树结构，它是红黑树的数据的存储结构，用于红黑树中存储数据，当链表的节点数大于8时会转换成红黑树的结构。
```java
static final class TreeNode<K,V> extends Node<K,V> {
        TreeNode<K,V> parent;  // red-black tree links
        TreeNode<K,V> left;
        TreeNode<K,V> right;
        TreeNode<K,V> prev;    // needed to unlink next upon deletion
        boolean red;

        TreeNode(int hash, K key, V val, Node<K,V> next,
                 TreeNode<K,V> parent) {
            super(hash, key, val, next);
            this.parent = parent;
        }

        Node<K,V> find(int h, Object k) {
            return findTreeNode(h, k, null);
        }

        /**
         * Returns the TreeNode (or null if not found) for the given key
         * starting at given root.
         */
        final TreeNode<K,V> findTreeNode(int h, Object k, Class<?> kc) {
            if (k != null) {
                TreeNode<K,V> p = this;
                do  {
                    int ph, dir; K pk; TreeNode<K,V> q;
                    TreeNode<K,V> pl = p.left, pr = p.right;
                    if ((ph = p.hash) > h)
                        p = pl;
                    else if (ph < h)
                        p = pr;
                    else if ((pk = p.key) == k || (pk != null && k.equals(pk)))
                        return p;
                    else if (pl == null)
                        p = pr;
                    else if (pr == null)
                        p = pl;
                    else if ((kc != null ||
                              (kc = comparableClassFor(k)) != null) &&
                             (dir = compareComparables(kc, k, pk)) != 0)
                        p = (dir < 0) ? pl : pr;
                    else if ((q = pr.findTreeNode(h, k, kc)) != null)
                        return q;
                    else
                        p = pl;
                } while (p != null);
            }
            return null;
        }
    }

```

## TreeBin
TreeBin从字面含义中可以理解为存储树形结构的容器，而树形结构就是指TreeNode，所以TreeBin就是封装TreeNode的容器，它提供转换黑红树的一些条件和锁的控制。

## 构造方法
ConcurrentHashMap构造方法中是参数的赋值。

## put操作

```java
public V put(K key, V value) {
        return putVal(key, value, false);
    }

    /** Implementation for put and putIfAbsent */
    final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();
        int hash = spread(key.hashCode());
        int binCount = 0;
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            //1、如果数组还没有初始化，先初始化表initTable
            if (tab == null || (n = tab.length) == 0)
                tab = initTable();
            //2、如果当前索引位置为null，cas操作更新为Node
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
            //3、f经过上面的判断，会取到i索引位置的Node节点，如果Node的hash表示正在扩容，该线程就帮忙进行扩
            else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);
            else {
                //4、如果找到对应的数组节点，就加锁来保证线程安全，这里有两种情况，一种是链表形式就直接遍历到尾端插入，一种是红黑树就按照红黑树结构插入
                V oldVal = null;
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        if (fh >= 0) {
                            binCount = 1;
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                //找到了节点
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                Node<K,V> pred = e;
                                //到链尾了，就链尾接一个新的
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
                        else if (f instanceof TreeBin) {
                            Node<K,V> p;
                            binCount = 2;
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                    }
                }
                //如果链表遍历到链尾数量大于等于了8个就得转成红黑树
                //但是，注意进到这方法里面还有一个判断就是如果table的长度小于64就执行扩容
                if (binCount != 0) {
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
        addCount(1L, binCount);
        return null;
    }
```
上述流程主要步骤

1. 首先是initTable，就是通过自旋改状态SIZECTL为-1，然后new数组出来
```java
private final Node<K,V>[] initTable() {
        Node<K,V>[] tab; int sc;
        while ((tab = table) == null || tab.length == 0) {
            if ((sc = sizeCtl) < 0)
                Thread.yield(); // lost initialization race; just spin
            else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                try {
                    if ((tab = table) == null || tab.length == 0) {
                        //sc大于0，说明sc是要初始化的表长的大小
                        int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = tab = nt;
                        //设置扩容阈值大小，这个操作相当于n*0.75
                        sc = n - (n >>> 2);
                    }
                } finally {
                    sizeCtl = sc;
                }
                break;
            }
        }
        return tab;
    }
```
2. 如果没有hash冲突就直接CAS插入，首先volatile读最新的数组位置节点是不是空，为空就cas加入新的Node
```java
    static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
        return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
    }

    static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                        Node<K,V> c, Node<K,V> v) {
        return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
    }
```

3. f经过上面的判断，会取到i索引位置的Node节点，如果Node的hash表示我正在扩容，该线程就帮忙进行扩

```java
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
        Node<K,V>[] nextTab; int sc;
        if (tab != null && (f instanceof ForwardingNode) &&
            (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
            int rs = resizeStamp(tab.length);
            while (nextTab == nextTable && table == tab &&
                   (sc = sizeCtl) < 0) {
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || transferIndex <= 0)
                    break;
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                    transfer(tab, nextTab);
                    break;
                }
            }
            return nextTab;
        }
        return table;
    }
    static final int resizeStamp(int n) {
        return Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1));
    }

```
扩容操作稍后详谈！！

4. 如果存在hash冲突，就加锁来保证线程安全，这里有两种情况，一种是链表形式就直接遍历到尾端插入，一种是红黑树就按照红黑树结构插入。

5. 如果链表遍历到链尾数量大于等于了8个就得这一条链转成红黑树。

6. 如果添加成功就调用addCount()方法统计size，并且检查是否需要扩容。

```java
// 从 putVal 传入的参数是 (1， binCount)，binCount 默认是0，只有 hash 冲突了才会大于 1.且他的大小是链表的长度（如果不是红黑数结构的话）。
private final void addCount(long x, int check) {
        CounterCell[] as; long b, s;
        //这一块儿if是计数
        //如果计数盒子不为空 或者 CAS使用baseCount来计算，如果有争用计数失败了就进if语句块
        if ((as = counterCells) != null ||
            !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
            CounterCell a; long v; int m;
            boolean uncontended = true;
            //如果计数盒子是空 或者 计数盒子不为空，但是取一个计数盒子内随机索引的值为空 或者 计数盒子取一个随机索引不为空，但是cas向这个盒子的这个索引位置写+x操作失败了 就进入if语句，执行fullAddCount 并结束。
            if (as == null || (m = as.length - 1) < 0 ||
                (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
                !(uncontended =
                  U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
                //fullAddCount包含了CounterCell初始化，扩容，计数累加的功能
                fullAddCount(x, uncontended);
                return;
            }
            if (check <= 1)
                return;
            s = sumCount();//统计计数盒子里的所有值也就是baseCount+counterCells里的所有值，即目前map里元素数量有多少，size()
        }
        //这一块儿if是扩容操作
        //check大于0说明bitCount是大于等于0的说明，在put的时候bitCount默认就是大于等于0的
        if (check >= 0) {
            Node<K,V>[] tab, nt; int n, sc;
            //这里开始进行扩容操作，扩容也支持多个线程同时扩，使用SIZECTL记录参与扩容的线程数量。
            //SIZECTL的高16位 是扩容戳，扩容戳计算和当前数组长度有关，resizeStamp
            //SIZECTL的低16位，记录参与扩容的线程数量，线程数量等于低16位-1
            //当map.size()大于等于sizeCtl(表示要进行扩容的阈值) 且 table数组不为空 且 数组长度小于最大容积 
            //当进入这循环的时候如果开启了扩容，这个sizeCtl就一直是复数，想要退出循环，必须等到扩容结束。
            while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
                   (n = tab.length) < MAXIMUM_CAPACITY) {
                /*
                    static final int resizeStamp(int n) {
                        return Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1));
                    }
                    Integer.numberOfLeadingZeros(n)这个方法返回n的二进制形式下最高位到第一个非0位中间0的数量，这个得到的数的范围一定是0~32
                    然后与 1<<15 相或，1左移15位就是在第16位上面，意思就是第16位一定是1，为了后面将其左移16位变为负数。那为什么随便取一个0~32的数就可以用呢？因为后面记录扩容线程数会先 左移16位，除了保证是一个负数以外，低16位也全部变成了0用来记录扩容线程数，至于高16位是什么数不关心，也没有运算会修改它，当作一个本次扩容期间不会变化的标识即可。
                */
                int rs = resizeStamp(n);
                //第一个进来的线程一定不小于0，
                //sc小于0说明正在扩容
                if (sc < 0) {
                    //int RESIZE_STAMP_BITS = 16;
                    //int RESIZE_STAMP_SHIFT = 32 - RESIZE_STAMP_BITS;
                    //sc右移16位应该与rs标识符相等才对，总之就是这些满足其一就跳出不再进入扩容
                    if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                        sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                        transferIndex <= 0)
                        break;
                    //sizeCtl现在记录的就是扩容线程数，不过只能看其低16位，sc+1也只影响低16位，高16位是跟扩容前数组长度有关的扩容戳标识符，标识此次扩容，不会变的。CAS操作+1 没什么问题 
                    if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                        transfer(tab, nt);
                }
                //第一次进来的线程到这儿，如果sizeCtl没被其他其他线程更改，
                else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                             (rs << RESIZE_STAMP_SHIFT) + 2))
                    transfer(tab, null);
                s = sumCount();
            }
        }
    }
```
这个方法从名字上来看是对哈希表得元素计数的。  
**计数操作整体思路:**  
如果我们只使用一个Long类型进行扩容。那势必在扩容的时候我们需要对计数器的++操作需要加锁。如果我们使用CAS避免锁操作，但是在多线程的情况下，也只有一个线程能CAS成功，其他线程会浪费CPU的资源。所以为了提高计数效率，ConcurrentHashMap就采用数组进行计数，将多个线程的++操作散列到数组的不同位置，这样可以有效的缓解线程的竞争，提升技术效率。

当然，如果我们虽然使用了ConcurrentHashMap，但是不存在多线程竞争计数的情况，次数就没必要使用数组技术，可以直接使用一个Long类型即可。

所以ConcurrentHashMap的计数就分为两部分，一个是baseCount，在不存在线程竞争的情况下使用。如果有多线程竞争计数，则会创建一个CounterCells数组来计数。最终中的数据 = baseCount + 数组数量总和。

然后我们分析一下fullAddCount(x, uncontended)是如何初始化counterCells的
```java
    private final void fullAddCount(long x, boolean wasUncontended) {
        int h;
        //从ThreadLocalRandom中获取随机数
        if ((h = ThreadLocalRandom.getProbe()) == 0) {
        	//如果发现获得的probe值为0，说明ThreadLocalRandom还没有被初始化，需要先进行初始化。
            ThreadLocalRandom.localInit();      // force initialization
            //初始化后，再重新获取一个随机数
            h = ThreadLocalRandom.getProbe();
            wasUncontended = true;
        }
        boolean collide = false;                // True if last slot nonempty
        for (;;) {
            CounterCell[] as; CounterCell a; int n; long v;
            //如果counterCells为null，说明counterCells还没初始化，所以需要对counterCells进行初始化
            if ((as = counterCells) != null && (n = as.length) > 0) {
                //如果counterCells不为null，就获取counterCells相应位置的counterCell
                //如果定位到的counterCell为null，说明counterCells相应位置的counterCell还是null，需要new一个counterCell
                if ((a = as[(n - 1) & h]) == null) {
                	//执行到这儿，说明定位到的位置CounterCell为null，需要给相应位置new一个CounterCell，并赋值。
                	//只有cellsBusy为0的情况下，才允许进行CountCells对应数组位置的CountCell初始化操作，因为此时可能正在扩容
                    if (cellsBusy == 0) {            // Try to attach new Cell
                    	//创建一个CounterCell
                        CounterCell r = new CounterCell(x); // Optimistic create
                        //进行CountCells对应数组位置的CountCell初始化操作，也不允许扩容操作
                        if (cellsBusy == 0 &&
                            U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                            boolean created = false;
                            try {               // Recheck under lock
                                CounterCell[] rs; int m, j;
                                //给数组指定位置赋值
                                if ((rs = counterCells) != null &&
                                    (m = rs.length) > 0 &&
                                    rs[j = (m - 1) & h] == null) {
                                    rs[j] = r;
                                    created = true;
                                }
                            } finally {
                                cellsBusy = 0;
                            }
                            if (created)
                                break;
                            continue;           // Slot is now non-empty
                        }
                    }
                    collide = false;
                }
                else if (!wasUncontended)       // CAS already known to fail
                    wasUncontended = true;      // Continue after rehash
                //执行到这儿，说明定位到的counterCell一定不为null
                //通过CAS对count进行累加。
                //如果线程的数量很多，即使散列但是依旧存在多线程竞争的情况，这里依旧会失败。
                //如果依旧失败，则会去尝试扩容
                else if (U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))
                    break;
                //这里的判断，就是为了看是否还能扩容
                //如果counterCells != as，说明肯定已经有其他线程已经扩容完毕了，因为扩容会创建新的数组，那就不需要再进行扩容了
                //n >= NCPU，如果当前counterCells 数组的长度大于等于CPU的核数，也不能再继续扩容。
                else if (counterCells != as || n >= NCPU)
                    collide = false;            // At max size or stale
                else if (!collide)
                    collide = true;
               	//执行到这儿，说明散列后还存在竞争，并且还能够继续扩容
               	//通过CAS设置cellsBusy，保证只能有一个线程扩容，并且扩容期间不允许进行CountCells对应数组位置的CountCell初始化操作
                else if (cellsBusy == 0 &&
                         U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                    try {
                        if (counterCells == as) {// Expand table unless stale
                        //counterCells 的长度扩大为原来的两倍，
                        //将counterCell移到新的数组中，
                        //并用新的counterCells替换旧的counterCells 
                            CounterCell[] rs = new CounterCell[n << 1];
                            for (int i = 0; i < n; ++i)
                                rs[i] = as[i];
                            counterCells = rs;
                        }
                    } finally {
                        cellsBusy = 0;
                    }
                    collide = false;
                    continue;                   // Retry with expanded table
                }
                h = ThreadLocalRandom.advanceProbe(h);
            }
            //对counterCells进行初始化，因为存在多线程，所以通过CAS设置状态值cellsBusy来保证只有一个线程能够初始化成功
            else if (cellsBusy == 0 && counterCells == as &&
                     U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                boolean init = false;
                try {                           // Initialize table
                    if (counterCells == as) {
                    	//初始化的过程，就是创建一个长度为2的CounterCell数组
                        CounterCell[] rs = new CounterCell[2];
                        //因为是第一初始化，直接给数组的位置创建一个CounterCell并赋值即可。
                        rs[h & 1] = new CounterCell(x);
                        counterCells = rs;
                        init = true;
                    }
                } finally {
                	//退出的时候恢复cellsBusy 
                    cellsBusy = 0;
                }
                if (init)
                    break;
            }
            else if (U.compareAndSwapLong(this, BASECOUNT, v = baseCount, v + x))
                break;                          // Fall back on using base
        }
    }
```
- 一开始，就先获取一个随机数，用于后面的散列操作。fullAddCount计数因为没有hash进行散列，所以他通过ThreadLocalRandom获取随机数的方式来散列实现多线程计数，对ThreadLocalRandom不太熟悉的，可以看 [Random和ThreadLocalRandom原理分析](https://blog.csdn.net/b_x_p/article/details/105237672)

- 如果counterCells为空，就会对其初始化，初始化之前通过CAS将cellsBusy设置为1，保证只有一个线程能进行初始化。counterCells的初始化长度为2.

- 如果counterCells不为null，则开始计数。如果拿到的counterCells数组对应位置的counterCell为null，则需要初始化当前位置的counterCell。如果不为null，则直接通过CAS进行计数。

- 当在用数组进行计数还是存在竞争导致CAS失败的情况下，判读当前的counterCells数组长度是否大于等于当前操作系统的CPU核数，如果不大于，则进行扩容，如果已经大于等于了，则只能通过不断的CAS完成最终的计数。

- addCount结束后，会通过sumCount计算总的count数量，用于后面的扩容判断，sunCount实现如下，就是一个累加的操作。


## transfer操作 扩容
最后来聊一聊transfer扩容操作。

ConcurrentHashMap支持多线程同时扩容，其实描述为支持多个线程同时迁移节点数据到扩容后的新的tab中更为准确。所以可想而知，为了实现多线程迁移数据，必须划定好迁移范围，每个线程迁移一定范围的数据。

```java
//参数中：tab是原tab，nextTab是扩容后新的tab
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    int n = tab.length, stride;
    //扩容过程由于tab长度变化，导致数据在tab中的位置也会发生变化，所以需要转移数据。
    //转移数据支持多个线程共同转移，所以要划分好每个线程转移那一部分。
    //stride就是用于定义每个线程转移tab的长度。每个线程处理的tab数量
    //这里，如果CPU核数为1，则stride = n， 则只一个线程进行处理。
    //如果CPU核数大于1，则将数组分为 （CPU数 * 8）段。允许（CPU数 * 8）个线程处理。
    //但是，最小的线程处理单位长度为16
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE; // subdivide range
    if (nextTab == null) {            // initiating
        try {
            @SuppressWarnings("unchecked")
            //如果是第一个线程来扩容，则创建一个新的tab，长度为原tab的2倍
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
            nextTab = nt;
        } catch (Throwable ex) {      // try to cope with OOME
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        nextTable = nextTab;
        //这里给transferIndex 赋值为原tab的长度lenth
        //下面就会通过这个transferIndex分配每个线程的处理长度
        //后面分配处理的时候，用bound 和i分别指向范围的左端和右端
        //i = transferIndex - 1; bound = transferIndex - stride; transferIndex - bound
        //当然，这里也只有第一个来扩容的线程才会进入到这里进行赋值
        transferIndex = n;
    }
    int nextn = nextTab.length;
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
    boolean advance = true;
    boolean finishing = false; // to ensure sweep before committing nextTab
   	//这里for循环里面，就是真正的迁移。
   	//给每个线程分配处理的范围，范围大小为前面计算的stride 。
   	//用bound 和i分别指向范围的左端和右端，用全局变量transferIndex来控制每个线程处理范围的不重复。
   	//分配好范围后，通过--i来遍历处理每一个Node，如果当前节点Node处理完成，将其置为fwd
    for (int i = 0, bound = 0;;) {
        Node<K,V> f; int fh;
        //这里的while循环：线程第一次来的时候，用于分配处理范围。分配好之后，通过--i实现遍历处理
        //advance：开始处理当前的tab[i]的时候，advance会置为false
        //         当前的tab[i]处理完成的时候，advance会置为true
        while (advance) {
            int nextIndex, nextBound;
            //这里需要关注一下，--i就是用于遍历使用的。
            //如果线程第一次来，还没有分配范围，此时i = bound = 0，所以条件不满足，这里走else
            //非第一次来的话，bound < i，i移动到下一个位置
            if (--i >= bound || finishing)
                advance = false;
            //如果这个if满足条件，说明，tab已经全部分配完了。
            else if ((nextIndex = transferIndex) <= 0) {
                i = -1;
                advance = false;
            }
            //执行到这里，说明当前线程第一次进入，还没有分配长度，这里就会就行长度分配
            //首先因为是多个线程同时进入，所以通过CAS修改transferIndex。修改成功，说明当前线程获得了此范围的转移权
            //用bound 和i分别指向范围的左端和右端
            else if (U.compareAndSwapInt
                     (this, TRANSFERINDEX, nextIndex,
                      nextBound = (nextIndex > stride ?
                                   nextIndex - stride : 0))) {
                bound = nextBound;
                i = nextIndex - 1;
                advance = false;
            }
        }
        //在前面的while中分配处理范围的时候，如果已经分配完了，则 i = -1
       	//所以这里，只有分配到的最后一个线程才会执行到这里
        if (i < 0 || i >= n || i + n >= nextn) {
            int sc;
            //finishing为true，说明已经全部转移完成了，则重新为sizeCtl 何 tab赋值。
            if (finishing) {
                nextTable = null;
                table = nextTab;
                sizeCtl = (n << 1) - (n >>> 1);
                return;
            }
            //每个参与扩容的线程，如果扩容结束，就将参与扩容的线程减一
            //只有当所有的参与扩容的线程全部执行结束，才会将finishing 设置为true
            if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                //如果当前线程减完自己这个线程数量以后 得到的线程数不等于0，说明还有其他线程在跑，当前线程就直接return即可
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    return;
                //上面的if失败，说明现在的线程是最后一个线程
                finishing = advance = true;
                i = n; // recheck before commit 最后一个线程，你单独将整个桶扫一遍确保，每一个桶都标上了fwd
            }
        }
        else if ((f = tabAt(tab, i)) == null)//原数组在i这个位置是不可能为空的，这里主要做赋值操作
            advance = casTabAt(tab, i, null, fwd);
        else if ((fh = f.hash) == MOVED)//已经有线程在移动节点了，那当前线程分配的区间作废
            advance = true; // already processed
        else {
        	//这里是转移每个槽位的逻辑，需要加锁
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    Node<K,V> ln, hn;
                    //说明此时还是链表
                    if (fh >= 0) {
                        //第一步是找最后可以复用的一段链，最高位是0还是1判断是高位链还是低位链，lastRun就是最后一段相同链的头节点
                        int runBit = fh & n;
                        Node<K,V> lastRun = f;
                        for (Node<K,V> p = f.next; p != null; p = p.next) {
                            int b = p.hash & n;
                            if (b != runBit) {
                                runBit = b;
                                lastRun = p;
                            }
                        }
                        //第二步是判断最后一段链是在高位还是在低位，链过去
                        if (runBit == 0) {
                            ln = lastRun;
                            hn = null;
                        }
                        else {
                            hn = lastRun;
                            ln = null;
                        }
                        //第三步，遍历链，直到lastRun节点
                        //这里是完全创建了新的节点，而非继续使用原节点
                        //这样做，保证了原tab不变，所以在扩容期间，依旧可以进行get。
                        for (Node<K,V> p = f; p != lastRun; p = p.next) {
                            int ph = p.hash; K pk = p.key; V pv = p.val;
                            //判断节点是高位还是低位，然后new一个新的，头插
                            if ((ph & n) == 0)
                                ln = new Node<K,V>(ph, pk, pv, ln);
                            else
                                hn = new Node<K,V>(ph, pk, pv, hn);
                        }
                        //将低位链设置到原索引，这个set就是unsafe的put操作
                        setTabAt(nextTab, i, ln);
                        //将高位链设置到原索引+n处
                        setTabAt(nextTab, i + n, hn);
                        //这一条链跑完了，就将原链表i索引的节点设成fwd节点，这个节点
                        setTabAt(tab, i, fwd);
                        //这一块区间分配完了，将advanced置true，即又可以进入while(advanced)循环去领区间任务了
                        advance = true;
                    }
                    //说明已经是树结构，树结构不分析。
                    else if (f instanceof TreeBin) {
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> lo = null, loTail = null;
                        TreeNode<K,V> hi = null, hiTail = null;
                        int lc = 0, hc = 0;
                        for (Node<K,V> e = t.first; e != null; e = e.next) {
                            int h = e.hash;
                            TreeNode<K,V> p = new TreeNode<K,V>
                                (h, e.key, e.val, null, null);
                            if ((h & n) == 0) {
                                if ((p.prev = loTail) == null)
                                    lo = p;
                                else
                                    loTail.next = p;
                                loTail = p;
                                ++lc;
                            }
                            else {
                                if ((p.prev = hiTail) == null)
                                    hi = p;
                                else
                                    hiTail.next = p;
                                hiTail = p;
                                ++hc;
                            }
                        }
                        ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                            (hc != 0) ? new TreeBin<K,V>(lo) : t;
                        hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                            (lc != 0) ? new TreeBin<K,V>(hi) : t;
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                }
            }
        }
    }
}
```
以上就是transfer方法的解析。

> 问：什么情况下get/put会被阻塞?  
get：文中已经分析过，任何情况下都可以进行get操作。因为put的时候属于尾插，不影响get。而扩容转移链表数据的时候会创建新的Node，所以扩容过程中也不影响原链表。
>
>put：put操作的时候会对tab[ i ]进行synchronized加锁，在对tab[i]进行put的时候，会阻塞tab[ i ]上的其他put操作，但是不影响tab[ j ]的put操作。
在扩容过程中迁移的时候，也会对tab[ i ]进行synchronized加锁，所以迁移过程中，也会阻塞当前正在迁移的tab[ i ]的put操作。


