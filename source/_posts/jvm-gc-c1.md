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

每个region都具有自己的Remembered Set（Rset），是point into的设计，即逻辑上记录的是其他region的card中的对象（这些对象引用了当前region中某一card的某个对象），实际上Rset的设计是一个HashTable，一条记录的key值是其他region的起始地址，value是一个集合，记录当前region的card索引值。

## GC过程
G1提供了两种GC方式
- Young GC: 回收所有年轻代的Region。当Eden的Region不能再分配内存就会触发，促发以后的GC与普通新生代GC的复制过程类似。
- Mixed GC: 回收所有年轻代的Region与部分老年代的Region。根据预测停顿模型选择回收收益高的old Region满足指定用户开销。-XX:MaxGCPauseMillis 指定G1停顿时间的预测值。

### Mixed GC
主要分为两个阶段:

1. 全局并发标记阶段:
   - 初始标记: stw，标记GC Root直接可达的对象。
   - 并发标记: 进行GC Root Tracing，同时会记录这段时间对象引用状态变化。
   - 最终标记: stw，修正Rset的引用关系。
   - 清除垃圾: stw，根据一个Region的Rset有没有其他Region引用，如果完全没有就认为没有活对象，可以整体清除Region。
2. 拷贝存活对象:
   - stw，把一部分region的活对象拷贝到空的region中去，然后回收原本的region空间。如何选择region并构成Collection Set（Cset），依赖于停顿预测模型，选择停顿少，回收内存多的region并满足用户指定的开销。

## 特点
- 空间整合: 符合停顿预测模型的标记-整理算法，局部的region存活对象拷贝到同一region中，减少内存碎片。
- 可预测停顿
