---
title: volatile原理分析
date: 2020-05-24 21:37:58
tags:
- volatile
- spring
---

# volatile原理详解
## volatile基础含义:
1. 可见性 （一个线程对数据进行的修改对其他线程都是可见的） 即指的是被volatile修饰的变量在读取数据的时候呢，会从主内存读到各级缓存然后读到cpu里面，写的时候呢除了写到三级缓存里面，还会直接写到主内存里面，这样其他cpu的缓存如果保存了这条修改的缓存行通过总线嗅探到缓存值失效了就从主存重新读一下，保证数据可见性。
2. 禁止指令重排序 （禁止jvm为了优化字节码指令而对打乱指令的执行顺序）（dcl单例是否需要加volatile）

## 补充基础知识:
一台计算机的组成包括一个cpu、内存、磁盘、其他外设、驱动，cpu与内存之间进行数据交换，内存与硬盘进行数据交换。
由于cpu与内存的速度相差100倍，为了提高数据交互效率，在cpu与内存之间添加了三级缓存。
多核cpu与内存结构如下:

![多核cpu结构](图1.png)

上图主板上两个cpu，每个cpu有两个核，每个核内有独立的L1、L2缓存，每个cpu的核都共享主板上的属于自己cpu的L3缓存，所有的cpu共享主内存。
一般来讲，一个cpu核只能只能被一个线程占用（所谓线程是cpu调度的基本单位，线程会占用cpu的寄存器，切换线程会保存现场），但是支持超线程的cpu中（例:四核八线程）有两套寄存器，可以同时跑两个线程，切换只是alu的切换。
JMM模型的主内存与线程工作内存的真面目就是主内存与缓存。

### 缓存行 cache line 缓存一致性协议 与volatile无关的
指从内存中读取数据一次读取的是一个数据块，其大小是64Byte。 
如果数据X和数据Y位于同一缓存行，这个缓存行同时被cpu核A和cpu核B读取，每次cpu核A的修改数据X到缓存L1中并未立即将X写入RAM，cpu核B读取的缓存行中包括数据X，以及其他所有cpu核发现自己的高速缓存包含这个数据的缓存行就会收到通知，使自身的缓存行失效，然后等cpu核A将数据写到RAM中后重新从RAM中读取缓存行到高速缓存里。

所以当cpu核A对数据X写，cpu核B对数据Y写，即便不是同一个数据由于存在于同一缓存行，每次修改自身数据到缓存都会使其他缓存行失效，并重新从内存读取，效率会下降。

缓存行会依次从主内存->L3->L2->L1。


## volatile的五层实现:
1. java源码
2. ByteCode字节码
3. JVM虚拟机规范
4. Hotspot实现
5. CPU级别


### java源码
volatile int i;

### ByteCode字节码
Access flags: Acc_volatile  
在字节码中为这个i添加了一个标记，标记为Acc_volatile

### JVM虚拟机规范
对于volatile的基础含义，JVM设计了一部分规范来达成这两条基础含义。

首先是规定了八种happens-before原则（JVM规定对指令重排序要遵守的原则）
为了禁止指令重排序的发生，在两条指令之间添加一条fence或者barriar，内存屏障。

JVM层级的有四条逻辑层面的内存屏障，即要求所有的jvm都实现的规范:
- LoadLoad屏障：对于连续的读操作，保证屏障前面的读操作一定在屏障后面的读操作前完成。
- StoreStore屏障：对于连续的写操作，保证屏障前面的写操作一定在屏障后面的写操作前完成。
- LoadStore屏障：对于连续读写操作，保证屏障前面的读操作一定在屏障后面的写操作前完成。
- StoreLoad屏障：对于连续的写读操作，保证屏障前面的写操作一定在屏障后面的读操作前完成。

那么当使用volatile修饰一个变量时，对这个变量所占的内存进行读写前后都会加上内存屏障。

---
- StoreStoreBarrier
- volatile写
- StoreLoadBarrier
---
- volatile读
- LoadLoadBarrier
- LoadStoreBarrier
---

在volatile变量读写前后会设置内存屏障。
StoreStoreBarrier屏障保证屏障之前volatile写操作一定完成于屏障之后的volatile写操作之前
StoreLoadBarrier屏障保证屏障之前的volatile写操作一定完成于volatile读操作之前
LoadLoadBarrier屏障保证屏障之前的volatile读操作一定完成于屏障之后的volatile读操作之前
LoadStoreBarrier屏障保证屏障之前的volatile读操作一定发生于屏障之后的volatile写操作之前

以上是JVM层面对volatile实现的要求规范，为了实现volatile的基本含义。

### Hotspot实现
为了实现上述的内存屏障，hotspot在将class文件加载到内存中以后会翻译成字节码指令并编译为当前机器cpu的机器语言，为了查看这种机器语言，可以使用HSDIS工具反汇编字节码生成的机器码为汇编代码。

```
public class Test{
    public static volatile int i = 0;

    public static void main(String[] args){
        for(int i=0;i<1000000;i++){
            m();
            n();
        }
    }

    public static synchronized void m(){

    }

    public static void n(){
        i=1;
    }
}
```

```
java -XX:+UnlockDiagnosticVMOptions -XX:+PrintAssembly java文件 >> 1.txt
```
生成汇编代码导入到1.txt文件

查看汇编代码可知，在i的赋值之前判断如果是多处理器就的lock addl $0x0,(%rsp) 指令，这条指令加上了lock前缀，这个对寄存器加0的空操作，没有实际意义，只是保证当前cpu持有了总线锁，使本cpu的cache写入主内存，这个操作也会被其他cpu的cache嗅探到从而更新缓存，因此这条指令使得线程只能单条通过且更新内存，保证了内存对其他cpu得可见性。

### CPU级别的实现

cpu级别的实现除了lock总线锁以外还有一种实现是高速缓存锁。

## 例：dcl单例是否需要加volatile?（Double Check Lock）
关于单例实现中的双重锁定方式
```
public class Singleton{
    private static Singleton instance = null;
    private Singleton(){}
    public static Singleton getInstance(){ ----------------------- A
        if(instance == null){              ----------------------- B
            synchronized(Singleton.class){ ----------------------- C
                if(instance == null){      ----------------------- D
                    instance = new Singleton(); ------------------ E
                }
            }
        }
        return instance;
    }
}
```
关于此种方式中添加了synchronized锁，保证其中的代码只有一个线程能够执行，且保证加锁的代码具有原子性，理论上来说这一块儿代码不会有什么问题，但是由于jvm会对指令顺序进行优化，那么对于instance = new Singleton()这句代码，反汇编机器码出来的汇编指令包括8条汇编指令。

主要做了三件事：
1. 为Singleton对象分配内存空间
2. 初始化Singleton对象的构造器
3. 将instance对象指向分配的内存空间 （到这一步认为instance对象非空）

由于jvm为了优化指令，提高代码效率，允许指令重排序，这样一条代码的指令顺序会被优化成如下的样子：

1. 为Singleton对象分配内存空间           
2. 将instancde对象指向分配的内存空间 （到这一步认为instance对象非空）
3. 初始化Singleton对象的构造器

因此当线程一执行到E的优化后的第二条指令时，就认为instance非空了，如果这时候恰好线程二执行到B，判断instance非空，并返回了这个instance，由于instance尚未初始化，使用这个对象的时候就报错了。

因此dcl写法改为:

```
public class Singleton{
    private volatile static Singleton instance = null;
    private Singleton(){}
    public static Singleton getInstance(){ 
        if(instance == null){              
            synchronized(Singleton.class){ 
                if(instance == null){      
                    instance = new Singleton(); 
                }
            }
        }
        return instance;
    }
}
```