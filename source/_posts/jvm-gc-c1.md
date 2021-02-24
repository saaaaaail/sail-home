---
title: jvm垃圾收集器G1
date: 2020-04-24 20:45:09
tags:
- jvm
- G1
- gc
---
# jvm垃圾收集器G1
## G1（Garbage-First）
G1面向服务端应用的垃圾收集器。  

其他垃圾收集器将堆区分为新生代与老年代来进行垃圾收集，而G1直接对新生代与老年代一起回收。  

G1将堆划分为多个大小相同的region，记录region垃圾回收的时间与回收获得的空间，维护一个垃圾回收的优先列表。便于预测停顿时间模型。region被分类为Eden、Survivor、Old、Humongous四种类型。

region被分为多个card，一个card分片一般为512Bytes。

每个region都具有自己的Remembered Set（Rset），是point into的设计，即逻辑上记录的是其他region的card中的对象（这些对象引用了当前region中某一card的某个对象），实际上Rset的设计是一个HashTable，一条记录的key值是其他region的起始地址，value是一个集合，记录region中的card数组的索引值。

## GC过程
G1提供了两种GC方式
- Young GC: 回收所有年轻代的Region。当Eden的Region不能再分配内存就会触发，促发以后的GC与普通新生代GC的复制过程类似，不过其标记阶段除了使用GC Root还使用了RSet。
- Mixed GC: 回收所有年轻代的Region与部分老年代的Region。根据预测停顿模型选择回收收益高的old Region满足指定用户开销。-XX:MaxGCPauseMillis 指定G1停顿时间的预测值。


### Mixed GC
主要分为两个阶段:

#### 1.全局并发标记阶段:
   - 初始标记: stw，标记GC Root对象以及GC Root直接可达的对象，将这些对象压入扫描栈，并使用外部的bitmap记录mark信息。
   - 并发标记: 进行GC Root Tracing，同时会记录这段时间对象引用状态变化到RSet Log里，即不断从扫描栈取出对象引用，在bitmap里将对象mark，然后将对象引用的对象压入扫描栈。
   - 最终标记: stw，使用RSet Log里的变化修正Rset的引用关系。注意这里最终扫描的是SATB Buffer而不是整个young gen（CMS的最终标记里扫描的就是整个根集合），SATB Buffer是在并发标记阶段，利用write barrier将新插入的引用关系记录下来；利用pre write buffer将即将被删除的引用关系记录下来，用这些新的引用关系与旧的引用关系为根，重新扫描一遍避免漏标。
   - 清除垃圾: stw，根据next bitmap里存活对象以及RSet统计存活的对象，对RSet进行排序用于下一次CSet收集Region，根据一个Region的Rset有没有其他Region引用，如果完全没有就认为没有活对象，可以整体清除Region。


#### 2.拷贝存活对象:
   - stw，1、依赖于停顿预测模型，选择停顿少，回收内存多的region进行收集，构成Collect Set，2、扫描这些Region的RSet以及根对象，确认存活的对象，把存活的对象集中拷贝到空闲的Region中，然后回收这些Region。

## 特点
- 空间整合: 符合停顿预测模型的标记-整理算法，局部的region存活对象拷贝到同一region中，减少内存碎片。
- 可预测GC停顿时间
- 只处理部分Region而不是全堆扫，对大内存比较高效


### 介绍一下G1 
G1是一个基于可停顿预测模型的多线程的垃圾处理器，这个处理器同时管理整个堆，将堆内容划分为大小相等的多个Region，Region依然被划分为Eden、Suvivor、Old这些区。

Region的垃圾处理过程分为三块儿，第一块儿是young GC，与普通的新生代GC类似，如果Eden分区满了向其中使用着的Suvivor区放，如果也放不下触发young GC。

然后第二块就是全局并发标记，首先初始标记stw，找到GC Root以及跟它直接管理的对象进行标记，然后并发标记，与用户线程一起运行，进行GC Root tracing，标记所有存活对象，期间将对象引用的变化保存到RSet Log里面。然后第三阶段最终标记将RsetLog里的变化修正到RSet里面，同时修改对象的标记关系，漏标的标上，多标的去除标记，第四阶段清除，做两件事吧，根据bitmap与RSet统计不同Region里面存活对象的情况，对Region进行排序。第二件事将没有引用指向当前Region的Region回收掉。 第三块儿，进行存活对象拷贝，根据停顿预测模型选择部分Region构成CSet，将里面存活的对象集中拷贝到另一个Region。回收当前Region。

优点就是使用了复制算法，没有内存碎片
可以预测GC停顿时间
可以处理大内存GC，因为只处理部分Region不是全堆扫