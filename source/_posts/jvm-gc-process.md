---
title: jvm垃圾收集过程
date: 2020-04-26 10:26:53
tags:
- java
- gc
---
# jvm垃圾收集过程
1. 一般对象产生会首先放到Eden区，Eden区空间不足则会放入到From Survivor区，如果仍然放不下，就会触发一次新生代的Minor GC。
   - 在Eden区中存活的对象会放入To Survivor区中
   - 在From Survivor区中存活的对象会放入To Survivor区中，同时给From Survivor区的对象年龄+1，如果达到阈值，就会被放入老年代
   - 最后会清空Eden区与From Survivor区，不会产生内存碎片，如果在一次Minor GC结束以后，To Survivor区放不下存活的对象，就将这些对象移入老年代
   - 大对象会直接放入老年代
2. 当执行完一次Minor GC以后，有对象移入老年代，就得分析有没有超过老年代最大剩余空间，超过了就会触发Full GC。