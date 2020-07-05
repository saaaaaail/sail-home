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


